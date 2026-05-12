# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AG H5 是一个多仓库工作区，面向在线德州扑克/博彩平台（Alano），以 Telegram Mini App 形式运行，采用移动端优先的 H5 页面。

| 子仓库 | 技术栈 | 说明 |
|--------|--------|------|
| `yekes-web-javascript/` | Vue 3 + Vite + TypeScript + Vant UI | 主 H5 前端（Telegram 小程序） |
| `yekes-java/` | Spring Boot + MyBatis-Plus + ioGame | 后端（Maven 多模块） |
| `new-yekes-game-javascript/` | Vue 3 + PixiJS 8 + Spine | 德州扑克游戏客户端（嵌入 H5 的 iframe） |
| `yekes-admin-javascript/` | Vue 3 + Vite（yudao-ui-admin） | 后台管理系统 |

跨仓库集成规则和安全准则见工作区根目录的 `AGENTS.md`。

## 命令速查

### H5 前端（yekes-web-javascript）

```bash
cd yekes-web-javascript

pnpm install                        # 安装依赖
pnpm xiaofu                         # 日常开发（xiaofu 环境，最常用）
pnpm dev                            # 默认开发服务器（port 5173, HTTPS）
pnpm tony                           # tony 环境
pnpm type-check                     # vue-tsc 类型检查
pnpm test:unit                      # 运行全部单元测试（vitest + jsdom）
pnpm vitest run <file>              # 运行单个测试文件
pnpm test:unit:mtt                  # 仅运行 MTT 模块的测试
pnpm lint                           # ESLint 自动修复
pnpm format                         # Prettier 格式化 src/
pnpm build                          # type-check + vite 生产构建（自动剔除 console.log）
pnpm analyze                        # 构建 + bundle 可视化分析
```

开发环境代理：所有 `/prod-api/*` 请求代理到 `https://m.testgames.org`，自动重写 Origin 和 cookie domain。

### 游戏客户端（new-yekes-game-javascript）

```bash
cd new-yekes-game-javascript

pnpm dev                            # 开发服务器（player 模式）
pnpm build                          # vue-tsc + vite 生产构建
pnpm build:with-atlas               # 重新生成 Sprite Atlas 后构建
pnpm test:mtt-runtime               # 运行 MTT 运行时状态测试
```

### 后端（yekes-java）

**JDK 要求：JDK 21**（pom.xml: `<java.version>21</java.version>`）

```bash
cd yekes-java

mvn -T 1C clean install -DskipTests          # 全量构建（跳过测试）
mvn test                                      # 全部测试
mvn -pl yudao-module-game/yudao-module-game-biz -am test   # 单模块测试
```

### 后台管理（yekes-admin-javascript）

```bash
cd yekes-admin-javascript

pnpm i
pnpm dev                            # 本地开发（mode: env.local）
pnpm build:prod                     # 生产构建
pnpm lint:eslint                    # ESLint 修复
```

## H5 前端架构（yekes-web-javascript）

### 路径别名
- `/@/` → `src/`
- `/#/` → `types/`

### 核心目录（src/ 下）

| 目录 | 职责 |
|------|------|
| `api/` | 各域 API 封装（每域一文件）。`api/model/` 存放请求/响应类型 |
| `stores/` | Pinia stores。`app.ts` 为根 store，各域 store 与 api 文件同名 |
| `hooks/` | 可复用 composable（useSocket、useI18n、useCopy、usePageBack 等） |
| `composables/` | 应用级 composable（useAuthBootstrap、useAppInit、useGlobalDialogs、useIframeToken） |
| `views/` | 页面组件，按功能分组。`views/new/` 存放新版页面 |
| `views/mtt/` | MTT 锦标赛模块——最大功能，约 60 个文件 |
| `components/` | 共享组件（弹窗、表单、数据表格） |
| `locales/lang/` | i18n（ar-SA、en-US、ko-KR、pt-PT、ru-RU、zh-TW） |
| `utils/request/` | Axios 封装，含拦截器（`Axios.ts`、`checkStatus.ts`） |

### 关键设计模式

- **API + Model 配对**：每个 `api/*.ts` 对应 `api/model/` 中的类型文件。API 函数使用 `httpClient.get/post<T>()`。
- **View + Flow/State 分离**：复杂功能（mtt、security）将业务逻辑拆分为 `*Flow.ts`（副作用/流程）和 `*State.ts`（纯函数），`.vue` 文件只负责 UI 渲染。
- **测试位置**：测试文件放在功能目录内的 `__tests__/` 子目录。业务逻辑测试用 vitest，API 测试使用属性驱动测试（`fast-check`）。
- **CSS**：SCSS scoped 到组件。多组件共享的弹窗样式提取到 `.shared.scss`。PostCSS px-to-rem（rootValue: 16.5）。
- **Vant UI**：通过 `unplugin-vue-components` + `VantResolver` 自动导入，无需手动 import。底部弹出面板使用 `<van-popup position="bottom">` 模式。

### Telegram 集成
- `@twa-dev/sdk` 提供 Telegram WebApp API
- `useTelegramBackButton`：管理系统返回键
- `useIframeToken`：向嵌入的游戏 iframe 注入 token

## 游戏客户端架构（new-yekes-game-javascript）

渲染层与逻辑层彻底解耦：

| 目录 | 职责 |
|------|------|
| `src/render/` | PixiJS 渲染层（游戏画面、Spine 骨骼动画） |
| `src/logic/` | 游戏业务逻辑（德州扑克规则、牌局流程） |
| `src/network/` | WebSocket / HTTP 通信 |
| `src/state/` | Pinia stores |
| `src/ui/` | Vue UI 组件（HUD、弹窗、菜单） |
| `src/services/` | 服务层（API、MTT 映射等） |

Sprite Atlas 由 `scripts/gen-*-atlas.mjs` 脚本生成，修改图片资源后需重新生成。

## 后端架构（yekes-java）

### 模块结构

```
yekes-java/
├── yudao-gateway/              # API 网关
├── yudao-framework/            # 公共框架
│   ├── yudao-common/           # 通用工具
│   ├── yudao-spring-boot-starter-security/   # TokenAuthenticationFilter、HMAC
│   ├── yudao-spring-boot-starter-mybatis/    # MyBatis-Plus 封装
│   ├── yudao-spring-boot-starter-redis/      # Redis 封装
│   └── yudao-spring-boot-starter-websocket/  # WebSocket 支持
├── yudao-module-system/        # 用户/认证/系统管理
├── yudao-module-game/          # 核心游戏逻辑
│   ├── yudao-module-game-api/     # API 接口 & DTO
│   ├── yudao-module-game-biz/     # 业务逻辑（主模块）
│   ├── yudao-module-game-broker/  # 消息代理
│   └── yudao-module-game-common/  # 共享常量
├── yudao-module-infra/
└── yudao-module-mail-sender/
```

### Game Biz 包结构（`cn.iocoder.yudao.module.game`）

| 包 | 职责 |
|----|------|
| `controller/admin/` `controller/app/` | REST 控制器（管理端/用户端分离） |
| `service/` | 业务服务（agent、club、credit、mttv2、privateroom、rake、security、texas 等） |
| `dal/` | 数据访问层（MyBatis-Plus Mapper + DO） |
| `iogame/` | ioGame WebSocket 服务端（房间逻辑、事件路由） |
| `framework/task/` | 异步任务框架（见下） |
| `poker/` | 扑克算法（手牌评估、牌局逻辑） |
| `robot/` | 机器人玩家逻辑 |
| `enums/` | 领域枚举 |
| `convert/` | MapStruct 转换器 |

### 异步任务框架（`framework/task/`）

```
config/   ExecutorConfig — 7 个命名线程池（task, game, broadcast, WAL, robot, outbox, room）
schedule/ CustomTaskExecutor — 按 roomKey 缓存 Future，提交前取消旧任务
          ThreadPoolMonitorTask, RoomClusterCleanupTask, RoomSessionExpireTask
service/  InternalTaskService — 线程任务生命周期（computeIfAbsent 去重）
thread/   RoomTask 接口, TexasPokerExceptionHandler
```

### 关键子系统

**私人房间配额系统**
- `PlayOwnerDO`（表 `play_owner`）：`roomLimit` 字段可为 null（null = 使用策略默认值）
- `PrivateRoomQuotaPolicy`：默认 normal=2，super=30；支持 DB 中per-owner 自定义
- `RoomServiceImpl.buildQuotaSnapshot()` 先读 DB，null 时回退策略默认值

**安全认证（Token + Google 2FA）**
- `TokenAuthenticationFilter`：4 参构造（`SecurityProperties, GlobalExceptionHandler, OAuth2TokenApi, Environment`）
- Google 2FA：`GoogleAuthenticatorUtil`（TOTP, HmacSHA1, window_size=1 允许 ±30s 容差）
- 所有安全端点带 `@RateLimiter(time=60, count=5)`

**数据库**：MySQL 5.7 + Redis 6，通过 `yekes-java/docker-compose.yml` 本地运行。

## 后端编码规则

### DO 类 → 实际表名（类名 ≠ 表名）

修改 DO 类字段前，必须先读 `@TableName` 注解确认实际表名，禁止按类名猜测。

| DO Class | 实际表名 |
|----------|---------|
| `AppUserDO` | `uc_member` |
| `MemberBetDO` | `uc_member_betting` |
| `MemberWalletDO` | `uc_member_wallet` |
| `MemberTransactionDO` | `ac_member_transaction` |
| `CreditWalletDO` | `play_credit_wallet` |
| `SalesmanRelationDO` | `play_club_invite_relation` |
| `OwnerRakeRecordDO` | `game_club_rake_record` |
| `TexasPokerActionLogDO` | `texas_poker_action_log` |
| `TexasPokerRoomRealDataDO` | `game_room_real_data` |
| `AnnouncementDO` | `cms_announcement` |

### Java 编码注意事项

- **中文引号**：Java 源码中禁止使用中文弯引号 `\u201c` / `\u201d`，必须使用 ASCII 直引号 `"`。
- **`@Slf4j`**：使用 `log` 变量的类必须添加 `@Slf4j` 注解。
- **接口同步**：`*ServiceImpl` 新增 public 方法时，必须在对应 `*Service` 接口中同步声明。
- **Spring Bean 构造函数**：构造函数参数变更后，`@Bean` 工厂方法必须同步更新参数列表。

## 跨仓库工作原则

- 涉及业务逻辑、API 调用、认证、支付、游戏流程、用户状态的任务，默认需要同时分析前后端两个仓库。
- API/字段/枚举/状态码任意一侧发生变更，必须在同一任务内验证另一侧是否匹配。
- Bug 诊断请沿完整链路追查：`前端页面 → composable/store → API wrapper → 后端 controller → service → DTO 合同`。

## 基础设施

- **数据库**：MySQL 5.7.44（docker-compose）
- **缓存**：Redis 6.0.20（docker-compose）
- **WebSocket 引擎**：ioGame（房间/游戏实时通信）
- **消息队列**：Spring Boot MQ starter（outbox 模式）

## Skills 自动触发规则

以下 Skill 在匹配场景下自动使用，无需手动调用：

### 全局通用
- **`simplify`** — 代码编写完成后审查复用性和简洁性
- **`code-review`** — 功能开发完成或提交 PR 前
- **`build-fix`** — 构建失败时
- **`systematic-debugging`** — 遇到 bug 或运行时异常
- **`tdd-workflow`** — TDD 工作流（强制 80%+ 覆盖率）
- **`security-review`** — 涉及认证、支付、用户数据时
- **`api-design`** — 新增/修改 REST 接口时

### 后端专属（yekes-java）
- **`java-coding-standards`** — 编写/修改 Java 代码
- **`springboot-patterns`** — 新增 Spring Boot 功能、Bean、配置
- **`springboot-security`** — 修改认证/授权/Token/安全相关代码
- **`springboot-tdd`** — 后端 TDD（JUnit 5、Mockito、MockMvc）
- **`jpa-patterns`** — 数据访问层、MyBatis 操作
- **`database-migrations`** — DDL 变更、数据库迁移

### 前端专属（yekes-web-javascript / new-yekes-game-javascript）
- **`frontend-design`** — 新增/修改 Vue 组件或页面布局
- **`frontend-patterns`** — 编写 Vue/TypeScript 代码时
