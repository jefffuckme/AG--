# Private Room Credit Withdraw Apply Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Unify private-room top-up and withdraw requests under the new “credit apply” flow, keep historical legacy top-up applications visible, add a new withdraw-request approval chain, and send result notifications back to the applicant.

**Architecture:** Stop creating new legacy `play_club_application.apply_type=2` records and route all new top-up/withdraw requests through `play_credit_adjust_log`. Extend the aggregated query layer so old top-up history and new request records can coexist in the same UI, add an explicit approval remark field for rejection reasons, and keep balance mutation tied to approval success rather than request creation.

**Tech Stack:** Vue 3, TypeScript, Vite, Vitest, Java, Spring Boot, MyBatis XML mapper, JUnit 5, Mockito, Flyway SQL

---

### Task 1: Add the backend schema support for withdraw requests and approval remarks

**Files:**
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/resources/db/migration/2026xxxx__private_room_credit_withdraw_request.sql`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/dal/dataobject/game/CreditAdjustLogDO.java`

- [ ] **Step 1: Write the migration**

Add:

- `approval_remark` column to `play_credit_adjust_log`
- comment updates documenting `WITHDRAW_REQUEST`

- [ ] **Step 2: Run targeted validation**

Check that the migration is syntactically valid and the DO includes:

- `TOPUP_REQUEST`
- `WITHDRAW_REQUEST`
- `approvalRemark`

- [ ] **Step 3: Update the DO**

Expose the new field and refresh comments so future code reads the model correctly.


### Task 2: Add failing backend tests for withdraw request creation and approval

**Files:**
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/.../AppPrivateRoomCreditControllerWithdrawRequestTest.java`
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/.../CreditAdjustLogServiceWithdrawRequestTest.java`

- [ ] **Step 1: Write a failing service test for request creation**

Cover:

- create `WITHDRAW_REQUEST`
- duplicate pending request rejected
- amount greater than balance rejected

- [ ] **Step 2: Write a failing controller test for approval**

Cover:

- approve -> calls `DECREASE`, then marks request approved
- reject -> marks request rejected and stores `approvalRemark`

- [ ] **Step 3: Run focused backend tests**

Run: `mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=AppPrivateRoomCreditControllerWithdrawRequestTest,CreditAdjustLogServiceWithdrawRequestTest test`

Expected: FAIL before implementation


### Task 3: Extend the credit-adjust-log service for withdraw requests

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/.../CreditAdjustLogService.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/.../CreditAdjustLogServiceImpl.java`

- [ ] **Step 1: Add service methods**

Add methods for:

- `hasPendingWithdrawRequest(ownerId, memberId)`
- `createWithdrawRequest(ownerId, memberId, amount, currentBalance, remark)`
- `getPendingWithdrawRequests(ownerId)`
- shared approval status update with `approvalRemark`

- [ ] **Step 2: Implement balance guardrails**

Reject request creation when:

- wallet not found
- amount is non-positive
- amount exceeds current balance

- [ ] **Step 3: Re-run focused tests**

Run: `mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=CreditAdjustLogServiceWithdrawRequestTest test`

Expected: PASS


### Task 4: Add app APIs for submit/list/approve withdraw requests

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/.../AppPrivateRoomCreditController.java`
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/.../vo/privateroom/MemberWithdrawRequestReqVO.java`
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/.../vo/privateroom/WithdrawRequestApproveReqVO.java`
- Modify: `yekes-java/yudao-framework/yudao-common/src/main/java/.../I18nCodeEnum.java`

- [ ] **Step 1: Add the request VOs**

Include:

- `ownerId`
- `amount`
- optional `remark`
- reject path `reason`

- [ ] **Step 2: Implement submit endpoint**

Add endpoint parallel to top-up request submit:

- validate membership and balance
- create `WITHDRAW_REQUEST`
- notify room owner

- [ ] **Step 3: Implement pending-list endpoint**

Return pending withdraw requests for the owner/admin view.

- [ ] **Step 4: Implement approval endpoint**

Approve path:

- run `DECREASE`
- update request as approved
- send success notification

Reject path:

- update request as rejected
- persist `approvalRemark`
- send rejected notification

- [ ] **Step 5: Re-run focused backend tests**

Run: `mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=AppPrivateRoomCreditControllerWithdrawRequestTest test`

Expected: PASS


### Task 5: Unify aggregated query output for legacy and new request records

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/resources/mapper/clubapplication/ClubApplicationMapper.xml`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/.../vo/room/MyApplicationRespVO.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/.../vo/room/AuditWorkbenchRecordRespVO.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/.../vo/room/PendingApplicationRespVO.java`

- [ ] **Step 1: Add failing mapper assertions if mapper tests exist**

If XML mapper tests exist, add fixtures for:

- legacy `apply_type=2`
- new `TOPUP_REQUEST`
- new `WITHDRAW_REQUEST`

If not, document manual SQL verification and cover through controller/service integration tests.

- [ ] **Step 2: Extend union SQL**

Map:

- legacy `apply_type=2` -> top-up apply type
- `TOPUP_REQUEST` -> new top-up apply type
- `WITHDRAW_REQUEST` -> withdraw apply type

- [ ] **Step 3: Update VO comments and enums**

Replace old “充值/授权额度” wording with explicit “上分申请/下分申请” wording.

- [ ] **Step 4: Verify ordering and status mapping**

Ensure:

- pending view still sorts by `create_time`
- history view sorts by `processed_time`
- reject reason is available to the frontend when needed


### Task 6: Add new notify templates and senders for withdraw approval results

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/.../service/notification/PrivateRoomNotificationService.java`
- Create or modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/resources/db/migration/2026xxxx__private_room_credit_request_notify_templates.sql`

- [ ] **Step 1: Add template rows**

At minimum add:

- owner receives pending withdraw request
- member receives withdraw approved
- member receives withdraw rejected

- [ ] **Step 2: Implement sender methods**

Add dedicated methods rather than overloading wallet-withdraw template codes.

- [ ] **Step 3: Wire the methods into submit/approve/reject flows**

Keep notification failures non-blocking, but log them clearly.


### Task 7: Add frontend API types and request wrappers

**Files:**
- Modify: `yekes-web-javascript/src/api/privateRoom.ts`
- Modify: `yekes-web-javascript/src/api/model/playerModel.ts`

- [ ] **Step 1: Add API wrappers**

Add:

- submit withdraw request
- fetch pending withdraw request list
- approve/reject withdraw request

- [ ] **Step 2: Update shared models**

Expand application types so the frontend can distinguish:

- join apply
- legacy top-up apply
- new top-up request
- new withdraw request

- [ ] **Step 3: Run type-check**

Run: `pnpm type-check`

Expected: FAIL before all calling code is updated


### Task 8: Update the frontend workbench and my-applications UI

**Files:**
- Modify: `yekes-web-javascript/src/views/new/privateRooms/components/TopupRequestDialog.vue`
- Modify: `yekes-web-javascript/src/views/new/audit-center/index.vue`
- Modify: `yekes-web-javascript/src/views/new/audit-workbench/index.vue`
- Modify: `yekes-web-javascript/src/views/new/my-applications/index.vue`

- [ ] **Step 1: Change the user-facing labels**

Replace:

- “充值申请”
- “授权额度”

with unified “充提申请” entry wording and explicit request-type labels inside cards.

- [ ] **Step 2: Add withdraw-request action path**

Users must be able to submit withdraw requests and room owners must be able to review them.

- [ ] **Step 3: Add reject-reason UI**

In owner review flow for withdraw requests:

- prompt for reason on reject
- submit reason to backend

- [ ] **Step 4: Keep legacy top-up history visible**

Do not hide old `apply_type=2` rows; only stop using them as new-entry writes.

- [ ] **Step 5: Run frontend verification**

Run:

```bash
pnpm type-check
pnpm vitest run
```

Expected: targeted UI/type tests PASS


### Task 9: Update locale copy

**Files:**
- Modify: `yekes-web-javascript/src/locales/lang/zh-TW.ts`
- Modify: `yekes-web-javascript/src/locales/lang/en-US.ts`
- Modify: other locale files touched by the current feature wording

- [ ] **Step 1: Replace mixed wording**

Normalize copy for:

- 充提申请
- 上分申请
- 下分申请
- 下分成功通知
- 下分申请未通过

- [ ] **Step 2: Preserve semantic distinctions**

Do not collapse all labels to one string; keep:

- entry label
- card subtype
- notification title
- approval action copy


### Task 10: End-to-end verification

**Files:**
- Modify: all files changed above

- [ ] **Step 1: Run backend verification**

Run:

```bash
cd yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=AppPrivateRoomCreditControllerWithdrawRequestTest,CreditAdjustLogServiceWithdrawRequestTest test
```

- [ ] **Step 2: Run frontend verification**

Run:

```bash
cd yekes-web-javascript
pnpm type-check
pnpm vitest run
```

- [ ] **Step 3: Manual business verification**

Verify:

1. player can submit top-up request
2. player can submit withdraw request
3. owner sees both request types in review list
4. owner approve withdraw -> credit decreases
5. owner reject withdraw -> no balance change, reason persists
6. applicant receives result notification
7. old top-up history remains visible

- [ ] **Step 4: Commit**

Suggested split:

```bash
git add yekes-java
git commit -m "feat(private-room): add withdraw request approval flow"

git add yekes-web-javascript
git commit -m "feat(private-room): unify credit request ui"
```
