# Google Auth Replace Two-Phase Design

## Background

Current Google Auth replacement flow couples three actions into one submission:
1. generate new Google secret and QR code
2. verify the currently bound Google code
3. verify the new Google code and replace the secret

This creates a timing problem because TOTP codes are short-lived. Users may spend too much time scanning the new QR code and entering the new code, causing the old code or the final request chain to expire.

## Goal

Split Google Auth replacement into two distinct phases:
1. verify old identity first using payment password and old Google code
2. then bind a new Google Auth secret using a short-lived server-issued replace token

This should improve success rate without weakening security.

## Scope

### In scope
- Frontend security center Google Auth modify flow in `yekes-web-javascript/`
- Backend app security controller and user service in `yekes-java/`
- New short-lived replacement session state in Redis
- New API contracts for verify, prepare, and confirm steps
- Frontend UI split into old identity verification and new Google binding confirmation

### Out of scope
- First-time Google Auth binding flow
- Email binding or payment password reset flow changes
- Admin-side Google Auth logic
- Full security challenge framework generalization beyond this feature

## Current State

### Frontend
- `yekes-web-javascript/src/views/personal/security/components/ModifyGoogleAuthDialog.vue`
- On dialog open, frontend immediately calls `generateGoogleAuthAPI()`
- Same dialog collects old Google code and new Google code
- Final submission calls `modifyGoogleAuthAPI(oldCode, secretKey, newCode)`

### Backend
- `yekes-java/.../AppSecurityController.java` exposes `/google/replace`
- `yekes-java/.../AppUserServiceImpl.java#replaceGoogle` verifies old code and new code in one request
- Redis is already used for temporary Google secrets, but current replacement flow still assumes a single-step finalization

## Proposed Design

### High-level flow
1. User taps `修改` for Google Auth in security center
2. Frontend opens `VerifyGoogleReplaceIdentityDialog`
3. User enters payment password and old Google code
4. Frontend calls `POST /google/replace/verify`
5. Backend validates payment password and old Google code, then returns `replaceToken`
6. Frontend opens `ConfirmNewGoogleAuthDialog`
7. Frontend calls `POST /google/replace/prepare` with `replaceToken`
8. Backend generates a replacement-specific new secret and QR code, stores them against the token, and returns the QR payload
9. User scans QR code with the new authenticator and enters the new Google code
10. Frontend calls `POST /google/replace/confirm` with `replaceToken` and `newCode`
11. Backend validates the new code against the token-bound secret, updates the stored Google secret, consumes the token, and clears temp state

## Frontend Design

### UI structure
Use two dialogs managed by the security center page:

1. `VerifyGoogleReplaceIdentityDialog.vue`
- Purpose: verify old identity only
- Inputs:
  - payment password
  - old Google code
- Success output:
  - emits `verified` with `replaceToken`

2. `ConfirmNewGoogleAuthDialog.vue`
- Purpose: bind the new Google Auth only
- Inputs:
  - new Google code
- Displays:
  - QR code
  - secret key
  - copy action
- Depends on:
  - `replaceToken`

### Parent orchestration
In `yekes-web-javascript/src/views/personal/security/index.vue`:
- Clicking `修改` when Google is already bound should open the first dialog
- The page should hold the active `replaceToken`
- On first-step success, close the verification dialog and open the confirmation dialog
- On completion, clear token state and refresh security status
- On cancellation or expiry, clear token state

### Frontend API additions
Add to `yekes-web-javascript/src/api/security.ts`:
- `verifyGoogleReplaceIdentityAPI(payPassword: string, oldGoogleCode: string)`
- `prepareGoogleReplaceAPI(replaceToken: string)`
- `confirmGoogleReplaceAPI(replaceToken: string, newCode: string)`

### Frontend error handling
Expected cases:
- payment password invalid
- old Google code invalid
- replacement session expired
- new Google code invalid
- replacement session already consumed

UX guidance:
- If verify step fails, remain in step 1 and show direct error
- If prepare or confirm returns expired token, close step 2 and return user to step 1 with a message telling them to re-verify

## Backend Design

### New app endpoints
In `AppSecurityController.java` add:
- `POST /google/replace/verify`
- `POST /google/replace/prepare`
- `POST /google/replace/confirm`

### Service responsibilities
In `AppUserService` and `AppUserServiceImpl` split current replacement logic into three methods:

1. `verifyGoogleReplaceIdentity`
- Validate logged-in user exists and has Google bound
- Validate payment password
- Validate old Google code
- Create short-lived `replaceToken`
- Store replacement session in Redis
- Return token

2. `prepareGoogleReplace`
- Validate token belongs to current user and is not expired/consumed
- Generate a new secret specific to this replacement session
- Store the new secret against the replacement session
- Return secret, URL, and link

3. `confirmGoogleReplace`
- Validate token belongs to current user and is not expired/consumed
- Load the token-bound new secret
- Validate `newCode` against the token-bound secret
- Update user `googleAuthSecret`
- Mark session consumed and delete temp state
- Clear any old generic Google secret cache if present

### Redis state
Use a replacement-specific key instead of relying only on the generic Google secret key.

Recommended key pattern:
- `app:google:replace:{memberId}:{token}`

Stored state should include at least:
- `memberId`
- `verified=true`
- `newSecret` when prepared
- `prepared=true/false`
- `createdAt` or TTL-based expiration
- `consumed=false/true`

### TTL
Recommended replacement session TTL:
- 2 to 5 minutes total

### Backward compatibility
- Keep existing `/google/replace` endpoint temporarily for compatibility if needed
- Stop using it from the current frontend
- Keep first-time bind flow unchanged: `/google/secret` + `/google/bind`

## Security Considerations

- Do not trust client-provided `newSecret` for final replacement confirmation
- Bind secret generation to the server-side replacement session token
- Tokens must be user-scoped and one-time use
- Expired or consumed tokens must be rejected consistently
- Replacement confirmation must require both valid token and valid new Google code

## Data Contract Proposal

### Verify request
```json
{
  "payPassword": "123456",
  "oldGoogleCode": "654321"
}
```

### Verify response
```json
{
  "replaceToken": "token-value"
}
```

### Prepare request
```json
{
  "replaceToken": "token-value"
}
```

### Prepare response
```json
{
  "secret": "ABCDEF",
  "url": "otpauth://totp/...",
  "link": "user@issuer"
}
```

### Confirm request
```json
{
  "replaceToken": "token-value",
  "newCode": "123456"
}
```

### Confirm response
```json
true
```

## Testing Strategy

### Frontend
- Verify step 1 blocks step 2 until validation succeeds
- Verify step 2 only asks for the new Google code
- Verify expired token returns user to step 1
- Verify success refreshes security status and closes dialogs

### Backend
- Invalid payment password cannot issue token
- Invalid old Google code cannot issue token
- Expired token cannot prepare or confirm
- Confirm cannot succeed before prepare
- Confirm succeeds only with the token-bound secret
- Confirm consumes token and replaces stored Google secret

## Risks

### Main risks
- Token and temp secret lifecycle bugs in Redis
- Concurrent replacement sessions for the same user
- Reusing generic Google secret generation code without enough isolation

### Mitigations
- Replacement-specific Redis key namespace
- One active replacement session per user, or overwrite policy defined explicitly
- Explicit cleanup on success and expiry

## Recommended Implementation Direction

Use a lightweight two-phase replacement design:
- split frontend into old identity verification + new binding confirmation
- split backend into verify + prepare + confirm
- keep first-time bind unchanged
- keep old replace endpoint temporarily, but stop using it in the frontend
