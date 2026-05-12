# Block Tron Icon Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 `deploy-dashboard.html` 的 `Block Tron` 卡片图标替换为更通用的区块链风格图标

**Architecture:** 先补一个只针对 `Block Tron` 图标映射的页面测试，再以最小改动更新工作流配置中的 `icon` 定义。保留卡片结构和渲染逻辑不变，只替换图标源。

**Tech Stack:** 静态 HTML、内联 JavaScript、Node.js `node:test`

---

### Task 1: Add Failing Block Tron Icon Test

**Files:**
- Modify: `deploy-dashboard.test.mjs`
- Test: `deploy-dashboard.test.mjs`

- [ ] **Step 1: Write the failing test**

```js
test('Block Tron uses the blocks chain icon instead of the Tron brand icon', async () => {
  const html = await loadDashboardHtml()
  assert.match(html, /label: 'Block Tron',[\s\S]*icon: LUCIDE\('blocks'\)/)
  assert.doesNotMatch(html, /label: 'Block Tron',[\s\S]*icon: SIMPLE\('tron'\)/)
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test --test-name-pattern "Block Tron uses the blocks chain icon" deploy-dashboard.test.mjs`
Expected: FAIL because the card still uses `SIMPLE('tron')`.

### Task 2: Replace The Icon Mapping

**Files:**
- Modify: `deploy-dashboard.html`
- Test: `deploy-dashboard.test.mjs`

- [ ] **Step 1: Update the icon definition**

```js
{
  label: 'Block Tron',
  icon: LUCIDE('blocks'),
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `node --test --test-name-pattern "Block Tron uses the blocks chain icon" deploy-dashboard.test.mjs`
Expected: PASS
