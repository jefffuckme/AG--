# Club Table Credit Placement Design

**Goal:** Show the current user's club-specific CREDIT directly on each table card, including the "All" list, so players can compare their available club credit against the table's minimum buy-in before tapping in.

## Decision

Use placement option `B`:
- Keep the existing right-side buy-in block as the primary action area
- Add a secondary line directly under the buy-in amount: `额度 <value>`
- Apply this to every CREDIT table card in:
  - the "All" list
  - the single-club table list

This keeps the user's decision path in one place:
- minimum buy-in
- my credit in this club

## Current State

Frontend currently shows club-level balance in the club context bar, but not on each table card:
- club context bar: [yekes-web-javascript/src/views/new/home/components/ClubContextBar.vue](/Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript/src/views/new/home/components/ClubContextBar.vue)
- table card: [yekes-web-javascript/src/components/RoomCard/RoomListRow.vue](/Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript/src/components/RoomCard/RoomListRow.vue)

The club list item already carries a club-specific `balance` field:
- frontend model: [yekes-web-javascript/src/api/model/partyModel.ts](/Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript/src/api/model/partyModel.ts)
- backend VO: [yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/app/game/vo/room/PartyUserRespVO.java](/Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/app/game/vo/room/PartyUserRespVO.java)
- backend query source: [yekes-java/yudao-module-game/yudao-module-game-biz/src/main/resources/mapper/game/ClubMemberMapper.xml](/Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java/yudao-module-game/yudao-module-game-biz/src/main/resources/mapper/game/ClubMemberMapper.xml)

That query reads `play_credit_wallet.balance` by `owner_id + member_id + coin_id='CREDIT'`, so the design assumes `PartyItem.balance` is the correct club-specific CREDIT wallet for the current user.

## Required Consistency Fix

This UI change must not be display-only.

Current home-page table entry still blocks CREDIT entry with a global `creditBalance`:
- [yekes-web-javascript/src/views/new/home/index.vue](/Users/mingge/Documents/IdeaProjects/AG游戏/yekes-web-javascript/src/views/new/home/index.vue)

The chosen design requires:
- table card display uses club-specific CREDIT
- click-to-enter validation uses the same club-specific CREDIT for that table's `ownerId`

Otherwise the UI can say the player has enough credit while click validation rejects them, or the reverse.

## Display Rules

For each table card:
- If `currencyType === 'CREDIT'`, show a secondary credit line below the minimum buy-in amount
- If `currencyType !== 'CREDIT'`, do not show the secondary credit line
- If the club-specific credit is known, render the formatted value
- If the value is temporarily unavailable, render `--`
- Do not silently coerce missing value to `0`

Suggested wording:
- Chinese: `额度 12.5K`
- English fallback: `Credit 12.5K`

## Visual Hierarchy

Maintain existing hierarchy on the card:
- primary: table number and main room identity
- primary action metric: minimum buy-in
- secondary support metric: club-specific CREDIT

Visual treatment:
- buy-in amount remains the largest and brightest value in the right column
- credit line uses smaller text and lower emphasis
- keep it in the same color family as CREDIT, but weaker than the buy-in figure
- do not add a new bottom row or floating corner badge

## Data Flow

Expected data relationship:
- club list items already provide `ownerId -> balance`
- table list items already provide `ownerId`
- table-card display should resolve the table's club-specific balance by `ownerId`

For the "All" list:
- every table card must resolve against its own `ownerId`
- do not reuse the currently selected club's balance for all cards

For the single-club list:
- the resolved balance should be the same number now shown in the club context bar

## Scope

Frontend:
- add club-specific CREDIT display to the table card component
- pass / resolve per-owner credit data for all displayed club tables
- align CREDIT entry validation with the same per-owner value source
- add or update i18n text if a dedicated label is needed

Backend:
- no contract change is required if `PartyUserRespVO.balance` remains club-specific CREDIT
- only revisit backend if frontend inspection finds missing owner-balance mapping for the "All" list path

## Risks

- If the frontend uses a stale club list cache, the card may show an outdated credit line
- If validation still relies on global credit, the UX becomes contradictory
- If missing values are rendered as `0`, users may interpret that as a real balance reset

## Verification

- Frontend: confirm CREDIT tables in "All" show the correct per-club value under minimum buy-in
- Frontend: confirm single-club view shows the same CREDIT in the club header/context and on each table card
- Frontend: confirm USDT tables do not show the extra CREDIT line
- Frontend: confirm tapping a CREDIT table uses the same club-specific balance that the card displays
- Frontend: run `pnpm type-check`
- Frontend: run targeted tests covering room-card rendering and home-page table entry logic
