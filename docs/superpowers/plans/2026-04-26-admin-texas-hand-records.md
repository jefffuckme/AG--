# Admin Texas Hand Records Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade admin `/gaming/texas` hand records so the list shows the richer record summary and the detail action opens a full hand-history view.

**Architecture:** Keep the existing party-room record summary endpoint for the list because it already returns the fields needed by the mock. Add a management-side Texas hand detail endpoint that reuses the private-room hand-history/table-detail data shape but applies admin permissions instead of app participant permissions. The admin frontend calls the new endpoint from the current Texas page and renders summary, community cards, player results, and street actions in one drawer.

**Tech Stack:** Vue 3 + Element Plus + Vitest for admin; Spring MVC + Maven + Java 11 backend.

---

### Task 1: Frontend Contract Tests

**Files:**
- Modify: `yekes-admin-javascript/src/views/game/texasPoker/texasPokerViewModel.ts`
- Test: `yekes-admin-javascript/src/views/game/texasPoker/__tests__/texasPokerViewModel.test.ts`

- [x] Add tests proving `gameNo` is sent as a game number filter and `memberId` stays a player filter.
- [ ] Run the focused Vitest file and verify the new assertions fail before implementation. Blocked: this admin package has no `vitest` binary or dependency.
- [x] Implement the minimal parameter mapping change.
- [ ] Re-run the focused Vitest file. Blocked: this admin package has no `vitest` binary or dependency.

### Task 2: Backend Admin Detail API

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/admin/game/AdminTexasPokerController.java`
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/admin/game/vo/texas/TexasPokerHandDetailRespVO.java`
- Modify service/mapper only if the existing app/private-room logic is not reusable from controller-safe dependencies.

- [x] Add an admin response VO with summary, players, community cards, fairness values, and betting rounds.
- [x] Add `GET /game/texas-poker/hand-detail?roomId=&dockingId=` guarded by existing Texas admin permissions.
- [x] Reuse existing persisted `play_game_round`, `uc_member_betting`, and action-log data; do not call app endpoints from admin.

### Task 3: Admin UI Upgrade

**Files:**
- Modify: `yekes-admin-javascript/src/api/game/texasPoker/index.ts`
- Modify: `yekes-admin-javascript/src/views/game/texasPoker/index.vue`

- [x] Add TypeScript types and API wrapper for the admin hand detail endpoint.
- [x] Update list columns to match the mock: hand info, table/blinds, time, player count, pot/rake, winner/profit, action.
- [x] Replace the small player-profit drawer with a wider full detail drawer.
- [x] Render cards as compact text cards using suit color, avoiding new image dependencies.

### Task 4: Verification

**Commands:**
- `rtk pnpm vitest run src/views/game/texasPoker/__tests__/texasPokerViewModel.test.ts`
- `rtk mvn -pl yudao-module-game/yudao-module-game-biz -DskipTests compile`

- [ ] Run frontend focused tests. Blocked: `rtk pnpm vitest ...` fails because `vitest` is not installed in this package.
- [ ] Run backend compile for the game biz module. Blocked by existing unrelated missing enum constants in credit-withdrawal code.
- [x] Report any command that cannot run due to environment or dependency constraints.
