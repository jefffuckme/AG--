# MTT 玩家侧 Telegram 推送设计

**日期：** 2026-04-23

## 背景

当前项目里，MTT 已经具备两套与通知相关的能力：

- MTT 站内通知：落在 `mtt_notification`
- MTT 实时事件：通过 WebSocket / 时间线 / 等待室等方式在前端展示

同时，项目已经存在一套 Telegram Bot 发送底座：

- `yekes-java/yudao-module-game/.../TelegramBotService.java`
- `yekes-java/yudao-module-game/.../TelegramWebhookController.java`
- 用户 `telegramUserId` 字段可作为默认发送目标

本次需求不做“再设计一套新通知体系”，而是为 MTT 的玩家关键节点补一层 Telegram 强提醒。

## 目标

为 MTT 玩家补齐 Telegram Bot 推送，覆盖用户不在线也应立即获知的关键结果与关键提醒。

要求：

- 只推玩家侧，不推主办方 / 管理侧
- 只覆盖高价值、低骚扰的少量消息类型
- Telegram 失败不影响 MTT 主流程
- 文案采用短文案风格
- 按钮只做最小必要跳转

## 非目标

- 不推主办方审核、领奖处理、人数不足等运营消息
- 不推普通赛中播报
- 不推赛事创建、赛事更新等弱提醒
- 不在本次需求中重做 MTT 站内通知体系
- 不要求前端增加新的通知中心页面

## 当前事实

### 现有 MTT 通知类型

前端通知面板和后端常量中已经存在完整的 MTT 通知类型集合，例如：

- `registration_approved`
- `registration_rejected`
- `tournament_starting`
- `tournament_finished`
- `tournament_cancelled`
- `prize_claimable`

相关位置：

- `yekes-web-javascript/src/views/mtt/MttNotificationPanel.vue`
- `yekes-java/yudao-module-game/.../MttConstants.java`

### 现有 MTT 事件触发点

后端 MTT 事件支持里已经会在状态变化或结果固化时写入站内通知：

- `EVENT_TOURNAMENT_STATUS_CHANGED`
- `EVENT_RESULTS_FINALIZED`
- 玩家报名审核通过 / 拒绝
- 赛事取消
- 可领奖

相关位置：

- `yekes-java/yudao-module-game/.../MttEventSupport.java`

### 当前 Telegram 底座能力

Telegram 发送底座已经支持：

- 纯文本消息
- 带 WebApp 按钮消息
- 基础配置校验

因此本次重点不在 Telegram 基础建设，而在“哪些 MTT 事件要发、发什么、跳哪里”。

## 方案对比

### 方案 A：直接把所有现有 MTT 站内通知都镜像到 Telegram

优点：

- 开发路径最短
- 站内与 TG 类型天然一致

缺点：

- 消息过多
- 会把审核中、提交成功、赛事更新等弱提醒也推到 TG
- 用户骚扰风险高

### 方案 B：只挑选玩家关键节点做 Telegram 推送

优点：

- 更符合 TG 的“强提醒”定位
- 触达价值高
- 更容易控制文案和频率

缺点：

- 需要人工定义 V1 推送白名单

### 方案 C：只做赛前赛后提醒，不做报名和领奖

优点：

- 范围最小

缺点：

- 用户会漏掉报名结果和领奖结果
- 对实际体验帮助不完整

## 选定方案

选定 **方案 B：只挑选玩家关键节点做 Telegram 推送**。

原因：

- TG 更适合“必须马上知道”的消息
- 站内通知可以承载长尾信息
- V1 应优先控制骚扰率，而不是追求覆盖全部事件

## 推送范围

V1 只推以下 6 类玩家消息：

1. 报名通过 `registration_approved`
2. 报名拒绝 `registration_rejected`
3. 即将开赛 `tournament_starting`
4. 赛事取消 `tournament_cancelled`
5. 比赛结束 `tournament_finished`
6. 可领奖 `prize_claimable`

## 不推范围

V1 明确不推：

- `registration_submitted`
- `registration_pending_review`
- `table_assigned`
- `tournament_created`
- `tournament_updated`
- `owner_transferred`
- `claim_submitted`
- `claim_processing`
- 普通淘汰播报
- 主办方 / 管理侧所有消息

说明：

- `table_assigned` 在赛中频率更高，容易打扰，且等待室与页面状态足以承接
- `tournament_finished` 比 `table_assigned` 更稳定、更完整，适合保留为赛果提醒

## 文案策略

V1 采用 **短文案**。

原则：

- 一眼看懂结果
- 详情交给按钮落地页承接
- 不在 Telegram 里塞过多说明文字

### Demo 文案

#### 1. 报名通过

正文：

`✅ You are registered for {mttName}.`

按钮：

`🎯 View Tournament`

#### 2. 报名拒绝

正文：

`❌ Your registration for {mttName} was not approved.`

按钮：

`📋 View Details`

#### 3. 即将开赛

正文：

`⏰ {mttName} starts in {minutes} minutes.`

按钮：

`🏁 Get Ready`

#### 4. 赛事取消

正文：

`⚠️ {mttName} has been cancelled.`

按钮：

无按钮，纯文本

#### 5. 比赛结束

正文：

`🏁 {mttName} has finished.`

按钮：

`📊 View Result`

#### 6. 可领奖

正文：

`🏆 Your prize is ready to claim.`

补充行：

`Prize: {payoutText}`

按钮：

`💰 Claim Prize`

## 按钮跳转

按钮跳转建议复用现有 MTT 通知动作路径，不引入新的前端协议。

映射如下：

- 报名通过：`/m/{mttCode}`
- 报名拒绝：`/m/{mttCode}`
- 即将开赛：`/m/wait/{mttCode}`
- 赛事取消：无按钮
- 比赛结束：`/m/{mttCode}`
- 可领奖：`/m/rc?mttId={mttCode}`

这与现有 MTT 通知动作基本一致，可参考：

- `yekes-java/yudao-module-game/.../MttNotificationServiceImpl.java`

## 发送规则

### 基本规则

- 仅对存在 `telegramUserId` 的 MEMBER 用户发送
- Telegram 发送失败不影响 MTT 主业务返回
- 没有匹配文案或按钮场景时，优先退化为纯文本

### 时机规则

- 报名通过 / 拒绝：在审核结果落定后立即发送
- 即将开赛：在赛事进入 waiting 阶段时发送一次
- 赛事取消：在赛事取消时发送一次
- 比赛结束：在赛事状态进入 finished 时发送一次
- 可领奖：在结果固化且玩家进入可领奖状态时发送一次

### 去重规则

同一用户、同一赛事、同一通知类型，V1 只允许发送一次。

建议使用如下维度去重：

- `user_id`
- `mtt_id`
- `notice_type`

如果已有相同发送记录，则跳过。

## 推荐实现边界

建议不要在所有 MTT 业务方法里零散直接调用 `TelegramBotService`。

建议新增一个 MTT 专用 Telegram 协调服务，例如：

- `MttTelegramPushService`

职责：

- 判断事件是否属于 V1 白名单
- 渲染短文案
- 解析按钮跳转 URL
- 调用 Telegram 发送
- 记录发送结果 / 去重状态

事件源建议直接复用现有 `MttEventSupport` / `MttNotification` 触发点，不再额外造第三套事件。

## 验收标准

1. 玩家报名通过后，若存在 `telegramUserId`，收到一条 TG 消息，按钮跳赛事详情。
2. 玩家报名被拒后，收到一条 TG 消息。
3. 赛事进入 waiting 阶段后，已报名玩家收到一条“即将开赛”提醒。
4. 赛事取消后，相关玩家收到一条取消提醒。
5. 赛事结束后，相关玩家收到一条“比赛结束”提醒。
6. 玩家进入可领奖状态后，收到一条“可领奖”提醒，并能跳领奖页。
7. 同一用户同一赛事同一类型消息不会重复推送。
8. TG 发送失败不会导致报名审核、赛事状态流转、结果固化等主流程失败。
9. 未绑定 `telegramUserId` 的用户不会报错，只会跳过发送。

## 视觉稿

本次文案和效果稿已输出到：

- `docs/mockups/2026-04-23-mtt-tg-demo/mtt-tg-demo.png`
- `docs/mockups/2026-04-23-mtt-tg-demo/mtt-tg-copy-compare.png`

其中 V1 采用：

- 玩家侧 6 类消息
- 短文案风格

## 后续扩展

V2 如需扩展，可再考虑：

- 主办方侧 TG 推送
- 领奖成功 / 领奖驳回
- 赛中关键淘汰提醒
- 站内通知自动镜像 Telegram，而不是事件白名单直发
