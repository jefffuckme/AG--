# 私人房间充提申请统一改造设计

**日期：** 2026-04-22

## 背景

当前俱乐部申请体系里同时存在两套“上分/充值”链路：

- 旧链路：`play_club_application.apply_type=2`
- 新链路：`play_credit_adjust_log.change_type='TOPUP_REQUEST'`

前端展示口径也不统一，页面上同时出现“申请上分”“充值申请”“授权额度”等名称。用户这次确认的新需求是：

- “俱乐部-充值申请”统一改为“充提申请”
- 在现有“上分申请”之外，补齐“下分申请”
- 房主收到下分申请后，可以审核同意或拒绝
- 审核后，需要给申请人发送审核结果通知
- 旧的“申请上分”历史数据仍然要保留展示
- 旧链路不再新增，新申请统一并入新的“充提申请”体系

## 现状

### 前端

- `yekes-web-javascript/src/views/new/privateRooms/components/TopupRequestDialog.vue`
  当前是新链路“充值申请”提交弹窗。
- `yekes-web-javascript/src/views/new/audit-center/index.vue`
  当前待审核页对 `TOPUP_REQUEST` 仍显示“充值申请/授权额度”。
- `yekes-web-javascript/src/views/new/audit-workbench/index.vue`
  当前审核工作台把：
  - `applyType=2` 显示为“上分申请”
  - `applyType=3` 显示为“充值/授权额度”
- `yekes-web-javascript/src/views/new/my-applications/index.vue`
  当前“我的申请”也沿用上述旧枚举口径。
- 多语言文件里保留了大量 `topup/recharge/authorization` 混合文案。

### 后端

- `yekes-java/.../PlayCreditWalletController.java`
  仍保留旧的“申请上分”入口 `applyTopUp`。
- `yekes-java/.../CreditApplicationDelegate.java`
  仍负责旧链路 `play_club_application.apply_type=2` 的创建、审批和通知。
- `yekes-java/.../AppPrivateRoomCreditController.java`
  已有新链路 `TOPUP_REQUEST` 的提交、待审核列表和审批。
- `yekes-java/.../CreditAdjustLogServiceImpl.java`
  目前只支持 `TOPUP_REQUEST`，没有对应的“下分申请”能力。
- `yekes-java/.../ClubApplicationMapper.xml`
  “我的申请”“审核工作台”“历史记录”已经把：
  - 旧 `play_club_application`
  - 新 `play_credit_adjust_log(TOPUP_REQUEST)`
  做了联合查询。

## 关键判断

这次不是简单文案调整，而是把“新单据入口”和“展示口径”统一到一套业务模型下：

- 新申请只走 `play_credit_adjust_log`
- 旧 `play_club_application.apply_type=2` 只保留为历史数据来源
- 新体系下同时支持：
  - `TOPUP_REQUEST` 上分申请
  - `WITHDRAW_REQUEST` 下分申请

## 方案对比

### 方案 A：继续保留双链路，只做文案统一

优点：

- 改动最小

缺点：

- 新旧链路继续并存
- 后续维护成本高
- 无法自然补齐“下分申请”审批链路

### 方案 B：新申请统一进 `play_credit_adjust_log`，旧数据兼容展示

优点：

- 满足本次“统一并入新充提申请”的确认口径
- 不需要迁移历史数据
- 后端新增“下分申请”最自然
- 风险可控

缺点：

- 聚合查询和前端枚举口径需要系统性调整

### 方案 C：彻底迁移旧数据，只保留一张表

优点：

- 模型最干净

缺点：

- 需要历史数据迁移脚本
- 上线与回滚风险高
- 超出本次需求必要范围

## 选定方案

选定 **方案 B：新申请统一进 `play_credit_adjust_log`，旧数据兼容展示**。

## 详细设计

### 1. 业务口径

- 总入口名称统一为“充提申请”
- 业务子类型分为：
  - 上分申请
  - 下分申请
- 历史数据兼容规则：
  - `play_club_application.apply_type=2` 继续显示为“上分申请”
  - 不再写入新数据
- 新数据规则：
  - 上分申请写入 `play_credit_adjust_log.change_type='TOPUP_REQUEST'`
  - 下分申请写入 `play_credit_adjust_log.change_type='WITHDRAW_REQUEST'`

### 2. 单据状态

新旧展示层统一映射为：

- `0 = 待处理`
- `1 = 已同意`
- `2 = 已拒绝`
- `3 = 已过期`

对 `play_credit_adjust_log`：

- `approval_status='PENDING'` -> `0`
- `approval_status='APPROVED'` -> `1`
- `approval_status='REJECTED'` -> `2`

### 3. 下分申请提交流程

1. 玩家选择俱乐部，发起“下分申请”
2. 后端校验：
   - 已加入该俱乐部
   - 存在信用钱包
   - 申请金额大于 0
   - 申请金额不超过当前信用余额
   - 不存在同俱乐部、同用户、同类型的待审核下分申请
3. 创建 `play_credit_adjust_log` 记录：
   - `change_type='WITHDRAW_REQUEST'`
   - `approval_status='PENDING'`
   - `balance_before` 为申请时余额
   - `balance_after` 先等于 `balance_before`
4. 通知房主进入审核

### 4. 下分申请审核流程

审核通过：

1. 校验记录存在、属于当前房主/管理员、状态仍为待审核
2. 先执行真实额度调整：`DECREASE`
3. 调整成功后再把申请单据标记为 `APPROVED`
4. 给申请人发送“下分成功通知”

审核拒绝：

1. 校验记录存在且状态为待审核
2. 不做额度扣减
3. 记录拒绝原因
4. 把状态标记为 `REJECTED`
5. 给申请人发送“下分申请未通过”

### 5. 拒绝原因

当前 `TOPUP_REQUEST` 审核接口没有 `reason`。本次新增统一规则：

- 上分申请拒绝可不强制填写原因
- 下分申请拒绝必须可传 `reason`

数据库层建议新增单独字段：

- `approval_remark`

原因：

- 不污染申请人提交时的 `remark`
- 区分“申请备注”和“审核备注”

### 6. 列表聚合规则

#### 我的申请

统一展示三类来源：

- `play_club_application.apply_type=1` -> 入房申请
- `play_club_application.apply_type=2` -> 历史上分申请
- `play_credit_adjust_log.change_type='TOPUP_REQUEST'` -> 新上分申请
- `play_credit_adjust_log.change_type='WITHDRAW_REQUEST'` -> 新下分申请

#### 审核工作台

房主/管理员视角：

- 待处理列表显示：
  - 入房申请
  - 历史上分申请
  - 新上分申请
  - 新下分申请

申请人视角：

- 待处理和历史记录同样展示以上四类

展示层不再使用“充值申请/授权额度”作为业务名称。

### 7. 前端交互

推荐前端交互调整为：

- “充提申请”下分为两个动作：
  - 上分申请
  - 下分申请
- 审核卡片上明确显示类型标签：
  - 上分申请
  - 下分申请
- 下分申请拒绝时弹出原因输入框
- “我的申请”里同样明确显示申请类型与处理状态

### 8. 通知设计

本次至少补齐新体系站内信模板，不直接复用平台钱包提现模板。

下分申请审核通过：

- 标题：`下分成功通知`
- 内容：
  - `您的下分申请已审核通过。`
  - `下分金额：{amount}`
  - `处理时间：{time}`
  - `请留意到账情况，如有疑问请联系客服。`

下分申请审核拒绝：

- 标题：`下分申请未通过`
- 内容：
  - `您的下分申请未通过审核。`
  - `申请金额：{amount}`
  - `原因：{reason}`
  - `请确认信息无误后重新提交，或联系房主处理。`

同时建议把新上分申请通知文案也统一到“充提申请”体系下，避免界面和通知口径割裂。

## 数据与接口建议

### 数据层

`play_credit_adjust_log` 增补：

- `change_type='WITHDRAW_REQUEST'`
- `approval_remark` 字段

### 接口层

新增或扩展：

- 玩家提交下分申请
- 房主查询待审核下分申请
- 房主审核下分申请
- 审核接口支持 `reason`

## 测试设计

前端至少覆盖：

1. “充提申请”文案替换正确
2. 审核工作台可区分历史上分、新上分、新下分
3. 下分申请拒绝时必须能输入并提交原因
4. “我的申请”中历史旧上分仍可见

后端至少覆盖：

1. 可创建 `WITHDRAW_REQUEST`
2. 重复待审核下分申请会被拦截
3. 下分申请金额超过余额会失败
4. 审核通过时先扣减额度，再改单据状态
5. 审核拒绝时不扣减额度，但会保存原因
6. 聚合查询能同时返回旧上分和新充提申请

## 非目标

- 不做历史数据迁表
- 不下线旧 `play_club_application.apply_type=2` 的历史展示
- 不修改平台钱包提现业务

## 风险与备注

- 旧“申请上分”和新 `TOPUP_REQUEST` 都表示上分申请，列表标题与类型映射必须统一，否则会让用户误以为是两套业务。
- 当前用户说“同意/取消”，但通知文案是“通过/未通过”。实现层建议统一落为“通过/拒绝”，按钮是否展示为“取消”可由前端单独决定。
- 如果后续确认“上分申请拒绝”也必须带原因，则可以复用本次新增的 `approval_remark` 能力，不需要二次改表。
