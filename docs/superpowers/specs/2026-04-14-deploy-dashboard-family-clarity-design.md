# Deploy Dashboard Family Clarity Design

**Goal**
Reduce accidental deployments in `deploy-dashboard.html` by making frontend and backend categories visually obvious and by surfacing the deployment family in the confirmation modal.

**Scope**
- Single file UI change in `deploy-dashboard.html`
- No workflow changes
- No frontend repo or backend repo code changes

**Approved Direction**
- Strengthen section-level separation with high-contrast category banners
- Add stronger per-card family identification so a card still reads as frontend/backend without relying on section position
- Keep current one-click flow, but make the confirmation modal explicitly say whether the target is a frontend or backend deployment
- Preserve existing production warnings and combine them with family warnings

**Families**
- `backend`: Alano Java backend cards
- `frontend`: H5/admin frontend cards
- `wallet-backend`: business wallet Java backend cards, visually distinct from general backend

**Success Criteria**
- A user scanning the dashboard can identify frontend vs backend at a glance
- Clicking deploy shows a confirmation title/body that explicitly names the deployment family
- Production confirmation still remains prominent

**Out of Scope**
- Extra confirmation checkboxes
- Typed confirmation
- Workflow dispatch parameter changes
