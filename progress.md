# Progress

本文件只保留当前状态和已完成任务摘要。已完成任务的中间过程日志已压缩，详细命令和临时排查过程不再长期保留。

## 当前待跟进

### 2026-05-13 - 管理员玩家管理显示俱乐部信息
- 状态：已开始，尚未完成。
- 目标：管理员角色打开玩家管理时显示当前对应俱乐部信息，避免页面缺少俱乐部上下文。
- 影响范围：`yekes-web-javascript` 玩家管理页、俱乐部上下文解析、入口传参；必要时核对后端是否已返回俱乐部名称/房主信息。
- 下一步：从玩家管理页组件、路由 query、私房/俱乐部 store 和 API 返回字段继续追踪。

### 2026-05-08 - 等待室确认参赛记录实现
- 状态：未完成，验证被既有编译问题阻塞。
- 目标：实现独立的“等待室确认参赛”记录，用于区分赛前进入等待室确认参赛与仅报名未确认。
- 影响范围：`yekes-java` MTT 等待室接口、分桌/开赛参赛范围、数据表/Mapper、测试。
- 阻塞：`mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=MttTableArrangeSupportTest#autoArrangeTournamentTables_shouldOnlySeatConfirmedPlayersWhenWaitingRoomEnabledBeforeStart test` 在 main compile 阶段被既有错误阻塞，集中在 `UserHookRoomAction`、`TexasPokerConcurrencyManager`、`MessageType` 等非本次改动文件。

### 2026-05-11 - 查看 admin Google 弹窗效果
- 状态：仅启动检查任务，未记录完成结果。
- 目标：启动 `yekes-admin-javascript` 并截图当前 Google 绑定弹窗样式。
- 下一步：如仍需要视觉验收，重新启动本地 admin 页面并截图确认。

## 最近完成

### 2026-05-13 - 取消充提申请 Google 验证残留
- 状态：已完成。
- 目标：根据截图修复玩家提交上分/下分申请仍提示“Google 验证码不能为空”的问题，同时确认审核授权、玩家申请上分、玩家申请下分均不再要求 Google 验证。
- 影响范围：`yekes-java` 后端玩家充提申请 VO 与控制器测试；`yekes-web-javascript` 只做接口契约复验和进度记录。
- 根因：前端玩家申请接口已经不传 `googleCode`，但后端 `MemberTopupRequestReqVO`、`MemberWithdrawRequestReqVO` 仍有 `googleCode` 和 `@NotBlank`，请求被 Bean Validation 拦截。
- 实现：删除后端两个玩家申请 VO 的 `googleCode` 字段与必填校验；补后端契约测试断言 VO 不暴露 `googleCode`，并保留提交申请不消费 Google 验证码的断言。
- 验证：前端 `pnpm vitest run src/api/__tests__/privateRoom.creditRequestApi.test.ts` 通过，5 tests；前后端 `git diff --check` 通过。后端 `mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=AppPrivateRoomCreditControllerTest test` 仍被既有 main compile 错误阻塞，未进入测试执行。
- 推送完成：yekes-java `dd8403150 fix(club): 取消玩家充提申请谷歌校验` 已推送到 `origin/main`；yekes-web-javascript `5138c000 docs(club): 记录充提申请验证排查` 已推送到 `origin/main`。

### 2026-05-13 - 提交代码推送
- 状态：处理中。
- 目标：按用户要求提交并推送当前已完成代码改动。
- 影响范围：先盘点根工作区和各子仓库状态，再只提交可确认的相关改动，避免混入临时文件或无关未跟踪内容。
- 当前状态：根仓库存在 `progress.md` 本地记录、多个子仓库 dirty 标记以及 `prototypes/`、`screenshots/` 未跟踪目录；继续进入子仓库核对具体改动。

### 2026-05-13 - 记录自动提交推送规则
- 状态：已完成。
- 结果：已在 `CLAUDE.md` 新增“提交与推送规则”，明确以后代码改完后默认验证、提交并推送；同时要求只提交本次相关文件，并记录验证阻塞原因。
- 影响范围：根工作区 `CLAUDE.md`、`progress.md`。

### 2026-05-13 - 额度调整 ownerId 修复推送
- 状态：已完成并推送。
- 结果：`yekes-web-javascript/main` 已 rebase 到最新 `origin/main` 并推送。
- 提交：`1e1aa5c8 fix(club): 修复额度调整房主上下文`；同时推送之前未推送的 `b3098864 fix(club): 放开申请类操作谷歌验证`。
- 验证：相关 Vitest 2 files / 9 tests 通过；`pnpm type-check` 通过；`git diff --check` 通过。

### 2026-05-12 - 手动输入上分金额无法提交
- 状态：已完成并提交，后续已推送。
- 根因：额度调整请求已提交到后端，但 `ownerId + memberId` 成员关系校验失败；管理员场景下前端可能把 `ownerId` 回退成管理员自己的用户 ID。
- 修复：`player-management` 新增 `effectiveOwnerId`；成员列表查询、跳转额度调整、旧 `ChipSettingDialog` 调账均使用当前俱乐部 ownerId。
- 提交：`2b55e1f0 fix(club): 修复额度调整房主上下文`，rebase 后为 `1e1aa5c8`。
- 验证：`pnpm vitest run src/views/new/__tests__/clubRoleGatingPolicy.test.ts src/views/new/player-management/components/__tests__/ChipSettingDialog.source.test.ts` 通过；`pnpm type-check` 通过；`git diff --check` 通过。

### 2026-05-12 - 放开部分业务 Google 验证
- 状态：已完成并推送。
- 结果：审核授权/入会申请、玩家申请上分、玩家申请下分不再触发 Google 验证；审核上/下分、直接调账、退出房间、管理员/大代理等高风险操作仍保留 Google 验证。
- 提交：`yekes-web-javascript` `b3098864`；`yekes-java` `c5da71ea5`。
- 验证：前端相关 Vitest 5 files / 31 tests 通过；`pnpm type-check` 通过；前后端 `git diff --check` 通过。
- 限制：后端目标 Maven 测试被既有编译错误阻塞，主要为 `MessageType.PRIVATE_ROOM_*`、`UserHookRoomAction`、`TexasPokerPlayerRotationManager` 相关缺失符号。

### 2026-05-12 - 游戏内断网提示
- 状态：已完成并推送。
- 结果：`new-yekes-game-javascript` WebSocket 心跳增加 ACK 超时兜底，牌桌内网络异常可进入 reconnecting 并显示连接异常/重连浮层。
- 提交：rebase 后推送为 `39840b1e fix: 修复游戏内断网提示不显示`。
- 验证：`test:socket-core`、`test:overlay-core`、`build:test`、`git diff --check` 均通过。

### 2026-05-12 - 房主 CREDIT 牌桌上分入口
- 状态：已完成。
- 结果：游戏 iframe 保留 `CREDIT` 币种；房主在 CREDIT/CHIP 桌点击上分入口直接 `manualBuyChip`，非房主通过 postMessage 打开上分申请，USDT 仍打开充值页。
- 验证：CREDIT 进房、roomStore、菜单策略定向测试通过；`test:state-core` 40 tests 通过；`build:test` 通过；`yekes-web-javascript` `texaspokerHost.integration.test.ts` 29 tests 通过。

### 2026-05-12 - H5 俱乐部管理弹窗裁剪
- 状态：已完成并推送。
- 根因：关闭牌桌卡片内的 `VanActionSheet` 渲染在 `.room-list-row` 内，被卡片 `overflow: hidden` 裁剪。
- 修复：`RoomListRow.vue` 的关闭房间管理 ActionSheet 增加 `teleport="body"`。
- 提交：`aaccace3 fix(h5): 修复关闭牌桌管理弹窗裁剪`。
- 验证：目标 Vitest 3/3 通过；`pnpm type-check` 通过。

### 2026-05-11 - admin Google 登录预认证
- 状态：已完成并同步远端。
- 结果：admin 账号密码验证后只返回短期 `preAuthToken`；Google 绑定/验证成功后才签发正式 access token。
- 提交：`yekes-java` `ab71ba986 fix(admin): 增加登录前谷歌验证临时态`；`yekes-admin-javascript` `8edb8b2 fix(admin): 接入谷歌登录预认证流程`。
- 验证：后端 `AdminAuthPreAuthServiceTest` 5 tests 通过；前端 `googlePreAuthLogin.test.ts` 2 tests 通过；`pnpm ts:check` 通过。

### 2026-05-11 - admin Google 弹窗样式
- 状态：已完成并推送。
- 结果：优化登录前 Google 绑定/验证码弹窗的宽度、暗色卡片、二维码框、验证码输入布局、按钮区域和移动端适配。
- 提交：`c30fd28 style(admin): 优化谷歌验证弹窗样式`。
- 验证：目标 Vitest 2/2 通过；`pnpm ts:check` 通过。
- 限制：`build:test` 因本地缺少现有依赖 `iconify-icon` 失败，不是本次样式改动引入。

### 2026-05-11 - MTT 选手弹层与代码同步
- 状态：已完成并同步。
- 结果：修复“选手”列表混入未报名授权候选人、自动刷新后清空选手列表；随后 `yekes-web-javascript` rebase 并推送。
- 提交：`0b1984ea fix(mtt): 选手列表隐藏未报名授权候选人`；`cffdefe1 fix(mtt): 避免刷新清空选手列表`。
- 验证：相关 MTT owner players / owner management Vitest 通过；同步后 `main` 与 `origin/main` 对齐。

## 历史归档摘要

### 2026-05-10 - MTT 与德州扑克问题
- MTT 运行时桌布：修复 `modify-settings` 运行时白名单未保存 `tableLogoUrl/tableBackgroundUrl/verticalTableBackgroundUrl/customTableEnabled`，提交 `9840ca753 fix(mtt): 保存运行时桌布设置`。目标 Maven 测试被既有编译错误阻塞。
- MTT 奖励弹层：`MttPayoutSheet` 改为单正文奖励弹层，提交 `f895fb5e fix(mtt): 调整奖励弹层展示样式`，目标 Vitest 通过。
- MTT 延时报名按钮：分析结论为 H5 详情页开赛边界能力刷新/快照合并风险，本轮未改代码。
- 锦标赛重复下注死循环：在投注校验中将 `gameInfo/currentPlayer` 为空视为非法行动，避免轮转/阶段空窗连续下注；目标 Maven 测试被既有编译错误阻塞。
- MTT 查看选手未报名玩家：默认“选手”列表过滤 authorization-only 用户，保留主办方转移搜索能力。

### 2026-05-09 - 安全中心、离线状态与 MTT 字段
- 安全中心首次点击：修复懒加载弹窗首次 `v-if` 创建时 `watch(props.show)` 未立即打开的问题，提交 `c7ae8ab7`。
- 安全弹窗 TDZ：修复 immediate watcher 同步执行访问后声明 const 的 `ReferenceError`，验证 Vitest 与 type-check 通过。
- 德州扑克离线/all-in：`temporaryLeave` 语义收口为仅游戏中站起；断线、退出/返回、切房、MTT 超时等标记 `online=false` 并清理旧 temporaryLeave；all-in 计算不再排除仍在本手的离线/暂离玩家，提交 `b917da49f fix: 修正德州扑克离线状态语义`。目标 Maven 测试被既有编译错误阻塞。
- MTT 背景字段：分析 `vertical_table_background_url` 缺列，并生成增量 SQL；未改业务代码。
- Hermes Agent：完成安装入口修复，避免旧 symlink 覆盖 venv entrypoint；`hermes version` / `hermes --help` 已验证过。

### 2026-05-08 - H5 俱乐部/房间/记录 UI
- 俱乐部“我的”未选中高亮：修复 ClubAvatarRow 选中态和间距问题，提交 `38a44e55`、`a1eaf832`。
- 管理员游戏记录入口：管理员可见“游戏记录”，替换“牌局”文案；相关 compactActions / ClubAvatarRow 测试通过。
- 游戏记录时间筛选：补充默认最近 30 天筛选，后续改为本月/本周/今日/自订胶囊交互，提交 `5dfdf286 fix(records): 优化牌局记录时间筛选`。
- 房间卡片倒计时：删除首页房间卡片倒计时，提交并推送 `fix(home): 删除房间卡片倒计时`。
- 关闭牌桌操作：先调整为“只显示重开”，后按方案 B 增加状态胶囊/重开/编辑，再合并为“管理”按钮 ActionSheet，最终提交 `6a04babc fix(home): 合并关闭牌桌管理操作`。
- 关闭牌桌原型：生成 `prototypes/closed-room-actions-prototype.svg` 与 `.png`，未改业务代码。

### 2026-05-08 - 申请/管理员权限与 Google 验证
- 申请加入房间：按“申请不需要，审核才需要”移除申请前 Google 验证，审核/授权接口保持校验。提交：`yekes-web-javascript` `6dc85ba6`；`yekes-java` `d2efbc799`。验证：前端 11 tests、`pnpm type-check`、后端 `AppPrivateRoomInviteControllerTest` 7 tests 通过。
- 管理员玩家管理入口：将俱乐部管理员“充值”入口改为“玩家管理”，多语言沿用 `roomManagement.player_management`；管理员视角只显示玩家管理，路由绕过被重定向到 playerManagement。前端验证通过。
- AM 房间管理员增删权限：分析并定位需限制普通管理员增删管理员；记录了前后端涉及点。

### 2026-05-08 - MTT #1077/#1079 与等待室
- #1079 延报终局等待：后端已满足延报期内单活进入 `final_waiting` 15 分钟、延报截止后直接 finished；前端补充 `finalWaitDeadline/countdownSeconds` 映射和等待室同步，提交 `8075487c fix(mtt): 使用服务端最终等待倒计时`。
- #1077 等待室恢复：开赛后等待室恢复可见时，未淘汰且有入桌能力/桌位的玩家回详情页展示入桌入口，不停留等待室自动入桌。提交 `5e0b089e fix(mtt): 开赛后等待室恢复回详情页`。
- 未报名误入等待室：前端限制等待室入口身份；waiting-room 403 且详情已加载时静默回详情/排名，不弹“等待室载入失败”。相关 Vitest 与 type-check 通过。
- MTT 详情人数口径：详情“当前/上限”改用报名统计口径，与列表一致，提交 `268900e5 fix(mtt): 修正详情报名人数展示`。

### 2026-05-08 - 德州扑克/MTT 规则与线上问题分析
- Issue 1075 三人桌翻前首行动：当前 main 已包含 `d4cbccbbd` 修复，代码和测试均指向大盲下家先行动；线上异常更像部署包未包含修复或运行环境非当前 main。
- all-in 公共牌节奏：后端公共牌推进改为 Flop 2.2s、Turn 1.8s、River 2.3s，提交 `7c79f6e9f fix(texas): 调整梭哈公共牌展示节奏`。
- timeline 时间偏差：本地 main 使用 `yyyy-MM-dd HH:mm:ss` 24 小时制，`serverTimeMillis` 正确；正式服 12 小时偏差更像部署包/时区/JDBC 配置差异。
- userAuth 503：根据代码与日志判断更像网关/上游服务不可用或 system-server 内部异常；用户敏感 curl 未重放。

### 2026-05-07 - MTT 终局、排名与同步
- MTT 终局排名：前端接收 `player_final_result` 并按服务端 `rankNo` 展示淘汰名次；后端推送玩家终局结果事件。提交：`new-yekes-game-javascript` `aecd2918`；`yekes-java` `3117f7efa`。
- MTT 单人生存等待遮罩：`final_waiting` 且当前玩家仍应留桌等待时按“等待入桌”展示，避免闪“赛事最终结算中”。验证 `test:mtt-runtime` 与 `build:test` 通过。
- 最终排名弹窗自动消失：根因是终局弹窗与等待入桌遮罩互斥，且淘汰玩家未排除运行时等待遮罩；修复后提交 `cebd99ce fix(mtt): 避免淘汰玩家显示等待入桌`。
- 代码同步：`yekes-web-javascript` 和 `yekes-java` 均已完成当时的 rebase/pull/push 或 fast-forward 同步。

### 2026-05-06 - MTT early_settlement 与运行态并发
- 日志分析确认两类问题：`early_settlement` 零和校验误排除其他桌未结底池；玩家运行态同步高并发下乐观锁冲突。
- 修复：`early_settlement` 只排除当前桌已结算底池；玩家运行态更新增加有限乐观锁重读重试，且副作用延后到 DB 写入成功后执行。
- 验证：`MttV2ServiceImplTest` 227 tests 通过。
- 后续：提交由 `yekes-java` 历史记录确认，未在本根进度中保留所有过程日志。

### 2026-05-06 - 工作区进度规则
- 状态：已完成。
- 结果：根目录与四个子仓库写入进度记录规则；四个子仓库 `AGENTS.md` / `progress.md` 已提交并推送。
- 最终提交：`yekes-java` `a55604d06`；`yekes-web-javascript` `1a9e86cb`；`new-yekes-game-javascript` `09df5b31`；`yekes-admin-javascript` `c88ac99`。

## 2026-05-13 - progress.md 清理
- 任务目标：按用户要求清理 `progress.md`，删除已完成任务的详细过程流水，保留最终摘要、验证、提交号和未完成事项。
- 状态：已完成本次压缩整理。
- 追加处理：按用户要求同步清理四个子仓库 `progress.md`，并在根目录及四个子仓库 `AGENTS.md` 增加“任务完成后自动压缩详细过程日志”的规则。
- 结果：`progress.md` 总行数从约 3254 行压缩到约 561 行，当前/阻塞任务保留在待跟进区，已完成任务保留摘要、验证和提交号。

## 2026-05-13 - 管理员玩家管理显示俱乐部信息
- 任务目标：管理员角色打开玩家管理时，需要显示当前对应俱乐部信息。
- 影响范围：`yekes-web-javascript` 玩家管理页、首页玩家管理入口传参、私房俱乐部上下文。
- 当前状态：已确认玩家管理页查询和额度调整已使用当前俱乐部 `ownerId`，但页面未渲染当前俱乐部名称/头像/角色上下文；部分入口未把 `ownerName` 放进路由 query。下一步按 TDD 补源码契约测试后修复展示和入口传参。
- 红灯验证：`pnpm vitest run src/views/new/__tests__/clubRoleGatingPolicy.test.ts src/views/new/home/components/__tests__/compactActions.test.ts` 失败，符合预期；失败点为玩家管理页缺少 `club-context-card`，以及 OwnerManagementPanel 跳转缺少俱乐部 `name`。
- 完成状态：`yekes-web-javascript` 已新增玩家管理俱乐部信息卡片，优先显示路由 `name/ownerName`，回退 `privateRoomStore.roomName`；首页、OwnerManagementPanel 与路由守卫进入玩家管理时保留俱乐部名称。
- 验证：`pnpm vitest run src/views/new/__tests__/clubRoleGatingPolicy.test.ts src/views/new/home/components/__tests__/compactActions.test.ts src/router/__tests__/adminAgentGating.test.ts` 通过，19 tests；`pnpm type-check` 通过；`git diff --check` 通过。
- 提交：`yekes-web-javascript` `b91fede1 fix(club): 显示玩家管理俱乐部信息`。未推送。剩余未处理：根工作区和子仓库仍有既有 `AGENTS.md`、`progress.md`、其它子仓库、`prototypes/`、`screenshots/` 等无关本地差异。
- 最终复核：提交后重新运行同一组 Vitest、`pnpm type-check`、`git diff --check` 均通过；提交范围确认仅包含 7 个业务源码/测试文件。
- 推送完成：`yekes-web-javascript/main` 已推送到 `origin/main`，当前本地与远端提交对齐；推送包含 `b91fede1 fix(club): 显示玩家管理俱乐部信息`、`03364c60 docs: 压缩进度记录`，以及随后出现并已推送的 `573125f8 fix(records): 调整我的牌局记录统计展示`。

## 2026-05-13 01:00 CST - 分析管理员给房主上分谷歌验证失败
- 任务目标：分析管理员角色在玩家管理中给房主执行额度上分时，页面提示“谷歌验证失败”的原因。
- 影响范围：`yekes-web-javascript` 额度调整页/玩家管理入口与 `yekes-java` 私房 credit adjust 接口、Google 2FA 校验链路。
- 当前状态：开始按前端提交参数 -> 后端 controller/service -> Google 校验工具链路排查；本轮先分析原因，不改代码。
- 结论：截图当前的“该用户尚未加入您的房间”来自后端 service 层二次成员校验。前端玩家管理进入额度调整会传 `memberId=620006` 与 `ownerId=620006`；Controller 层已对 `targetOwnerId == memberId` 做豁免，但 `PrivateRoomApplicationServiceImpl.adjustCredit` 又无条件执行 `clubMemberService.isMember(ownerId, memberId)`，房主通常不是自己房间的成员记录，因此管理员给房主调额会被误拦截。若提示“谷歌验证失败”，则是更早阶段按当前管理员本人校验 `googleCode` 失败。
- 修复进展：准备在 `yekes-java` 服务层补回归测试并修复 `ownerId == memberId` 时的成员校验豁免；本次只提交/推送后端相关文件。

## 2026-05-13 调整我的牌局记录统计口径
- 任务目标：我的-牌局记录入口按当前用户维度展示，总流水改为我自己的总下注，总抽水改为我自己的盈亏，避免与私人房间所有游戏记录混用。
- 影响范围：优先追踪 new-yekes-game-javascript 牌局记录页面/接口映射，必要时核对 yekes-java 后端记录接口。
- 当前状态：开始搜索页面文案、字段映射和接口来源。
- 定位进展：截图页面对应 yekes-web-javascript/src/views/new/game-records/index.vue；当前列表仍使用 totalFlow/totalRake 文案和字段。继续核对 normalize 数据逻辑与后端 VO。
- 后端核对：PrivateRoomGameRecordRespVO 已有 betAmount=我的总下注、netProfit=净盈亏，totalFlow/totalRake 注释为全桌口径；问题主要在 H5 前端列表映射和文案。
- 已先改回归测试：我的牌局记录列表应展示“總下注/盈虧”，数值来自 betAmount/netProfit，而不是 totalFlow/totalRake。准备验证红测。
- 实现完成：列表文案改为總下注/盈虧，数据改用 betAmount/netProfit；补齐 6 个语言包。单测 gameRecordsFinancialDisplay 已通过，继续跑 type-check。
- 验证进展：pnpm type-check 已通过；继续执行 pnpm build-only。
- 验证结果：pnpm build-only 已通过。下一步检查 diff 和工作区状态，准备排除无关文件后提交前端修复。
- diff 检查：本次改动仅涉及 game-records 页面、对应单测和语言包；准备执行 git diff --check 并提交 scoped commit。
- 提交完成：yekes-web-javascript 提交 573125f8 fix(records): 调整我的牌局记录统计展示。验证命令：gameRecordsFinancialDisplay 单测、pnpm type-check、pnpm build-only、git diff --check 均通过。

## 2026-05-13 - 提交并推送进度清理
- 任务目标：按用户要求提交并推送根工作区与子仓库的进度清理、自动压缩规则相关改动。
- 影响范围：根工作区、`yekes-web-javascript`、`yekes-java`、`new-yekes-game-javascript`、`yekes-admin-javascript`。
- 完成状态：子仓库进度压缩与自动压缩规则已提交并推送；`yekes-java` 合并远端新增统计口径进度摘要后推送，`yekes-admin-javascript` 推送完成，`new-yekes-game-javascript` 已与远端对齐。
- 提交：`yekes-web-javascript` `55a12bbd docs: 记录进度推送结果`；`yekes-java` `0bc6c1440 docs: 压缩进度记录`、`9e61fb59a docs: 合并进度记录`；`yekes-admin-javascript` `f010ea0 docs: 压缩进度记录`。
- 验证：根工作区及四个子仓库 `git diff --check -- AGENTS.md progress.md` 通过；冲突标记扫描无残留。
- 剩余：根工作区 `AGENTS.md` / `progress.md` 正在准备单独提交并推送，子仓库指针与 `prototypes/`、`screenshots/` 等既有本地差异不纳入本次提交。

## 2026-05-13 确认我的牌局记录修复提交
- 任务目标：确认 yekes-web-javascript 中我的牌局记录统计展示修复是否已提交。
- 当前状态：检查前端子仓库提交状态和最新提交。
- 结果：`yekes-web-javascript` 工作区干净，修复提交 `573125f8 fix(records): 调整我的牌局记录统计展示` 已在历史中；进度记录提交 `55a12bbd docs: 记录进度推送结果` 已随本轮推送同步到 `origin/main`。

## 2026-05-13 - 牌桌内屏蔽申请弹窗
- 任务目标：玩家已进入牌桌时，上下分申请和授权申请不要弹出遮挡游戏。
- 影响范围：优先检查 yekes-web-javascript H5 弹窗/通知逻辑，必要时核对 new-yekes-game-javascript 牌桌 iframe 状态边界。
- 当前状态：开始定位申请弹窗来源和牌桌状态判断。
- 红测：新增 gameOverlayDialogs 规则测试，初次运行因规则模块不存在失败，确认测试覆盖牌桌内屏蔽审核弹窗。
- 实现：新增 shouldSuppressReviewDialogsInGame，并在 App.vue 屏蔽 /game 路由下的申请审核与上下分审核弹窗渲染，保留通知状态。
- 验证：gameOverlayDialogs 与 bootstrapGlobalDialogsPolicy 单测通过；pnpm type-check 通过。继续核对其它同类上分/授权弹窗入口。
- 范围补齐：发现授权申请 authTableShow 与上分申请 topupRequestShow 也是全局弹窗，已一起纳入 /game 牌桌路由屏蔽；审核通知状态不清空，离开牌桌后仍可处理。
- 验证完成：相关 Vitest 4 tests 通过，pnpm type-check 通过，pnpm build-only 通过，git diff --check 通过。准备提交 yekes-web-javascript scoped commit。
- 提交完成：yekes-web-javascript `691747fb fix(game): 牌桌内屏蔽申请弹窗`。验证：相关 Vitest、pnpm type-check、pnpm build-only、git diff --check 均通过。注意：子仓库 progress.md 存在既有无关未提交分析记录，未纳入本次提交；未推送。
- 文档收尾：`yekes-web-javascript` 进度记录已提交 `bbf57ac9 docs: 记录牌桌弹窗推送结果` 并推送到 `origin/main`。

## 2026-05-13 - 推送牌桌内屏蔽申请弹窗修复
- 任务目标：按用户要求推送已提交的牌桌内屏蔽申请弹窗修复。
- 影响范围：yekes-web-javascript。
- 当前状态：开始确认分支状态并准备推送。
- 推送完成：`yekes-web-javascript/main` 已推送到 `origin/main`，包含 `691747fb fix(game): 牌桌内屏蔽申请弹窗`。推送后本地与远端提交对齐；子仓库仍有既有未提交 progress.md 本地记录，未纳入推送。
- 补充推送：`bbf57ac9 docs: 记录牌桌弹窗推送结果` 已推送；`yekes-web-javascript` 当前本地与 `origin/main` 对齐。

## 2026-05-13 - 推送房主本人调额修复
- 任务目标：提交并推送管理员给房主本人 CREDIT 调额的后端修复。
- 影响范围：`yekes-java`。
- 完成状态：`PrivateRoomApplicationServiceImpl.adjustCredit` 已兼容 `ownerId == memberId`，跳过成员关系校验；补充回归测试覆盖管理员给房主本人调额不应调用成员校验。
- 验证：`mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=PrivateRoomApplicationServiceImplCreditAdjustTest test` 仍被既有 main compile 错误阻塞，阻塞点为 `MessageType.PRIVATE_ROOM_*`、`UserHookRoomAction`、`TexasPokerPlayerRotationManager` 等非本次改动文件；`git diff --check` 通过。
- 提交与推送：`db2aefdd8 test(room): 补充房主本人调额回归` 与 `14d62ac54 docs: 记录房主调额修复推送结果` 已推送到 `origin/main`。

## 2026-05-13 - H5 邀请链接复制失败
- 任务目标：修复点击首页「邀請連結」后提示「複製失敗」的问题。
- 影响范围：`yekes-web-javascript` 通用复制 Hook `useCopy`。
- 完成状态：Telegram `copyTextToClipboard` 回调失败时继续降级到浏览器 Clipboard API 和 `execCommand`，只有所有复制路径失败后才提示失败；补充 `useCopy` 回归测试。
- 验证：`pnpm vitest run src/hooks/__tests__/useCopy.test.ts` 通过，2 tests；`pnpm type-check` 通过；`pnpm build-only` 通过；`git diff --check` 通过。
- 提交与推送：`yekes-web-javascript` `c649c129 fix(copy): 修复邀请链接复制降级` 已推送到 `origin/main`。

## 2026-05-13 - H5 邀请链接复制失败
- 任务目标：修复点击首页「邀請連結」后提示「複製失敗」的问题。
- 影响范围：yekes-web-javascript；当前判断为前端复制兼容链路，暂未发现需要后端配合的字段或接口变更。
- 当前状态：已定位到 src/hooks/useCopy.ts；Telegram copyTextToClipboard 失败后直接返回，没有继续执行浏览器 Clipboard/execCommand 回退。下一步先补回归测试。

- 红测：useCopy 回归测试已失败在预期断言：Telegram 复制失败后未进入浏览器剪贴板/execCommand 回退。

- 实现：useCopy 改为 Telegram -> navigator.clipboard -> execCommand 三段降级；Telegram 失败、异常或无回调时不再立即弹失败。

- 验证进展：useCopy 单测 3 tests 通过；pnpm type-check 通过。继续执行 build-only 与 diff 检查。

- 补充修复：将 Telegram copyTextToClipboard 方法提取为局部常量，修复 strict type-check 下可能为 undefined 的类型收窄问题。最终验证 useCopy 单测 3 tests、pnpm type-check、pnpm build-only、git diff --check 均通过。

## 2026-05-13 - Redmine #1116 信用分不足弹窗
- 任务目标：修复 #1116，信用分不足提示改为统一标准对话框，并增加申请上分数量入口。
- 影响范围：优先检查 yekes-web-javascript H5 前端，必要时核对后端上分申请 API。
- 当前状态：已读取 Redmine 问题描述，开始下载参考图并定位现有信用分不足提示。

## 2026-05-13 - 根工作区提交推送
- 任务目标：按用户要求提交并推送当前已完成的子仓库指针和根进度记录。
- 影响范围：根工作区 `progress.md` 与四个子仓库 gitlink 指针；不纳入 `prototypes/`、`screenshots/` 和子仓库未跟踪日志。
- 当前状态：四个子仓库均已确认与各自远端对齐；准备基于远端干净 `master` 快照提交根工作区更新，避免推送旧本地历史中的敏感提交。

## 2026-05-13 01:28 CST - 放开低风险俱乐部动作 Google 验证
- 任务目标：审核授权、玩家申请上分、玩家申请下分不再要求 Google 验证，减少频繁弹窗。
- 影响范围：跨 `yekes-web-javascript` 前端弹窗/请求参数与 `yekes-java` 后端 private-room invite/credit 接口校验。
- 当前状态：开始按 TDD 核对前后端现状，若仍有 Google 校验残留则直接修复并提交推送。
- 红测：Home.texaspokerEntry 新期望失败，当前代码仍调用 insufficient_chips toast，未显示标准弹窗和去申请按钮。准备实现。
- 继续处理：确认玩家提交上/下分申请和入会授权审批已有免 Google 覆盖；残留点集中在房主/管理员审核上分、下分申请，准备先更新测试契约再改实现。
- 红测与实现：前端审核中心/工作台/全局审核弹窗已改为审核上/下分不弹 Google、不传 `googleCode`；后端 approve 上/下分接口已移除 Google 校验入参与调用。下一步执行前端定向测试、类型检查与后端定向 Maven 验证。
- 验证：前端定向 Vitest 28 tests、`pnpm type-check`、`pnpm build-only`、前后端 `git diff --check` 均通过；后端定向 Maven 测试仍被既有 main compile 错误阻塞，未进入本次测试执行。

## 2026-05-13 - 分析 issue #1114
- 任务目标：读取并分析 https://abcadwiki.org/issues/1114 对应问题，核对当前 AG 前后端实现边界。
- 影响范围：待 issue 内容确认后定位；当前先读取 issue 原文并按前端/接口/后端链路排查。
- 当前状态：开始读取 issue 与本地仓库状态。

- Issue 内容：#1114 标题为“推广链接页面，没推广链接可以复制”，截图实际位于 H5 推廣記錄查詢/邀請列表页。
- 前端定位：/promotion-data 只渲染 SalesmanDataView/HostDataView 和房主横幅，没有推广链接复制入口；/promote-link 页面已有链接展示/复制，并调用 /game/club-promote/my-info 获取 inviteCode。
- 后端核对：AppClubInviteController#getMyPromoteInfo(ownerId) 已按房主 ownerCode 或业务员 play_club_invite.inviteCode 返回推广码，后端接口能力存在。
- 绿测：Home.texaspokerEntry 9 tests 通过；本地余额不足与后端 CREDIT_INSUFFICIENT 已打开标准弹窗，去申请可打开上分申请并携带房主 ownerId。
- 验证完成：Home.texaspokerEntry 9 tests、pnpm type-check、pnpm build-only、git diff --check 均通过。发现工作区有既有无关差异，提交时只纳入 #1116 相关 Home 页面、Home 测试和多语言文案。
- 提交完成：yekes-web-javascript `e0ec7ea9 fix(home): 优化积分不足上分申请入口`。改动范围仅包含 Home 页面、Home 进入牌桌测试、多语言文案和进度记录；未提交既有无关本地差异。

## 2026-05-13 - 推送 Redmine #1116 修复
- 任务目标：按用户要求推送已提交的 #1116 积分不足上分申请入口修复。
- 影响范围：yekes-web-javascript。
- 当前状态：开始确认分支 ahead 提交并准备推送。

## 2026-05-13 - 修复 issue #1114 推广记录页缺少复制链接
- 任务目标：在 H5 推廣記錄查詢/邀請列表页提供可复制推广链接入口。
- 影响范围：yekes-web-javascript；复用现有 /game/club-promote/my-info 和 useCopy 复制链路，暂不改后端。
- 当前状态：准备先补回归测试，再修改 promotion-data 页面。
- 推送完成：`yekes-web-javascript/main` 已推送到 `origin/main`，包含 `e0ec7ea9 fix(home): 优化积分不足上分申请入口`；推送后本地与远端提交对齐。

## 2026-05-13 - 自己房间大代理添加按钮消失
- 任务目标：分析并修复“我的房间”进入大代理页面后添加按钮不显示的问题。
- 影响范围：yekes-web-javascript H5 前端，大代理/下线/推广数据入口与权限判断。
- 当前状态：开始定位页面组件、按钮显示条件和自己的房间上下文传参。

- 定位进展：大代理页面当前模板只有空态和列表，`add-btn` 样式与添加弹窗样式仍在 CSS 中，但 Vue 模板/脚本已没有打开添加弹窗或调用新增接口的逻辑；继续从历史提交和后端接口确认恢复方式。

- 验证进展：前端 promotionDataAccess 6 tests 通过；git diff --check 通过；pnpm type-check 通过。继续执行 build-only。

- 验证完成：promotionDataAccess 6 tests、pnpm type-check、pnpm build-only、git diff --check 均通过。准备提交 #1114 相关前端文件。

- 修复方案：恢复 H5 房主大代理添加入口，并恢复 APP `/game/agent/set` 后端能力；新增/调整测试要求添加大代理必须通过 Google 2FA，防止敏感授权无校验。

- 提交推送完成：yekes-web-javascript `5e88ee26 fix(promotion): 增加推广记录页复制链接` 已推送到 origin/main。提交范围仅包含 #1114 相关页面、测试和进度记录；其它既有未提交审核/上分相关改动未纳入。

- 代码修改：前端恢复大代理添加按钮/成员选择/Google 2FA 后调用 setAgentAPI；后端恢复 APP 设置大代理接口并默认 1% 返佣比例，设置/撤销均保留审计和 Google 2FA。开始运行前后端定向测试。

- 验证结果：前端 `pnpm vitest run src/views/new/__tests__/clubSensitiveTwoFaSource.test.ts` 通过；后端 `mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=AppAgentControllerTest test` 被现有模块编译错误阻塞，错误集中在 AppPrivateRoomCreditController、UserHookRoomAction、TexasPokerPlayerRotationManager 等非本次改动文件。

- 修正前端类型：成员接口结果显式按数组归一化，重新运行 `pnpm type-check` 和相关 Vitest。

- 完成状态：前端恢复大代理添加按钮与弹窗，新增 setAgentAPI 并通过 Google 2FA 提交；后端恢复 `/game/agent/set`，设置大代理前校验 Google 2FA、写 AGENT_SET 审计并调用 AgentService.setAgent，缺省返佣比例 1%。验证：前端定向 Vitest、pnpm type-check、前后端 diff --check 通过；后端定向 Maven 测试仍被既有模块编译错误阻塞。

- 推送完成：yekes-web-javascript `adb11928 feat(club): 支持房主设置大代理` 已在 `origin/main`；yekes-java `0fc449e78 docs(club): 记录俱乐部验证规则复验结果` 已推送到 `origin/main`，包含前置 `ba6ed56ed fix(club): 放开上下分审批谷歌验证` 与 `c613377f3 feat(club): 支持房主设置大代理`。根仓库准备只提交 `progress.md` 和两个子仓库指针，继续排除 `prototypes/`、`screenshots/`、`yekes-admin-javascript` 临时差异。
