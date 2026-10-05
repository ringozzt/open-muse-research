# True Open-Muse 构想：全开源、厂商无关的个人 Agent 栈

- 日期：2026-10-05
- 背景：见 [open-muse 调研](./README.md)。open-muse 开源了客户端和编排服务，但 agent runtime（Managed Agents）是闭源云服务。本构想探讨：用已有的开源项目补上这一层，拼出一套全开源、厂商无关的栈。

## 1. 缺口分析

open-muse 的栈按开放程度拆成四层（详见调研 §7）：

| 层 | 状态 | 说明 |
|---|---|---|
| 客户端（iOS/Mac/Web/Android） | 开源 | `src/`，四端全在仓库里 |
| 编排服务（账号/密钥/调度/同步） | 开源 | `server/`，Workers + D1 |
| Agent runtime（loop、工具执行、沙箱、session） | **闭源** | 火山 Ark MA / Anthropic MA |
| 模型 | 闭源 | 各家 API key |

缺口只有一个：**MA runtime**。而它的接口是文档化的——`shared/ma-contract.json` 注册了 52 个操作（agent 6、environment 5、session 11、memory 12、连接与凭证 13、skills 2、文件 3）。接口现成，缺的是一个开源实现。

## 2. 拼法

```
┌──────────────────────────────────────────────┐
│ open-muse UI（基本不动）                      │
│  · ma-provider.ts 加第三个 provider: "open"   │
│  · baseUrl 指向自建的开源 MA 服务             │
└──────────────────┬───────────────────────────┘
                   │  MA 52 操作（接口定义现成）
┌──────────────────▼───────────────────────────┐
│ 开源 MA runtime（新建，唯一的新组件）          │
│  ┌────────────────────────────────────────┐  │
│  │ loop / session / 工具 / 沙箱 / 记忆    │  │
│  │ → dsh 插件运行时（headless，不用其 UI） │  │
│  └────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────┐  │
│  │ 模型可移植层 → oar                     │  │
│  │ （统一 harness 接口，用户自带模型）     │  │
│  └────────────────────────────────────────┘  │
└──────────────────┬───────────────────────────┘
                   │
┌──────────────────▼───────────────────────────┐
│ Open Muse 服务（基本不动）                     │
│  · ArkRemote 之外加一个 OpenRemote            │
│  · 账号/密钥/调度/同步逻辑复用               │
└──────────────────────────────────────────────┘
```

为什么是 dsh 做底座：概念几乎一一对应——dsh 的 profile ≈ MA agent，session log ≈ MA session，sandbox/工具/记忆都有对应插件；且 dsh 是 Cordis 插件化的，UI 本来就是可拆的插件，headless 跑正是其设计所允许的。

为什么是 oar 做模型层：oar 是统一 7 个 harness 编程接口的 TS library。在 MA server 内部用 oar 驱动用户本地已有的 harness，"模型"这一层就不被任何一家绑死——用户有 Codex 就用 Codex，有 Claude 就用 Claude。

## 3. 概念映射（MA → dsh）

| MA 操作组 | dsh 对应物 | 备注 |
|---|---|---|
| Agents（6）：创建/更新/版本 | profile + bundle 组合 | profile 即"配好的 agent"，bundle 即插件集合 |
| Environments（5） | sandbox 插件 | 每 session 一个隔离环境 |
| Sessions（11）：消息/历史/事件流 | session log + 事件 | 需做事件协议翻译（见 §4） |
| Memory（12）：记忆库/批量写 | 记忆工具插件 | GOALS.md 等五个文档的 CRUD |
| Skills（2）：ZIP 上传/按 ID 查 | skills 即插件 | 需 ZIP → 插件安装桥 |
| Files（3）：上传/下载/删除 | 文件工具 | — |
| Connections（13）：vault/OAuth | — | 首版可砍（交集策略） |

## 4. 诚实的难点

1. **事件协议翻译**（最磨人）：dsh 的 session log 语义（model-visible-means-logged）要翻译成 MA 的 SSE 事件流（`session.status_idle`、`agent.message` 等）。两套事件模型的粒度不同，这是适配层的主战场。
2. **审批流对接**：MA 的 `requires_action`（工具需用户确认）要对接到 dsh 的审批机制，语义必须对齐，否则安全模型穿帮。
3. **长尾**：文件上传、skills ZIP 安装、记忆版本历史——都能做，都是工时。

工作量估计（参考 chyroc 单人 6 天 482 commits 的产能样本）：核心链路（聊天 + 记忆 + session）一人加 AI 数周可跑通；52 个操作全齐是数月量级。策略上沿用 open-muse 自己的交集哲学：**先实现 UI 真正用的 20 来个操作，其余声明不支持**，不追求首版全兼容。

## 5. 前置条件

1. **UI 的 LICENSE**：open-muse 仓库目前无 LICENSE 文件。这是 step zero——没有授权，一切白搭。需请作者补一个开源协议。
2. **"厂商无关"的边界**：指的是 runtime 层不被任何一家云绑死；模型本身仍需各家的 API key，不是零成本。

## 6. 分阶段路线

- **Phase 0**：拿到 UI 的开源授权；冻结 MA contract 的目标子集（UI 核心链路所需的 ~20 个操作）。
- **Phase 1**：dsh headless + MA 适配插件，跑通"建 session → 发消息 → 收事件流 → 读写记忆"。
- **Phase 2**：oar 接入，模型层可换（用户自带 harness）。
- **Phase 3**：补长尾操作（文件、skills ZIP、审批对齐），向 52 个操作收敛。
- **Phase 4**：Open Muse 服务加 `OpenRemote`，多端同步链路打通。

## 7. 模式命名

**Supabase for Firebase**：不抄 UI，重写它文档化的 API。Firebase 定义了接口，Supabase 开源实现了它；这里 MA 定义了 52 个操作，我们用 dsh + oar 开源实现它。UI 和服务已经开源且 provider 无关，等的只是一个开源的 runtime。
