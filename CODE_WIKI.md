# DeerFlow 代码百科全书

> **项目**: DeerFlow (Deep Exploration and Efficient Research Flow)  
> **版本**: 2.0  
> **技术栈**: Python 3.12+ / Next.js 16 / LangGraph / LangChain  
> **许可证**: MIT

---

## 目录

1. [项目概述](#1-项目概述)
2. [系统架构](#2-系统架构)
3. [项目结构](#3-项目结构)
4. [后端模块详解](#4-后端模块详解)
5. [前端模块详解](#5-前端模块详解)
6. [核心组件](#6-核心组件)
7. [技能系统](#7-技能系统)
8. [依赖关系](#8-依赖关系)
9. [配置说明](#9-配置说明)
10. [运行方式](#10-运行方式)

---

## 1. 项目概述

### 1.1 简介

DeerFlow 是一个开源的**超级 Agent Harness**，基于 LangGraph 构建，编排**子 Agent**、**内存**和**沙箱**来执行各种任务，并通过**可扩展技能**增强能力。

### 1.2 核心特性

| 特性 | 描述 |
|------|------|
| **多模型支持** | OpenAI、Anthropic、DeepSeek、vLLM、MiniMax 等 |
| **子 Agent** | 并行任务执行，结果汇总 |
| **沙箱执行** | Local/Docker/Kubernetes 三种模式 |
| **长期记忆** | 跨会话持久化用户偏好和上下文 |
| **技能系统** | 可扩展的领域特定工作流 |
| **IM 集成** | Telegram、Slack、飞书、微信、企业微信、钉钉 |
| **MCP 支持** | Model Context Protocol 服务器集成 |

### 1.3 技术栈

```
┌─────────────────────────────────────────────────────────────┐
│                        前端 (Frontend)                        │
│  Next.js 16 + React 19 + Tailwind CSS 4 + Shadcn UI        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Nginx (Port 2026)                       │
│              反向代理，统一入口                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Gateway API (Port 8001)                  │
│  FastAPI + LangGraph 嵌入式运行时 + SSE 流式响应             │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   沙箱层      │    │    模型层     │    │   技能层      │
│  Sandbox     │    │   Models      │    │   Skills      │
└───────────────┘    └───────────────┘    └───────────────┘
```

---

## 2. 系统架构

### 2.1 整体架构图

```
                           ┌──────────────────────────────────────┐
                           │          Nginx (Port 2026)            │
                           │      Unified Reverse Proxy           │
                           └───────┬──────────────────┬───────────┘
                                   │                  │
                   /api/langgraph/*│                  │/api/*
                                   ▼                  ▼
               ┌────────────────────────────────────────────────┐
               │              Gateway API (8001)                 │
               │                                               │
               │  Models  MCP  Skills  Memory  Uploads  Artifacts│
               │                                               │
               │  ┌────────────────────────────────────────────┐ │
               │  │          Lead Agent (主 Agent)             │ │
               │  │  Middleware Chain │ Tools │ Subagents      │ │
               │  └────────────────────────────────────────────┘ │
               └────────────────────────────────────────────────┘
```

### 2.2 请求路由

| 路径 | 目标 | 说明 |
|------|------|------|
| `/api/langgraph/*` | Gateway LangGraph 兼容 API | Agent 交互、线程、流式响应 |
| `/api/*` | Gateway REST API | 模型、MCP、技能、记忆、产物、上传 |
| `/*` | Frontend (3000) | Next.js Web 界面 |

### 2.3 Agent 运行时架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      make_lead_agent(config)                     │
└────────────────────────────────────┬────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Middleware Chain                          │
│  1. ThreadDataMiddleware      - 初始化工作目录                    │
│  2. UploadsMiddleware        - 注入上传文件                      │
│  3. SandboxMiddleware         - 获取沙箱环境                      │
│  4. SummarizationMiddleware  - 上下文压缩                        │
│  5. TitleMiddleware          - 自动生成标题                      │
│  6. TodoListMiddleware       - 任务跟踪                          │
│  7. ViewImageMiddleware      - 视觉模型支持                      │
│  8. ClarificationMiddleware  - 处理澄清请求                      │
└────────────────────────────────────┬────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                           Agent Core                             │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │    Model     │  │    Tools     │  │     System Prompt      │ │
│  │  (Factory)   │  │  (Sandbox+   │  │   (Skills + Memory)    │ │
│  │              │  │   MCP+builtin│  │                        │ │
│  └──────────────┘  └──────────────┘  └────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 项目结构

```
deer-flow/
├── backend/                    # Python 后端
│   ├── app/                    # 应用代码
│   │   ├── channels/          # IM 渠道集成
│   │   │   ├── base.py        # 渠道基类
│   │   │   ├── service.py     # 渠道服务
│   │   │   ├── manager.py     # 渠道管理
│   │   │   ├── commands.py    # 渠道命令
│   │   │   ├── message_bus.py # 消息总线
│   │   │   ├── store.py       # 状态存储
│   │   │   ├── dingtalk.py    # 钉钉
│   │   │   ├── discord.py     # Discord
│   │   │   ├── feishu.py      # 飞书
│   │   │   ├── slack.py       # Slack
│   │   │   ├── telegram.py    # Telegram
│   │   │   ├── wechat.py      # 微信
│   │   │   └── wecom.py       # 企业微信
│   │   └── gateway/           # FastAPI 网关
│   │       ├── app.py         # 应用入口
│   │       ├── deps.py        # 依赖注入
│   │       ├── config.py      # 网关配置
│   │       ├── auth_middleware.py    # 认证中间件
│   │       ├── csrf_middleware.py    # CSRF 中间件
│   │       ├── authz.py       # 授权
│   │       ├── internal_auth.py      # 内部认证
│   │       ├── langgraph_auth.py     # LangGraph 认证
│   │       ├── pagination.py  # 分页
│   │       ├── path_utils.py  # 路径工具
│   │       ├── services.py     # 服务层
│   │       ├── utils.py       # 工具函数
│   │       ├── auth/          # 认证模块
│   │       │   ├── providers.py        # 认证提供者
│   │       │   ├── models.py           # 认证模型
│   │       │   ├── jwt.py              # JWT 处理
│   │       │   ├── password.py         # 密码处理
│   │       │   ├── errors.py           # 错误定义
│   │       │   ├── credential_file.py  # 凭证文件
│   │       │   ├── local_provider.py   # 本地认证
│   │       │   ├── reset_admin.py     # 重置管理员
│   │       │   ├── config.py           # 认证配置
│   │       │   └── repositories/       # 认证仓库
│   │       └── routers/        # 路由模块
│   │           ├── agents.py           # Agent 管理
│   │           ├── artifacts.py        # 产物服务
│   │           ├── assistants_compat.py # 助手兼容
│   │           ├── auth.py             # 认证
│   │           ├── channels.py         # 渠道
│   │           ├── feedback.py         # 反馈
│   │           ├── mcp.py              # MCP 配置
│   │           ├── memory.py           # 记忆
│   │           ├── models.py           # 模型
│   │           ├── runs.py             # 运行
│   │           ├── skills.py           # 技能
│   │           ├── suggestions.py      # 建议
│   │           ├── thread_runs.py      # 线程运行
│   │           ├── threads.py          # 线程
│   │           └── uploads.py          # 上传
│   ├── packages/               # 核心包 (harness)
│   │   └── harness/
│   │       └── deerflow/       # DeerFlow 核心
│   │           ├── agents/     # Agent 系统
│   │           ├── config/     # 配置系统
│   │           ├── models/     # 模型工厂
│   │           ├── sandbox/    # 沙箱系统
│   │           ├── skills/     # 技能系统
│   │           ├── tools/      # 工具集
│   │           ├── mcp/        # MCP 集成
│   │           ├── community/  # 社区工具
│   │           ├── persistence/# 持久化
│   │           ├── reflection/ # 动态加载
│   │           ├── runtime/    # 运行时
│   │           ├── tracing/    # 链路追踪
│   │           └── utils/     # 工具函数
│   ├── docs/                   # 文档
│   ├── tests/                  # 测试
│   ├── pyproject.toml          # Python 依赖
│   ├── Makefile               # 构建命令
│   └── Dockerfile             # 容器构建
├── frontend/                  # Next.js 前端
│   ├── src/
│   │   ├── app/              # App Router 页面
│   │   ├── components/       # React 组件
│   │   │   ├── ui/          # UI 基础组件
│   │   │   ├── workspace/    # 工作区组件
│   │   │   ├── landing/      # 着陆页组件
│   │   │   └── ai-elements/  # AI 元素组件
│   │   ├── core/            # 核心业务逻辑
│   │   │   ├── api/         # API 客户端
│   │   │   ├── artifacts/   # 产物管理
│   │   │   ├── auth/        # 认证
│   │   │   ├── config/      # 配置
│   │   │   ├── i18n/        # 国际化
│   │   │   ├── mcp/         # MCP
│   │   │   ├── memory/      # 记忆
│   │   │   ├── messages/    # 消息
│   │   │   ├── models/      # 模型
│   │   │   ├── settings/    # 设置
│   │   │   ├── skills/      # 技能
│   │   │   ├── threads/     # 线程
│   │   │   ├── todos/       # 待办
│   │   │   ├── tools/       # 工具
│   │   │   └── uploads/     # 上传
│   │   ├── hooks/           # 自定义 Hooks
│   │   ├── lib/             # 共享库
│   │   └── styles/          # 样式
│   ├── package.json         # Node 依赖
│   └── Makefile            # 构建命令
├── skills/                   # 技能目录
│   └── public/              # 内置技能
│       ├── academic-paper-review/
│       ├── bootstrap/
│       ├── chart-visualization/
│       ├── claude-to-deerflow/
│       ├── code-documentation/
│       ├── consulting-analysis/
│       ├── data-analysis/
│       ├── deep-research/
│       ├── find-skills/
│       ├── frontend-design/
│       ├── github-deep-research/
│       ├── image-generation/
│       ├── newsletter-generation/
│       ├── podcast-generation/
│       ├── ppt-generation/
│       ├── skill-creator/
│       ├── surprise-me/
│       ├── systematic-literature-review/
│       ├── vercel-deploy-claimable/
│       ├── video-generation/
│       └── web-design-guidelines/
├── docker/                   # Docker 配置
│   ├── nginx/               # Nginx 配置
│   ├── provisioner/         # K8s 配置器
│   └── docker-compose*.yaml # 编排文件
├── scripts/                  # 脚本
│   └── wizard/              # 设置向导
├── config.example.yaml       # 配置示例
└── CODE_WIKI.md            # 本文档
```

---

## 4. 后端模块详解

### 4.1 Gateway API (`app/gateway/`)

#### 4.1.1 应用入口 (`app.py`)

```python
# 核心函数
def create_app() -> FastAPI
def lifespan(app: FastAPI) -> AsyncGenerator[None, None]
```

**职责**:
- 创建 FastAPI 应用
- 配置中间件 (Auth、CSRF、CORS)
- 注册路由
- 管理应用生命周期

**中间件顺序**:
1. `AuthMiddleware` - 认证拦截
2. `CSRFMiddleware` - CSRF 防护
3. `CORSMiddleware` - 跨域资源共享

#### 4.1.2 路由模块 (`routers/`)

| 路由文件 | 路径 | 功能 |
|----------|------|------|
| `models.py` | `/api/models` | 模型列表与配置 |
| `mcp.py` | `/api/mcp` | MCP 服务器配置 |
| `skills.py` | `/api/skills` | 技能管理 |
| `memory.py` | `/api/memory` | 记忆系统 |
| `uploads.py` | `/api/threads/{id}/uploads` | 文件上传 |
| `artifacts.py` | `/api/threads/{id}/artifacts` | 产物服务 |
| `threads.py` | `/api/threads/{id}` | 线程清理 |
| `agents.py` | `/api/agents` | 自定义 Agent |
| `runs.py` | `/api/runs` | 运行管理 |
| `thread_runs.py` | `/api/threads/{id}/runs` | 线程运行 |
| `suggestions.py` | `/api/threads/{id}/suggestions` | 建议生成 |
| `channels.py` | `/api/channels` | IM 渠道 |
| `feedback.py` | `/api/threads/{id}/runs/{run_id}/feedback` | 反馈 |
| `auth.py` | `/api/v1/auth` | 认证 |
| `assistants_compat.py` | `/api/assistants` | 助手兼容 |

### 4.2 IM 渠道 (`app/channels/`)

#### 4.2.1 支持的渠道

| 渠道 | 传输方式 | 难度 |
|------|----------|------|
| Telegram | Bot API (长轮询) | 简单 |
| Slack | Socket Mode | 中等 |
| 飞书/ Lark | WebSocket | 中等 |
| 微信 | 腾讯 iLink (长轮询) | 中等 |
| 企业微信 | WebSocket | 中等 |
| 钉钉 | Stream Push (WebSocket) | 中等 |

#### 4.2.2 核心类

```python
# 基类
class Channel(ABC)
    - connect() -> None
    - disconnect() -> None
    - handle_message(message: ChannelMessage) -> None

# 服务
class ChannelService
    - start_channel_service(config) -> ChannelService
    - stop_channel_service() -> None
    - get_status() -> str
```

### 4.3 沙箱系统 (`packages/harness/deerflow/sandbox/`)

#### 4.3.1 抽象接口

```python
class SandboxProvider(ABC):
    """沙箱提供者工厂"""
    def acquire(self, thread_id: str) -> Sandbox
    def release(self, sandbox: Sandbox) -> None

class Sandbox(ABC):
    """沙箱实例"""
    def execute_command(self, command: str, timeout: int = 30) -> str
    def read_file(self, path: str) -> str
    def write_file(self, path: str, content: str) -> None
    def list_dir(self, path: str) -> list[str]
```

#### 4.3.2 提供者实现

| 提供者 | 类 | 用途 |
|--------|-----|------|
| Local | `LocalSandboxProvider` | 开发环境，直接执行 |
| Docker | `AioSandboxProvider` | 隔离容器 |
| K8s | `AioSandboxProvider` + Provisioner | Kubernetes Pod |

#### 4.3.3 虚拟路径映射

| 虚拟路径 | 物理路径 |
|----------|----------|
| `/mnt/user-data/workspace` | `.deer-flow/threads/{thread_id}/user-data/workspace` |
| `/mnt/user-data/uploads` | `.deer-flow/threads/{thread_id}/user-data/uploads` |
| `/mnt/user-data/outputs` | `.deer-flow/threads/{thread_id}/user-data/outputs` |
| `/mnt/skills` | `skills/` |

### 4.4 模型工厂 (`packages/harness/deerflow/models/`)

```python
def create_chat_model(
    name: str,
    thinking_enabled: bool = False,
    attach_tracing: bool = True
) -> BaseChatModel
```

**支持的提供者**:

| 提供者 | 类路径 | 说明 |
|--------|--------|------|
| OpenAI | `langchain_openai:ChatOpenAI` | GPT 系列 |
| Anthropic | `langchain_anthropic:ChatAnthropic` | Claude 系列 |
| DeepSeek | `langchain_deepseek:ChatDeepSeek` | DeepSeek 系列 |
| vLLM | `deerflow.models.vllm_provider:VllmChatModel` | vLLM 兼容 |
| MiniMax | `deerflow.models.patched_minimax:PatchedChatMiniMax` | MiniMax 系列 |
| 本地 Ollama | `langchain_ollama:ChatOllama` | Ollama 本地模型 |

---

## 5. 前端模块详解

### 5.1 目录结构

```
frontend/src/
├── app/                      # Next.js App Router
│   ├── (auth)/              # 认证页面
│   ├── blog/                # 博客
│   ├── workspace/           # 主工作区
│   │   ├── page.tsx        # 工作区首页
│   │   └── workspace-content.tsx
│   ├── layout.tsx          # 根布局
│   └── page.tsx            # 根页面 (着陆页)
├── components/              # 组件库
│   ├── ui/                 # 基础 UI 组件 (Shadcn)
│   │   ├── button.tsx
│   │   ├── dialog.tsx
│   │   ├── dropdown-menu.tsx
│   │   ├── input.tsx
│   │   ├── select.tsx
│   │   └── ...
│   ├── workspace/          # 工作区组件
│   │   ├── workspace-container.tsx
│   │   ├── workspace-header.tsx
│   │   ├── workspace-sidebar.tsx
│   │   ├── input-box.tsx
│   │   ├── thread-title.tsx
│   │   └── ...
│   ├── landing/            # 着陆页组件
│   │   ├── hero.tsx
│   │   ├── header.tsx
│   │   ├── footer.tsx
│   │   └── ...
│   └── ai-elements/        # AI 元素
│       ├── message.tsx
│       ├── artifact.tsx
│       ├── code-block.tsx
│       ├── plan.tsx
│       └── ...
├── core/                    # 核心业务逻辑
│   ├── api/                # API 客户端
│   │   ├── api-client.ts   # 主客户端
│   │   ├── fetcher.ts      # 数据获取
│   │   └── stream-mode.ts  # 流式模式
│   ├── threads/            # 线程管理
│   │   ├── api.ts          # 线程 API
│   │   ├── hooks.ts        # React Hooks
│   │   └── types.ts        # 类型定义
│   ├── skills/             # 技能管理
│   ├── memory/             # 记忆管理
│   ├── artifacts/          # 产物管理
│   ├── models/             # 模型管理
│   └── uploads/            # 上传管理
├── hooks/                  # 自定义 Hooks
│   ├── use-global-shortcuts.ts
│   └── use-mobile.ts
└── lib/                    # 共享库
    ├── utils.ts            # 工具函数
    └── ime.ts              # IME 处理
```

### 5.2 页面结构

| 路径 | 组件 | 说明 |
|------|------|------|
| `/` | `page.tsx` | 着陆页 |
| `/workspace` | `workspace/page.tsx` | 工作区首页 |
| `/workspace/{thread_id}` | `workspace/page.tsx` | 特定线程 |

### 5.3 核心 API 客户端 (`core/api/api-client.ts`)

```typescript
// LangGraph SDK 封装
class DeerFlowApiClient {
  // 线程管理
  createThread(): Promise<Thread>
  getThread(threadId: string): Promise<Thread>
  listThreads(limit?: number): Promise<Thread[]>

  // 消息发送
  streamMessage(threadId: string, input: MessageInput): AsyncGenerator<StreamEvent>

  // 文件上传
  uploadFiles(threadId: string, files: File[]): Promise<UploadResponse>

  // 技能管理
  listSkills(): Promise<Skill[]>
  updateSkill(name: string, enabled: boolean): Promise<Skill>
}
```

---

## 6. 核心组件

### 6.1 ThreadState

```python
class ThreadState(AgentState):
    """扩展 LangGraph AgentState"""
    messages: list[BaseMessage]        # 消息历史
    sandbox: dict                      # 沙箱环境信息
    artifacts: list[str]               # 产物文件路径
    thread_data: dict                  # {workspace, uploads, outputs}
    title: str | None                  # 自动生成标题
    todos: list[dict]                   # 计划模式任务
    viewed_images: dict               # 视觉模型图片数据
```

### 6.2 Middleware Chain

| # | 中间件 | 职责 |
|---|--------|------|
| 1 | `ThreadDataMiddleware` | 创建线程隔离目录 |
| 2 | `UploadsMiddleware` | 注入上传文件 |
| 3 | `SandboxMiddleware` | 获取沙箱环境 |
| 4 | `SummarizationMiddleware` | 上下文压缩 |
| 5 | `TitleMiddleware` | 生成标题 |
| 6 | `TodoListMiddleware` | 任务跟踪 |
| 7 | `ViewImageMiddleware` | 视觉处理 |
| 8 | `ClarificationMiddleware` | 澄清请求处理 |

### 6.3 工具生态

| 类别 | 工具 |
|------|------|
| **沙箱** | `bash`, `ls`, `read_file`, `write_file`, `str_replace` |
| **内置** | `present_files`, `ask_clarification`, `view_image`, `task` |
| **社区** | Tavily (搜索), Jina AI (抓取), Firecrawl, DuckDuckGo |
| **MCP** | 任意 MCP 服务器 |
| **技能** | 领域特定工作流 |

### 6.4 DeerFlowClient (嵌入式 Python 客户端)

```python
from deerflow.client import DeerFlowClient

client = DeerFlowClient()

# 聊天
response = client.chat("Analyze this paper", thread_id="my-thread")

# 流式响应
for event in client.stream("hello"):
    if event.type == "messages-tuple":
        print(event.data)

# 配置查询
models = client.list_models()
skills = client.list_skills()
```

---

## 7. 技能系统

### 7.1 技能结构

```
skills/public/<skill-name>/
├── SKILL.md              # 技能定义 (必需)
├── references/          # 参考文档
├── templates/           # 模板文件
├── scripts/             # 辅助脚本
└── assets/              # 静态资源
```

### 7.2 SKILL.md 格式

```markdown
---
name: skill-name
description: 技能描述
license: MIT
allowed-tools:
  - read_file
  - write_file
  - bash
---

# 技能指导

技能的具体指令和最佳实践...
```

### 7.3 内置技能

| 技能 | 功能 |
|------|------|
| `deep-research` | 深度研究 |
| `systematic-literature-review` | 系统文献综述 |
| `academic-paper-review` | 学术论文评审 |
| `data-analysis` | 数据分析 |
| `chart-visualization` | 图表可视化 |
| `ppt-generation` | PPT 生成 |
| `image-generation` | 图片生成 |
| `video-generation` | 视频生成 |
| `podcast-generation` | 播客生成 |
| `newsletter-generation` | 新闻订阅生成 |
| `frontend-design` | 前端设计 |
| `web-design-guidelines` | Web 设计指南 |
| `skill-creator` | 技能创建器 |
| `code-documentation` | 代码文档 |
| `consulting-analysis` | 咨询分析 |
| `github-deep-research` | GitHub 深度研究 |

---

## 8. 依赖关系

### 8.1 后端依赖 (`backend/pyproject.toml`)

```toml
[project]
requires-python = ">=3.12"
dependencies = [
    "deerflow-harness",           # 核心包
    "fastapi>=0.115.0",           # Web 框架
    "httpx>=0.28.0",              # HTTP 客户端
    "python-multipart>=0.0.27",   # 文件上传
    "sse-starlette>=2.1.0",       # SSE 支持
    "uvicorn[standard]>=0.34.0",  # ASGI 服务器
    "lark-oapi>=1.4.0",           # 飞书 SDK
    "slack-sdk>=3.33.0",          # Slack SDK
    "python-telegram-bot>=21.0", # Telegram SDK
    "langgraph-sdk>=0.1.51",      # LangGraph SDK
    "markdown-to-mrkdwn>=0.3.1",  # Markdown 转换
    "wecom-aibot-python-sdk>=0.1.6",  # 企业微信
    "dingtalk-stream>=0.24.3",    # 钉钉
    "bcrypt>=4.0.0",             # 密码加密
    "pyjwt>=2.9.0",              # JWT
    "email-validator>=2.0.0",    # 邮箱验证
]

[project.optional-dependencies]
postgres = ["deerflow-harness[postgres]"]
discord = ["discord.py>=2.7.0"]
```

### 8.2 前端依赖 (`frontend/package.json`)

```json
{
  "dependencies": {
    "next": "^16.2.6",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@langchain/langgraph-sdk": "^1.5.3",
    "ai": "^6.0.33",
    "@tanstack/react-query": "^5.90.17",
    "tailwindcss": "^4.0.15",
    "shadcn-ui": "latest",
    "@radix-ui/react-*": "latest",
    "lucide-react": "^0.562.0",
    "clsx": "^2.1.1",
    "zod": "^3.24.2"
  }
}
```

### 8.3 核心依赖图

```
┌─────────────────────────────────────────────────────────────┐
│                     DeerFlow Application                     │
└─────────────────────────────┬───────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   Frontend    │    │    Gateway    │    │   Harness     │
│  (Next.js)    │    │   (FastAPI)   │    │  (LangGraph)  │
└───────────────┘    └───────┬───────┘    └───────┬───────┘
                             │                     │
                             ▼                     ▼
                    ┌───────────────┐    ┌───────────────┐
                    │  LangGraph    │    │   LangChain   │
                    │    SDK        │    │   Abstractions│
                    └───────────────┘    └───────────────┘
```

---

## 9. 配置说明

### 9.1 配置文件

| 文件 | 用途 |
|------|------|
| `config.yaml` | 主配置 (模型、工具、沙箱、技能) |
| `extensions_config.json` | MCP 服务器和技能状态 |
| `.env` | 环境变量 (API 密钥) |

### 9.2 config.yaml 结构

```yaml
# 模型配置
models:
  - name: gpt-4o
    display_name: GPT-4o
    use: langchain_openai:ChatOpenAI
    model: gpt-4o
    api_key: $OPENAI_API_KEY
    supports_thinking: false
    supports_vision: true

# 工具配置
tools:
  - name: web_search
    group: web
    use: deerflow.community.ddg_search.tools:web_search_tool
  - name: read_file
    group: file:read
    use: deerflow.sandbox.tools:read_file_tool

# 沙箱配置
sandbox:
  use: deerflow.sandbox.local:LocalSandboxProvider
  allow_host_bash: false

# 技能配置
skills:
  container_path: /mnt/skills

# 记忆配置
memory:
  enabled: true
  storage_path: memory.json
  debounce_seconds: 30
  max_facts: 100

# 数据库配置
database:
  backend: sqlite
  sqlite_dir: .deer-flow/data

# IM 渠道配置
channels:
  telegram:
    enabled: true
    bot_token: $TELEGRAM_BOT_TOKEN
```

### 9.3 extensions_config.json 结构

```json
{
  "mcpServers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_TOKEN": "$GITHUB_TOKEN"}
    }
  },
  "skills": {
    "pdf-processing": {"enabled": true}
  }
}
```

---

## 10. 运行方式

### 10.1 快速启动

```bash
# 1. 克隆仓库
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow

# 2. 运行设置向导
make setup

# 3. 启动开发服务
make dev

# 访问 http://localhost:2026
```

### 10.2 Docker 部署 (推荐)

```bash
# 开发模式
make docker-init    # 拉取沙箱镜像
make docker-start   # 启动服务

# 生产模式
make up             # 构建并启动
make down           # 停止服务
```

### 10.3 本地开发

```bash
# 后端
cd backend
make install        # 安装依赖
make dev           # 运行网关 (端口 8001)

# 前端
cd frontend
pnpm install       # 安装依赖
pnpm dev          # 运行开发服务器 (端口 3000)
```

### 10.4 环境变量

```bash
# API 密钥
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
DEEPSEEK_API_KEY=...

# 可选服务
TAVILY_API_KEY=...
SERPER_API_KEY=...
FIRECRAWL_API_KEY=...

# 追踪
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=lsv2_pt_...

# IM 渠道
TELEGRAM_BOT_TOKEN=...
SLACK_BOT_TOKEN=...
FEISHU_APP_ID=...
```

---

## 附录

### A. 关键类与函数索引

| 模块 | 类/函数 | 说明 |
|------|---------|------|
| `app.gateway.app` | `create_app()` | 创建 FastAPI 应用 |
| `app.gateway.app` | `lifespan()` | 应用生命周期管理 |
| `deerflow.agents.lead_agent.agent` | `make_lead_agent()` | 创建主 Agent |
| `deerflow.sandbox.local` | `LocalSandboxProvider` | 本地沙箱提供者 |
| `deerflow.models.factory` | `create_chat_model()` | 模型工厂 |
| `deerflow.client` | `DeerFlowClient` | Python 嵌入式客户端 |
| `deerflow.skills.loader` | `SkillLoader` | 技能加载器 |

### B. API 端点一览

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/models` | 获取模型列表 |
| GET | `/api/skills` | 获取技能列表 |
| PUT | `/api/skills/{name}` | 更新技能 |
| POST | `/api/skills/install` | 安装技能 |
| GET | `/api/memory` | 获取记忆数据 |
| POST | `/api/memory/reload` | 重载记忆 |
| GET | `/api/mcp/config` | 获取 MCP 配置 |
| PUT | `/api/mcp/config` | 更新 MCP 配置 |
| POST | `/api/threads/{id}/uploads` | 上传文件 |
| GET | `/api/threads/{id}/artifacts/{path}` | 获取产物 |
| POST | `/api/runs` | 创建运行 (无状态) |
| POST | `/api/threads/{id}/runs` | 创建线程运行 |
| GET | `/health` | 健康检查 |

### C. 端口映射

| 端口 | 服务 | 说明 |
|------|------|------|
| 2026 | Nginx | 统一入口 |
| 8001 | Gateway API | 后端 API |
| 3000 | Frontend Dev | 前端开发服务器 |

---

*本文档由代码分析自动生成，如需更新请提交 PR。*
