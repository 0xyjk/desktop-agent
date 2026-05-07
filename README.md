# Desktop Agent

一个完整但保持 minimal 的桌面端 general agent 模板。项目不是只做聊天 UI，而是把一个通用 Agent 所需的核心能力都串成了可运行的最小闭环：

| # | 能力 | 作用 |
|---|------|------|
| 1 | 基础 Chatbot | 与 LLM 文本对话 |
| 2 | Tool Calling | LLM 执行函数 |
| 3 | MCP | 接入外部服务 |
| 4 | Skill 系统 | 可插拔能力 |
| 5 | 代码执行 | 沙箱中运行代码 |
| 6 | 记忆系统 | 持久化上下文 |

核心目标：用 Electron + React 做桌面壳，用本地 Hono server 承载 Agent runtime，用 Vercel AI SDK 的 `ToolLoopAgent` 统一 LLM、工具、MCP、Skills、代码执行和长期记忆。

## 实际架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Electron Desktop                           │
│                                                                     │
│  ┌────────────────────────────┐        ┌──────────────────────────┐ │
│  │ Main Process               │        │ Renderer Process          │ │
│  │                            │        │ React + ai-elements       │ │
│  │  ┌──────────────────────┐  │ HTTP   │                          │ │
│  │  │ Hono API Server       │◄────────►│ useChat + ChatWindow      │ │
│  │  │ localhost:3315        │  │        │ MCP / Memory / Skill UI   │ │
│  │  └──────────┬───────────┘  │        └──────────────────────────┘ │
│  │             │              │                                      │
│  │  ┌──────────▼───────────┐  │                                      │
│  │  │ ToolLoopAgent         │  │                                      │
│  │  │ AI SDK runtime        │  │                                      │
│  │  └──────────┬───────────┘  │                                      │
│  │             │              │                                      │
│  │  ┌──────────▼────────────────────────────────────────────────┐    │
│  │  │ Tool Set                                                   │    │
│  │  │ - shell: 执行本机命令                                       │    │
│  │  │ - execute_python: 持久 IPython kernel                       │    │
│  │  │ - use_skill: 按需加载 ~/.agents/skills/*/SKILL.md           │    │
│  │  │ - MCP tools: 动态聚合外部 MCP server 暴露的工具             │    │
│  │  └──────────┬────────────────────────────────────────────────┘    │
│  │             │              │                                      │
│  │  ┌──────────▼───────────┐  │                                      │
│  │  │ Persistent Context    │  │                                      │
│  │  │ ~/.agents/memory.md   │  │                                      │
│  │  │ userData/mcp.json     │  │                                      │
│  │  └──────────────────────┘  │                                      │
│  └────────────────────────────┘                                      │
└─────────────────────────┬───────────────────────────────────────────┘
                          │ OpenAI-compatible HTTP API
                          ▼
                 ┌────────────────────┐
                 │ LLM Provider        │
                 │ OpenAI / DeepSeek   │
                 │ Ollama / compatible │
                 └────────────────────┘
```

### 运行链路

1. Electron 主进程启动本地 Hono API server，默认监听 `http://localhost:3315`。
2. Renderer 里的 `ChatWindow` 使用 `useChat` 和 `DefaultChatTransport` 调用 `/api/chat`。
3. `/api/chat` 把前端消息交给 `ToolLoopAgent`，由 Agent 自主决定是否调用工具。
4. `ToolLoopAgent` 的 system instructions 由三部分组成：基础助手指令、`~/.agents/memory.md`、当前可用 Skills 列表。
5. Agent 可调用内置工具、动态 MCP 工具，或通过 `/skill-name` / `use_skill` 加载 skill 指令。
6. 工具执行结果以 AI SDK UI message parts 返回，前端按工具类型渲染终端输出、Python 图表或通用 tool panel。

## 能力拆解

### 1. 基础 Chatbot

- 前端：`src/renderer/src/components/ChatWindow.tsx`
- 后端：`src/main/server.ts`
- 使用 AI SDK 的流式 UI message 协议，支持边生成边渲染。

### 2. Tool Calling

- Agent runtime：`ToolLoopAgent`
- 内置工具：
  - `shell`：执行本机 shell 命令，带超时和输出截断
  - `execute_python`：执行 Python 代码并返回文本和图片
  - `use_skill`：加载指定 Skill 的完整说明
- 工具统一注册在 `src/main/server.ts` 的 `rebuildAgent()` 中。

### 3. MCP

- 实现位置：`src/main/mcp.ts`
- 前端入口：`src/renderer/src/components/MCPSettings.tsx`
- 支持通过 UI 添加、删除、重连 HTTP MCP server。
- MCP 配置持久化在 Electron `userData` 目录下的 `mcp.json`。
- 每次 MCP 配置变化后会重新构建 Agent，把最新 MCP tools 合并进工具集。

### 4. Skill 系统

- 实现位置：`src/main/skills.ts`
- 前端入口：`src/renderer/src/components/SkillPicker.tsx`
- Skill 存放在 `~/.agents/skills/<skill-name>/SKILL.md`。
- 启动时扫描 Skill metadata，并把可用 Skill 列表注入 system prompt。
- 用户可以输入 `/skill-name` 直接把 Skill 内容注入当前请求，也可以让 Agent 调用 `use_skill` 按需加载。

### 5. 代码执行

- Shell 工具：`src/main/tools/shell.ts`
- Python kernel：`src/main/tools/python_kernel.ts`
- Python 服务脚本：`resources/kernel_server.py`
- Python 代码运行在独立的持久 IPython kernel 中，变量、import 和上下文会在同一会话内保留。
- Python 执行结果支持 stdout、stderr、异常 traceback，以及 matplotlib 生成的 PNG 图片。

### 6. 记忆系统

- 实现位置：`src/main/server.ts` 和 `src/renderer/src/components/MemoryEditor.tsx`
- 记忆文件：`~/.agents/memory.md`
- 用户在 UI 中编辑 Markdown，全局记忆会被注入 Agent system instructions。
- 保存记忆后会立即 `rebuildAgent()`，后续对话使用最新上下文。

## 项目结构

```
src/
├── main/
│   ├── index.ts              # Electron 主进程入口，启动窗口和本地 API server
│   ├── server.ts             # Hono API + ToolLoopAgent 组装
│   ├── mcp.ts                # MCP server 配置、连接和工具聚合
│   ├── skills.ts             # Skill 扫描、提示注入和 use_skill 工具
│   └── tools/
│       ├── shell.ts          # shell tool
│       └── python_kernel.ts  # Python execution tool
├── preload/
│   └── index.ts              # Electron preload
└── renderer/
    └── src/
        ├── App.tsx
        ├── components/
        │   ├── ChatWindow.tsx
        │   ├── MCPSettings.tsx
        │   ├── MemoryEditor.tsx
        │   ├── SkillPicker.tsx
        │   ├── PythonResult.tsx
        │   ├── ai-elements/
        │   └── ui/
        └── main.tsx

resources/
└── kernel_server.py          # 持久 IPython kernel JSON 协议服务

docs/chapters/                # 6 个能力模块的逐章实现说明
```

## 技术栈

| 层级 | 技术 |
|------|------|
| 桌面框架 | Electron + electron-vite |
| 前端 | React + TypeScript + Tailwind CSS + shadcn/ui + ai-elements |
| Agent runtime | Vercel AI SDK `ToolLoopAgent` |
| API server | Hono + `@hono/node-server` |
| LLM 接入 | OpenAI-compatible provider |
| 外部工具 | `@ai-sdk/mcp` |
| 代码执行 | Shell + persistent IPython kernel |
| 包管理器 | pnpm |

## 快速开始

```bash
pnpm install
pnpm dev
```

如果要使用 Python 代码执行能力，需要本机可运行 `python3`，并安装 Jupyter kernel 相关依赖：

```bash
python3 -m pip install jupyter_client ipykernel matplotlib
```

## 配置 LLM

在项目根目录创建 `.env`：

```env
LLM_API_KEY=your-api-key-here
LLM_BASE_URL=https://api.openai.com/v1
LLM_MODEL=gpt-4o-mini
```

`LLM_BASE_URL` 支持任何 OpenAI-compatible provider，例如 OpenAI、DeepSeek、本地 Ollama 兼容接口等。

## 本地数据

| 数据 | 位置 | 说明 |
|------|------|------|
| 全局记忆 | `~/.agents/memory.md` | 每次构建 Agent 时注入 system instructions |
| Skills | `~/.agents/skills/<name>/SKILL.md` | 启动时扫描，可通过 `/skill-name` 调用 |
| MCP 配置 | Electron `userData/mcp.json` | 保存 MCP server URL 和 headers |

## API

| Method | Path | 说明 |
|--------|------|------|
| `POST` | `/api/chat` | Agent 对话入口 |
| `GET` | `/api/skills` | 获取已安装 Skills |
| `GET` | `/api/memory` | 读取全局记忆 |
| `PUT` | `/api/memory` | 保存全局记忆并重建 Agent |
| `GET` | `/api/mcp/servers` | 查看 MCP server 状态 |
| `POST` | `/api/mcp/servers` | 添加 MCP server |
| `DELETE` | `/api/mcp/servers/:name` | 删除 MCP server |
| `POST` | `/api/mcp/servers/:name/reconnect` | 重连 MCP server |

## 教程章节

这个仓库也可以作为从零搭建 Agent 的教程代码。每章对应一个能力模块：

| 篇 | 标题 | 核心功能 |
|----|------|---------|
| 1 | 基础 Chatbot | 与 LLM 文本对话 |
| 2 | Tool Calling | LLM 执行函数 |
| 3 | MCP | 接入外部服务 |
| 4 | Skill 系统 | 可插拔能力 |
| 5 | 代码执行 | 沙箱中运行代码 |
| 6 | 记忆系统 | 持久化上下文 |

实现指南位于 [`docs/chapters/`](docs/chapters/)。

## 开发命令

```bash
pnpm install          # 安装依赖
pnpm dev              # 启动开发环境
pnpm typecheck        # TypeScript 检查
pnpm lint             # ESLint 检查
pnpm build            # 生产构建
```

## 开发约定

- 语言使用 TypeScript，保持 strict 类型约束。
- UI 文案使用中文。
- 不硬编码 API Key，统一通过 `.env` 或用户配置读取。
- 聊天和 Agent 流式输出优先使用 AI SDK 的 stream / UI message 机制。
- 新增工具放在 `src/main/tools/`，再在 `rebuildAgent()` 中注册。
- 新增前端 Agent 输出形态时，优先复用 `ai-elements` 组件。
