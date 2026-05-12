# Notify Message Telegram Mirror Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Telegram mirror delivery for every member-facing message-center notification created in `system_notify_message`, while keeping message creation as the source of truth and recording Telegram delivery state without retries.

**Architecture:** The system module remains the owner of notification creation and rendering. After `NotifyMessageServiceImpl.createNotifyMessage(...)` inserts the primary notification row, it calls a new system-side mirror coordinator that persists Telegram delivery state and invokes a game-api RPC to send the escaped final text. The game module exposes a minimal `TelegramMessageSendApi`, implemented in game-biz by adapting the existing `TelegramBotService`.

**Tech Stack:** Java 11, Spring Boot, OpenFeign RPC, MyBatis-Plus, Flyway SQL migrations, JUnit 5, Mockito, Maven multi-module build.

---

## File Map

### Existing Files To Modify

- `yekes-java/yudao-module-game/yudao-module-game-api/src/main/java/cn/iocoder/yudao/module/game/enums/ApiConstants.java`
  Confirm or reuse API prefix constants for the new Telegram RPC path if needed.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/service/telegram/TelegramBotService.java`
  Reuse the existing send capability from the new RPC implementation.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageServiceImpl.java`
  Trigger mirror creation only after the primary message row is inserted.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageService.java`
  Add any new helper contract needed for rendering or mirror invocation if extraction requires interface exposure.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserService.java`
  Reuse existing user lookup methods needed by mirror rendering and Telegram eligibility checks.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/user/AppUserServiceImpl.java`
  Reuse current language lookup behavior if shared helper extraction is needed.

### New Files To Create

- `yekes-java/yudao-module-game/yudao-module-game-api/src/main/java/cn/iocoder/yudao/module/game/api/telegram/TelegramMessageSendApi.java`
  Feign-style RPC contract for minimal Telegram text send.
- `yekes-java/yudao-module-game/yudao-module-game-api/src/main/java/cn/iocoder/yudao/module/game/api/telegram/dto/TelegramSendMessageReqDTO.java`
  RPC request DTO carrying `chatId` and final message text.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/api/telegram/TelegramMessageSendApiImpl.java`
  RPC implementation delegating to `TelegramBotService`.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/api/telegram/TelegramMessageSendApiImplTest.java`
  Unit test for RPC adapter behavior.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/notify/NotifyMessageTelegramDO.java`
  Delivery status data object for `system_notify_message_telegram`.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/mysql/notify/NotifyMessageTelegramMapper.java`
  Mapper for delivery status persistence.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorService.java`
  Mirror service contract.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImpl.java`
  Mirror coordinator that creates `PENDING`, renders content, checks eligibility, sends Telegram, and persists final state.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageRenderResult.java`
  Small value object for resolved title/content/lang.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImplTest.java`
  Unit tests for `SKIPPED`, `SUCCESS`, `FAILED`, and escaped-content cases.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageServiceImplTelegramMirrorTest.java`
  Focused test that `createNotifyMessage(...)` inserts the main row and then triggers the mirror path without surfacing Telegram failure.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/resources/db/migration/V1.1.12__add_notify_message_telegram_mirror_table.sql`
  Flyway migration for the new status table and indexes.

### Optional Existing Files To Inspect During Implementation

- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/mysql/notify/NotifyMessageMapper.java`
  Existing notification persistence style to mirror in the new mapper.
- `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/controller/app/member/AppNotifyMessageController.java`
  Verify no controller change is required.
- `yekes-java/yudao-module-game/yudao-module-game-api/src/main/java/cn/iocoder/yudao/module/game/api/session/GameSessionApi.java`
  Reference pattern for new game-api RPC design.
- `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/api/session/GameSessionApiImpl.java`
  Reference pattern for new game-biz RPC implementation.

## Task 1: Add The Game RPC Contract For Telegram Text Sending

**Files:**
- Create: `yekes-java/yudao-module-game/yudao-module-game-api/src/main/java/cn/iocoder/yudao/module/game/api/telegram/TelegramMessageSendApi.java`
- Create: `yekes-java/yudao-module-game/yudao-module-game-api/src/main/java/cn/iocoder/yudao/module/game/api/telegram/dto/TelegramSendMessageReqDTO.java`
- Create: `yekes-java/yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/api/telegram/TelegramMessageSendApiImpl.java`
- Test: `yekes-java/yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/api/telegram/TelegramMessageSendApiImplTest.java`

- [ ] **Step 1: Write the failing RPC adapter test**

```java
@ExtendWith(MockitoExtension.class)
class TelegramMessageSendApiImplTest {

    @Mock
    private TelegramBotService telegramBotService;

    @InjectMocks
    private TelegramMessageSendApiImpl api;

    @Test
    @DisplayName("RPC send should delegate final text to TelegramBotService")
    void sendMessageShouldDelegateToTelegramBotService() {
        TelegramSendMessageReqDTO req = new TelegramSendMessageReqDTO();
        req.setChatId("123456");
        req.setText("Title\n\nBody");
        when(telegramBotService.sendMessage("123456", "Title\n\nBody")).thenReturn(true);

        CommonResult<Boolean> result = api.sendMessage(req);

        assertTrue(result.isSuccess());
        assertTrue(Boolean.TRUE.equals(result.getData()));
        verify(telegramBotService).sendMessage("123456", "Title\n\nBody");
    }
}
```

- [ ] **Step 2: Run the game-biz test to verify it fails**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=TelegramMessageSendApiImplTest test
```

Expected: FAIL because `TelegramMessageSendApiImplTest` and related API classes do not exist yet.

- [ ] **Step 3: Add the minimal game-api RPC contract**

```java
@FeignClient(name = ApiConstants.NAME)
@Tag(name = "RPC 服务 - Telegram 消息发送")
public interface TelegramMessageSendApi {

    String PREFIX = ApiConstants.PREFIX + "/telegram/message";

    @PostMapping(PREFIX + "/send")
    CommonResult<Boolean> sendMessage(@Valid @RequestBody TelegramSendMessageReqDTO reqDTO);
}
```

```java
@Data
public class TelegramSendMessageReqDTO {

    @NotBlank(message = "chatId 不能为空")
    private String chatId;

    @NotBlank(message = "text 不能为空")
    private String text;
}
```

- [ ] **Step 4: Implement the game-biz adapter with no extra behavior**

```java
@RestController
@Validated
public class TelegramMessageSendApiImpl implements TelegramMessageSendApi {

    @Resource
    private TelegramBotService telegramBotService;

    @Override
    public CommonResult<Boolean> sendMessage(TelegramSendMessageReqDTO reqDTO) {
        return success(telegramBotService.sendMessage(reqDTO.getChatId(), reqDTO.getText()));
    }
}
```

- [ ] **Step 5: Run the adapter test again**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=TelegramMessageSendApiImplTest test
```

Expected: PASS with `Tests run: 1, Failures: 0, Errors: 0`.

- [ ] **Step 6: Commit the RPC contract**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add yudao-module-game/yudao-module-game-api/src/main/java/cn/iocoder/yudao/module/game/api/telegram \
        yudao-module-game/yudao-module-game-biz/src/main/java/cn/iocoder/yudao/module/game/api/telegram \
        yudao-module-game/yudao-module-game-biz/src/test/java/cn/iocoder/yudao/module/game/api/telegram/TelegramMessageSendApiImplTest.java
git commit -m "feat(game): add telegram message send rpc"
```

## Task 2: Create The Telegram Mirror Status Table And Persistence Layer

**Files:**
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/resources/db/migration/V1.1.12__add_notify_message_telegram_mirror_table.sql`
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/notify/NotifyMessageTelegramDO.java`
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/mysql/notify/NotifyMessageTelegramMapper.java`
- Test: `yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImplTest.java`

- [ ] **Step 1: Write the failing system mirror service test around initial `PENDING` persistence**

```java
@Test
@DisplayName("mirror start should create PENDING telegram delivery row")
void mirrorShouldCreatePendingDeliveryRow() {
    NotifyMessageDO notify = buildNotifyMessage(9001L, 1001L, "PRIVATE_ROOM_AUTH_APPLY");
    when(notifyMessageMapper.selectById(9001L)).thenReturn(notify);
    when(appUserService.getUserById(1001L)).thenReturn(buildUserWithTelegram("777888"));

    mirrorService.mirror(9001L);

    verify(notifyMessageTelegramMapper).insert(deliveryCaptor.capture());
    assertEquals("PENDING", deliveryCaptor.getValue().getStatus());
    assertEquals(9001L, deliveryCaptor.getValue().getNotifyMessageId());
}
```

- [ ] **Step 2: Run the system-biz test to verify it fails**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=NotifyTelegramMirrorServiceImplTest test
```

Expected: FAIL because the mirror service, DO, mapper, and migration do not exist yet.

- [ ] **Step 3: Add the Flyway migration and persistence objects**

```sql
CREATE TABLE system_notify_message_telegram (
    id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    notify_message_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    telegram_user_id VARCHAR(64) DEFAULT NULL,
    template_code VARCHAR(100) DEFAULT NULL,
    status VARCHAR(20) NOT NULL,
    telegram_message_id VARCHAR(64) DEFAULT NULL,
    error_code VARCHAR(64) DEFAULT NULL,
    error_message VARCHAR(512) DEFAULT NULL,
    sent_content TEXT,
    sent_at DATETIME DEFAULT NULL,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted BIT(1) DEFAULT b'0',
    UNIQUE KEY uk_notify_message_id (notify_message_id),
    KEY idx_user_create_time (user_id, create_time)
);
```

```java
@TableName("system_notify_message_telegram")
public class NotifyMessageTelegramDO extends BaseDO {
    @TableId
    private Long id;
    private Long notifyMessageId;
    private Long userId;
    private String telegramUserId;
    private String templateCode;
    private String status;
    private String telegramMessageId;
    private String errorCode;
    private String errorMessage;
    private String sentContent;
    private LocalDateTime sentAt;
}
```

- [ ] **Step 4: Add the mapper with small helper methods only**

```java
@Mapper
public interface NotifyMessageTelegramMapper extends BaseMapperX<NotifyMessageTelegramDO> {

    default NotifyMessageTelegramDO selectByNotifyMessageId(Long notifyMessageId) {
        return selectOne(new LambdaQueryWrapperX<NotifyMessageTelegramDO>()
                .eq(NotifyMessageTelegramDO::getNotifyMessageId, notifyMessageId));
    }
}
```

- [ ] **Step 5: Run the system-biz test again**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=NotifyTelegramMirrorServiceImplTest test
```

Expected: FAIL, but now only because the mirror service behavior is still missing.

- [ ] **Step 6: Commit the persistence slice**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add yudao-module-system/yudao-module-system-biz/src/main/resources/db/migration/V1.1.12__add_notify_message_telegram_mirror_table.sql \
        yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/notify/NotifyMessageTelegramDO.java \
        yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/dal/mysql/notify/NotifyMessageTelegramMapper.java \
        yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImplTest.java
git commit -m "feat(system): add telegram mirror delivery table"
```

## Task 3: Build Shared Notification Rendering And Telegram Escaping

**Files:**
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageRenderResult.java`
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageServiceImpl.java`
- Test: `yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImplTest.java`

- [ ] **Step 1: Extend the failing mirror test to require rendered title/content and escaped text**

```java
@Test
@DisplayName("mirror should send escaped final title and content text")
void mirrorShouldSendEscapedRenderedText() {
    NotifyMessageDO notify = buildNotifyMessage(9002L, 1002L, "PRIVATE_ROOM_AUTH_APPLY");
    notify.setTemplateParams(Map.of("owner_name", "A&B", "player_name", "<Neo>"));
    when(notifyMessageMapper.selectById(9002L)).thenReturn(notify);
    when(appUserService.getUserById(1002L)).thenReturn(buildUserWithTelegram("999000"));
    when(telegramMessageSendApi.sendMessage(any())).thenReturn(CommonResult.success(true));

    mirrorService.mirror(9002L);

    verify(telegramMessageSendApi).sendMessage(reqCaptor.capture());
    assertEquals("Private Room Notice\n\nOwner A&amp;B approved &lt;Neo&gt;", reqCaptor.getValue().getText());
}
```

- [ ] **Step 2: Run the mirror test to verify the render/escape case fails**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=NotifyTelegramMirrorServiceImplTest#mirrorShouldSendEscapedRenderedText test
```

Expected: FAIL because no reusable render helper or escape logic exists.

- [ ] **Step 3: Extract a small render result object and helper methods**

```java
public record NotifyMessageRenderResult(String title, String content, String lang) {
}
```

```java
protected NotifyMessageRenderResult renderNotifyMessage(NotifyMessageDO notifyMessageDO) {
    String lang = resolveMemberLang(notifyMessageDO.getUserId(), notifyMessageDO.getUserType());
    NotifyTemplateDO template = resolveTemplate(notifyMessageDO.getTemplateCode(), lang);
    String title = template == null ? notifyMessageDO.getTemplateCode() : template.getTitle();
    String content = template == null ? "" :
            defaultString(notifyTemplateService.formatNotifyTemplateContent(
                    template.getContent(), notifyMessageDO.getTemplateParams()));
    return new NotifyMessageRenderResult(title, content, lang);
}
```

```java
static String escapeTelegramHtml(String text) {
    return defaultString(text)
            .replace("&", "&amp;")
            .replace("<", "&lt;")
            .replace(">", "&gt;");
}
```

- [ ] **Step 4: Keep the message-center read path using the same render helper**

```java
NotifyMessageRenderResult rendered = renderNotifyMessage(notifyMessageDO);
vo.setTitle(rendered.title());
vo.setTemplateContent(rendered.content());
```

- [ ] **Step 5: Run the mirror test again**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=NotifyTelegramMirrorServiceImplTest#mirrorShouldSendEscapedRenderedText test
```

Expected: Still FAIL until the mirror service is wired, but render and escape helpers now compile and are reusable.

- [ ] **Step 6: Commit the rendering extraction**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageRenderResult.java \
        yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageServiceImpl.java \
        yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImplTest.java
git commit -m "refactor(system): extract notify render helper for mirror delivery"
```

## Task 4: Implement The Mirror Coordinator With `SKIPPED`, `SUCCESS`, And `FAILED` States

**Files:**
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorService.java`
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImpl.java`
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImplTest.java`

- [ ] **Step 1: Add the failing mirror state tests**

```java
@Test
@DisplayName("missing telegramUserId should mark delivery as SKIPPED")
void mirrorShouldSkipWhenTelegramUserIdMissing() { }

@Test
@DisplayName("telegram RPC false result should mark delivery as FAILED")
void mirrorShouldMarkFailedWhenTelegramSendReturnsFalse() { }

@Test
@DisplayName("telegram RPC success should mark delivery as SUCCESS and sentAt")
void mirrorShouldMarkSuccessWhenTelegramSendSucceeds() { }
```

- [ ] **Step 2: Run the mirror service test suite and confirm full failure set**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=NotifyTelegramMirrorServiceImplTest test
```

Expected: FAIL with missing service classes and unresolved behavior.

- [ ] **Step 3: Implement the mirror service with the minimum branching needed**

```java
public interface NotifyTelegramMirrorService {
    void mirror(Long notifyMessageId);
}
```

```java
@Service
public class NotifyTelegramMirrorServiceImpl implements NotifyTelegramMirrorService {

    @Override
    public void mirror(Long notifyMessageId) {
        try {
            NotifyMessageDO notify = notifyMessageMapper.selectById(notifyMessageId);
            if (notify == null) {
                return;
            }
            NotifyMessageTelegramDO delivery = createPendingDelivery(notify);
            if (!isMemberNotify(notify)) {
                markSkipped(delivery, "NOT_MEMBER");
                return;
            }
            AppUserDO user = appUserService.getUserById(notify.getUserId());
            if (user == null || StrUtil.isBlank(user.getTelegramUserId())) {
                markSkipped(delivery, "NO_TELEGRAM_USER");
                return;
            }
            if (!telegramAvailable()) {
                markSkipped(delivery, "BOT_UNAVAILABLE");
                return;
            }
            NotifyMessageRenderResult rendered = notifyMessageService.renderNotifyMessage(notify);
            String text = buildTelegramText(rendered);
            CommonResult<Boolean> result = telegramMessageSendApi.sendMessage(
                    new TelegramSendMessageReqDTO().setChatId(user.getTelegramUserId()).setText(text));
            if (result != null && result.isSuccess() && Boolean.TRUE.equals(result.getData())) {
                markSuccess(delivery, text);
            } else {
                markFailed(delivery, text, "SEND_FAILED", result == null ? "null result" : result.getMsg());
            }
        } catch (Exception e) {
            markFailedSafely(notifyMessageId, "EXCEPTION", e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run the mirror service tests until they pass**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=NotifyTelegramMirrorServiceImplTest test
```

Expected: PASS with all `SKIPPED`, `SUCCESS`, `FAILED`, and escaped-text cases green.

- [ ] **Step 5: Commit the mirror coordinator**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorService.java \
        yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImpl.java \
        yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyTelegramMirrorServiceImplTest.java
git commit -m "feat(system): add telegram mirror coordinator"
```

## Task 5: Trigger The Mirror Flow Only After Primary Message Insert

**Files:**
- Modify: `yekes-java/yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageServiceImpl.java`
- Create: `yekes-java/yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageServiceImplTelegramMirrorTest.java`

- [ ] **Step 1: Write the failing create-notify-message trigger test**

```java
@Test
@DisplayName("createNotifyMessage should insert main row before invoking telegram mirror")
void createNotifyMessageShouldInsertMainRowBeforeTelegramMirror() {
    when(notifyTemplateService.selectByCode("PRIVATE_ROOM_AUTH_APPLY")).thenReturn(List.of(template));

    notifyMessageService.createNotifyMessage(1001L, UserTypeEnum.MEMBER.getValue(), template, params);

    InOrder inOrder = inOrder(notifyMessageMapper, notifyTelegramMirrorService);
    inOrder.verify(notifyMessageMapper).insert(any(NotifyMessageDO.class));
    inOrder.verify(notifyTelegramMirrorService).mirror(anyLong());
}
```

- [ ] **Step 2: Run the focused service test and verify it fails**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=NotifyMessageServiceImplTelegramMirrorTest test
```

Expected: FAIL because `NotifyMessageServiceImpl` does not yet invoke the mirror service.

- [ ] **Step 3: Add the minimal trigger call after insert**

```java
notifyMessageMapper.insert(message);
Long messageId = message.getId();
try {
    notifyTelegramMirrorService.mirror(messageId);
} catch (Exception e) {
    log.warn("[notifyMessage] telegram mirror trigger failed messageId={}", messageId, e);
}
return messageId;
```

- [ ] **Step 4: Run the focused trigger test**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am -Dtest=NotifyMessageServiceImplTelegramMirrorTest test
```

Expected: PASS with ordered verification of insert then mirror.

- [ ] **Step 5: Run the broader notify-related test set**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am \
  -Dtest=NotifyMessageServiceImplTelegramMirrorTest,NotifyTelegramMirrorServiceImplTest test
```

Expected: PASS with no regression in message creation flow.

- [ ] **Step 6: Commit the trigger wiring**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add yudao-module-system/yudao-module-system-biz/src/main/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageServiceImpl.java \
        yudao-module-system/yudao-module-system-biz/src/test/java/cn/iocoder/yudao/module/system/service/notify/sms/NotifyMessageServiceImplTelegramMirrorTest.java
git commit -m "feat(system): trigger telegram mirror after notify insert"
```

## Task 6: Run Cross-Module Verification And Update The Design Record If Needed

**Files:**
- Modify: `docs/superpowers/specs/2026-04-22-notify-message-telegram-mirror-design.md`
  Only if implementation reality requires a small correction.

- [ ] **Step 1: Run the game module Telegram RPC tests**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz -am -Dtest=TelegramMessageSendApiImplTest test
```

Expected: PASS.

- [ ] **Step 2: Run the system module mirror tests**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-system/yudao-module-system-biz -am \
  -Dtest=NotifyTelegramMirrorServiceImplTest,NotifyMessageServiceImplTelegramMirrorTest test
```

Expected: PASS.

- [ ] **Step 3: Run a combined targeted multi-module verification**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
mvn -pl yudao-module-game/yudao-module-game-biz,yudao-module-system/yudao-module-system-biz -am \
  -Dtest=TelegramMessageSendApiImplTest,NotifyTelegramMirrorServiceImplTest,NotifyMessageServiceImplTelegramMirrorTest test
```

Expected: PASS with all targeted modules building together.

- [ ] **Step 4: Confirm no frontend change is required**

Run:

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏
rtk rg -n "system_notify_message_telegram|telegram mirror" yekes-web-javascript
```

Expected: No required frontend integration work discovered for V1.

- [ ] **Step 5: If any implementation detail diverged from the spec, update the spec**

```markdown
- Update only the sections that changed after implementation reality was confirmed.
- Do not expand scope to retries, buttons, or non-message-center sources.
```

- [ ] **Step 6: Commit the final verified slice**

```bash
cd /Users/mingge/Documents/IdeaProjects/AG游戏/yekes-java
git add .
git commit -m "feat(system): mirror message center notifications to telegram"
```

## Notes For Execution

- Keep the first pass synchronous. Do not introduce `@Async`, MQ, or retry logic in this plan.
- Reuse message-center rendering logic. Do not create Telegram-only templates.
- Keep front-end untouched unless verification proves an unexpected dependency.
- If `telegrambots-spring-boot-starter` already gives enough wiring in system-biz, do not shortcut the planned API boundary anyway. The approved design depends on system -> game-api, not system -> game-biz.
- If the worker discovers the exact next Flyway version is not `V1.1.12`, update the filename to the correct monotonic version before implementation.
- Before each commit, run only the smallest relevant test command for the task you just completed.
