# Zellij Workspace Manager Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Zellij plugin (Rust/WASM) that provides a persistent sidebar for managing named workspaces, with Claude Code notification integration and a layout undo stack.

**Architecture:** A `workspace-manager` Zellij plugin renders a sidebar UI and owns all workspace state. Pure business logic (config, state machine, undo) lives in focused modules and is unit-tested with `cargo test`. The plugin entry point wires Zellij events to those modules. A companion shell script `wm-notify` sends pipe messages to the plugin from Claude Code hooks.

**Tech Stack:** Rust (stable), wasm32-wasip1 target, zellij-tile 0.43.x, kdl 6.x (config parsing), serde_json (state file)

**Spec:** `docs/superpowers/specs/2026-03-13-zellij-workspace-manager-design.md`

---

## Chunk 1: Prerequisites + Scaffold + Config Parsing

### Task 1: Install Prerequisites

**Files:** none

- [ ] **Step 1: Install Zellij**
  ```bash
  sudo pacman -S zellij
  zellij --version
  ```
  Expected: `zellij 0.43.1`

- [ ] **Step 2: Install Rust**
  ```bash
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
  source "$HOME/.cargo/env"
  rustc --version && cargo --version
  ```
  Expected: `rustc 1.XX.X (...)` and `cargo 1.XX.X (...)` (any recent stable version)

- [ ] **Step 3: Add WASM target**
  ```bash
  rustup target add wasm32-wasip1
  rustup target list --installed | grep wasm
  ```
  Expected: `wasm32-wasip1 (installed)`

- [ ] **Step 4: Verify zellij-tile crate version**
  ```bash
  cargo search zellij-tile 2>/dev/null | head -3
  ```
  Note the latest version matching 0.43.x — use that exact version in Cargo.toml below.

---

### Task 2: Scaffold Project

**Files:**
- Create: `~/projects/workspace-manager/Cargo.toml`
- Create: `~/projects/workspace-manager/src/lib.rs`
- Create: `~/projects/workspace-manager/.cargo/config.toml`

- [ ] **Step 1: Create project directory**
  ```bash
  mkdir -p ~/projects/workspace-manager/src
  mkdir -p ~/projects/workspace-manager/.cargo
  cd ~/projects/workspace-manager
  git init
  ```

- [ ] **Step 2: Write Cargo.toml**

  Create `~/projects/workspace-manager/Cargo.toml`:
  ```toml
  [package]
  name = "workspace-manager"
  version = "0.1.0"
  edition = "2021"

  [lib]
  crate-type = ["cdylib"]

  [dependencies]
  zellij-tile = "0.43.1"   # verify exact version from Task 1 Step 4
  kdl = "6"
  serde = { version = "1", features = ["derive"] }
  serde_json = "1"

  [profile.release]
  opt-level = "s"           # smaller WASM binary
  ```

- [ ] **Step 3: Write .cargo/config.toml**

  Create `~/projects/workspace-manager/.cargo/config.toml`:
  ```toml
  [build]
  target = "wasm32-wasip1"
  ```

- [ ] **Step 4: Write minimal lib.rs that compiles**

  Create `~/projects/workspace-manager/src/lib.rs`:
  ```rust
  use std::collections::BTreeMap;
  use zellij_tile::prelude::*;

  #[derive(Default)]
  struct WorkspaceManager;

  register_plugin!(WorkspaceManager);

  impl ZellijPlugin for WorkspaceManager {
      fn load(&mut self, _config: BTreeMap<String, String>) {
          subscribe(&[EventType::Key]);
      }

      fn update(&mut self, _event: Event) -> bool {
          false
      }

      fn render(&mut self, _rows: usize, cols: usize) {
          print!("{:width$}", "workspace-manager loading...", width = cols);
      }
  }
  ```

- [ ] **Step 5: Verify it compiles**
  ```bash
  cd ~/projects/workspace-manager
  cargo build --release 2>&1
  ```
  Expected: `Compiling workspace-manager` ... `Finished release`

  If `zellij-tile` version mismatch error appears: update version in Cargo.toml to match crates.io, re-run.

- [ ] **Step 6: Commit**
  ```bash
  git add .
  git commit -m "chore: scaffold workspace-manager plugin"
  ```

---

### Task 3: Workspace Config Types

**Files:**
- Create: `~/projects/workspace-manager/src/workspace.rs`
- Create: `~/projects/workspace-manager/src/config.rs`
- Test: inside `config.rs` as `#[cfg(test)]`

- [ ] **Step 1: Write workspace.rs**

  Create `~/projects/workspace-manager/src/workspace.rs`:
  ```rust
  #[derive(Debug, Clone, PartialEq)]
  pub struct PaneSpec {
      pub name: String,
      pub command: Option<String>,
  }

  #[derive(Debug, Clone, PartialEq)]
  pub struct WorkspaceSpec {
      pub name: String,
      pub root: String,
      pub panes: Vec<PaneSpec>,
  }

  #[derive(Debug, Clone, PartialEq)]
  pub enum NotificationStatus {
      Idle,
      Active,
      Waiting,
  }

  impl Default for NotificationStatus {
      fn default() -> Self {
          NotificationStatus::Idle
      }
  }

  #[derive(Debug, Clone)]
  pub struct Workspace {
      pub spec: WorkspaceSpec,
      pub status: NotificationStatus,
      /// Zellij tab index for this workspace (-1 = not open)
      pub tab_index: i32,
  }

  impl Workspace {
      pub fn new(spec: WorkspaceSpec) -> Self {
          Workspace {
              spec,
              status: NotificationStatus::Idle,
              tab_index: -1,
          }
      }
  }
  ```

- [ ] **Step 2: Write failing test for KDL config parsing**

  Create `~/projects/workspace-manager/src/config.rs`:
  ```rust
  use crate::workspace::{PaneSpec, WorkspaceSpec};

  pub fn parse_workspaces(kdl_text: &str) -> Result<Vec<WorkspaceSpec>, String> {
      todo!()
  }

  #[cfg(test)]
  mod tests {
      use super::*;

      #[test]
      fn test_parse_single_workspace() {
          let input = r#"
  workspace "Nano" {
      root "~/Nano"
      pane name="claude-1" command="claude"
      pane name="git"
  }
  "#;
          let result = parse_workspaces(input).unwrap();
          assert_eq!(result.len(), 1);
          assert_eq!(result[0].name, "Nano");
          assert_eq!(result[0].root, "~/Nano");
          assert_eq!(result[0].panes.len(), 2);
          assert_eq!(result[0].panes[0].name, "claude-1");
          assert_eq!(result[0].panes[0].command, Some("claude".to_string()));
          assert_eq!(result[0].panes[1].name, "git");
          assert_eq!(result[0].panes[1].command, None);
      }

      #[test]
      fn test_parse_multiple_workspaces() {
          let input = r#"
  workspace "Nano" {
      root "~/Nano"
      pane name="shell"
  }
  workspace "Licom" {
      root "~/Licom"
      pane name="claude" command="claude"
  }
  "#;
          let result = parse_workspaces(input).unwrap();
          assert_eq!(result.len(), 2);
          assert_eq!(result[1].name, "Licom");
      }

      #[test]
      fn test_parse_empty_returns_empty_vec() {
          let result = parse_workspaces("").unwrap();
          assert_eq!(result.len(), 0);
      }
  }
  ```

- [ ] **Step 3: Run tests to verify they fail**
  ```bash
  cd ~/projects/workspace-manager
  cargo test --lib config 2>&1
  ```
  Expected: tests compile but fail at runtime — `test tests::test_parse_single_workspace ... FAILED` (panicked at 'not yet implemented')

- [ ] **Step 4: Implement parse_workspaces**

  Replace `todo!()` in `config.rs` with:
  ```rust
  use kdl::KdlDocument;

  pub fn parse_workspaces(kdl_text: &str) -> Result<Vec<WorkspaceSpec>, String> {
      if kdl_text.trim().is_empty() {
          return Ok(vec![]);
      }
      let doc: KdlDocument = kdl_text.parse().map_err(|e: kdl::KdlError| e.to_string())?;
      let mut result = Vec::new();
      for node in doc.nodes() {
          if node.name().value() != "workspace" {
              continue;
          }
          let name = node
              .entries()
              .first()
              .and_then(|e| e.value().as_string())
              .ok_or("workspace node missing name")?
              .to_string();
          let children = node.children().ok_or("workspace node missing block")?;
          let root = children
              .get("root")
              .and_then(|n| n.entries().first())
              .and_then(|e| e.value().as_string())
              .unwrap_or("~")
              .to_string();
          let mut panes = Vec::new();
          for child in children.nodes() {
              if child.name().value() != "pane" {
                  continue;
              }
              let pane_name = child
                  .get("name")
                  .and_then(|e| e.value().as_string())
                  .unwrap_or("pane")
                  .to_string();
              let command = child
                  .get("command")
                  .and_then(|e| e.value().as_string())
                  .map(|s| s.to_string());
              panes.push(PaneSpec { name: pane_name, command });
          }
          result.push(WorkspaceSpec { name, root, panes });
      }
      Ok(result)
  }
  ```

- [ ] **Step 5: Add module declarations to lib.rs**

  Edit `src/lib.rs` — add at top (before `use` lines):
  ```rust
  mod config;
  mod workspace;
  ```

- [ ] **Step 6: Run tests to verify they pass**
  ```bash
  cargo test --lib config 2>&1
  ```
  Expected: `test tests::test_parse_single_workspace ... ok`, all 3 pass

- [ ] **Step 7: Commit**
  ```bash
  git add src/workspace.rs src/config.rs src/lib.rs
  git commit -m "feat: workspace config types and KDL parser"
  ```

---

### Task 4: State File Persistence

**Files:**
- Create: `~/projects/workspace-manager/src/state_file.rs`
- Test: inside `state_file.rs` as `#[cfg(test)]`

The state file lives at `~/.config/zellij/workspace-manager/state.json`. It persists which workspaces are open and their notification status between Zellij restarts.

- [ ] **Step 1: Write failing tests**

  Create `~/projects/workspace-manager/src/state_file.rs`:
  ```rust
  use serde::{Deserialize, Serialize};

  #[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
  pub enum SavedStatus {
      Idle,
      Active,
      Waiting,
  }

  #[derive(Debug, Clone, Serialize, Deserialize)]
  pub struct SavedWorkspace {
      pub name: String,
      pub status: SavedStatus,
      pub tab_index: i32,
  }

  #[derive(Debug, Default, Clone, Serialize, Deserialize)]
  pub struct PersistedState {
      pub workspaces: Vec<SavedWorkspace>,
  }

  pub fn serialize_state(state: &PersistedState) -> Result<String, String> {
      todo!()
  }

  pub fn deserialize_state(json: &str) -> Result<PersistedState, String> {
      todo!()
  }

  #[cfg(test)]
  mod tests {
      use super::*;

      #[test]
      fn test_roundtrip() {
          let state = PersistedState {
              workspaces: vec![
                  SavedWorkspace {
                      name: "Nano".to_string(),
                      status: SavedStatus::Waiting,
                      tab_index: 0,
                  },
                  SavedWorkspace {
                      name: "Licom".to_string(),
                      status: SavedStatus::Idle,
                      tab_index: 1,
                  },
              ],
          };
          let json = serialize_state(&state).unwrap();
          let restored = deserialize_state(&json).unwrap();
          assert_eq!(restored.workspaces.len(), 2);
          assert_eq!(restored.workspaces[0].name, "Nano");
          assert_eq!(restored.workspaces[0].status, SavedStatus::Waiting);
      }

      #[test]
      fn test_deserialize_empty_json_returns_default() {
          let result = deserialize_state("{}").unwrap();
          assert_eq!(result.workspaces.len(), 0);
      }
  }
  ```

- [ ] **Step 2: Run to verify tests fail**
  ```bash
  cargo test --lib state_file 2>&1
  ```
  Expected: tests compile but fail at runtime — `test tests::test_roundtrip ... FAILED` (panicked at 'not yet implemented')

- [ ] **Step 3: Implement**

  Replace the two `todo!()` bodies:
  ```rust
  pub fn serialize_state(state: &PersistedState) -> Result<String, String> {
      serde_json::to_string_pretty(state).map_err(|e| e.to_string())
  }

  pub fn deserialize_state(json: &str) -> Result<PersistedState, String> {
      serde_json::from_str(json).map_err(|e| e.to_string())
  }
  ```

- [ ] **Step 4: Add module to lib.rs**

  Add `mod state_file;` to `src/lib.rs`.

- [ ] **Step 5: Run tests**
  ```bash
  cargo test --lib state_file 2>&1
  ```
  Expected: 2 tests pass

- [ ] **Step 6: Commit**
  ```bash
  git add src/state_file.rs src/lib.rs
  git commit -m "feat: state file serialization"
  ```

---

## Chunk 2: State Machine + Pipe Messages + Undo Stack

### Task 5: Notification State Machine

**Files:**
- Create: `~/projects/workspace-manager/src/app_state.rs`
- Test: inside `app_state.rs`

`AppState` is the central struct the plugin holds. It owns the workspace list and handles all state transitions.

- [ ] **Step 1: Write failing tests**

  Create `~/projects/workspace-manager/src/app_state.rs`:
  ```rust
  use crate::workspace::{Workspace, WorkspaceSpec, NotificationStatus, PaneSpec};

  pub struct AppState {
      pub workspaces: Vec<Workspace>,
      pub focused_index: usize,
      /// maps Zellij pane_id to workspace index
      pub pane_map: std::collections::HashMap<u32, usize>,
  }

  impl AppState {
      pub fn new(specs: Vec<WorkspaceSpec>) -> Self {
          AppState {
              workspaces: specs.into_iter().map(Workspace::new).collect(),
              focused_index: 0,
              pane_map: std::collections::HashMap::new(),
          }
      }

      /// Called when wm-notify sends "waiting pane_id=<id>"
      pub fn notify_waiting(&mut self, pane_id: u32) {
          todo!()
      }

      /// Called when wm-notify sends "active pane_id=<id>"
      pub fn notify_active(&mut self, pane_id: u32) {
          todo!()
      }

      /// Called when user switches focus to workspace at index
      pub fn focus_workspace(&mut self, index: usize) {
          todo!()
      }

      /// Returns the index of the next workspace with Waiting status, wrapping around
      pub fn next_waiting(&self) -> Option<usize> {
          todo!()
      }
  }

  #[cfg(test)]
  mod tests {
      use super::*;

      fn make_spec(name: &str) -> WorkspaceSpec {
          WorkspaceSpec {
              name: name.to_string(),
              root: "~".to_string(),
              panes: vec![PaneSpec { name: "shell".to_string(), command: None }],
          }
      }

      #[test]
      fn test_notify_waiting_sets_status() {
          let mut state = AppState::new(vec![make_spec("Nano"), make_spec("Licom")]);
          state.pane_map.insert(42, 1); // pane 42 belongs to workspace index 1
          state.notify_waiting(42);
          assert_eq!(state.workspaces[1].status, NotificationStatus::Waiting);
          assert_eq!(state.workspaces[0].status, NotificationStatus::Idle);
      }

      #[test]
      fn test_notify_active_clears_waiting() {
          let mut state = AppState::new(vec![make_spec("Nano")]);
          state.pane_map.insert(10, 0);
          state.notify_waiting(10);
          state.notify_active(10);
          assert_eq!(state.workspaces[0].status, NotificationStatus::Active);
      }

      #[test]
      fn test_focus_workspace_clears_waiting_badge() {
          let mut state = AppState::new(vec![make_spec("Nano"), make_spec("Licom")]);
          state.workspaces[1].status = NotificationStatus::Waiting;
          state.focus_workspace(1);
          assert_eq!(state.focused_index, 1);
          assert_eq!(state.workspaces[1].status, NotificationStatus::Idle);
      }

      #[test]
      fn test_next_waiting_returns_correct_index() {
          let mut state = AppState::new(vec![
              make_spec("Nano"), make_spec("Licom"), make_spec("Elsa")
          ]);
          state.workspaces[2].status = NotificationStatus::Waiting;
          assert_eq!(state.next_waiting(), Some(2));
      }

      #[test]
      fn test_next_waiting_returns_none_when_none_waiting() {
          let state = AppState::new(vec![make_spec("Nano")]);
          assert_eq!(state.next_waiting(), None);
      }

      #[test]
      fn test_unknown_pane_id_is_ignored() {
          let mut state = AppState::new(vec![make_spec("Nano")]);
          // pane 99 not in pane_map — must not panic
          state.notify_waiting(99);
          assert_eq!(state.workspaces[0].status, NotificationStatus::Idle);
      }
  }
  ```

- [ ] **Step 2: Run to verify tests fail**
  ```bash
  cargo test --lib app_state 2>&1
  ```
  Expected: compile errors on `todo!()`

- [ ] **Step 3: Implement**

  Replace `todo!()` bodies in `app_state.rs`:
  ```rust
  pub fn notify_waiting(&mut self, pane_id: u32) {
      if let Some(&idx) = self.pane_map.get(&pane_id) {
          self.workspaces[idx].status = NotificationStatus::Waiting;
      }
  }

  pub fn notify_active(&mut self, pane_id: u32) {
      if let Some(&idx) = self.pane_map.get(&pane_id) {
          self.workspaces[idx].status = NotificationStatus::Active;
      }
  }

  pub fn focus_workspace(&mut self, index: usize) {
      self.focused_index = index;
      if let Some(ws) = self.workspaces.get_mut(index) {
          ws.status = NotificationStatus::Idle;
      }
  }

  pub fn next_waiting(&self) -> Option<usize> {
      self.workspaces
          .iter()
          .enumerate()
          .find(|(_, ws)| ws.status == NotificationStatus::Waiting)
          .map(|(i, _)| i)
  }
  ```

- [ ] **Step 4: Add module to lib.rs**

  Add `mod app_state;` to `src/lib.rs`.

- [ ] **Step 5: Run tests**
  ```bash
  cargo test --lib app_state 2>&1
  ```
  Expected: 6 tests pass

- [ ] **Step 6: Commit**
  ```bash
  git add src/app_state.rs src/lib.rs
  git commit -m "feat: notification state machine"
  ```

---

### Task 6: Pipe Message Parsing

**Files:**
- Create: `~/projects/workspace-manager/src/pipe.rs`

- [ ] **Step 1: Write failing tests**

  Create `~/projects/workspace-manager/src/pipe.rs`:
  ```rust
  #[derive(Debug, PartialEq)]
  pub enum PipeEvent {
      Waiting { pane_id: u32 },
      Active { pane_id: u32 },
  }

  /// Parse a raw pipe payload string like "waiting pane_id=42"
  pub fn parse_pipe_message(msg: &str) -> Option<PipeEvent> {
      todo!()
  }

  #[cfg(test)]
  mod tests {
      use super::*;

      #[test]
      fn test_parse_waiting() {
          let result = parse_pipe_message("waiting pane_id=42");
          assert_eq!(result, Some(PipeEvent::Waiting { pane_id: 42 }));
      }

      #[test]
      fn test_parse_active() {
          let result = parse_pipe_message("active pane_id=7");
          assert_eq!(result, Some(PipeEvent::Active { pane_id: 7 }));
      }

      #[test]
      fn test_unknown_event_returns_none() {
          let result = parse_pipe_message("unknown pane_id=1");
          assert_eq!(result, None);
      }

      #[test]
      fn test_missing_pane_id_returns_none() {
          let result = parse_pipe_message("waiting");
          assert_eq!(result, None);
      }

      #[test]
      fn test_malformed_pane_id_returns_none() {
          let result = parse_pipe_message("waiting pane_id=abc");
          assert_eq!(result, None);
      }
  }
  ```

- [ ] **Step 2: Run to verify tests fail**
  ```bash
  cargo test --lib pipe 2>&1
  ```

- [ ] **Step 3: Implement**

  Replace `todo!()`:
  ```rust
  pub fn parse_pipe_message(msg: &str) -> Option<PipeEvent> {
      let parts: Vec<&str> = msg.trim().splitn(2, ' ').collect();
      let event_type = parts.first()?;
      let pane_id = parts
          .get(1)
          .and_then(|s| s.strip_prefix("pane_id="))
          .and_then(|v| v.parse::<u32>().ok())?;
      match *event_type {
          "waiting" => Some(PipeEvent::Waiting { pane_id }),
          "active" => Some(PipeEvent::Active { pane_id }),
          _ => None,
      }
  }
  ```

- [ ] **Step 4: Add module to lib.rs**

  Add `mod pipe;` to `src/lib.rs`.

- [ ] **Step 5: Run tests**
  ```bash
  cargo test --lib pipe 2>&1
  ```
  Expected: 5 tests pass

- [ ] **Step 6: Commit**
  ```bash
  git add src/pipe.rs src/lib.rs
  git commit -m "feat: pipe message parsing"
  ```

---

### Task 7: Undo Stack

**Files:**
- Create: `~/projects/workspace-manager/src/undo.rs`

- [ ] **Step 1: Write failing tests**

  Create `~/projects/workspace-manager/src/undo.rs`:
  ```rust
  /// Operations that can be undone.
  /// Each variant stores the data needed to reverse the operation.
  /// Note: ClosedPane undo (reopening panes) is deferred to post-v0.1
  /// because the correct Zellij 0.43 API for opening a named pane with a
  /// specific command needs to be verified. Rename and reorder are v0.1 scope.
  #[derive(Debug, Clone)]
  pub enum Operation {
      RenamedWorkspace { index: usize, old_name: String },
      ReorderedWorkspaces { old_order: Vec<String> },
  }

  pub struct UndoStack {
      ops: Vec<Operation>,
  }

  impl UndoStack {
      pub fn new() -> Self {
          UndoStack { ops: Vec::new() }
      }

      pub fn push(&mut self, op: Operation) {
          self.ops.push(op);
      }

      pub fn pop(&mut self) -> Option<Operation> {
          self.ops.pop()
      }

      pub fn is_empty(&self) -> bool {
          self.ops.is_empty()
      }

      pub fn len(&self) -> usize {
          self.ops.len()
      }
  }

  #[cfg(test)]
  mod tests {
      use super::*;

      #[test]
      fn test_push_and_pop() {
          let mut stack = UndoStack::new();
          stack.push(Operation::RenamedWorkspace {
              index: 0,
              old_name: "Nano".to_string(),
          });
          assert_eq!(stack.len(), 1);
          let op = stack.pop().unwrap();
          assert!(matches!(op, Operation::RenamedWorkspace { old_name, .. } if old_name == "Nano"));
          assert!(stack.is_empty());
      }

      #[test]
      fn test_pop_empty_returns_none() {
          let mut stack = UndoStack::new();
          assert_eq!(stack.pop().is_none(), true);
      }

      #[test]
      fn test_lifo_order() {
          let mut stack = UndoStack::new();
          stack.push(Operation::RenamedWorkspace { index: 0, old_name: "First".to_string() });
          stack.push(Operation::RenamedWorkspace { index: 1, old_name: "Second".to_string() });
          let top = stack.pop().unwrap();
          assert!(matches!(top, Operation::RenamedWorkspace { old_name, .. } if old_name == "Second"));
      }
  }
  ```

- [ ] **Step 2: Run tests**
  ```bash
  cargo test --lib undo 2>&1
  ```
  Expected: 3 tests pass (`test_push_and_pop`, `test_pop_empty_returns_none`, `test_lifo_order`)

- [ ] **Step 3: Add module to lib.rs**

  Add `mod undo;` to `src/lib.rs`.

- [ ] **Step 4: Commit**
  ```bash
  git add src/undo.rs src/lib.rs
  git commit -m "feat: undo stack"
  ```

---

## Chunk 3: Rendering + Plugin Wiring + wm-notify + Install

### Task 8: Sidebar Renderer

**Files:**
- Create: `~/projects/workspace-manager/src/render.rs`

The renderer writes ANSI escape sequences to stdout. Zellij's `render()` function is called each time the plugin needs to repaint. The sidebar is 28 columns wide.

- [ ] **Step 1: Write render.rs**

  Create `~/projects/workspace-manager/src/render.rs`:
  ```rust
  use crate::app_state::AppState;
  use crate::workspace::NotificationStatus;

  // ANSI color helpers
  const RESET: &str = "\x1b[0m";
  const DIM: &str = "\x1b[2m";
  const BOLD: &str = "\x1b[1m";
  const PURPLE: &str = "\x1b[38;5;135m";
  const AMBER: &str = "\x1b[38;5;214m";
  const GREEN: &str = "\x1b[38;5;71m";
  const BLUE: &str = "\x1b[38;5;75m";
  const GRAY: &str = "\x1b[38;5;240m";
  const BG_DARK: &str = "\x1b[48;5;234m";
  const BG_ACTIVE: &str = "\x1b[48;5;235m";

  pub fn render(state: &AppState, rows: usize, cols: usize) {
      // Header
      println!("{}{}{:<width$}{}", BG_DARK, GRAY, " WORKSPACES", RESET, width = cols);

      let max_workspaces = rows.saturating_sub(4); // reserve rows for header + footer
      for (i, ws) in state.workspaces.iter().enumerate().take(max_workspaces) {
          let is_focused = i == state.focused_index;
          let (border_color, bg, name_color) = match (&ws.status, is_focused) {
              (_, true) => (PURPLE, BG_ACTIVE, BOLD),
              (NotificationStatus::Waiting, false) => (AMBER, BG_DARK, AMBER),
              _ => (GRAY, BG_DARK, RESET),
          };

          // Left border character (full-height block for active indicator)
          let border = if is_focused || ws.status == NotificationStatus::Waiting {
              "▌"
          } else {
              " "
          };

          // Name line
          let display_name = truncate(&ws.spec.name, cols.saturating_sub(4));
          println!(
              "{}{}{}{} {}{:<width$}{}",
              bg, border_color, border, name_color, display_name,
              "", RESET,
              width = cols.saturating_sub(3)
          );

          // Status badge line
          let badge = status_badge(&ws.status);
          println!(
              "{}{} {}{:<width$}{}",
              bg, border_color, GRAY, badge,
              RESET, width = cols.saturating_sub(3)
          );

          // Blank separator
          if i < state.workspaces.len() - 1 {
              println!("{}{}{}", bg, " ".repeat(cols), RESET);
          }
      }

      // Footer keybind hint (last 2 rows)
      let footer_row = rows.saturating_sub(2);
      // Move cursor to footer area — Zellij repaints from top so we just print remaining rows
      let printed_rows = 1 + state.workspaces.len().min(max_workspaces) * 3;
      for _ in printed_rows..footer_row {
          println!("{}{}{}", BG_DARK, " ".repeat(cols), RESET);
      }
      println!("{}{}{:<width$}{}", BG_DARK, GRAY, " n new  d del  s snap", RESET, width = cols);
      println!("{}{}{:<width$}{}", BG_DARK, GRAY, " tab:next ⏳  ^z undo", RESET, width = cols);
  }

  fn status_badge(status: &NotificationStatus) -> String {
      match status {
          NotificationStatus::Idle => format!("{}idle{}", DIM, RESET),
          NotificationStatus::Active => format!("{}● active{}", GREEN, RESET),
          NotificationStatus::Waiting => format!("{}⏳ waiting{}", AMBER, RESET),
      }
  }

  fn truncate(s: &str, max: usize) -> &str {
      if s.len() <= max {
          s
      } else {
          &s[..max]
      }
  }
  ```

- [ ] **Step 2: Add module to lib.rs**

  Add `mod render;` to `src/lib.rs`.

- [ ] **Step 3: Verify it compiles**
  ```bash
  cargo build --release 2>&1
  ```
  Expected: compiles without errors

- [ ] **Step 4: Commit**
  ```bash
  git add src/render.rs src/lib.rs
  git commit -m "feat: sidebar renderer"
  ```

---

### Task 9: Plugin Wiring

**Files:**
- Modify: `~/projects/workspace-manager/src/lib.rs`

Wire all modules into the `ZellijPlugin` implementation. This replaces the stub from Task 2.

- [ ] **Step 1: Overwrite lib.rs with full implementation**

  Overwrite `~/projects/workspace-manager/src/lib.rs`:
  ```rust
  mod app_state;
  mod config;
  mod pipe;
  mod render;
  mod state_file;
  mod undo;
  mod workspace;

  use std::collections::BTreeMap;
  use zellij_tile::prelude::*;

  use app_state::AppState;
  use config::parse_workspaces;
  use pipe::{parse_pipe_message, PipeEvent};
  use render::render;
  use undo::{Operation, UndoStack};
  use workspace::WorkspaceSpec;

  const CONFIG_PATH: &str = ".config/zellij/workspace-manager/workspaces.kdl";
  const STATE_PATH: &str = ".config/zellij/workspace-manager/state.json";

  struct WorkspaceManager {
      state: Option<AppState>,
      undo: UndoStack,
      sidebar_focused: bool,
  }

  impl Default for WorkspaceManager {
      fn default() -> Self {
          WorkspaceManager {
              state: None,
              undo: UndoStack::new(),
              sidebar_focused: false,
          }
      }
  }

  register_plugin!(WorkspaceManager);

  impl ZellijPlugin for WorkspaceManager {
      fn load(&mut self, _config: BTreeMap<String, String>) {
          subscribe(&[
              EventType::TabUpdate,
              EventType::PaneUpdate,
              EventType::Key,
              EventType::CustomMessage,
              EventType::ModeUpdate,
          ]);

          // Load config
          let home = std::env::var("HOME").unwrap_or_else(|_| "/root".to_string());
          let config_path = format!("{}/{}", home, CONFIG_PATH);
          let specs = std::fs::read_to_string(&config_path)
              .ok()
              .and_then(|text| parse_workspaces(&text).ok())
              .unwrap_or_default();

          self.state = Some(AppState::new(specs));
      }

      fn update(&mut self, event: Event) -> bool {
          let state = match self.state.as_mut() {
              Some(s) => s,
              None => return false,
          };

          match event {
              Event::Key(key) if self.sidebar_focused => {
                  handle_sidebar_key(state, &mut self.undo, key)
              }
              Event::CustomMessage(_name, payload) => {
                  // wm-notify pipe messages
                  if let Some(pipe_event) = parse_pipe_message(&payload) {
                      match pipe_event {
                          PipeEvent::Waiting { pane_id } => state.notify_waiting(pane_id),
                          PipeEvent::Active { pane_id } => state.notify_active(pane_id),
                      }
                      true
                  } else {
                      false
                  }
              }
              Event::PaneUpdate(manifest) => {
                  // Rebuild pane_id → workspace map from current pane state
                  // Each pane's tab_index maps to a workspace by position
                  state.pane_map.clear();
                  for (tab_index, panes) in manifest.panes.iter() {
                      if let Some(ws_index) = state
                          .workspaces
                          .iter()
                          .position(|ws| ws.tab_index == *tab_index as i32)
                      {
                          for pane_info in panes.values() {
                              state.pane_map.insert(pane_info.id, ws_index);
                          }
                      }
                  }
                  true
              }
              Event::TabUpdate(tabs) => {
                  // Sync tab indices to workspaces by name match
                  for tab in &tabs {
                      if let Some(ws) = state
                          .workspaces
                          .iter_mut()
                          .find(|ws| ws.spec.name == tab.name)
                      {
                          ws.tab_index = tab.position as i32;
                      }
                  }
                  true
              }
              Event::ModeUpdate(mode_info) => {
                  // Track whether this plugin pane has focus
                  self.sidebar_focused = mode_info.style.colors.fg != mode_info.style.colors.bg;
                  // Note: use focus detection from Zellij's PluginFocus event if available in 0.43
                  false
              }
              _ => false,
          }
      }

      fn render(&mut self, rows: usize, cols: usize) {
          if let Some(state) = &self.state {
              render(state, rows, cols);
          }
      }
  }

  fn handle_sidebar_key(state: &mut AppState, undo: &mut UndoStack, key: Key) -> bool {
      match key {
          Key::Up | Key::Char('k') => {
              if state.focused_index > 0 {
                  state.focused_index -= 1;
              }
              true
          }
          Key::Down | Key::Char('j') => {
              if state.focused_index + 1 < state.workspaces.len() {
                  state.focused_index += 1;
              }
              true
          }
          Key::Char('\n') => {
              // Switch to focused workspace tab
              let idx = state.focused_index;
              state.focus_workspace(idx);
              if let Some(ws) = state.workspaces.get(idx) {
                  if ws.tab_index >= 0 {
                      go_to_tab(ws.tab_index as u32 + 1);
                  }
              }
              true
          }
          Key::Char('\t') => {
              // Jump to next waiting workspace
              if let Some(next) = state.next_waiting() {
                  state.focused_index = next;
                  if let Some(ws) = state.workspaces.get(next) {
                      if ws.tab_index >= 0 {
                          go_to_tab(ws.tab_index as u32 + 1);
                      }
                  }
              }
              true
          }
          Key::Ctrl('z') => {
              apply_undo(state, undo);
              true
          }
          _ => false,
      }
  }

  fn apply_undo(state: &mut AppState, undo: &mut UndoStack) {
      match undo.pop() {
          Some(Operation::RenamedWorkspace { index, old_name }) => {
              if let Some(ws) = state.workspaces.get_mut(index) {
                  ws.spec.name = old_name;
              }
          }
          Some(Operation::ReorderedWorkspaces { old_order }) => {
              state.workspaces.sort_by_key(|ws| {
                  old_order.iter().position(|n| n == &ws.spec.name).unwrap_or(usize::MAX)
              });
          }
          None => {}
      }
  }
  ```

  > **Note for implementer — focus detection:** The `ModeUpdate` heuristic may not work reliably. Zellij 0.43 may expose a `PluginFocus` or `Focused` event — check the `EventType` enum in `zellij-tile` docs and prefer that. If keybinds are silently not firing, this is the first thing to check.

  > **Note for implementer — pipe messages:** Zellij 0.43 may deliver pipe messages via a separate `pipe_message_received()` trait method on `ZellijPlugin` rather than `Event::CustomMessage`. Check the `ZellijPlugin` trait in `zellij-tile` 0.43 docs and add that method if needed. The `wm-notify` payload format is identical either way.

  > **Deferred to post-v0.1:** The `n` (new workspace), `d` (delete), and `s` (snapshot) keybinds are listed in the sidebar footer but not yet handled in `handle_sidebar_key`. These require interactive prompting (name input, confirmation dialogs) which depends on Zellij's input API. Add them in a follow-up task after the core rendering and navigation are verified working.

- [ ] **Step 2: Build**
  ```bash
  cargo build --release 2>&1
  ```
  Expected: compiles. Fix any API mismatches by consulting `zellij-tile` 0.43 docs — especially `PaneManifest` field names and `Key` enum variants.

- [ ] **Step 3: Run all unit tests**
  ```bash
  cargo test --lib 2>&1
  ```
  Expected: all prior unit tests still pass

- [ ] **Step 4: Commit**
  ```bash
  git add src/lib.rs
  git commit -m "feat: wire plugin entry point"
  ```

---

### Task 10: Load Plugin into Zellij + Manual Smoke Test

**Files:**
- Create: `~/projects/workspace-manager/workspace-manager.kdl` (Zellij layout file)
- Create: `~/.config/zellij/workspace-manager/workspaces.kdl` (user config)

- [ ] **Step 1: Create user config directory and sample workspaces.kdl**
  ```bash
  mkdir -p ~/.config/zellij/workspace-manager
  ```

  Create `~/.config/zellij/workspace-manager/workspaces.kdl`:
  ```kdl
  workspace "test-ws" {
      root "~"
      pane name="shell"
  }
  ```

- [ ] **Step 2: Create Zellij layout that loads the plugin**

  Create `~/projects/workspace-manager/workspace-manager.kdl`:
  ```kdl
  layout {
      pane size=28 {
          plugin location="file:target/wasm32-wasip1/release/workspace_manager.wasm" {}
      }
      pane
  }
  ```

- [ ] **Step 3: Launch Zellij with the layout**
  ```bash
  cd ~/projects/workspace-manager
  zellij --layout workspace-manager.kdl
  ```
  Expected: Zellij opens with sidebar on left showing "WORKSPACES" header and "test-ws" entry

- [ ] **Step 4: Verify sidebar renders without crashing**

  Resize the terminal window — sidebar should repaint cleanly.

- [ ] **Step 4b: Verify keyboard navigation works**

  Click on the sidebar pane to focus it (or use Zellij's pane navigation — default `Ctrl+P` then arrow keys to reach the plugin pane). Press `↓` then `↑`.
  Expected: the highlighted workspace entry changes. If nothing happens, focus detection is broken — revisit the `ModeUpdate` heuristic in `lib.rs` (see implementer note in Task 9).

- [ ] **Step 5: Commit layout file**
  ```bash
  git add workspace-manager.kdl ~/.config/zellij/workspace-manager/workspaces.kdl || true
  git commit -m "feat: Zellij layout for development testing"
  ```

---

### Task 11: wm-notify Shell Script

**Files:**
- Create: `~/projects/workspace-manager/wm-notify`

- [ ] **Step 1: Write wm-notify**

  Create `~/projects/workspace-manager/wm-notify`:
  ```bash
  #!/usr/bin/env bash
  # wm-notify — send notification event to workspace-manager plugin
  # Usage: wm-notify waiting | wm-notify active
  #
  # Reads $ZELLIJ_PANE_ID from environment (set by Zellij in every pane).
  # Silently exits if not running inside Zellij.

  set -euo pipefail

  EVENT="${1:-}"

  if [[ -z "$EVENT" ]]; then
    echo "Usage: wm-notify <waiting|active>" >&2
    exit 1
  fi

  if [[ -z "${ZELLIJ:-}" ]]; then
    # Not running inside Zellij — no-op
    exit 0
  fi

  PANE_ID="${ZELLIJ_PANE_ID:-0}"

  zellij pipe \
    --plugin workspace-manager \
    --name "wm-event" \
    -- "${EVENT} pane_id=${PANE_ID}"
  ```

- [ ] **Step 2: Make executable**
  ```bash
  chmod +x ~/projects/workspace-manager/wm-notify
  ```

- [ ] **Step 3: Manual test outside Zellij**
  ```bash
  ~/projects/workspace-manager/wm-notify waiting
  ```
  Expected: exits silently (no Zellij env, graceful no-op)

- [ ] **Step 4: Manual test inside Zellij**

  Open a Zellij session with the plugin loaded (from Task 10). In a terminal pane:
  ```bash
  ~/projects/workspace-manager/wm-notify waiting
  ```
  Expected: sidebar entry for the current workspace shows amber `⏳ waiting` badge

- [ ] **Step 5: Commit**
  ```bash
  git add wm-notify
  git commit -m "feat: wm-notify shell script"
  ```

---

### Task 12: install.sh

**Files:**
- Create: `~/projects/workspace-manager/install.sh`

Automates: copying `wm-notify` to `~/.local/bin`, patching fish config with Ctrl+Z bind, patching `~/.claude/settings.json` with hooks.

- [ ] **Step 1: Write install.sh**

  Create `~/projects/workspace-manager/install.sh`:
  ```bash
  #!/usr/bin/env bash
  set -euo pipefail

  SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
  BIN_DIR="$HOME/.local/bin"
  FISH_CONFIG="$HOME/.config/fish/config.fish"
  CLAUDE_SETTINGS="$HOME/.claude/settings.json"

  echo "==> Installing workspace-manager"

  # 1. Build WASM plugin
  echo "==> Building plugin..."
  cd "$SCRIPT_DIR"
  cargo build --release
  echo "    Built: target/wasm32-wasip1/release/workspace_manager.wasm"

  # 2. Install wm-notify to PATH
  echo "==> Installing wm-notify..."
  mkdir -p "$BIN_DIR"
  cp "$SCRIPT_DIR/wm-notify" "$BIN_DIR/wm-notify"
  chmod +x "$BIN_DIR/wm-notify"
  echo "    Installed: $BIN_DIR/wm-notify"

  # 3. Patch fish config (idempotent)
  echo "==> Patching fish config..."
  if ! grep -q "bind \\\\cZ undo" "$FISH_CONFIG" 2>/dev/null; then
    echo "" >> "$FISH_CONFIG"
    echo "# workspace-manager: Ctrl+Z undo while typing" >> "$FISH_CONFIG"
    echo "bind \\cZ undo" >> "$FISH_CONFIG"
    echo "    Added bind to $FISH_CONFIG"
  else
    echo "    Fish bind already present — skipping"
  fi

  # 4. Patch Claude Code settings.json (idempotent, preserves existing hooks)
  echo "==> Patching Claude Code hooks..."
  if [[ -f "$CLAUDE_SETTINGS" ]]; then
    # Check if wm-notify is already wired
    if grep -q "wm-notify" "$CLAUDE_SETTINGS"; then
      echo "    Hooks already present — skipping"
    else
      # Use python3 to merge hooks (avoids sed fragility on JSON)
      python3 - "$CLAUDE_SETTINGS" <<'PYEOF'
  import json, sys
  path = sys.argv[1]
  with open(path) as f:
      data = json.load(f)
  hooks = data.setdefault("hooks", {})
  hooks.setdefault("Stop", [])
  hooks["Stop"].append({"type": "command", "command": "wm-notify waiting"})
  hooks.setdefault("PreToolUse", [])
  hooks["PreToolUse"].append({"type": "command", "command": "wm-notify active"})
  with open(path, "w") as f:
      json.dump(data, f, indent=2)
  print("    Hooks added to", path)
  PYEOF
    fi
  else
    echo "    $CLAUDE_SETTINGS not found — skipping hooks (run Claude Code first)"
  fi

  echo ""
  echo "==> Done. Start Zellij with:"
  echo "    zellij --layout $SCRIPT_DIR/workspace-manager.kdl"
  ```

- [ ] **Step 2: Make executable**
  ```bash
  chmod +x ~/projects/workspace-manager/install.sh
  ```

- [ ] **Step 3: Dry-run test**
  ```bash
  bash -n ~/projects/workspace-manager/install.sh
  ```
  Expected: no syntax errors

- [ ] **Step 4: Run install**
  ```bash
  ~/projects/workspace-manager/install.sh
  ```
  Expected: builds plugin, installs `wm-notify`, patches fish config and Claude settings

- [ ] **Step 5: Verify fish bind**
  ```bash
  grep "cZ" ~/.config/fish/config.fish
  ```
  Expected: `bind \cZ undo`

- [ ] **Step 6: Verify Claude hooks**
  ```bash
  grep "wm-notify" ~/.claude/settings.json
  ```
  Expected: two entries for `wm-notify waiting` and `wm-notify active`

- [ ] **Step 7: Commit**
  ```bash
  git add install.sh
  git commit -m "feat: install script — fish bind, Claude hooks, wm-notify"
  ```

---

### Task 13: End-to-End Smoke Test

- [ ] **Step 1: Launch workspace-manager in Zellij**
  ```bash
  zellij --layout ~/projects/workspace-manager/workspace-manager.kdl
  ```

- [ ] **Step 2: Verify sidebar shows workspaces from workspaces.kdl**

  Expected: sidebar shows workspace entries, keybind hints at bottom

- [ ] **Step 3: Test Ctrl+Z typing undo**

  In a terminal pane, type some text then press Ctrl+Z.
  Expected: last typed character is undone (not process suspended)

- [ ] **Step 4: Test Claude Code notification**

  In a terminal pane, run `claude`. When Claude Code finishes a response and waits for input, the sidebar should show `⏳ waiting` for that workspace.

- [ ] **Step 5: Test Tab to jump to waiting workspace**

  Focus the sidebar (use `Ctrl+P` to enter pane mode in default Zellij config, navigate with arrow keys to the sidebar pane, press Enter to focus it). Then press Tab.
  Expected: cursor jumps to the next workspace with a waiting badge

- [ ] **Step 6: Tag release**
  ```bash
  cd ~/projects/workspace-manager
  git tag v0.1.0
  echo "workspace-manager v0.1.0 complete"
  ```
