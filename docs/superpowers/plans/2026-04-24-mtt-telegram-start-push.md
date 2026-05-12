# MTT Telegram Start Push Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Send MTT Telegram notifications for both "tournament starting" and "tournament started" to approved players and the tournament host.

**Architecture:** Extend the existing MTT Telegram push abstraction with a started-event push type, then update the MTT event dispatcher so `waiting` and `running/late_reg` status transitions include the host in Telegram recipients while preserving the current record-based dedupe and skip logic.

**Tech Stack:** Java 11, Spring Boot, Maven, JUnit 5, Mockito

---

### Task 1: Cover event-dispatch behavior with tests

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttEventSupportTelegramPushTest.java`

- [ ] **Step 1: Write the failing test**

Add assertions that:
- `STATUS_WAITING` pushes to the approved player and the host
- `STATUS_RUNNING` pushes to the approved player and the host
- rejected or pending registrations still do not receive Telegram pushes

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java && rtk mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=MttEventSupportTelegramPushTest test`
Expected: FAIL because the current dispatcher excludes the host and has no started push method

- [ ] **Step 3: Write minimal implementation**

Update the event-dispatch branch in `MttEventSupport` to:
- include host in Telegram recipients for `STATUS_WAITING`
- invoke a new started Telegram push for `STATUS_RUNNING` and `STATUS_LATE_REG`

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java && rtk mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=MttEventSupportTelegramPushTest test`
Expected: PASS


### Task 2: Cover Telegram payload generation with tests

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImplTest.java`

- [ ] **Step 1: Write the failing test**

Add a test asserting `pushTournamentStarted(...)` sends the expected text, button text, and target URL and records `tournament_started` as the push type.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java && rtk mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=MttTelegramPushServiceImplTest test`
Expected: FAIL because the service interface and implementation do not yet expose `pushTournamentStarted`

- [ ] **Step 3: Write minimal implementation**

Update:
- `MttTelegramPushService`
- `MttConstants`
- `MttTelegramPushServiceImpl`

to add the started push type and payload.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java && rtk mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=MttTelegramPushServiceImplTest test`
Expected: PASS


### Task 3: Verify the integrated change set

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttEventSupport.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushService.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImpl.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttConstants.java`
- Test: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttEventSupportTelegramPushTest.java`
- Test: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImplTest.java`

- [ ] **Step 1: Run focused verification**

Run: `cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java && rtk mvn -pl yudao-module-game/yudao-module-game-biz -Dtest=MttEventSupportTelegramPushTest,MttTelegramPushServiceImplTest test`
Expected: PASS

- [ ] **Step 2: Summarize runtime behavior**

Confirm in the final handoff:
- `waiting` now pushes to approved players and host
- `running/late_reg` now pushes to approved players and host
- no change to rejected/pending recipient filtering
- no change to skip conditions such as missing `telegramUserId` or unavailable bot
