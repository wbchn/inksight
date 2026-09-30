# 本地部署与自托管

本文档用于说明如何在本地启动 InkSight 的后端与 WebApp，并说明当前代码中的关键环境变量与验证方式。
如果你只是想了解产品本身，请先看仓库根目录的 `README.md`。
这篇更适合 **开发者、想自托管的用户、以及需要联调前后端与设备流程的人**。

## 1. 适用场景

本页主要面向以下场景：

- 本地开发与调试
- 自己部署一套后端 + WebApp
- 联调刷机、配置、预览和 API

## 2. 当前项目结构

仓库当前主要包含三部分：

- `backend/`：FastAPI 后端，负责配置、渲染、天气、模式管理、统计等
- `webapp/`：Next.js Web 应用，负责官网、在线刷机、登录、设备配置、在线预览
- `firmware/`：ESP32 固件（PlatformIO / Arduino）

## 3. 环境要求

### 后端

- Python **3.10+**
- `pip`

### 前端

- Node.js **20+**（推荐）
- `npm`

### 固件（可选）

- PlatformIO

## 4. Docker Compose 部署

在仓库根目录执行：

```bash
cp backend/.env.example backend/.env
```

编辑 `backend/.env`，按需填写模型 API Key，并设置 `ADMIN_TOKEN`。局域网部署时还需填写（将 IP 换为 Docker 宿主机的局域网地址）：

```env
INKSIGHT_ALLOWED_HOSTS=backend,192.168.1.100
INKSIGHT_CORS_ORIGINS=http://192.168.1.100:3000
INKSIGHT_WEB_BASE_URL=http://192.168.1.100:3000
```

启动并检查：

```bash
docker compose up -d --build
docker compose ps
curl http://192.168.1.100:8080/api/health
```

WebApp 地址为 `http://192.168.1.100:3000`。设备连接 `InkSight-XXXX` 热点后，在 `http://192.168.4.1` 的自定义服务器选项填写 `http://192.168.1.100:8080`，不要附加 `/api`。配网完成后，手动打开 `http://192.168.1.100:3000/claim?code=配对码` 完成设备认领；当前固件的自定义服务器模式会尝试跳转到 `localhost:3000`。

Compose 使用 `backend_data` 保存 SQLite 数据库和登录签名密钥，`backend_uploads` 保存上传文件，`backend_modes` 保存文件型自定义模式，`backend_vocab` 保存词库。重建容器不会清空这些卷；备份时需要包含这些卷。需要额外英文词库时，执行 `docker compose exec backend python scripts/import_kylebing_vocab.py`，然后运行 `docker compose restart backend` 导入数据库。`INKSIGHT_DATA_DIR` 是实际数据库目录，旧版 `DB_PATH` 配置项不生效。查看日志可用 `docker compose logs -f backend web`。

公网部署建议由 Caddy/Nginx 提供统一的 HTTPS 域名，并将所有请求转发到 WebApp `3000`；WebApp 会把其未处理的 `/api/*` 请求转发给后端。这样在线刷机等 WebApp 自有 API 也能正常工作。同时将 `INKSIGHT_ALLOWED_HOSTS` 设置为 `backend,你的域名`、`INKSIGHT_CORS_ORIGINS` 和 `INKSIGHT_WEB_BASE_URL` 都设置为该 HTTPS 域名，配网时填写该域名（不附加 `/api`）。公网环境应限制后端 `8080` 端口只在内网可访问。固件当前信任 Let's Encrypt 的 ISRG Root X1；其他 CA 或自签名证书需要更新固件信任链。

镜像构建默认通过华为云镜像获取 Debian 系统包、PyPI 包和 npm 包。仓库已包含后端字体和基础词库；WebApp 的 `next/font` 仍需从 Google Fonts 获取字体。华为云 SWR 上已核实的 Python/Node 基础镜像目前仅有 `amd64` 版本，因此默认保留支持多架构的官方基础镜像。`amd64` 主机如需基础镜像也走华为云，可运行：

```bash
INKSIGHT_PYTHON_BASE_IMAGE=swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/library/python:3.11-slim \
INKSIGHT_NODE_BASE_IMAGE=swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/library/node:20-bookworm-slim \
docker compose up -d --build
```

## 5. 后端启动

```bash
cd backend

pip install -r requirements.txt
python scripts/setup_fonts.py
python scripts/import_kylebing_vocab.py

cp .env.example .env
# 按需填写环境变量

python -m uvicorn api.index:app --host 0.0.0.0 --port 8080
```

### 后端环境变量

后端示例环境变量在：`backend/.env.example`

当前代码中最重要的变量包括：

- `DEEPSEEK_API_KEY`
- `DASHSCOPE_API_KEY`
- `MOONSHOT_API_KEY`
- `DEBUG_MODE`
- `DEFAULT_CITY`
- `DB_PATH`
- `ADMIN_TOKEN`

说明：

- 如果用户没有在个人信息页配置自己的模型与 API Key，后端会回退到环境变量中的平台级 Key。
- `DEFAULT_CITY` 是系统级天气默认城市，默认为 `杭州`。

## 6. WebApp 启动

```bash
cd webapp

cp .env.example .env
npm install
npm run dev
```

### WebApp 环境变量

前端示例环境变量在：`webapp/.env.example`

当前主要变量：

- `INKSIGHT_BACKEND_API_BASE=http://127.0.0.1:8080`
- `NEXT_PUBLIC_FIRMWARE_API_BASE=`（可选）

建议本地开发时保持：

- 后端：`http://127.0.0.1:8080`
- 前端：`http://127.0.0.1:3000`

## 7. 移动端（Expo）启动

移动端工程位于：`inksight-mobile/`，使用 Expo（`expo-router`）开发。

### 安装依赖

```bash
cd inksight-mobile
npm install
```

### 启动命令（package.json scripts）

- 启动开发模式（Metro）：

```bash
cd inksight-mobile
npm run start
```

- 启动 Web（浏览器）：

```bash
cd inksight-mobile
npm run web
```

- 启动 Android（需要本机 Android 环境/设备）：

```bash
cd inksight-mobile
npm run android
```

- 启动 iOS（需要 macOS/Xcode）：

```bash
cd inksight-mobile
npm run ios
```

### 常用补充（不在 scripts，但可直接运行）

- 指定 dev server 端口（避免与后端端口混淆/冲突）：

```bash
cd inksight-mobile
npx expo start --port 19006
```

- Web 模式下同样指定端口：

```bash
cd inksight-mobile
npx expo start --web --port 19006
```

### 移动端环境变量（重点）

移动端的 `.env` 中常用：

- `EXPO_PUBLIC_INKSIGHT_API_BASE`: **移动端请求后端 API 的基地址**（代码会自动补上 `/api` 后缀）。

说明：

- 该变量只影响移动端向后端发起的 HTTP 请求（例如 `${EXPO_PUBLIC_INKSIGHT_API_BASE}/api/...`）。
- 它**不会决定** Expo/Metro/Web 开发服务监听的端口；Expo dev server 端口由 `expo start` 的 `--port` 控制（默认 8081）。

## 8. 本地入口

启动完成后，通常使用以下入口：

| 入口 | 地址 | 说明 |
|------|------|------|
| WebApp | `http://127.0.0.1:3000` | 官网、本地开发、在线刷机、登录、设备配置、预览 |
| Backend API | `http://127.0.0.1:8080` | FastAPI 接口 |
| 兼容预览接口 | `http://127.0.0.1:8080/api/preview?persona=WEATHER` | 模式级调试入口 |

后端仍保留一些兼容页面（如旧版配置页、仪表盘、编辑器），但当前推荐统一从 WebApp 的**设备配置页**进入配置流程。

## 9. 账号、模型与 API Key

当前代码中：

- **设备配置页** 负责：
  - 模式选择
  - 个性化设置
  - 共享成员
  - 状态查看
- **个人信息页** 负责：
  - 文本模型提供商 / 模型 / API Key
  - 图像模型提供商 / 模型 / API Key
  - 免费额度与访问模式

也就是说，**模型与 API Key 配置不在设备配置页，而在个人信息页**。

## 10. 固件本地编译（可选）

如果你需要本地编译或烧录固件：

```bash
cd firmware
pio run
pio run --target upload
pio device monitor
```

默认环境为：

- `epd_42_wsv2_ssd1683_c3_promini`

更多硬件组合请参考：

- `firmware/platformio.ini`
- `docs/hardware.md`

## 11. 常用检查命令

### 后端

```bash
cd backend
pytest
```

### 前端

```bash
cd webapp
npm run lint
npx tsc --noEmit
```

## 12. 常见问题

### 字体下载 / Next.js 构建问题

当前 WebApp 使用 `next/font` 拉取在线字体。
如果执行 `npm run build` 时网络无法访问 Google Fonts，构建可能失败。

这类问题不会影响日常 `npm run dev` 开发，但在离线或受限网络环境下需要额外处理。

### 端口冲突

- 前端默认 `3000`
- 后端默认 `8080`

如果端口被占用，请修改启动命令中的端口并同步更新 `INKSIGHT_BACKEND_API_BASE`。

### API 调用失败

优先检查：

- 后端 `.env` 是否已填写平台级 API Key
- 是否已在**个人信息页**中配置个人模型与 API Key
- 后端日志中是否有鉴权、额度或上游接口错误
