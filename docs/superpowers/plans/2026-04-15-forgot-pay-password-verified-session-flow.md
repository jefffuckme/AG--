# Forgot Pay Password Verified Session Flow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current one-shot forgot-pay-password reset with a two-phase verified-session flow where `emailCode + googleCode` must be verified first, and only then can the user set a new payment password with a short-lived `resetToken`.

**Architecture:** The backend in `yekes-java` will add a Redis-backed reset session patterned after the existing Google replace session flow, exposing `/pay-password/forget/verify` and `/pay-password/forget/confirm`. The frontend in `yekes-web-javascript` will refactor the dialog into a three-step state machine and switch to the new API contract while clearing sensitive state aggressively.

**Tech Stack:** Vue 3 + TypeScript + Vitest, Java 11 + Spring Boot + JUnit 5, Redis, Maven multi-module.

---

## File Map

**Backend repo:** `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java`

- Modify: `yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/redis/RedisKeyConstants.java`
- Modify: `yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppSecurityController.java`
- Modify: `yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserService.java`
- Modify: `yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java`
- Create: `yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyForgotPayPasswordReqVO.java`
- Create: `yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyForgotPayPasswordRespVO.java`
- Create: `yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppConfirmForgotPayPasswordReqVO.java`
- Create: `yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/dto/AppForgotPayPasswordSessionDTO.java`
- Create: `yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppForgotPayPasswordSessionDTOTest.java`
- Create: `yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppForgotPayPasswordWorkflowTest.java`

**Frontend repo:** `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript`

- Modify: `src/api/security.ts`
- Modify: `src/api/__tests__/security.api.property.test.ts`
- Modify: `src/views/personal/security/components/ForgotPaymentPasswordDialog.vue`
- Modify: `src/views/personal/security/components/forgotPaymentPasswordResetFlow.ts`
- Modify: `src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts`
- Modify: `src/locales/lang/en-US.ts`
- Modify: `src/locales/lang/pt-PT.ts`
- Create: `src/api/__tests__/security.forgotPayPasswordApi.test.ts`

## Task 1: Add Backend Contract Objects And Redis Keys

**Files:**

- Create: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyForgotPayPasswordReqVO.java`
- Create: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyForgotPayPasswordRespVO.java`
- Create: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppConfirmForgotPayPasswordReqVO.java`
- Create: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/dto/AppForgotPayPasswordSessionDTO.java`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/redis/RedisKeyConstants.java`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserService.java`
- Test: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppForgotPayPasswordSessionDTOTest.java`

- [ ] **Step 1: Write the failing contract test**

```java
class AppForgotPayPasswordSessionDTOTest {

    @Test
    void shouldExposeExpectedRedisKeyAndSessionFields() throws Exception {
        assertEquals("app:pay-password:forget:%s:%s", RedisKeyConstants.APP_FORGOT_PAY_PASSWORD_SESSION);
        assertEquals("app:pay-password:forget:active:%s", RedisKeyConstants.APP_FORGOT_PAY_PASSWORD_ACTIVE_TOKEN);

        AppForgotPayPasswordSessionDTO session = new AppForgotPayPasswordSessionDTO();
        session.setMemberId(10001L);
        session.setVerified(true);
        session.setConsumed(false);

        assertEquals(10001L, session.getMemberId());
        assertTrue(session.getVerified());
        assertFalse(session.getConsumed());
    }

    @Test
    void shouldKeepForgotPayPasswordRequestAndResponseFieldNamesAlignedWithApiContract() throws Exception {
        assertHasField(AppVerifyForgotPayPasswordReqVO.class, "emailCode");
        assertHasField(AppVerifyForgotPayPasswordReqVO.class, "googleCode");
        assertHasField(AppVerifyForgotPayPasswordRespVO.class, "resetToken");
        assertHasField(AppConfirmForgotPayPasswordReqVO.class, "resetToken");
        assertHasField(AppConfirmForgotPayPasswordReqVO.class, "newPayPassword");
        assertHasField(AppConfirmForgotPayPasswordReqVO.class, "confirmPayPassword");
    }
}
```

- [ ] **Step 2: Run the targeted backend test to verify it fails**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am test -Dtest=AppForgotPayPasswordSessionDTOTest
```

Expected: FAIL because the new VO classes, DTO, and Redis keys do not exist yet.

- [ ] **Step 3: Add the minimal contract objects and service signatures**

Implement:

- `AppVerifyForgotPayPasswordReqVO` with `emailCode` and `googleCode`
- `AppVerifyForgotPayPasswordRespVO` with `resetToken`
- `AppConfirmForgotPayPasswordReqVO` with `resetToken`, `newPayPassword`, `confirmPayPassword`
- `AppForgotPayPasswordSessionDTO` with `memberId`, `verified`, `consumed`
- `RedisKeyConstants.APP_FORGOT_PAY_PASSWORD_SESSION`
- `RedisKeyConstants.APP_FORGOT_PAY_PASSWORD_ACTIVE_TOKEN`
- `AppUserService.verifyForgotPayPasswordIdentity(...)`
- `AppUserService.confirmForgotPayPasswordReset(...)`

- [ ] **Step 4: Re-run the targeted backend test to verify it passes**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am test -Dtest=AppForgotPayPasswordSessionDTOTest
```

Expected: PASS.

- [ ] **Step 5: Commit the backend contract slice**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java add \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyForgotPayPasswordReqVO.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppVerifyForgotPayPasswordRespVO.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/vo/AppConfirmForgotPayPasswordReqVO.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/dto/AppForgotPayPasswordSessionDTO.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/redis/RedisKeyConstants.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserService.java \
  yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppForgotPayPasswordSessionDTOTest.java
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java commit -m "feat(system): add forgot pay password reset session contracts"
```

## Task 2: Implement Backend Verify Endpoint And Reset Session Issuance

**Files:**

- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppSecurityController.java`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java`
- Test: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppForgotPayPasswordWorkflowTest.java`

- [ ] **Step 1: Write the failing verify-flow test**

```java
@Test
void verifyForgotPayPasswordIdentity_shouldValidateBothFactors_andIssueSingleActiveToken() {
    // Arrange a user with bound email and Google secret.
    // Mock validateMailCode success, checkGoogleCode success.
    // Mock a previous active token existing in Redis.

    AppVerifyForgotPayPasswordRespVO response =
            service.verifyForgotPayPasswordIdentity(new AppVerifyForgotPayPasswordReqVO("123456", "654321"));

    assertNotNull(response.getResetToken());
    verify(redisService).del(String.format(RedisKeyConstants.APP_FORGOT_PAY_PASSWORD_SESSION, memberId, previousToken));
    verify(redisService).set(eq(String.format(RedisKeyConstants.APP_FORGOT_PAY_PASSWORD_ACTIVE_TOKEN, memberId)), eq(response.getResetToken()), anyLong());
}
```

- [ ] **Step 2: Run the targeted backend workflow test and confirm it fails**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am test -Dtest=AppForgotPayPasswordWorkflowTest#verifyForgotPayPasswordIdentity_shouldValidateBothFactors_andIssueSingleActiveToken
```

Expected: FAIL because the method and endpoint behavior do not exist.

- [ ] **Step 3: Implement the verify endpoint and service method**

Add:

- `POST /app-api/system/security/pay-password/forget/verify`
- service method `verifyForgotPayPasswordIdentity`

Implementation rules:

- Load current user
- Require both bound email and bound Google
- Require `emailCode` and `googleCode`
- Reuse existing `validateMailCode` and `checkGoogleCode`
- Invalidate prior active token and prior session if present
- Create a new `resetToken` with 5-minute TTL
- Store `AppForgotPayPasswordSessionDTO(memberId, verified=true, consumed=false)` in Redis
- Return `resetToken`
- Apply the same per-user rate limiter pattern already used on security endpoints

- [ ] **Step 4: Re-run the targeted verify-flow test**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am test -Dtest=AppForgotPayPasswordWorkflowTest#verifyForgotPayPasswordIdentity_shouldValidateBothFactors_andIssueSingleActiveToken
```

Expected: PASS.

- [ ] **Step 5: Commit the backend verify slice**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java add \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppSecurityController.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java \
  yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppForgotPayPasswordWorkflowTest.java
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java commit -m "feat(system): add forgot pay password verify endpoint"
```

## Task 3: Implement Backend Confirm Endpoint And Token Consumption

**Files:**

- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppSecurityController.java`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppForgotPayPasswordWorkflowTest.java`

- [ ] **Step 1: Extend the workflow test with confirm scenarios**

```java
@Test
void confirmForgotPayPasswordReset_shouldRequireValidToken_updatePassword_andCleanupAfterCommit() {
    // Arrange a valid session bound to the current member.
    // Mock Redis active token and session lookup.

    service.confirmForgotPayPasswordReset(new AppConfirmForgotPayPasswordReqVO("token-1", "123456", "123456"));

    verify(appUserMapper).updateById(argThat(user -> user.getPayPassword() != null));
    verify(redisService).del(String.format(RedisKeyConstants.PAY_PASSWORD_FAIL_COUNT, memberId));
}

@Test
void confirmForgotPayPasswordReset_shouldRejectExpiredOrForeignToken() {
    // Arrange token mismatch or missing session.
    // Assert service throws the reset-session-expired error.
}
```

- [ ] **Step 2: Run the targeted confirm tests and confirm they fail**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am test -Dtest=AppForgotPayPasswordWorkflowTest#confirmForgotPayPasswordReset_shouldRequireValidToken_updatePassword_andCleanupAfterCommit,AppForgotPayPasswordWorkflowTest#confirmForgotPayPasswordReset_shouldRejectExpiredOrForeignToken
```

Expected: FAIL because the confirm flow does not exist yet.

- [ ] **Step 3: Implement confirm endpoint and token validation helpers**

Add:

- `POST /app-api/system/security/pay-password/forget/confirm`
- service method `confirmForgotPayPasswordReset`
- helper methods similar to Google replace active-token/session validation

Implementation rules:

- Require `resetToken`, `newPayPassword`, and `confirmPayPassword`
- Ensure the active token matches the requested token
- Ensure the session exists, belongs to the current member, and is not consumed
- Ensure passwords match
- Encode and update the new pay password
- Clear `PAY_PASSWORD_FAIL_COUNT`
- Register after-commit Redis cleanup to delete session and active token
- Return an expired-session style error when token is invalid or expired

- [ ] **Step 4: Run the full backend workflow test class**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am test -Dtest=AppForgotPayPasswordSessionDTOTest,AppForgotPayPasswordWorkflowTest,AppGoogleReplaceSessionDTOTest
```

Expected: PASS.

- [ ] **Step 5: Commit the backend confirm slice**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java add \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppSecurityController.java \
  yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java \
  yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppForgotPayPasswordWorkflowTest.java
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java commit -m "feat(system): add forgot pay password confirm endpoint"
```

## Task 4: Add Frontend API Wrappers And Flow Helpers

**Files:**

- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/api/security.ts`
- Create: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/api/__tests__/security.forgotPayPasswordApi.test.ts`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/api/__tests__/security.api.property.test.ts`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/views/personal/security/components/forgotPaymentPasswordResetFlow.ts`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts`

- [ ] **Step 1: Write the failing frontend API tests**

```ts
it('verifyForgotPaymentPasswordIdentityAPI posts emailCode and googleCode to verify endpoint', async () => {
  vi.mocked(httpClient.post).mockResolvedValue({ resetToken: 'token-1' })
  await verifyForgotPaymentPasswordIdentityAPI({ emailCode: '123456', googleCode: '654321' })
  expect(httpClient.post).toHaveBeenCalledWith({
    url: '/prod-api/app-api/system/security/pay-password/forget/verify',
    data: { emailCode: '123456', googleCode: '654321' }
  })
})

it('confirmForgotPaymentPasswordResetAPI posts resetToken and passwords to confirm endpoint', async () => {
  vi.mocked(httpClient.post).mockResolvedValue(true)
  await confirmForgotPaymentPasswordResetAPI({
    resetToken: 'token-1',
    newPayPassword: '123456',
    confirmPayPassword: '123456'
  })
  expect(httpClient.post).toHaveBeenCalledWith({
    url: '/prod-api/app-api/system/security/pay-password/forget/confirm',
    data: {
      resetToken: 'token-1',
      newPayPassword: '123456',
      confirmPayPassword: '123456'
    }
  })
})
```

- [ ] **Step 2: Run the targeted frontend API and helper tests to verify they fail**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript
pnpm vitest run \
  src/api/__tests__/security.forgotPayPasswordApi.test.ts \
  src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts
```

Expected: FAIL because the new API wrappers and helper logic do not exist.

- [ ] **Step 3: Add the new frontend API wrappers and helper functions**

Implement:

- `verifyForgotPaymentPasswordIdentityAPI`
- `confirmForgotPaymentPasswordResetAPI`
- helper predicates for:
  - can submit verify step
  - can enter password step only when token exists
  - can submit confirm step only when both password entries are valid

Keep the old `forgetPaymentPasswordAPI` exported until the dialog has been migrated, then remove it in the final frontend slice.

- [ ] **Step 4: Re-run the targeted frontend API and helper tests**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript
pnpm vitest run \
  src/api/__tests__/security.forgotPayPasswordApi.test.ts \
  src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit the frontend contract slice**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript add \
  src/api/security.ts \
  src/api/__tests__/security.forgotPayPasswordApi.test.ts \
  src/api/__tests__/security.api.property.test.ts \
  src/views/personal/security/components/forgotPaymentPasswordResetFlow.ts \
  src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript commit -m "feat(security): add forgot pay password verify and confirm apis"
```

## Task 5: Refactor The Frontend Dialog To Verified-First State Machine

**Files:**

- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/views/personal/security/components/ForgotPaymentPasswordDialog.vue`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/views/personal/security/components/forgotPaymentPasswordResetFlow.ts`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts`

- [ ] **Step 1: Extend the flow-helper test for the new state ordering**

```ts
it('only allows password-entry progression after a resetToken exists', () => {
  expect(canEnterForgotPayPasswordSetStep({ resetToken: '' })).toBe(false)
  expect(canEnterForgotPayPasswordSetStep({ resetToken: 'token-1' })).toBe(true)
})

it('clears verify codes after verify success and keeps only resetToken', () => {
  const next = buildForgotPayPasswordVerifiedState({
    emailCode: '123456',
    googleCode: '654321',
    resetToken: 'token-1'
  })
  expect(next.emailCode).toBe('')
  expect(next.googleCode).toBe('')
  expect(next.resetToken).toBe('token-1')
})
```

- [ ] **Step 2: Run the helper tests before touching the dialog**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript
pnpm vitest run src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts
```

Expected: FAIL until the helper behavior is updated.

- [ ] **Step 3: Refactor the dialog**

Change `ForgotPaymentPasswordDialog.vue` so that:

- Step 1 is `verify_identity`
- Verify button calls `verifyForgotPaymentPasswordIdentityAPI`
- Success stores `resetToken`, clears `emailCode` and `googleCode`, then advances to password entry
- Step 2 captures `newPassword`
- Step 3 captures `confirmPassword`
- Final submit calls `confirmForgotPaymentPasswordResetAPI`
- Token-expired errors send the user back to Step 1 and clear the token
- Close and back handlers clear token and password state correctly

- [ ] **Step 4: Run the frontend tests and a build-oriented type check**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript
pnpm vitest run \
  src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts \
  src/api/__tests__/security.forgotPayPasswordApi.test.ts \
  src/api/__tests__/security.googleReplaceApi.test.ts
pnpm type-check
```

Expected: PASS.

- [ ] **Step 5: Commit the dialog refactor**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript add \
  src/views/personal/security/components/ForgotPaymentPasswordDialog.vue \
  src/views/personal/security/components/forgotPaymentPasswordResetFlow.ts \
  src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript commit -m "feat(security): move forgot pay password verification to step one"
```

## Task 6: Update Locale Copy, Remove Old Frontend Usage, And Verify End To End

**Files:**

- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/locales/lang/en-US.ts`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/locales/lang/pt-PT.ts`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/api/security.ts`
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript/src/api/__tests__/security.api.property.test.ts`
- Optional cleanup: keep or deprecate `forgetPaymentPasswordAPI` explicitly, matching the rollout decision

- [ ] **Step 1: Add the failing locale/assertion coverage**

```ts
it('Property forgot-pay-password verify api should always send only verify fields', async () => {
  vi.mocked(httpClient.post).mockResolvedValue({ resetToken: 'token' })
  await verifyForgotPaymentPasswordIdentityAPI({ emailCode: '123456', googleCode: '654321' })
  expect(httpClient.post).toHaveBeenCalledWith({
    url: '/prod-api/app-api/system/security/pay-password/forget/verify',
    data: { emailCode: '123456', googleCode: '654321' }
  })
})
```

- [ ] **Step 2: Update locale strings and remove direct dialog use of old one-shot reset API**

Ensure the visible copy matches:

- Step 1: verify identity
- Step 2: set new password
- Step 3: confirm new password

If `forgetPaymentPasswordAPI` remains exported for compatibility, add a `TODO` comment noting that H5 no longer uses it.

- [ ] **Step 3: Run the final verification suite**

Backend:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am test -Dtest=AppForgotPayPasswordSessionDTOTest,AppForgotPayPasswordWorkflowTest,AppGoogleReplaceSessionDTOTest
```

Frontend:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript
pnpm vitest run \
  src/api/__tests__/security.forgotPayPasswordApi.test.ts \
  src/api/__tests__/security.googleReplaceApi.test.ts \
  src/views/personal/security/components/__tests__/forgotPaymentPasswordResetFlow.test.ts
pnpm type-check
```

Manual verification:

1. Open Security Center
2. Click forgot payment password
3. Confirm Step 1 requires both email code and Google code
4. Confirm Step 2 is unreachable before Step 1 succeeds
5. Confirm successful Step 1 advances without reusing the codes
6. Confirm token expiry returns the dialog to Step 1
7. Confirm successful reset updates the payment password

- [ ] **Step 4: Commit the final frontend polish**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript add \
  src/locales/lang/en-US.ts \
  src/locales/lang/pt-PT.ts \
  src/api/security.ts \
  src/api/__tests__/security.api.property.test.ts
git -C /Users/mingge/Documents/IdeaProjects/AG_H5/yekes-web-javascript commit -m "refactor(security): finalize forgot pay password verified session flow"
```

## Notes For The Implementer

- Follow the Google replace session pattern in `AppUserServiceImpl` closely instead of inventing a new session style.
- Do not remove the backend legacy `/pay-password/forget` endpoint in the same change unless client compatibility has been explicitly confirmed.
- Keep sensitive fields in memory only. Do not persist `emailCode`, `googleCode`, or passwords in stores or Redis.
- If you discover existing mobile or native clients calling the old endpoint, stop and document that dependency before removing or changing compatibility behavior.
