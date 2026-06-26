# Shannon 桌面应用

使用  [Next.js](https://nextjs.org/) 构建的 Shannon AI agents 桌面应用。

## 快速开始

### 前置条件

桌面应用会连接到 Shannon 后端服务。请先启动后端：

```bash
# From repository root
make dev

# Verify services are running
curl http://localhost:8080/health
```

### 选项 1：本地 Web UI（开发）

以本地 Web 应用的方式运行 UI，无需构建原生二进制文件：

```bash
cd desktop

# Install dependencies
npm install

# Start development server
npm run dev

# Open http://localhost:3000
```

**Web 模式下的特性：**

* 实时 SSE 事件流
* Session 和任务管理
* 可视化 workflow 执行
* 支持深色模式
* 开发时即时热重载

### 选项 2：原生桌面应用

#### 下载预构建二进制文件

从 [GitHub Releases](https://github.com/Kocoro-lab/Shannon/releases/latest) 下载适用于你平台的最新 release：

| 平台                               | 格式                                     |
| -------------------------------- | -------------------------------------- |
| **Windows**                      | `.msi` 安装包、`.exe` NSIS 安装包             |
| **Linux**                        | `.AppImage`（便携版）、`.deb`（Debian/Ubuntu） |

#### 从源码构建

为你的平台构建原生桌面应用：

```bash
cd desktop

# Install dependencies
npm install

# Build for your platform
npm run tauri:build

# Output locations:
# macOS:   src-tauri/target/universal-apple-darwin/release/bundle/dmg/
# Windows: src-tauri/target/release/bundle/msi/
# Linux:   src-tauri/target/release/bundle/appimage/
```

## Web UI 与原生应用对比

| 特性     | Web UI            | 原生应用           |
| ------ | ----------------- | -------------- |
| 快速测试   | 即时（`npm run dev`） | 需要构建           |
| 系统集成   | 有限                | 系统托盘、通知        |
| 离线历史   | 无                 | Dexie.js 本地数据库 |
| 性能     | 浏览器开销             | 原生渲染           |
| 文件系统访问 | 有限                | 完整 Tauri APIs  |
| 自动更新   | 无                 | 内置 updater     |
| 内存使用   | 较高（浏览器）           | 已优化            |

## 开发

### 项目结构

```
desktop/
├── app/              # Next.js app router pages
├── components/       # React components
│   ├── ui/          # shadcn/ui components
│   └── ...          # Custom components
├── lib/             # Utilities and helpers
├── hooks/           # React hooks
├── src-tauri/       # Tauri Rust backend
│   ├── src/        # Rust source code
│   ├── icons/      # App icons
│   └── Cargo.toml  # Rust dependencies
├── public/          # Static assets
└── package.json    # Node dependencies
```

### 可用脚本

```bash
# Development
npm run dev          # Next.js dev server (web mode)
npm run tauri:dev    # Tauri dev mode (native app with hot reload)

# Production
npm run build        # Build Next.js static export
npm run tauri:build  # Build native app for your platform

# Linting
npm run lint         # Run ESLint
```

### 环境配置

为开发创建 `.env.local`：

```bash
# Backend API endpoint
NEXT_PUBLIC_API_URL=http://localhost:8080

# Optional: Enable debug mode
NEXT_PUBLIC_DEBUG=true
```

查看 [`.env.local.example`](.env.local.example) 获取所有可用选项。

## 技术栈

| 组件                 | 技术                                                                                        |
| ------------------ | ----------------------------------------------------------------------------------------- |
| Frontend Framework | 带 App Router 的 [Next.js 16](https://nextjs.org/)                                          |
| UI Components      | [shadcn/ui](https://ui.shadcn.com/) + [Radix UI](https://www.radix-ui.com/)               |
| Styling            | [Tailwind CSS](https://tailwindcss.com/)                                                  |
| Desktop Runtime    | [Tauri v2](https://tauri.app/)                                                            |
| State Management   | [Zustand](https://zustand-demo.pmnd.rs/) + [Redux Toolkit](https://redux-toolkit.js.org/) |
| Local Database     | [Dexie.js](https://dexie.org/)（IndexedDB wrapper）                                         |
| Flow Diagrams      | [@xyflow/react](https://reactflow.dev/)                                                   |
| Markdown Rendering | [react-markdown](https://github.com/remarkjs/react-markdown)                              |

## 生产构建

### 前置条件

* **Node.js** 20+
* **Rust**（最新稳定版）— 从 [rustup.rs](https://rustup.rs/) 安装
* **平台特定依赖**：

  * **Windows**：Microsoft C++ Build Tools
  * **Linux**：查看 [Tauri Prerequisites](https://tauri.app/v2/guides/prerequisites/)

### 构建命令

```bash

# Windows
npm run tauri:build -- --target x86_64-pc-windows-msvc

# Linux
npm run tauri:build -- --target x86_64-unknown-linux-gnu

# iOS (macOS only, requires Xcode)
npm run tauri ios build
```

## 自动更新

桌面应用包含自动更新检查：

* **启动时检查**：从 GitHub 查找新的 releases
* **后台下载**：静默下载 updates
* **用户提示**：安装 updates 前询问用户

在 `src-tauri/tauri.conf.json` 中配置 update 行为。

## 故障排查

### Web UI 无法启动

```bash
# Clear Next.js cache
rm -rf .next node_modules/.cache
npm install
npm run dev
```

### 后端连接失败

```bash
# Verify backend is running
curl http://localhost:8080/health

# Check API URL in .env.local
cat .env.local | grep NEXT_PUBLIC_API_URL
```



