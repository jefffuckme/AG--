# Deploy Dashboard Latest Run Sort Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Sort services inside each deploy dashboard group by their latest deployment time, newest first.

**Architecture:** Keep workflow definitions in their current static arrays, but convert group loading into a two-phase flow: fetch all runs for the group into stable entries, sort those entries by latest run time, then render or reorder cards based on that sorted list. Preserve `cardId` stability so input values, timers, and incremental patching continue to work.

**Tech Stack:** HTML, inline CSS, vanilla JavaScript, Node.js built-in test runner

---

### Task 1: Add the failing sort test

**Files:**
- Modify: `deploy-dashboard.test.mjs`
- Modify: `deploy-dashboard.html`

- [ ] **Step 1: Write the failing test**

```js
test('sortWorkflowEntries orders latest deployments first and keeps empty runs last', async () => {
  const { sortWorkflowEntries } = await loadDashboardHelpers()
  const entries = sortWorkflowEntries([
    { cardId: 'a', index: 0, runs: [{ created_at: '2026-04-14T08:00:00Z' }] },
    { cardId: 'b', index: 1, runs: [] },
    { cardId: 'c', index: 2, runs: [{ created_at: '2026-04-14T09:00:00Z' }] }
  ])

  assert.deepEqual(entries.map(entry => entry.cardId), ['c', 'a', 'b'])
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test deploy-dashboard.test.mjs`
Expected: FAIL because the sort helper does not exist yet

### Task 2: Implement the sorting

**Files:**
- Modify: `deploy-dashboard.html`
- Test: `deploy-dashboard.test.mjs`

- [ ] **Step 3: Write minimal helpers**

```js
function getLatestRunTimestamp(runs) {
  return runs[0]?.created_at ? new Date(runs[0].created_at).getTime() : 0
}

function sortWorkflowEntries(entries) {
  return [...entries].sort(...)
}
```

- [ ] **Step 4: Load a group into entries, then sort before render**

```js
const entries = await Promise.all(workflows.map(...))
const sortedEntries = sortWorkflowEntries(entries)
```

- [ ] **Step 5: Reorder existing cards without recreating them**

```js
sortedEntries.forEach(({ cardId }) => {
  grid.appendChild(document.getElementById(`card-${cardId}`))
})
```

### Task 3: Verify

**Files:**
- Test: `deploy-dashboard.test.mjs`

- [ ] **Step 6: Run tests to verify they pass**

Run: `node --test deploy-dashboard.test.mjs`
Expected: PASS

- [ ] **Step 7: Sanity-check sort helper wiring**

Run: `rg -n "getLatestRunTimestamp|sortWorkflowEntries|reorderGroupCards" deploy-dashboard.html deploy-dashboard.test.mjs`
Expected: helper usage appears in implementation and tests
