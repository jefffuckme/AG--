# MTT Player Telegram Push Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add player-side Telegram push delivery for 6 approved MTT notification types using short-copy templates, without affecting existing MTT business flows when Telegram sending fails.

**Architecture:** Reuse the existing MTT event/notification trigger points in `MttEventSupport`, but route Telegram delivery through a dedicated `MttTelegramPushService` that enforces the V1 whitelist, renders short-copy text, resolves the target Mini App URL, and records per-user per-tournament per-type delivery for deduplication. Keep existing in-app `mtt_notification` creation unchanged; Telegram is an additional side-channel in the game module, not a replacement for current inbox or WebSocket paths.

**Tech Stack:** Java 11, Spring Boot, MyBatis-Plus, existing Telegram Bot service, JUnit 5, Mockito, Maven multi-module build.

---

## File Map

### Existing Files To Modify

- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttEventSupport.java`
  Hook Telegram side-delivery onto the already existing MTT event notification points for the approved 6 message types only.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttConstants.java`
  Add or reuse Telegram-specific constants for supported message types and button actions if keeping them centralized improves clarity.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/telegram/TelegramBotService.java`
  Reuse existing text/WebApp button send methods; only modify if a small helper extraction is needed.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/resources/application-dev.yaml`
  Confirm the `telegram.bot.web-app-url` comment or example matches the real `/app-api/game/telegram/webhook` + Mini App expectations if the current placeholder is misleading.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/resources/application-prod.yaml`
  Keep prod comments aligned with the same behavior and avoid future `BUTTON_URL_INVALID` regressions.

### New Files To Create

- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/dal/dataobject/mttv2/MttTelegramPushRecordDO.java`
  Persist deduped send attempts/results for MTT Telegram push.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/dal/mysql/mttv2/MttTelegramPushRecordMapper.java`
  Mapper for insert/select-by-dedup-key/update-result operations.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushService.java`
  Service contract for V1 Telegram push orchestration.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImpl.java`
  Main orchestration service: whitelist, dedup, copy rendering, button target, Telegram send, result record.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImplTest.java`
  Unit tests for whitelist, copy rendering, dedup skip, success, and failure-not-blocking behavior.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttEventSupportTelegramPushTest.java`
  Focused tests proving the 6 selected MTT events trigger Telegram push and excluded events do not.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/resources/db/migration/V1.1.12__add_mtt_telegram_push_record.sql`
  Migration for the Telegram push record table and unique dedup index.

### Optional Existing Files To Inspect During Implementation

- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttNotificationServiceImpl.java`
  Reuse current action-target resolution patterns for `/m/{mttCode}`, `/m/wait/{mttCode}`, and `/m/rc?mttId={mttCode}`.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/config/TelegramConfig.java`
  Reuse existing Telegram config accessors for base Mini App URL and availability checks.
- `yekes-java/yudao-module-system/yudao-module-system-api/src/main/java/cn/iocoder/yudao/module/system/api/user/AppUserApi.java`
  Resolve player `telegramUserId` from the existing user contract rather than duplicating lookup logic.

## Task 1: Create Telegram Push Record Storage And Dedup Boundary

**Files:**
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/resources/db/migration/V1.1.12__add_mtt_telegram_push_record.sql`
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/dal/dataobject/mttv2/MttTelegramPushRecordDO.java`
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/dal/mysql/mttv2/MttTelegramPushRecordMapper.java`
- Test: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImplTest.java`

- [ ] **Step 1: Write the failing dedup test**

```java
@Test
@DisplayName("same user same mtt same type should skip duplicate telegram push")
void pushShouldSkipWhenDuplicateRecordExists() {
    when(recordMapper.selectOneByUserIdAndMttIdAndType(1001L, 88L, "tournament_finished"))
            .thenReturn(existingRecord("SUCCESS"));

    service.pushTournamentFinished(buildTournament(), 1001L);

    verifyNoInteractions(telegramBotService);
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=MttTelegramPushServiceImplTest test
```

Expected: FAIL because the record table, mapper, and service do not exist yet.

- [ ] **Step 3: Add the migration**

```sql
CREATE TABLE mtt_telegram_push_record (
    id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    mtt_id BIGINT NOT NULL,
    mtt_code VARCHAR(64) NOT NULL,
    push_type VARCHAR(64) NOT NULL,
    telegram_user_id VARCHAR(64) DEFAULT NULL,
    status VARCHAR(20) NOT NULL,
    button_target VARCHAR(255) DEFAULT NULL,
    sent_text TEXT,
    error_message VARCHAR(512) DEFAULT NULL,
    sent_at DATETIME DEFAULT NULL,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted BIT(1) DEFAULT b'0',
    UNIQUE KEY uk_user_mtt_type (user_id, mtt_id, push_type),
    KEY idx_mtt_create_time (mtt_id, create_time)
);
```

- [ ] **Step 4: Add DO and mapper**

```java
@TableName("mtt_telegram_push_record")
public class MttTelegramPushRecordDO extends BaseDO {
    @TableId(type = IdType.AUTO)
    private Long id;
    private Long userId;
    private Long mttId;
    private String mttCode;
    private String pushType;
    private String telegramUserId;
    private String status;
    private String buttonTarget;
    private String sentText;
    private String errorMessage;
    private LocalDateTime sentAt;
}
```

```java
public interface MttTelegramPushRecordMapper extends BaseMapperX<MttTelegramPushRecordDO> {

    default MttTelegramPushRecordDO selectOneByUserIdAndMttIdAndType(Long userId, Long mttId, String pushType) {
        return selectOne(new LambdaQueryWrapperX<MttTelegramPushRecordDO>()
                .eq(MttTelegramPushRecordDO::getUserId, userId)
                .eq(MttTelegramPushRecordDO::getMttId, mttId)
                .eq(MttTelegramPushRecordDO::getPushType, pushType));
    }
}
```

- [ ] **Step 5: Re-run the targeted test**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=MttTelegramPushServiceImplTest test
```

Expected: still FAIL, but now because the service implementation is missing rather than the storage layer.

- [ ] **Step 6: Commit the storage layer**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add yudao-module-game/yudao-module-game-biz/src/main/resources/db/migration/V1.1.12__add_mtt_telegram_push_record.sql \
        yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/dal/dataobject/mttv2/MttTelegramPushRecordDO.java \
        yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/dal/mysql/mttv2/MttTelegramPushRecordMapper.java
git commit -m "feat(game): add mtt telegram push record table"
```

## Task 2: Implement The MTT Telegram Push Service With Short Copy

**Files:**
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushService.java`
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImpl.java`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttConstants.java`
- Test: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImplTest.java`

- [ ] **Step 1: Write failing tests for the 6 approved copy variants**

```java
@Test
@DisplayName("prize claimable should render short copy and claim button")
void prizeClaimableShouldRenderShortCopy() {
    PushPayload payload = service.buildPrizeClaimablePayload(buildTournament(), 1001L, "520 USDT");

    assertEquals("🏆 Your prize is ready to claim.\nPrize: 520 USDT", payload.getText());
    assertEquals("/m/rc?mttId=MTT-001", payload.getButtonTarget());
    assertEquals("💰 Claim Prize", payload.getButtonText());
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=MttTelegramPushServiceImplTest test
```

Expected: FAIL because the service and payload builder do not exist yet.

- [ ] **Step 3: Add the service contract and minimal payload builder**

```java
public interface MttTelegramPushService {
    void pushRegistrationApproved(MttTournamentDO tournament, Long userId);
    void pushRegistrationRejected(MttTournamentDO tournament, Long userId);
    void pushTournamentStarting(MttTournamentDO tournament, Long userId, Integer minutes);
    void pushTournamentCancelled(MttTournamentDO tournament, Long userId);
    void pushTournamentFinished(MttTournamentDO tournament, Long userId);
    void pushPrizeClaimable(MttTournamentDO tournament, Long userId, String payoutText);
}
```

- [ ] **Step 4: Implement whitelist, copy rendering, and button-target mapping**

```java
protected String resolveButtonTarget(String pushType, String mttCode) {
    return switch (pushType) {
        case "registration_approved", "registration_rejected", "tournament_finished" -> "/m/" + mttCode;
        case "tournament_starting" -> "/m/wait/" + mttCode;
        case "prize_claimable" -> "/m/rc?mttId=" + mttCode;
        default -> null;
    };
}
```

```java
protected String resolveShortText(String pushType, String mttName, Integer minutes, String payoutText) {
    return switch (pushType) {
        case "registration_approved" -> "✅ You are registered for " + mttName + ".";
        case "registration_rejected" -> "❌ Your registration for " + mttName + " was not approved.";
        case "tournament_starting" -> "⏰ " + mttName + " starts in " + minutes + " minutes.";
        case "tournament_cancelled" -> "⚠️ " + mttName + " has been cancelled.";
        case "tournament_finished" -> "🏁 " + mttName + " has finished.";
        case "prize_claimable" -> "🏆 Your prize is ready to claim.\nPrize: " + payoutText;
        default -> "";
    };
}
```

- [ ] **Step 5: Implement the send flow with graceful failure**

```java
if (duplicate != null) {
    return;
}
if (user == null || user.getTelegramUserId() == null) {
    saveSkipped(...);
    return;
}
try {
    boolean success = hasButton
            ? telegramBotService.sendMessageWithWebApp(chatId, text, buttonText, webAppUrl)
            : telegramBotService.sendMessage(chatId, text);
    saveResult(success ? "SUCCESS" : "FAILED", ...);
} catch (Exception e) {
    saveResult("FAILED", ...);
    log.warn("[mtt][tg] push failed type={}, userId={}, mttId={}", pushType, userId, tournament.getId(), e);
}
```

- [ ] **Step 6: Run the service tests again**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=MttTelegramPushServiceImplTest test
```

Expected: PASS with the short-copy payload tests, duplicate-skip tests, and failure-record tests all green.

- [ ] **Step 7: Commit the push service**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushService.java \
        yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImpl.java \
        yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttConstants.java \
        yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImplTest.java
git commit -m "feat(game): add mtt telegram push service"
```

## Task 3: Wire The 6 Telegram Pushes Into Existing MTT Event Flow

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttEventSupport.java`
- Test: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttEventSupportTelegramPushTest.java`

- [ ] **Step 1: Write failing event-routing tests**

```java
@Test
@DisplayName("finished event should push telegram for approved players")
void resultsFinalizedShouldTriggerPrizeClaimableTelegramPush() {
    support.handleEvent(buildFinishedEvent());

    verify(mttTelegramPushService).pushTournamentFinished(any(), eq(1001L));
}
```

```java
@Test
@DisplayName("table assigned should not trigger telegram push in V1")
void tableAssignedShouldNotTriggerTelegramPush() {
    support.handleEvent(buildTableAssignedEvent());

    verifyNoInteractions(mttTelegramPushService);
}
```

- [ ] **Step 2: Run the event-routing tests to verify they fail**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=MttEventSupportTelegramPushTest test
```

Expected: FAIL because `MttEventSupport` does not yet delegate to the Telegram push service.

- [ ] **Step 3: Inject `MttTelegramPushService` into `MttEventSupport`**

```java
@Resource
protected MttTelegramPushService mttTelegramPushService;
```

- [ ] **Step 4: Add only the approved 6 trigger points**

```java
case EVENT_PLAYER_APPROVED -> {
    sendNotificationInternal(... NOTICE_REGISTRATION_APPROVED ...);
    mttTelegramPushService.pushRegistrationApproved(tournament, targetUserId);
}
```

```java
if (STATUS_WAITING.equals(toStatus)) {
    approvedReceivers.forEach(userId ->
            mttTelegramPushService.pushTournamentStarting(tournament, userId, 10));
}
```

```java
else if (STATUS_FINISHED.equals(toStatus)) {
    approvedReceivers.forEach(userId ->
            mttTelegramPushService.pushTournamentFinished(tournament, userId));
}
```

```java
notifyClaimablePlayers(tournament, createTime);
results.stream()
        .filter(this::isClaimable)
        .forEach(result -> mttTelegramPushService.pushPrizeClaimable(
                tournament, result.getUserId(), defaultString(result.getPayoutText(), "Reward pending")));
```

- [ ] **Step 5: Re-run the targeted event tests**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=MttEventSupportTelegramPushTest test
```

Expected: PASS, and explicitly prove `table_assigned` is still excluded.

- [ ] **Step 6: Commit the event wiring**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/mttv2/MttEventSupport.java \
        yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttEventSupportTelegramPushTest.java
git commit -m "feat(game): wire mtt telegram pushes into event flow"
```

## Task 4: Verify Config Safety, Regression Coverage, And Delivery Behavior

**Files:**
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/resources/application-dev.yaml`
- Modify: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/resources/application-prod.yaml`
- Test: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImplTest.java`

- [ ] **Step 1: Add a regression test for invalid or missing Mini App URL fallback**

```java
@Test
@DisplayName("cancelled message should still send as text when button target is absent")
void cancelledShouldUseTextOnly() {
    service.pushTournamentCancelled(buildTournament(), 1001L);

    verify(telegramBotService).sendMessage(eq("8323696355"), contains("has been cancelled"));
    verify(telegramBotService, never()).sendMessageWithWebApp(any(), any(), any(), any());
}
```

- [ ] **Step 2: Run the focused tests and verify they fail if text-only fallback is missing**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=MttTelegramPushServiceImplTest test
```

Expected: FAIL until the service explicitly chooses text-only mode for buttonless types.

- [ ] **Step 3: Update YAML comments to prevent future bad config values**

```yaml
telegram:
  bot:
    # Mini App HTTPS entry URL, used by Telegram WebApp buttons. Do not use t.me direct links here.
    web-app-url: ${TELEGRAM_WEB_APP_URL:https://your-domain.com/game}
```

- [ ] **Step 4: Make the service fall back to text-only when no button target exists**

```java
if (buttonTarget == null || buttonTarget.isBlank()) {
    telegramBotService.sendMessage(chatId, text);
} else {
    telegramBotService.sendMessageWithWebApp(chatId, text, buttonText, buildAbsoluteWebAppUrl(buttonTarget));
}
```

- [ ] **Step 5: Run the targeted suite and then the module tests**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=MttTelegramPushServiceImplTest,MttEventSupportTelegramPushTest,TelegramMessageSendApiImplTest test
```

Expected: PASS with all new Telegram push tests green.

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am test
```

Expected: PASS or, if unrelated pre-existing failures exist, document them explicitly before proceeding.

- [ ] **Step 6: Commit the regression hardening**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add yudao-module-game/yudao-module-game-biz/src/main/resources/application-dev.yaml \
        yudao-module-game/yudao-module-game-biz/src/main/resources/application-prod.yaml \
        yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/service/mttv2/MttTelegramPushServiceImplTest.java
git commit -m "test(game): harden mtt telegram push config and fallback"
```

## Final Verification Checklist

- [ ] Confirm the 6 approved push types are the only ones that reach `MttTelegramPushService`.
- [ ] Confirm `table_assigned` and other excluded MTT notifications do not trigger Telegram send.
- [ ] Confirm duplicate sends are suppressed by the unique `(user_id, mtt_id, push_type)` boundary.
- [ ] Confirm `tournament_cancelled` sends pure text only.
- [ ] Confirm `prize_claimable` resolves `/m/rc?mttId={mttCode}`.
- [ ] Confirm missing `telegramUserId` records are stored as skipped or ignored without exception.
- [ ] Confirm Telegram failure does not roll back MTT event processing.
- [ ] Confirm `TELEGRAM_WEB_APP_URL` comments clearly say to use a real HTTPS Mini App URL, not a `t.me` direct link.

## Rollout Notes

- Enable the feature only after the bot Mini App URL is verified in the target environment.
- Use dev/staging manual checks for one sample event per type before broad rollout.
- Watch logs for `[mtt][tg]` warnings and inspect `mtt_telegram_push_record` for duplicate or failed sends.

## Manual QA Script

1. Approve one player registration and verify a single TG push arrives.
2. Reject a different registration and verify the reject copy arrives.
3. Transition a tournament into waiting and verify the starting reminder arrives once.
4. Cancel a tournament and verify pure-text cancel copy arrives.
5. Finish a tournament and verify the finish message arrives for participants.
6. Mark one winner claimable and verify the claim button opens the claim page.
7. Repeat one event twice and verify the second Telegram send is skipped.
