# Club Table Credit Placement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show club-specific CREDIT on each private table card and use that same per-club CREDIT for entry validation in the home page table list.

**Architecture:** Keep the backend contract unchanged and treat the frontend club list `PartyItem.balance` as the source of truth for each `ownerId`'s CREDIT wallet. Thread that per-owner balance into transformed table records for display, and eliminate the home page's global CREDIT gate so rendering and click validation consume the same club-specific value.

**Tech Stack:** Vue 3, TypeScript, Vitest, Vite, existing H5 room-card components

---

### Task 1: Define the club-credit source of truth

**Files:**
- Create: `yekes-web-javascript/src/views/new/home/composables/clubCredit.ts`
- Create: `yekes-web-javascript/src/views/new/home/composables/__tests__/clubCredit.test.ts`
- Modify: `yekes-web-javascript/src/api/model/partyModel.ts`
- Modify: `yekes-web-javascript/src/views/new/home/composables/useClubTables.ts`

- [ ] **Step 1: Write the failing helper tests**

Add tests that lock in these behaviors:
- resolve CREDIT by table `ownerId` in the "All" list
- resolve CREDIT for a selected single-club view
- return `null` for unknown CREDIT instead of coercing to `0`
- compare required buy-in against the resolved club CREDIT

- [ ] **Step 2: Run the helper tests to verify they fail**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript
pnpm vitest run src/views/new/home/composables/__tests__/clubCredit.test.ts
```

Expected: FAIL because the helper module does not exist yet.

- [ ] **Step 3: Implement minimal helper + room mapping**

Implement a focused helper that:
- resolves a room's club CREDIT from `ownerId`
- preserves missing values as `null`
- provides a single comparison function for CREDIT sufficiency

Then update the room transformation path so club table rows carry a raw club-credit field such as `clubCreditBalanceRaw`.

- [ ] **Step 4: Re-run the helper tests**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript
pnpm vitest run src/views/new/home/composables/__tests__/clubCredit.test.ts
```

Expected: PASS

- [ ] **Step 5: Commit the helper baseline**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript add \
  src/views/new/home/composables/clubCredit.ts \
  src/views/new/home/composables/__tests__/clubCredit.test.ts \
  src/api/model/partyModel.ts \
  src/views/new/home/composables/useClubTables.ts
git -C /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript commit -m "feat(home): resolve club credit by owner"
```

### Task 2: Render club CREDIT in the table card right column

**Files:**
- Create: `yekes-web-javascript/src/components/RoomCard/__tests__/RoomListRow.creditLine.test.ts`
- Modify: `yekes-web-javascript/src/components/RoomCard/RoomListRow.vue`
- Modify: `yekes-web-javascript/src/components/RoomCard/__tests__/RoomListRow.tableNoOwnerName.test.ts`
- Modify: `yekes-web-javascript/src/components/RoomCard/__tests__/RoomListRow.stageCapsules.test.ts`

- [ ] **Step 1: Write the failing card-render tests**

Add tests that verify:
- CREDIT tables show a secondary line under minimum buy-in
- the line uses the club-specific value carried on the room record
- unknown CREDIT renders `--`
- USDT tables do not render the extra line

- [ ] **Step 2: Run the card tests to verify they fail**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript
pnpm vitest run \
  src/components/RoomCard/__tests__/RoomListRow.creditLine.test.ts \
  src/components/RoomCard/__tests__/RoomListRow.tableNoOwnerName.test.ts \
  src/components/RoomCard/__tests__/RoomListRow.stageCapsules.test.ts
```

Expected: FAIL because the new CREDIT line is not rendered yet.

- [ ] **Step 3: Implement the card UI**

Update `RoomListRow.vue` to:
- accept the transformed club-credit field on the room object
- render the secondary CREDIT line directly below the buy-in amount
- keep buy-in as the primary metric
- hide the line for non-CREDIT currencies

Prefer reusing an existing i18n key such as `newHome.context_credit` if it keeps the label consistent.

- [ ] **Step 4: Re-run the card tests**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript
pnpm vitest run \
  src/components/RoomCard/__tests__/RoomListRow.creditLine.test.ts \
  src/components/RoomCard/__tests__/RoomListRow.tableNoOwnerName.test.ts \
  src/components/RoomCard/__tests__/RoomListRow.stageCapsules.test.ts
```

Expected: PASS

- [ ] **Step 5: Commit the card rendering change**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript add \
  src/components/RoomCard/RoomListRow.vue \
  src/components/RoomCard/__tests__/RoomListRow.creditLine.test.ts \
  src/components/RoomCard/__tests__/RoomListRow.tableNoOwnerName.test.ts \
  src/components/RoomCard/__tests__/RoomListRow.stageCapsules.test.ts
git -C /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript commit -m "feat(home): show club credit on table cards"
```

### Task 3: Align click-to-enter validation with the displayed club CREDIT

**Files:**
- Modify: `yekes-web-javascript/src/views/new/home/index.vue`
- Modify: `yekes-web-javascript/src/views/new/home/composables/useAssets.ts`
- Test: `yekes-web-javascript/src/views/new/home/composables/__tests__/clubCredit.test.ts`

- [ ] **Step 1: Extend the helper tests for entry gating**

Add or refine tests so they cover:
- enough club CREDIT allows entry
- insufficient club CREDIT blocks entry
- missing club CREDIT is treated as unavailable, not silently as zeroed display state

- [ ] **Step 2: Run the helper tests before wiring the page**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript
pnpm vitest run src/views/new/home/composables/__tests__/clubCredit.test.ts
```

Expected: PASS, then proceed to page wiring.

- [ ] **Step 3: Replace the home page global CREDIT gate**

Update `index.vue` so `handleClubTableClick`:
- keeps USDT validation unchanged
- stops relying on the global `creditBalance` value for club tables
- reads the same per-room club CREDIT field used by the card display, or the same helper that generated it

Then reduce `useAssets.ts` home-page responsibility so it no longer drives this private-table CREDIT decision path. Keep it only where its global credit fetch is still truly needed.

- [ ] **Step 4: Run targeted validation tests and smoke checks**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript
pnpm vitest run src/views/new/home/composables/__tests__/clubCredit.test.ts
```

Expected: PASS

Manual smoke checklist:
- in "All", two tables from different clubs show different CREDIT values when expected
- in a selected club view, card CREDIT matches the club context bar CREDIT
- tapping a CREDIT table succeeds/fails according to the value shown on that same card

- [ ] **Step 5: Commit the validation alignment**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript add \
  src/views/new/home/index.vue \
  src/views/new/home/composables/useAssets.ts \
  src/views/new/home/composables/__tests__/clubCredit.test.ts
git -C /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript commit -m "fix(home): align table entry with club credit"
```

### Task 4: Final verification

**Files:**
- Test: `yekes-web-javascript/src/views/new/home/composables/__tests__/clubCredit.test.ts`
- Test: `yekes-web-javascript/src/components/RoomCard/__tests__/RoomListRow.creditLine.test.ts`
- Test: `yekes-web-javascript/src/components/RoomCard/__tests__/RoomListRow.tableNoOwnerName.test.ts`
- Test: `yekes-web-javascript/src/components/RoomCard/__tests__/RoomListRow.stageCapsules.test.ts`

- [ ] **Step 1: Run the targeted frontend tests**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript
pnpm vitest run \
  src/views/new/home/composables/__tests__/clubCredit.test.ts \
  src/components/RoomCard/__tests__/RoomListRow.creditLine.test.ts \
  src/components/RoomCard/__tests__/RoomListRow.tableNoOwnerName.test.ts \
  src/components/RoomCard/__tests__/RoomListRow.stageCapsules.test.ts
```

Expected: PASS

- [ ] **Step 2: Run frontend type-check**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript
pnpm type-check
```

Expected: PASS

- [ ] **Step 3: Manual regression sweep**

Verify:
- CREDIT table cards in "All" display the correct per-owner CREDIT
- USDT table cards remain visually unchanged apart from no extra credit line
- my-room cards are unaffected
- owner/admin editing controls are unaffected

- [ ] **Step 4: Create the final frontend commit**

Run:

```bash
git -C /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript status --short
git -C /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript commit -m "feat(home): surface club credit on private tables"
```

Expected: working tree clean after commit

## Notes

- Backend was checked and already provides club-specific CREDIT through `PartyUserRespVO.balance`; no backend change is planned unless frontend implementation exposes a contract gap.
- The workspace root is not a git repository, so implementation commits should stay scoped to `yekes-web-javascript/`.
- Subagent plan review is intentionally skipped in this session because delegation has not been requested explicitly.
