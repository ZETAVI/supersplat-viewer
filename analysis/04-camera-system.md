# 相机系统与交互控制

## 1. 相机模式总览

项目支持 4 种相机模式，通过 `CameraManager` 统一管理切换：

| 模式 | 类型标识 | 适用场景 | 键盘快捷键 |
|------|----------|----------|------------|
| **轨道模式** (Orbit) | `'orbit'` | 围绕焦点旋转观察物体 | `1` |
| **飞行模式** (Fly) | `'fly'` | 6自由度自由漫游 | `2` |
| **第一人称模式** (FPS) | `'fps'` | 地面行走（带重力/碰撞） | `3` |
| **动画模式** (Anim) | `'anim'` | 预录制的相机动画轨道播放 | — |

## 2. 相机架构

### 2.1 Camera 基础类 (`cameras/camera.ts`)

```typescript
class Camera {
    position: Vec3;      // 世界空间位置
    angles: Vec3;        // 欧拉角 (pitch, yaw, roll)
    distance: number;    // 到焦点的距离
    fov: number;         // 视角

    look(target: Vec3): void          // 朝向目标
    calcFocusPoint(): Vec3            // 计算焦点世界坐标
    lerp(a: Camera, b: Camera, t: number): void  // 两个位姿间插值
}
```

### 2.2 CameraController 接口

所有相机控制器必须实现此接口：

```typescript
interface CameraController {
    onEnter(camera: Camera): void;     // 切入该模式时调用
    update(dt: number, frame: InputFrame): void;  // 每帧更新
    onExit(): void;                    // 切出该模式时调用
}
```

### 2.3 CameraManager (`camera-manager.ts`)

核心编排器，负责：

- 维护当前激活的相机控制器
- 管理模式切换的平滑过渡（位姿插值）
- 处理全局输入事件（frame、reset、play-pause、toggle-fps）
- 协调 WalkSource 自动行走
- 响应标注(annotation)点击切换相机

```
CameraManager
├── camera: Camera (当前位姿状态)
├── controllers: Map<CameraMode, CameraController>
│   ├── 'orbit'  → OrbitController
│   ├── 'fly'    → FlyController
│   ├── 'fps'    → FpsController
│   └── 'anim'   → AnimController
├── walkSource: WalkSource (自动行走)
└── transition: 位姿过渡状态
```

## 3. 各相机控制器详解

### 3.1 轨道控制器 (OrbitController)

基于 PlayCanvas 内置的 `OrbitControllerPC`，实现围绕焦点的球面运动：

**交互方式：**
- 🖱️ 鼠标左键拖动 → 旋转（水平 + 垂直）
- 🖱️ 鼠标右键拖动 / 中键拖动 → 平移焦点
- 🖱️ 滚轮 → 缩放（改变距离）
- 📱 单指滑动 → 旋转
- 📱 双指捏合 → 缩放
- 📱 双指平移 → 移动焦点

**约束：**
- 俯仰角 (Pitch) 限制在 ±90°
- 带平滑阻尼

### 3.2 飞行控制器 (FlyController)

6自由度自由飞行，适合大范围场景漫游：

**交互方式：**
- `W/A/S/D` 或 `↑/↓/←/→` → 前后左右移动
- 鼠标移动 → 视角旋转（需要指针锁定）
- `Q/E` → 上升/下降
- `Shift` → 加速 (2x)
- `Ctrl` → 减速 (0.25x)

**碰撞检测：**
- 使用球体碰撞（半径 0.2m）
- 通过 VoxelCollider 进行碰撞查询
- 碰撞时沿法线推出（滑动效果）

### 3.3 第一人称控制器 (FpsController)

带物理的地面行走模式：

**物理参数：**
| 参数 | 值 | 说明 |
|------|-----|------|
| 胶囊体高度 | 1.8m | 角色碰撞高度 |
| 眼睛高度 | 1.6m | 相机离地高度 |
| 碰撞半径 | 0.3m | 角色宽度 |
| 重力加速度 | 9.8 m/s² | 垂直方向 |

**特性：**
- ✅ 重力系统 — 自由落体直到接触地面
- ✅ 跳跃 — 空格键（仅地面时可用）
- ✅ 胶囊体碰撞 — 墙壁阻挡、天花板碰头
- ✅ 地面检测 — 碰撞解析时判断接地状态
- ✅ 水平移动约束 — 只在 XZ 平面移动，Y 由重力控制

### 3.4 动画控制器 (AnimController)

播放预录制的相机动画轨道：

```typescript
class AnimController {
    animState: AnimState;   // 动画状态（样条插值）

    update(dt) {
        animState.cursor.step(dt);  // 推进时间线
        const { position, target, fov } = animState.evaluate();
        // 更新相机位姿
    }
}
```

**动画数据结构：**
```typescript
type AnimTrack = {
    name: string,
    duration: number,        // 总时长(秒)
    frameRate: number,       // 帧率
    loopMode: 'none' | 'repeat' | 'pingpong',
    interpolation: 'step' | 'spline',
    smoothness: number,      // 平滑度
    keyframes: {
        times: number[],     // 关键帧时间
        values: {
            position: number[],  // [x,y,z, x,y,z, ...]
            target: number[],    // [x,y,z, x,y,z, ...]
            fov: number[]        // [fov1, fov2, ...]
        }
    }
};
```

## 4. 输入系统 (`input-controller.ts`)

### 4.1 输入源

```
InputController
├── KeyboardMouseSource   // 桌面键鼠输入
│   ├── 键盘 → WASD/方向键/空格/Shift/Ctrl
│   ├── 鼠标移动 → 视角旋转
│   └── 鼠标按键 → 拾取/行走
├── MultiTouchSource      // 移动端触屏
│   ├── 单指 → 旋转/点击行走
│   └── 双指 → 缩放/平移
└── GamepadSource         // 游戏手柄
    ├── 左摇杆 → 移动
    └── 右摇杆 → 视角
```

### 4.2 输入帧 (InputFrame)

每帧收集的统一输入数据：

```typescript
type InputFrame = {
    deltas: {
        move: [x, y, z],    // 移动增量
        rotate: [x, y, z]   // 旋转增量
    }[]
};
```

### 4.3 快捷键

| 快捷键 | 功能 |
|--------|------|
| `1` | 切换到轨道模式 |
| `2` | 切换到飞行模式 |
| `3` | 切换到第一人称模式 |
| `F` | 聚焦/自动构图 |
| `R` | 重置相机位置 |
| `Space` | 播放/暂停动画 |

### 4.4 点击行走 (Walk-to)

**桌面端：** 鼠标点击地面 → Picker 获取 3D 位置 → WalkSource 自动行走
**移动端：** 触屏点击地面 → 同上逻辑

WalkSource 会持续生成移动输入，直到：
- 到达目标（XZ 距离 < 0.5m）
- 被阻挡（速度过低持续 0.2s）
- 用户手动操作（取消行走）

## 5. 相机过渡动画

模式切换时，CameraManager 执行位姿过渡：

```
当前位姿(A) ──────[lerp(t)]──────→ 目标位姿(B)
                    │
                    ├── position: Vec3.lerp
                    ├── angles: Vec3.lerp
                    ├── distance: lerp
                    └── fov: lerp
```

过渡时间默认由 `easeOut` 缓动函数控制，提供自然流畅的视觉体验。

## 6. XR 支持 (`xr.ts`)

```typescript
function initXr(global: Global) {
    // 检测 AR/VR 可用性
    // 管理 XR 会话生命周期
    // 保存/恢复相机状态
    // 设置 XR 控制器导航
}
```

支持 WebXR 标准的 VR 头显和 AR 设备，在 XR 模式下自动禁用常规画布调整。
