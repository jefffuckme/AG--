# Ops Desk Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the dashboard's top summary area with a single operations-desk module that reuses current GitHub Actions data and status calculations.

**Architecture:** Keep the existing fetch, polling, deploy, and service-card flows intact. Add a pure derived-data layer for duty metrics, alerts, queue items, and recent incidents, then render a new top-level `ops-desk` DOM section from that model.

**Tech Stack:** Standalone HTML, CSS, browser JavaScript, Node `node:test`, `vm`

---

## File Structure

- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.html`
  - Replace the old top summary DOM and CSS
  - Add operations-desk CSS
  - Add run-analysis helpers
  - Add operations-desk model building and rendering
  - Hook top-level rendering into existing load/patch flow
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.test.mjs`
  - Add helper extraction for new analysis functions
  - Add tests for alert/incident derivation
  - Replace the old activity-panel presence assertion

### Task 1: Add failing tests for ops-desk helper logic

**Files:**
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.test.mjs`
- Test: `/Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.test.mjs`

- [ ] **Step 1: Write failing helper tests for run classification**

```js
test('isFailureLikeConclusion treats failure, cancelled, and timed_out as failure-like', async () => {
  const { isFailureLikeConclusion } = await loadDashboardHelpers()
  assert.equal(isFailureLikeConclusion('failure'), true)
  assert.equal(isFailureLikeConclusion('cancelled'), true)
  assert.equal(isFailureLikeConclusion('timed_out'), true)
  assert.equal(isFailureLikeConclusion('success'), false)
})
```

- [ ] **Step 2: Write failing tests for streak, long-running, and recovery rules**

```js
test('getConsecutiveFailureCount counts latest failure streak only', async () => {
  const { getConsecutiveFailureCount } = await loadDashboardHelpers()
  assert.equal(getConsecutiveFailureCount([{ conclusion: 'failure' }, { conclusion: 'cancelled' }, { conclusion: 'success' }]), 2)
})
```

- [ ] **Step 3: Run the test file to verify failure**

Run: `node --test /Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.test.mjs`
Expected: FAIL with missing-function assertions for the new helpers.

- [ ] **Step 4: Commit the red test state if working in a repository**

```bash
git add /Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.test.mjs
git commit -m "test: add ops desk helper coverage"
```

### Task 2: Implement derived helper functions and top module rendering

**Files:**
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.html`

- [ ] **Step 1: Replace old top-section markup with an ops-desk container**

```html
<section class="ops-desk" id="ops-desk">
  <div class="ops-desk-loading">正在汇总值班视图...</div>
</section>
```

- [ ] **Step 2: Replace old status/activity CSS with ops-desk styles**

```css
.ops-desk { ... }
.ops-metric { ... }
.ops-alert-item { ... }
.ops-incident-item { ... }
```

- [ ] **Step 3: Add minimal helper implementations**

```js
function isFailureLikeConclusion(conclusion) {
  return conclusion === 'failure' || conclusion === 'cancelled' || conclusion === 'timed_out'
}
```

- [ ] **Step 4: Add the derived model builder and renderer**

```js
function buildOpsDeskModel() {
  return {
    metrics: ...,
    alerts: ...,
    dutyItems: ...,
    incidents: ...
  }
}
```

- [ ] **Step 5: Add scrolling helpers from ops items to service cards**

```js
function focusCard(cardId) {
  document.getElementById(`card-${cardId}`)?.scrollIntoView({ behavior: 'smooth', block: 'start' })
}
```

- [ ] **Step 6: Hook ops-desk rendering into `patchCard()` and `loadAll()`**

Run after card data changes so the top module stays synchronized.

- [ ] **Step 7: Commit implementation checkpoint if working in a repository**

```bash
git add /Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.html
git commit -m "feat: add ops desk dashboard"
```

### Task 3: Update the tests to green and cover HTML structure

**Files:**
- Modify: `/Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.test.mjs`

- [ ] **Step 1: Extend helper extraction to include the new functions**

```js
for (const name of ['isFailureLikeConclusion', 'getConsecutiveFailureCount', 'isLongRunningRun', 'isRecoveredRun']) {
  vm.runInContext(extractFunction(js, name), context)
}
```

- [ ] **Step 2: Replace the removed old-top-section assertion with new markup assertions**

```js
test('ops desk container replaces the old activity panel', async () => {
  const html = await loadDashboardHtml()
  assert.match(html, /id="ops-desk"/)
  assert.doesNotMatch(html, /id="stats-panel"/)
})
```

- [ ] **Step 3: Run the test file to verify green**

Run: `node --test /Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.test.mjs`
Expected: PASS for all tests.

- [ ] **Step 4: Commit the green test state if working in a repository**

```bash
git add /Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.test.mjs /Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.html
git commit -m "test: cover ops desk dashboard rules"
```

### Task 4: Manual verification in the browser

**Files:**
- Verify: `/Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.html`

- [ ] **Step 1: Open the page locally and check top-level layout**

Confirm:
- ops desk replaces the old summary bar and 24h panel
- refresh bar and service groups still render correctly

- [ ] **Step 2: Verify state consistency**

Compare:
- ops-desk monitoring counts
- alert rows
- duty queue rows
- recent incidents

against the underlying service cards and latest run data.

- [ ] **Step 3: Verify interactions**

Check:
- clicking alert/duty/incident rows scrolls to the right card
- no JS errors occur while auto-refreshing

- [ ] **Step 4: Record any residual risks**

Examples:
- runtime threshold is heuristic
- environment extraction may be partial for some workflows without explicit inputs
