# Deploy Dashboard Family Clarity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make frontend/backend deployment categories obvious in `deploy-dashboard.html` and add family-aware confirmation copy to reduce mis-deployments.

**Architecture:** Keep the dashboard as a single static HTML file. Add lightweight family metadata helpers in the inline script, introduce stronger section/card styling hooks in the markup/CSS, and verify the expected cues with a small Node-based regression test that reads the HTML source.

**Tech Stack:** Static HTML, inline CSS/JavaScript, Node.js built-in `fs` and `assert`

---

### Task 1: Add Regression Coverage

**Files:**
- Create: `tests/deploy-dashboard.test.mjs`
- Test: `tests/deploy-dashboard.test.mjs`

- [ ] **Step 1: Write the failing test**

```js
assert.match(html, /section-banner section-banner-frontend/)
assert.match(html, /section-banner section-banner-backend/)
assert.match(html, /确认前端部署/)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node tests/deploy-dashboard.test.mjs`
Expected: FAIL because the new visual and confirmation markers do not exist yet.

### Task 2: Implement Family Clarity In Dashboard

**Files:**
- Modify: `deploy-dashboard.html`
- Test: `tests/deploy-dashboard.test.mjs`

- [ ] **Step 1: Add family-aware styling hooks**

Add section banner styles and card family variants for frontend, backend, and wallet backend.

- [ ] **Step 2: Add family metadata helpers**

Add small helper functions that map a workflow to its deployment family and display copy.

- [ ] **Step 3: Update rendered cards**

Render stronger family badges/markers on each card and apply family-specific card classes.

- [ ] **Step 4: Update confirmation modal copy**

Show explicit frontend/backend deployment wording in the title/body while preserving prod warning severity.

- [ ] **Step 5: Run test to verify it passes**

Run: `node tests/deploy-dashboard.test.mjs`
Expected: PASS

### Task 3: Verify Final Behavior

**Files:**
- Modify: `deploy-dashboard.html`
- Test: `tests/deploy-dashboard.test.mjs`

- [ ] **Step 1: Run targeted verification**

Run: `node tests/deploy-dashboard.test.mjs`
Expected: PASS

- [ ] **Step 2: Sanity-check generated HTML markers**

Run: `rg -n "section-banner|family-badge|确认前端部署|确认后端部署" deploy-dashboard.html`
Expected: matching lines for section banners, card family markers, and confirm copy.
