# Desktop Client Rust Design

## Purpose
Define the Rust-side architecture for a desktop-first Matrix client that can later be reused by mobile and web shells.

The design should reuse the existing Matrix SDK layers instead of duplicating protocol logic in the UI layer.

## Architectural Goal
The desktop app should be split into three boundaries:

1. Platform shell: windowing, menus, shortcuts, native integrations, and rendering.
2. Application core: state coordination, view models, actions, and feature policy.
3. Matrix SDK: sync, room state, timeline, encryption, search, media, and transport.

The first two layers are the app's responsibility. The third layer already exists in `matrix-rust-sdk` and should remain the source of truth for Matrix behavior.

## Grounded SDK Dependencies
The desktop client should build on the existing SDK pieces as follows:

- `matrix-sdk::Client` for login, session restore, room-independent requests, and sync startup.
- `matrix-sdk::Room` for room-specific operations like sending events and reading room state.
- `matrix-sdk-ui::RoomListService` for room list retrieval and active room entries.
- `matrix-sdk-ui::Timeline` for room timelines, event aggregation, and high-level message presentation.
- `matrix-sdk-ui::SyncService` for coordinating the room list and encryption sync loops.
- `matrix-sdk-ui::EncryptionSyncService` for E2EE-related sync responsibilities.
- `matrix-sdk::SlidingSync` and related builders for sync transport.
- Store backends from `matrix-sdk-sqlite`, `matrix-sdk-indexeddb`, or memory stores depending on platform.

## Proposed Rust Crate Split
The desktop client should eventually grow into a small set of Rust crates:

- `client-core`: platform-independent application state, reducers, orchestration, and domain services.
- `client-desktop`: desktop shell, window management, menus, keyboard shortcuts, and platform integration.
- `client-ui`: reusable presentation components and state adapters if a non-native UI framework is used.
- `client-ffi`: optional bindings for mobile or web shells if the desktop core is shared externally.

The key rule is that `client-core` must not depend on desktop-only APIs.

## Core Runtime Model
The runtime should be event-driven.

- User actions enter the core as typed commands.
- Matrix SDK updates enter the core as async streams or callbacks.
- The core converts both into a single application state model.
- The UI reads derived state and renders it.

This keeps state transitions testable and avoids letting the desktop shell directly manipulate Matrix state.

## Core State Buckets
The application state should be split into explicit buckets:

- Session state: active account, device identity, login state, and restore status.
- Workspace state: selected server, room, thread, search context, and focused panel.
- Sync state: connection status, unread sync progress, and service health.
- Timeline state: events, edits, replies, reactions, pins, and media metadata.
- Policy state: tier, retention limits, upload quotas, and feature flags.
- UI state: drawer open state, selected tabs, drafts, modal dialogs, and transient notifications.

The core should keep these as separate sub-states so desktop and mobile shells can derive different layouts from the same data.

## Core Services

### Session Service
Owns sign-in, session restore, logout, device trust, and account rehydration.

Inputs:

- Credentials or existing session tokens.
- Homeserver and account discovery data.

Outputs:

- Signed-in client instance.
- Restored store and device metadata.

### Sync Orchestrator
Owns the lifecycle of `SyncService` and `SlidingSync`.

Responsibilities:

- Start and stop sync loops.
- Merge room list updates and encryption updates into one state stream.
- Surface connection and retry state.
- Trigger initial room and timeline hydration.

### Room Graph Service
Owns server-space-room relationships and navigation data.

Responsibilities:

- Translate Matrix spaces into server-like navigation groups.
- Track room membership, favorites, unread counts, and pinned rooms.
- Support quick room switching and hierarchy traversal.

### Timeline Service
Owns room timeline presentation state.

Responsibilities:

- Merge Matrix events into timeline items.
- Keep reply context and thread grouping stable.
- Surface reactions, edits, redactions, and media attachments.
- Expose jump-to-unread and jump-to-latest anchors.

### Composer Service
Owns message drafting and send preparation.

Responsibilities:

- Maintain per-room drafts.
- Preserve attachments and reply targets.
- Validate content against policy state.
- Dispatch send, edit, reply, reaction, and attachment actions.

### Policy Service
Owns tier-aware behavior.

Responsibilities:

- Enforce upload limits and retention rules.
- Expose feature availability for hosted and self-hosted modes.
- Translate quota and billing data into UI-ready summaries.

### Notification Service
Owns notification eligibility and summary data.

Responsibilities:

- Convert sync events into notification candidates.
- Deduplicate noisy updates.
- Track mentions, replies, and priority rooms.

## Desktop Shell Responsibilities
The desktop shell should stay thin.

It should handle:

- Window creation and layout persistence.
- Native menus and global keyboard shortcuts.
- Drag and drop.
- File picking and media capture permissions.
- Notification integration.
- System tray or dock behavior if needed.

It should not:

- Interpret Matrix event semantics.
- Track unread counts directly.
- Store business policy.
- Rewrite SDK event data.

## Data Flow
1. The shell creates the application core.
2. The core creates or restores a `matrix-sdk::Client`.
3. `matrix-sdk-ui::SyncService` starts room list and encryption sync.
4. The SDK emits updates into the application core.
5. The core derives timeline, room, and policy state.
6. The shell renders that state and dispatches user actions.

## Desktop UX Mapping in Rust
The desktop layout should map directly onto core state slices:

- Left rail: server and room graph service.
- Channel rail: room list service and unread aggregation.
- Main pane: timeline service plus composer service.
- Right rail: selected room detail, members, search, or thread state.

Because the same state model is used everywhere, a future mobile shell can collapse the same slices into tabs and sheets without changing Matrix logic.

## Storage Strategy
Choose the backend by platform and deployment mode.

- Desktop hosted: SQLite store for session, state, and media metadata.
- Desktop self-hosted: SQLite or another local store with configurable retention.
- Web: IndexedDB stores.
- Testing: memory stores for fast deterministic tests.

If store encryption is enabled, it should sit beneath the application core and above raw persistence.

## Threading and Concurrency
The design should avoid shared mutable UI state.

- Long-running sync work should live in async tasks.
- The core should expose immutable snapshots or observable state.
- Commands should return typed results or stream updates.
- Cross-service coordination should happen through a small number of channels or coordinators, not ad hoc locks.

## Error Model
The app should classify errors into a few buckets:

- Auth errors: expired session, invalid credentials, missing device trust.
- Sync errors: connectivity, backoff, server capability issues.
- Room errors: permission denied, missing room, unreadable state.
- Policy errors: quota exceeded, upload denied, feature not enabled.
- Media errors: upload failure, unsupported type, size limit reached.

Each error bucket should have a user-facing summary and a recoverable action where possible.

## Testing Strategy
The Rust design should be testable in layers.

- Unit tests for state reducers and policy rules.
- Service tests for timeline and room graph coordination.
- Integration tests against a Synapse instance for sync and room behavior.
- Snapshot tests for derived UI state and event grouping.
- Platform smoke tests for desktop-specific shell concerns.

## Initial Delivery Order
The first desktop implementation should land in this order:

1. Session restore and login.
2. Sync startup and room list hydration.
3. Room timeline rendering and message send.
4. Replies, edits, reactions, and unread handling.
5. Search, members, and room detail panel.
6. Encryption and notification integration.
7. Voice and media policy surfaces.

## What Not To Do
- Do not embed Matrix protocol logic in the desktop shell.
- Do not make mobile-specific layout decisions in the core state model.
- Do not duplicate timeline aggregation outside `matrix-sdk-ui::Timeline` unless there is a clear reason.
- Do not couple billing or retention logic to the transport layer.
