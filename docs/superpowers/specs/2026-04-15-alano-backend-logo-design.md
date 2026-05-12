# Alano Service Label Logo Integration Design

**Goal:** 在根目录独立页面 `deploy-dashboard.html` 的“Alano 后台服务”和“Alano 前端服务”分组标签前集成同一个 Alano logo，同时保持顶部标题仍为“快速部署中心”。

## Scope

- 仅修改根目录页面 `deploy-dashboard.html`
- 复用仓库现有图片资源，不引入构建流程
- 不调整页面其余交互、部署逻辑或卡片布局

## Design

logo 应挂载在服务分组标签，而不是顶部 Hero 标题。实现方式是在分组标签渲染逻辑中，对 `label-backend` 和 `label-frontend` 标签拼接同一个 `<img>` 图标节点；顶部标题区保持原始“快速部署中心”文案。

logo 资源优先复用仓库已有 PNG：`./yekes-web-javascript/public/AlanoGames.png`。这样能避免把上传图片二次落盘，也不会引入额外资源转换步骤。

## Styling

- 为分组标签新增小尺寸图标样式，控制尺寸和对齐
- 保持标签文字和 logo 在同一行显示
- 不改变现有按钮区和响应式结构

## Verification

- 增加页面测试，断言顶部标题仍为“快速部署中心”
- 断言后端和前端分组标签渲染逻辑都包含 logo 图片引用
- 运行 `node --test deploy-dashboard.test.mjs` 进行验证

## Constraints

当前工作区根目录不是 Git 仓库，因此本次仅创建文档文件，不执行设计文档提交。
