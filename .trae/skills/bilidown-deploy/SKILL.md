---
name: "bilidown-deploy"
description: "Bilidown 项目打包部署指南。Invoke when user asks how to build, package, or deploy the bilidown project."
---

# Bilidown 打包部署

## 项目概述

Bilidown 是哔哩哔哩视频解析下载工具，前端使用 VanJS + Bootstrap，后端使用 Go + SQLite，通过系统托盘驻留运行。

## 环境准备

- **Node.js**: 前端构建
- **pnpm**: 包管理器
- **Go 1.21+**: 后端编译，需启用 CGO
- **Docker**: 跨平台交叉编译（可选）

## 基础构建流程

### 1. 构建前端

```bash
cd client
pnpm install
pnpm build
```

前端打包后的静态文件输出到 `static/` 目录。

### 2. 构建后端

```bash
cd ../server
go mod tidy
CGO_ENABLED=1 go build
```

> **注意**: Windows 下直接 `go build` 即可；Linux/macOS 可能需要安装 GCC。

## 正式发布打包（GoReleaser）

项目使用 GoReleaser 自动化打包，支持多平台交叉编译。

### 配置文件：`.goreleaser.yaml`

关键配置：
- **Windows**: 使用 `-H windowsgui` 隐藏控制台窗口，打包格式为 `.zip`，内置 `bin/ffmpeg.exe`
- **macOS/Linux**: 打包格式为 `.tar.gz`
- **CGO_ENABLED=1**: 所有平台启用 CGO（SQLite 依赖）

### 交叉编译（推荐：Docker 方式）

#### 1. 拉取镜像和源码

```bash
docker pull iuroc/cgo-cross-build:latest
git clone https://github.com/iuroc/bilidown
```

#### 2. 进入容器

```bash
docker run --rm -it -v .:/usr/src/data iuroc/cgo-cross-build
```

#### 3. 在容器内打包

```bash
cd server

# 前置准备
git tag v2.1.1

# 测试构建（不发布）
goreleaser release --snapshot --clean

# 正式发行（需设置 GITHUB_TOKEN）
# GITHUB_TOKEN=xxx goreleaser release --clean
```

> **前置要求**: 将 `ffmpeg.exe` 放入 `server/bin/` 目录内。

### 编译指定平台

```bash
# 进入 Docker 容器后

# Windows amd64
GOOS=windows GOARCH=amd64 CC=x86_64-w64-mingw32-gcc CGO_ENABLED=1 go build

# Windows 386
GOOS=windows GOARCH=386 CC=i686-w64-mingw32-gcc CGO_ENABLED=1 go build

# macOS amd64
GOOS=darwin GOARCH=amd64 CC=o64-clang CGO_ENABLED=1 go build

# macOS arm64
GOOS=darwin GOARCH=arm64 CC=o64-clang CGO_ENABLED=1 go build

# Linux amd64
GOOS=linux GOARCH=amd64 CGO_ENABLED=1 go build

# Linux arm64
GOOS=linux GOARCH=arm64 CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 go build
```

## 非 Docker 环境编译

Linux amd64 平台可能需要安装依赖：

```bash
sudo apt install pkg-config gcc libayatana-appindicator3-dev
```

## 开发环境

```bash
# 前端
cd client
pnpm install
pnpm dev

# 后端
cd ../server
go build && ./bilidown
```

## 输出产物

| 平台 | 格式 | 包含文件 |
|------|------|----------|
| Windows | `.zip` | 可执行文件 + `static/` + `bin/ffmpeg.exe` |
| macOS | `.tar.gz` | 可执行文件 + `static/` |
| Linux | `.tar.gz` | 可执行文件 + `static/` |

## 部署说明

1. 解压打包文件
2. Windows 用户直接运行可执行文件
3. 非 Windows 用户需提前安装 FFmpeg
4. 程序会驻留系统托盘，自动打开浏览器访问 `http://127.0.0.1:8098`
