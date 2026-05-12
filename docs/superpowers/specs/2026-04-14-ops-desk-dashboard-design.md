# Deploy Dashboard Ops Desk Design

## Context

`deploy-dashboard.html` is a standalone GitHub Actions deployment dashboard. The top area currently contains:

- a status summary bar
- a 24-hour deploy activity panel

The user wants those two sections replaced with a single "值班运营台" view while keeping:

- the existing GitHub Actions data sources
- the existing status derivation based on workflow runs
- the existing polling, card refresh, deploy trigger, and service card sections

The new top module must be derived strictly from current GitHub Actions run data. No new backend, storage, or manual annotations are allowed.

## Goals

- Replace the current top summary area with a single operations desk module
- Reuse existing `CARD_RUNS` and workflow metadata as the only runtime data source
- Surface four duty-oriented views:
  - 状态监控
  - 异常告警
  - 值班视图
  - 最近故障
- Keep the lower backend/frontend/wallet service cards unchanged in behavior
- Make alert and incident judgments deterministic and explainable from workflow run fields

## Non-Goals

- No new API endpoints or persistence
- No changes to deployment trigger flows
- No changes to workflow definitions or GitHub repository configuration
- No redesign of the three service-card sections below the top module

## Data Source And Constraints

The new module will continue using the existing page flow:

1. `loadGroup()` fetches workflow runs from GitHub Actions
2. `CARD_RUNS[cardId]` stores the latest runs per workflow
3. workflow metadata comes from `WF_MAP`
4. polling continues through `schedulePoll()`

The top module must be a pure derived view from these in-memory values.

## Derived Status Rules

### Status Monitoring

Each service is classified from its latest available run:

- `正常`: latest run conclusion is `success`
- `部署中`: latest run status is `in_progress` or `queued`
- `失败`: latest run conclusion is `failure`, `cancelled`, or `timed_out`
- `禁用`: workflow metadata has `disabled: true`

The monitoring cards display counts and a short operator-facing hint.

### Alert Rules

Alerts are generated only from real workflow run states. A workflow contributes at most one alert, using the highest-severity match:

1. `连续失败`
   - the latest run is failure-like
   - and the most recent consecutive failure streak is at least 2
2. `失败`
   - the latest run is failure-like but does not meet the streak rule
3. `运行过久`
   - the latest run is `in_progress` or `queued`
   - and runtime exceeds the threshold

Initial runtime threshold: 15 minutes.

Failure-like conclusions include:

- `failure`
- `cancelled`
- `timed_out`

### Duty Queue

The duty queue is a prioritized operator list derived from the same runs. It should rank services by urgency:

1. consecutive failures
2. current failure
3. long-running deployment
4. recently recovered service

`最近恢复` means:

- latest run is `success`
- previous run exists
- previous run conclusion was failure-like

Each item includes a short action hint such as checking logs, confirming whether a deploy is stuck, or continuing observation after recovery.

### Recent Incidents

The incident feed is built from recent failure-like runs across all workflows:

- collect failure-like runs from cached workflow runs
- sort by time descending
- display the latest few entries

Each entry shows:

- service name
- repo or service family context
- environment if derivable from inputs or title
- failure time
- triggering actor
- branch or ref when available
- conclusion

Only real failure-like runs appear here. Running and successful runs are excluded.

## Layout

The new top area becomes one `ops-desk` panel with four internal sections:

1. Header
   - title: 值班运营台
   - subtitle summarizing current duty state
   - last update / refresh context remains visible

2. 状态监控
   - four compact metric cards for 正常 / 部署中 / 失败 / 禁用
   - stronger emphasis on 失败 and 部署中

3. 异常告警
   - compact alert list ordered by severity
   - explicit empty state when no alerts exist

4. 值班视图
   - prioritized operator queue
   - each row includes service, state, and suggested next action

5. 最近故障
   - compact incident timeline/list ordered newest first
   - lower visual intensity than the live alert section

## Interaction

- clicking an alert, duty item, or incident row scrolls to the related service card
- top module refreshes only through the existing page refresh and polling cycle
- no extra polling loop is added
- hover or title attributes can expose longer run metadata without requiring a new detail panel

## Architecture

Implementation should introduce a clear derived-data layer:

- `buildOpsDeskModel()`
  - reads `WF_MAP` and `CARD_RUNS`
  - returns the full view model for monitoring, alerts, duty queue, and recent incidents

- `renderOpsDesk(model)`
  - renders the top module DOM from the model

- helper functions for deterministic run analysis
  - failure-like conclusion detection
  - consecutive failure streak calculation
  - long-running detection
  - recovered-state detection
  - service scrolling/targeting helpers

This keeps data derivation separate from GitHub fetching and card patching.

## Error Handling

- If a workflow has no runs yet, it does not create false alerts or incidents
- If one workflow fetch fails, the rest of the operations desk still renders from available data
- Before first load completes, the top module shows a loading state instead of zeroed metrics
- Empty states must be explicit for alerts, duty queue, and recent incidents

## Testing

The dashboard already has Node-based tests for script helpers. The new work should add coverage for:

- failure-like conclusion detection
- consecutive failure streak calculation
- long-running deployment detection
- recovery detection
- operations desk aggregation ordering and filtering
- HTML-level presence of the new `ops-desk` section instead of the removed old top sections

## Implementation Boundary

This change is intentionally scoped to `deploy-dashboard.html` and its related local test file(s). No frontend repo or backend repo changes are required because the page is a standalone local dashboard that already talks directly to GitHub Actions.

## Review Notes

- Workspace root is not a Git repository, so this spec can be saved locally but cannot be committed at the workspace root without additional repository setup.
