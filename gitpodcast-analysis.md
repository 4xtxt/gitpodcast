# GitPodcast 项目深度分析

> **项目地址**: https://github.com/BandarLabs/gitpodcast
> **官网**: https://www.gitpodcast.com
> **协议**: MIT | **Stars**: 807+ | **Forks**: 92+

---

## 1. 项目概述

GitPodcast 是一个将任意公开 GitHub 仓库转化为 AI 生成播客节目的工具。用户输入一个 GitHub 仓库 URL，系统自动抓取仓库的文件树、README 和关键源码文件，利用 LLM 生成两位主持人（Ava 和 Dustin）的对话式 SSML 脚本，再通过 Azure TTS 合成语音，最终输出一集约 5~20 分钟的播客音频，配有实时字幕和可交互的幻灯片。

该项目从同一组织的 **GitDiagram**（仓库架构图生成器）演化而来，数据库表前缀仍为 `gitdiagram_`，并保留了 Mermaid 图表生成管线。

---

## 2. 整体架构

```
┌──────────────┐
│  用户浏览器   │
└──────┬───────┘
       │
       v
┌──────────────────────────────────────────┐
│  Clerk Auth Middleware                    │
└──────┬───────────────────────────────────┘
       │
       v
┌──────────────────────────────────────────┐
│  Next.js 15 前端 (Vercel 部署)            │
│  - App Router / Server Actions           │
│  - Drizzle ORM ↔ PostgreSQL (Neon)       │
│  - 缓存层：90% 概率命中缓存              │
└──────┬──────────────────┬────────────────┘
       │ Cache Miss        │ Cache Hit
       v                   │ → 直接返回
┌──────────────────────┐   │
│  FastAPI 后端         │   │
│  (Docker / AWS EC2)  │   │
│  Nginx 反向代理       │   │
└──────┬───────────────┘   │
       │                   │
       ├──→ GitHub API (仓库数据抓取)
       ├──→ Azure OpenAI GPT-4o (SSML脚本 + 文件重要性排序)
       ├──→ Anthropic Claude 3.5 Sonnet (架构图生成)
       ├──→ Google Gemini 2.0 Flash (备选SSML生成)
       ├──→ Azure Batch TTS (SSML → MP3)
       │
       v
┌──────────────────────┐
│  PostgreSQL (Neon)   │ ← 缓存音频/字幕/图表
└──────────────────────┘
```

**核心架构特点**:

| 特性 | 说明 |
|------|------|
| **前后端分离** | Next.js 前端部署在 Vercel，FastAPI 后端运行在 AWS EC2 Docker 容器中 |
| **概率性缓存** | 前端以 90% 概率读取 PostgreSQL 缓存，10% 概率重新生成以保持内容新鲜度 |
| **并行生成** | 长播客的两段 SSML 通过 Python `ThreadPoolExecutor` 并行生成；音频和幻灯片通过 `Promise.all` 并行请求 |
| **LRU 内存缓存** | 后端对 GitHub 数据做 LRU 缓存 (maxsize=100)，避免重复 API 调用 |
| **多模型协作** | GPT-4o 生成内容、Claude 生成图表、Azure TTS 合成语音 |
| **优雅降级** | GitHub 认证支持 GitHub App → PAT → 匿名三级回退 |
| **认证门控** | "深入模式"（长播客）需要 Clerk 登录认证 |

---

## 3. 技术栈总览

### 3.1 前端

| 技术 | 版本 | 用途 |
|------|------|------|
| **Next.js** | 15.2+ | React 全栈框架，App Router + Turbopack |
| **TypeScript** | 5.5 | 类型安全 |
| **Tailwind CSS** | 3.4 | 样式框架 |
| **ShadCN/UI** | — | UI 组件库 (基于 Radix UI) |
| **ReactFlow** (@xyflow/react) | 12.4 | 可交互幻灯片导航画布 |
| **Mermaid.js** | 11.4 | 架构图渲染 |
| **wavesurfer.js** | — | 音频波形可视化 |
| **audiomotion-analyzer** | — | 音频频谱可视化 |
| **react-markdown** | — | Markdown 渲染 |
| **Reveal.js** | 5.1 | 幻灯片演示框架 |
| **Clerk** (@clerk/nextjs) | 6.13 | 前端认证 |
| **PostHog** | — | 产品分析 |
| **Drizzle ORM** | 0.33 | 数据库 ORM |

### 3.2 后端

| 技术 | 版本 | 用途 |
|------|------|------|
| **FastAPI** | 0.115 | Python 异步 Web 框架 |
| **Uvicorn** | 0.34 | ASGI 服务器 |
| **Python** | 3.12 | 运行时 |
| **slowapi** | 0.1.9 | 请求限流 |
| **pydub** + ffmpeg | — | 音频处理 |
| **Pydantic** | 2.10 | 数据校验 |
| **PyJWT** | — | GitHub App JWT 认证 |

### 3.3 AI/LLM 服务

| 服务 | 模型 | 用途 |
|------|------|------|
| **Azure OpenAI** | GPT-4o | SSML 播客脚本生成 + 文件重要性排名 |
| **Anthropic** | Claude 3.5 Sonnet | 架构图生成 (3步管线) + Token 计数 |
| **Google Gemini** | 2.0 Flash Exp | 备选 SSML 生成器 |
| **Azure Cognitive Services Speech** | Batch Synthesis API | SSML → MP3 语音合成 |

**TTS 语音**:
- **Ava** (主持人): `en-US-AvaMultilingualNeural`
- **Dustin** (嘉宾): `en-US-DustinMultilingualNeural`

### 3.4 基础设施

| 组件 | 技术 |
|------|------|
| **数据库** | PostgreSQL (Neon Serverless / Docker 本地) |
| **前端部署** | Vercel |
| **后端部署** | AWS EC2 + Docker + Nginx 反向代理 |
| **CI/CD** | GitHub Actions (SSH 部署到 EC2) |
| **容器化** | Docker + docker-compose |
| **包管理** | pnpm 9.13 |
| **项目脚手架** | create-t3-app 7.38.1 |

---

## 4. 项目目录结构

```
gitpodcast/
├── .github/
│   └── workflows/
│       └── deploy.yml                  # CI/CD: SSH 部署后端到 EC2
│
├── backend/                            # FastAPI Python 后端
│   ├── Dockerfile                      # Python 3.12-slim, ffmpeg, ALSA
│   ├── requirements.txt
│   ├── deploy.sh                       # EC2 部署脚本
│   ├── entrypoint.sh                   # Docker 入口
│   ├── app/
│   │   ├── main.py                     # FastAPI 应用：CORS、中间件
│   │   ├── prompts.py                  # 所有 LLM Prompt 定义
│   │   ├── core/
│   │   │   └── limiter.py              # slowapi 限流器实例
│   │   ├── routers/
│   │   │   ├── generate.py             # POST /generate (播客+音频+幻灯片+成本)
│   │   │   └── modify.py              # POST /modify (修改 Mermaid 图表)
│   │   └── services/
│   │       ├── github_service.py       # GitHub API：文件树、README、文件内容
│   │       ├── openai_service.py       # Azure OpenAI：SSML 生成 + 文件排名
│   │       ├── gemini_service.py       # Google Gemini：备选 SSML
│   │       ├── claude_service.py       # Claude：图表生成 + Token 计数
│   │       ├── speech_service.py       # Azure TTS：批量合成、SSML 校验、WebVTT
│   │       └── slide_service.py        # 幻灯片 Markdown 生成
│   └── nginx/
│       ├── api.conf                    # Nginx 反向代理配置
│       └── setup_nginx.sh
│
├── src/                                # Next.js 前端
│   ├── app/
│   │   ├── layout.tsx                  # 根布局
│   │   ├── page.tsx                    # 首页 (Hero + MainCard)
│   │   ├── providers.tsx               # PostHog + 全局状态
│   │   ├── [username]/[repo]/
│   │   │   └── page.tsx               # 仓库页：音频播放器、字幕、幻灯片、图表
│   │   └── _actions/
│   │       ├── cache.ts               # Server Actions：DB 缓存读写
│   │       ├── github.ts             # 获取 Star 数
│   │       └── repo.ts               # 获取最近生成时间
│   ├── components/
│   │   ├── main-card.tsx              # URL 输入表单 + 认证门控
│   │   ├── hero.tsx / header.tsx / footer.tsx
│   │   ├── api-key-dialog.tsx         # API Key 输入对话框
│   │   ├── mermaid-diagram.tsx        # Mermaid 图表渲染
│   │   ├── slide-paginator.tsx        # ReactFlow 幻灯片
│   │   └── ui/                        # ShadCN UI 组件
│   ├── hooks/
│   │   └── useDiagram.ts             # 核心 Hook：编排音频/幻灯片/图表生成
│   ├── lib/
│   │   ├── fetch-backend.ts          # 后端 API 调用 + 缓存逻辑
│   │   ├── exampleRepos.ts           # 首页示例仓库
│   │   └── utils.ts                  # 工具函数：WebVTT 解析、字幕同步
│   ├── server/db/
│   │   ├── index.ts                  # Drizzle 数据库连接
│   │   └── schema.ts                 # 数据库 Schema
│   └── env.js                        # T3 环境变量校验
│
├── middleware.ts                       # Clerk 认证中间件
├── docker-compose.yml                  # 后端服务定义
├── drizzle.config.ts                   # Drizzle Kit 配置
├── start-database.sh                   # 本地 PostgreSQL 启动脚本
└── .env.example                        # 环境变量模板
```

---

## 5. 核心工作流：从 GitHub URL 到播客

### 阶段 1: 用户输入

1. 用户在 `MainCard` 组件中输入 GitHub 仓库 URL
2. 选择音频长度："Basic"（约 5 分钟，无需登录）或 "In Depth"（约 10-20 分钟，需 Clerk 登录）
3. 前端跳转到 `/[username]/[repo]` 动态路由页面

### 阶段 2: 自动触发生成

4. `useDiagram` Hook 在组件挂载时通过 `useEffect` 自动调用 `handleAudio()`
5. `handleAudio()` 并行发起两个请求：
   - `generateAudio()` — 音频生成
   - `generateSlide()` — 幻灯片生成

### 阶段 3: 缓存检查

6. 两个函数均以 **90% 概率**检查 PostgreSQL 缓存（10% 概率绕过以保持新鲜度）
7. 缓存键格式：`username + repo + "|" + audio_length`
8. 命中则直接返回 base64 音频 + WebVTT 字幕；未命中则调用后端

### 阶段 4: GitHub 数据抓取

9. `POST /generate` 到达后端 `generate.py` 路由
10. `GitHubService` 调用 GitHub API：
    - 获取默认分支名
    - 递归获取文件树（过滤二进制文件、assets、依赖目录）
    - 获取 README 内容
11. `OpenAIService.get_important_files()` 将文件树发给 GPT-4o，用 Pydantic 结构化输出选出 **最重要的 10 个文件**
12. 逐个获取这些文件的内容（每个最多 50,000 字符）

### 阶段 5: SSML 脚本生成

13. **短播客**: 单次 LLM 调用，输入文件树 + README + 文件内容，使用 `PODCAST_SSML_PROMPT`
14. **长播客**: 两次 **并行** LLM 调用（`ThreadPoolExecutor`）：
    - Part 1 (上半段): 文件树 + README → `PODCAST_SSML_PROMPT_BEFORE_BREAK`
    - Part 2 (下半段): 重要文件内容 → `PODCAST_SSML_PROMPT_AFTER_BREAK`
15. 生成的 SSML 经 XML 解析校验、清洗（移除非 `<voice>` 子节点），失败则最多重试 3 次
16. 长播客的两段 SSML 合并为单个 `<speak>` 信封

**Prompt 特点**: 要求生成 Ava（主持人）和 Dustin（嘉宾）之间的自然对话，包含口语化填充词（"umm"、"uh"），每位至少 200 个 `<voice>` 标签，目标时长约 20 分钟。

### 阶段 6: 语音合成

17. `SpeechService.text_to_mp3()` 将 SSML 提交给 **Azure Batch Synthesis API**：
    - 输出格式：`audio-16khz-32kbitrate-mono-mp3`
    - 轮询状态直到 "Succeeded"
    - 下载 ZIP 结果，提取音频文件
18. 音频以 `audio/mpeg` HTTP 响应返回
19. `SpeechService.ssml_to_webvtt()` 从 SSML 生成 WebVTT 字幕：
    - 剥离 XML 标签提取纯文本
    - 基于实际音频时长和词数计算时间戳
    - 生成带时间戳的字幕条目
20. WebVTT 经 base64 编码放入 `X-VTT-Content` 响应头

### 阶段 7: 缓存与播放

21. 前端接收 MP3 Blob + base64 VTT
22. 音频转 base64 后存入 PostgreSQL `audioBlobStorage` 表
23. 通过 `URL.createObjectURL()` 生成音频 URL 供 `<audio>` 元素播放
24. VTT 解码后解析为字幕条目，通过 `timeupdate` 事件实现实时字幕同步
25. 幻灯片按 `---` 分隔后渲染为 ReactFlow 节点

---

## 6. 关键依赖详解

### 前端依赖 (package.json)

| 依赖 | 用途 |
|------|------|
| `next` 15.2 | App Router, Server Actions, SSR, Turbopack |
| `react` / `react-dom` 18.3 | UI 渲染 |
| `@clerk/nextjs` 6.13 | 用户认证（登录、会话、中间件路由保护） |
| `@neondatabase/serverless` 0.10 | Neon PostgreSQL 无服务器驱动 |
| `drizzle-orm` 0.33 | 类型安全 SQL ORM |
| `@t3-oss/env-nextjs` + `zod` | 运行时环境变量校验 |
| `@radix-ui/react-*` | 无头 UI 原语 (Dialog, Label, Progress, RadioGroup 等) |
| `class-variance-authority` + `clsx` + `tailwind-merge` | Tailwind CSS 工具组合 (ShadCN 模式) |
| `lucide-react` | 图标库 |
| `mermaid` 11.4 | Mermaid.js 图表渲染 |
| `@xyflow/react` 12.4 | ReactFlow 交互式画布 |
| `react-markdown` + `remark-gfm` + `rehype-highlight` | Markdown 渲染 + 语法高亮 |
| `reveal.js` 5.1 | 幻灯片演示 |
| `wavesurfer` + `audiomotion-analyzer` | 音频波形/频谱可视化 |
| `posthog-js` | 产品数据分析 |
| `geist` | 字体 |

### 后端依赖 (requirements.txt)

| 依赖 | 用途 |
|------|------|
| `fastapi` 0.115 | 异步 Web 框架 |
| `uvicorn` 0.34 | ASGI 服务器 |
| `slowapi` 0.1.9 | API 限流 |
| `openai` | Azure OpenAI SDK（SSML 生成 + 文件排名） |
| `anthropic` 0.42 | Claude SDK（图表生成 + Token 计数） |
| `google-generativeai` | Gemini SDK（备选 SSML 生成器） |
| `azure-cognitiveservices-speech` | Azure TTS 语音合成 |
| `pydub` | 音频操作（时长计算） |
| `requests` | HTTP 客户端（GitHub API 调用） |
| `PyJWT` | GitHub App JWT 认证 |
| `pydantic` 2.10 | 数据校验/序列化 |
| `clerk-backend-api` | Clerk 服务端认证 |
| `api-analytics` 1.2.5 | API 使用分析中间件 |

---

## 7. 环境变量配置

| 变量 | 必需 | 所属 | 说明 |
|------|------|------|------|
| `SPEECH_KEY` | 是 | 后端 | Azure 语音服务订阅密钥 |
| `SPEECH_REGION` | 是 | 后端 | Azure 语音服务区域 (如 "eastus") |
| `AZURE_OPENAI_API_KEY` | 是 | 后端 | Azure OpenAI API 密钥 |
| `AZURE_OPENAI_ENDPOINT` | 是 | 后端 | Azure OpenAI 端点 URL |
| `AZURE_OPENAI_MODEL_NAME` | 否 | 后端 | 模型部署名 (默认 gpt-4o) |
| `ANTHROPIC_API_KEY` | 是 | 后端 | Anthropic Claude API 密钥 |
| `GEMINI_API_KEY` | 否 | 后端 | Google Gemini API 密钥 (备选) |
| `POSTGRES_URL` | 是 | 前端 | PostgreSQL 连接字符串 |
| `NEXT_PUBLIC_API_DEV_URL` | 否 | 前端 | 后端 API 地址 (默认 api.gitpodcast.com) |
| `GITHUB_PAT` | 否 | 后端 | GitHub PAT (提升速率限制到 5000 req/hr) |
| `GITHUB_CLIENT_ID` | 否 | 后端 | GitHub App Client ID |
| `GITHUB_PRIVATE_KEY` | 否 | 后端 | GitHub App 私钥 |
| `GITHUB_INSTALLATION_ID` | 否 | 后端 | GitHub App Installation ID |
| `CLERK_SECRET_KEY` | 是 | 后端 | Clerk 后端认证密钥 |
| `NEXT_PUBLIC_CLERK_*` | 是 | 前端 | Clerk 前端密钥 |
| `NEXT_PUBLIC_POSTHOG_KEY` | 否 | 前端 | PostHog 分析密钥 |
| `API_ANALYTICS_KEY` | 否 | 后端 | api-analytics 密钥 |

---

## 8. 关键源文件速查

| 文件 | 职责 |
|------|------|
| `backend/app/routers/generate.py` | **核心编排器**：串联所有服务，处理并行 SSML 生成、音频转换、幻灯片、成本估算 |
| `backend/app/services/github_service.py` | GitHub API 封装：文件树、README、文件内容获取，支持多种认证方式 |
| `backend/app/services/openai_service.py` | Azure OpenAI 封装：SSML 生成 + 结构化输出文件排名 |
| `backend/app/services/speech_service.py` | Azure TTS 批量合成、SSML 校验/清洗、WebVTT 字幕生成 |
| `backend/app/services/claude_service.py` | Claude 封装：3 步图表生成管线 + Token 计数 |
| `backend/app/services/gemini_service.py` | Gemini 封装：备选 SSML 生成器 |
| `backend/app/prompts.py` | 所有 LLM Prompt 定义 |
| `src/hooks/useDiagram.ts` | 前端核心 Hook：编排音频/幻灯片/图表生成全流程 |
| `src/lib/fetch-backend.ts` | 前端 API 客户端：后端调用 + 90% 概率缓存逻辑 |
| `src/app/_actions/cache.ts` | Next.js Server Actions：PostgreSQL 缓存 CRUD |
| `src/server/db/schema.ts` | Drizzle 数据库 Schema：`diagramCache` + `audioBlobStorage` |
| `src/components/main-card.tsx` | 首页 URL 输入表单 + 认证门控选项 |
| `src/app/[username]/[repo]/page.tsx` | 仓库页：音频播放器、实时字幕、幻灯片查看器 |

---

## 9. 值得注意的设计细节

1. **从 GitDiagram 演化**: 数据库表前缀为 `gitdiagram_`，图表生成管线（3 步 Claude Prompt）在播客流程中大部分是遗留功能。

2. **概率性缓存策略**: 90% 命中 / 10% 绕过的设计比较独特——意味着即使有缓存，约 1/10 的请求仍会重新生成内容（成本较高）。

3. **字幕时间近似**: WebVTT 字幕并非使用 Azure TTS 返回的精确词级时间戳，而是根据总音频时长和词数推算每词时间，可能存在累积漂移。

4. **幻灯片校验为空操作**: `SlideService.is_valid_markdown()` 始终返回 `True`。

5. **Token 计数跨模型**: 虽然内容生成用 Azure OpenAI，Token 计数却使用 Anthropic SDK 的 `count_tokens`。

6. **硬编码图表回退**: 当 `audio=false` 时，`/generate` 端点会返回一个与当前仓库无关的静态 Mermaid 图表。

---

## 10. 本地开发快速指引

```bash
# 1. 克隆项目
git clone https://github.com/BandarLabs/gitpodcast.git
cd gitpodcast

# 2. 安装前端依赖
pnpm install

# 3. 启动本地 PostgreSQL
./start-database.sh

# 4. 配置环境变量
cp .env.example .env
# 编辑 .env 填入所有必需的 API 密钥

# 5. 数据库迁移
pnpm db:push

# 6. 启动前端
pnpm dev

# 7. 启动后端 (另一个终端)
cd backend
docker-compose up --build
```
