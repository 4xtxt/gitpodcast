# GitPodcast 项目深度分析报告

> **项目地址**: https://github.com/BandarLabs/gitpodcast
> **一句话概括**: 将任意 GitHub 仓库转化为一段可收听的播客音频 + 可浏览的幻灯片，帮助快速理解开源项目。
> **在线体验**: https://gitpodcast.com （也可将任意 GitHub URL 中的 `hub` 替换为 `podcast` 直接访问）

---

## 1. 项目整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户浏览器                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ URL 输入框    │  │ 音频播放器    │  │ React Flow 幻灯片视图 │   │
│  │ (MainCard)   │  │ + WebVTT字幕  │  │ (可自动播放/方向键翻页)│   │
│  └──────┬───────┘  └──────▲───────┘  └──────────▲───────────┘   │
└─────────┼──────────────────┼─────────────────────┼───────────────┘
          │                  │                     │
    POST /generate     audio/mpeg            slide_markdown
          │            + X-VTT-Content             │
┌─────────▼────────────────────────────────────────────────────────┐
│                     Frontend (Next.js 15)                         │
│                                                                  │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐   │
│  │ Clerk Auth │  │Server Actions│  │   fetch-backend.ts     │   │
│  │ (登录认证)  │  │ (DB缓存CRUD) │  │ (调用后端API + 缓存)   │   │
│  └────────────┘  └──────┬───────┘  └──────────┬─────────────┘   │
└──────────────────────────┼─────────────────────┼─────────────────┘
                           │                     │
                    PostgreSQL            HTTP POST
                    (Drizzle ORM)              │
┌──────────────────────────────────────────────▼────────────────────┐
│                   Backend (FastAPI + Docker)                       │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    API Routes                                │  │
│  │  POST /generate        → 生成播客 SSML + 音频 MP3           │  │
│  │  POST /generate/slide  → 生成幻灯片 Markdown                │  │
│  │  POST /generate/cost   → 估算本次生成费用                    │  │
│  │  POST /modify          → 用户指令修改 Mermaid 图(备用)       │  │
│  └──────────────────────┬──────────────────────────────────────┘  │
│                         │                                         │
│  ┌──────────────────────▼──────────────────────────────────────┐  │
│  │                    Service Layer                             │  │
│  │                                                              │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                 │  │
│  │  │  GitHubService   │  │  OpenAIService   │                 │  │
│  │  │  获取文件树/README│  │  Azure GPT-4o    │                 │  │
│  │  │  获取关键文件内容 │  │  生成SSML播客稿  │                 │  │
│  │  └──────────────────┘  │  选择重要文件    │                 │  │
│  │                        └──────────────────┘                 │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                 │  │
│  │  │  ClaudeService   │  │  GeminiService   │                 │  │
│  │  │  Claude 3.5 Son. │  │  Gemini 2.0 Flash│                 │  │
│  │  │  Token计数/修改图│  │  (可选)替代生成   │                 │  │
│  │  └──────────────────┘  │  SSML            │                 │  │
│  │                        └──────────────────┘                 │  │
│  │  ┌──────────────────┐  ┌──────────────────┐                 │  │
│  │  │  SpeechService   │  │  SlideService    │                 │  │
│  │  │  Azure TTS Batch │  │  调用OpenAI生成  │                 │  │
│  │  │  SSML→MP3音频    │  │  Marp Markdown   │                 │  │
│  │  │  SSML→WebVTT字幕 │  │  幻灯片内容      │                 │  │
│  │  └──────────────────┘  └──────────────────┘                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────────────────┐   │
│  │ SlowAPI    │  │ API Analytics│  │ Clerk Backend Auth      │   │
│  │ 速率限制   │  │ 请求统计      │  │ 验证登录态              │   │
│  └────────────┘  └──────────────┘  └─────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
                           │
                ┌──────────▼──────────┐
                │     PostgreSQL       │
                │  ┌────────────────┐  │
                │  │ diagram_cache  │  │
                │  │ (图表缓存)     │  │
                │  ├────────────────┤  │
                │  │audio_blob_store│  │
                │  │(音频+字幕缓存) │  │
                │  └────────────────┘  │
                └──────────────────────┘
```

---

## 2. 核心工作流程（Pipeline）

### 2.1 端到端流程

```
用户输入 GitHub URL
       │
       ▼
 ① 解析 username/repo
       │
       ▼
 ② GitHubService: 获取文件树 (git trees API, recursive)
    GitHubService: 获取 README 内容
    GitHubService: 获取默认分支名
       │
       ▼
 ③ OpenAIService.get_important_files():
    用 GPT-4o structured output (Pydantic response_format)
    从文件树中选出 ≤10 个最重要的文件路径
       │
       ▼
 ④ GitHubService: 逐个获取这些重要文件的源码内容
    (每个文件截断到 50,000 字符)
       │
       ▼
 ⑤ 组装内容 → 送入 LLM 生成 SSML 播客稿
    ├── short (~5min): 单次调用，内容 = file_tree + readme + 重要文件
    └── long (~10min): 并发两次调用 (ThreadPoolExecutor)
        ├── 前半段: file_tree + readme → BEFORE_BREAK prompt
        └── 后半段: 重要文件内容 → AFTER_BREAK prompt
        然后拼接两段 SSML，包裹在 <speak> 标签中
       │
       ▼
 ⑥ SpeechService.text_to_mp3():
    Azure Cognitive Services 批量合成 API
    提交 SSML → 轮询状态 → 下载 ZIP → 提取 WAV/MP3
    (16kHz, 32kbps, mono MP3)
       │
       ▼
 ⑦ SpeechService.ssml_to_webvtt():
    解析 SSML 中的纯文本 → 按语速(WPM)计算时间戳
    生成 WebVTT 字幕文件（用于前端同步显示）
       │
       ▼
 ⑧ 同时 SlideService: 用 OpenAI 生成 Marp Markdown 幻灯片
    按 `---` 分隔符切分为多张幻灯片
       │
       ▼
 ⑨ 前端接收:
    - MP3 音频 → URL.createObjectURL → <audio> 标签播放
    - WebVTT 字幕 → 解析为时间轴 → 实时同步显示当前字幕
    - 幻灯片 Markdown → React Flow 画布展示（支持方向键翻页+自动播放）
       │
       ▼
 ⑩ 结果缓存到 PostgreSQL:
    - 音频 base64 + WebVTT → audio_blob_storage 表
    - 下次同一仓库直接读缓存（90% 概率命中缓存）
```

### 2.2 关键设计决策

| 决策 | 原因 |
|------|------|
| 长播客分两段并发生成 | 避免单次 LLM 输出过长导致质量下降或超时 |
| 用 Claude 做 token 计数 | Anthropic 提供精确的 `count_tokens` API，用于预判费用 |
| SSML 用 Azure Batch Synthesis | 支持复杂 SSML 标签（多角色 voice、break），质量高于实时 API |
| 音频存 base64 到数据库 | 避免对象存储依赖，简化自部署；适合中等规模 |
| 前端 90% 缓存命中概率 | 降低重复请求成本，同时留 10% 机会刷新内容 |
| `lru_cache` 缓存 GitHub 数据 | 5 分钟内同一仓库的 generate + cost 请求共享数据，减少 GitHub API 调用 |

---

## 3. 项目目录结构

```
gitpodcast/
├── backend/                          # FastAPI 后端
│   ├── app/
│   │   ├── core/
│   │   │   └── limiter.py            # SlowAPI 速率限制器
│   │   ├── routers/
│   │   │   ├── generate.py           # 核心路由: 播客/幻灯片/费用估算
│   │   │   └── modify.py             # 修改 Mermaid 图（遗留功能）
│   │   ├── services/
│   │   │   ├── github_service.py     # GitHub API 交互（JWT App 认证 + PAT）
│   │   │   ├── openai_service.py     # Azure OpenAI GPT-4o（SSML生成 + 文件选择）
│   │   │   ├── gemini_service.py     # Google Gemini 2.0 Flash（可选 SSML 生成）
│   │   │   ├── claude_service.py     # Anthropic Claude 3.5 Sonnet（token计数 + 图修改）
│   │   │   ├── speech_service.py     # Azure Speech（TTS批量合成 + SSML→WebVTT）
│   │   │   └── slide_service.py      # 幻灯片 Markdown 生成
│   │   ├── prompts.py                # 所有 Prompt 模板（核心资产）
│   │   └── main.py                   # FastAPI 应用入口 + 中间件
│   ├── nginx/                        # Nginx 反向代理配置
│   ├── Dockerfile                    # Python 3.12-slim 镜像
│   ├── entrypoint.sh                 # Docker 启动脚本
│   ├── deploy.sh                     # EC2 部署脚本
│   └── requirements.txt              # Python 依赖
├── src/                              # Next.js 前端
│   ├── app/
│   │   ├── page.tsx                  # 首页（输入框 + 示例仓库）
│   │   ├── [username]/[repo]/
│   │   │   └── page.tsx              # 播客页面（音频+字幕+幻灯片）
│   │   ├── _actions/
│   │   │   ├── cache.ts              # Server Actions: 数据库缓存 CRUD
│   │   │   ├── github.ts             # Server Actions: GitHub star 数获取
│   │   │   └── repo.ts               # Server Actions: 查询生成时间
│   │   ├── layout.tsx                # 根布局
│   │   └── providers.tsx             # 全局状态 Provider
│   ├── components/
│   │   ├── main-card.tsx             # 核心输入卡片（URL输入 + 时长选择）
│   │   ├── hero.tsx                  # 首页标题组件
│   │   ├── loading.tsx               # 加载动画（含费用估算显示）
│   │   ├── loading-animation.tsx     # 加载动画细节
│   │   ├── slide-paginator.tsx       # React Flow 自定义幻灯片节点
│   │   ├── mermaid-diagram.tsx       # Mermaid 图渲染（遗留）
│   │   ├── customization-dropdown.tsx# 自定义/修改下拉菜单
│   │   ├── api-key-dialog.tsx        # 用户自有 API Key 对话框
│   │   ├── api-key-button.tsx        # API Key 按钮
│   │   ├── copy-button.tsx           # 复制按钮
│   │   ├── action-button.tsx         # 操作按钮
│   │   ├── header.tsx                # 页头
│   │   ├── footer.tsx                # 页脚
│   │   └── ui/                       # ShadCN/UI 基础组件
│   ├── hooks/
│   │   └── useDiagram.ts            # 核心 Hook：管理生成/修改/音频/幻灯片全流程
│   ├── lib/
│   │   ├── fetch-backend.ts          # 封装后端 API 调用 + 缓存逻辑
│   │   ├── utils.ts                  # WebVTT 解析/字幕同步工具
│   │   └── exampleRepos.ts           # 示例仓库列表
│   ├── server/db/
│   │   ├── index.ts                  # Drizzle ORM 数据库连接
│   │   └── schema.ts                 # 数据库 Schema（diagram_cache + audio_blob_storage）
│   └── env.js                        # T3 Env 环境变量验证
├── docker-compose.yml                # 后端容器编排
├── middleware.ts                      # Clerk 认证中间件
├── drizzle.config.ts                 # Drizzle Kit 配置
├── tailwind.config.ts                # Tailwind CSS 配置
├── next.config.js                    # Next.js 配置
├── package.json                      # 前端依赖（pnpm）
├── start-database.sh                 # 本地 PostgreSQL 启动脚本
└── .env.example                      # 环境变量模板
```

---

## 4. 技术栈详解

### 4.1 前端

| 层级 | 技术 | 版本 | 用途 |
|------|------|------|------|
| **框架** | Next.js (App Router) | 15.2+ | SSR/SSG、路由、Server Actions |
| **语言** | TypeScript | 5.5+ | 类型安全 |
| **样式** | Tailwind CSS | 3.4+ | 原子化 CSS |
| **UI 组件** | ShadCN/UI + Radix UI | 最新 | 无障碍基础组件（Dialog, Radio, Tooltip等） |
| **认证** | Clerk (@clerk/nextjs) | 6.13+ | 用户登录/注册、Session Token |
| **数据库 ORM** | Drizzle ORM | 0.33+ | 类型安全的 SQL 查询 |
| **画布渲染** | React Flow (@xyflow/react) | 12.4+ | 幻灯片空间导航 |
| **状态管理** | React Context (providers.tsx) | - | 全局状态（audioLength等） |
| **动画** | ldrs | 1.0+ | 加载动画效果 |
| **图标** | Lucide React | 0.468+ | SVG 图标库 |
| **分析** | PostHog (posthog-js) | 1.203+ | 用户行为分析 |
| **Schema 验证** | Zod | 3.23+ | 环境变量验证 (T3 Env) |
| **包管理** | pnpm | 9.13 | 快速/节省磁盘的包管理器 |

### 4.2 后端

| 层级 | 技术 | 版本 | 用途 |
|------|------|------|------|
| **框架** | FastAPI | 0.115+ | ASGI Web 框架 |
| **语言** | Python | 3.12 | 后端逻辑 |
| **服务器** | Uvicorn + uvloop | 0.34+ | 高性能 ASGI 服务器 |
| **速率限制** | SlowAPI | 0.1.9 | 基于 IP 的请求限速 |
| **API 分析** | api-analytics | 1.2.5 | API 请求统计 |
| **HTTP 客户端** | httpx + requests | 0.28 / 2.32 | 外部 API 调用 |
| **数据验证** | Pydantic | 2.10+ | 请求/响应模型验证 |
| **认证** | Clerk Backend API | - | 验证前端传来的 Session Token |
| **音频处理** | pydub | - | 音频时长计算 |
| **容器化** | Docker + docker-compose | - | 后端部署 |
| **反向代理** | Nginx | - | SSL 终止 + 请求转发 |

### 4.3 数据库

| 技术 | 用途 |
|------|------|
| PostgreSQL | 主数据库 |
| Drizzle Kit | Schema 迁移/管理 (`db:push`, `db:studio`) |

**Schema 设计:**
```sql
-- 图表/解释缓存（遗留自 GitDiagram 项目）
diagram_cache (
    username VARCHAR(256),    -- GitHub 用户名
    repo     VARCHAR(256),    -- 仓库名
    diagram  VARCHAR(10000),  -- Mermaid 图代码
    explanation VARCHAR(10000), -- AI 生成的解释
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    PRIMARY KEY (username, repo)
)

-- 音频和字幕缓存
audio_blob_storage (
    username    VARCHAR(256),
    repo        VARCHAR(256),  -- 实际键为 repo|audio_length
    audio_base64 TEXT,         -- MP3 音频的 Base64 编码
    duration    INTEGER,       -- 音频时长（秒）
    format      VARCHAR(50),   -- 音频格式
    webvtt      TEXT,          -- WebVTT 字幕内容
    created_at  TIMESTAMP,
    updated_at  TIMESTAMP,
    PRIMARY KEY (username, repo)
)
```

### 4.4 AI / ML 服务

| 服务 | 模型 | 用途 | 备注 |
|------|------|------|------|
| **Azure OpenAI** | GPT-4o | 生成 SSML 播客稿、选择重要文件、生成幻灯片 Markdown | 核心 LLM，使用 structured output (Pydantic) |
| **Anthropic** | Claude 3.5 Sonnet | Token 计数、修改 Mermaid 图 | 精确 token 计数 API |
| **Google** | Gemini 2.0 Flash Exp | (可选) 替代 GPT-4o 生成 SSML | 免费方案，15 calls/min 限制 |
| **Azure Cognitive Services** | Speech (TTS) | SSML → MP3 音频合成 | 批量合成 API，支持多角色 SSML |

### 4.5 部署

| 组件 | 平台 | 说明 |
|------|------|------|
| **前端** | Vercel | Next.js 原生部署，自动 CI/CD |
| **后端** | AWS EC2 | Docker 容器，Nginx 反代 |
| **数据库** | Neon (Serverless PostgreSQL) | 生产环境使用 Neon；本地可用 Docker PostgreSQL |
| **CI/CD** | GitHub Actions | 后端变更时 SSH 到 EC2 自动部署 |

---

## 5. 关键依赖清单

### 5.1 后端 Python 依赖 (`requirements.txt`)

```
# Web 框架
fastapi==0.115.6
uvicorn==0.34.0
uvloop==0.21.0
starlette==0.41.3

# 速率限制
slowapi==0.1.9
limits==3.14.1

# AI / LLM SDK
openai                          # Azure OpenAI (GPT-4o)
anthropic==0.42.0               # Claude 3.5 Sonnet
google-generativeai             # Gemini 2.0 Flash
google.ai.generativelanguage    # Gemini 底层库

# 语音合成
azure-cognitiveservices-speech  # Azure TTS SDK

# 认证
clerk-backend-api               # Clerk 后端验证
PyJWT==2.10.1                   # GitHub App JWT 签名

# 工具
pydub                           # 音频处理
requests==2.32.3                # HTTP 客户端
httpx==0.28.1                   # 异步 HTTP
python-dotenv==1.0.1            # 环境变量
api-analytics==1.2.5            # API 统计
websockets==14.1                # WebSocket 支持
```

### 5.2 前端核心依赖 (`package.json`)

```json
{
  "框架": {
    "next": "^15.2.3",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "认证": {
    "@clerk/nextjs": "^6.13.0"
  },
  "数据库": {
    "drizzle-orm": "^0.33.0",
    "postgres": "^3.4.4",
    "@neondatabase/serverless": "^0.10.4"
  },
  "UI": {
    "@radix-ui/*": "多个 Radix 原语组件",
    "lucide-react": "^0.468.0",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "tailwind-merge": "^2.5.5"
  },
  "可视化": {
    "@xyflow/react": "^12.4.4",
    "mermaid": "^11.4.1",
    "reveal.js": "^5.1.0"
  },
  "音频": {
    "wavesurfer": "^1.3.4",
    "audiomotion-analyzer": "^4.5.0"
  },
  "分析": {
    "posthog-js": "^1.203.1"
  },
  "验证": {
    "zod": "^3.23.3",
    "@t3-oss/env-nextjs": "^0.10.1"
  }
}
```

---

## 6. 核心 Prompt 工程

项目最核心的资产是 `backend/app/prompts.py` 中的 Prompt 模板：

### 6.1 播客 SSML 生成 Prompt

**目标**: 将代码仓库内容转化为双人对话式 SSML 播客稿

**关键要求**:
- 主持人: `en-US-AvaMultilingualNeural` (女声)
- 嘉宾: `en-US-DustinMultilingualNeural` (男声)
- 长播客分两段: BEFORE_BREAK（file_tree + README）和 AFTER_BREAK（重要文件源码）
- 要求至少 200 个 voice 标签/角色（确保对话充分来回）
- 包含填充词（umm、uh）使对话自然
- 讨论：架构模式、设计原则、组件关系、技术亮点
- 短回答穿插，避免长篇独白

### 6.2 重要文件选择 Prompt

使用 GPT-4o 的 **Structured Output** 能力：
```python
response_format=FileListFormat  # Pydantic model: file_list: List[str]
```
从文件树中选出 ≤10 个最能体现代码架构和高层设计的文件。

### 6.3 幻灯片生成 Prompt

```
Need a markdown (marp) formatted slides from the following github repo code,
which can be presented to users to make them understand the github repo better.
```
输出 Marp Markdown，用 `---` 分隔幻灯片。

---

## 7. 认证与限流体系

### 7.1 认证

```
用户 → Clerk 前端 SDK → Session Token
                ↓
前端请求头: Authorization: Bearer <session_token>
                ↓
后端: Clerk SDK.authenticate_request() 验证 Token
                ↓
未登录用户: 只能使用 short (~5min) 播客
已登录用户: 可使用 long (~10min) 深度播客
```

### 7.2 速率限制

| 端点 | 限制 |
|------|------|
| `GET /` | 100/day |
| `POST /modify` | 2/minute, 10/day |
| `POST /generate` | 当前已临时关闭（注释状态） |
| `POST /generate/cost` | 当前已临时关闭 |

### 7.3 GitHub API 认证（三级降级）

```
1. GitHub App 认证 (JWT + Installation Token) — 最高优先级
2. GitHub Personal Access Token (PAT) — 降级
3. 无认证 (60 req/hr 限制) — 最低降级
```

---

## 8. 缓存策略

```
┌─────────────────────────────────────────────────────┐
│                  三级缓存体系                         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  L1: 内存缓存 (lru_cache, maxsize=100)              │
│      └─ GitHub 数据 (file_tree, readme, file_content)│
│      └─ 5分钟有效，同一仓库多次请求共享               │
│                                                     │
│  L2: 数据库缓存 (PostgreSQL)                         │
│      └─ 音频 base64 + WebVTT 字幕                    │
│      └─ 按 (username, repo|audio_length) 为键       │
│      └─ 90% 概率优先查缓存，避免重新生成              │
│                                                     │
│  L3: React cache (前端 Server Actions)               │
│      └─ GitHub star 数: 5分钟 revalidate             │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 9. 环境变量一览

```bash
# Azure Speech (TTS)
SPEECH_KEY=
SPEECH_REGION=

# Google Gemini (可选 SSML 生成)
GEMINI_API_KEY=

# 数据库
POSTGRES_URL="postgresql://postgres:password@localhost:5432/gitdiagram"

# 后端 API 地址
NEXT_PUBLIC_API_DEV_URL=http://localhost:8000

# Anthropic Claude (token 计数)
ANTHROPIC_API_KEY=

# Azure OpenAI (核心 LLM)
AZURE_OPENAI_API_KEY=
AZURE_OPENAI_ENDPOINT=
AZURE_OPENAI_MODEL_NAME=

# GitHub API (可选，提高速率限制)
GITHUB_PAT=
# 或 GitHub App 认证:
GITHUB_CLIENT_ID=
GITHUB_PRIVATE_KEY=
GITHUB_INSTALLATION_ID=

# Clerk 认证
CLERK_SECRET_KEY=

# API 分析
API_ANALYTICS_KEY=
```

---

## 10. 项目历史与彩蛋

- **前身项目**: 这个项目最初叫 **GitDiagram**（代码中多处残留此名称），是一个将 GitHub 仓库转化为 Mermaid 架构图的项目。后来转型为播客生成，但数据库表名前缀 `gitdiagram_`、部分 prompt（如 `SYSTEM_FIRST_PROMPT` 讲系统架构图）仍保留。
- **`/generate` 端点的 `body.audio=false` 路径**: 返回硬编码的 Mermaid 图代码（clickclickclick 项目的架构图），这是 GitDiagram 的遗留逻辑。
- **Prompt 注释**: `prompts.py` 头部注释提到 originally from GitDiagram，三步 Prompt 流程（explanation → component_mapping → Mermaid.js）仍完整保留，目前只被 `/modify` 路由使用。
- **致谢**: README 中感谢了 [Gitingest](https://gitingest.com/) 和 [Gitdiagram](https://gitdiagram.com/) 的启发和样式参考。

---

## 11. 本地开发快速启动

```bash
# 1. 克隆
git clone https://github.com/BandarLabs/gitpodcast.git && cd gitpodcast

# 2. 前端依赖
pnpm i

# 3. 环境变量
cp .env.example .env   # 填入各服务 API Key

# 4. 后端 (Docker)
docker-compose up --build -d

# 5. 数据库
chmod +x start-database.sh && ./start-database.sh
pnpm db:push            # 初始化 Schema

# 6. 前端开发
pnpm dev                # → localhost:3000
```

---

## 12. 总结

GitPodcast 是一个精巧的 **AI 内容转换管道**，其核心创新在于：

1. **代码→播客** 的跨模态转换：将静态代码仓库转化为沉浸式的音频对话
2. **多模型协作**：GPT-4o（内容理解+SSML生成）+ Claude（Token计数）+ Azure Speech（语音合成）+ Gemini（可选替代）
3. **SSML 多角色对话**：精心设计的 Prompt 让 LLM 输出带 `<voice>` 标签的 SSML，实现自然的双人播客对话
4. **三级缓存 + 并发处理**：在控制成本的同时保证响应速度
5. **幻灯片同步播放**：音频 + WebVTT 字幕 + React Flow 空间导航的沉浸式体验

整个项目基于 **T3 Stack**（Next.js + TypeScript + Tailwind + Drizzle + PostgreSQL），后端 **FastAPI + Docker**，部署在 **Vercel + EC2** 上，是一个典型的全栈 AI 应用架构。
