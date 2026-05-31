# Changelog

## v0.10.1

### Security

- **rustls-webpki 0.103.12 → 0.103.13** — fixes a denial of service via panic on a malformed CRL `BIT STRING` (GHSA-82j2-j2ch-gfr8, high). Reached at runtime through `rustls` (TLS used by `ureq` and `adb_client`).
- **rmcp 0.2.1 → 1.7.0** — clears the Streamable HTTP server transport DNS-rebinding advisory (GHSA-89vp-x53w-74fx, high) and moves onto the supported 1.x line. This server uses only the stdio transport, so the vulnerable path was never reachable, but the upgrade resolves the alert. Internal API migration only — the tool surface is unchanged (same 48 tools, same protocol version).
- **rand 0.8.5 → 0.8.6** — resolves an unsoundness when using a custom logger with `rand::rng()` (GHSA-cq8v-f236-94qc, low).

## v0.9.3

### CDP

- **Collapsed the CDP snapshot surface to DOM-only.** Removed `cdp_take_ax_snapshot` and the paired `a<N>` UID namespace. The native macOS `take_ax_snapshot` (still `a<N>`) and browser-side `cdp_take_dom_snapshot` / `cdp_find_elements` (always `d<N>`) now split cleanly — no more overlapping "which snapshot do I take?" for CDP. **Breaking:** existing callers that invoked `cdp_take_ax_snapshot` must switch to `cdp_find_elements` (for targeted lookups) or `cdp_take_dom_snapshot` (for the full page).
- **Parallelized `DOM.describeNode` resolution.** `cdp_take_dom_snapshot` and `cdp_find_elements` previously did three sequential CDP round trips per element (get ref, describe, release) — for 500 elements that was ~1500 serial round trips. The per-element chain now runs through `futures::join_all`, pipelining over the single CDP WebSocket.
- **`include_snapshot` auto-appends capped at 100 nodes.** `cdp_click` / `cdp_hover` / `cdp_fill` / `cdp_press_key` with `include_snapshot=true` previously appended a full 500-node DOM snapshot; they now append a 100-node snapshot. The user-facing `cdp_take_dom_snapshot(max_nodes=500)` default is unchanged.
- **`cdp_wait_for` snapshot is now opt-in.** Added an `include_snapshot` flag (default `false`). On success the response is now a one-line `"Text appeared after Xms: [...]"` header unless `include_snapshot=true`, in which case a 100-node DOM snapshot is appended after the header. **Breaking:** callers that relied on `cdp_wait_for` implicitly returning a snapshot must pass `include_snapshot=true`.
- **`cdp_element_at_point` description corrected.** Now accurately documents that the tool always returns `backend_node_id` and only carries a `d`-prefixed UID / role / name when the current DOM snapshot already contains the hit-tested node.

## v0.9.2

### macOS

- **`launch_app` gains a `background` flag.** When `true`, the app is launched via `open -g -a`, so it starts without being brought to the foreground. Useful when the next step uses CDP or AX dispatch (both focus-preserving) and you don't want the target window stealing focus. Default is `false`; Windows ignores the flag.

### CDP

- **Label fallback prefers the element's own text nodes.** The v0.9.1 DOM walker still concatenated sibling descendant text when those descendants had no aria/title/alt/role hints, producing composite labels like `"Note to Self 1 week Verified"` on wrapper buttons. `getLabel()` now first concatenates only the element's direct Text-node children and returns immediately on a non-empty result; the prior recursive walk remains as a secondary fallback for wrappers whose visible text lives inside an inner span. Elements with `role` or `data-testid` are also treated as self-contained semantic units so the recursive fallback no longer swallows badge text.

## v0.9.1

### CDP

- **DOM walker no longer returns composite labels.** `getLabel()` previously fell through to `el.textContent` when an element had no `aria-label` / `aria-labelledby` / `title` / `alt`, concatenating all descendant text. A header button wrapping avatar + chat name + badges produced labels like `"Note to Self1 weekVerified"`, which misled agents into clicking the wrong element. Replaced with a direct-text collector that walks only direct text nodes plus descendant subtrees that do not carry their own label and are not themselves interactive; falls back to the tag name when no direct text exists.
- **DOM snapshot now renders parent context.** Each line shows `(in <role> "<name>")` at the end, using the `parentRole` / `parentName` already captured by the walker. Lets a reader disambiguate, for example, a sidebar list item from a chat-header button that would otherwise print the same label.

## v0.9.0

### Element-precise AX dispatch (macOS)

Three macOS-only tools that dispatch against accessibility-tree elements by uid, without moving the cursor or stealing focus. Complement — not replace — coordinate-based `click` / `type_text`.

- **`ax_click`** — press a button, menu item, checkbox, or toolbar item by AX uid via `AXPress`.
- **`ax_set_value`** — write to a text field's `kAXValueAttribute`. Value assignment, not keystroke typing: no `keydown`/`keyup`, no IME composition, no undo-stack entry. Fall back to `click` + `type_text` when key-event semantics are required.
- **`ax_select`** — select a row inside `NSOutlineView` / `NSTableView` by writing `AXSelectedRows` on the enclosing outline/table. Use for sidebars (System Settings, Mail, Xcode, Finder) and rule lists where rows refuse `AXPress`.

All three return `{ ok, dispatched_via, bbox }` on success; on failure, a typed error (`snapshot_expired`, `uid_not_found`, `not_dispatchable`, `no_row_ancestor`, `no_outline_container`, `ax_error`) with an optional `fallback: {x, y}` coordinate for coordinate-based retry.

### Session-stateful `take_ax_snapshot` (macOS)

`take_ax_snapshot` on macOS is now session-backed: each call bumps a monotonic generation and emits uids as `a<N>g<gen>` (e.g. `a42g3`). Uids from prior snapshots are rejected by `ax_click` / `ax_set_value` / `ax_select` with `snapshot_expired`, eliminating the silent wrong-element-clicked failure mode. Snapshot immediately before each dispatch; every branch or retry starts with a fresh snapshot. Windows behavior is unchanged — bare `a<N>` uids, no session.

### MCP tool metadata

- **ToolAnnotations on every tool** — `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` safety hints let MCP clients surface the right permission prompts and defaults.
- **`click` coordinate variants are mutually exclusive** — schema uses `oneOf` (screen / window / screenshot), enforced at runtime. Mixing variants now produces a clear validation error instead of silent coordinate misinterpretation.
- **`focus_window`** returns structured JSON (`{ app_name, pid, kind }`) instead of free-form text.

### CDP

- **CDP tools are listed unconditionally.** Previously they appeared only after `cdp_connect`; they now appear at session start and return a stable "not connected" error until connected, so callers can discover the API up front.

### Dependencies

- `rmcp` bumped to `0.2` to unlock `ToolAnnotations`.
- `rand` and `rustls-webpki` bumped for low-severity advisories.

## v0.8.0

### New tools

- **`cdp_element_at_point`** — resolve the CDP accessibility snapshot UID of the DOM element at given screen coordinates. Returns the element's UID, role, name, and backend_node_id. Bridges native screen coordinates with CDP's DOM model.
- **`probe_app`** — classify an app's automation capabilities (native AX, CDP debug port, embedded debug server) to help agents pick the right tool strategy.

### Fixes

- **Screen recorder** — add `Drop` cleanup and reduce default `max_duration` from 5 minutes to 1 minute to prevent runaway recordings.
- **`cdp_element_at_point`** — validate coordinates and check URL staleness before snapshot lookup to avoid stale results.

## v0.7.1

### Windows fixes

- **Implement `take_ax_snapshot` on Windows** — added `collect_uia_tree` using UI Automation, enabling accessibility tree snapshots on Windows (previously macOS-only)
- **Map all 41 UIA control types** — `take_ax_snapshot` now correctly identifies all standard Windows control types (buttons, tabs, menus, data grids, semantic elements, etc.) instead of falling back to "Unknown"

### Other

- Shorten `server.json` description to meet MCP registry 100-char limit

## v0.7.0

### Chrome DevTools Protocol support

native-devtools-mcp now supports the **Chrome DevTools Protocol (CDP)** — the same protocol that powers Puppeteer, Playwright, and chrome-devtools-mcp. Connect to any Chrome, Chromium, or Electron app and automate it with 16 new tools, all from a single native binary with zero Node.js dependencies.

This means you can now automate **Chrome browsers** and **Electron apps** (Signal, Discord, VS Code, Slack) with DOM-level precision — clicking elements by accessibility UID, filling forms, navigating pages, and evaluating JavaScript — alongside the existing native desktop and Android automation.

#### 16 new `cdp_*` tools

- **`cdp_connect` / `cdp_disconnect`** — connect to a running Chrome/Electron instance on a given port
- **`cdp_take_ax_snapshot`** — accessibility tree snapshot of the browser page (element UIDs prefixed `a`, roles, names)
- **`cdp_take_dom_snapshot`** — DOM-native snapshot of interactive elements (element UIDs prefixed `d`)
- **`cdp_find_elements`** — search the live DOM for interactive elements matching a text query
- **`cdp_evaluate_script`** — evaluate JavaScript in the page, with optional element references from the snapshot
- **`cdp_click`** — click a DOM element by UID (scroll-into-view, more reliable than screen coordinates for web content)
- **`cdp_hover`** — hover over a DOM element by UID
- **`cdp_fill`** — type text into an input/textarea or select an option from a `<select>` element
- **`cdp_press_key`** — press a key or key combination (e.g., `Enter`, `Control+A`, `Control+Shift+R`)
- **`cdp_type_text`** — character-by-character keyboard input into a focused element, with optional submit key
- **`cdp_handle_dialog`** — accept or dismiss JavaScript dialogs (alert, confirm, prompt)
- **`cdp_navigate`** — navigate to a URL, or go back/forward/reload (configurable timeout, handles slow-loading pages)
- **`cdp_new_page`** — create a new browser tab and navigate to a URL
- **`cdp_close_page`** — close a browser tab by index
- **`cdp_wait_for`** — wait for any of multiple texts to appear on the page (lightweight JS polling with timeout)
- **`cdp_list_pages` / `cdp_select_page`** — tab management

`cdp_click`, `cdp_hover`, `cdp_fill`, and `cdp_press_key` support `include_snapshot` to return a fresh snapshot with the action result, saving a round-trip.

#### Getting started

```bash
# Launch Chrome with remote debugging
launch_app(app_name="Google Chrome", args=["--remote-debugging-port=9222", "--user-data-dir=/tmp/chrome-profile"])

# Connect and automate
cdp_connect(port=9222)
cdp_navigate(url="https://example.com")
cdp_take_ax_snapshot()
cdp_fill(uid="a10", value="search query")
cdp_press_key(key="Enter")
```

Chrome 136+ requires `--user-data-dir` alongside `--remote-debugging-port`. Electron apps only need `--remote-debugging-port`.

### Accessibility tree snapshot

New `take_ax_snapshot` tool that serializes the full macOS Accessibility (AX) tree into a structured text format with unique element IDs, roles, and names. Works for any app without requiring a debug port.

## v0.6.0

### Accessibility tree snapshot

New `take_ax_snapshot` tool that serializes the full macOS Accessibility (AX) tree into a structured text format with unique element IDs, roles, and names. Works for any app without requiring a debug port.

### Screen recording

New `start_recording` / `stop_recording` tools for capturing screen activity as MP4 video. Useful for recording UI flows, repro steps, and demo clips.

- Configurable FPS (default 5), region cropping, and max duration
- Supported on macOS (CGWindowListCreateImage) and Windows (BitBlt)

### Windows feature parity

Windows now supports all tools that were previously macOS-only:

- **Hover tracking** — `start_hover_tracking` / `get_hover_events` / `stop_hover_tracking` via UI Automation and GetCursorPos
- **Screen recording** — `start_recording` / `stop_recording` via BitBlt capture loop
- **`element_at_point`** — added `app_name` scoping and container fallback
- **`find_text`** — now searches UIA `Value` and `HelpText` properties in addition to `Name`
- **`get_cursor_position`** — new Windows implementation via GetCursorPos

### Fixes

- **Drag pre-move cursor** — cursor now moves to the start position before initiating a drag, ensuring correct start coordinates (Windows)
- **Hover dwell accuracy** — fixed dwell time calculation to use arrival/departure timestamps correctly, preventing inflated dwell values from pass-through elements
- **Frontmost app detection** — fixed macOS frontmost app resolution to use CGWindowList stacking order instead of NSWorkspace

### Other

- Windows code refactored: deduplicated PID resolution, extracted text property helper, simplified capture_window_jpeg and UIA element search
- Updated rustls-webpki to 0.103.10 (CVE fix)

## v0.5.1

### `element_at_point` improvements

- **AXSubrole** — `element_at_point` now includes the `subrole` field (from `AXSubrole`) in its response on macOS, giving LLMs finer-grained element classification (e.g., distinguishing "AXCloseButton" from a generic button)

### Hover tracking

- **Absolute timestamps** — hover events now use absolute Unix milliseconds instead of relative "ms since tracking started", making it easier to correlate events with external timelines and logs

## v0.5.0

### Hover tracking

New `start_hover_tracking` tool that continuously polls cursor position and the accessibility element under it, recording transitions as the user moves between UI elements. Designed for LLMs to observe user navigation patterns (e.g., tooltip triggers, dropdown reveals, panel expansions).

- **`start_hover_tracking`** — begins a polling session with configurable interval, max duration, and dwell threshold
- **`get_hover_events`** — drains buffered transition events (cursor position, element role/name/bounds, dwell time)
- **`stop_hover_tracking`** — ends the session and returns remaining events
- **Dwell threshold** (`min_dwell_ms`, default 300ms) — filters out pass-through elements during fast mouse movement, so only intentional hovers are recorded
- **Compact output** — element `value` field is dropped to avoid bloat (e.g., terminal buffers); remaining string fields are truncated to 100 chars. Use `element_at_point` with the event's cursor coordinates for full element details
- Tools appear dynamically: `get_hover_events` and `stop_hover_tracking` only show up while a session is active
- Supported on macOS and Windows

### App lifecycle tools

- **`launch_app`** — now accepts optional `args` parameter for CLI arguments (e.g., `--remote-debugging-port=9222`). Returns an error if the app is already running with args specified
- **`quit_app`** — new tool for graceful or force termination of running applications

### Fixes

- `list_apps` / `is_app_running` now uses `CGWindowListCopyWindowInfo` to supplement `NSWorkspace.runningApplications`, fixing stale data for recently launched apps
- `list_apps` filters to user-facing apps only, excluding system agents and daemons
- CI changelog extraction in release workflow fixed

## v0.4.4

### `element_at_point` tool

New tool that returns the accessibility element at given screen coordinates. Given an (x, y) point, returns the element's name, role, label, value, bounds, pid, and app_name. Optional `app_name` parameter scopes the lookup to a specific application (useful when windows overlap). Uses `AXUIElementCopyElementAtPosition` on macOS and `IUIAutomation::ElementFromPoint` on Windows.

### Fixes

- `element_at_point` now drills deeper into Electron/Chromium accessibility trees to return meaningful elements instead of top-level web area containers
- `verify` subcommand now detects source builds and shows an informational message instead of a checksum mismatch error

### Security

- Updated `aws-lc-sys` to 0.38.0 to resolve 3 high-severity vulnerabilities (PKCS7_verify signature validation bypass, PKCS7_verify certificate chain validation bypass, AES-CCM timing side-channel)

## v0.4.3

### `find_text` result ranking

Results are now ranked by relevance: exact matches appear before substring matches, and interactive elements (buttons, links, inputs) rank above static text. A `role` field (from AXRole on macOS, UIA ControlType on Windows) is included in the JSON output.

### `focus_window` fix for bundle-less apps

`focus_window` now reliably brings windows to front for apps without a proper macOS bundle (e.g., Tauri dev builds). After activation, it sets AXFrontmost and AXRaise via the Accessibility API as a fallback.

### Security & trust

- **`verify` subcommand** — hashes the running binary and checks it against official checksums from the GitHub release (exit 0 = verified, exit 1 = mismatch, exit 2 = inconclusive)
- **`setup` subcommand** — guided wizard that checks macOS permissions (Accessibility, Screen Recording) and auto-configures MCP clients (Claude Desktop, Claude Code, Cursor)
- **CI checksums** — every release now publishes `checksums.txt` with SHA-256 hashes for all binaries, archives, and the DMG
- **`SECURITY_AUDIT.md`** — documents which permissions are used, where in the code, and includes an LLM audit prompt
- **`scripts/build-from-source.sh`** — one-liner to clone, review, build, and set up from source

### CLI improvements

- Unknown commands and options now show an error and help text instead of silently starting the MCP server

### Docs

- Added Security & Trust section to README
- Restructured README: `setup` is now the primary post-install path, manual configuration collapsed into a details block

## v0.4.2

### Smarter `find_text` on empty results

When `find_text` (desktop) or `android_find_text` returns no matches, the response now includes an `available_elements` array listing all visible UI element names from the accessibility tree. This lets LLMs see what's actually on screen and retry with the correct name — solving the common issue where accessibility APIs use semantic names (e.g., "multiply" instead of "×", "All Clear" instead of "AC").

Applies to all platforms: macOS (Accessibility API), Windows (UI Automation), and Android (uiautomator).

### Fixes

- Fixed server.json description exceeding MCP Registry 100-character limit
- Removed outdated Android feature flag references from README

## v0.4.1

- Added MCP Registry publishing to the release workflow
- Server is now discoverable at `io.github.sh3ll3x3c/native-devtools` on the MCP Registry

## v0.4.0

### Android support included by default

Android device automation is now built into every release — no feature flags, no separate builds. Install from npm and the `android_*` tools are ready to use.

**Getting started:**
1. Connect an Android device with USB debugging enabled
2. Call `android_list_devices` to discover it
3. Call `android_connect(serial='...')` to unlock all Android tools

### Improved LLM instructions

Rewrote the server instructions that LLMs see to prevent tool misrouting between desktop and Android:

- **"Which tools to use" routing section** at the top — clear rules for when to use desktop vs Android vs app debug tools
- **Parallel workflow examples** for both platforms (find text → click)
- **Key differences called out** — no OCR on Android screenshots, no `focus_window` needed, absolute pixel coordinates
- Removed implementation jargon (CGEvent, AppDebugKit) in favor of plain descriptions

### Breaking changes

- The `android` Cargo feature flag has been removed. `adb_client` and `quick-xml` are now default dependencies. If you were building without `--features android`, your binary will now be slightly larger (~1MB) but functionally identical — Android tools remain hidden until you call `android_connect`.

## v0.3.6

- Accessibility API element tree search for `find_text` on macOS (faster, more accurate than OCR alone)
- Added `app_name` and `window_id` params to `find_text` for window-scoped search
- Added `launch_app` tool to start applications by name
- Disabled OCR language correction by default for better UI text detection
- `find_text` returns empty JSON array instead of prose on zero matches
- Windows compatibility fix for screenshot OCR
