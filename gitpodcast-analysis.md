# GitPodcast 项目深度分析

## 1. 项目概述

**GitPodcast** 是一个将任意 GitHub 仓库转换为播客（podcast）的 Web 应用。用户输入 GitHub 仓库 URL，系统会自动分析仓库结构、关键文件和 README，利用大语言模型生成 SSML（语音合成标记语言）格式的播客脚本，再通过 Azure Speech Service 将其转换为带有字幕的音频播客。此外还支持生成基于 Marp 格式的幻灯片演示。

- **官网**: [https://gitpodcast.com](https://gitpodcast.com)
- **GitHub**: [https://github.com/BandarLabs/gitpodcast](https://github.com/BandarLabs/gitpodcast)
- **许可证**: MIT
- **灵感来源**: [Gitingest](https://gitingest.com/) 和 [Gitdiagram](https://gitdiagram.com/)

---

## 2. 整体架构

项目采用 **前后端分离** 架构：

```
┌─────────────────────────────────────────────────────────┐
│                      用户浏览器                          │
│  Next.js 前端 (Vercel/localhost:3000)                    │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ 页面组件 │  │ Server Actions│  │ Drizzle ORM/PG    │  │
│  │ (React)  │  │ (缓存逻辑)   │  │ (数据库缓存)      │  │
│  └────┬─────┘  └──────┬───────┘  └───────────────────┘  │
│       │               │                                  │
│       └───────┬───────┘                                  │
│               │ HTTP API 调用                            │
└───────────────┼─────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────────────┐
│  FastAPI 后端 (EC2 Docker/localhost:8000)                  │
│  ┌────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ GitHub     │  │ LLM 服务层   │  │ Azure Speech      │  │
│  │ Service    │  │ (OpenAI/     │  │ Service           │  │
│  │ (获取仓库) │  │  Gemini/     │  │ (SSML→MP3+VTT)    │  │
│  │            │  │  Claude)     │  │                   │  │
│  └────────────┘  └──────────────┘  └───────────────────┘  │
└───────────────────────────────────────────────────────────┘
        │                  │                   │
        ▼                  ▼                   ▼
  GitHub API        Azure OpenAI /        Azure Cognitive
  (仓库数据)        Gemini Flash /         Services Speech
                    Anthropic Claude       (TTS Batch API)
```

---

## 3. 核心处理流程

### 3.1 播客生成主流程

```
用户输入 GitHub URL
        │
        ▼
① GitHubService: 获取仓库数据
   ├── get_default_branch()     → 仓库默认分支
   ├── get_github_file_paths_as_list() → 文件树（过滤掉 node_modules 等）
   ├── get_github_readme()       → README 内容
   └── get_github_file_content() → OpenAI 筛选的重要文件内容（≤10个文件）
        │
        ▼
② OpenAIService.get_important_files() → 从文件树中筛选最关键的 ≤10 个文件
        │
        ▼
③ 组装内容 + 发送给 LLM 生成 SSML
   ├── 短播客 (~5min): 单次调用 → PODCAST_SSML_PROMPT
   └── 长播客 (~10min): 并发两次调用
       ├── 线程1: file_tree + readme → PODCAST_SSML_PROMPT_BEFORE_BREAK（上半场）
       └── 线程2: important files   → PODCAST_SSML_PROMPT_AFTER_BREAK（下半场）
       → 合并两个 SSML 片段为完整播客
        │
        ▼
④ SpeechService: SSML → 音频 + 字幕
   ├── text_to_mp3()     → Azure Batch Synthesis API → MP3 字节
   ├── ssml_to_webvtt()  → 根据语速估算时间戳 → WebVTT 字幕
   └── 返回 MP3 + base64(VTT) 在 HTTP 响应头中
        │
        ▼
⑤ 前端缓存 + 播放
   ├── audioBlob → <audio> 播放器
   ├── WebVTT → 实时字幕同步显示
   └── PostgreSQL 缓存（90% 概率命中缓存）
```

### 3.2 幻灯片生成流程

```
文件树 + 文件内容 → OpenAIService → Marp 格式 Markdown
→ 前端用 @xyflow/react 渲染为可导航的幻灯片画布
→ 每张幻灯片用 unified/remark/rehype 渲染为 HTML
→ 支持键盘方向键导航 + 自动播放
```

### 3.3 缓存机制

- **后端**: `@lru_cache(maxsize=100)` 缓存 GitHub 数据（5分钟内避免重复 API 调用）
- **前端**: PostgreSQL 双表缓存
  - `diagram_cache` — 缓存 Mermaid 图表 + 解释文本
  - `audio_blob_storage` — 缓存 base64 音频 + WebVTT 字幕 + 幻灯片 Markdown
- 缓存键: `(username, repo)` 或 `(username, repo|audio_length)`
- 90% 概率优先读缓存，10% 概率跳过缓存强制重新生成

---

## 4. 技术栈详解

### 4.1 前端

| 技术 | 版本 | 用途 |
|------|------|------|
| **Next.js** | ^15.2.3 | React 全栈框架，SSR/SSG + Server Actions |
| **React** | ^18.3.1 | UI 渲染 |
| **TypeScript** | ^5.5.3 | 类型安全 |
| **Tailwind CSS** | ^3.4.3 | 原子化 CSS 样式 |
| **ShadCN/Radix UI** | 多个包 | 无障碍 UI 组件库（Dialog, Radio, Tooltip 等） |
| **Clerk** | @clerk/nextjs ^6.13.0 | 用户认证（Sign in → 解锁长播客） |
| **@xyflow/react** | ^12.4.4 | 幻灯片画布导航（React Flow） |
| **Drizzle ORM** | ^0.33.0 | 类型安全 ORM，操作 PostgreSQL |
| **PostHog** | posthog-js ^1.203.1 | 前端行为分析 |
| **Mermaid** | ^11.4.1 | Mermaid 图表渲染（继承自 GitDiagram） |
| **unified + remark + rehype** | 多个包 | Markdown → HTML 渲染管线（幻灯片内容） |
| **Zod** | ^3.23.3 | 环境变量 Schema 验证（via @t3-oss/env-nextjs） |

### 4.2 后端

| 技术 | 版本 | 用途 |
|------|------|------|
| **FastAPI** | 0.115.6 | Python 异步 Web 框架 |
| **Uvicorn** | 0.34.0 | ASGI 服务器 |
| **Python** | 3.12 | 运行时 |
| **Azure Cognitive Services Speech** | — | TTS 语音合成（Batch Synthesis API） |
| **OpenAI (Azure)** | openai 包 | Azure OpenAI GPT-4o 调用 + 结构化输出 |
| **Google Generative AI** | google-generativeai | Gemini 2.0 Flash Exp（免费 SSML 生成） |
| **Anthropic** | anthropic 0.42.0 | Claude 3.5 Sonnet（token 计数 + 图表修改） |
| **SlowAPI** | 0.1.9 | API 限流 |
| **Pydantic** | 2.10.3 | 请求/响应数据验证 |
| **Clerk Backend API** | clerk-backend-api | JWT 认证验证（长播客权限控制） |
| **pydub** | — | 音频处理（获取时长） |
| **api-analytics** | 1.2.5 | API 调用分析 |

### 4.3 数据库

| 技术 | 用途 |
|------|------|
| **PostgreSQL** | 主数据库，存储缓存 |
| **Drizzle ORM** | TypeScript ORM，Schema 定义 + 迁移 |
| **Neon Database** | 生产环境无服务器 Postgres（Vercel 集成） |
| **postgres.js** | 本地开发 Postgres 驱动 |

数据库表:

- `gitdiagram_diagram_cache` — 缓存仓库架构图 + 解释文本
  - 主键: `(username, repo)`
  - 字段: `diagram(varchar 10000)`, `explanation(varchar 10000)`, `created_at`, `updated_at`
- `gitdiagram_audio_blob_storage` — 缓存音频 + 字幕 + 幻灯片
  - 主键: `(username, repo)`
  - 字段: `audioBase64(text)`, `duration(integer)`, `format(varchar)`, `webVtt(text)`, `created_at`, `updated_at`

### 4.4 部署

| 组件 | 平台 | 方式 |
|------|------|------|
| **前端** | Vercel | `next build` + 自动部署 |
| **后端** | AWS EC2 | Docker 容器 + Nginx 反向代理 |
| **数据库** | Neon (生产) / Docker Postgres (本地) | — |
| **CI/CD** | GitHub Actions | `deploy.yml` → SSH 到 EC2 执行部署脚本 |

---

## 5. 关键依赖详解

### 5.1 AI/LLM 服务层

项目使用了 **三套 LLM 服务**，各司其职：

| 服务 | 实现类 | 模型 | 用途 |
|------|--------|------|------|
| **Azure OpenAI** | `OpenAIService` | GPT-4o | (1) 筛选重要文件 (`get_important_files`，使用 structured output / Pydantic 模式) (2) 生成 SSML 脚本 |
| **Google Gemini** | `GeminiService` | Gemini 2.0 Flash Exp | 免费替代方案，用于 SSML 生成（需手动切换代码） |
| **Anthropic Claude** | `ClaudeService` | Claude 3.5 Sonnet | (1) Token 计数 (2) 图表修改 (`/modify` 端点) |

**当前默认管线**: Azure OpenAI 生成 SSML → Azure Speech 生成音频。README 提到也可切换为 Gemini Flash（免费方案）。

### 5.2 Azure Speech Service

- **模式**: Batch Synthesis API（异步批处理，非实时流式）
- **流程**: PUT 请求创建合成任务 → 轮询状态 → 下载 ZIP（内含 WAV/MP3）
- **输出格式**: `audio-16khz-32kbitrate-mono-mp3`
- **SSML 语音角色**:
  - 主持人: `en-US-AvaMultilingualNeural`
  - 嘉宾: `en-US-DustinMultilingualNeural`
- **重试机制**: 最多 3 次重试，每次间隔 2 秒

### 5.3 GitHub API

- **认证方式**: 支持三种（优先级递减）
  1. GitHub App（JWT + Installation Token，5000 请求/小时）
  2. Personal Access Token（PAT，5000 请求/小时）
  3. 无认证（60 请求/小时，会打印警告）
- **获取数据**:
  - `GET /repos/{owner}/{repo}/git/trees/{branch}?recursive=1` → 文件树
  - `GET /repos/{owner}/{repo}/readme` → README
  - `GET /repos/{owner}/{repo}/contents/{path}` → 文件内容（base64 解码）
- **过滤**: 排除 `node_modules/`, `.min.`, 图片, 缓存, 锁文件等

### 5.4 认证

- **Clerk**: 前端 `@clerk/nextjs` + 后端 `clerk-backend-api`
- **用途**: 长播客（~10min）需要登录后才能访问
- **中间件**: `middleware.ts` 使用 `clerkMiddleware` 保护路由

---

## 6. 项目目录结构

```
gitpodcast/
├── .env.example                  # 环境变量模板
├── docker-compose.yml            # 后端 Docker 编排
├── drizzle.config.ts             # Drizzle ORM 配置
├── next.config.js                # Next.js 配置
├── package.json                  # 前端依赖 + 脚本
├── middleware.ts                  # Clerk 认证中间件
│
├── backend/                      # ========== Python 后端 ==========
│   ├── Dockerfile                # Python 3.12-slim + ffmpeg
│   ├── entrypoint.sh             # Docker 启动脚本
│   ├── deploy.sh                 # EC2 部署脚本
│   ├── requirements.txt          # Python 依赖
│   ├── nginx/                    # Nginx 反向代理配置
│   │   ├── api.conf
│   │   └── setup_nginx.sh
│   └── app/
│       ├── main.py               # FastAPI 入口，CORS + 限流 + 路由注册
│       ├── prompts.py            # 所有 LLM Prompt 模板
│       ├── core/
│       │   └── limiter.py         # SlowAPI 限流配置
│       ├── routers/
│       │   ├── generate.py        # 核心生成端点 (/generate, /generate/slide, /generate/cost)
│       │   └── modify.py          # 图表修改端点 (/modify)
│       └── services/
│           ├── github_service.py  # GitHub API 封装
│           ├── openai_service.py  # Azure OpenAI 封装
│           ├── gemini_service.py  # Gemini Flash 封装
│           ├── claude_service.py  # Anthropic Claude 封装
│           ├── speech_service.py  # Azure TTS 封装 + VTT 生成
│           └── slide_service.py   # 幻灯片 Markdown 生成
│
├── src/                          # ========== Next.js 前端 ==========
│   ├── app/
│   │   ├── layout.tsx            # 根布局（Clerk + ReactFlow + PostHog + GlobalState）
│   │   ├── page.tsx             # 首页
│   │   ├── providers.tsx        # PostHog + GlobalState Context Provider
│   │   ├── [username]/[repo]/
│   │   │   └── page.tsx         # 仓库播客详情页
│   │   └── _actions/
│   │       ├── cache.ts         # Server Actions: 缓存读写
│   │       ├── repo.ts          # Server Actions: 仓库元数据查询
│   │       └── github.ts        # GitHub Star 计数
│   ├── components/
│   │   ├── main-card.tsx        # 主交互卡片（URL 输入 + 提交 + 示例仓库）
│   │   ├── hero.tsx             # 首页 Hero 区域
│   │   ├── header.tsx / footer.tsx
│   │   ├── slide-paginator.tsx  # 幻灯片组件（React Flow 节点 + Markdown 渲染）
│   │   ├── customization-dropdown.tsx
│   │   ├── api-key-dialog.tsx
│   │   ├── loading.tsx / loading-animation.tsx
│   │   └── ui/                  # ShadCN 基础组件
│   ├── hooks/
│   │   └── useDiagram.ts        # 核心自定义 Hook：编排所有 API 调用 + 状态管理
│   ├── lib/
│   │   ├── fetch-backend.ts     # 后端 API 封装（generate, modify, cost, audio, slide）
│   │   ├── exampleRepos.ts      # 示例仓库列表
│   │   └── utils.ts             # 工具函数（WebVTT 解析 + 字幕同步）
│   ├── server/
│   │   └── db/
│   │       ├── index.ts         # Drizzle 数据库连接（Neon/本地自动切换）
│   │       └── schema.ts        # 数据库 Schema 定义
│   ├── styles/globals.css
│   └── env.js                   # 环境变量 Schema 验证（@t3-oss/env-nextjs）
│
├── .github/workflows/
│   └── deploy.yml                # CI/CD: 推送到 EC2（当前触发分支: donotdeploy）
└── public/                      # 静态资源
```

---

## 7. API 端点

| 方法 | 路径 | 功能 | 关键参数 |
|------|------|------|----------|
| POST | `/generate` | 生成播客音频或架构图 | `username`, `repo`, `instructions`, `audio: bool`, `audio_length: "short"/"long"` |
| POST | `/generate/slide` | 生成幻灯片 Markdown | `username`, `repo`, `instructions` |
| POST | `/generate/cost` | 预估生成成本 | `username`, `repo` |
| POST | `/modify` | 修改已有架构图 | `username`, `repo`, `instructions`, `current_diagram`, `explanation` |
| GET | `/` | 健康检查 | — |

---

## 8. Prompt 工程

项目在 `backend/app/prompts.py` 中定义了所有 LLM Prompt，核心策略：

1. **SSML 播客脚本生成**: 要求 LLM 将仓库信息转换为双角色（Host: Ava, Guest: Dustin）对话式播客 SSML
   - 要求至少 200 个 voice 标签（主持人 + 嘉宾各 200 个）
   - 添加自然语气词（"umm", "uh"）
   - 短回答和长回答交替，避免单调
   - 讨论架构模式、设计原则、组件关系、独特技术点
   - 长播客分两段：上半场（文件结构+README，带 break 提示）+ 下半场（重要文件深入讨论，匿名嘉宾）

2. **架构图生成**（继承自 GitDiagram）: 三步 Prompt
   - Prompt 1: 文件树 + README → 架构解释
   - Prompt 2: 解释 + 文件树 → 组件→路径映射
   - Prompt 3: 解释 + 映射 → Mermaid.js 代码

3. **幻灯片生成**: 单 Prompt，要求生成 Marp 格式 Markdown

4. **图表修改**: 接受用户指令修改已有 Mermaid 图表，无效指令返回 `BAD_INSTRUCTIONS`

---

## 9. 环境变量

| 变量 | 必需 | 用途 |
|------|------|------|
| `SPEECH_KEY` | 是 | Azure Speech Service 密钥 |
| `SPEECH_REGION` | 是 | Azure Speech Service 区域 |
| `GEMINI_API_KEY` | 否 | Google Gemini API 密钥（免费方案） |
| `AZURE_OPENAI_API_KEY` | 是 | Azure OpenAI 密钥 |
| `AZURE_OPENAI_ENDPOINT` | 是 | Azure OpenAI 端点 |
| `AZURE_OPENAI_MODEL_NAME` | 否 | 模型名（默认 gpt-4o） |
| `ANTHROPIC_API_KEY` | 否 | Claude API 密钥（token 计数 + 图表修改） |
| `POSTGRES_URL` | 是 | PostgreSQL 连接字符串 |
| `NEXT_PUBLIC_API_DEV_URL` | 否 | 后端 API 地址（默认 https://api.gitpodcast.com） |
| `GITHUB_PAT` | 否 | GitHub PAT（提高 API 限额到 5000/hr） |
| `GITHUB_CLIENT_ID` | 否 | GitHub App Client ID |
| `GITHUB_PRIVATE_KEY` | 否 | GitHub App 私钥 |
| `GITHUB_INSTALLATION_ID` | 否 | GitHub App Installation ID |
| `CLERK_SECRET_KEY` | 否 | Clerk 后端密钥（认证验证） |
| `API_ANALYTICS_KEY` | 否 | API 分析密钥 |

---

## 10. 限流策略

| 端点 | 限制 | 备注 |
|------|------|------|
| `/generate` | 当前未启用（注释中: 1/min; 5/day） | 临时关闭以支持增长 |
| `/modify` | 2/min; 10/day | — |
| `/generate/cost` | 当前未启用 | — |
| Azure Speech | 50 万字符/月 | SSML → 语音 |
| Gemini | 15 调用/分钟 | Gemini Flash Exp 2.0 |

---

## 11. 特色设计

1. **URL 魔法**: 将 GitHub URL 中的 `hub` 替换为 `podcast` 即可访问播客（如 `github.com/user/repo` → `gitpodcast.com/user/repo`）

2. **双长度模式**: 未登录用户只能生成短播客（~5min），登录用户可选长播客（~10min），长播客通过并发调用两段 Prompt 实现

3. **SSML 重试 + 校验**: 生成 SSML 后自动校验 XML 合法性，不合法则重试最多 3 次；还自动清理非 `<voice>` 子节点

4. **WebVTT 字幕**: 基于语速估算（默认 135 WPM，实际根据总词数/音频时长计算）生成时间轴字幕

5. **90% 缓存命中率策略**: 前端请求时 90% 概率读缓存，10% 概率强制刷新，保证大部分用户秒级响应

6. **重要文件智能筛选**: 先用 OpenAI structured output 从文件树中选出 ≤10 个最关键文件，只读取这些文件的内容，控制 token 消耗

---

## 12. 已知局限与未来计划

**当前局限**:
- Azure Speech 有字符配额限制（50万/月）
- Claude 仅用于 token 计数和图表修改，存在对 Anthropic 的依赖
- 幻灯片 Markdown 验证始终返回 `True`（未实际校验）
- 部分 Prompt 模板与 GitDiagram 共用，存在架构图相关代码未完全清理

**未来计划**（来自 README）:
- 允许用户选择语音角色
- 移除对 Anthropic token 计数的依赖
- 允许自定义 Prompt
- API 正式开放（目前标注 WIP）
