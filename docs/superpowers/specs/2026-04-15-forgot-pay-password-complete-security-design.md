# Forgot Payment Password Complete Security Design

**Date:** 2026-04-15

**Status:** Approved in conversation, pending written-spec review

**Scope:** `yekes-web-javascript` + `yekes-java`

## Goal

Change "forgot payment password" from a single-step reset into a two-phase secure flow:

1. Verify identity first with both email code and Google code
2. Only after verification succeeds, allow the user to set a new payment password

This must be true in both frontend behavior and backend enforcement.

## Current State

Current implementation is a single request flow:

- Frontend collects new password, confirm password, email code, and Google code in one dialog
- Frontend submits all fields to `/system/security/pay-password/forget`
- Backend validates email and Google codes, then updates pay password directly

Current issues:

- UI order is "set password first, verify later"
- Security semantics do not match the requested behavior
- Google TOTP may expire while the user is entering and confirming the new password
- There is no short-lived verified session proving "identity verification happened first"

## Requirements

- If the user has both email and Google bound, forgot-password reset must require both factors
- Identity verification must happen before the user can enter the new payment password
- Backend must enforce the phase boundary; frontend-only sequencing is not acceptable
- Verified state must be represented by a short-lived, one-time token
- Token must be bound to the current logged-in member
- Token must expire automatically
- Token must be invalidated after successful password reset
- Token must not store raw email code, Google code, or payment password

## Non-Goals

- Do not redesign normal "modify payment password" flow
- Do not generalize all high-risk operations into a single universal verification framework in this task
- Do not change wallet withdrawal second-factor rules
- Do not change "set first payment password" flow in this task

## Chosen Approach

Use a two-phase flow modeled after the existing Google replace session pattern:

1. `verify` endpoint validates `emailCode + googleCode`
2. Backend creates a short-lived `resetToken` in Redis
3. Frontend enters password-setting steps only after receiving `resetToken`
4. `confirm` endpoint validates `resetToken + newPayPassword + confirmPayPassword`
5. Backend updates the pay password and consumes the token

This is the smallest change that provides correct security semantics.

## User Flow

New dialog sequence:

1. Step 1: Verify identity
2. Step 2: Enter new payment password
3. Step 3: Confirm new payment password

Behavior:

- Step 1 requires both email code and Google code
- Step 2 is inaccessible until Step 1 succeeds
- Step 3 compares the two password entries locally before submit
- Final submit sends only `resetToken`, `newPayPassword`, and `confirmPayPassword`
- If token expires, the dialog returns to Step 1
- Closing the dialog clears the token and all sensitive state

## Frontend Design

### Affected Area

- `yekes-web-javascript/src/views/personal/security/components/ForgotPaymentPasswordDialog.vue`
- `yekes-web-javascript/src/api/security.ts`
- `yekes-web-javascript/src/views/personal/security/components/forgotPaymentPasswordResetFlow.ts`
- locale files under `yekes-web-javascript/src/locales/lang/`

### State Machine

Use explicit local states:

- `verify_identity`
- `set_new_password`
- `confirm_new_password`
- `expired`

Local state fields:

- `emailCode`
- `googleCode`
- `resetToken`
- `newPassword`
- `confirmPassword`
- `loading`
- `countdown`

Rules:

- `emailCode` and `googleCode` are only used in `verify_identity`
- After verify success, clear `emailCode` and `googleCode`
- `resetToken` is stored only in component memory, never persisted
- On close, success, or expiry, clear all state

### Frontend API Contract

Add:

- `verifyForgotPaymentPasswordIdentityAPI(data: { emailCode: string; googleCode: string })`
- `confirmForgotPaymentPasswordResetAPI(data: { resetToken: string; newPayPassword: string; confirmPayPassword: string })`

Deprecate frontend use of:

- `forgetPaymentPasswordAPI`

### UX Notes

- Step 1 title should be "验证身份"
- Step 2 title should be "设置新支付密码"
- Step 3 title should be "确认新支付密码"
- If verify succeeds, show no extra success toast; just advance to next step
- If confirm succeeds, close dialog and show reset success toast
- If confirm returns token-expired error, show message and reset to Step 1

## Backend Design

### Affected Area

- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppSecurityController.java`
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserService.java`
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java`
- new request/response VO classes under `.../controller/app/member/vo/`
- new DTO for session state under service DTO package
- Redis key constants in the system module

### New Endpoints

#### 1. Verify identity

`POST /prod-api/app-api/system/security/pay-password/forget/verify`

Request:

```json
{
  "emailCode": "123456",
  "googleCode": "654321"
}
```

Response:

```json
{
  "resetToken": "32-char-random-token"
}
```

Server behavior:

- Load current member
- Ensure member has both bound email and Google secret
- Validate email code against current bound email
- Validate Google code against current bound secret
- Invalidate prior active reset token for this member if present
- Create new short-lived reset session in Redis
- Return `resetToken`

#### 2. Confirm reset

`POST /prod-api/app-api/system/security/pay-password/forget/confirm`

Request:

```json
{
  "resetToken": "32-char-random-token",
  "newPayPassword": "123456",
  "confirmPayPassword": "123456"
}
```

Response:

```json
true
```

Server behavior:

- Validate reset token exists and belongs to current member
- Validate token is active and not consumed
- Validate password format and equality
- Update member pay password
- Clear pay-password fail-count lock key
- Mark token consumed and remove session after commit

### Deprecated Endpoint Strategy

Current endpoint:

- `POST /prod-api/app-api/system/security/pay-password/forget`

Recommended rollout:

- Keep old endpoint temporarily for compatibility
- Migrate H5 frontend to the new verify/confirm pair
- Add server log warning when old endpoint is used
- Remove old endpoint in a later cleanup release after client migration is confirmed

If there are no other clients, direct replacement is possible but has higher rollout risk.

## Redis Session Design

### TTL

- `5 minutes`

Reason:

- Long enough for user interaction
- Short enough to reduce replay window

### Keys

Suggested keys:

- `APP_FORGOT_PAY_PASSWORD_SESSION:%s:%s`
  - `%s = memberId`
  - `%s = resetToken`
- `APP_FORGOT_PAY_PASSWORD_ACTIVE_TOKEN:%s`
  - `%s = memberId`

### Session DTO

Suggested fields:

- `memberId: Long`
- `verified: Boolean`
- `consumed: Boolean`
- `createdAt: LocalDateTime`

Optional fields if needed later:

- `verifiedEmail: String`
- `verificationSource: String`

Current task does not need to store raw codes.

### Session Rules

- Only one active token per member
- New verify invalidates prior token
- Confirm requires token to match active token
- Consumed or missing token is invalid
- Session cleanup should happen after DB commit

## Error Handling

### New Error Cases

- reset session expired
- reset token invalid
- reset token does not belong to current user
- identity verification required before reset

### Frontend Handling

- verify failure: stay on Step 1 and clear only invalid codes
- confirm password mismatch: stay on Step 3 and clear confirm password
- token expired: show toast, clear token, jump back to Step 1
- dialog close: clear all in-memory state

### Backend Handling

- Reuse existing i18n error style
- Add a dedicated error code for forgot-pay-password reset session expired
- Keep responses aligned with the Google replace session style where practical

## Security Considerations

- This change removes the current false sequencing problem
- TOTP is consumed immediately at identity verification time
- Password is never accepted without a verified session token
- Token is scoped to the authenticated member
- Token is single-use and short-lived
- Sensitive verification inputs are not stored in Redis
- Old token invalidation prevents multi-device replay with stale sessions

## Rate Limiting

Apply rate limiting on both endpoints:

- `verify`: strict per-user rate limit
- `confirm`: lighter per-user rate limit

Suggested starting point:

- `verify`: `5 requests / 60 seconds`
- `confirm`: `5 requests / 60 seconds`

If abuse is observed later, tighten `verify` further or add IP-based limiting.

## Data and Contract Impact

### Frontend

- API wrapper changes from one endpoint to two endpoints
- Dialog component logic changes from two-phase local flow to three-phase verified flow
- Locale strings must be updated for step order and validation messages

### Backend

- Controller adds two new endpoints
- Service layer adds two new methods
- New VO classes for verify and confirm
- New Redis key constants
- New session DTO

No database schema change is required.

## Testing Plan

### Frontend Tests

- Verify Step 1 blocks progression until both email and Google code are present
- Verify successful Step 1 stores token and clears verification codes
- Verify Step 2 and Step 3 are inaccessible without token
- Verify confirm submits only `resetToken + newPayPassword + confirmPayPassword`
- Verify token-expired response returns dialog to Step 1
- Verify closing dialog clears token and passwords

### Backend Tests

- Verify endpoint succeeds with valid email and Google code
- Verify endpoint rejects missing email code
- Verify endpoint rejects missing Google code
- Verify endpoint invalidates previous active token
- Confirm endpoint succeeds with valid token and matching passwords
- Confirm endpoint rejects expired token
- Confirm endpoint rejects token from another member
- Confirm endpoint rejects mismatched passwords
- Confirm endpoint consumes token after success
- Confirm endpoint clears pay-password fail-count key after success

### Regression Tests

- Existing set payment password flow still works
- Existing update payment password flow still works
- Existing Google replace flow still works

## Rollout Plan

1. Add backend verify/confirm endpoints and tests
2. Add frontend API wrappers and dialog state-machine changes
3. Switch H5 security center to new flow
4. Observe logs for any old `/pay-password/forget` traffic
5. Remove old endpoint in a later cleanup release if no clients depend on it

## Risks

- If old clients still call `/pay-password/forget`, direct replacement can break them
- Token-expiry UX must be explicit or users will think reset randomly failed
- Frontend step transitions can become inconsistent if token state is not cleared carefully
- Error-code design should stay close to existing Google replace flow to avoid duplicated patterns drifting apart

## Recommendation

Implement the forgot-pay-password reset as a two-phase verified session flow:

- Phase 1: `emailCode + googleCode -> resetToken`
- Phase 2: `resetToken + new password -> update pay password`

This is the most secure approach that stays aligned with current project patterns and avoids unnecessary framework-level generalization.
