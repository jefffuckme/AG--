# Room User Count Contract Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace legacy room user-count fields with a breaking-change contract built on `connectedUsers` / `seatedUsers`.

**Architecture:** Rename response contracts at the backend boundary first, then update frontend models and consumers to the new names. Remove all compatibility fallbacks so compile-time and test failures expose every remaining old-field dependency.

**Tech Stack:** Java Spring Boot, Maven, Vue 3, TypeScript, Vitest

---

### Task 1: Frontend tests and helpers

**Files:**
- Modify: `yekes-web-javascript/src/views/new/home/composables/roomPresence.ts`
- Modify: `yekes-web-javascript/src/views/new/home/composables/__tests__/roomPresenceMapping.test.ts`
- Modify: `yekes-web-javascript/src/components/RoomCard/__tests__/RoomListRow.stageCapsules.test.ts`
- Modify: `yekes-web-javascript/src/views/new/public-rooms/__tests__/PublicRooms.seatedCount.test.ts`

- [ ] Rename helper contract to `connectedUsers` / `seatedUsers`
- [ ] Remove fallback logic to old fields in tests
- [ ] Run targeted tests and confirm failures before implementation catches up

### Task 2: Backend contract rename

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/app/game/vo/room/TexasPokerRoomVO.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/app/game/vo/room/MyPrivateRoomRespVO.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/app/game/vo/room/RoomCountDiagnosticsRespVO.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/app/game/vo/room/RoomClusterListRespVO.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/room/TexasPokerRoomQueryService.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/app/game/AppPrivateRoomController.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/controller/app/game/AppPrivateRoomAdminController.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/room/RoomCountDiagnosticsServiceImpl.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/room/RoomAllocationService.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/room/vo/RoomClusterStatistics.java`

- [ ] Rename room fields to `connectedUsers` / `seatedUsers`
- [ ] Rename cluster fields to `totalConnectedUsers` / `totalSeatedUsers`
- [ ] Update every setter and schema string
- [ ] Compile backend module

### Task 3: Frontend contract adoption

**Files:**
- Modify: `yekes-web-javascript/src/api/model/privateRoomModel.ts`
- Modify: `yekes-web-javascript/src/components/RoomCard/RoomListRow.vue`
- Modify: `yekes-web-javascript/src/components/RoomCard/MyRoomCard.vue`
- Modify: `yekes-web-javascript/src/views/new/room-management/index.vue`
- Modify: `yekes-web-javascript/src/views/new/home/composables/useClubTables.ts`
- Modify: `yekes-web-javascript/src/views/new/home/composables/useRoomData.ts`

- [ ] Replace old field names in models and templates
- [ ] Remove all fallback usage
- [ ] Update derived mapping helpers to the new contract only

### Task 4: Verification

**Files:**
- Test: `yekes-web-javascript/src/views/new/home/composables/__tests__/roomPresenceMapping.test.ts`
- Test: `yekes-web-javascript/src/components/RoomCard/__tests__/RoomListRow.stageCapsules.test.ts`
- Test: `yekes-web-javascript/src/views/new/public-rooms/__tests__/PublicRooms.seatedCount.test.ts`

- [ ] Run backend compile
- [ ] Run targeted frontend tests
- [ ] Run frontend type-check
