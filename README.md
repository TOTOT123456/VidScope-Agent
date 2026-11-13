# VidScope-Agent

面向长视频内容理解的 Video Agent 平台。支持视频上传、语音转写、关键帧 OCR、证据检索、多步分析与结构化产物生成。

> 基于 [Xiaoc7r/DOVideo-AI](https://github.com/Xiaoc7r/DOVideo-AI) 整理，保留原作者版权和 MIT 许可证。当前版本主要调整项目命名、展示文案与仓库结构；不将上游功能和指标作为新增开发成果。

## 核心能力

- **可靠任务链路**：分片上传、断点续传、RocketMQ 异步执行、状态查询与失败重试。
- **时序多模态上下文**：并行处理 ASR 与关键帧 OCR，按时间轴组织 VideoContext。
- **Agent 分析工作流**：Planner 拆解目标，Executor 生成结果，Critic 检查覆盖和证据，支持有限轮次修正。
- **长视频证据检索**：摘要、关键词与 Embedding 检索，按需加载时间戳证据。
- **阶段恢复**：通过 Checkpoint 保存产物，支持失败后的恢复与重执行。
- **调用成本控制**：限流、有界并发、超时与有限重试。

## 技术栈

Java 21、Spring Boot、MyBatis-Plus、MySQL、Redis、Redisson、RocketMQ、MinIO、Qdrant、LangChain4j、Vue 3、FFmpeg、Tesseract。

## 系统流程

```text
视频上传 / 链接接入
        ↓
任务受理 → RocketMQ → 分析消费者
        ↓
ASR 与 OCR 并行 → VideoContext → 分段与证据检索
        ↓
Planner → Executor → Critic → 结构化产物
        ↓
Checkpoint / 结果保存 → SSE 进度与后续追问
```

## 目录

```text
client/            Vue 工作台
server/            Spring Boot API 与 Agent 服务
extension/         浏览器侧边栏入口
rocketmq/          Broker 配置
scripts/           本地启动辅助脚本
docs/              技术说明与上游检查记录
```

## 本地运行


### 环境要求

| 组件 | 要求 | 说明 |
| :--- | :--- | :--- |
| JDK | 21 | 后端运行环境 |
| Node.js | 22 | Vue 与 Vite 构建环境 |
| Docker | Compose v2 | 启动 MySQL、Redis、MinIO、Qdrant 与 RocketMQ |
| FFmpeg | 可在终端调用 | 音频切分与关键帧抽取 |
| Tesseract | 安装 `chi_sim` 与 `eng` | 中英文关键帧 OCR |
| yt-dlp | 可选 | 仅解析在线视频链接时需要 |

建议先确认命令均可用：

```bash
java -version
node --version
docker compose version
ffmpeg -version
tesseract --version
```

### 1. 准备配置

```bash
cp .env.example .env
```

编辑 `.env`，至少替换数据库、Redis、MinIO、Qdrant 的示例密码并设置 `SILICONFLOW_API_KEY`。全新数据库中 `DB_USERNAME` 与 `MYSQL_APP_USER` 应保持一致；`MYSQL_ROOT_PASSWORD` 仅供数据库初始化使用。密钥只保存在本地 `.env`，不要提交到仓库。

默认 LLM 为 `deepseek-ai/DeepSeek-V3.2`。历史示例模型 `deepseek-ai/DeepSeek-R1-Distill-Qwen-32B` 已被硅基流动禁用，会返回 `Model disabled`。`LLM_TIMEOUT_SECONDS` 默认是 `300`，用于避免长视频证据分析在模型响应尚未返回时过早超时；模型或超时配置变更后需要重启后端。

### 2. 启动中间件

```bash
./scripts/dev-up.sh
```

脚本会检查本机命令与版本、校验 Compose 配置，并等待 MySQL、Redis、MinIO、Qdrant 和 RocketMQ 启动。中间件与后端默认只监听 `127.0.0.1`，不会直接暴露到局域网；远程部署时再显式修改 `SERVER_ADDRESS` 并配置反向代理。

### 3. 启动后端

```bash
set -a
source .env
set +a

cd server
./mvnw spring-boot:run
```

后端默认地址为 `http://localhost:9090`，启动时会初始化项目所需数据表。另开终端确认服务可用：

```bash
curl http://localhost:9090/health
```

成功时返回 `{"code":0,"message":"success","data":"UP"}`。

### 4. 启动前端

```bash
set -a
source .env
set +a

cd client
npm ci
npm run dev
```

浏览器访问 `http://localhost:5173`。开发环境默认通过 Vite 代理访问后端；后端地址不同时修改 `VITE_DEV_PROXY_TARGET`，前后端分开部署时再设置 `VITE_API_BASE_URL`。

只查看前端 Agent 工作台时，可以打开 `http://localhost:5173/?demo`。Demo 模式使用内置示例数据，不依赖后端服务。

### 常见问题

| 现象 | 处理方式 |
| :--- | :--- |
| 后端无法连接 MySQL 或 Redis | 运行 `docker compose --env-file .env ps`，确认服务健康且 `.env` 密码一致 |
| 页面提示无法连接后端 | 先访问 `/health`；再检查 `VITE_DEV_PROXY_TARGET` 或 `VITE_API_BASE_URL` |
| 视频解析提示命令不存在 | 确认 `ffmpeg`、`tesseract` 可在终端执行，必要时配置 `FFMPEG_DIR`、`OCR_COMMAND` |
| AI 接口返回 401 或模型不可用 | 检查 `SILICONFLOW_API_KEY` 与模型名称，修改后重启后端 |
| Maven 提示 `maven-default-http-blocker` | 在 `server` 目录执行 `./mvnw -s .mvn/central-settings.xml spring-boot:run`，临时绕过失效的用户级镜像 |

停止本地中间件：

```bash
docker compose --env-file .env down
```

该命令不会删除 `mysql/data`、`redis/data`、`minio/data`、`qdrant/data` 或 RocketMQ 命名卷。需要完全重置时请先备份，再使用 `docker compose --env-file .env down --volumes` 并手动清理这些数据目录。


## 来源与验证范围

- 上游仓库：[Xiaoc7r/DOVideo-AI](https://github.com/Xiaoc7r/DOVideo-AI)。
- 引入版本：`421dcd2e70e5298ff4d719d2ac8cc4f7c063785b`。
- 原始项目说明见 [docs/UPSTREAM_README.md](docs/UPSTREAM_README.md)。
- 原始许可见 [LICENSE](LICENSE)，保留 `Copyright (c) 2026 Majst`。
- 本次整理未新增实测性能指标，也未完成真实模型、存储、MQ 全依赖端到端验证；本地部署后的结果需独立记录。

## License

沿用上游 [MIT License](LICENSE)。
