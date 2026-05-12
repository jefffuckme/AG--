# 消息中心 Telegram 镜像发送设计

**日期：** 2026-04-22

## 背景

当前项目内已经存在三类通知能力：

- 消息中心站内信：统一落在 `system_notify_message`
- 私人房间 Telegram Bot 通知：部分场景直接由业务服务发 Telegram
- WebSocket / 临时待处理提醒：用于页面实时弹出和业务状态同步

本次需求已经明确为：

- 仅处理“消息中心里能看到的站内信”
- Telegram 发送以 `system_notify_message` 为唯一源
- 只要用户存在 `telegramUserId`，默认自动同步到 Telegram
- Telegram 发送失败只标记失败，不回滚站内信，不做重试

这意味着本次目标不是扩展新的通知中心，而是为现有消息中心增加一个 Telegram 镜像发送旁路。

## 现状

### 消息中心主链路

- `yekes-java/yudao-module-system/.../NotifySendServiceImpl.java`
  负责校验模板并调用消息创建逻辑。
- `yekes-java/yudao-module-system/.../NotifyMessageServiceImpl.java`
  负责写入 `system_notify_message`。
- `yekes-web-javascript/src/api/message.ts`
  前端消息中心读取消息分页、未读数、已读状态。
- `yekes-web-javascript/src/views/new/messages/index.vue`
  消息中心页面。

### Telegram 现有能力

- `yekes-java/yudao-module-game/.../TelegramBotService.java`
  已具备 Telegram 发送文本、发送 WebApp 按钮消息、Webhook 管理能力。
- `yekes-java/yudao-module-game/.../TelegramWebhookController.java`
  已具备 Telegram webhook 接入和 callback 处理能力。
- `AppUserRespDTO.telegramUserId`
  已有用户 Telegram 标识字段，可作为默认发送目标。

### 当前问题

- 同一业务域内部分通知走站内信，部分直接走 Telegram，发送口径分散。
- 消息中心与 Telegram 没有统一镜像关系，无法保证“消息中心可见”的消息自动进入 Telegram。
- Telegram 发送结果没有独立可查询状态，排障只能看日志。

## 方案对比

### 方案 A：业务场景逐个补 Telegram 发送

在私人房间、俱乐部、钱包、好友等业务代码中，逐个场景补发 Telegram。

优点：

- 改动点直观

缺点：

- 继续放大现有分叉
- 容易遗漏场景
- 无法保证和消息中心完全一致

### 方案 B：以 `system_notify_message` 为唯一源统一镜像到 Telegram

站内信成功写入后，统一判断是否需要同步 Telegram，并独立记录发送结果。

优点：

- 覆盖所有消息中心可见消息
- 不改前端消费协议
- 后续新增模板天然继承 Telegram 镜像能力
- 与本次“失败仅标记失败”的要求完全匹配

缺点：

- 需要增加一张发送状态表
- 需要在 system 与 game 模块之间明确 Telegram 调用边界

### 方案 C：消息总线 / Outbox / MQ 异步渠道中心

为站内信再建立统一渠道分发中心，引入 Outbox 或消息队列处理 Telegram。

优点：

- 长期扩展性最好

缺点：

- 对本次范围过重
- 当前没有重试、补偿、多渠道扩展诉求，投入与收益不匹配

## 选定方案

选定 **方案 B：以 `system_notify_message` 为唯一源统一镜像到 Telegram**。

核心原则：

- `system_notify_message` 是唯一通知源
- Telegram 是消息中心的旁路镜像渠道
- 消息中心成功优先于 Telegram 成功
- Telegram 失败不影响主流程
- Telegram 发送状态必须落库可查

## 详细设计

### 架构边界

建议把镜像触发点放在 `NotifyMessageServiceImpl.createNotifyMessage()` 成功写入 `system_notify_message` 之后，而不是放在模板校验或业务调用入口。

原因：

- 只有消息成功落库之后，才符合“以消息中心为唯一源”的语义
- 可以确保 Telegram 只镜像真实存在于消息中心的记录
- 不会把“准备发送”误判成“消息已产生”

建议新增一个 system 模块内的协调服务，例如：

- `NotifyTelegramMirrorService`

它只负责：

- 根据 `notifyMessageId` 发起 Telegram 镜像流程
- 判断该消息是否满足发送条件
- 渲染最终 Telegram 文本
- 调用 Telegram 发送接口
- 记录发送状态

### 模块依赖

system 模块不应直接依赖 game-biz 中的 `TelegramBotService` 实现类。

建议边界：

- 在 `yudao-module-game-api` 中新增 Telegram 发送接口，例如 `TelegramMessageSendApi`
- 接口只暴露最小文本发送能力，例如 `sendMessage(String chatId, String text)`
- 在 `yudao-module-game-biz` 中用现有 `TelegramBotService` 实现该接口
- `NotifyTelegramMirrorService` 只依赖 game-api 中的接口

这样可以避免 system 反向绑定 game-biz 实现，同时复用现有 Telegram 配置、发送和 webhook 底座。

### 数据模型

不建议往 `system_notify_message` 主表直接增加 Telegram 状态字段。

建议新增旁路状态表：

- `system_notify_message_telegram`

建议字段：

- `id`
- `notify_message_id`
- `user_id`
- `telegram_user_id`
- `template_code`
- `status`
- `telegram_message_id`
- `error_code`
- `error_message`
- `sent_content`
- `sent_at`
- `create_time`
- `update_time`

建议状态：

- `PENDING`
- `SUCCESS`
- `FAILED`
- `SKIPPED`

状态定义：

- `PENDING`：站内信已写入，Telegram 镜像已开始但尚未完成
- `SUCCESS`：Telegram API 调用成功
- `FAILED`：已尝试调用 Telegram，但接口失败或抛异常
- `SKIPPED`：明确不发送 Telegram

`SKIPPED` 适用场景建议包括：

- `userType` 不是 MEMBER
- 用户不存在
- 用户没有 `telegramUserId`
- Telegram Bot 未启用
- Telegram Bot 配置不完整

约束建议：

- `notify_message_id` 唯一索引，保证一条消息最多镜像一次
- `user_id, create_time` 普通索引，方便后台筛查

### 发送流程

建议流程如下：

1. 业务调用 `NotifySendService`
2. 模板校验通过
3. `NotifyMessageServiceImpl.createNotifyMessage()` 写入 `system_notify_message`
4. 拿到 `notifyMessageId` 后调用 `NotifyTelegramMirrorService.mirror(notifyMessageId)`
5. 镜像服务先写入一条 `system_notify_message_telegram` 记录，状态为 `PENDING`
6. 读取消息、用户、模板与用户语言
7. 判断是否满足 Telegram 发送条件
8. 不满足则把状态改为 `SKIPPED`
9. 满足则渲染最终文本并调用 `TelegramMessageSendApi.sendMessage(chatId, text)`
10. 成功则更新为 `SUCCESS`
11. 失败则更新为 `FAILED`，记录错误码、错误消息
12. 主流程始终返回站内信创建结果，不因 Telegram 失败而回滚

### 内容生成

V1 不建议单独维护 Telegram 专属模板。

建议直接复用消息中心最终展示文案：

- 站内信标题仍由通知模板解析得到
- 站内信正文仍由模板内容和参数渲染得到
- Telegram 镜像直接拼装最终文本

建议格式：

```text
{title}

{content}
```

原因：

- 可以保证消息中心与 Telegram 文案一致
- 避免同一模板维护两套文案
- 降低后续国际化分叉风险

建议在 system 模块内抽一个公共渲染对象，例如：

- 输入：`NotifyMessageDO`
- 输出：`RenderedNotifyMessage { title, content, lang }`

这样消息中心页面与 Telegram 镜像都复用同一套模板解析逻辑。

### 语言与模板

语言决策继续沿用当前消息中心逻辑：

- MEMBER 用户优先读取用户语言
- 先按用户语言找模板
- 找不到则回退默认语言
- 默认语言仍与当前消息中心保持一致

这样 Telegram 收到的文案与消息中心看到的文案保持一致。

### Telegram 文本安全

现有 Telegram 发送使用 `parse_mode=HTML`。

因此 Telegram 镜像在发送前必须统一做 HTML 转义，至少覆盖：

- `&`
- `<`
- `>`

否则模板渲染后的普通内容可能被 Telegram 当作 HTML 片段解析，导致文本异常或发送失败。

### 失败语义

本次需求明确为：

- 失败只标记失败
- 不回滚消息中心
- 不做重试

因此失败策略固定为：

- 所有 Telegram 发送异常都在镜像服务内部捕获
- 异常不向上抛回站内信主流程
- 失败落 `FAILED`
- 仅记录状态和错误信息，不进入补偿逻辑

V1 不引入：

- 重试任务
- 死信队列
- 后台补发
- 人工重发按钮

### 前端影响

本次不改前端消息中心协议。

前端继续：

- 从 `system_notify_message` 读取消息分页
- 使用已有未读数、已读、详情展示逻辑

Telegram 镜像状态仅用于后端观测和排障，不进入前端消息中心展示。

### 实现清单

后端 `yekes-java`：

- `yudao-module-game-api`
  新增 `TelegramMessageSendApi`
- `yudao-module-game-biz`
  用现有 `TelegramBotService` 实现 `TelegramMessageSendApi`
- `yudao-module-system-biz`
  新增 Telegram 镜像状态表对应的 DO、Mapper、Service
- `NotifyMessageServiceImpl`
  在 `createNotifyMessage()` 成功落库后调用镜像服务
- system 通知模块
  抽取统一消息渲染逻辑
- DB migration
  新增 `system_notify_message_telegram` 表和索引

前端 `yekes-web-javascript`：

- 无需改动

## 测试设计

至少覆盖以下后端测试：

1. MEMBER 用户存在 `telegramUserId` 且 Telegram 发送成功时：
   - `system_notify_message` 创建成功
   - 镜像表创建成功
   - 状态为 `SUCCESS`

2. MEMBER 用户没有 `telegramUserId` 时：
   - `system_notify_message` 创建成功
   - 镜像表状态为 `SKIPPED`

3. 用户类型不是 MEMBER 时：
   - 镜像表状态为 `SKIPPED`

4. Telegram Bot 不可用时：
   - `system_notify_message` 创建成功
   - 镜像表状态为 `SKIPPED`

5. Telegram API 抛异常或返回失败时：
   - `system_notify_message` 创建成功
   - 镜像表状态为 `FAILED`
   - 错误信息被记录

6. 同一 `notify_message_id` 重复触发镜像时：
   - 唯一索引防止重复记录

7. Telegram 文本内容与消息中心渲染结果一致：
   - 标题一致
   - 正文一致
   - HTML 特殊字符正确转义

## 风险与备注

- 如果未来希望 Telegram 带跳转按钮，V1 的纯文本接口可能不够，需要扩展 game-api。
- 当前选择“不重试”，意味着临时网络故障导致的 Telegram 失败不会自动补发。
- 业务侧历史上直接发送 Telegram 的逻辑不会因本次方案自动消失，后续若希望口径完全统一，还需要逐步收敛旧逻辑。
- 由于消息中心只面向站内信，本次不会覆盖 WebSocket 临时提醒和私有待处理接口。

## 非目标

- 不改前端消息中心页面展示
- 不改站内信模板管理后台
- 不为 Telegram 增加单独模板体系
- 不增加发送重试、补发、补偿
- 不覆盖非消息中心来源的通知
