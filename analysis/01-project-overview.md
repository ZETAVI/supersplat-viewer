# SuperSplat Viewer 项目总览

## 1. 项目简介

SuperSplat Viewer 是 [PlayCanvas](https://playcanvas.com/) 官方开源的 **3D Gaussian Splatting (3DGS) 场景查看器**，用于在浏览器中以交互方式可视化已训练好的 3DGS 场景。项目发布在 npm 上，包名为 `@playcanvas/supersplat-viewer`。

**核心定位：** 将 3DGS 训练产出的 `.ply` / `.sog` / `.meta.json` 等场景文件，通过 WebGL/WebGPU 渲染引擎在网页端进行实时、可交互的三维可视化。

## 2. 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 渲染引擎 | [PlayCanvas Engine](https://github.com/playcanvas/engine) ^2.17.0 | 底层 WebGL/WebGPU 渲染，内置 GSplat 组件 |
| 语言 | TypeScript (ES2022) | 全部源码使用 TypeScript 编写 |
| 构建工具 | Rollup | 多入口构建，产出 ESM 模块 |
| 样式 | SCSS + PostCSS + Autoprefixer | 响应式 UI 样式 |
| 代码规范 | ESLint (PlayCanvas 配置) | 统一代码风格 |
| 包管理 | npm | Node.js 18+ |

## 3. 项目结构

```
supersplat-viewer/
├── src/                         # 源代码目录
│   ├── index.ts                 # 应用主入口
│   ├── index.html               # HTML 模板
│   ├── index.scss               # 全局样式
│   ├── app.ts                   # PlayCanvas 应用初始化
│   ├── viewer.ts                # 核心查看器（渲染、后处理、动画）
│   ├── camera-manager.ts        # 相机模式管理器
│   ├── input-controller.ts      # 输入控制器（键鼠/触屏/手柄）
│   ├── picker.ts                # 3D 拾取（射线投射）
│   ├── annotations.ts           # 场景标注系统
│   ├── ui.ts                    # 完整 UI 层
│   ├── xr.ts                    # VR/AR 支持
│   ├── settings.ts              # 配置版本迁移
│   ├── types.ts                 # 核心类型定义
│   ├── tooltip.ts               # 工具提示
│   ├── voxel-collider.ts        # 体素碰撞系统
│   ├── voxel-debug-overlay.ts   # 体素调试叠加层
│   ├── walk-indicator.ts        # 行走目标指示器
│   ├── cameras/                 # 相机控制器实现
│   │   ├── camera.ts            # 相机基础模型
│   │   ├── orbit-controller.ts  # 轨道相机（环绕观察）
│   │   ├── fly-controller.ts    # 飞行相机（6自由度）
│   │   ├── fps-controller.ts    # 第一人称相机（带重力）
│   │   ├── anim-controller.ts   # 动画轨道相机
│   │   └── walk-source.ts       # 自动行走输入源
│   ├── animation/               # 动画系统
│   │   ├── anim-cursor.ts       # 动画播放游标
│   │   ├── anim-state.ts        # 动画状态（样条插值）
│   │   └── create-rotate-track.ts # 自动旋转动画生成
│   ├── core/                    # 工具库
│   │   ├── math.ts              # 数学工具函数
│   │   ├── observe.ts           # 响应式状态观察
│   │   └── spline.ts            # 三次样条插值
│   ├── module/                  # npm 包导出
│   │   ├── index.ts             # 导出 html/css/js 字符串
│   │   └── index.d.ts           # 类型声明
│   └── schemas/                 # 设置配置模式
│       ├── v1.ts                # V1 版本配置类型
│       └── v2.ts                # V2 版本配置类型（当前）
├── rollup.config.mjs            # Rollup 构建配置
├── tsconfig.json                # TypeScript 配置
├── eslint.config.mjs            # ESLint 配置
├── package.json                 # 项目依赖与脚本
├── serve.json                   # 本地开发服务器配置
└── release.sh                   # 发布脚本
```

## 4. 构建产物

项目通过 Rollup 产出两套构建：

| 构建目标 | 入口 | 产出 | 用途 |
|----------|------|------|------|
| `buildPublic` | `src/index.ts` | `public/index.js` + `public/index.html` | 独立静态网站部署 |
| `buildCss` | `src/index.scss` | `public/index.css` | 样式文件 |
| `buildDist` | `src/module/index.ts` | `dist/index.js` | npm 包，导出 html/css/js 字符串 |

## 5. 支持的场景文件格式

| 格式 | 说明 |
|------|------|
| `.ply` | 标准点云/高斯 splat 文件 |
| `.compressed.ply` | 压缩的高斯 splat 文件（默认） |
| `.sog` | SuperSplat 优化格式 |
| `.meta.json` | 多层级元数据描述文件 |
| `.lod-meta.json` | LOD（层次细节）元数据 |

## 6. 核心特性

- ✅ **实时 3DGS 渲染** — 基于 PlayCanvas 引擎的 GSplat 组件
- ✅ **多种相机模式** — 轨道环绕 / 自由飞行 / 第一人称 / 动画轨道
- ✅ **后处理效果** — 泛光 / 锐化 / 色彩校正 / 暗角 / 色差
- ✅ **场景标注** — 支持可交互的 3D 标注热点
- ✅ **VR/AR 支持** — WebXR 沉浸式体验
- ✅ **体素碰撞** — 基于稀疏体素八叉树的物理碰撞检测
- ✅ **LOD 流式加载** — 渐进式细节加载与流式传输
- ✅ **移动端适配** — 触屏手势、虚拟摇杆
- ✅ **天空盒** — 支持等距柱状投影的环境贴图
- ✅ **npm 包集成** — 可作为模块嵌入到其他 Web 项目

## 7. 开发与调试

```bash
# 安装依赖
npm install

# 启动开发模式（热重载）
npm run develop

# 仅构建
npm run build

# 类型检查
npm run type:check

# 代码检查
npm run lint
```

开发服务器默认运行在 `http://localhost:3000`。
