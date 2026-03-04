# 配置系统与自定义

## 1. 配置层次

项目有两层配置机制：

| 层级 | 类型 | 来源 | 用途 |
|------|------|------|------|
| **Config** | 不可变启动配置 | URL 参数 / HTML 页面 | 控制加载哪些资源、启用哪些特性 |
| **ExperienceSettings** | 场景体验配置 | `settings.json` 文件 | 控制相机、背景、动画、后处理等 |

## 2. Config（启动配置）

在 `index.html` 中通过 URL 参数或直接在页面脚本中设置 `window.sse` 对象：

```typescript
type Config = {
    contentUrl: string;      // 场景文件 URL（默认 ./scene.compressed.ply）
    contents: any;           // 预加载的场景内容
    skyboxUrl: string;       // 天空盒图像 URL
    voxelUrl: string;        // 体素碰撞数据 URL
    poster: string;          // 加载海报图像 URL

    // 特性开关
    webgpu: boolean;         // 是否使用 WebGPU（否则 WebGL）
    gpusort: boolean;        // GPU 端高斯排序
    heatmap: boolean;        // 热力图模式
    ministats: boolean;      // 显示性能统计
    noui: boolean;           // 隐藏 UI
    noanim: boolean;         // 禁止自动播放动画
    unified: boolean;        // 强制统一渲染模式
    aa: boolean;             // 抗锯齿
};
```

### URL 参数示例

```
https://your-site.com/viewer/?content=https://example.com/scene.ply
    &settings=https://example.com/settings.json
    &skybox=https://example.com/env.hdr
    &noui
    &aa
```

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `settings` | settings.json 文件 URL | `./settings.json` |
| `content` | 场景文件 URL | `./scene.compressed.ply` |
| `skybox` | 天空盒图像 URL | 无 |
| `poster` | 加载时显示的海报 URL | 无 |
| `noui` | 隐藏全部 UI | 否 |
| `noanim` | 暂停启动动画 | 否 |
| `ministats` | 显示性能面板 | 否 |
| `unified` | 强制统一渲染 | 否 |
| `aa` | 启用抗锯齿 | 否 |

## 3. ExperienceSettings（体验配置）

### 3.1 V2 配置结构（当前版本）

```typescript
type ExperienceSettings = {
    version: 2,

    // 色调映射
    tonemapping: 'none' | 'linear' | 'filmic' | 'hejl' | 'aces' | 'aces2' | 'neutral',

    // 高精度渲染
    highPrecisionRendering: boolean,

    // 背景音乐
    soundUrl?: string,

    // 背景设置
    background: {
        color: [number, number, number],  // RGB (0-1)
        skyboxUrl?: string
    },

    // 后处理效果
    postEffectSettings: {
        sharpness: { enabled: boolean, amount: number },
        bloom: { enabled: boolean, intensity: number, blurLevel: number },
        grading: {
            enabled: boolean,
            brightness: number,
            contrast: number,
            saturation: number,
            tint: [number, number, number]
        },
        vignette: {
            enabled: boolean,
            intensity: number,
            inner: number,
            outer: number,
            curvature: number
        },
        fringing: { enabled: boolean, intensity: number }
    },

    // 动画轨道
    animTracks: AnimTrack[],

    // 相机配置（可多个）
    cameras: Camera[],

    // 3D 标注
    annotations: Annotation[],

    // 启动模式
    startMode: 'default' | 'animTrack' | 'annotation',

    // 是否有初始位姿
    hasStartPose?: boolean
};
```

### 3.2 AnimTrack（动画轨道）

```typescript
type AnimTrack = {
    name: string,                    // 动画名称
    duration: number,                // 持续时间(秒)
    frameRate: number,               // 帧率
    loopMode: 'none' | 'repeat' | 'pingpong',  // 循环模式
    interpolation: 'step' | 'spline',           // 插值方式
    smoothness: number,              // 样条平滑度
    keyframes: {
        times: number[],             // 关键帧时间点
        values: {
            position: number[],      // 位置数组 [x1,y1,z1, x2,y2,z2, ...]
            target: number[],        // 目标点数组
            fov: number[]            // 视角数组
        }
    }
};
```

### 3.3 Camera（相机位姿）

```typescript
type CameraPose = {
    position: [number, number, number],
    target: [number, number, number],
    fov: number
};

type Camera = {
    initial: CameraPose   // 初始位姿
};
```

### 3.4 Annotation（3D 标注）

```typescript
type Annotation = {
    position: [number, number, number],  // 标注在 3D 空间中的位置
    title: string,                       // 标题
    text: string,                        // 描述文本
    extras: any,                         // 自定义扩展数据
    camera: Camera                       // 查看该标注时的相机位姿
};
```

## 4. 配置版本迁移 (`settings.ts`)

项目支持 V1 → V2 的自动配置迁移：

```typescript
function importSettings(settingsJson: any): ExperienceSettings {
    // 如果是 V2，直接使用（带必要迁移）
    // 如果是 V1，转换为 V2 格式
    // 如果无版本号，假设为 V1
}
```

### V1 → V2 迁移映射

| V1 字段 | V2 字段 |
|---------|---------|
| `camera.position` | `cameras[0].initial.position` |
| `camera.target` | `cameras[0].initial.target` |
| `camera.fov` | `cameras[0].initial.fov` |
| `camera.startAnim: 'orbit'` | `startMode: 'animTrack'` + 自动生成旋转轨道 |
| `camera.startAnim: 'animTrack'` | `startMode: 'animTrack'` |
| `background.color` | `background.color` (RGB, 忽略 alpha) |

## 5. settings.json 示例

### 最小配置

```json
{
    "version": 2,
    "tonemapping": "neutral",
    "highPrecisionRendering": false,
    "background": {
        "color": [0.4, 0.4, 0.4]
    },
    "postEffectSettings": {
        "sharpness": { "enabled": false, "amount": 0.5 },
        "bloom": { "enabled": false, "intensity": 0.2, "blurLevel": 6 },
        "grading": {
            "enabled": false,
            "brightness": 1,
            "contrast": 1,
            "saturation": 1,
            "tint": [1, 1, 1]
        },
        "vignette": {
            "enabled": false,
            "intensity": 0.5,
            "inner": 0.3,
            "outer": 1.0,
            "curvature": 0.5
        },
        "fringing": { "enabled": false, "intensity": 0 }
    },
    "cameras": [
        {
            "initial": {
                "position": [0, 1, -3],
                "target": [0, 0, 0],
                "fov": 60
            }
        }
    ],
    "annotations": [],
    "animTracks": [],
    "startMode": "default"
}
```

### 带动画和标注的完整配置

```json
{
    "version": 2,
    "tonemapping": "aces2",
    "highPrecisionRendering": true,
    "soundUrl": "https://example.com/ambient.mp3",
    "background": {
        "color": [0.1, 0.1, 0.15],
        "skyboxUrl": "https://example.com/env.hdr"
    },
    "postEffectSettings": {
        "sharpness": { "enabled": true, "amount": 0.3 },
        "bloom": { "enabled": true, "intensity": 0.15, "blurLevel": 6 },
        "grading": {
            "enabled": true,
            "brightness": 1.1,
            "contrast": 1.05,
            "saturation": 1.2,
            "tint": [1, 0.98, 0.95]
        },
        "vignette": {
            "enabled": true,
            "intensity": 0.3,
            "inner": 0.4,
            "outer": 1.2,
            "curvature": 0.5
        },
        "fringing": { "enabled": false, "intensity": 0 }
    },
    "cameras": [
        {
            "initial": {
                "position": [2, 1.5, -4],
                "target": [0, 0.5, 0],
                "fov": 55
            }
        }
    ],
    "annotations": [
        {
            "position": [1, 1, 0],
            "title": "入口",
            "text": "这是场景的主入口",
            "extras": {},
            "camera": {
                "initial": {
                    "position": [2, 1, 1],
                    "target": [1, 1, 0],
                    "fov": 60
                }
            }
        }
    ],
    "animTracks": [
        {
            "name": "环绕展示",
            "duration": 10,
            "frameRate": 30,
            "loopMode": "repeat",
            "interpolation": "spline",
            "smoothness": 0.5,
            "keyframes": {
                "times": [0, 2.5, 5, 7.5, 10],
                "values": {
                    "position": [3,1,0, 0,1,3, -3,1,0, 0,1,-3, 3,1,0],
                    "target": [0,0,0, 0,0,0, 0,0,0, 0,0,0, 0,0,0],
                    "fov": [60, 60, 60, 60, 60]
                }
            }
        }
    ],
    "startMode": "animTrack"
}
```

## 6. 运行时状态 (State)

可观察的运行时状态字段：

```typescript
type State = {
    loaded: boolean;            // 场景是否加载完成
    readyToRender: boolean;     // 是否准备好渲染
    retinaDisplay: boolean;     // 是否使用 Retina 分辨率
    progress: number;           // 加载进度 (0-100)
    inputMode: 'desktop' | 'touch';  // 输入模式
    cameraMode: CameraMode;     // 当前相机模式
    hasAnimation: boolean;      // 是否有动画轨道
    animationDuration: number;  // 动画总时长
    animationTime: number;      // 当前动画时间
    animationPaused: boolean;   // 动画是否暂停
    hasAR: boolean;             // 是否支持 AR
    hasVR: boolean;             // 是否支持 VR
    hasCollision: boolean;      // 是否有碰撞数据
    isFullscreen: boolean;      // 是否全屏
    controlsHidden: boolean;    // UI 控件是否隐藏
    gamingControls: boolean;    // 是否启用游戏控制模式
};
```
