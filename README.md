> 融合大模型、RAG、本地攻略与高德地图能力的智能旅行规划助手

智能旅行规划助手是一个面向中文旅行场景的 AI 旅行规划项目。用户输入目的地、日期、预算、人数和偏好后，系统会自动生成结构化旅行方案，并进一步补充地图点位、天气信息、预算拆分、景点图片与可导出的旅行文档。

相比只输出一段文本的 LLM Demo，这个项目更强调完整链路落地：从 **行程生成、攻略检索、地图信息补全、天气补充，到历史管理与文档导出**，尽量把 AI 能力组织成一个可交互、可保存、可展示的产品原型。

---

## ✨ 项目亮点

- 🧠 **LLM 行程生成**：基于 LangChain + DashScope 调用 `qwen-max` 生成结构化旅行计划
- 📚 **RAG 攻略增强**
  - 本地 Markdown 攻略 + Chroma 向量检索，为生成结果补充目的地上下文
  - 在线阶段通过 LLM-based Query Rewrite + Cross-encoder Rerank（qwen3-rerank）+ 噪声预过滤持续优化检索质量，Top1 命中率 93.3%，MRR 0.967
- 🗺️ **高德地图接入**：补充景点地址、经纬度、POI ID、路线距离、耗时和景点图片，并支持虚线箭头路线可视化与 🚩 打卡标记
- 🌦️ **天气感知提示**：前端展示天气预报，并根据雨天/阴天自动修正旅行提示
- ⚡ **Redis 缓存层**：覆盖天气、地图、RAG 检索与 Rerank 结果缓存，减少重复外部调用开销
- 📊 **Token 消耗统计**：按 Query Rewrite、Query Embedding、Rerank、Planner 分项统计输入/输出 token，并在后端日志与接口响应中返回总量
- 💰 **预算拆分**：按交通、住宿、餐饮、门票、其他费用拆分，并支持按天展示
- 🪄 **智能编辑**：支持用户用自然语言调整某一天行程
- 🗂️ **历史管理**：支持保存、查看、打开、删除历史 itinerary
- 📄 **文档导出**：支持 Markdown 和中文 PDF 导出，导出前自动同步当前页面数据
- 🖥️ **前端可视化**：提供规划页、结果页和历史页，完成核心业务闭环展示

---

## 🏗️ 技术架构

### 技术栈

- 后端：FastAPI + Pydantic + SQLAlchemy
- LLM：LangChain + DashScope (`qwen-max`)
- 向量库：ChromaDB
- 缓存：Redis
- 外部服务：HTTPX + 高德地图 Web 服务 + 高德 JavaScript API
- 前端：Vue 3 + Vite
- 数据库：SQLite

### 核心架构分层

| 层级     | 关键文件                   | 职责                                          |
| :------- | :------------------------- | :-------------------------------------------- |
| 前端     | `frontend/src/views/*.vue` | 规划页、结果页、历史页展示与交互              |
| 接口层   | `backend/app/api/routes/`  | trip、export、weather 路由                    |
| 服务层   | `backend/app/services/`    | 行程编排、地图 enrich、天气、缓存、导出、存储 |
| Agent 层 | `backend/app/agents/`      | LLM 行程生成 + LLM-based Query Rewrite        |
| RAG 层   | `backend/app/rag/`         | 向量入库、检索、Cross-encoder Rerank          |
| 数据层   | `backend/data/`            | 本地 Markdown 攻略文档                        |

### 系统数据流

```mermaid
flowchart TD
    Client(("浏览器"))

    %% ------- Frontend -------
    subgraph Frontend["Frontend"]
        Vue["Vue 页面"]
        Api["api.ts"]
    end
    class Frontend frontendBg;

    %% ------- Backend -------
    subgraph Backend["Backend"]
        Main["FastAPI main.py"]

        subgraph Routes["Routes"]
            Trip["trip.py"]
            Export["export.py"]
            Weather["weather.py"]
        end

        subgraph Services["Services"]
            TripSvc["trip_service.py"]
            MapSvc["map_service.py"]
            WeatherSvc["weather_service.py"]
            ExportSvc["export_service.py"]
            StorageSvc["storage_service.py"]
            CacheSvc["cache_service.py"]
        end

        subgraph Agent["Agent"]
            Planner["trip_planner_agent.py"]
            RagTool["rag_tool.py"]
        end

        subgraph RAG["RAG"]
            Retriever["retriever.py"]
            VectorDB["vector_db.py"]
            ChromaDB[("ChromaDB")]
        end

        Schemas["schemas.py"]
        DBModels["db_models.py"]
        Redis[("Redis")]
        SQLite[("SQLite")]
    end
    class Backend backendBg;

    %% ------- 主流程（实线） -------
    Client --> Vue --> Api --> Main

    Main --> Trip
    Main --> Export
    Main --> Weather

    Trip --> TripSvc
    Trip --> Schemas
    Weather --> WeatherSvc
    Export --> ExportSvc

    TripSvc --> Planner
    TripSvc --> MapSvc
    TripSvc --> StorageSvc
    TripSvc --> CacheSvc

    Planner --> RagTool
    RagTool --> Retriever
    Retriever --> VectorDB
    VectorDB --> ChromaDB
    Retriever --> CacheSvc

    CacheSvc --> Redis
    StorageSvc --> DBModels
    DBModels --> SQLite

    %% ------- 返回路径（虚线） -------
    TripSvc -.-> Api
    WeatherSvc -.-> Api
    ExportSvc -.-> Api

    %% ------- Colors -------
    classDef frontend fill:#eef2ff,stroke:#818cf8,color:#111;
    classDef backend fill:#fefce8,stroke:#facc15,color:#111;
    classDef routes fill:#f0fdfa,stroke:#2dd4bf,color:#111;
    classDef services fill:#f5f3ff,stroke:#a78bfa,color:#111;
    classDef agent fill:#fff1f2,stroke:#fb7185,color:#111;
    classDef rag fill:#ecfeff,stroke:#22d3ee,color:#111;
    classDef data fill:#f0fdf4,stroke:#4ade80,color:#111;
    classDef storage fill:#fff7ed,stroke:#fb923c,color:#111;

    %% 背景框颜色（Frontend、Backend）
    classDef frontendBg fill:#eef2ff,stroke:#818cf8,stroke-width:2px,color:#111;
    classDef backendBg fill:#fffbea,stroke:#facc15,stroke-width:2px,color:#111;

    %% ------- Assign Colors -------
    class Client,Vue,Api frontend;
    class Main backend;
    class Trip,Export,Weather routes;
    class TripSvc,MapSvc,WeatherSvc,ExportSvc,StorageSvc,CacheSvc services;
    class Planner,RagTool agent;
    class Retriever,VectorDB,ChromaDB rag;
    class Schemas,DBModels data;
    class Redis,SQLite storage;
```

数据流路径：前端收集用户输入 → 后端调用 LLM + RAG 生成结构化行程 → 地图服务补充地址、坐标、路线和图片 → 前端展示地图、天气、预算和每日行程 → 用户可保存、编辑、查看历史并导出文档。

### 数据存储与缓存分工

项目中将长期业务数据和短期高频查询结果分开处理：

- **SQLite：负责持久化存储**
  - 实现位置：`backend/app/config.py`、`backend/app/models/db_models.py`、`backend/app/services/storage_service.py`
  - 使用场景：保存用户生成后的完整旅行方案，并支持历史列表、详情查询、删除和 Markdown/PDF 导出。
  - 存储方式：通过 SQLAlchemy 定义 `TripRecord` 表，核心字段包括 `trip_id`、`destination`、`summary`、`itinerary_json`、`created_at`、`updated_at`。
  - 设计原因：旅行方案属于用户主动保存的业务数据，需要长期保留、可查询、可删除；当前阶段采用 SQLite 轻量部署，适合个人项目和 Demo 场景。

- **Redis：负责缓存加速**
  - 实现位置：`backend/app/services/cache_service.py`，并被 `weather_service.py`、`map_service.py`、`retriever.py` 复用。
  - 使用场景：缓存天气查询、高德地图地理编码/POI/路线结果、RAG 检索结果和 qwen3-rerank 重排序结果。
  - 存储方式：业务模块生成缓存 key，`cache_service.py` 统一加上 `trip_planner` 前缀，将 Python `dict/list` 序列化为 JSON 字符串写入 Redis，并设置 TTL 自动过期。
  - 设计原因：天气、地图和 RAG/Rerank 结果存在明显重复查询，且在一段时间内相对稳定；使用 Redis 可以减少外部 API 调用和重复检索开销，提升接口响应速度与稳定性。

简言之：**SQLite 存“用户要留下来的行程数据”，Redis 存“短时间内可复用的中间查询结果”。**

### RAG 检索流程

```mermaid
%%{init: {"layout": "elk"}}%%
flowchart TD
    %% ------- Offline -------
    subgraph Offline
        Guides[("data 攻略文档")]
        Ingest["ingest_data.py"]
        Embed["text-embedding-v4"]
        DB[("ChromaDB")]

        Guides --> Ingest
        Ingest --> Embed
        Embed --> DB
    end

    %% ------- Online -------
    subgraph Online
        Input("用户输入 目的地 偏好 节奏 备注")
        QR{"Query Rewrite"}
        LLM_QR["LLM-based qwen-max"]
        Rule_QR["规则级 fallback"]
        Cache{"RAG 缓存命中?"}
        Vector["ChromaDB 向量召回"]
        Noise["噪声预过滤"]
        Rerank{"Cross-encoder Rerank"}
        DS["qwen3-rerank"]
        Rule_RR["规则级 fallback"]
        SetCache["写入 Redis 缓存"]
        Output("返回 top-k 片段给 LLM")

        Input --> QR
        QR -->|优先| LLM_QR
        QR -->|fallback| Rule_QR
        LLM_QR --> Cache
        Rule_QR --> Cache
        Cache -->|命中| Output
        Cache -->|未命中| Vector
        Vector --> Noise
        Noise --> Rerank
        Rerank -->|优先| DS
        Rerank -->|fallback| Rule_RR
        DS --> SetCache
        Rule_RR --> SetCache
        SetCache --> Output
    end

    DB --> Vector

    %% ------- Color definitions -------
    classDef offline fill:#fefce8,stroke:#facc15;
    classDef online_input fill:#eef2ff,stroke:#818cf8;
    classDef online_logic fill:#f0fdfa,stroke:#2dd4bf;
    classDef retrieve fill:#fdf4ff,stroke:#e879f9;
    classDef rerank fill:#fff1f2,stroke:#fb7185;
    classDef output fill:#f0fdf4,stroke:#4ade80;

    class Guides,Ingest,Embed,DB offline;
    class Input online_input;
    class QR,LLM_QR,Rule_QR,Cache,Vector,Noise online_logic;
    class Rerank,DS,Rule_RR rerank;
    class SetCache,Output output;
```

**离线阶段**

```text
本地 Markdown 攻略 → 按标题切块（49 个片段） → text-embedding-v4 转向量 → 写入 ChromaDB
```

这一步只做一次，数据入库后就不再动了。

**在线阶段**

```text
用户输入（目的地 / 偏好 / 节奏 / 备注）
    ↓
① Query Rewrite（LLM-based / 规则 fallback）
    输出：检索关键词，如"大理 美食 拍照 古城 洱海"
    ↓
② Embedding（同一个 text-embedding-v4）
    把检索关键词转向量，才能和 ChromaDB 里的文档向量做相似度计算
    ↓
③ 向量召回（ChromaDB）
    用向量相似度找到 top-6 候选片段
    ↓
④ 噪声预过滤
    去掉"文档开头"等低信息量片段，避免浪费 rerank 的 API 调用
    ↓
⑤ Cross-encoder Rerank（qwen3-rerank / 规则 fallback）
    语义级重排序，选出 top-3 最相关片段
    ↓
⑥ 写入 Redis 缓存
    RAG 缓存：query → top-k 文本
    Rerank 缓存：query + 候选哈希 → 排序分数
    ↓
⑦ 返回 top-k 文本给 LLM
    和用户信息一起组装成 prompt，调 qwen-max 生成行程
```

---

## 📁 项目结构

```text
TripPlannerDemo/
├── backend/
│   ├── app/
│   │   ├── config.py          # 环境变量、数据库 Base、全局配置
│   │   ├── agents/
│   │   │   ├── trip_planner_agent.py    # LLM 行程生成与单日编辑逻辑
│   │   │   └── tools/
│   │   │       └── rag_tool.py          # Query Rewrite：LLM-based 改写 + 规则级 fallback
│   │   ├── api/
│   │   │   ├── main.py                  # FastAPI 应用入口
│   │   │   └── routes/
│   │   │       ├── trip.py              # 生成、编辑、保存、查询、删除接口
│   │   │       ├── export.py            # Markdown / PDF 导出接口
│   │   │       └── weather.py           # 天气预报接口
│   │   ├── models/
│   │   │   ├── schemas.py               # Pydantic 请求体 / 响应体 / itinerary 模型
│   │   │   └── db_models.py             # SQLAlchemy 数据库表定义
│   │   ├── rag/
│   │   │   ├── vector_db.py             # Markdown 切片、Chroma 入库与检索
│   │   │   └── retriever.py             # 检索封装、RAG 缓存、Cross-encoder Rerank + 规则级 fallback
│   │   └── services/
│   │       ├── trip_service.py          # 行程主编排逻辑、预算计算、地图 enrich
│   │       ├── cache_service.py         # Redis 缓存封装与降级逻辑
│   │       ├── map_service.py           # 高德地图 POI、地理编码、路线、图片补充
│   │       ├── weather_service.py       # 高德天气服务封装
│   │       ├── storage_service.py       # SQLite 保存、查询、列表、删除
│   │       └── export_service.py        # Markdown / PDF 渲染与导出
│   ├── data/                  # 本地攻略文档
│   ├── eval/                  # RAG 检索评估样例集
│   ├── scripts/               # ingest、地图验证、RAG 调试与评估脚本
│   ├── tests/                 # pytest 测试
│   ├── Dockerfile             # 后端容器打包配置
│   ├── .dockerignore          # Docker 构建排除规则
│   ├── .env.example           # 后端环境变量模板
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── services/
│   │   │   └── api.ts                   # Axios 封装与前端 API 调用
│   │   ├── types/
│   │   │   └── index.ts                 # TypeScript 数据类型定义
│   │   ├── views/
│   │   │   ├── Home.vue                 # 规划页
│   │   │   ├── Result.vue               # 结果展示页
│   │   │   └── History.vue              # 历史列表页
│   │   ├── components/
│   │   │   └── AmapTripMap.vue          # 地图展示组件
│   │   ├── App.vue                      # 页面切换入口
│   │   └── main.ts                      # 前端入口
│   ├── Dockerfile             # 前端两阶段构建（Node 编译 → Nginx 托管）
│   ├── nginx.conf             # Nginx 配置（静态文件 + API 反向代理）
│   ├── .dockerignore          # Docker 构建排除规则
│   ├── .env.example           # 前端环境变量模板
│   └── package.json
├── docker-compose.yaml        # 服务编排（后端 + 前端 + Redis）
├── assets/
│   └── showcase/              # README 展示截图
├── CHANGELOG.md               # 项目功能与架构更新日志
├── .gitignore
└── README.md
```

> `docs/` 是本地开发与面试准备文档目录，默认已被 `.gitignore` 忽略，不随 GitHub 上传。

### 关键文件职责

**后端**

- `backend/app/services/trip_service.py`
  itinerary 主流程编排，包括天数拆分、预算估算、地图 enrich 以及编辑后的统一刷新。
- `backend/app/services/cache_service.py`
  Redis 客户端懒加载、JSON 缓存读写与 Redis 不可用时的优雅降级。
- `backend/app/agents/trip_planner_agent.py`
  调用大模型生成结构化旅行草稿，并处理单日编辑时的 LLM 输出。
- `backend/app/agents/tools/rag_tool.py`
  RAG 在线阶段的 Query Rewrite，优先 LLM-based 改写（qwen-max），fallback 到规则级关键词提取。
- `backend/app/rag/retriever.py`
  向量召回结果封装、RAG 缓存、Cross-encoder Rerank（qwen3-rerank）+ Rerank 缓存，fallback 到规则级打分。
- `backend/app/services/map_service.py`
  对接高德地图 Web 服务，结合 Redis 缓存补充地址、经纬度、路线估算和景点图片。
- `backend/app/services/export_service.py`
  itinerary 渲染为 Markdown 与中文 PDF。
- `backend/app/services/storage_service.py`
  SQLite 数据保存、读取、历史列表和删除。
- `backend/scripts/debug_rag_retrieval.py`
  RAG 在线阶段调试，输出检索 query、top-k 召回片段、`rerank_score` 与 `rerank_reasons`。
- `backend/scripts/evaluate_rag_retrieval.py`
  RAG 检索效果评估，输出 Top1/TopK 命中率、MRR、Noise Rate、Latency 与跨目的地污染指标。
- `backend/eval/rag_eval_cases.json`
  RAG 检索评估样例集，用于对比优化前后的效果变化。

**前端**

- `frontend/src/services/api.ts`
  Axios 封装与后端接口通信。
- `frontend/src/views/Home.vue`
  规划页，收集用户输入并发起行程生成请求。
- `frontend/src/views/Result.vue`
  结果展示页，承接 itinerary、地图、天气和导出交互。
- `frontend/src/views/History.vue`
  历史列表页，支持查看、打开和删除历史行程。
- `frontend/src/components/AmapTripMap.vue`
  高德地图组件，展示路线可视化与景点标记。

---

## 🚀 快速启动

以下命令默认从项目根目录 `TripPlannerDemo/` 开始执行。

### 1. 启动 Redis（可选）

```bash
docker run -d --name tripplanner-redis -p 6379:6379 redis:7
```

如果已创建过容器：

```bash
docker start tripplanner-redis
```

在 `backend/.env` 中设置 `REDIS_ENABLED=true` 开启缓存（天气、地图、RAG 检索与 Rerank 结果）。

### 2. 启动后端

```bash
cd TripPlannerDemo
cd backend
pip install -r requirements.txt
# 手动复制 .env.example 为 .env，并填写你的配置
uvicorn app.api.main:app --host 0.0.0.0 --port 8000
```

启动后访问：

```text
http://127.0.0.1:8000/
http://127.0.0.1:8000/docs
```

### 3. 启动前端

```bash
cd TripPlannerDemo
cd frontend
npm install
# 手动复制 .env.example 为 .env，并填写你的配置
npm run dev
```

启动后访问：

```text
http://127.0.0.1:5173
```

---

## 🐳 Docker 部署

项目支持 Docker Compose 一键部署，将后端、前端和 Redis 打包为三个容器，统一编排管理。

### 架构

```text
浏览器
  │
  ▼
┌─────────────────────────────────┐
│  Nginx（前端容器，端口 80）       │
│  ├── 返回 Vue 静态文件           │
│  └── /api/* 代理到后端 ─────────┼──┐
└─────────────────────────────────┘  │
                                     ▼
┌─────────────────────────────────┐
│  FastAPI（后端容器，端口 8000）   │
│  行程生成 / 编辑 / 天气 / 导出   │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Redis（缓存容器，端口 6379）     │
│  缓存天气、地图、RAG 结果        │
└─────────────────────────────────┘
```

### 新增文件说明

| 文件                     | 作用                                                         |
| ------------------------ | ------------------------------------------------------------ |
| `backend/Dockerfile`     | 后端打包：Python 3.11 精简镜像，安装依赖后运行 uvicorn       |
| `frontend/Dockerfile`    | 前端两阶段构建：Node 20 编译 Vue → Nginx alpine 托管静态文件 |
| `frontend/nginx.conf`    | Nginx 配置：静态文件托管 + API 反向代理到后端容器            |
| `docker-compose.yaml`    | 服务编排：定义 Redis、后端、前端三个容器的依赖关系和端口映射 |
| `backend/.dockerignore`  | 排除 `__pycache__`、`.env`、`db/` 等不需要打包的文件         |
| `frontend/.dockerignore` | 排除 `node_modules`、`dist` 等不需要打包的文件               |

### 启动

> **前提**：需要先安装并启动 [Docker Desktop](https://www.docker.com/products/docker-desktop/)，等待左下角显示"Engine running"后再执行以下命令。

```bash
# 首次构建（需要几分钟下载镜像和安装依赖）
docker compose up --build

# 后台运行
docker compose up --build -d
```

启动后访问 `http://localhost`。

### 停止

```bash
docker compose down
```

### 关键设计

- **两阶段构建**：前端用 Node 编译出静态文件后，只把产物复制到 Nginx 镜像，最终镜像不含 Node.js，体积从几百 MB 缩小到几十 MB。
- **Nginx 反向代理**：前端静态文件由 Nginx 直接返回，API 请求通过 `proxy_pass` 转发给后端容器，统一入口，避免跨域问题。
- **层缓存优化**：后端 Dockerfile 先复制 `requirements.txt` 安装依赖，再复制代码。改代码时不会重新安装依赖。
- **环境变量隔离**：`.env` 文件通过 `env_file` 注入容器，不打包进镜像，避免泄露 API Key。
- **数据持久化**：Redis 数据、SQLite 数据库和知识库 markdown 文件通过 Docker volumes 挂载，容器重建后数据不丢失。

---

## 🔐 环境变量

### 后端 `backend/.env`

```env
# LLM
LLM_PROVIDER=openai_compatible          # 固定值，使用 OpenAI 兼容接口
LLM_API_KEY=your_dashscope_api_key      # DashScope API Key
LLM_MODEL=qwen-max                      # 生成模型
LLM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_TIMEOUT_SECONDS=60                  # 单次 LLM 调用超时
LLM_MAX_RETRIES=1                       # 失败重试次数

# RAG / 向量库
CHROMA_DB_DIR=db/chroma_db              # ChromaDB 持久化目录
CHROMA_COLLECTION_NAME=travel_guides    # 集合名称
EMBEDDING_MODEL=text-embedding-v4       # DashScope 嵌入模型
EMBEDDING_BATCH_SIZE=10                 # 单批嵌入条数
RERANK_MODEL=qwen3-rerank              # DashScope Rerank 模型

# Redis / 缓存
REDIS_ENABLED=false                     # 是否开启缓存（需先启动 Redis）
REDIS_URL=redis://127.0.0.1:6379/0     # Redis 连接地址
REDIS_KEY_PREFIX=trip_planner           # 缓存 key 前缀，避免多项目冲突
REDIS_DEFAULT_TTL_SECONDS=1800          # 默认缓存 30 分钟
REDIS_WEATHER_TTL_SECONDS=1800          # 天气缓存 30 分钟
REDIS_MAP_TTL_SECONDS=86400             # 地图缓存 24 小时
REDIS_RAG_TTL_SECONDS=21600             # RAG 检索缓存 6 小时
REDIS_RERANK_TTL_SECONDS=21600          # Rerank 缓存 6 小时

# 高德地图
AMAP_API_KEY=your_amap_web_service_key  # 高德 Web 服务 Key
AMAP_BASE_URL=https://restapi.amap.com/v3
AMAP_DEFAULT_CITY=                      # 默认城市（可留空）
AMAP_TIMEOUT_SECONDS=20                 # 高德接口超时
ENABLE_AMAP_ENRICHMENT=true             # 是否开启地图信息补全
```

### 前端 `frontend/.env`

```env
VITE_API_BASE_URL=http://你的服务器地址:8000
VITE_AMAP_JS_KEY=your_amap_javascript_api_key
```

注意：

- 如果浏览器在本机打开，`VITE_API_BASE_URL` 不要写远程服务器内部的 `127.0.0.1`
- 后端高德 key 使用 Web 服务 key
- 前端地图 key 使用 JavaScript API key
- 修改 `.env` 后需要重启对应服务

---

## 🧠 RAG 数据初始化

首次使用 Chroma 检索前，执行：

```bash
cd backend
python scripts/ingest_data.py
```

成功后会看到类似结果：

```text
written_count: 9
```

---

## 📡 核心接口

| 方法     | 路径                         | 说明                            |
| :------- | :--------------------------- | :------------------------------ |
| `GET`    | `/`                          | 服务启动检查                    |
| `GET`    | `/health`                    | 健康检查                        |
| `POST`   | `/trip/generate`             | 生成行程                        |
| `GET`    | `/trip/stats`                | 查询已保存行程的 token 消耗统计 |
| `POST`   | `/trip/edit`                 | 智能编辑行程                    |
| `POST`   | `/trip/save`                 | 保存行程                        |
| `GET`    | `/trip`                      | 历史列表                        |
| `GET`    | `/trip/{trip_id}`            | 行程详情                        |
| `DELETE` | `/trip/{trip_id}`            | 删除行程                        |
| `GET`    | `/export/{trip_id}/markdown` | 导出 Markdown                   |
| `GET`    | `/export/{trip_id}/pdf`      | 导出 PDF                        |
| `GET`    | `/weather/forecast`          | 查询天气                        |

---

## 🧪 测试与验证

### 后端 API 测试

```bash
cd backend
pytest tests/test_api_trip.py -q
```

如果服务器测试目录是 `backend/test`：

```bash
cd backend/test
pytest test_api_trip.py -q
```

### 高德服务测试

```bash
cd backend/scripts
python test_map_service.py
```

### 真实行程生成测试

```bash
cd backend/scripts
python test_trip_service_real.py
```

---

## 🔄 关键业务链路

### 显式编排工作流

项目采用显式编排（而非 Agent 自主决策）的方式组织业务流程，每个步骤由 `trip_service.py` 按固定顺序调用，适合当前业务确定性强、步骤可预期的场景。

```mermaid
flowchart TD
    User(("用户"))
    FE["Frontend"]
    Route["trip.py 路由层"]
    TripSvc["trip_service.py 主编排"]

    subgraph 编排步骤
        Step1["① RAG 检索"]
        Step2["② LLM 行程生成"]
        Step3["③ 地图信息补全"]
        Step4["④ 天气查询"]
        Step5["⑤ 预算拆分"]
    end

    RAG["rag_tool.py + retriever.py"]
    LLM["trip_planner_agent.py qwen-max"]
    Map["map_service.py 高德地图"]
    Weather["weather_service.py 高德天气"]
    Result["返回 Itinerary"]

    User --> FE --> Route --> TripSvc
    TripSvc --> Step1 --> RAG
    RAG --> Step2 --> LLM
    LLM --> Step3 --> Map
    Map --> Step4 --> Weather
    Weather --> Step5 --> Result
    Result -.-> FE -.-> User

    classDef user fill:#eef2ff,stroke:#818cf8,color:#111;
    classDef route fill:#f0fdfa,stroke:#2dd4bf,color:#111;
    classDef svc fill:#fffbea,stroke:#facc15,color:#111;
    classDef step fill:#fdf4ff,stroke:#e879f9,color:#111;
    classDef ext fill:#fff1f2,stroke:#fb7185,color:#111;
    classDef out fill:#f0fdf4,stroke:#4ade80,color:#111;

    class User,FE user;
    class Route route;
    class TripSvc svc;
    class Step1,Step2,Step3,Step4,Step5 step;
    class RAG,LLM,Map,Weather ext;
    class Result out;
```

### 行程生成

```text
POST /trip/generate
  -> trip.py（路由层）
    -> trip_service.py（主编排）
      -> ① rag_tool.py
           Query Rewrite（LLM-based / 规则 fallback）
           -> retriever.py
               RAG 缓存检查
               -> ChromaDB 向量召回
               -> 噪声预过滤
               -> Cross-encoder Rerank（缓存 -> API -> 规则 fallback）
      -> ② trip_planner_agent.py
           组装 Prompt（用户输入 + RAG 上下文）
           -> qwen-max 生成结构化行程
           -> Pydantic 校验输出
      -> ③ map_service.py（逐景点）
           地理编码 -> POI 搜索 -> 路线估算 -> 图片补充
           （每步都有 Redis 缓存）
      -> ④ weather_service.py
           天气预报查询（Redis 缓存）
      -> ⑤ 预算拆分计算
      -> ⑥ 记录 token_usage（Query Rewrite / Query Embedding / Rerank / Planner / Total）
      -> 返回 Itinerary
```

### 智能编辑

```text
POST /trip/edit
  -> trip.py（路由层）
    -> trip_service.py（主编排）
      -> ① 定位目标 DayPlan（根据 edit_scope 解析 day_index）
      -> ② trip_planner_agent.py
           generate_day_edit_draft（LLM 生成单日编辑）
           -> 失败则 fallback 到规则编辑（关键词匹配）
      -> ③ 替换目标 DayPlan（theme / spots / meals / notes）
      -> ④ map_service.py 重新 enrich（清除旧坐标，重新查询）
      -> ⑤ 更新 tips 和 source_notes
      -> 返回更新后的 Itinerary
```

### 保存与导出

```text
POST /trip/save
  -> storage_service.py -> SQLite 持久化

GET /export/{trip_id}/markdown
  -> storage_service.py 读取 itinerary
  -> export_service.py -> Jinja2 渲染 Markdown

GET /export/{trip_id}/pdf
  -> storage_service.py 读取 itinerary
  -> export_service.py -> ReportLab 生成中文 PDF
  -> Content-Disposition 返回下载文件名（RFC 编码兼容中文）
```

---

## 🛠️ 常见问题

### 前端生成失败

优先检查：

- 后端是否启动在 `8000`
- `frontend/.env` 的 `VITE_API_BASE_URL` 是否正确
- 修改 `.env` 后是否重启前端
- 浏览器控制台是否有网络错误

### 地图不显示

优先检查：

- `VITE_AMAP_JS_KEY` 是否配置
- 高德 JavaScript API key 是否可用
- itinerary 中是否有经纬度字段
- 后端 `ENABLE_AMAP_ENRICHMENT` 是否为 `true`

### PDF 导出空白页

正常导出时后端应看到：

```text
POST /trip/save
GET /export/{trip_id}/pdf
```

如果只有 `POST /trip/save`，说明前端没有成功跳转到导出地址，需要刷新前端或重启 Vite。

### `npm run dev` 找不到 `package.json`

说明目录错了。前端命令必须在 `frontend/` 目录执行：

```bash
cd frontend
```

---

## ✅ 当前完成度

- ✅ **后端能力**：行程生成、智能编辑、保存查询、历史列表、删除、天气查询、Markdown 导出与 PDF 导出接口
- ✅ **AI 与数据能力**：LangChain 行程生成链路、5 个目的地攻略 RAG 检索、Chroma 入库检索、高德地图地址/坐标/路线/图片补充
- ✅ **RAG 在线优化**：LLM-based Query Rewrite + Cross-encoder Rerank（qwen3-rerank）+ 噪声预过滤 + Rerank 缓存、检索调试脚本与 15 条评估样例集、量化评估指标体系（Top1/TopK Hit Rate、MRR、Noise Rate、Latency、Cross-destination Pollution）
- ✅ **Token 观测能力**：`/trip/generate` 返回本次 Query Rewrite、Query Embedding、Rerank、Planner 的分项 token 消耗，后端终端同步打印 prompt/completion/total，`/trip/stats` 汇总已保存行程的 token 统计
- ✅ **前端能力**：规划页、结果页、历史列表页，以及地图/天气/预算展示、导出与历史管理主流程
- ✅ **缓存与持久化**：SQLite 持久化存储 + Redis 缓存层（覆盖天气、地图、RAG 检索与 Rerank 结果）
- ✅ **验证情况**：核心链路稳定跑通，Redis 缓存 key 可在本地容器中验证写入

---

## 🌱 后续优化方向

- ✅ **缓存与工程化能力**
  已完成 Redis 缓存层，覆盖天气查询、地图查询、RAG 检索结果与 Rerank 结果缓存；后续可扩展到会话态管理、热点目的地复用与更细粒度的缓存命中统计。
- ✅ **RAG 检索增强**
  - ✅ 规则级 Query Rewrite → LLM-based Query Rewrite（qwen-max），Top1 80%→86.7%，MRR 0.889→0.922。
  - ✅ 规则级 Rerank → Cross-encoder Rerank（qwen3-rerank）+ 噪声预过滤 + Rerank 缓存，Top1 86.7%→93.3%，MRR 0.922→0.967。
  - ✅ 知识库扩充至 5 个目的地，评估样例集 15 条，量化评估指标体系完整。
- 🚧 **Token 成本分析看板**
  已完成后端 token 统计与 `/trip/stats` 汇总接口，后续可在前端增加成本分析面板，对比不同 RAG 策略下的 token 消耗、延迟和生成质量。
- 🚧 **检索结果压缩与去冗**
  RAG 召回片段可能存在重复或冗余信息，送入 LLM 前做一次压缩去重，减少 token 消耗，提升生成质量。
- 🚧 **混合检索（向量 + BM25）**
  当前只用向量检索，加上 BM25 关键词检索后用 RRF（Reciprocal Rank Fusion）融合排序，同时覆盖语义相似和关键词精确匹配的场景。
- 🚧 **PDF 导出优化**
  当前 PDF 可读性较低，后续可优化排版（分栏、卡片式布局）、中文字体、景点图片嵌入、天气图标和路线示意图，生成更接近旅行手册风格的导出文档。
- 🚧 **知识库来源扩充**
  可接入小红书等社交平台的旅行帖子，通过多模态解析（图文提取、结构化摘要）将真实游记转化为本地知识库素材，补充官方攻略覆盖不到的体验细节和实用 tips。
- 🚧 **真实商户信息展示**
  后端接入真实餐饮、酒店/民宿数据（如高德 POI 详情、大众点评等），泛化为结构化数据（名称、地址、评分、人均、图片等），前端以卡片形式展示，提升行程的实用性和可信度。
- 🚧 **联网搜索增强**
  可接入联网搜索能力，补充景点营业状态、近期热门地点、节假日信息与实时出行建议，让本地攻略 RAG 与实时信息形成互补。
- 🚧 **旅行方案质量评估体系**
  建立生成结果的量化评估指标，例如结构完整性、预算合理性、景点覆盖率、天气一致性和用户偏好满足度，实现端到端的效果度量。
- 🚧 **性能与稳定性**
  可以加入异步任务队列、请求限流、失败重试、日志追踪与监控告警，提升真实部署场景下的稳定性。
- 🚧 **产品能力延展**
  可以继续增强移动端适配、用户登录、多用户隔离、行程对比和行程分享等产品能力。
