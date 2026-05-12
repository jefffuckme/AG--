# Deploy Dashboard Latest Run Sort Design

**Goal:** Reorder services inside each dashboard project group so the service with the most recent deployment appears first.

## Scope

- Only update the standalone page [deploy-dashboard.html](/Users/mingge/Documents/IdeaProjects/AG_H5/deploy-dashboard.html)
- Sorting applies within each existing group only
- No backend, API, or workflow-dispatch changes

## Sorting Rule

- Compare each service by the time of its latest workflow run
- Use `runs[0].created_at` as the latest deployment time source
- Sort in descending order, newest first
- Services with no deployment record go to the end
- When two services have the same latest timestamp, keep their original configured order

## Implementation Notes

- Keep card IDs stable so refreshes still preserve form inputs and running-state timers
- Fetch all runs for a group first, then sort entries before initial render
- On refresh and polling, patch each existing card and then reorder the DOM nodes instead of rebuilding the whole grid
- Extract small helpers for latest-run timestamp and group sorting so they can be tested directly

## Verification

- Add a targeted local test for the sorting helper
- Run the targeted test suite after the dashboard update
