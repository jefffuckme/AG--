# Deploy Dashboard Interaction Upgrades Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Improve the deploy dashboard's day-to-day usability by tightening the main interaction loop around history expansion, deployment safety, refresh feedback, active filters, and prod confirmation.

**Architecture:** Keep all changes inside [`deploy-dashboard.html`](/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html) and extend the existing state-driven rendering model. Reuse current UI zones such as `card-time-meta`, `hero-alert-slot`, `confirm-modal`, and `showToast` instead of introducing new subsystems.

**Tech Stack:** Static HTML, CSS, vanilla JavaScript, GitHub Actions API fetches, local browser verification with Python HTTP server + Chrome headless.

---

### File Map

**Primary file**
- Modify: [`/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html`](/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html)

**Relevant code areas**
- Card header meta and time badge:
  - `renderCardTimeMeta`
  - `renderCard`
  - `patchCard`
- History expansion:
  - `toggleRunHistory`
  - `RUN_HISTORY_EXPANDED`
  - `renderRunList`
- Deploy flow:
  - `triggerDeploy`
  - `openConfirmModal`
  - `doConfirmedDeploy`
  - `getDeployButtonLabel`
- Refresh flow:
  - `loadAll`
  - `refreshCardData`
- Filtering:
  - `filterByStatus`
  - `clearStatusFilter`
  - hero toolbar around `btn-attention`

**Verification commands**
- Run local preview:
  - `python3 -m http.server 8123`
- Take full screenshot:
  - `'/Applications/Google Chrome.app/Contents/MacOS/Google Chrome' --headless --disable-gpu --hide-scrollbars --window-size=1600,2200 --virtual-time-budget=8000 --screenshot=/private/tmp/deploy-dashboard-verify.png 'http://127.0.0.1:8123/deploy-dashboard.html'`

---

### Task 1: Make Header Time Meta The Primary History Toggle

**Files:**
- Modify: [`/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html`](/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html)

- [ ] **Step 1: Update `renderCardTimeMeta` to output an interactive element**

Use `onclick="toggleRunHistory('<cardId>')"` on the rendered time pill or wrap it in a small button-like container. Keep the current running/success/failure visual language.

- [ ] **Step 2: Add accessibility and state hints**

Include `title`, `role="button"`, and a lightweight expanded/collapsed cue such as:
- collapsed: `最后部署 2 小时前 · 点击查看历史`
- expanded: `最后部署 2 小时前 · 点击收起历史`

- [ ] **Step 3: Extend `toggleRunHistory(cardId)` so header state stays in sync**

When history expands/collapses:
- toggle history area
- update expand button text
- update the header time meta content if necessary

- [ ] **Step 4: Add hover and active styling for the time meta area**

Modify the CSS around `.card-time-meta`, `.last-deploy-pill`, and `.elapsed-badge` so the element clearly looks interactive.

- [ ] **Step 5: Verify behavior manually**

Run:
```bash
python3 -m http.server 8123
```

Check:
- Clicking the header time pill expands history
- Clicking again collapses history
- A running card remains expanded after refresh

---

### Task 2: Lock Deploy Button While A Deploy Request Is In Flight

**Files:**
- Modify: [`/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html`](/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html)

- [ ] **Step 1: Add a deploy request state store**

Near the existing global state objects (`CARD_RUNS`, `RUN_HISTORY_EXPANDED`, `IN_FLIGHT`), add:
```js
const CARD_DEPLOYING = {}
```

- [ ] **Step 2: Centralize button label resolution**

Update `getDeployButtonLabel` or equivalent logic so it can return:
- `部署`
- `部署中…`
- `已触发`

based on `CARD_DEPLOYING[cardId]` and current run state.

- [ ] **Step 3: Block duplicate deploy entry**

At the beginning of `triggerDeploy(wf, cardId)`, return early if the card is already in deploy-request state.

- [ ] **Step 4: Set and clear deploy-request state in the confirmed deploy flow**

In `doConfirmedDeploy()` and the fetch success/failure branches:
- set `CARD_DEPLOYING[cardId] = true` before request
- clear it on success or failure response
- re-render the affected button state

- [ ] **Step 5: Disable the actual deploy button while locked**

When rendering the card footer, set `disabled` if:
- workflow is disabled
- or the card is currently sending a deploy request

- [ ] **Step 6: Verify duplicate-click prevention**

Check:
- Double-clicking deploy only sends once
- Button text changes immediately
- Failure restores button usability

---

### Task 3: Add Global Refresh Progress Feedback

**Files:**
- Modify: [`/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html`](/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html)

- [ ] **Step 1: Add refresh progress state**

Add a lightweight state object, for example:
```js
const GLOBAL_REFRESH_STATE = { active: false, done: 0, total: 0 }
```

- [ ] **Step 2: Update `loadAll()` to report progress**

When starting a full refresh:
- set `active=true`
- set `total=enabled.length`
- increment `done` after each card finishes

- [ ] **Step 3: Update the hero refresh button label dynamically**

For the existing button near `btn-hero-refresh`, render:
- idle: `↺ 刷新全部`
- active: `↺ 刷新中 6/15`

- [ ] **Step 4: Prevent repeated global refresh clicks while active**

Disable or no-op the button during active refresh.

- [ ] **Step 5: Optionally surface completion**

Only after the whole batch finishes, show a single compact completion toast. Avoid noisy per-card refresh success toasts.

- [ ] **Step 6: Verify progress UX**

Check:
- progress increments while cards load
- button is not spammable
- label resets after completion

---

### Task 4: Show A Persistent Active Filter Banner

**Files:**
- Modify: [`/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html`](/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html)

- [ ] **Step 1: Add a dedicated filter banner slot below the hero area**

Reuse the hero/banner styling patterns instead of inventing a new panel style.

- [ ] **Step 2: Create a small render helper for active filter status**

Add a helper like:
```js
function renderActiveFilterBanner() { ... }
```

It should cover:
- `attention`
- `fail`
- `running`
- `ok`

- [ ] **Step 3: Add a clear action**

The banner should include a clear button that calls the existing filter reset flow.

- [ ] **Step 4: Wire banner refresh into filter state updates**

Whenever `filterByStatus` or clear-filter runs:
- update the cards
- update the hero button active state
- update the persistent banner

- [ ] **Step 5: Verify discoverability**

Check:
- Entering any filtered mode shows a visible banner
- Clearing the banner restores all cards
- Refreshing data does not desync the banner state

---

### Task 5: Add Strong Prod Confirmation

**Files:**
- Modify: [`/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html`](/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html)

- [ ] **Step 1: Extend modal payload with risk context**

Pass these values into `openConfirmModal`:
- service label
- target environment
- branch
- workflow file
- whether the action is batch deploy

- [ ] **Step 2: Add a prod-only confirmation input**

Inside the confirm modal, show an extra input only when `isProd === true`.

Expected rule:
- user must type the service name or a fixed keyword such as `PROD`

- [ ] **Step 3: Block submission until the prod confirmation value matches**

In `doConfirmedDeploy()`, validate before sending the request.

- [ ] **Step 4: Add stronger visual warning for prod**

Reuse the existing warning palette around failure/attention UI. The modal should make prod look qualitatively different from test deploy.

- [ ] **Step 5: Verify both test and prod paths**

Check:
- test deploy still uses a lightweight confirm flow
- prod deploy cannot continue without the extra confirmation
- wrong confirmation input produces a clear inline message

---

### Task 6: End-To-End Verification Pass

**Files:**
- Modify: [`/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html`](/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html)

- [ ] **Step 1: Run the local preview**

```bash
python3 -m http.server 8123
```

- [ ] **Step 2: Capture a fresh screenshot**

```bash
'/Applications/Google Chrome.app/Contents/MacOS/Google Chrome' --headless --disable-gpu --hide-scrollbars --window-size=1600,2200 --virtual-time-budget=8000 --screenshot=/private/tmp/deploy-dashboard-verify.png 'http://127.0.0.1:8123/deploy-dashboard.html'
```

- [ ] **Step 3: Manually verify the five target behaviors**

Checklist:
- header time meta expands and collapses history
- deploy button cannot be double-triggered
- refresh all shows progress
- active filter is visibly persistent
- prod confirmation is stronger than test confirmation

- [ ] **Step 4: Smoke-check existing flows for regression**

Check:
- log modal still opens
- section filters still work
- card patch updates still update state badges and history

