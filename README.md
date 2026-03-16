# Cloud Points - AI Music DJ Assistant

> 基于 LangGraph Agent + Apple Music 的智能音乐 DJ 助手，支持自然语言对话式音乐播放控制。

## 项目架构

```
cloudpoints/
├── apps/
│   ├── backend/          # Python FastAPI + LangGraph Agent
│   ├── web/              # React + TypeScript Web Player
│   ├── ios/              # SwiftUI 原生 iOS App
│   ├── gateway/          # Cloudflare Worker 网关层
│   ├── backend-worker/   # Cloudflare Durable Objects 容器编排
│   ├── admin/            # 管理后台
│   └── landing/          # 落地页
├── packages/
│   └── auth/             # 共享认证模块 (Better Auth + Drizzle ORM)
└── supabase/             # 数据库迁移与配置
```

## 技术栈总览

| 层级 | 技术选型 |
|------|----------|
| **前端** | React 18, TypeScript, Vite, Zustand, Tailwind CSS |
| **后端** | Python 3.11+, FastAPI, LangGraph, LangChain |
| **LLM** | Claude Sonnet 4.6 / GPT-5-mini (可切换) |
| **音乐** | Apple Music API, MusicKit JS SDK |
| **认证** | Better Auth, Apple Sign-In, Magic Link |
| **数据库** | PostgreSQL (Supabase) + Cloudflare D1 (SQLite) |
| **基础设施** | Cloudflare Workers, Durable Objects, Pages |
| **移动端** | SwiftUI (iOS) |

---

## 模块技术亮点

### 1. Backend — LangGraph Agent 引擎

- **LangGraph 多工具 Agent**: 基于 `create_react_agent` 构建，集成 7 个音乐控制工具（搜索、播放、跳过、队列管理等），支持 Claude web_search 原生工具
- **SSE 实时流式响应**: 通过 `astream()` 的 `messages + custom` 双模式流，实时推送 text / thinking / tool_start / tool_end / action 五类事件
- **Fire-and-Forget Action 模式**: 工具执行结果通过 `_emit_action()` 以 SSE 事件直接推送前端执行 MusicKit 操作，避免 LangGraph interrupt 并发 bug
- **ContextVar 线程隔离**: 使用 Python `contextvars.ContextVar` 实现请求级别的 session 上下文、数据库连接、用户身份隔离，无需层层传参
- **PostgreSQL 异步检查点**: 基于 `AsyncPostgresSaver` + `psycopg` 连接池实现 LangGraph 对话状态持久化，支持多轮对话恢复
- **连接池预热**: 启动时通过 `warmup_pool()` 预创建数据库连接，消除首次请求的 TCP+SSL 握手延迟
- **Apple Music JWT 签名**: ES256 算法动态生成开发者令牌，内存缓存 + 过期检测，最长 6 个月 TTL

### 2. Web — React 实时音乐播放器

- **Zustand 集中状态管理**: 单一 `chatStore` 管理消息、会话、播放列表状态，支持 SSE 流式解析与增量更新
- **SSE 流式消息解析**: 自定义 `handleStreamingResponse` 解析多类型事件流，实时构建多 Part 消息（text / thinking / tool_call）
- **MusicKit JS 深度集成**: 封装 `useAppleMusic` Hook 实现播放控制、队列管理、状态同步，支持跨会话独立播放
- **防抖播放列表同步**: 500ms debounce 机制将前端播放状态同步到后端，避免频繁写入
- **Per-Conversation 播放隔离**: 通过 `playingSessionIdRef` 追踪当前播放所属会话，切换对话时保存/恢复独立播放状态
- **消息格式向后兼容**: 支持旧版纯文本格式与新版多 Part 结构（thinking + text + tool_call）的无缝降级

### 3. Gateway — Cloudflare Worker 网关

- **统一路由入口**: 单一 Worker 聚合 Auth、D1 数据库、Profile、Backend 等多服务路由
- **Durable Objects 容器编排**: 通过 `backend-worker` 管理 Python 容器生命周期，支持自动休眠（5 分钟超时）与唤醒
- **Lane 路由**: 支持 preview 部署的多环境分流
- **Better Auth 集成**: 服务端 Auth 中间件处理 Apple Sign-In / Google OAuth / Magic Link 三种认证流

### 4. Auth — 共享认证模块

- **Better Auth Server Factory**: 基于 Drizzle ORM + D1 的认证服务工厂，支持多 Worker 复用
- **Apple Sign-In**: ES256 JWT 动态生成 client_secret，集成 Apple OAuth 回调流
- **Magic Link 邮箱认证**: 集成 Resend API 发送无密码登录链接
- **Waitlist 门控**: 通过用户 metadata 字段控制访问权限

### 5. 数据库 — 双数据库策略

- **PostgreSQL (Supabase)**: 存储 LangGraph 检查点、对话历史，asyncpg 异步驱动 + 连接池（pool_size=3, max_overflow=2）
- **Cloudflare D1 (SQLite)**: 存储会话元数据、用户配置，通过 REST API 访问，读密集场景性能优异
- **两层状态模型**: `conversation`（元数据）+ `conversationState`（消息 JSON + 上下文 JSON）分离设计
- **SSL 强制 + 连接回收**: Supabase 连接强制 SSL，300 秒连接回收防止连接泄漏

### 6. iOS — SwiftUI 原生客户端

- **SwiftUI 原生实现**: 原生 iOS 音乐播放体验
- **MusicKit 框架集成**: 调用系统级 Apple Music 能力

---

## 数据流示例

### 用户对话 → 音乐播放

```
User: "播放周杰伦的歌"
  → POST /chat (SSE Stream)
    → LangGraph Agent: search_music("周杰伦")
      → Apple Music API 搜索
      → [SSE: tool_start / tool_end]
    → Agent: add_to_queue(track_id)
      → _emit_action("add_to_queue", {...})
      → [SSE: action 事件]
    → Agent: play_track(1)
      → _emit_action("play_track", {index: 0})
      → [SSE: action 事件]
  → Frontend: MusicKit.setQueue() → play()
  → Debounced sync → /state/sync → D1
```

## 开发

```bash
make dev              # 启动 Web + Backend
make test             # 运行测试
make lint             # 代码检查
make install          # 安装依赖
```

## Apple Music

Backend Apple Music Kit 文档: `apps/backend/APPLE_MUSIC.md`
