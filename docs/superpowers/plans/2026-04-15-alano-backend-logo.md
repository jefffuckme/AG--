# Alano Service Label Logo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `deploy-dashboard.html` 的“Alano 后台服务”和“Alano 前端服务”分组标签前添加同一个 Alano logo，并保持顶部标题为“快速部署中心”

**Architecture:** 先为静态 HTML 页面补充一个针对顶部标题与前后端分组标签的页面测试，再以最小改动更新分组标签渲染逻辑和标签样式。图片资源直接复用工作区内已有 PNG 文件，避免引入额外构建或资源处理逻辑。

**Tech Stack:** 静态 HTML、内联 CSS、Node.js `node:test`

---

### Task 1: Add Failing Title-And-Section Test

**Files:**
- Modify: `deploy-dashboard.test.mjs`
- Test: `deploy-dashboard.test.mjs`

- [ ] **Step 1: Write the failing test**

```js
test('hero title stays as deploy center and backend plus frontend section labels carry the Alano logo', async () => {
  const html = await loadDashboardHtml()
  assert.match(html, /<h1>快速部署中心<\/h1>/)
  assert.doesNotMatch(html, /<img class="hero-logo"/)
  assert.match(html, /class="section-label-icon"/)
  assert.match(html, /src="\.\/yekes-web-javascript\/public\/AlanoGames\.png"/)
  assert.match(html, /labelClass === 'label-backend' \|\| labelClass === 'label-frontend'/)
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test deploy-dashboard.test.mjs`
Expected: FAIL because the page does not yet render the logo for both backend and frontend section labels.

### Task 2: Implement Backend And Frontend Label Logo Integration

**Files:**
- Modify: `deploy-dashboard.html`
- Test: `deploy-dashboard.test.mjs`

- [ ] **Step 1: Add minimal section label logo styles**

```css
.section-label-icon {
  width: 16px;
  height: 16px;
  object-fit: contain;
  flex: 0 0 auto;
}
```

- [ ] **Step 2: Render the logo for both backend and frontend section labels while keeping the hero title unchanged**

```js
const labelContent = labelClass === 'label-backend' || labelClass === 'label-frontend'
  ? `<img class="section-label-icon" src="./yekes-web-javascript/public/AlanoGames.png" alt="Alano logo">${labelText}`
  : labelText
```

- [ ] **Step 3: Run test to verify it passes**

Run: `node --test deploy-dashboard.test.mjs`
Expected: PASS for the targeted test

### Task 3: Verify Final HTML Output

**Files:**
- Modify: none
- Test: `deploy-dashboard.test.mjs`

- [ ] **Step 1: Re-check final markup**

Run: `rg -n "快速部署中心|section-label-icon|AlanoGames.png|label-backend|label-frontend" deploy-dashboard.html`
Expected: The hero title text, section label icon class, image source, and backend/frontend logo condition are all present in the HTML.
