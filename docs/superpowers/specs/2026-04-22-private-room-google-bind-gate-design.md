# 私人房间上分下分 Google 绑定拦截设计

**日期：** 2026-04-22

## 背景

私人房间“操作上分”实际落在 H5 前端的信用调整页，包含上分和下分两类操作。当前页面在提交后直接进入 Google 验证码输入弹层；若用户未绑定 Google，界面没有专门的阻断提示和直达绑定入口。

用户要求：

- 私人房间上分需要 Google 验证
- 如果用户没有绑定 Google，需要在界面上明确提示先去绑定
- 提示必须通过弹窗呈现
- 弹窗必须提供“去绑定”入口
- 上分和下分统一处理
- 点击后直接跳转到 Google 验证绑定页，而不是安全中心首页

## 现状

前端相关位置：

- `yekes-web-javascript/src/views/new/credit-adjust/index.vue`
  当前上分/下分共用该页面与提交流程。
- `yekes-web-javascript/src/stores/security.ts`
  已维护 `securityStatus` 与 `isGoogleAuthEnabled`。
- `yekes-web-javascript/src/router/index.ts`
  已存在 Google 绑定页路由 `securityGoogle`，路径为 `/personal/security/google`。

后端相关位置：

- `yekes-java/yudao-module-game/.../AppPrivateRoomCreditController.java`
  已校验 `googleCode`。
- `yekes-java/yudao-module-system/.../AppUserServiceImpl.java`
  `verifyGoogleCode()` 在“未绑定 Google”时返回 `true`，因此当前后端不会把“未绑定”视为拒绝条件。

## 方案对比

### 方案 A：复用现有 2FA 输入弹窗

未绑定时在原有 Google 验证码区域展示“去绑定”提示。

优点：

- 改动小

缺点：

- 用户先进入验证码弹窗，再发现无法完成操作，流程绕
- 不符合“先提示再引导绑定”的预期

### 方案 B：提交前专用拦截弹窗

在点击上分/下分提交按钮后，先检查 Google 绑定状态。若未绑定，不打开验证码弹窗，直接弹出专用提示弹窗，提供“取消”和“去绑定”两个按钮；点击“去绑定”后跳转到 Google 绑定页。

优点：

- 用户路径最清晰
- 改动范围集中在前端信用调整页
- 满足“弹窗提示 + 去绑定入口 + 直达绑定页”

缺点：

- 仅为前端体验拦截，不改变后端未绑定可绕过的既有行为

### 方案 C：方案 B + 后端强制未绑定拒绝

在方案 B 基础上，后端同时把“未绑定 Google”视为失败。

优点：

- 体验和安全策略一致

缺点：

- 改动跨前后端
- 会改变既有接口行为与业务规则，超出本次已确认范围

## 选定方案

当前选定方案升级为 **方案 C：前端专用拦截弹窗 + 后端强制未绑定拒绝**。

本次修改同时覆盖：

- 前端 `yekes-web-javascript`
- 后端 `yekes-java`

## 详细设计

### 交互流程

1. 用户在信用调整页输入金额，点击“上分”或“下分”提交。
2. 页面先完成既有的金额、玩家、余额等前置校验。
3. 页面拉取或读取最新安全状态。
4. 若 `googleBound=false`：
   - 不进入现有 Google 验证码输入弹层
   - 打开专用提示弹窗
   - 弹窗说明该操作需要先绑定 Google 验证
   - 提供“取消”和“去绑定”
5. 用户点击“去绑定”：
   - 关闭提示弹窗
   - 跳转到 `/personal/security/google`
6. 若 `googleBound=true`：
   - 保持现有流程，继续展示 6 位 Google 验证码输入弹层

### 后端校验流程

1. 信用调整接口继续调用 `appUserApi.verifyGoogleCode(currentUserId, reqVO.getGoogleCode())`。
2. `AppUserServiceImpl.verifyGoogleCode()` 改为：
   - 用户不存在或 `memberId` 为空 → `false`
   - 未绑定 Google → `false`
   - 已绑定但验证码为空 → `false`
   - 已绑定且验证码正确 → `true`
3. `AppPrivateRoomCreditController.adjustCredit()` 在拿到 `false` 时继续沿用现有 `GOOGLE_VERIFICATION_FAILED` 逻辑拒绝请求。

这样即使绕过前端弹窗直接调接口，未绑定用户也无法完成上分/下分。

### 页面状态

在信用调整页新增一个布尔状态，例如：

- `showGoogleBindGateDialog`

该状态只负责“未绑定提醒”弹窗，不与现有 `showTwoFaDialog` 混用。

### 安全状态读取

为避免页面只依赖旧缓存，信用调整页在挂载时主动调用 `securityStore.fetchSecurityStatus()`。

若接口失败：

- 不阻断页面打开
- 继续保留当前页面功能
- 提交时优先使用 store 中现有状态；若无状态且无法确认绑定情况，则仍走现有 2FA 流程，由后续接口返回兜底错误

本次不新增后端接口，也不新增新的错误码。

### 文案

新增一组 `creditAdjust` 文案，至少包括：

- 标题：未绑定 Google 验证
- 说明：上分/下分前需要先绑定 Google 验证
- 按钮：取消
- 按钮：去绑定

### 跳转目标

使用命名路由：

- `router.push({ name: 'securityGoogle' })`

避免硬编码路径字符串散落在组件逻辑中。

## 测试设计

至少覆盖以下前端行为：

1. 未绑定 Google 时，提交不会打开 2FA 输入弹层，而是进入“去绑定”拦截分支。
2. 已绑定 Google 时，提交继续进入现有 2FA 流程。
3. “去绑定”按钮跳转到 `securityGoogle` 路由。
4. 新增文案 key 在主语言文件中存在，避免界面直接显示 key。

建议将“提交前如何决定走绑定拦截还是 2FA 验证”的判断提取为一个小的纯函数，以便做稳定单测。

至少覆盖以下后端行为：

1. `verifyGoogleCode()` 在用户未绑定 Google 时返回 `false`。
2. `verifyGoogleCode()` 在用户已绑定且验证码正确时返回 `true`。
3. 信用调整接口继续依赖该返回值，因此未绑定用户会被统一拒绝。

## 非目标

- 不调整钱包、转账、提币等其它场景的 Google 验证交互
- 不重构现有 2FA 弹层样式

## 风险与备注

- `verifyGoogleCode()` 是跨模块公共校验方法，修改后会同步影响所有复用它的场景。
- 从安全角度这是更一致的行为，但上线前仍应确认这些调用点都接受“未绑定即失败”的策略。
