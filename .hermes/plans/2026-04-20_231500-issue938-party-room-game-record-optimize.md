# Plan: Issue #938 — 私人房间牌局记录页面数据优化

**来源**: https://abcadwiki.org/issues/938  
**优先级**: 紧急  
**指派**: 前端 bcWbE  
**创建时间**: 2026-04-20

---

## 需求概述

私人房间牌局记录页面（yekes-admin-javascript）有 3 个数据问题：

1. **抽水(rake)显示为 ¥0** — 数据未正确从后端取值
2. **玩家数可点击** — 点击后展示参与本局的完整玩家列表
3. **玩家ID列补充玩家名称** — 目前只显示 ID，要同时显示名称

---

## 现状分析

### 问题 1：抽水始终为 ¥0

**根因**（后端 SQL，`RoomMapper.xml` 约第 355 行）：

```sql
ROUND(t.totalBet * 0, 2) as rake
```

rake 被硬编码为 `totalBet * 0`，完全忽略了 `play_game_round` 表中真实的抽水字段：
- `total_rake_amount` — 本局实际总抽水额
- `rake_rate`         — 本局抽水比例
- `rake_skipped`      — 是否跳过抽水

**前端层**：rake 字段正确读取并渲染，无问题。问题完全在 SQL。

---

### 问题 2：玩家数需可点击展示参与玩家列表

**现状**：
- `playerCount` 列只显示数字，无交互
- 详情弹窗（el-dialog）只展示聚合摘要字段，无玩家列表
- 已有 API `getPartyRoomPlayerRecordPage` (`/game/room/partyRoomPlayerRecordPage`)，但其参数是 `ownerId / salesmanId`（业务员维度下钻），**不支持按 gameNo/dockingId 查询指定局的玩家**

**需要新增**：一个按 `gameNo`（或 `dockingId`）查询参与玩家列表的接口。  
数据来源：`uc_member_betting` 表，关联 `uc_member` 获取用户名。

---

### 问题 3：玩家ID列只显示数字，无名称

**现状**：
- `PartyRoomGameRecordRespVO` 已有 `memberName` 字段
- SQL 查询**未填充** `memberName`（无对应 JOIN）
- 前端也只渲染了 `memberId`，`memberName` 完全未使用

**注意**：`memberId/memberName` 此处含义为该条记录对应的"代表玩家"（业务员维度），
并非参与本局所有玩家列表。需确认这里应该显示哪位玩家（房主？下注最多者？胜者？）。  
→ **假设**：与截图对比，"玩家ID"列展示的是胜者（winnerName 已有），
或此处意图是显示业务员归属玩家。**待 PM 确认**。

---

## 改动范围

### 后端（yekes-java）

#### 改动 1：修复 rake SQL

**文件**: `yudao-module-game-biz/src/main/resources/mapper/playroom/RoomMapper.xml`

将：
```sql
ROUND(t.totalBet * 0, 2) as rake
```
改为从 `play_game_round` 读取实际抽水额：
```sql
pgr.total_rake_amount as rake
```
（`pgr` 是已有的 `play_game_round pgr` JOIN）

> 注意：`total_rake_amount` 为本局理论总抽水额，若 `rake_skipped=1` 则该字段可能为 null，
> 需用 `COALESCE(pgr.total_rake_amount, 0)` 兜底。

#### 改动 2：填充 memberName（问题 3，待 PM 确认语义）

选项 A（推荐）：`memberName` 字段改为显示胜者名称（与 winnerName 一致），如与需求不符待确认后再改。  
选项 B：SQL 中补充对 `uc_member` 的 JOIN，按业务员关联逻辑填充 `memberId + memberName`。

**先挂起，不在本 PR 内处理，等 PM 确认语义。**

#### 改动 3：新增"局内玩家列表"接口

**新文件**: `AdminRoomController.java` 新增一个接口

```
GET /game/room/partyRoomGamePlayerList
参数: gameNo (String, required)
返回: List<PartyRoomGamePlayerRespVO>
```

**新 VO**: `PartyRoomGamePlayerRespVO.java`
```java
Long   memberId      // 玩家ID
String memberName    // 玩家名称
BigDecimal betAmount // 本局投注额
BigDecimal profit    // 本局盈亏
String result        // 输/赢
```

**SQL 逻辑**（查 `uc_member_betting` + `uc_member`）:
```sql
SELECT
  mb.member_id,
  um.username  AS memberName,
  mb.betting_amount AS betAmount,
  mb.profit,
  CASE WHEN mb.profit > 0 THEN '赢' ELSE '输' END AS result
FROM uc_member_betting mb
JOIN uc_member um ON um.id = mb.member_id
WHERE mb.game_no = #{gameNo}
ORDER BY mb.profit DESC
```

---

### 前端（yekes-admin-javascript）

#### 改动 1：玩家数列可点击（对应需求 2）

**文件**: `src/views/partyRoom/gameRecord/index.vue`

- 将「玩家数」列的纯文本改为可点击的 `el-button link`
- 点击触发 `handleViewPlayers(row)` — 设置 `playerRow = row`，弹出玩家列表对话框
- 新增 `playerListVisible` ref 和 `playerList` ref
- 弹窗调用新接口 `getPartyRoomGamePlayerList({ gameNo: row.gameNo })`，显示 el-table

**新增玩家列表弹窗列**:
| 列 | 字段 |
|----|------|
| 玩家ID | memberId |
| 玩家名称 | memberName |
| 投注额 | betAmount |
| 盈亏 | profit |
| 结果 | result |

#### 改动 2：玩家ID列补充名称（对应需求 3，待后端确认后实施）

**文件**: `src/views/partyRoom/gameRecord/index.vue`

将「玩家ID」列（Col 8）从：
```html
<span class="member-id">{{ row.memberId }}</span>
```
改为（仿「获胜者」列风格）：
```html
<div>
  <div>{{ row.memberName }}</div>
  <div class="member-id">{{ row.memberId }}</div>
</div>
```

#### 改动 3：新增 API 函数

**文件**: `src/api/game/index.ts`

```typescript
export interface PartyRoomGamePlayerParams {
  gameNo: string
}

export const getPartyRoomGamePlayerList = async (params: PartyRoomGamePlayerParams) => {
  return await request.get({ url: `/game/room/partyRoomGamePlayerList`, params })
}
```

---

## 文件变更清单

| 仓库 | 文件 | 操作 |
|------|------|------|
| yekes-java | `mapper/playroom/RoomMapper.xml` | 修改 rake SQL |
| yekes-java | `controller/admin/game/AdminRoomController.java` | 新增接口 |
| yekes-java | `controller/admin/game/vo/PartyRoomGamePlayerRespVO.java` | 新增 VO |
| yekes-admin-javascript | `src/views/partyRoom/gameRecord/index.vue` | 3 处改动 |
| yekes-admin-javascript | `src/api/game/index.ts` | 新增 API 函数 |

---

## 待确认问题（开工前需 PM 确认）

1. **玩家ID / 玩家名称的语义**：截图第 3 张标注的"玩家名称也加上"，这里的"玩家"指的是哪位玩家？业务员归属的玩家？还是某个特定角色（如房主、胜者）？当前 VO 的 `memberId/memberName` 未被 SQL 填充，语义不明。
2. **点击玩家数展示的列表**：玩家列表弹窗需要展示哪些字段？投注额、盈亏是否必须？是否需要分页？

---

## 执行顺序

1. 后端：修复 rake SQL（改一行，风险最低，优先上）
2. 后端：新增 `partyRoomGamePlayerList` 接口 + VO
3. 前端：新增 `getPartyRoomGamePlayerList` API 函数
4. 前端：玩家数列改为可点击 + 玩家列表弹窗
5. 前端：玩家ID列补充名称（等 PM 确认语义后再做）

---

## 风险与注意事项

- `total_rake_amount` 为理论抽水额，若该局抽水被跳过（`rake_skipped=1`）则值为 null 或 0，用 `COALESCE(..., 0)` 处理
- 玩家列表接口若局内玩家多（>= 10），考虑是否需要分页
- `uc_member_betting` 按 `game_no` 查询需确认有索引，避免慢查询
