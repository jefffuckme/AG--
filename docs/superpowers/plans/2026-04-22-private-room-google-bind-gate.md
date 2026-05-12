# Private Room Google Bind Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a pre-submit Google-binding gate on the private-room credit-adjust page and enforce the same rule on the backend so unbound users are blocked both in the UI and at the API layer.

**Architecture:** Keep the frontend change isolated to the H5 credit-adjust flow while tightening the shared backend Google-verification helper. Refresh security status on page mount, route submit decisions through a small pure helper so the binding gate is testable, and change `AppUserServiceImpl.verifyGoogleCode()` so unbound users fail cross-module verification instead of being silently allowed through.

**Tech Stack:** Vue 3, TypeScript, Pinia, Vue Router, Vant, Vitest, Java 21, JUnit 5, Mockito

---

### Task 1: Add a failing decision test for the bind gate

**Files:**
- Create: `yekes-web-javascript/src/views/new/credit-adjust/__tests__/submitSecurityGate.test.ts`
- Create: `yekes-web-javascript/src/views/new/credit-adjust/submitSecurityGate.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import { resolveSubmitSecurityGate } from '../submitSecurityGate'

describe('resolveSubmitSecurityGate', () => {
  it('returns bind-google when Google is not bound', () => {
    expect(resolveSubmitSecurityGate(false)).toBe('bind-google')
  })

  it('returns enter-2fa when Google is already bound', () => {
    expect(resolveSubmitSecurityGate(true)).toBe('enter-2fa')
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run src/views/new/credit-adjust/__tests__/submitSecurityGate.test.ts`
Expected: FAIL because `submitSecurityGate.ts` does not exist yet

- [ ] **Step 3: Write minimal implementation**

```ts
export type SubmitSecurityGateResult = 'bind-google' | 'enter-2fa'

export const resolveSubmitSecurityGate = (isGoogleBound: boolean): SubmitSecurityGateResult =>
  isGoogleBound ? 'enter-2fa' : 'bind-google'
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm vitest run src/views/new/credit-adjust/__tests__/submitSecurityGate.test.ts`
Expected: PASS


### Task 2: Add a failing i18n key test for the new bind dialog copy

**Files:**
- Create: `yekes-web-javascript/src/views/new/credit-adjust/__tests__/creditAdjustGoogleBindCopy.test.ts`
- Modify: `yekes-web-javascript/src/locales/lang/zh-TW.ts`
- Modify: `yekes-web-javascript/src/locales/lang/en-US.ts`

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import zhTW from '/@/locales/lang/zh-TW'
import enUS from '/@/locales/lang/en-US'

describe('creditAdjust google bind copy', () => {
  it('defines the bind gate copy in zh-TW and en-US', () => {
    expect(zhTW.creditAdjust.google_bind_gate_title).toBeTruthy()
    expect(zhTW.creditAdjust.google_bind_gate_message).toBeTruthy()
    expect(zhTW.creditAdjust.google_bind_gate_action).toBeTruthy()
    expect(enUS.creditAdjust.google_bind_gate_title).toBeTruthy()
    expect(enUS.creditAdjust.google_bind_gate_message).toBeTruthy()
    expect(enUS.creditAdjust.google_bind_gate_action).toBeTruthy()
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run src/views/new/credit-adjust/__tests__/creditAdjustGoogleBindCopy.test.ts`
Expected: FAIL because the copy keys do not exist yet

- [ ] **Step 3: Add minimal locale entries**

Add new `creditAdjust` keys:

```ts
google_bind_gate_title
google_bind_gate_message
google_bind_gate_action
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm vitest run src/views/new/credit-adjust/__tests__/creditAdjustGoogleBindCopy.test.ts`
Expected: PASS


### Task 3: Integrate the bind gate into the credit-adjust page

**Files:**
- Modify: `yekes-web-javascript/src/views/new/credit-adjust/index.vue`
- Modify: `yekes-web-javascript/src/stores/security.ts` (only if page integration reveals a missing helper; otherwise leave untouched)

- [ ] **Step 1: Add a failing page-level behavior test or identify the smallest existing test seam**

If a full component test is too heavy in this repo, keep the seam at the new pure helper from Task 1 and use manual verification for the router/action wiring.

- [ ] **Step 2: Run the relevant focused tests**

Run: `pnpm vitest run src/views/new/credit-adjust/__tests__/submitSecurityGate.test.ts src/views/new/credit-adjust/__tests__/creditAdjustGoogleBindCopy.test.ts`
Expected: PASS before component integration

- [ ] **Step 3: Write minimal implementation in `index.vue`**

Add:

- `useSecurityStore()` and `useRouter()`
- `onMounted` refresh of `fetchSecurityStatus()` alongside existing stats refresh
- local dialog state for the bind gate
- submit branch using `resolveSubmitSecurityGate(securityStore.isGoogleAuthEnabled)`
- `go bind` handler that closes the dialog and routes to `securityGoogle`
- Vant popup/dialog markup for the new gate
- localized title/message/button text

- [ ] **Step 4: Verify the page logic manually and with focused tests**

Run:

```bash
pnpm vitest run src/views/new/credit-adjust/__tests__/submitSecurityGate.test.ts src/views/new/credit-adjust/__tests__/creditAdjustGoogleBindCopy.test.ts
pnpm type-check
```

Expected:

- focused tests PASS
- type-check PASS
- manual page check confirms:
  - unbound user sees bind gate dialog
  - bound user still sees the 2FA popup
  - clicking "go bind" routes to `/personal/security/google`


### Task 4: Final verification

**Files:**
- Modify: `yekes-web-javascript/src/views/new/credit-adjust/index.vue`
- Modify: `yekes-web-javascript/src/views/new/credit-adjust/submitSecurityGate.ts`
- Modify: `yekes-web-javascript/src/views/new/credit-adjust/__tests__/submitSecurityGate.test.ts`
- Modify: `yekes-web-javascript/src/views/new/credit-adjust/__tests__/creditAdjustGoogleBindCopy.test.ts`
- Modify: `yekes-web-javascript/src/locales/lang/zh-TW.ts`
- Modify: `yekes-web-javascript/src/locales/lang/en-US.ts`

- [ ] **Step 1: Run final targeted verification**

Run:

```bash
pnpm vitest run src/views/new/credit-adjust/__tests__/submitSecurityGate.test.ts src/views/new/credit-adjust/__tests__/creditAdjustGoogleBindCopy.test.ts
pnpm type-check
```

Expected: PASS

- [ ] **Step 2: Record non-goals in the handoff**

Document that this plan intentionally reuses the existing backend rejection path in:

- `yekes-java/yudao-module-game/.../AppPrivateRoomCreditController.java`

and changes the shared verification rule in:

- `yekes-java/yudao-module-system/.../AppUserServiceImpl.java`

- [ ] **Step 3: Commit**

```bash
git add yekes-web-javascript/src/views/new/credit-adjust/index.vue \
  yekes-web-javascript/src/views/new/credit-adjust/submitSecurityGate.ts \
  yekes-web-javascript/src/views/new/credit-adjust/__tests__/submitSecurityGate.test.ts \
  yekes-web-javascript/src/views/new/credit-adjust/__tests__/creditAdjustGoogleBindCopy.test.ts \
  yekes-web-javascript/src/locales/lang/zh-TW.ts \
  yekes-web-javascript/src/locales/lang/en-US.ts
git commit -m "feat(credit-adjust): gate unbound users to google bind flow"
```


### Task 5: Add a failing backend verification test

**Files:**
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppUserVerifyGoogleCodeTest.java`

- [ ] **Step 1: Write the failing test**

```java
@Test
void verifyGoogleCode_shouldReturnFalse_whenGoogleNotBound() {
    AppUserDO user = new AppUserDO();
    user.setId(10001L);
    user.setGoogleAuthSecret(null);
    when(appUserMapper.selectById(10001L)).thenReturn(user);

    assertFalse(service.verifyGoogleCode(10001L, "123456"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl yudao-module-system/yudao-module-system-biz -Dtest=AppUserVerifyGoogleCodeTest test`
Expected: FAIL because current implementation returns `true` when Google is not bound

- [ ] **Step 3: Add one passing-path assertion in the same test class**

Add a second test:

```java
@Test
void verifyGoogleCode_shouldReturnTrue_whenCodeMatchesBoundSecret() {
    // mock bound user and GoogleAuthenticatorUtil.check_code(...)
}
```

- [ ] **Step 4: Run test again**

Run: `mvn -pl yudao-module-system/yudao-module-system-biz -Dtest=AppUserVerifyGoogleCodeTest test`
Expected: first test FAILS, second test may pass or error until implementation is updated


### Task 6: Tighten backend verifyGoogleCode behavior

**Files:**
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java`
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/user/AppUserVerifyGoogleCodeTest.java`

- [ ] **Step 1: Write minimal implementation**

Change `verifyGoogleCode()` so:

- blank `memberId` -> `false`
- missing user -> `false`
- blank `googleAuthSecret` -> `false`
- blank `code` -> `false`
- otherwise check the code

- [ ] **Step 2: Update inline comments/Javadoc**

Make implementation comments match the new rule: unbound users must fail verification.

- [ ] **Step 3: Run backend test to verify it passes**

Run: `mvn -pl yudao-module-system/yudao-module-system-biz -Dtest=AppUserVerifyGoogleCodeTest test`
Expected: PASS


### Task 7: Cross-repo verification

**Files:**
- Modify: `yekes-web-javascript/src/views/new/credit-adjust/index.vue`
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java`

- [ ] **Step 1: Run focused frontend verification**

Run:

```bash
cd yekes-web-javascript
pnpm vitest run src/views/new/credit-adjust/__tests__/submitSecurityGate.test.ts src/views/new/credit-adjust/__tests__/creditAdjustGoogleBindCopy.test.ts
pnpm type-check
```

Expected: PASS

- [ ] **Step 2: Run focused backend verification**

Run:

```bash
cd yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -Dtest=AppUserVerifyGoogleCodeTest test
```

Expected: PASS

- [ ] **Step 3: Hand off behavior change explicitly**

Document that `verifyGoogleCode()` is a shared backend helper and now rejects unbound users across all callers, not just private-room credit adjustment.
