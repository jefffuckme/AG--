# Deploy Dashboard Run Initiator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a second metadata line to each deploy run item showing the initiator account.

**Architecture:** Keep the dashboard as a single standalone HTML file, but extract run-item rendering into small helpers so the initial card render and incremental patch path share one source of truth. Use GitHub Actions run fields already returned by the existing API call, with `triggering_actor` as the primary source and `actor` as fallback.

**Tech Stack:** HTML, inline CSS, vanilla JavaScript, Node.js built-in test runner

---

### Task 1: Add the failing test

**Files:**
- Create: `deploy-dashboard.test.mjs`
- Modify: `deploy-dashboard.html`

- [ ] **Step 1: Write the failing test**

```js
test('renderRunItem includes initiator metadata from triggering_actor', async () => {
  const { getRunInitiator, renderRunItem } = await loadHelpers()
  const html = renderRunItem({
    status: 'completed',
    conclusion: 'success',
    display_title: 'deploy-main',
    created_at: '2026-04-14T10:00:00Z',
    html_url: 'https://example.com',
    triggering_actor: { login: 'mingge' }
  })

  assert.equal(getRunInitiator({ triggering_actor: { login: 'mingge' } }), 'mingge')
  assert.match(html, /发起人：mingge/)
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test deploy-dashboard.test.mjs`
Expected: FAIL because the helper functions or metadata markup do not exist yet

### Task 2: Implement the minimal dashboard change

**Files:**
- Modify: `deploy-dashboard.html`
- Test: `deploy-dashboard.test.mjs`

- [ ] **Step 3: Write minimal implementation**

```js
function getRunInitiator(run) {
  return run?.triggering_actor?.login || run?.actor?.login || '-'
}
```

- [ ] **Step 4: Reuse one run-item renderer in both render paths**

```js
function renderRunItem(run) {
  return `
    <div class="run-item">
      ...
      <div class="run-main">
        <span class="run-title">...</span>
        <span class="run-meta">发起人：...</span>
      </div>
      ...
    </div>`
}
```

- [ ] **Step 5: Add the minimal CSS**

```css
.run-main { min-width: 0; flex: 1; }
.run-meta { color: #8b949e; font-size: 11px; }
```

### Task 3: Verify

**Files:**
- Test: `deploy-dashboard.test.mjs`

- [ ] **Step 6: Run test to verify it passes**

Run: `node --test deploy-dashboard.test.mjs`
Expected: PASS

- [ ] **Step 7: Sanity-check the HTML references**

Run: `rg -n "getRunInitiator|renderRunItem|run-meta|发起人：" deploy-dashboard.html deploy-dashboard.test.mjs`
Expected: helper and metadata references appear in both the implementation and test
