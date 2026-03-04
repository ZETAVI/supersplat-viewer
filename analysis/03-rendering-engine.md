# 3DGS 渲染管线与 PlayCanvas 集成

## 1. 3D Gaussian Splatting 渲染原理

3D Gaussian Splatting (3DGS) 是一种基于显式表示的三维重建方法。与 NeRF 的隐式表示不同，3DGS 使用大量三维高斯椭球体来表达场景：

- 每个高斯体有：**位置** (xyz)、**协方差矩阵**（控制形状/旋转）、**球谐系数**（视角相关的颜色）、**不透明度**
- 渲染时将 3D 高斯体投影为 2D splat，按深度排序后逐像素混合

## 2. PlayCanvas GSplat 组件

SuperSplat Viewer 直接使用 PlayCanvas 引擎内置的 `GSplatComponentSystem` 进行渲染，不自行实现 splat 渲染。

### 2.1 资源加载

```typescript
// src/index.ts - loadGsplat()
const asset = new Asset(filename, 'gsplat', {
    url: contentUrl,
    filename,
    contents: c       // 可选的预加载内容
}, data);             // 可选的元数据 (meta.json)

app.assets.add(asset);
app.assets.load(asset);
```

PlayCanvas 的 `GSplatHandler` 负责：
- 解析 `.ply` / `.compressed.ply` / `.sog` 文件格式
- 构建 GPU 数据结构（位置、协方差、颜色纹理等）
- 实现基于 GPU 的高斯排序（配置项 `config.gpusort`）

### 2.2 实体创建

```typescript
const entity = new Entity('gsplat');
entity.setLocalEulerAngles(0, 0, 180);  // 翻转Y/Z轴以适配坐标系
entity.addComponent('gsplat', {
    unified: unified || filename.endsWith('lod-meta.json'),
    asset
});
```

**关键参数：**
- `unified` — 统一渲染模式，所有 splat 共享一个材质。LOD 场景强制启用
- `alphaClip: 1/255` — 透明度裁剪阈值，过滤几乎完全透明的 splat
- `GSPLAT_AA` — 反锯齿宏定义开关

### 2.3 LOD 流式加载

对于大型场景，项目支持渐进式 LOD 加载：

```typescript
// src/viewer.ts
const gsplatComponent = gsplatEntity.gsplat;
if (gsplatComponent.lodManager) {
    gsplatComponent.lodManager.streaming = true;  // 启用流式传输
}
```

LOD 管理器会根据相机距离和视角自动请求加载不同精度层次的 splat 数据。

## 3. 渲染管线

### 3.1 渲染模式

```typescript
// viewer.ts 中的渲染控制
app.renderNextFrame = true;  // 懒渲染 — 仅在需要时渲染
```

项目采用**按需渲染**策略，而非持续渲染：
- 相机移动/旋转时触发渲染
- 动画播放时连续渲染
- 窗口大小变化时触发渲染
- LOD 数据加载完成时触发渲染

### 3.2 CameraFrame 后处理管线

`CameraFrame` 是 PlayCanvas 提供的后处理框架，在 `viewer.ts` 中初始化：

```typescript
const cameraFrame = new CameraFrame(app, camera.camera);
cameraFrame.rendering.toneMapping = ToneMapping.NEUTRAL;
cameraFrame.rendering.renderTargetScale = 1;
```

### 3.3 后处理效果

项目支持丰富的后处理效果，均在 `applyPostEffectSettings()` 中配置：

| 效果 | 配置项 | 说明 |
|------|--------|------|
| **锐化** (Sharpness) | `sharpness.enabled`, `sharpness.amount` | 增强场景细节 |
| **泛光** (Bloom) | `bloom.enabled`, `bloom.intensity`, `bloom.blurLevel` | 发光效果 |
| **色彩校正** (Grading) | `grading.brightness/contrast/saturation/tint` | 调色 |
| **暗角** (Vignette) | `vignette.enabled/intensity/inner/outer/curvature` | 镜头暗角 |
| **色差** (Fringing) | `fringing.enabled`, `fringing.intensity` | 色散效果 |

```typescript
// 后处理效果应用示例
const applyPostEffectSettings = (cameraFrame, settings) => {
    const pe = settings.postEffectSettings;
    cameraFrame.bloom.enabled = pe.bloom.enabled;
    cameraFrame.bloom.intensity = pe.bloom.intensity;
    cameraFrame.grading.brightness = pe.grading.brightness;
    cameraFrame.grading.contrast = pe.grading.contrast;
    cameraFrame.grading.saturation = pe.grading.saturation;
    // ... 更多效果配置
};
```

### 3.4 色调映射 (Tone Mapping)

支持多种色调映射算法：

| 模式 | 说明 |
|------|------|
| `none` | 无色调映射（线性输出） |
| `linear` | 简单线性映射 |
| `filmic` | 电影风格 |
| `hejl` | Hejl-Burgess-Dawson |
| `aces` | ACES 标准 |
| `aces2` | ACES 2.0 |
| `neutral` | 中性映射（默认） |

## 4. 天空盒与环境

```typescript
// src/index.ts
const loadSkybox = (app, url) => {
    const asset = new Asset('skybox', 'texture', { url }, {
        type: 'rgbp',          // RGBP 编码
        mipmaps: false,
        addressu: 'repeat',    // 水平循环
        addressv: 'clamp'      // 垂直钳制
    });
    // ...
    app.scene.envAtlas = asset.resource;  // 设为环境贴图
};
```

支持等距柱状投影 (Equirectangular) 的 HDR/LDR 天空盒图像。

## 5. 拾取系统 (Picker)

```typescript
// src/picker.ts
class Picker {
    pick(x: number, y: number): Promise<Vec3 | null>
}
```

拾取系统使用自定义的 Render Pass：
1. 渲染一个特殊的深度 pass，将深度值编码到颜色通道
2. 读回点击位置的像素值
3. 将编码的深度值转换为世界坐标位置

这种方案避免了在 CPU 端进行光线与数百万个高斯体的求交计算。

## 6. 体素碰撞系统

```typescript
// src/voxel-collider.ts
class VoxelCollider {
    static load(url: string): Promise<VoxelCollider>
    querySphere(center, radius): Vec3     // 球体碰撞查询
    queryCapsule(a, b, radius): Vec3      // 胶囊体碰撞查询
}
```

体素碰撞数据是预计算的稀疏体素八叉树，存储为 `.voxel.json` + `.voxel.bin` 格式。用于：
- **飞行模式** — 防止穿墙（球体碰撞，半径 0.2m）
- **第一人称模式** — 地面检测 + 墙壁碰撞（胶囊体，高 1.8m，半径 0.3m）

## 7. 渲染性能优化

| 优化手段 | 实现位置 | 说明 |
|----------|----------|------|
| 按需渲染 | `viewer.ts` | 仅在相机/场景变化时渲染 |
| GPU 排序 | PlayCanvas Engine | 在 GPU 上进行高斯体深度排序 |
| LOD 流式加载 | `viewer.ts` | 远处使用低精度 splat |
| DPI 自适应 | `index.ts` | 移动端限制最大像素 1080p，桌面端 2160p |
| 半分辨率模式 | `state.retinaDisplay` | 可选降低渲染分辨率至 50% |
| Alpha 裁剪 | `alphaClip: 1/255` | 跳过几乎透明的 splat |
