# Research Report: cmux Architecture and Implementation

## Introduction
cmux is a native macOS terminal application built on top of the Ghostty terminal engine. It extends the core terminal functionality with features specifically designed for AI coding agents, such as vertical tabs, integrated notifications, and a scriptable embedded browser.

## Parallel with Ghostty
Ghostty provides a high-performance terminal emulator engine (`libghostty`) with a C API.
- **Ghostty macOS**: Written in Zig (mostly), uses AppKit/Metal for rendering.
- **Ghostty Linux**: Written in Zig, uses GTK4 for rendering.
- **cmux**: A consumer of `libghostty`. It is written in Swift and uses AppKit and SwiftUI for the UI. It links against the same `libghostty` C API that the original Ghostty apps use.

## Historical Timeline
- **Late Jan 2025**: Project started as a fork/extension of Ghostty, focusing on native macOS implementation with Swift.
- **Early Feb 2025**: Core terminal functionality with tab management and basic socket API.
- **Mid Feb 2025**: Addition of NSPopover-based notifications, sidebar metadata (git branch, ports), and a scriptable browser.
- **Late Feb 2025**: Transitioned to AGPL-3.0 license. Introduction of v2 JSON-RPC style socket API for more robust automation.
- **March 2025**: Advanced features like keyboard copy mode, markdown viewer, and multi-window support.

## Key Features Implementation

### 1. Sidebar Architecture
The sidebar is implemented using SwiftUI (`VerticalTabsSidebar.swift` inside `ContentView.swift`).
- **Dynamic Binding**: It binds directly to the `Workspace` object, which acts as a state container for a group of terminal/browser panels.
- **Metadata Reporting**: Shell integration scripts and agents report metadata (git branch, current directory, PR status, listening ports) via the socket API. This metadata is stored in the `Workspace` and automatically reflected in the sidebar.
- **Performance**: Uses `Equatable` view protocols and `ObservedObject` to minimize SwiftUI re-renders during high-frequency terminal updates.
- **Drag-and-Drop**: Implements custom `DropDelegate` for workspace reordering and cross-window tab moves.

### 2. Notification System
Notifications flow through a centralized `TerminalNotificationStore`.
- **Sources**:
    - **Terminal Sequences**: OSC 9, 99, and 777 sequences are captured by the Ghostty engine and delivered to the app via the `GHOSTTY_ACTION_DESKTOP_NOTIFICATION` callback.
    - **CLI/Socket**: The `cmux notify` command allows external processes (like agents) to trigger notifications.
    - **Internal Events**: App-level events (like build completion signals).
- **Processing**: `TerminalNotificationStore` manages unread counts, persistence, and triggers macOS system notifications via `UNUserNotificationCenter`.
- **UI Integration**: Tab items in the sidebar and panes in the workspace show visual indicators (blue dots, rings) when unread notifications are present.

### 3. CLI and Agent Hookability
The CLI (`cmux`) is a thin wrapper that communicates with the app over a Unix domain socket.
- **Dual Protocols**:
    - **v1**: Simple line-based plain text protocol for quick commands.
    - **v2**: JSON-RPC style protocol for complex interactions and rich data exchange.
- **Automation Primitives**:
    - **Terminal Control**: Directly injecting input events into the Ghostty engine.
    - **Screen Reading**: Capturing terminal scrollback and current screen state.
    - **Browser Automation**: A powerful `browser.*` API that uses injected JavaScript to extract an accessibility-tree-like snapshot of the page and perform interactions (click, type, etc.) on `WKWebView`.
- **Metadata reporting**: Primitives like `set-status`, `report-git-branch`, and `log` allow agents to "own" parts of the UI to provide status updates.

## Path to Linux Implementation
To recreate cmux on Linux, one would follow a similar architecture but with different technologies:
1. **Language**: Zig or C/C++ to easily interface with `libghostty` and GTK.
2. **UI Framework**: GTK4 is the natural choice, matching Ghostty's own Linux implementation.
3. **Sidebar**: Use `GtkListView` or `GtkListBox` bound to a data model representing the workspaces.
4. **Notifications**: Use `libnotify` for system-level notifications and internal GTK widgets for in-app indicators.
5. **CLI/Socket**: Implement a similar Unix socket listener. GLib provides excellent support for Unix sockets and JSON parsing (`json-glib`).
6. **Browser**: Use `WebKitGTK` (`WebKitWebView`). It provides similar capabilities to `WKWebView` on macOS, including JavaScript injection and interaction.

## Conclusion
cmux demonstrates that `libghostty` is a powerful, embeddable engine. By treating the terminal as a component rather than an entire application, developers can build rich, specialized environments tailored for modern workflows like AI-assisted coding.
