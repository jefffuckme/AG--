# 好友功能优化需求汇总

> 创建日期：2026-04-13
> 状态：待开发
> 涉及端：前端 H5 (Vue 3) + 后端 (Spring Boot)

---

## 一、需求背景

当前好友体系中存在两个功能缺失：

1. **站内转账缺失入口**：平台已具备完整的 USDT 内部转账能力（`InternalTransferPopup`），但好友列表中没有入口，用户无法直接对好友发起转账。
2. **邀请入桌功能缺失**：玩家在德州扑克游戏中，无法邀请好友加入当前牌桌，且缺少俱乐部关系校验和实时通知机制。

---

## 二、功能一：好友间站内转账

### 2.1 需求描述

在好友列表中为每个好友提供「转账」入口，点击后打开现有的内部转币弹窗（`InternalTransferPopup`），自动预填好友信息。

### 2.2 设计原则

- **纯入口，不新建转账逻辑**：复用现有 `InternalTransferPopup` 组件及其完整的转账流程（余额查询 → 风险确认 → 三因子验证 → 转账结果）
- **零后端改动**：现有 `POST /system/member/internal-transfer` 和 `GET /system/member/lookup-user` 完全满足需求

### 2.3 前端改动

| 文件 | 改动内容 |
|------|----------|
| `components/Friends/OnlinePlayersDrawer.vue` | 好友行 action 区域新增「转账」按钮，点击后打开 `InternalTransferPopup` 并传入 `prefillUserId`、`prefillUsername`、`prefillAvatar` |

### 2.4 后端改动

无。复用现有接口。

### 2.5 交互流程

```
好友列表 → 点击「转账」→ 打开 InternalTransferPopup（预填好友ID/昵称/头像）
→ 输入金额 → 风险确认 → 安全验证（支付密码+Google 2FA+邮箱验证码）
→ 转账成功 → 目标用户收到 MWS 实时通知
```

---

## 三、功能二：邀请好友入桌

### 3.1 需求描述

玩家在德州扑克游戏中，可通过好友列表邀请好友加入当前牌桌。邀请通过 MWS 实时推送给目标用户，对方接受后自动导航至游戏房间。

### 3.2 设计原则

- **邀请不持久化**：纯实时通知，不存储邀请记录
- **无过期机制**：对方收到即可处理
- **H5 端实现 UI**：邀请入口和好友列表均在 H5 宿主页面，游戏 iframe 通过 `postMessage` 触发
- **仅游戏中可邀请**：好友列表的邀请功能仅在玩家处于德州扑克游戏中时可用
- **前后端双重校验**：前端控制按钮状态（灰/绿），后端做权限校验防绕过

### 3.3 按钮状态逻辑

| 条件 | 按钮状态 | 点击行为 |
|------|----------|----------|
| 好友与当前玩家在同一俱乐部（相同 ownerId） | 绿色可点击 | 发送邀请请求 |
| 好友与当前玩家不在同一俱乐部 | 灰色不可点击 | Toast 提示"该玩家与您不在同一俱乐部，无法邀请入桌" |

### 3.4 整体数据流

```
┌─────────────┐  postMessage    ┌──────────────┐  REST API      ┌──────────────┐  MWS push     ┌──────────────┐
│ 游戏iframe    │ ─────────────→ │ H5 宿主页面    │ ─────────────→ │ 后端          │ ─────────────→ │ 目标用户      │
│ (Cocos)      │ invite-friend  │ baccarat.vue │ POST /invite   │ Spring Boot  │ invite-to-    │ H5 页面       │
│              │                │ ↓            │                │ ↓            │ table         │ useSocket.ts │
│              │                │ 打开好友列表   │                │ 校验+推送     │               │ ↓            │
│              │                │ 点击邀请按钮   │                │              │               │ 弹出邀请弹窗   │
└─────────────┘                └──────────────┘                 └──────────────┘               └──────────────┘
```

### 3.5 交互流程

#### 发起邀请方（游戏中的玩家）

```
游戏中 → 游戏iframe发送 postMessage('invite-friend')
→ H5 宿主打开好友列表抽屉（invite 模式）
→ 列表显示好友，每行有「邀请」和「转账」按钮
→ 邀请按钮根据俱乐部关系显示绿色/灰色
→ 点击绿色邀请按钮 → 调用后端 API → 成功 Toast
```

#### 接收邀请方（被邀请的好友）

```
正常使用 H5 页面（任何页面）
→ useSocket.ts 收到 MWS 消息 type="invite-to-table"
→ 全局弹出邀请弹窗：显示邀请人头像、昵称、房间名称
→ 点击「接受邀请」→ 跳转到德州扑克游戏页 { roomId, ownerId, rt: 'private' }
→ 点击「拒绝」→ 关闭弹窗（无需通知后端，不持久化）
```

### 3.6 后端校验链

邀请接口 `POST /game/private-room/table-invite` 的校验顺序：

| 序号 | 校验项 | 方法 | 失败处理 |
|------|--------|------|----------|
| 1 | 两人是否好友 | `friendService.isFriend(userId, targetUserId)` | 返回错误 |
| 2 | 房间是否存在 | 根据 roomId 查询房间，提取 ownerId | 返回错误 |
| 3 | 邀请人是否该俱乐部成员 | `clubMemberService.isMember(ownerId, loginUserId)` | 返回错误 |
| 4 | 目标是否同俱乐部成员 | `clubMemberService.isMember(ownerId, targetUserId)` | 返回错误 |
| 5 | 邀请人是否在房间中 | 可选：查询 ioGame 玩家状态 | 返回错误 |

校验全部通过后，通过 `WebSocketSenderApi` 向目标用户推送 MWS 消息。

### 3.7 MWS 消息格式

```json
{
  "type": "invite-to-table",
  "content": "{\"fromUserId\":123,\"fromUsername\":\"Alice\",\"fromAvatar\":\"https://...\",\"roomId\":456,\"roomName\":\"德州扑克 #456\",\"ownerId\":789}"
}
```

---

## 四、技术要点

### 4.1 关键发现

| 项目 | 结论 |
|------|------|
| 德州扑克页面 | 实际使用 `views/baccarat/index.vue`（非 `game/index.vue`） |
| 房间上下文 | 页面已有 `route.query.roomId`、`route.query.ownerId`、`route.query.rt` |
| 现有 postMessage 通道 | 已有 `recharge`、`mining`、`page-jump` 等消息类型，新增 `invite-friend` 类型即可 |
| 好友列表组件 | `OnlinePlayersDrawer.vue` 目前仅在 `public-rooms` 页面使用 |
| 转账组件 | `InternalTransferPopup.vue` 支持 `prefillUserId/prefillUsername/prefillAvatar` 预填 |
| 俱乐部成员校验 | `ClubMemberService.isMember(ownerId, memberId)` 已有，直接复用 |

### 4.2 可复用的现有模式

| 需要实现 | 最接近的已有模式 | 参考位置 |
|----------|------------------|----------|
| MWS 推送邀请通知 | `credit-authorization` 推送模式 | `CreditWalletServiceImpl.sendAuthorization()` |
| 全局弹窗显示 | `useGlobalDialogs.ts` watch store | `authContent` → `GameAuthForm` |
| 接受/拒绝操作 | `GameAuthForm` 组件 | `components/GameAuthForm/index.vue` |
| 入桌导航 | `navigateToGame()` | `home/index.vue` |
| 俱乐部成员校验 | `ClubMemberService.isMember()` | `ClubMemberServiceImpl` |
| 消息类型常量 | `MessageType.java` | 新增 `INVITE_TO_TABLE` |

---

## 五、改动文件清单

### 5.1 前端

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `views/baccarat/index.vue` | 修改 | 挂载 OnlinePlayersDrawer + InternalTransferPopup，新增 `invite-friend` postMessage 处理 |
| `components/Friends/OnlinePlayersDrawer.vue` | 修改 | 新增 invite 模式 props（roomId, ownerId, mode），好友行增加转账/邀请按钮 |
| `components/Friends/TableInviteNotifyDialog.vue` | **新建** | 邀请入桌通知弹窗（接受/拒绝） |
| `hooks/useSocket.ts` | 修改 | messageHandlers 新增 `invite-to-table` 处理 |
| `stores/app.ts` | 修改 | 新增 `tableInviteNotify` 状态 + setter |
| `composables/useGlobalDialogs.ts` | 修改 | watch `tableInviteNotify` 触发邀请弹窗 |
| `api/party.ts` 或新建 `api/tableInvite.ts` | 修改/新建 | 新增邀请 API `inviteToTableAPI()` |

### 5.2 后端

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `MessageType.java` | 修改 | 新增常量 `INVITE_TO_TABLE = "invite-to-table"` |
| `AppPrivateRoomInviteController.java` | 修改 | 新增 `inviteToTable()` 接口 |
| `TableInviteReqVO.java` | **新建** | 请求 VO：`{ targetUserId, roomId }` |
| `FriendRespVO.java` | 可选修改 | 如需在好友列表返回俱乐部关系，新增 `sameClub` 字段 |
| `FriendServiceImpl.java` | 可选修改 | 如需批量查询俱乐部关系，新增相关方法 |

### 5.3 游戏端（Cocos iframe）

| 改动 | 说明 |
|------|------|
| 新增「邀请好友」按钮 UI | 在游戏内适当位置增加按钮 |
| 发送 postMessage | 点击后 `postMessage({ target: 'yg-game', data: 'invite-friend' })` |

### 5.4 不需要改动的

- **数据库**：无 DDL 变更
- **`InternalTransferPopup.vue`**：完全复用
- **`TransferVerifyDialog.vue`**：完全复用
- **Telegram 通知**：本轮不涉及（邀请不持久化）

---

## 六、API 设计

### 6.1 新增：邀请入桌

```
POST /app-api/game/private-room/table-invite
```

**请求体：**
```json
{
  "targetUserId": 620006,
  "roomId": 12345
}
```

**后端逻辑：**
1. 校验好友关系
2. 根据 roomId 获取 ownerId
3. 校验邀请人是该俱乐部成员
4. 校验目标用户是该俱乐部成员
5. 通过 MWS 推送邀请通知给目标用户

**响应：**
```json
{ "code": 0, "msg": "success" }
```

**错误场景：**

| 错误码 | 说明 |
|--------|------|
| NOT_FRIEND | 两人不是好友 |
| NOT_IN_SAME_CLUB | 目标用户与邀请人不在同一俱乐部 |
| NOT_CLUB_MEMBER | 邀请人不是该俱乐部成员 |
| ROOM_NOT_FOUND | 房间不存在 |

### 6.2 可选：批量查询好友俱乐部关系

```
GET /app-api/game/private-room/member/friends-club-relation?ownerId={ownerId}
```

**响应：**
```json
{
  "code": 0,
  "data": [
    { "friendId": 123, "sameClub": true },
    { "friendId": 456, "sameClub": false }
  ]
}
```

用于好友列表初始化时一次性获取所有好友的俱乐部关系，避免 N+1 查询。

---

## 七、风险与注意事项

| 风险项 | 应对措施 |
|--------|----------|
| 后端校验不可省略 | 即使前端按钮灰态，后端仍须做 `isMember()` 校验 |
| 俱乐部关系前后端一致性 | 前后端均使用 `ClubMemberService.isMember()` 同一数据源 |
| 好友列表性能 | 批量查询俱乐部关系，避免逐个好友调用接口 |
| 目标用户离线 | MWS 推送后对方不在线时，邀请丢失（不持久化，可接受） |
| 邀请方退出房间 | 接受方入桌时后端需再次校验房间状态和邀请方是否仍在房间 |
