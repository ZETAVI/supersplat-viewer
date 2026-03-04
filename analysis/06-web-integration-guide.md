# Web 项目嵌入集成指南

## 1. 集成方案概览

将 SuperSplat Viewer 嵌入到自有 Web 项目中，有以下几种方案：

| 方案 | 复杂度 | 灵活度 | 适用场景 |
|------|--------|--------|----------|
| **A. iframe 嵌入** | ⭐ 低 | ⭐⭐ 中 | 快速集成、隔离环境 |
| **B. npm 包 + 模板注入** | ⭐⭐ 中 | ⭐⭐⭐ 高 | 需要定制化的 Web 应用 |
| **C. 源码级集成** | ⭐⭐⭐ 高 | ⭐⭐⭐⭐⭐ 最高 | 深度定制、组件化嵌入 |

---

## 2. 方案 A：iframe 嵌入

### 2.1 原理

将 Viewer 构建为独立的静态网站，通过 `<iframe>` 嵌入到宿主页面中。

### 2.2 步骤

**1) 构建静态网站：**

```bash
cd supersplat-viewer
npm install
npm run build
```

构建产物在 `public/` 目录中：
- `public/index.html`
- `public/index.js`
- `public/index.css`

**2) 部署静态资源：**

将 `public/` 目录部署到你的静态资源服务器（如 Nginx、CDN 等），同时在同目录放置：
- `scene.compressed.ply` — 你的 3DGS 场景文件
- `settings.json` — 场景配置文件

**3) 在宿主页面嵌入 iframe：**

```html
<iframe
    src="https://your-cdn.com/viewer/?content=scene.compressed.ply&settings=settings.json"
    width="100%"
    height="600px"
    style="border: none;"
    allow="xr-spatial-tracking; fullscreen"
></iframe>
```

**4) 动态传参（加载不同场景）：**

```javascript
const iframe = document.getElementById('viewer-iframe');
const sceneUrl = 'https://your-server.com/scenes/my-scene.ply';
const settingsUrl = 'https://your-server.com/scenes/settings.json';

iframe.src = `https://your-cdn.com/viewer/?content=${encodeURIComponent(sceneUrl)}&settings=${encodeURIComponent(settingsUrl)}&noui`;
```

### 2.3 优缺点

✅ 零代码修改，开箱即用  
✅ 完全隔离，不影响宿主页面  
✅ 支持通过 URL 参数控制行为  
❌ 宿主页面无法直接控制 Viewer 内部状态  
❌ 跨域通信需要 postMessage  
❌ UI 定制能力有限  

---

## 3. 方案 B：npm 包 + 模板注入

### 3.1 原理

通过 npm 安装 `@playcanvas/supersplat-viewer`，获取 html/css/js 字符串，在服务端或构建时注入到你的页面模板中。

### 3.2 步骤

**1) 安装 npm 包：**

```bash
npm install @playcanvas/supersplat-viewer
```

**2) 使用导出的资源字符串：**

```typescript
import { html, css, js } from '@playcanvas/supersplat-viewer';

// html — index.html 的完整源码（字符串）
// css  — index.css 的完整源码（字符串）
// js   — index.js 的完整源码（字符串）
```

**3) 在服务端动态生成页面（Node.js 示例）：**

```javascript
import express from 'express';
import { html, css, js } from '@playcanvas/supersplat-viewer';

const app = express();

app.get('/viewer', (req, res) => {
    const sceneUrl = req.query.scene || './scene.ply';

    // 将资源内联到页面中
    const page = html
        .replace('</head>', `<style>${css}</style></head>`)
        .replace('</body>', `<script type="module">${js}</script></body>`);

    res.send(page);
});

app.listen(3000);
```

**4) 在前端框架中使用（React 示例）：**

```jsx
import { useRef, useEffect } from 'react';
import { html, css, js } from '@playcanvas/supersplat-viewer';

function GaussianViewer({ sceneUrl, settingsUrl }) {
    const iframeRef = useRef(null);

    useEffect(() => {
        const iframe = iframeRef.current;
        const doc = iframe.contentDocument;

        // 构建完整页面
        const fullHtml = html
            .replace('</head>', `<style>${css}</style></head>`)
            .replace('</body>', `<script type="module">${js}</script></body>`);

        doc.open();
        doc.write(fullHtml);
        doc.close();
    }, [sceneUrl]);

    return (
        <iframe
            ref={iframeRef}
            style={{ width: '100%', height: '500px', border: 'none' }}
            sandbox="allow-scripts allow-same-origin"
        />
    );
}
```

### 3.3 优缺点

✅ 可以在构建时自定义模板内容  
✅ 可以注入自定义配置  
✅ 不依赖外部 CDN  
❌ 本质上仍然是 iframe 隔离  
❌ 对 Viewer 内部逻辑的控制有限  

---

## 4. 方案 C：源码级集成（推荐）

### 4.1 原理

直接引用 Viewer 的源码模块，将其作为你的 Web 应用的一部分进行构建。这提供了最大的灵活性。

### 4.2 核心入口分析

查看 `src/index.ts` 中的 `main()` 函数签名：

```typescript
const main = async (
    canvas: HTMLCanvasElement,    // HTML Canvas 元素
    settingsJson: any,            // settings.json 解析后的对象
    config: Config                // 启动配置
) => {
    // 返回 Viewer 实例
    return new Viewer(global, gsplatLoad, skyboxLoad, voxelLoad);
};
```

### 4.3 最小集成示例

**步骤 1：复制源码到你的项目中（或作为 Git 子模块）**

```bash
# 作为子模块
git submodule add https://github.com/playcanvas/supersplat-viewer.git lib/supersplat-viewer

# 或直接复制 src 目录
cp -r supersplat-viewer/src your-project/lib/supersplat-viewer/
```

**步骤 2：安装 PlayCanvas 依赖**

```bash
npm install playcanvas@^2.17.0
```

**步骤 3：在你的应用中调用**

```typescript
// your-app/src/components/SceneViewer.ts
import { main } from '../lib/supersplat-viewer/src/index';
import type { Config } from '../lib/supersplat-viewer/src/types';

async function initViewer(canvasElement: HTMLCanvasElement) {
    const settings = {
        version: 2,
        tonemapping: 'neutral',
        highPrecisionRendering: false,
        background: { color: [0.2, 0.2, 0.25] },
        postEffectSettings: {
            sharpness: { enabled: false, amount: 0.5 },
            bloom: { enabled: false, intensity: 0.2, blurLevel: 6 },
            grading: { enabled: false, brightness: 1, contrast: 1, saturation: 1, tint: [1,1,1] },
            vignette: { enabled: false, intensity: 0.5, inner: 0.3, outer: 1.0, curvature: 0.5 },
            fringing: { enabled: false, intensity: 0 }
        },
        cameras: [{ initial: { position: [0, 1, -3], target: [0, 0, 0], fov: 60 } }],
        annotations: [],
        animTracks: [],
        startMode: 'default'
    };

    const config: Config = {
        contentUrl: 'https://your-server.com/scenes/my-scene.ply',
        contents: fetch('https://your-server.com/scenes/my-scene.ply'),
        skyboxUrl: '',
        voxelUrl: '',
        poster: '',
        webgpu: false,
        gpusort: true,
        heatmap: false,
        ministats: false,
        noui: true,      // 隐藏默认 UI，使用你自己的 UI
        noanim: false,
        unified: false,
        aa: true
    };

    const viewer = await main(canvasElement, settings, config);
    return viewer;
}
```

**步骤 4：React 组件封装示例**

```tsx
import React, { useRef, useEffect, useState } from 'react';
import { main } from '../lib/supersplat-viewer/src/index';

interface Props {
    sceneUrl: string;
    width?: string;
    height?: string;
}

export function GaussianSplatViewer({ sceneUrl, width = '100%', height = '600px' }: Props) {
    const canvasRef = useRef<HTMLCanvasElement>(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        if (!canvasRef.current) return;

        const canvas = canvasRef.current;

        const settings = {
            version: 2,
            tonemapping: 'neutral',
            highPrecisionRendering: false,
            background: { color: [0.15, 0.15, 0.2] },
            postEffectSettings: {
                sharpness: { enabled: false, amount: 0 },
                bloom: { enabled: false, intensity: 0, blurLevel: 6 },
                grading: { enabled: false, brightness: 1, contrast: 1, saturation: 1, tint: [1,1,1] },
                vignette: { enabled: false, intensity: 0, inner: 0, outer: 1, curvature: 0.5 },
                fringing: { enabled: false, intensity: 0 }
            },
            cameras: [{ initial: { position: [0, 1, -3], target: [0, 0, 0], fov: 60 } }],
            annotations: [],
            animTracks: [],
            startMode: 'default'
        };

        const config = {
            contentUrl: sceneUrl,
            contents: fetch(sceneUrl),
            skyboxUrl: '',
            voxelUrl: '',
            poster: '',
            webgpu: false,
            gpusort: true,
            heatmap: false,
            ministats: false,
            noui: false,
            noanim: false,
            unified: false,
            aa: true
        };

        main(canvas, settings, config).then(() => {
            setLoading(false);
        });
    }, [sceneUrl]);

    return (
        <div style={{ position: 'relative', width, height }}>
            <canvas
                ref={canvasRef}
                style={{ width: '100%', height: '100%', display: 'block' }}
            />
            {loading && (
                <div style={{
                    position: 'absolute', top: 0, left: 0,
                    width: '100%', height: '100%',
                    display: 'flex', alignItems: 'center', justifyContent: 'center',
                    background: 'rgba(0,0,0,0.5)', color: 'white'
                }}>
                    加载中...
                </div>
            )}
        </div>
    );
}
```

### 4.4 定制化要点

| 定制需求 | 修改位置 | 说明 |
|----------|----------|------|
| 隐藏默认 UI | `config.noui = true` | 不初始化内置 UI |
| 自定义 UI | 监听 `events` 事件 | 通过事件总线获取状态变化 |
| 控制相机 | `global.state.cameraMode` | 切换相机模式 |
| 监听加载进度 | `events.on('progress:changed')` | 获取加载百分比 |
| 切换场景 | 重新调用 `main()` | 需要先销毁旧实例 |
| 禁用 XR | `config.webgpu = true` | WebGPU 模式下 XR 被跳过 |
| 自定义后处理 | 修改 `settings.postEffectSettings` | 调整视觉效果 |

### 4.5 优缺点

✅ 完全控制 Viewer 的所有行为  
✅ 可以精细定制 UI  
✅ 与宿主应用在同一上下文中运行  
✅ 可以访问 PlayCanvas API 进行高级操作  
❌ 需要处理 TypeScript 编译配置  
❌ PlayCanvas 作为依赖会增加打包体积  
❌ 需要关注版本兼容性  

---

## 5. 场景文件准备

### 5.1 从训练输出到 Viewer 可用格式

标准的 3DGS 训练流程（如 [3D Gaussian Splatting](https://github.com/graphdeco-inria/gaussian-splatting)）产出的是 `.ply` 点云文件。SuperSplat Viewer 直接支持此格式。

**推荐工作流：**

```
训练输出(.ply) → SuperSplat编辑器(https://superspl.at) → 导出优化(.compressed.ply + settings.json)
                                                            ↓
                                                      部署到Web服务器
                                                            ↓
                                                      Viewer加载渲染
```

### 5.2 支持的文件格式

| 格式 | 来源 | 说明 |
|------|------|------|
| `.ply` | 直接训练输出 | 未压缩，文件较大 |
| `.compressed.ply` | SuperSplat 导出 | 压缩格式，推荐使用 |
| `.sog` | SuperSplat 优化格式 | 进一步优化的格式 |
| `.meta.json` + 数据文件 | SuperSplat LOD 导出 | 多级细节，适合大场景 |

### 5.3 文件部署注意事项

1. **CORS 配置** — 如果场景文件和 Viewer 不在同一域名下，需要配置服务器的 CORS 头
2. **MIME 类型** — 确保 `.ply` 文件使用正确的 MIME 类型 (`application/octet-stream`)
3. **压缩传输** — 建议启用 gzip/brotli 压缩，`.ply` 文件压缩率通常较高
4. **CDN 缓存** — 场景文件通常不频繁变化，建议设置较长的缓存时间

---

## 6. 常见集成场景

### 6.1 场景列表页 + 查看器

```
用户浏览场景列表 → 点击某个场景 → 打开/跳转到查看器页面
                                         │
                                   传递场景URL和配置
                                         │
                                   Viewer 加载渲染
```

### 6.2 嵌入式预览组件

```
产品详情页 / 资产管理后台
    ├── 其他内容
    └── [3DGS Viewer 组件]  ← 嵌入在页面某个区域
            │
      配置: noui=true（自定义 UI）
            │
      使用 iframe 或源码集成
```

### 6.3 全屏沉浸式体验

```
专门的展示页面
    └── 全屏 Canvas
            │
      配置: 自定义动画轨道 + 标注 + 后处理效果
            │
      用户可以自由探索场景
```

---

## 7. 调试与排错

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 场景不显示 | 文件URL错误或CORS | 检查浏览器控制台网络请求 |
| 渲染闪烁 | 画布大小不匹配 | 确保 canvas 尺寸正确设置 |
| 性能差 | 场景过大 | 使用压缩格式，启用 LOD |
| 触屏不响应 | 缺少 touch 事件支持 | 确保 canvas 可接收触摸事件 |
| WebGPU 不可用 | 浏览器不支持 | 保持 `webgpu: false` 使用 WebGL |
