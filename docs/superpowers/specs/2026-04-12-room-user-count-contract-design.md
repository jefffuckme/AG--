# Room User Count Contract Design

**Goal:** Replace legacy room user-count fields with a single breaking-change contract across backend and H5 frontend.

## Contract

Single-room responses use:
- `connectedUsers`: users currently connected to the room, including spectators
- `seatedUsers`: users currently seated / in-game, excluding spectators

Cluster responses use:
- `totalConnectedUsers`: total connected users across the cluster
- `totalSeatedUsers`: total seated / in-game users across the cluster

## Breaking Changes

Remove these response fields and all compatibility fallbacks:
- `displayUsers`
- `onlineUsers`
- `gameUsers`
- `seatNum`
- `totalOnlinePlayers`

## Backend Scope

- Update room-related response VOs and schema docs
- Update controller/service population logic
- Update cluster statistics data structures
- Update diagnostics responses to the new names

## Frontend Scope

- Update API models and derived room types
- Update room presence helpers and all call sites
- Update room cards, room management, public rooms, and related tests
- Remove all old-field fallback logic

## Verification

- Backend: module compile for `yudao-module-game-biz`
- Frontend: `pnpm type-check`
- Frontend: targeted room-related tests
