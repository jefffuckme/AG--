# Repository Guidelines

## Workspace Scope
- This directory is a multi-repo workspace for the AG H5 project.
- Frontend repo: `yekes-web-javascript/`
- Backend repo: `yekes-java/`
- Unless the user explicitly says otherwise, tasks that involve business logic, API calls, authentication, payment, game flow, user state, or field mismatches must be analyzed across both repos.

## Default Working Rules
- For frontend issues, always check whether the corresponding backend API, DTO, enum, validation, or response shape also needs review.
- For backend API changes, always check whether the frontend request parameters, response parsing, page state, and type assumptions still match.
- For bug diagnosis, trace the full chain when relevant:
  1. frontend page/component/composable/store
  2. frontend API wrapper
  3. backend controller
  4. backend service/domain logic
  5. DTO/VO/request-response contract
- Do not assume a bug is frontend-only or backend-only until the request/response contract has been checked.

## Progress Logging
- Codex must keep a `progress.md` file updated while working in this workspace.
- At the start of every user task, append a short entry to `progress.md` with the task goal, affected repo(s), and initial status.
- After every meaningful step or completed subtask, append or update `progress.md` with what changed, current status, and the next action.
- When a blocker, failed command, test result, commit, or push occurs, record it in `progress.md` immediately.
- When the task is complete, add a final entry with the outcome, verification commands, commit hash if any, and remaining risks or follow-up items.
- After a task is complete, compress its detailed step-by-step log into a concise final summary; keep active, blocked, or unpushed work detailed enough to resume.
- For cross-repo work, use the workspace-root `progress.md`. If a subproject later adds its own `AGENTS.md`, repeat this rule there and keep that project's local `progress.md` in sync for work scoped only to that subproject.

## Repo Map

### Frontend
- Path: `yekes-web-javascript/`
- Stack: Vue 3 + Vite + TypeScript + Vitest
- Common commands:
  - Install: `pnpm install`
  - Dev: `pnpm dev`
  - Type check: `pnpm type-check`
  - Unit test: `pnpm vitest run`
  - Build: `pnpm build`

### Backend
- Path: `yekes-java/`
- Stack: Maven multi-module, Java backend
- Root build file: `yekes-java/pom.xml`
- Common commands:
  - Build: `mvn -T 1C clean install -DskipTests`
  - Test: `mvn test`
  - Single module example: `mvn -pl yudao-module-game/yudao-module-game-biz -am test`

## How To Work In This Workspace
- Start from `/Users/mingge/Documents/IdeaProjects/AG_H5` when the task may span multiple repos.
- When referencing files in responses, include the repo prefix so it is always clear which codebase is being discussed.
- Prefer searching from the workspace root first, then narrow to the repo actually affected.
- If only one repo needs code changes, still mention that the other repo was checked when that check was relevant.

## API And Contract Changes
- If request fields, response fields, enums, status codes, or auth rules change on one side, verify the other side in the same task.
- For frontend-backend integration tasks, explicitly compare:
  - request field names
  - optional vs required fields
  - nullability/default values
  - enum values
  - pagination fields
  - signature/auth headers
  - error code and message handling

## Safety Rules
- Never overwrite or revert user changes in either repo unless explicitly requested.
- If both repos contain related local modifications, read them carefully before editing.
- Keep commits scoped to the repo(s) actually changed. Do not mix unrelated frontend and backend edits into one commit unless the user asks for that.
- Commit messages must be written in Chinese unless the user explicitly asks otherwise.
- GitHub PR content must be written in Chinese by default, including PR titles, PR descriptions, PR comments, and code review summaries, unless the user explicitly asks for another language.

## Response Expectations
- When a task is cross-repo, report findings and changes by repo.
- When a task is single-repo, say so explicitly if you checked the other repo and found no required changes.
- If blocked by missing context, state which repo or contract boundary is unclear instead of guessing.


<claude-mem-context>
# Memory Context

# [AG游戏] recent context, 2026-05-12 11:59pm GMT+8

Legend: 🎯session 🔴bugfix 🟣feature 🔄refactor ✅change 🔵discovery ⚖️decision 🚨security_alert 🔐security_note
Format: ID TIME TYPE TITLE
Fetch details: get_observations([IDs]) | Search: mem-search skill

Stats: 50 obs (9,455t read) | 3,205,499t work | 100% savings

### Apr 27, 2026
S4 俱乐部创建时长与重新打开时长不一致 Bug — 新 Bug 报告，同时确认上一个 MTT 修复已成功推送 (Apr 27 at 6:04 PM)
S6 修复俱乐部创建时长与重新打开时长不一致 — 实施代码变更阶段 (Apr 27 at 11:00 PM)
S7 Bug 分析：受邀玩家在审批中心看到「我发起」标签（应显示「受邀」），定位根因并提出修复方案 (Apr 27 at 11:07 PM)
### May 2, 2026
S9 用户询问 --dangerously-skip-permissions CLI 参数的用法 (May 2 at 10:47 PM)
S11 解释 Claude Code CLI 参数 --dangerously-skip-permissions 的作用、风险和替代方案 (May 2 at 11:16 PM)
S8 用户询问 --dangerously-skip-permissions CLI 参数的用法 (May 2 at 11:16 PM)
3342 11:20p 🔵 game.ts 盲抓消息处理的完整字段映射
3343 " 🔵 roundStore 状态结构：盲抓与其他可选功能的并列设计
S12 解释 --dangerously-skip-permissions 参数的用途与风险，以及德州扑克游戏返回按钮导致玩家离线状态问题分析 (May 2 at 11:20 PM)
S10 用户询问 --dangerously-skip-permissions 参数的作用和使用方法 (May 2 at 11:20 PM)
3344 11:21p 🔵 gameActions.ts 高级功能 WebSocket 命令完整列表
3345 " 🔵 GameShellView handleStraddleRespond 错误处理与 gameActions 绑定方式
3346 " 🔵 roundStore.resetRound 统一清除所有高级功能状态
3348 11:22p 🔵 AG游戏项目首页架构：ClubHomePage 组件结构
3347 " 🔵 useStraddleDialogActions 测试覆盖四个核心场景
3349 11:23p ✅ 从 GameShellView.vue 中移除 StraddleDialog 模板代码
3350 " 🔵 首页关键路径调度器：Bootstrap 快照 + 降级并行请求机制
3355 11:25p 🔵 首页可见性刷新策略：30秒节流，仅刷新高频变化数据
3351 11:26p 🔵 用户请求删除盲抓（Straddle）功能
3359 " 🔵 德州扑克游戏点击返回按钮后玩家在线状态问题分析
3352 " 🔄 删除盲抓功能的 Vue 组件和 Composable 文件
3353 " 🔵 盲抓功能残留引用分布在多个文件中
3354 11:27p 🔵 game.ts 中盲抓处理逻辑：自动拒绝 + 状态清理
3356 " 🔄 从 game.ts STRADDLE_POSTED 处理中移除 dismissStraddleRequest 调用
3357 11:28p 🔵 AG游戏首页 home composables 目录结构
3358 " 🔵 useRoomData：房间数据管理的轮询、倒计时增量更新和可见性暂停机制
3363 " 🔵 首页数据初始化顺序：setup 阶段同步调用 + onMounted 启动调度器
3366 " 🔵 首页实际 onMounted 实现：IntersectionObserver 延迟牌桌加载 + 非关键任务调度器
S14 将 Club 首页轮询定时器重构为 Page Visibility API 单次刷新模式，并修复所有相关 TypeScript 编译错误和测试文件引用 (May 2 at 11:30 PM)
3360 11:30p 🔵 德州扑克前端项目结构与返回/离开功能相关文件定位
3361 " 🔵 德州扑克游戏返回按钮触发玩家离线状态问题根因调查启动
3362 11:31p 🔵 GameShellView.vue 中返回房间逻辑由 handleBackRoom 和 exitRoomFlow 实现
3364 " 🔵 GameShellView.vue 中 exitRoomFlow 和 enterRoomFlow 的完整调用链路追踪
3365 " 🔵 exitRoomFlow 和 enterRoomFlow 的实现位于 services/room.ts
3367 " 🔵 handleSettingsBack 调用 handleBackRoom，换桌流程复用 exitRoomFlow+enterRoomFlow 模式
3369 " 🔵 useClubTables 完整实现：竞态控制、分批加载、8秒 TTL 缓存、智能排序算法
3368 " 🔵 room.ts 中 exitRoom 通过 WebSocket 命令实现，存在房间同步阻塞保护
3370 11:32p 🔵 handleBackRoom 在 GameShellView.vue 中是从外部导入的函数而非本地定义
3371 " 🔵 handleBackRoom 定义在 GameShellView.vue 第 2713 行附近的对象结构中
3372 " 🔵 useClubNavigation：俱乐部选择上下文同步 privateRoomStore 和 30 秒自动刷新
3373 " 🔵 GameShellView.vue 是超大型单文件组件，共 5850 行
3374 11:33p 🔵 handleBackRoom 完整实现：先通知 Lobby 再调用 exitRoomFlow，不主动断开 iogame 连接
3380 " 🔵 首页牌桌数据触发链：clubs 就绪 + 牌桌区域进入视口才触发加载
3375 " 🔵 exitRoomFlow 关键实现：immediate 模式下先清理状态再发送 exitRoom 命令，且不等待服务端响应
3376 " 🔵 服务端玩家离线逻辑由 UserHookRoomAction 和 TexasPokerOfflineUtil 处理
3377 " 🔵 服务端 Java 代码中不存在 exitRoom/leaveRoom/EXIT_ROOM 协议处理器
3378 11:34p 🔵 TexasPokerOfflineUtil 实现：用 Redis 计数器追踪玩家自动离线超时次数，非在线状态标记
3379 " 🔵 确认服务端完全不存在 exitRoom 协议处理器
3381 " 🔵 UserHookRoomAction 完整实现：玩家断线触发 quit 钩子，德州扑克断线保留座位设为暂时离开状态
3383 " 🔵 首页游戏预热机制：watch 驱动的 idle 预热 + 关键路径调度器 + 弹窗组件预加载
3382 11:35p 🔵 前端 WebSocket 协议定义从 JSON 文件加载，exitRoom 命令有效性需查 protocol-ws.json
3384 " 🔵 exitRoom 命令在 protocol-ws.json 中存在，cmdMerge=8060933，但服务端无对应处理器
3385 " 🔵 服务端无 exitRoom 命令处理器（cmdMerge=8060933），问题根因最终确认
3387 " 🔵 createHomeCriticalPathScheduler 真实实现：任务队列 + markCriticalReady 双重职责
3386 " 🔵 服务端 Action 层仅有 UserHookRoomAction、TexasPokerAction 和 BaccaratAction，无独立的房间退出 Action
3388 11:36p 🔵 服务端存在 quitRoom Action 方法，但前端发送的是 exitRoom 命令，两者命名不匹配
3389 " 🔵 TexasPokerCmd 接口定义德州扑克命令码，cmd 来自 GameModuleCmd.texasModule_123_cmd
3390 " 🔵 TexasPokerCmd 完整命令码表：quitRoom=5 是退出房间的正确服务端命令，前端使用了错误的命令名
3391 11:37p 🔵 quitRoom 处理器通过 RoomHandler 委托处理，支持多节点转发
S13 德州扑克游戏点击返回按钮后玩家在线状态未及时更新问题的完整根因分析 (May 2 at 11:37 PM)
**Investigated**: 前端：GameShellView.vue（5850行超大文件）、handleBackRoom 函数实现、exitRoomFlow 实现（services/room.ts）、protocol-ws.json 中的 exitRoom 命令定义。
    后端：UserHookRoomAction.java（断线钩子）、TexasPokerAction.java（quitRoom处理器）、TexasPokerCmd.java（命令码表）、TexasPokerOfflineUtil.java（超时计数器）。
    H5外层：yekes-web-javascript 的返回按钮处理逻辑（useTelegramBackButton）。

**Learned**: 1. 存在两套"返回"机制互不感知：H5外层（yekes-web-javascript）的返回按钮直接调用 router.push('home') 销毁 iframe，没有向游戏 iframe 发 postMessage，游戏客户端的 handleBackRoom() 完全未被调用。
    2. exitRoomFlow 使用 immediate=true 模式：先 cleanupRoomState() 再 fire-and-forget 发送 exitRoom WS 命令，不等待服务端响应。
    3. 前端发送的是 exitRoom 命令（cmdMerge=8060933），但服务端 TexasPokerAction 的处理器名为 quitRoom（子码=5），命名不匹配导致前端命令被服务端丢弃。
    4. 玩家最终变为离线依赖 TCP 断线触发 UserHookRoomAction.quit() hook，再经 quickExit() → handleTexasPokerDisconnect() 设置 online=false, temporaryLeave=true。
    5. 有座位玩家断线后标记为"暂时离开"而非直接移除，保留重连恢复能力。
    6. TexasPokerOfflineUtil 管理的是操作超时计数（3次触发自动离线），与主动返回场景无关。
    7. 断线到服务端感知有 50~500ms 不确定延迟，TCP 丢包时可能需等心跳超时（10s）才能感知。

**Completed**: 完成了完整的前后端链路分析，定位了玩家点击返回按钮后在线状态延迟更新的三个根本原因：
    1. H5外层返回按钮绕过了游戏客户端的 handleBackRoom()
    2. exitRoomFlow 的 exitRoom 命令与服务端 quitRoom 处理器命名不匹配
    3. 整个退出流程依赖被动 TCP 断线机制而非主动协议通知

**Next Steps**: 根据分析结论，下一步可能的修复方向：
    1. H5外层返回按钮在 router.push 前先向 iframe 发 postMessage，通知游戏客户端主动调用 handleBackRoom()
    2. 等待 exitRoom 命令响应（或修正命令名为 quitRoom）后再跳转，确保服务端状态即时同步
    3. 或在服务端为 exitRoom 命令添加处理器（等同于 quitRoom 的逻辑）


Access 3205k tokens of past work via get_observations([IDs]) or mem-search skill.
</claude-mem-context>
