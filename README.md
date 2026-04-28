<<<<<<< HEAD
# DecodePlayControl 功能模块代码解析

## 目录
- [1. 功能模块总览](#1-功能模块总览)
- [2. 功能模块间联系流程图](#2-功能模块间联系流程图)
- [3. 各功能模块详细解析](#3-各功能模块详细解析)
- [4. 代码执行流程](#4-代码执行流程)

---

## 1. 功能模块总览

本应用包含 **7个核心功能模块**，每个模块由前端UI代码、业务逻辑代码和Native底层代码组成。

| 功能模块 | 功能描述 | 前端代码 | 业务逻辑代码 | Native代码 |
|---------|---------|---------|------------|-----------|
| **模块1: 应用初始化** | 加载资源、创建播放器 | Index.ets | VideoPlayViewModel.initialize() | PlayerNative.createPlayer() |
| **模块2: 视频播放** | 解码播放视频 | VideoPlayView.XComponent | VideoPlayViewModel.onVideoSelected() | Player.Start() |
| **模块3: 播放控制** | 播放/暂停/恢复 | VideoPlayView.PlayPauseButton | VideoPlayViewModel.togglePlayPause() | Player.Pause/Resume() |
| **模块4: 进度控制** | Seek定位 | VideoPlayView.TimeSlider | VideoPlayViewModel.onSliderChange() | Player.Seek() |
| **模块5: 倍速播放** | 调整播放速度 | VideoPlayView.SpeedButton | VideoPlayViewModel.onSpeedSelected() | Player.SetSpeed() |
| **模块6: 视频切换** | 切换视频资源 | VideoPlayView.VideoSelectionView | VideoPlayViewModel.onVideoSelected() | Player.Release() + Create() |
| **模块7: 全屏播放** | 横屏全屏 | VideoPlayView.FullScreenButton | VideoPlayView.fullScreenSet() | Window API |

---

## 2. 功能模块间联系流程图

### 2.1 整体功能模块关系图

```mermaid
graph TB
    subgraph "用户界面层"
        UI1[Index.ets<br/>主页面]
        UI2[VideoPlayView.ets<br/>视频播放视图]
    end
    
    subgraph "业务逻辑层"
        BL1[VideoPlayViewModel.ets<br/>视图模型]
    end
    
    subgraph "Native接口层"
        NI1[Index.d.ts<br/>接口声明]
        NI2[libplayer.so<br/>Native库]
    end
    
    subgraph "功能模块"
        M1[模块1: 应用初始化]
        M2[模块2: 视频播放]
        M3[模块3: 播放控制]
        M4[模块4: 进度控制]
        M5[模块5: 倍速播放]
        M6[模块6: 视频切换]
        M7[模块7: 全屏播放]
    end
    
    UI1 --> UI2
    UI2 --> BL1
    BL1 --> NI1
    NI1 --> NI2
    
    M1 --> M2
    M2 --> M3
    M2 --> M4
    M2 --> M5
    M6 --> M2
    M7 -.-> UI2
    
    BL1 --> M1
    BL1 --> M2
    BL1 --> M3
    BL1 --> M4
    BL1 --> M5
    BL1 --> M6
    UI2 --> M7
    
    style M1 fill:#e1f5ff
    style M2 fill:#fff4e1
    style M3 fill:#d4edda
    style M4 fill:#f8d7da
    style M5 fill:#d1ecf1
    style M6 fill:#cce5ff
    style M7 fill:#fff3cd
```

### 2.2 功能模块调用链路图

```mermaid
sequenceDiagram
    participant User as 用户
    participant UI as UI层
    participant VM as ViewModel层
    participant Native as Native层
    participant System as 系统层
    
    Note over User,System: 模块1: 应用初始化
    User->>UI: 启动应用
    UI->>VM: initialize()
    VM->>Native: createPlayer()
    Native-->>VM: 返回播放器对象
    
    Note over User,System: 模块2: 视频播放
    User->>UI: 选择视频
    UI->>VM: onVideoSelected()
    VM->>Native: init() + play()
    Native->>System: 开始解码渲染
    System-->>UI: 显示视频画面
    
    Note over User,System: 模块3: 播放控制
    User->>UI: 点击播放/暂停
    UI->>VM: togglePlayPause()
    VM->>Native: pause()/resume()
    Native->>System: 控制解码线程
    
    Note over User,System: 模块4: 进度控制
    User->>UI: 拖动进度条
    UI->>VM: onSliderChange()
    VM->>Native: seekVideo()
    Native->>System: 定位到指定位置
    
    Note over User,System: 模块5: 倍速播放
    User->>UI: 选择倍速
    UI->>VM: onSpeedSelected()
    VM->>Native: setSpeed()
    Native->>System: 调整播放速度
    
    Note over User,System: 模块6: 视频切换
    User->>UI: 切换视频
    UI->>VM: onVideoSelected()
    VM->>Native: releasePlayer()
    VM->>Native: createPlayer()
    VM->>Native: init() + play()
    
    Note over User,System: 模块7: 全屏播放
    User->>UI: 点击全屏
    UI->>UI: fullScreenSet()
    UI->>System: setPreferredOrientation()
```

### 2.3 数据流向图

```mermaid
graph LR
    subgraph "输入"
        A1[用户操作]
        A2[视频文件]
    end
    
    subgraph "处理"
        B1[UI事件]
        B2[业务逻辑]
        B3[Native处理]
    end
    
    subgraph "输出"
        C1[UI更新]
        C2[视频画面]
        C3[音频播放]
    end
    
    A1 --> B1
    A2 --> B3
    B1 --> B2
    B2 --> B3
    B3 --> C1
    B3 --> C2
    B3 --> C3
    
    style A1 fill:#e1f5ff
    style C2 fill:#d4edda
    style C3 fill:#d4edda
```

---

## 3. 各功能模块详细解析

### 模块1: 应用初始化

#### 功能描述
应用启动时加载视频资源、创建播放器实例、初始化窗口属性。

#### 代码组成

```mermaid
graph TB
    subgraph "前端代码"
        A1["Index.ets (31-43行)<br/>aboutToAppear()"]
        A2["加载视频资源<br/>getRawFdSync()"]
    end
    
    subgraph "业务逻辑代码"
        B1["VideoPlayViewModel.ets (49-57行)<br/>initialize()"]
        B2["VideoPlayViewModel.ets (59-74行)<br/>setWindowProperties()"]
        B3["VideoPlayViewModel.ets (76-83行)<br/>registerEventListeners()"]
        B4["VideoPlayViewModel.ets (94-109行)<br/>startTimer()"]
    end
    
    subgraph "Native代码"
        C1["Index.d.ts (24行)<br/>createPlayer()"]
        C2["PlayerNative.cpp<br/>CreatePlayer()"]
        C3["Player.cpp<br/>Player构造函数"]
    end
    
    A1 --> A2
    A1 --> B1
    B1 --> B2
    B1 --> B3
    B1 --> B4
    B1 --> C1
    C1 --> C2
    C2 --> C3
    
    style A1 fill:#e1f5ff
    style B1 fill:#fff4e1
    style C1 fill:#d4edda
```

#### 关键代码片段

**前端代码 - Index.ets (31-43行)**
```typescript
aboutToAppear(): void {
  try {
    this.videoResources.length = 0;
    for (let i = 0; i< VIDEO_NAMES.length; i++) {
      let rawDes = this.getUIContext().getHostContext()?.resourceManager.getRawFdSync(VIDEO_NAMES[i]);
      if (rawDes) {
        this.videoResources.push(rawDes);
      }
    }
  } catch (error) {
    hilog.error(0x0000, TAG, `aboutToAppear catch error, code: ${error.code}, message: ${error.message}`);
  }
}
```

**业务逻辑代码 - VideoPlayViewModel.ets (49-57行)**
```typescript
async initialize(): Promise<void> {
  try {
    await this.setWindowProperties();
    this.registerEventListeners();
    this.startTimer();
  } catch (error) {
    hilog.error(DOMAIN, TAG, `initialize catch error, code: ${error.code}, message: ${error.message}`);
  }
}
```

**Native接口 - Index.d.ts (24行)**
```typescript
export const createPlayer: () => bigint;
```

#### 执行流程
1. Index.ets 页面加载时调用 `aboutToAppear()`
2. 加载3个视频文件资源到 `videoResources` 数组
3. VideoPlayView 初始化时调用 `viewModel.initialize()`
4. 设置窗口属性（状态栏颜色、保持屏幕常亮）
5. 注册前后台事件监听
6. 启动定时器用于更新播放进度

---

### 模块2: 视频播放

#### 功能描述
选择视频后，创建解码器、解封装器，开始解码播放视频。

#### 代码组成

```mermaid
graph TB
    subgraph "前端代码"
        A1["VideoPlayView.ets (355-360行)<br/>onVideoSelected()"]
        A2["VideoPlayView.ets (87-97行)<br/>XComponentView()"]
    end
    
    subgraph "业务逻辑代码"
        B1["VideoPlayViewModel.ets (197-241行)<br/>onVideoSelected()"]
        B2["创建播放器"]
        B3["初始化播放器"]
        B4["开始播放"]
    end
    
    subgraph "Native代码"
        C1["Index.d.ts (24行)<br/>createPlayer()"]
        C2["Index.d.ts (40-45行)<br/>init()"]
        C3["Index.d.ts (28行)<br/>play()"]
        C4["PlayerNative.cpp<br/>NAPI实现"]
        C5["Player.cpp<br/>Start()"]
        C6["Demuxer.cpp<br/>Open()"]
        C7["VideoDecoder.cpp<br/>Init()"]
        C8["AudioDecoder.cpp<br/>Init()"]
    end
    
    A1 --> B1
    A2 -.-> C4
    B1 --> B2
    B1 --> B3
    B1 --> B4
    B2 --> C1
    B3 --> C2
    B4 --> C3
    C1 --> C4
    C2 --> C4
    C3 --> C4
    C4 --> C5
    C5 --> C6
    C5 --> C7
    C5 --> C8
    
    style A1 fill:#e1f5ff
    style B1 fill:#fff4e1
    style C5 fill:#d4edda
```

#### 关键代码片段

**前端代码 - VideoPlayView.ets (355-360行)**
```typescript
private async onVideoSelected(index: number, rawDes: resourceManager.RawFileDescriptor): Promise<void> {
  const newState = await this.viewModel.onVideoSelected(index, rawDes);
  if (newState !== null) {
    this.viewModel.playState = newState;
  }
}
```

**业务逻辑代码 - VideoPlayViewModel.ets (197-241行)**
```typescript
async onVideoSelected(index: number, rawDes: resourceManager.RawFileDescriptor): Promise<PlayerState | null> {
  if (this.chooseNumber === index) {
    return null;
  }
  
  this.isSwitchEnable = false;
  this.durationTime = 0;
  this.chooseNumber = index;
  this.isUse = false;
  this.chooseSpeed = Const.VIDEO_DEFAULT_SPEED;
  this.speedText = $r('app.string.Default_speed_text');
  
  if (!rawDes) {
    hilog.error(DOMAIN, TAG, 'player inputFile is null');
    this.isSwitchEnable = true;
    return null;
  }
  
  try {
    await this.releasePlayer();
    this.nativePlayerObj = player.createPlayer();
    const data = await player.init(this.nativePlayerObj, rawDes.fd, rawDes.offset, rawDes.length);
    this.isSeek = false;
    
    if (data.code === 0) {
      this.durationTime = data.durationTime / Const.TIME_RATIO;
      this.isUse = true;
      this.isSwitchEnable = true;
      
      player.play(this.nativePlayerObj);
      return PlayerState.PLAYING;
    } else {
      hilog.error(DOMAIN, TAG, 'player init failed, err code is ' + data.code);
      await this.releasePlayer();
    }
    
    this.isSwitchEnable = true;
  } catch (error) {
    hilog.error(DOMAIN, TAG, `Switch video failed: ${error.code}, message: ${error.message}`);
    await this.releasePlayer();
    this.isSwitchEnable = true;
  }
  
  return null;
}
```

**Native接口 - Index.d.ts**
```typescript
export const createPlayer: () => bigint;

export const init: (
  objAddr: bigint,
  inputFileFd: number,
  inputFileOffset: number,
  inputFileSize: number
) => Promise<Response>;

export const play: (objAddr: bigint) => void;
```

#### 执行流程
1. 用户点击视频列表项，触发 `onVideoSelected()`
2. 调用 `releasePlayer()` 释放旧播放器
3. 调用 `createPlayer()` 创建新播放器实例
4. 调用 `init()` 初始化播放器（传入文件描述符）
5. 调用 `play()` 开始播放
6. Native层创建解封装器、视频解码器、音频解码器
7. 启动解码线程，开始解码渲染

---

### 模块3: 播放控制

#### 功能描述
控制视频的播放、暂停、恢复状态。

#### 代码组成

```mermaid
graph TB
    subgraph "前端代码"
        A1["VideoPlayView.ets (154-167行)<br/>PlayPauseButton()"]
        A2["播放/暂停按钮"]
    end
    
    subgraph "业务逻辑代码"
        B1["VideoPlayViewModel.ets (132-145行)<br/>togglePlayPause()"]
        B2["状态判断"]
        B3["状态切换"]
    end
    
    subgraph "Native代码"
        C1["Index.d.ts (30行)<br/>pause()"]
        C2["Index.d.ts (32行)<br/>resume()"]
        C3["PlayerNative.cpp<br/>Pause/Resume()"]
        C4["Player.cpp<br/>Pause/Resume()"]
        C5["阻塞/唤醒解码线程"]
    end
    
    subgraph "状态模型"
        D1["PlayerStateModel.ets<br/>IDLE/PLAYING/PAUSE"]
    end
    
    A1 --> A2
    A2 --> B1
    B1 --> B2
    B1 --> B3
    B1 --> C1
    B1 --> C2
    C1 --> C3
    C2 --> C3
    C3 --> C4
    C4 --> C5
    B2 --> D1
    B3 --> D1
    
    style A1 fill:#e1f5ff
    style B1 fill:#fff4e1
    style C4 fill:#d4edda
```

#### 关键代码片段

**前端代码 - VideoPlayView.ets (154-167行)**
```typescript
@Builder
PlayPauseButton() {
  Row() {
    Image(this.viewModel.playState === PlayerState.PLAYING ?
      $r('app.media.ic_public_pause') : $r('app.media.ic_public_play'))
      .width('25vp')
      .height('25vp')
      .onClick(() => {
        this.viewModel.playState = this.viewModel.togglePlayPause(this.viewModel.playState);
      })
  }
  .width('25vp')
  .height('100%')
  .justifyContent(FlexAlign.Center)
}
```

**业务逻辑代码 - VideoPlayViewModel.ets (132-145行)**
```typescript
togglePlayPause(playState: PlayerState): PlayerState {
  if (this.nativePlayerObj === BigInt(0)) {
    return playState;
  }
  
  if (playState === PlayerState.PLAYING) {
    player.pause(this.nativePlayerObj);
    return PlayerState.PAUSE;
  } else if (playState === PlayerState.PAUSE) {
    player.resume(this.nativePlayerObj);
    return PlayerState.PLAYING;
  }
  return playState;
}
```

**Native接口 - Index.d.ts**
```typescript
export const pause: (objAddr: bigint) => void;
export const resume: (objAddr: bigint) => void;
```

#### 执行流程
1. 用户点击播放/暂停按钮
2. 根据当前状态判断执行操作
3. 如果正在播放，调用 `pause()` 暂停
4. 如果已暂停，调用 `resume()` 恢复
5. Native层阻塞或唤醒解码线程
6. 更新UI状态显示

---

### 模块4: 进度控制

#### 功能描述
通过进度条控制视频播放位置，实现Seek定位功能。

#### 代码组成

```mermaid
graph TB
    subgraph "前端代码"
        A1["VideoPlayView.ets (169-196行)<br/>TimeSlider()"]
        A2["Slider组件"]
        A3["当前时间显示"]
        A4["总时长显示"]
    end
    
    subgraph "业务逻辑代码"
        B1["VideoPlayViewModel.ets (147-159行)<br/>onSliderChange()"]
        B2["VideoPlayViewModel.ets (161-174行)<br/>performSeek()"]
        B3["VideoPlayViewModel.ets (94-109行)<br/>startTimer()"]
        B4["定时更新进度"]
    end
    
    subgraph "Native代码"
        C1["Index.d.ts (34行)<br/>getRenderTime()"]
        C2["Index.d.ts (38行)<br/>seekVideo()"]
        C3["PlayerNative.cpp<br/>Seek()"]
        C4["Player.cpp<br/>Seek()"]
        C5["Demuxer.cpp<br/>SeekTo()"]
        C6["定位到关键帧"]
    end
    
    subgraph "工具类"
        D1["TimeUtils.ets<br/>getShowTime()"]
        D2["CommonConstants.ets<br/>TIME_RATIO"]
    end
    
    A1 --> A2
    A1 --> A3
    A1 --> A4
    A2 --> B1
    B1 --> B2
    B3 --> B4
    B4 --> C1
    B2 --> C2
    C1 --> C3
    C2 --> C3
    C3 --> C4
    C4 --> C5
    C5 --> C6
    A3 --> D1
    A4 --> D1
    B3 --> D2
    
    style A1 fill:#e1f5ff
    style B1 fill:#fff4e1
    style C4 fill:#d4edda
```

#### 关键代码片段

**前端代码 - VideoPlayView.ets (169-196行)**
```typescript
@Builder
TimeSlider() {
  Row() {
    Text(getShowTime(this.viewModel.currentTime))
      .fontColor(Color.White)
      .fontSize('12fp')

    Slider({
      value: this.viewModel.currentTime,
      min: 0,
      max: this.viewModel.durationTime
    })
      .width(this.isFullScreen ? '90%' : '70%')
      .selectedColor('#F67609')
      .trackColor('#464642')
      .enabled(!this.viewModel.isSeek)
      .onChange((value: number, mode: SliderChangeMode) => {
        this.viewModel.onSliderChange(value, mode);
      })

    Text(getShowTime(this.viewModel.durationTime))
      .fontColor(Color.White)
      .fontSize('12fp')
  }
  .width(this.isFullScreen ? '90%' : '70%')
  .height('100%')
  .justifyContent(FlexAlign.Center)
}
```

**业务逻辑代码 - VideoPlayViewModel.ets (147-174行)**
```typescript
onSliderChange(value: number, mode: SliderChangeMode): void {
  this.currentTime = value;
  
  if (this.durationTime > 0) {
    if (mode === SliderChangeMode.Begin) {
      this.isSeek = true;
    }
    
    if (mode === SliderChangeMode.End) {
      this.performSeek(value);
    }
  }
}

async performSeek(value: number): Promise<void> {
  if (this.nativePlayerObj === BigInt(0)) {
    return;
  }
  
  try {
    await player.seekVideo(this.nativePlayerObj, value);
    setTimeout(() => {
      this.isSeek = false;
    }, Const.RELOAD_TIME);
  } catch (error) {
    hilog.error(DOMAIN, TAG, `Seek failed: ${error.code}, message: ${error.message}`);
  }
}
```

**Native接口 - Index.d.ts**
```typescript
export const getRenderTime: (objAddr: bigint) => number;
export const seekVideo: (objAddr: bigint, desTime: number) => Promise<void>;
```

#### 执行流程
1. 定时器每秒调用 `getRenderTime()` 获取当前播放时间
2. 更新进度条显示
3. 用户拖动进度条时，触发 `onSliderChange()`
4. 开始拖动时设置 `isSeek = true` 停止更新
5. 结束拖动时调用 `performSeek()` 执行Seek
6. Native层定位到目标位置的关键帧
7. 清空缓冲区，重新填充数据
8. 恢复播放，设置 `isSeek = false`

---

### 模块5: 倍速播放

#### 功能描述
支持1.0X、2.0X、3.0X倍速播放，调整音频渲染速度。

#### 代码组成

```mermaid
graph TB
    subgraph "前端代码"
        A1["VideoPlayView.ets (198-211行)<br/>SpeedButton()"]
        A2["VideoPlayView.ets (230-277行)<br/>SpeedListOverlay()"]
        A3["倍速选择列表"]
    end
    
    subgraph "业务逻辑代码"
        B1["VideoPlayViewModel.ets (176-184行)<br/>toggleSpeedList()"]
        B2["VideoPlayViewModel.ets (186-195行)<br/>onSpeedSelected()"]
        B3["更新速度文本"]
    end
    
    subgraph "Native代码"
        C1["Index.d.ts (36行)<br/>setSpeed()"]
        C2["PlayerNative.cpp<br/>SetSpeed()"]
        C3["Player.cpp<br/>SetSpeed()"]
        C4["AudioDecoder.cpp<br/>SetSpeed()"]
        C5["调整音频渲染速度"]
    end
    
    subgraph "常量定义"
        D1["CommonConstants.ets<br/>VIDEO_SPEED_ARR"]
        D2["CommonConstants.ets<br/>VIDEO_SPEED_TEXT_ARR"]
    end
    
    A1 --> B1
    A2 --> A3
    A3 --> B2
    B1 --> A2
    B2 --> B3
    B2 --> C1
    C1 --> C2
    C2 --> C3
    C3 --> C4
    C4 --> C5
    A3 --> D1
    A3 --> D2
    
    style A1 fill:#e1f5ff
    style B2 fill:#fff4e1
    style C3 fill:#d4edda
```

#### 关键代码片段

**前端代码 - VideoPlayView.ets (198-211行, 230-277行)**
```typescript
@Builder
SpeedButton() {
  Row() {
    Text(this.viewModel.speedText)
      .fontColor(Color.White)
      .fontSize('14fp')
  }
  .width(this.isFullScreen ? '5%' : '14%')
  .height('100%')
  .justifyContent(FlexAlign.Center)
  .onClick(() => {
    this.viewModel.toggleSpeedList();
  })
}

@Builder
SpeedListOverlay() {
  Row() {
    Column() {
      List({ space: 0, initialIndex: 2 }) {
        ForEach(Const.VIDEO_SPEED_ARR, (item: number, index: number) => {
          ListItem() {
            Column() {
              Text(Const.VIDEO_SPEED_TEXT_ARR[index])
                .fontSize('16fp')
                .fontColor(this.viewModel.chooseSpeed === item ? '#F67609' : '#FFFFFF')
            }
            .width('100%')
            .height('50vp')
            .justifyContent(FlexAlign.Center)
            .alignItems(HorizontalAlign.Center)
            .onClick(() => {
              this.viewModel.onSpeedSelected(item, index);
            })
          }
        }, (item: number, index: number) => {
          return 'item: ' + item.toString() + 'index: ' + index.toString();
        })
      }
      .width('100%')
      .height('100%')
      .stackFromEnd(true)
    }
    .width(this.isFullScreen ? '180vp' : '120vp')
    .height('100%')
    .padding({ top: '16vp', bottom: '36vp' })
    .justifyContent(FlexAlign.End)
    .backgroundColor('#08ffffff')
  }
  .width('100%')
  .height('100%')
  .justifyContent(FlexAlign.End)
  .visibility(this.viewModel.isSpeedListVisible ? Visibility.Visible : Visibility.Hidden)
  .onClick(() => {
    this.viewModel.hideSpeedList();
  })
}
```

**业务逻辑代码 - VideoPlayViewModel.ets (176-195行)**
```typescript
toggleSpeedList(): void {
  this.isSpeedListVisible = true;
}

hideSpeedList(): void {
  if (this.isSpeedListVisible) {
    this.isSpeedListVisible = false;
  }
}

onSpeedSelected(speed: number, index: number): void {
  if (this.chooseSpeed !== speed) {
    this.chooseSpeed = speed;
    if (this.nativePlayerObj !== BigInt(0)) {
      player.setSpeed(this.nativePlayerObj, speed);
    }
    this.speedText = Const.VIDEO_SPEED_TEXT_ARR[index];
    this.isSpeedListVisible = false;
  }
}
```

**Native接口 - Index.d.ts (36行)**
```typescript
export const setSpeed: (objAddr: bigint, speed: number) => void;
```

#### 执行流程
1. 用户点击倍速按钮，显示倍速选择列表
2. 用户选择倍速值（1.0X/2.0X/3.0X）
3. 调用 `onSpeedSelected()` 处理选择
4. 调用 `setSpeed()` 设置播放速度
5. Native层调整音频渲染速度
6. 更新音画同步参数
7. 更新UI显示当前倍速

---

### 模块6: 视频切换

#### 功能描述
切换播放不同的视频资源，释放旧播放器，创建新播放器。

#### 代码组成

```mermaid
graph TB
    subgraph "前端代码"
        A1["VideoPlayView.ets (279-344行)<br/>VideoSelectionView()"]
        A2["视频列表"]
        A3["VideoPlayView.ets (355-360行)<br/>onVideoSelected()"]
    end
    
    subgraph "业务逻辑代码"
        B1["VideoPlayViewModel.ets (197-241行)<br/>onVideoSelected()"]
        B2["VideoPlayViewModel.ets (243-252行)<br/>releasePlayer()"]
        B3["重置状态"]
        B4["创建新播放器"]
    end
    
    subgraph "Native代码"
        C1["Index.d.ts (26行)<br/>releasePlayer()"]
        C2["Index.d.ts (24行)<br/>createPlayer()"]
        C3["Index.d.ts (40-45行)<br/>init()"]
        C4["Index.d.ts (28行)<br/>play()"]
        C5["Player.cpp<br/>Release()"]
        C6["释放解码器"]
        C7["释放解封装器"]
    end
    
    A1 --> A2
    A2 --> A3
    A3 --> B1
    B1 --> B3
    B1 --> B2
    B1 --> B4
    B2 --> C1
    B4 --> C2
    B4 --> C3
    B4 --> C4
    C1 --> C5
    C5 --> C6
    C5 --> C7
    
    style A1 fill:#e1f5ff
    style B1 fill:#fff4e1
    style C5 fill:#d4edda
```

#### 关键代码片段

**前端代码 - VideoPlayView.ets (279-344行)**
```typescript
@Builder
VideoSelectionView() {
  Column() {
    Row() {
      Text($r('app.string.Video_name'))
        .fontSize('24fp')
        .fontWeight(FontWeight.Bold)
        .fontColor(Color.White)
    }
    .width('100%')
    .height('25%')
    .justifyContent(FlexAlign.Start)
    .padding({ left: '20vp', right: '20vp' })

    Row() {
      Text($r('app.string.Episodes'))
        .fontSize('16fp')
        .fontWeight(FontWeight.Regular)
        .fontColor(Color.White)
    }
    .width('100%')
    .height('20%')
    .padding({ left: '20vp', right: '20vp' })

    Row() {
      List({ space: '16vp', initialIndex: 0 }) {
        ForEach(this.videoSources, (item: resourceManager.RawFileDescriptor, index: number) => {
          ListItem() {
            Column() {
              Text((index + 1).toString())
                .fontSize('20fp')
                .fontColor(this.viewModel.chooseNumber === index && this.viewModel.isUse ? '#F67609' : '#FFFFFF')
            }
            .width(40)
            .height(40)
            .backgroundColor('#3B3A37')
            .borderRadius($r('sys.float.corner_radius_level5'))
            .justifyContent(FlexAlign.Center)
            .alignItems(HorizontalAlign.Center)
            .borderColor(this.viewModel.chooseNumber === index && this.viewModel.isUse ? '#F67609' : '#3B3A37')
            .borderWidth(1)
            .onClick(() => {
              this.onVideoSelected(index, item);
            })
          }
        }, (item: resourceManager.RawFileDescriptor, index: number) => {
          return 'item: ' + item.toString() + 'index: ' + index.toString();
        })
      }
      .width('100%')
      .height('100%')
      .listDirection(Axis.Horizontal)
      .enabled(this.viewModel.isSwitchEnable)
    }
    .width('100%')
    .height('30%')
    .padding({ left: '20vp', right: '20vp' })
  }
  .width('100%')
  .height('35%')
  .linearGradient({
    direction: GradientDirection.Bottom,
    colors: [[0x1b1a1c, 0], [0x000000, 0.5]]
  })
  .visibility(this.isFullScreen ? Visibility.Hidden : Visibility.Visible)
}
```

**业务逻辑代码 - VideoPlayViewModel.ets (197-252行)**
```typescript
async onVideoSelected(index: number, rawDes: resourceManager.RawFileDescriptor): Promise<PlayerState | null> {
  if (this.chooseNumber === index) {
    return null;
  }
  
  this.isSwitchEnable = false;
  this.durationTime = 0;
  this.chooseNumber = index;
  this.isUse = false;
  this.chooseSpeed = Const.VIDEO_DEFAULT_SPEED;
  this.speedText = $r('app.string.Default_speed_text');
  
  if (!rawDes) {
    hilog.error(DOMAIN, TAG, 'player inputFile is null');
    this.isSwitchEnable = true;
    return null;
  }
  
  try {
    await this.releasePlayer();
    this.nativePlayerObj = player.createPlayer();
    const data = await player.init(this.nativePlayerObj, rawDes.fd, rawDes.offset, rawDes.length);
    this.isSeek = false;
    
    if (data.code === 0) {
      this.durationTime = data.durationTime / Const.TIME_RATIO;
      this.isUse = true;
      this.isSwitchEnable = true;
      
      player.play(this.nativePlayerObj);
      return PlayerState.PLAYING;
    } else {
      hilog.error(DOMAIN, TAG, 'player init failed, err code is ' + data.code);
      await this.releasePlayer();
    }
    
    this.isSwitchEnable = true;
  } catch (error) {
    hilog.error(DOMAIN, TAG, `Switch video failed: ${error.code}, message: ${error.message}`);
    await this.releasePlayer();
    this.isSwitchEnable = true;
  }
  
  return null;
}

async releasePlayer(): Promise<void> {
  if (this.nativePlayerObj !== BigInt(0)) {
    try {
      player.releasePlayer(this.nativePlayerObj);
      this.nativePlayerObj = BigInt(0);
    } catch (error) {
      hilog.error(DOMAIN, TAG, `Release player failed: ${error.code}, message: ${error.message}`);
    }
  }
}
```

**Native接口 - Index.d.ts**
```typescript
export const releasePlayer: (objAddr: bigint) => void;
export const createPlayer: () => bigint;
export const init: (
  objAddr: bigint,
  inputFileFd: number,
  inputFileOffset: number,
  inputFileSize: number
) => Promise<Response>;
export const play: (objAddr: bigint) => void;
```

#### 执行流程
1. 用户点击视频列表中的视频项
2. 判断是否为当前播放视频，是则不处理
3. 禁用切换按钮，重置状态
4. 调用 `releasePlayer()` 释放旧播放器
5. Native层释放解码器、解封装器等资源
6. 调用 `createPlayer()` 创建新播放器
7. 调用 `init()` 初始化新视频
8. 调用 `play()` 开始播放新视频
9. 启用切换按钮

---

### 模块7: 全屏播放

#### 功能描述
切换横屏全屏播放模式，调整窗口方向和布局。

#### 代码组成

```mermaid
graph TB
    subgraph "前端代码"
        A1["VideoPlayView.ets (213-228行)<br/>FullScreenButton()"]
        A2["VideoPlayView.ets (49-65行)<br/>fullScreenSet()"]
        A3["VideoPlayView.ets (116-136行)<br/>FullScreenHeaderView()"]
        A4["全屏按钮"]
        A5["返回按钮"]
    end
    
    subgraph "窗口API"
        B1["window.getLastWindow()"]
        B2["setPreferredOrientation()"]
        B3["setWindowLayoutFullScreen()"]
    end
    
    subgraph "状态管理"
        C1["isFullScreen状态"]
        C2["布局调整"]
    end
    
    A1 --> A4
    A4 --> C1
    A3 --> A5
    A5 --> C1
    C1 --> A2
    A2 --> B1
    B1 --> B2
    B1 --> B3
    C1 --> C2
    
    style A1 fill:#e1f5ff
    style A2 fill:#fff4e1
    style B2 fill:#d4edda
```

#### 关键代码片段

**前端代码 - VideoPlayView.ets (213-228行, 49-65行)**
```typescript
@Builder
FullScreenButton() {
  if (!this.isFullScreen) {
    Row() {
      Image($r('app.media.ic_public_enlarge'))
        .width('25vp')
        .height('25vp')
    }
    .width('8%')
    .height('100%')
    .justifyContent(FlexAlign.Center)
    .onClick(() => {
      this.isFullScreen = !this.isFullScreen;
    })
  }
}

fullScreenSet() {
  // Change window orientation and layout when setting full screen.
  window.getLastWindow(this.getUIContext().getHostContext()).then((topWindow) => {
    topWindow.setPreferredOrientation(this.isFullScreen ?
      window.Orientation.AUTO_ROTATION_LANDSCAPE : window.Orientation.PORTRAIT).catch((error: BusinessError) => {
      hilog.error(DOMAIN, TAG, `Failed to setPreferredOrientation. Cause: ${error.code}, message: ${error.message}`);
    });
    topWindow.setWindowLayoutFullScreen(this.isFullScreen ? true : false).catch((error: BusinessError) => {
      hilog.error(DOMAIN, TAG,
        `Failed to setWindowLayoutFullScreen. Cause: ${error.code}, message: ${error.message}`);
    });
  }).catch((error: BusinessError) => {
    hilog.error(DOMAIN, TAG, `Failed to getLastWindow. Cause: ${error.code}, message: ${error.message}`);
  });
}
```

**前端代码 - VideoPlayView.ets (116-136行)**
```typescript
@Builder
FullScreenHeaderView() {
  Row() {
    Image($r('app.media.back'))
      .width('40vp')
      .height('40vp')
      .onClick(() => {
        this.isFullScreen = false;
      })

    Text($r('app.string.Video_name'))
      .fontColor(Color.White)
      .fontSize('20fp')
      .fontWeight(FontWeight.Medium)
      .margin({ left: '12vp' })
  }
  .width('100%')
  .height('24%')
  .justifyContent(FlexAlign.Start)
  .padding({ left: '36vp' })
  .visibility(this.isFullScreen ? Visibility.Visible : Visibility.Hidden)
}
```

#### 执行流程
1. 用户点击全屏按钮
2. 切换 `isFullScreen` 状态
3. 触发 `fullScreenSet()` 方法
4. 获取窗口实例
5. 设置窗口方向（横屏/竖屏）
6. 设置全屏布局
7. UI根据 `isFullScreen` 状态调整布局
8. 全屏时显示返回按钮，隐藏视频列表
9. 点击返回按钮退出全屏

---

## 4. 代码执行流程

### 4.1 完整应用启动流程

```mermaid
sequenceDiagram
    participant App as 应用
    participant Index as Index.ets
    participant View as VideoPlayView
    participant VM as VideoPlayViewModel
    participant Native as libplayer.so
    participant Player as Player.cpp
    
    Note over App,Player: 1. 应用启动
    App->>Index: 创建页面
    Index->>Index: aboutToAppear()
    Index->>Index: 加载视频资源 (1.mp4, 2.mp4, 3.mp4)
    
    Note over App,Player: 2. 视图初始化
    Index->>View: 创建VideoPlayView
    View->>VM: initialize()
    VM->>VM: setWindowProperties()
    VM->>VM: registerEventListeners()
    VM->>VM: startTimer()
    
    Note over App,Player: 3. 用户选择视频
    View->>VM: onVideoSelected(0, rawFd)
    VM->>Native: createPlayer()
    Native-->>VM: nativePlayerObj
    VM->>Native: init(nativePlayerObj, fd, offset, length)
    Native->>Player: 初始化解码器
    Player-->>Native: {code: 0, durationTime: xxx}
    Native-->>VM: Response
    VM->>Native: play(nativePlayerObj)
    Native->>Player: Start()
    Player->>Player: 启动解码线程
    
    Note over App,Player: 4. 播放循环
    loop 每秒
        VM->>Native: getRenderTime()
        Native-->>VM: currentTime
        VM->>View: 更新进度条
    end
```

### 4.2 用户交互流程

```mermaid
stateDiagram-v2
    [*] --> 初始化: 应用启动
    初始化 --> 空闲: 加载完成
    
    空闲 --> 播放中: 选择视频
    播放中 --> 暂停: 点击暂停
    暂停 --> 播放中: 点击播放
    
    播放中 --> Seek中: 拖动进度条
    Seek中 --> 播放中: 完成Seek
    
    播放中 --> 倍速播放: 选择倍速
    倍速播放 --> 播放中: 恢复正常速度
    
    播放中 --> 切换中: 切换视频
    切换中 --> 播放中: 新视频开始播放
    
    播放中 --> 全屏播放: 点击全屏
    全屏播放 --> 播放中: 退出全屏
    
    播放中 --> [*]: 退出应用
    暂停 --> [*]: 退出应用
```

### 4.3 Native层解码流程

```mermaid
graph TB
    A[Player.Start] --> B[创建Demuxer]
    B --> C[打开视频文件]
    C --> D[获取音视频轨道]
    
    D --> E[创建VideoDecoder]
    D --> F[创建AudioDecoder]
    
    E --> G[配置视频解码器]
    F --> H[配置音频解码器]
    
    G --> I[启动解码线程]
    H --> I
    
    I --> J{解码循环}
    
    J --> K[Demuxer.ReadSample]
    K --> L{数据包类型}
    
    L -->|视频包| M[VideoDecoder.Decode]
    L -->|音频包| N[AudioDecoder.Decode]
    
    M --> O[获取解码帧]
    N --> P[获取音频数据]
    
    O --> Q[音画同步]
    P --> Q
    
    Q --> R{是否暂停}
    R -->|是| S[阻塞线程]
    R -->|否| T[继续解码]
    
    S --> U{是否恢复}
    U -->|是| T
    U -->|否| S
    
    T --> J
    
    style A fill:#e1f5ff
    style J fill:#fff4e1
    style Q fill:#d4edda
```

---

## 5. 总结

### 5.1 代码组织特点

1. **分层清晰**: 前端UI层、业务逻辑层、Native层职责明确
2. **MVVM架构**: View和ViewModel分离，便于维护
3. **NAPI桥接**: ArkTS与C++高效交互
4. **模块化设计**: 每个功能独立，代码复用性高

### 5.2 功能模块关系

- **模块1（初始化）** 是所有功能的基础
- **模块2（视频播放）** 是核心功能，其他模块依赖它
- **模块3-5（播放控制、进度、倍速）** 是播放功能的扩展
- **模块6（视频切换）** 复用了模块1和模块2的代码
- **模块7（全屏）** 是独立的UI功能

### 5.3 关键技术点

- **XComponent Surface渲染**: 高效的视频渲染方式
- **多线程解码**: input/output线程并行处理
- **音画同步**: 基于时间戳的精确同步
- **资源管理**: 完善的创建和释放机制