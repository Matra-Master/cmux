# cmux: Technical Research & Architecture Report

This report documents the architecture of `cmux`, its implementation of core features, and a guide for replicating this functionality on Linux using `libghostty`.

## 1. High-Level Architecture
`cmux` is a native macOS terminal application that embeds `libghostty` (the Ghostty terminal engine). It follows the "embedder" pattern:
- **Engine**: `libghostty` (Zig) handles terminal emulation, PTY I/O, and OpenGL rendering.
- **Host**: `cmux` (Swift/AppKit/SwiftUI) handles windowing, UI components (sidebar), and high-level features (CLI, agent hooks).
- **Communication**: The host communicates with the engine via a C API (`ghostty.h`). A Unix domain socket server allows external agents and the `cmux` CLI to interact with the host.

## 2. Core Feature Implementation

### 2.1 The Sidebar (Vertical Tabs & Metadata)
The sidebar is implemented in SwiftUI (`VerticalTabsSidebar`) and provides a timeline-focused view of the workspace.
- **Data Model**: Managed by `Workspace.swift`. It tracks a list of `Panel` objects, each representing a terminal surface.
- **Metadata Reporting**:
  - Ghostty surfaces report metadata (Current Working Directory, Git branch, active ports) via OSC sequences or the socket API.
  - The `TerminalController` listens for these updates and refreshes the `Workspace` state.
  - The UI uses Combine bindings to update the sidebar in real-time.

### 2.2 Notification System
`cmux` captures notifications from two sources:
1. **OSC Sequences**: Capture `OSC 9`, `OSC 99`, and `OSC 777` (standard terminal notification codes).
2. **Socket API**: The `cmux notify` command sends a message to the Unix socket, which the host app then displays using `NSUserNotificationCenter`.
- **Storage**: All notifications are kept in `TerminalNotificationStore.swift`, allowing for a "notification history" view.

### 2.3 CLI & Agent Hookability
This is the core strength of `cmux`, positioning it as a "Terminal for Agents."
- **Unix Domain Socket**: Located at `/tmp/cmux.socket`.
- **Protocols**:
  - **v1 (Plain Text)**: Legacy protocol for simple commands.
  - **v2 (JSON-RPC)**: Modern handle-based API (introduced Feb 2026). It supports granular control over windows, workspaces, and panels.
- **Agent Skills**: The repository includes a `skills/` directory providing specialized prompts and guidance for LLM agents to use `cmux` effectively.
- **Browser Automation**: `cmux` integrates an embedded `WKWebView` with agent-first capabilities:
  - **Accessibility Tree Snapshots**: Extracts a structured representation of the web page for LLM processing.
  - **Action Verification**: Commands like `browser fill` support optional post-action snapshots to verify results.
  - **Session Management**: Cookie and proxy sync across separate automation data stores.

## 3. History & Timeline
- **Jan 2025**: Project founded as a native macOS Ghostty embedder with vertical tabs.
- **Feb 2026**: Major evolution into an agent-focused platform.
  - Introduced **v2 JSON-RPC API** for robust automation.
  - Integrated **WKWebView** with specialized browser automation commands.
  - Relicensed to **AGPL-3.0**.
- **Active Development**: Ongoing focus on multi-window automation, specialized agent "skills," and refining the "timeline" sidebar UX.

## 4. Recreating cmux on Linux (libghostty + GTK4)

### 4.1 Prerequisites & Technical Limitations
To build a `cmux`-like app on Linux, you must link against `libghostty`.
*Important Note*: As of the current version, the `embedded` runtime (which powers the library build of Ghostty) is primarily implemented for Darwin (macOS/iOS).
- In `ghostty/src/apprt/embedded.zig`, the `Platform` union currently only contains `macos` and `ios` variants.
- In `ghostty/include/ghostty.h`, `ghostty_platform_u` reflects this limitation.

To replicate `cmux` on Linux, you will need to:
1. **Extend libghostty**: Modify `src/apprt/embedded.zig` to add a Linux-compatible platform tag (e.g., `GHOSTTY_PLATFORM_GTK4`) and include a field for the native window handle (like a `GtkWidget*` or `GdkSurface*`).
2. **Bridge the Renderer**: Ensure the OpenGL renderer in Ghostty can bind to the provided GTK4 surface when running in embedded mode.

### 4.2 Initialization Sequence (C API)
1. **Init**: Call `ghostty_init(argc, argv)` to bootstrap the Zig runtime.
2. **Configuration**:
   ```c
   ghostty_config_t config = ghostty_config_new();
   ghostty_config_load_default_files(config);
   ghostty_config_finalize(config);
   ```
3. **App Creation**: Initialize the `ghostty_app_t` with runtime callbacks.
   ```c
   ghostty_runtime_config_s rt_config = {
       .userdata = my_app_state,
       .wakeup_cb = my_wakeup_handler, // Vital: calls ghostty_app_tick
       .action_cb = my_action_handler, // Handles new windows/tabs/etc.
       .write_clipboard_cb = my_clipboard_writer,
       .read_clipboard_cb = my_clipboard_reader
   };
   ghostty_app_t app = ghostty_app_new(&rt_config, config);
   ```
4. **Surface Creation**:
   - On Linux/GTK4, you'll need to pass the widget handle.
   - You must call `ghostty_surface_set_size` and `ghostty_surface_set_content_scale` once the GTK widget is realized.

### 4.3 Implementing cmux Features on Linux
- **Sidebar**: Use GTK4's `GtkListView` or `AdwViewStack` for the sidebar. Use `libadwaita` for a modern "native" look similar to the macOS version.
- **Socket Server**: Implement a `GDBus` or raw Unix socket listener in your main event loop. This listener will modify your app state and call `ghostty_surface_process_output` or `ghostty_surface_binding_action`.
- **Metadata Capturing**: Intercept terminal actions in your `action_cb`. When Ghostty reports a PWD change or a notification, update your UI.

## 5. Summary for "From Scratch" Implementation
To start, don't try to build the whole app.
1. Build a minimal C/GTK4 app that initializes `libghostty`.
2. Get the "wakeup/tick" loop working (this is the heartbeat of the terminal).
3. Implement the Unix socket server; this is the key to "agent focus."
4. Finally, add the UI overlays (Sidebar) that bind to the data collected via the socket.
