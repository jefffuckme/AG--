# Deploy Dashboard Run Initiator Design

**Goal:** Show the initiator for each workflow run in the dashboard list without changing deploy behavior or data loading.

## Scope

- Only update the standalone page [deploy-dashboard.html](/Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.html)
- No backend or API contract change
- No change to workflow dispatch, polling, or log links

## UI Design

- Keep each run item as a compact row with status, content, time, and log link
- Change the middle content area from a single title line to two lines:
  - first line: existing run title
  - second line: gray helper text `发起人：xxx`
- Keep the new line visually secondary so the title remains the primary signal

## Data Mapping

- Read initiator from `run.triggering_actor.login` first
- Fallback to `run.actor.login`
- If neither exists, display `-`

## Implementation Notes

- Extract a small helper for initiator lookup so initial render and incremental patching use the same logic
- Extract a small helper for run-item markup so both render paths stay consistent
- Add only the CSS required for the new two-line layout and secondary metadata text

## Verification

- Add a targeted local test that fails before the helpers/layout exist
- Run the targeted test after the page update
