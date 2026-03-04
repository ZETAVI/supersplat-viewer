# 核心架构与模块关系

## 1. 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                      index.html (入口页面)                       │
│   解析 URL 参数 → 加载 CSS → 配置 window.sse → 调用 main()       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     index.ts (应用主入口)                         │
│   main(canvas, settingsJson, config)                            │
│   ├── createApp()  → 初始化 PlayCanvas 应用和图形设备              │
│   ├── observe()    → 创建响应式状态 (State)                       │
│   ├── initCanvas() → 设置画布尺寸和 DPI 适配                      │
│   ├── initPoster() → 初始化加载海报                               │
│   ├── initXr()     → 初始化 VR/AR 支持                           │
│   ├── initUI()     → 初始化完整 UI 层                             │
│   ├── loadGsplat() → 加载 3DGS 场景数据                          │
│   ├── loadSkybox() → 加载天空盒纹理                               │
│   ├── VoxelCollider.load() → 加载体素碰撞数据                     │
│   └── new Viewer() → 创建核心查看器实例                            │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Viewer (核心查看器)                           │
│   viewer.ts                                                     │
│   ├── CameraManager    → 相机系统管理                             │
│   ├── InputController  → 输入统一处理                             │
│   ├── Annotations      → 3D 标注系统                              │
│   ├── CameraFrame      → 后处理管线                               │
│   ├── WalkIndicator    → 行走指示器                               │
│   ├── VoxelDebugOverlay→ 碰撞调试叠加                             │
│   └── 渲染循环 (app.on('update')) → 驱动帧更新                     │
└─────────────────────────────────────────────────────────────────┘
```

## 2. 核心模块依赖关系

```
                        types.ts
                   (Config, State, Global)
                    ╱      │      ╲
                   ╱       │       ╲
          app.ts          │      settings.ts
     (PlayCanvas初始化)    │    (配置迁移/解析)
             ╲            │           ╱
              ╲           │          ╱
               ╲          │         ╱
                ╲         ▼        ╱
                 ╲   viewer.ts    ╱
                  ╲  (核心编排)  ╱
                   ╲    │     ╱
                    ╲   │    ╱
           ┌────────┼───┼───┼────────┐
           ▼        ▼   ▼   ▼        ▼
    CameraManager  Input  Picker  Annotations
    (相机管理)    Controller (拾取)  (标注)
         │          │
    ┌────┼────┐     │
    ▼    ▼    ▼     ▼
  Orbit Fly  FPS  InputSources
  Ctrl  Ctrl Ctrl (键鼠/触屏/手柄)
```

## 3. 关键模块详解

### 3.1 Global 对象 (`types.ts`)

`Global` 是贯穿整个应用的共享上下文，所有核心模块都通过它获取依赖：

```typescript
type Global = {
    app: AppBase;              // PlayCanvas 应用实例
    settings: ExperienceSettings; // 场景配置（相机、背景、动画等）
    config: Config;            // 不可变启动配置（URL、特性开关）
    state: State;              // 可观察的运行时状态
    events: EventHandler;      // 全局事件总线
    camera: Entity;            // 相机实体
};
```

### 3.2 响应式状态 (`core/observe.ts`)

项目使用自定义的 Proxy 观察模式来实现响应式状态管理：

```typescript
const state = observe(events, {
    loaded: false,
    progress: 0,
    cameraMode: 'orbit',
    // ... 更多状态字段
});

// 状态变化时自动触发事件
state.loaded = true;  // 自动触发 events.fire('loaded:changed', true)
```

**原理：** `observe()` 函数返回一个 Proxy 对象，在 `set` 陷阱中，比较新旧值并触发 `${property}:changed` 事件。

### 3.3 事件总线 (`EventHandler`)

PlayCanvas 内置的 `EventHandler` 作为全局事件总线，用于模块间解耦通信：

| 事件名 | 触发时机 | 消费者 |
|--------|----------|--------|
| `cameraMode:changed` | 相机模式切换 | CameraManager, UI |
| `loaded:changed` | 场景加载完成 | UI, Viewer |
| `progress:changed` | 加载进度更新 | UI |
| `inputEvent` | 用户输入事件 | CameraManager |
| `pick` | 3D 拾取完成 | CameraManager |
| `walkTo` | 点击行走 | CameraManager |
| `annotation.activate` | 标注被点击 | CameraManager |

### 3.4 应用初始化 (`app.ts`)

```typescript
class App extends AppBase {
    constructor(canvas, options) {
        // 注册组件系统
        CameraComponentSystem      // 相机组件
        LightComponentSystem       // 灯光组件
        RenderComponentSystem      // 渲染组件
        GSplatComponentSystem      // 高斯 Splat 组件 ← 核心
        ScriptComponentSystem      // 脚本组件

        // 注册资源处理器
        ContainerHandler           // 容器资源
        TextureHandler             // 纹理资源
        GSplatHandler              // GSplat 资源处理 ← 核心
        BinaryHandler              // 二进制资源
    }
}
```

## 4. 数据流

### 4.1 场景加载流程

```
URL参数解析 → 配置构建(Config) → 图形设备创建 → PlayCanvas App初始化
                                                       │
                                    ┌──────────────────┼──────────────────┐
                                    ▼                  ▼                  ▼
                              加载GSplat场景      加载天空盒           加载体素数据
                                    │                  │                  │
                                    ▼                  ▼                  ▼
                              创建GSplat实体     设置环境贴图        创建碰撞体
                                    │                  │                  │
                                    └──────────┬───────┘                  │
                                               ▼                         │
                                         Viewer初始化 ←──────────────────┘
                                               │
                                    ┌──────────┼──────────┐
                                    ▼          ▼          ▼
                              初始化相机    设置后处理   开始渲染
```

### 4.2 每帧更新流程

```
app.on('update', dt)
    │
    ├── InputController.update()
    │   ├── KeyboardMouseSource → 收集键鼠输入
    │   ├── MultiTouchSource    → 收集触屏输入
    │   └── GamepadSource       → 收集手柄输入
    │   └── 输出: InputFrame.deltas[]
    │
    ├── CameraManager.update()
    │   ├── ActiveController.update(dt, inputFrame)
    │   │   ├── OrbitController  → 环绕目标旋转
    │   │   ├── FlyController    → 6DoF自由移动（含碰撞）
    │   │   ├── FpsController    → 地面行走（含重力/跳跃）
    │   │   └── AnimController   → 动画轨道播放
    │   ├── 相机位姿过渡插值（模式切换时）
    │   └── 更新 Camera Entity Transform
    │
    ├── Viewer 后处理
    │   ├── applyPostEffectSettings() → Bloom/锐化/色彩/暗角/色差
    │   └── GSplat LOD 更新
    │
    └── PlayCanvas Engine 渲染
```

## 5. 模块通信模式

项目采用**事件驱动 + 共享状态**的架构模式：

1. **共享状态 (Global)** — 所有模块可直接访问 `global.app`、`global.state`、`global.settings`
2. **事件总线 (events)** — 模块间通过 `events.on/fire` 进行松耦合通信
3. **响应式属性 (observe)** — 状态变化自动触发 `xxx:changed` 事件，UI 层自动响应
4. **控制器模式 (CameraController)** — 统一的 `onEnter/update/onExit` 接口，支持热切换
