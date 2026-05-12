# Google Auth Replace Two-Phase Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current one-shot Google Auth modification flow with a two-phase flow that first verifies old identity using pay password + old Google code, then confirms binding of a new Google secret using a short-lived replacement token.

**Architecture:** The frontend security center flow in `yekes-web-javascript/` will be split into an identity verification step and a new-binding confirmation step coordinated by the parent security page. The backend in `yekes-java/` will expose three app endpoints (`verify`, `prepare`, `confirm`) backed by Redis replacement-session state so that old-code validation and new-code validation no longer compete for the same TOTP window.

**Tech Stack:** Vue 3 + TypeScript + Vite + Vitest, Java + Spring Boot + Maven, Redis, JUnit where applicable.

---

## File Structure

### Frontend files
- Modify: `yekes-web-javascript/src/views/personal/security/index.vue`
  - Orchestrates the two dialogs and owns the active `replaceToken`.
- Create: `yekes-web-javascript/src/views/personal/security/components/VerifyGoogleReplaceIdentityDialog.vue`
  - Step 1 dialog for pay password + old Google code.
- Create: `yekes-web-javascript/src/views/personal/security/components/ConfirmNewGoogleAuthDialog.vue`
  - Step 2 dialog for QR display + new Google code confirmation.
- Retire or slim down: `yekes-web-javascript/src/views/personal/security/components/ModifyGoogleAuthDialog.vue`
  - Either remove usage or turn into a thin wrapper only if needed temporarily.
- Modify: `yekes-web-javascript/src/api/security.ts`
  - Add new two-phase APIs and stop current page flow from using `modifyGoogleAuthAPI`.
- Modify: `yekes-web-javascript/src/api/model/securityModel.ts`
  - Add request/response types for verify/prepare/confirm.
- Create: `yekes-web-javascript/src/views/personal/security/components/googleAuthReplaceFlow.ts`
  - Pure helper state/validation functions to keep dialog logic focused and testable.
- Create: `yekes-web-javascript/src/views/personal/security/components/__tests__/googleAuthReplaceFlow.test.ts`
  - Vitest regression tests for frontend flow helpers.
- Modify: `yekes-web-javascript/src/api/__tests__/security.api.property.test.ts`
  - Add API contract/property assertions for the three new endpoints.

### Backend files
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppSecurityController.java`
  - Add verify/prepare/confirm endpoints.
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserService.java`
  - Add new service method signatures and keep old replace method only if compatibility still requires it.
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java`
  - Implement verify/prepare/confirm logic and Redis lifecycle.
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/redis/RedisKeyConstants.java`
  - Add replacement-session key constant.
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyGoogleReplaceReqVO.java`
  - Request body for old identity verification.
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppPrepareGoogleReplaceReqVO.java`
  - Request body for prepare step.
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppConfirmGoogleReplaceReqVO.java`
  - Request body for confirm step.
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyGoogleReplaceRespVO.java`
  - Response body carrying `replaceToken`.
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/dto/AppGoogleReplaceSessionDTO.java`
  - Redis-serialized replacement session state.
- Create or modify backend tests in the closest existing module test package for `AppUserServiceImpl` and controller-level contract coverage.

### Notes on boundaries
- Frontend dialogs should not own cross-step state beyond their own input fields. The parent page owns `replaceToken` and dialog transitions.
- Backend final confirmation must not trust a client-supplied `newSecret`; the secret must come from the server-side replacement session.

### Task 1: Frontend Flow Helpers And API Contract Types

**Files:**
- Create: `yekes-web-javascript/src/views/personal/security/components/googleAuthReplaceFlow.ts`
- Test: `yekes-web-javascript/src/views/personal/security/components/__tests__/googleAuthReplaceFlow.test.ts`
- Modify: `yekes-web-javascript/src/api/model/securityModel.ts`

- [ ] **Step 1: Write the failing frontend flow helper test**

```ts
import { describe, expect, it } from 'vitest'

import {
  canSubmitGoogleReplaceIdentity,
  canSubmitGoogleReplaceConfirmation,
  shouldResetGoogleReplaceToken
} from '../googleAuthReplaceFlow'

describe('googleAuthReplaceFlow', () => {
  it('requires both pay password and old google code before verify step can submit', () => {
    expect(canSubmitGoogleReplaceIdentity({ payPassword: '123456', oldGoogleCode: '654321' })).toBe(true)
    expect(canSubmitGoogleReplaceIdentity({ payPassword: '', oldGoogleCode: '654321' })).toBe(false)
  })

  it('requires replace token and new code before confirm step can submit', () => {
    expect(canSubmitGoogleReplaceConfirmation({ replaceToken: 'token', newGoogleCode: '123456' })).toBe(true)
    expect(canSubmitGoogleReplaceConfirmation({ replaceToken: '', newGoogleCode: '123456' })).toBe(false)
  })

  it('clears token state when replacement session expires or is cancelled', () => {
    expect(shouldResetGoogleReplaceToken('expired')).toBe(true)
    expect(shouldResetGoogleReplaceToken('cancelled')).toBe(true)
    expect(shouldResetGoogleReplaceToken('success')).toBe(false)
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm vitest run src/views/personal/security/components/__tests__/googleAuthReplaceFlow.test.ts`
Expected: FAIL because `googleAuthReplaceFlow.ts` does not exist yet.

- [ ] **Step 3: Write minimal helper implementation and type additions**

```ts
export function canSubmitGoogleReplaceIdentity(state: { payPassword: string; oldGoogleCode: string }) {
  return /^\d{6}$/.test(state.payPassword) && /^\d{6}$/.test(state.oldGoogleCode)
}

export function canSubmitGoogleReplaceConfirmation(state: { replaceToken: string; newGoogleCode: string }) {
  return !!state.replaceToken && /^\d{6}$/.test(state.newGoogleCode)
}

export function shouldResetGoogleReplaceToken(reason: 'expired' | 'cancelled' | 'success') {
  return reason === 'expired' || reason === 'cancelled'
}
```

Add frontend API model types for:
- `GoogleReplaceVerifyResponse`
- `GoogleReplacePrepareResponse`
- `GoogleReplaceVerifyRequest`
- `GoogleReplacePrepareRequest`
- `GoogleReplaceConfirmRequest`

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm vitest run src/views/personal/security/components/__tests__/googleAuthReplaceFlow.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript add \
  src/views/personal/security/components/googleAuthReplaceFlow.ts \
  src/views/personal/security/components/__tests__/googleAuthReplaceFlow.test.ts \
  src/api/model/securityModel.ts
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript commit -m "test(security): add google replace flow helpers"
```

### Task 2: Frontend API Layer For Verify/Prepare/Confirm

**Files:**
- Modify: `yekes-web-javascript/src/api/security.ts`
- Modify: `yekes-web-javascript/src/api/__tests__/security.api.property.test.ts`

- [ ] **Step 1: Write the failing API property tests**

Add tests asserting:
- `verifyGoogleReplaceIdentityAPI` posts to `/prod-api/app-api/system/security/google/replace/verify`
- `prepareGoogleReplaceAPI` posts to `/prod-api/app-api/system/security/google/replace/prepare`
- `confirmGoogleReplaceAPI` posts to `/prod-api/app-api/system/security/google/replace/confirm`
- verify request body uses `payPassword` and `oldGoogleCode`
- confirm request body does not send `newSecret`

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm vitest run src/api/__tests__/security.api.property.test.ts`
Expected: FAIL because the new APIs are not implemented yet.

- [ ] **Step 3: Implement the minimal API functions**

Add endpoints to `security.ts`:

```ts
verifyGoogleReplaceIdentityAPI(payPassword: string, oldGoogleCode: string)
prepareGoogleReplaceAPI(replaceToken: string)
confirmGoogleReplaceAPI(replaceToken: string, newCode: string)
```

Keep `modifyGoogleAuthAPI` for compatibility, but stop using it from the security page flow.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm vitest run src/api/__tests__/security.api.property.test.ts`
Expected: PASS including the new endpoint assertions.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript add \
  src/api/security.ts \
  src/api/__tests__/security.api.property.test.ts
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript commit -m "feat(security): add two-phase google replace apis"
```

### Task 3: Backend Request/Response VO And Redis Session Model

**Files:**
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyGoogleReplaceReqVO.java`
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppPrepareGoogleReplaceReqVO.java`
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppConfirmGoogleReplaceReqVO.java`
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyGoogleReplaceRespVO.java`
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/dto/AppGoogleReplaceSessionDTO.java`
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/redis/RedisKeyConstants.java`
- Test: backend unit tests near `AppUserServiceImpl` or controller tests for request binding

- [ ] **Step 1: Write the failing backend serialization/binding test**

Create a small test that asserts:
- Redis key constant for replacement sessions exists
- session DTO can represent `memberId`, `newSecret`, `prepared`, `consumed`
- VO field names match spec: `payPassword`, `oldGoogleCode`, `replaceToken`, `newCode`

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java && mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=AppGoogleReplaceSessionDTOTest test`
Expected: FAIL because the new VO/DTO classes and Redis key do not exist.

- [ ] **Step 3: Implement the minimal VO/DTO and Redis key additions**

Add:
- request classes with `@NotBlank`
- response class with `replaceToken`
- DTO for Redis session
- Redis key like `APP_GOOGLE_REPLACE_SESSION = "app:google:replace:%s:%s"`

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java && mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=AppGoogleReplaceSessionDTOTest test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java add \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyGoogleReplaceReqVO.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppPrepareGoogleReplaceReqVO.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppConfirmGoogleReplaceReqVO.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyGoogleReplaceRespVO.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/dto/AppGoogleReplaceSessionDTO.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/redis/RedisKeyConstants.java
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java commit -m "feat(security): add google replace session models"
```

### Task 4: Backend Verify/Prepare/Confirm Service Logic

**Files:**
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserService.java`
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java`
- Test: backend service tests for replace flow lifecycle

- [ ] **Step 1: Write the failing service tests**

Add tests covering:
- invalid pay password cannot issue token
- invalid old Google code cannot issue token
- valid pay password + old Google code issues token
- prepare stores a new secret in replacement session
- confirm rejects expired or unprepared token
- confirm succeeds with token-bound secret and consumes the session

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java && mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=AppUserServiceGoogleReplaceTest test`
Expected: FAIL because the three-phase methods do not exist yet.

- [ ] **Step 3: Implement the minimal service logic**

Implement methods similar to:
- `AppVerifyGoogleReplaceRespVO verifyGoogleReplaceIdentity(AppVerifyGoogleReplaceReqVO reqVO)`
- `AppGoogleSecretRespVO prepareGoogleReplace(AppPrepareGoogleReplaceReqVO reqVO)`
- `void confirmGoogleReplace(AppConfirmGoogleReplaceReqVO reqVO)`

Rules:
- validate current user has existing Google secret
- reuse pay password verification rules from existing pay password verification code
- validate old code against current stored Google secret
- create one-time `replaceToken`
- store session with TTL
- generate and store `newSecret` only in prepare step
- confirm against stored `newSecret`
- on success update user record and delete session state
- optionally clear old `APP_GOOGLE_SECRET` cache to avoid stale cross-flow state

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java && mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=AppUserServiceGoogleReplaceTest test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java add \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserService.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java commit -m "feat(security): split google replace into verify prepare confirm"
```

### Task 5: Backend Controller Endpoints And Request Wiring

**Files:**
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppSecurityController.java`
- Test: controller or API-layer request binding tests for three new endpoints

- [ ] **Step 1: Write the failing controller contract tests**

Add tests asserting:
- `/google/replace/verify` accepts verify request and returns `replaceToken`
- `/google/replace/prepare` accepts `replaceToken` and returns secret payload
- `/google/replace/confirm` accepts `replaceToken` + `newCode`
- existing `/google/replace` is still present only if compatibility requires it

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java && mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=AppSecurityControllerGoogleReplaceTest test`
Expected: FAIL because the endpoints are not yet exposed.

- [ ] **Step 3: Implement the minimal controller wiring**

Add controller methods:
- `verifyGoogleReplace`
- `prepareGoogleReplace`
- `confirmGoogleReplace`

Each should delegate to the corresponding service method and return `success(...)`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java && mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=AppSecurityControllerGoogleReplaceTest test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java add \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppSecurityController.java
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java commit -m "feat(security): expose google replace phase endpoints"
```

### Task 6: Frontend Step 1 Identity Verification Dialog

**Files:**
- Create: `yekes-web-javascript/src/views/personal/security/components/VerifyGoogleReplaceIdentityDialog.vue`
- Modify: `yekes-web-javascript/src/views/personal/security/index.vue`
- Modify: `yekes-web-javascript/src/views/personal/security/components/googleAuthReplaceFlow.ts`
- Test: frontend flow helper tests, plus targeted component behavior test if project setup allows it

- [ ] **Step 1: Write the failing behavior test or helper assertions**

Cover:
- dialog submit remains disabled until pay password and old Google code are both valid
- successful verify emits `verified` with `replaceToken`
- failed verify keeps dialog open

If SFC mount tests are not practical in current project setup, expand helper-level tests and keep the component logic thin.

- [ ] **Step 2: Run test to verify it fails**

Run the narrowest feasible command, for example:
`cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm vitest run src/views/personal/security/components/__tests__/googleAuthReplaceFlow.test.ts`
Expected: FAIL once new helper expectations are added.

- [ ] **Step 3: Implement the minimal dialog and parent wiring for step 1**

Behavior:
- `index.vue` opens `VerifyGoogleReplaceIdentityDialog` instead of `ModifyGoogleAuthDialog` for already-bound Google auth
- dialog calls `verifyGoogleReplaceIdentityAPI`
- on success emits `verified(replaceToken)`
- parent stores token and transitions to step 2 dialog

- [ ] **Step 4: Run test to verify it passes**

Run the same targeted frontend test command and then `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm type-check`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript add \
  src/views/personal/security/components/VerifyGoogleReplaceIdentityDialog.vue \
  src/views/personal/security/index.vue \
  src/views/personal/security/components/googleAuthReplaceFlow.ts
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript commit -m "feat(security): add google replace identity verification step"
```

### Task 7: Frontend Step 2 New Google Binding Confirmation Dialog

**Files:**
- Create: `yekes-web-javascript/src/views/personal/security/components/ConfirmNewGoogleAuthDialog.vue`
- Modify: `yekes-web-javascript/src/views/personal/security/index.vue`
- Retire or stop using: `yekes-web-javascript/src/views/personal/security/components/ModifyGoogleAuthDialog.vue`
- Test: frontend helper tests and any component behavior test that is feasible

- [ ] **Step 1: Write the failing behavior test or helper assertions**

Cover:
- dialog cannot submit without both `replaceToken` and a 6-digit new Google code
- opening the dialog triggers prepare API via parent or component lifecycle
- token expiry forces reset back to step 1
- confirm step never sends `oldGoogleCode` or `newSecret`

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm vitest run src/views/personal/security/components/__tests__/googleAuthReplaceFlow.test.ts`
Expected: FAIL after adding step-2 expectations.

- [ ] **Step 3: Implement the minimal step-2 dialog and transition cleanup**

Behavior:
- on show, call `prepareGoogleReplaceAPI(replaceToken)`
- display returned QR code + secret
- accept only new Google code
- call `confirmGoogleReplaceAPI(replaceToken, newCode)`
- on success clear token, close dialogs, refresh security status
- on expired token clear token and return to step 1 with explicit toast

- [ ] **Step 4: Run test to verify it passes**

Run:
- `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm vitest run src/views/personal/security/components/__tests__/googleAuthReplaceFlow.test.ts`
- `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm type-check`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript add \
  src/views/personal/security/components/ConfirmNewGoogleAuthDialog.vue \
  src/views/personal/security/index.vue \
  src/views/personal/security/components/ModifyGoogleAuthDialog.vue
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript commit -m "feat(security): add google replace confirmation step"
```

### Task 8: Cross-Repo Verification And Cleanup

**Files:**
- Verify both repos only; modify docs only if implementation diverged from spec.

- [ ] **Step 1: Run focused frontend verification**

Run:
- `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm vitest run src/views/personal/security/components/__tests__/googleAuthReplaceFlow.test.ts src/api/__tests__/security.api.property.test.ts`
- `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript && pnpm type-check`
Expected: PASS.

- [ ] **Step 2: Run focused backend verification**

Run:
- `cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java && mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=AppGoogleReplaceSessionDTOTest,AppUserServiceGoogleReplaceTest,AppSecurityControllerGoogleReplaceTest test`
Expected: PASS.

- [ ] **Step 3: Run final contract sanity check**

Manually verify:
- frontend no longer opens old modify dialog directly for bound Google auth
- frontend no longer depends on a client-supplied `newSecret` for final confirmation
- backend final confirmation reads `newSecret` from replacement session, not from request body
- first-time bind flow remains unchanged

- [ ] **Step 4: Commit final cleanup if needed**

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript status --short
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java status --short
```

If cleanup changes were needed, create one final repo-scoped commit with a focused message.

- [ ] **Step 5: Prepare integration summary**

Summarize by repo:
- frontend files changed and new flow entry point
- backend endpoints added and Redis key introduced
- exact verification commands run and results
