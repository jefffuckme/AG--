# Block Tron Icon Design

**Goal:** 将 `deploy-dashboard.html` 中 `Block Tron` 卡片的图标从 Tron 品牌图标替换为更通用的 Tron 链/区块链风格图标。

## Scope

- 仅修改根目录独立页面 `deploy-dashboard.html`
- 仅调整 `Block Tron` 单张卡片的图标定义
- 不改卡片标题、颜色、布局或其他服务卡片

## Design

当前 `Block Tron` 使用 `SIMPLE('tron')`，更偏品牌标识。根据需求，改为链路/区块链通用风格，使用 `LUCIDE('blocks')`，语义更接近区块链节点和链上服务。

## Verification

- 增加页面测试，断言 `Block Tron` 使用 `LUCIDE('blocks')`
- 断言不再使用 `SIMPLE('tron')`
- 运行定向测试验证修改

## Constraints

当前工作区根目录不是 Git 仓库，因此本次仅创建文档文件，不执行提交。
