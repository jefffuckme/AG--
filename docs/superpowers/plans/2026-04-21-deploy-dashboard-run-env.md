# Deploy Dashboard Run Environment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make each deployment history item in `deploy-dashboard.html` show whether it ran in the test or prod environment.

**Architecture:** Keep the change inside the standalone dashboard and reuse the existing `RUN_INPUTS` cache populated from GitHub Actions job details. Add one small environment resolver based on job labels, then render a stable `TEST` or `PROD` badge inside each run item without changing deploy or fetch flows.

**Tech Stack:** HTML, inline CSS, vanilla JavaScript, Node.js built-in test runner

---

### Task 1: Add the failing test

**Files:**
- Modify: `/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.test.mjs`
- Test: `/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.test.mjs`

- [ ] **Step 1: Write the failing test**

```js
test('renderRunItem shows the TEST or PROD environment badge from run labels', async () => {
  const { renderRunItem, RUN_INPUTS } = await loadDashboardHelpers()
  RUN_INPUTS[101] = { labels: ['self-hosted', 'test'], runner_name: 'runner-a' }
  RUN_INPUTS[202] = { labels: ['prod_env'], runner_name: 'runner-b' }

  assert.match(renderRunItem({ id: 101, ...baseRun }), /TEST/)
  assert.match(renderRunItem({ id: 202, ...baseRun }), /PROD/)
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test --test-name-pattern "environment badge" deploy-dashboard.test.mjs`
Expected: FAIL because `renderRunItem` does not yet render a dedicated environment badge.

### Task 2: Implement the minimal dashboard change

**Files:**
- Modify: `/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.html`
- Test: `/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.test.mjs`

- [ ] **Step 3: Add a run environment resolver**

```js
function getRunEnvironment(run) {
  const labels = RUN_INPUTS[run.id]?.labels || []
  if (labels.some(label => label.toLowerCase().includes('prod'))) return 'prod'
  if (labels.some(label => label.toLowerCase().includes('test'))) return 'test'
  return ''
}
```

- [ ] **Step 4: Render the environment badge in each run item**

```js
const env = getRunEnvironment(run)
const envBadge = env ? `<span class="run-input-badge run-env-badge run-input-${env}">${env.toUpperCase()}</span>` : ''
```

- [ ] **Step 5: Keep styling aligned with the existing run badge system**

```css
.run-env-badge {
  letter-spacing: .04em;
}
```

### Task 3: Verify

**Files:**
- Test: `/Users/mingge/Documents/IdeaProjects/AG游戏/deploy-dashboard.test.mjs`

- [ ] **Step 6: Run tests to verify they pass**

Run: `node --test deploy-dashboard.test.mjs`
Expected: PASS

- [ ] **Step 7: Sanity-check the references**

Run: `rg -n "getRunEnvironment|run-env-badge|TEST|PROD" deploy-dashboard.html deploy-dashboard.test.mjs`
Expected: the helper, badge class, and assertions appear in the implementation and test.
