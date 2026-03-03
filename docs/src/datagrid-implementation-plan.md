# DataGrid Implementation Plan — Full DataGrip Feature Parity for Zed

This document describes the complete implementation plan for adding a DataGrip-equivalent
data grid and database IDE experience to Zed, with agentic AI integration.

## Table of Contents

- [Existing Foundation](#existing-foundation)
- [Architecture Overview](#architecture-overview)
- [Folder Structure Conventions](#folder-structure-conventions)
- [UI/Interface Specification](#uiinterface-specification)
- [Security Architecture](#security-architecture)
- [Phase 0 — Infrastructure](#phase-0--infrastructure)
- [Phase 1 — Connection & Schema](#phase-1--connection--schema)
- [Phase 2 — Query Editor & Execution](#phase-2--query-editor--execution)
- [Phase 3 — Interactive Data Grid](#phase-3--interactive-data-grid)
- [Phase 4 — Export/Import & Comparison](#phase-4--exportimport--comparison)
- [Phase 5 — AI & Agentic Integration](#phase-5--ai--agentic-integration)
- [Phase 6 — Additional Drivers](#phase-6--additional-drivers)
- [Phase 7 — Advanced & Extensibility](#phase-7--advanced--extensibility)
- [Key Dependencies](#key-dependencies)
- [Feature Coverage Matrix](#feature-coverage-matrix)

---

## Existing Foundation

| Component | Location | Description |
|---|---|---|
| DataTable UI | `crates/ui/src/components/data_table.rs` | Virtualized table component with column resize, scroll, focus (1745 lines) |
| TableRow | same file | Newtype enforcing fixed column count at runtime |
| REPL TableView | `crates/repl/src/outputs/table.rs` | Tabular data rendering from Pandas/Polars (Frictionless Data Schema) |
| sqlez | `crates/sqlez/` | Local SQLite abstraction (connections, statements, migrations) |
| db | `crates/db/` | High-level DB layer with KV store, `query!` macros |
| collab DB | `crates/collab/` | Backend SeaORM with PostgreSQL (collaboration server) |
| Panel system | `crates/workspace/src/dock.rs` | Panel trait, dock positions, activation priority, serialization |
| Settings system | `crates/settings/` | Hierarchical JSON settings with project/global scopes |
| Credentials | `crates/credentials_provider/` | Platform keychain abstraction (macOS/Linux/Windows) |
| Agent tools | `crates/agent/src/tools/` | `AgentTool` trait with Input/Output/run(), tool registry |
| MCP | `crates/context_server/` | Model Context Protocol client for external context servers |
| Mentions | `crates/agent_ui/src/mention_set.rs` | Context mention system for AI (`@file`, `@symbol`, etc.) |
| Extensions | `crates/extension/` | WASM-based extension system with capabilities |

---

## Architecture Overview

### New Crates

Three new crates, following the separation of concerns pattern used across Zed
(e.g., `agent` / `agent_ui` / `context_server`). Full folder layout and
naming conventions are detailed in the
[Folder Structure Conventions](#folder-structure-conventions) section below.

| Crate | Responsibility | Key dependencies |
|---|---|---|
| `database_core` | Business logic, drivers, schema types — no UI | `sqlez`, `sqlx`, `tokio-postgres`, `russh` |
| `database_ui` | All UI: panels, grids, dialogs, editors | `database_core`, `gpui`, `ui`, `workspace` |
| `database_ai` | AI integration: tools, mentions, MCP server | `database_core`, `agent`, `context_server` |

### Core Traits

```rust
// crates/database_core/src/driver.rs

#[async_trait]
pub trait DatabaseDriver: Send + Sync {
    fn driver_name(&self) -> &str;
    fn supported_types(&self) -> Vec<DataType>;
    async fn connect(&self, config: &ConnectionConfig) -> Result<Box<dyn DatabaseConnection>>;
}

#[async_trait]
pub trait DatabaseConnection: Send + Sync {
    // Query execution
    async fn execute_query(&self, sql: &str) -> Result<QueryResult>;
    async fn execute_statement(&self, sql: &str) -> Result<u64>;
    async fn cancel(&self) -> Result<()>;

    // Introspection (3 levels)
    async fn introspect_names(&self) -> Result<Vec<SchemaObject>>;
    async fn introspect_columns(&self, table: &TableRef) -> Result<Vec<ColumnInfo>>;
    async fn introspect_ddl(&self, object: &SchemaObject) -> Result<String>;

    // Metadata
    async fn foreign_keys(&self, table: &TableRef) -> Result<Vec<ForeignKey>>;
    async fn indexes(&self, table: &TableRef) -> Result<Vec<IndexInfo>>;
    async fn explain(&self, sql: &str) -> Result<ExplainPlan>;

    // Schema modification
    async fn execute_ddl(&self, sql: &str) -> Result<()>;
}
```

### Integration Points

```rust
// Panel registration (in crates/zed/src/zed.rs alongside other panels)
impl Panel for DatabaseExplorer {
    fn persistent_name() -> &'static str { "Database Explorer" }
    fn panel_key() -> &'static str { "DatabaseExplorer" }
    fn activation_priority(&self) -> u32 { 15 }
    fn icon(&self, ..) -> Option<IconName> { Some(IconName::Database) }
    fn toggle_action(&self) -> Box<dyn Action> { Box::new(ToggleDatabaseExplorer) }
}

// Agent tools registration (in crates/agent/src/tools.rs macro)
tools!(
    // ... existing tools ...
    ExecuteQueryTool,
    DescribeObjectTool,
    ListObjectsTool,
    ExplainQueryTool,
    ModifyDataTool,
);

// New mention types (in crates/agent_ui/src/mention_set.rs)
enum MentionUri {
    // ... existing variants ...
    DatabaseSchema { connection: String, schema: String },
    DatabaseTable { connection: String, table: String },
    DatabaseQuery { connection: String, sql: String },
}
```

---

## Folder Structure Conventions

All new crates **must** follow Zed's established conventions. These rules are
enforced by CLAUDE.md and are consistent across the entire codebase.

### Rule 1: Named library root file, never `lib.rs`

Every crate uses a descriptively named root file declared in `Cargo.toml`:

```toml
# crates/database_core/Cargo.toml
[lib]
name = "database_core"
path = "src/database_core.rs"
doctest = false
```

Existing examples from the repo:
- `crates/agent/` → `[lib] path = "src/agent.rs"`
- `crates/workspace/` → `[lib] path = "src/workspace.rs"`
- `crates/ui/` → `[lib] path = "src/ui.rs"`
- `crates/gpui/` → `[lib] path = "src/gpui.rs"`
- `crates/project/` → `[lib] path = "src/project.rs"`
- `crates/db/` → `[lib] path = "src/db.rs"`

### Rule 2: Never use `mod.rs`

Subdirectories use a **sibling `.rs` file** with the same name as the directory
to declare and re-export the module. The `mod.rs` pattern is banned.

```
# CORRECT
src/
├── database_core.rs          # crate root, declares `mod tools;`
├── tools.rs                  # sibling file: declares sub-modules, re-exports
└── tools/
    ├── execute_query.rs
    ├── describe_object.rs
    └── list_objects.rs

# WRONG — never do this
src/
└── tools/
    ├── mod.rs               # ← BANNED
    ├── execute_query.rs
    └── ...
```

### Rule 3: Flat modules with logical subdirectories

Small crates keep everything flat under `src/`. Larger crates group related
files into subdirectories but keep the module tree shallow (1 level deep).

```
# Small crate (flat)
crates/credentials_provider/src/
└── credentials_provider.rs          # everything in one file

# Medium crate (flat with siblings)
crates/db/src/
├── db.rs                            # crate root
├── kvp.rs
└── query.rs

# Large crate (flat + subdirectories)
crates/agent/src/
├── agent.rs                         # crate root
├── db.rs                            # sibling module
├── thread.rs                        # sibling module
├── thread_store.rs                  # sibling module
├── tools.rs                         # declares tools/ sub-modules
├── tools/                           # subdirectory for logical grouping
│   ├── create_file.rs
│   ├── edit_file.rs
│   └── ...
├── edit_agent.rs                    # declares edit_agent/ sub-modules
├── edit_agent/                      # subdirectory
│   ├── inline_diff.rs
│   └── ...
└── tests/                           # test subdirectory
    └── ...
```

### Rule 4: Tests location

Tests live in one of two places (never at crate root `tests/`):
- **`src/tests/` subdirectory** — for large test suites (e.g., `agent/src/tests/`)
- **Inline `#[cfg(test)]` modules** — for unit tests within implementation files

### Applying these rules to our 3 new crates

Below is the complete folder structure for the database crates, following
every convention.

```
crates/
├── database_core/
│   ├── Cargo.toml                           # [lib] path = "src/database_core.rs"
│   └── src/
│       ├── database_core.rs                 # Crate root: mod declarations, re-exports
│       ├── driver.rs                        # DatabaseDriver + DatabaseConnection traits
│       ├── schema.rs                        # Schema types (Table, Column, FK, Index)
│       ├── query_executor.rs                # Async query execution with cancellation
│       ├── introspection.rs                 # Multi-level metadata loading with cache
│       ├── connection_pool.rs               # Connection pooling and watchdog
│       ├── history.rs                       # Query history per connection
│       ├── data_edit_history.rs             # Undo/redo stack for data modifications
│       ├── ssh_tunnel.rs                    # SSH tunnel support
│       ├── errors.rs                        # DatabaseError types
│       ├── drivers.rs                       # Declares drivers/ sub-modules
│       ├── drivers/
│       │   ├── sqlite.rs                    # SQLite implementation
│       │   ├── postgres.rs                  # PostgreSQL implementation
│       │   ├── mysql.rs                     # MySQL / MariaDB implementation
│       │   ├── mssql.rs                     # MS SQL Server implementation
│       │   └── duckdb.rs                    # DuckDB implementation
│       └── tests/
│           ├── driver_tests.rs
│           ├── introspection_tests.rs
│           └── query_executor_tests.rs
│
├── database_ui/
│   ├── Cargo.toml                           # [lib] path = "src/database_ui.rs"
│   └── src/
│       ├── database_ui.rs                   # Crate root: panel registration, mod declarations
│       ├── connection_manager.rs            # Entity<ConnectionManager>
│       ├── connection_dialog.rs             # Connection configuration modal
│       ├── database_explorer.rs             # Tree panel (Entity<DatabaseExplorer>)
│       ├── query_editor.rs                  # Specialized SQL buffer
│       ├── result_grid.rs                   # Extended DataTable for query results
│       ├── value_editor.rs                  # Side panel for cell editing
│       ├── record_view.rs                   # Single-row vertical view
│       ├── aggregate_view.rs               # Multi-cell aggregation panel
│       ├── quick_actions_toolbar.rs         # Floating context toolbar on cell selection
│       ├── export.rs                        # Multi-format export
│       ├── import.rs                        # CSV/clipboard import wizard
│       ├── table_designer.rs               # GUI dialog for CREATE/ALTER TABLE
│       ├── explain_plan_viewer.rs           # Execution plan diagram
│       ├── er_diagram.rs                    # Entity-relationship diagram viewer
│       ├── diff.rs                          # Declares diff/ sub-modules
│       ├── diff/
│       │   ├── data_diff.rs                # Data comparison viewer
│       │   └── schema_diff.rs              # Schema comparison + migration DDL
│       └── tests/
│           ├── grid_tests.rs
│           ├── explorer_tests.rs
│           └── export_tests.rs
│
├── database_ai/
│   ├── Cargo.toml                           # [lib] path = "src/database_ai.rs"
│   └── src/
│       ├── database_ai.rs                   # Crate root: mod declarations, tool registration
│       ├── schema_context.rs               # Provide schema context to AI
│       ├── mention_provider.rs             # @db:, @table:, @schema: mentions
│       ├── slash_commands.rs               # /db-schema, /db-query, /db-explain
│       ├── query_generator.rs              # Natural language to SQL
│       ├── query_optimizer.rs              # AI-powered query optimization
│       ├── plan_analyzer.rs               # Execution plan analysis
│       ├── autonomous_agent.rs            # Multi-step autonomous agent with allowlist
│       ├── mcp_server.rs                  # MCP server exposing DB resources/tools
│       ├── tools.rs                        # Declares tools/ sub-modules
│       ├── tools/
│       │   ├── execute_query.rs           # AgentTool: execute_query
│       │   ├── describe_object.rs         # AgentTool: describe_database_object
│       │   ├── list_objects.rs            # AgentTool: list_database_objects
│       │   ├── explain_query.rs           # AgentTool: explain_query
│       │   └── modify_data.rs             # AgentTool: modify_data
│       └── tests/
│           ├── tool_tests.rs
│           └── mention_tests.rs
```

### Crate root file template

Each crate root file follows this pattern:

```rust
// crates/database_core/src/database_core.rs

mod connection_pool;
mod data_edit_history;
mod driver;
mod drivers;
mod errors;
mod history;
mod introspection;
mod query_executor;
mod schema;
mod ssh_tunnel;

pub use driver::{DatabaseConnection, DatabaseDriver};
pub use errors::DatabaseError;
pub use schema::*;

// Re-export drivers
pub use drivers::{mysql, postgres, sqlite};
```

### Subdirectory module file template

```rust
// crates/database_core/src/drivers.rs
// Sibling .rs file for the drivers/ directory — declares sub-modules

mod duckdb;
mod mssql;
mod mysql;
mod postgres;
mod sqlite;

pub use mysql::MysqlDriver;
pub use postgres::PostgresDriver;
pub use sqlite::SqliteDriver;
```

### Cargo workspace registration

Each new crate must be added to the workspace root `Cargo.toml`:

```toml
# /Cargo.toml (workspace root)
[workspace]
members = [
    # ... existing crates ...
    "crates/database_core",
    "crates/database_ui",
    "crates/database_ai",
]
```

And the dependency graph between them:

```toml
# crates/database_ui/Cargo.toml
[dependencies]
database_core = { path = "../database_core" }
gpui = { path = "../gpui" }
ui = { path = "../ui" }
workspace = { path = "../workspace" }
settings = { path = "../settings" }
db = { path = "../db" }

# crates/database_ai/Cargo.toml
[dependencies]
database_core = { path = "../database_core" }
agent = { path = "../agent" }
context_server = { path = "../context_server" }
```

---

## UI/Interface Specification

This section provides a comprehensive design for every visual component, interaction
pattern, and layout decision in the database IDE experience. All designs reference
existing GPUI primitives and Zed workspace APIs discovered through codebase exploration.

Every design decision is grounded in established UX/UI research and principles from
recognized authorities in the field. The references below are woven throughout each
subsection to justify specific choices.

### Design Foundations and References

#### Reference Works

| Work | Author(s) | Key Contribution | Applied To |
|---|---|---|---|
| *The Visual Display of Quantitative Information* (1983) | **Edward Tufte** (Yale) | Data-ink ratio, data density, chartjunk elimination | Grid cell rendering, column layout, NULL indicators, heatmaps |
| *Don't Make Me Think* (2000, rev. 2013) | **Steve Krug** | Cognitive load reduction, self-explanatory design, visual hierarchy | Connection dialog, toolbar, context menus, error banners |
| *About Face: The Essentials of Interaction Design* (4th ed., 2014) | **Alan Cooper**, Robert Reimann, David Cronin, Christopher Noessel | Goal-directed design, sovereign applications, perpetual intermediates, excise reduction | Overall layout, keyboard navigation, vim mode, split editor |
| *Laws of UX* (2020) | **Jon Yablonski** | 21 psychological laws codified for product design | Throughout — specific laws cited below |
| *Data Tables: Four Major User Tasks* (2024) | **Nielsen Norman Group** (Katie Sherwin) | Find, compare, view/edit, take action — the four table tasks | ResultGrid modes, sorting/filtering, selection model |
| *Designing Tables for Desktop Apps* (video) | **NNg** | Frozen headers, zebra striping, column management | Grid rendering, sticky headers, column hide/show |
| *DataGrip UX Survey 2025* | **JetBrains** | Reduce visual complexity, progressive disclosure, keyboard-driven workflow | Explorer panel, toolbar, inline editing |

#### Psychological Laws Applied

**1. Fitts's Law** (Paul Fitts, 1954) — *"The time to reach a target is a
function of distance / target size."*

Applied to:
- **Toolbar buttons**: Execute, Cancel, and Format are large, high-contrast
  buttons on the right side of the toolbar — the most reachable area for
  right-handed mouse users. Execute (most frequent action) is the leftmost
  in the group, nearest to the editor.
- **Context menus**: Actions appear at the cursor position (zero distance).
  Most-used actions ("Copy", "Edit Cell") are placed at the top.
- **Split handle**: 6px height is deliberately larger than visual appearance
  (2px visible border) to increase the hit target without visual clutter.
- **Connection color dots**: 6px in tabs but 16px in the color picker dialog
  (larger target for the less frequent but more important action).

**2. Hick's Law** (William Hick, 1952) — *"Decision time increases with the
number and complexity of choices."*

Applied to:
- **Connection dialog**: Form fields appear progressively. SSH and SSL sections
  are collapsed by default (`Disclosure` component) — only 6 visible fields
  initially instead of 16. This reduces decision paralysis for simple connections
  while keeping advanced options accessible.
- **Context menus**: Different menus per node type in the explorer (connection
  node: 6 items, table node: 7 items, column node: 3 items) rather than a
  single 20-item universal menu.
- **Driver tabs**: The connection dialog shows only the fields relevant to the
  selected driver (SQLite hides host/port/user/password, showing only file path).
- **Filter popover**: Offers 6 filter types (Contains, Equals, Starts with,
  Regex, Is NULL, Is NOT NULL) — within the 5-9 range recommended by Miller.

**3. Miller's Law** (George A. Miller, 1956) — *"The average person can hold
7 ± 2 items in working memory."*

Applied to:
- **Connection colors**: 8 color options (within 7 ± 2) — enough variety
  to distinguish environments without overwhelming.
- **Toolbar actions**: Maximum 6 buttons visible (Execute, Cancel, Explain,
  Format, Record View, Detach) — within working memory capacity.
- **Explorer categories**: 4 categories per schema (Tables, Views, Functions,
  Sequences) — easily remembered.
- **Pagination page size options**: 4 choices (100, 500, 1000, 5000).
- **Aggregate view**: Shows 5 values (Count, Sum, Avg, Min, Max) by default.

**4. Jakob's Law** (Jakob Nielsen) — *"Users spend most of their time on other
sites/apps. They prefer your product to work the same way."*

Applied to:
- **Overall layout**: Left panel + center editor + right panel mirrors
  JetBrains IDEs (DataGrip, IntelliJ), VS Code, and Zed's own layout.
  Users of any modern IDE will feel at home instantly.
- **Tree explorer**: Follows the same expand/collapse pattern as Zed's
  ProjectPanel, macOS Finder, and every file tree users have ever used.
- **Keyboard shortcuts**: `Ctrl+Enter` for execute (matches DataGrip, DBeaver,
  pgAdmin, Azure Data Studio). `Ctrl+Z`/`Ctrl+Shift+Z` for undo/redo.
  `Arrow keys` for grid navigation (matches Excel, Google Sheets).
- **Tab behavior**: Preview tabs (italic) → permanent on edit, pin via icon —
  identical to Zed's existing editor tabs and VS Code.
- **Inline editing**: Double-click to edit (matches Excel, DataGrip, Numbers).
- **Sort indicators**: `▲`/`▼` arrows on column headers (universal convention).

**5. Gestalt Principles** (Max Wertheimer, 1923)

Applied to:
- **Proximity**: Related toolbar buttons are grouped (Execute + Cancel together,
  Format + View mode together) with separators between groups. Column header
  elements (name + sort indicator + filter icon) are tightly spaced.
- **Similarity**: All connection indicators use the same visual pattern
  (colored dot) across every context (explorer, tabs, status bar, breadcrumbs).
  All editable cells use the same blue border on focus. All pending changes use
  the same color language (green/yellow/red).
- **Common Region**: The explorer panel, editor, and grid are visually separated
  by borders (Gestalt common region), making each zone's purpose immediately clear.
- **Figure/Ground**: Selected rows use `element_selected` background color with
  sufficient contrast against both light and dark themes. Edited cell indicator
  triangle (top-left corner) is a strong figure against the cell background.
- **Prägnanz (Simplicity)**: Node icons in the explorer use simple, distinct
  shapes — one icon per object type, no decorative elements.

**6. Tufte's Principles** (Edward Tufte, 1983)

Applied to:
- **High data-ink ratio**: Grid cells contain data only — no decorative borders
  between cells (row borders only, controlled via `show_row_borders`). The striped
  row pattern provides visual scanning support without adding ink. Column resize
  handles are invisible until hover.
- **Data density**: `uniform_list` virtual scrolling enables displaying thousands
  of rows without pagination anxiety. The default view maximizes visible rows
  (28px row height = compact). Monospace font for numbers enables rapid
  columnar scanning.
- **Chartjunk elimination**: No gradient backgrounds, no 3D effects, no
  decorative icons in cells. NULL values use italic + muted color (typographic
  convention) instead of a special icon. Boolean values use colored text
  ("true"/"false") instead of elaborate checkbox widgets.
- **Small multiples**: Record view is essentially a small multiple — each
  row becomes its own mini-table. The explorer tree uses indentation (spatial
  encoding) rather than graphical connectors.
- **Lie factor = 1**: Heatmap colors scale linearly with values. Aggregate
  calculations (Sum, Avg) are shown with appropriate decimal precision.

**7. Cooper's Sovereign Application Model** (Alan Cooper, *About Face*)

Applied to:
- **Perpetual intermediates**: The UI is optimized for users who know the basics
  but haven't memorized everything. Keyboard shortcuts are shown in context menus
  and tooltips (learning aid), but the mouse always works. Vim mode is opt-in,
  not default.
- **Excise reduction**: No wizard for "Run first query" — the user just opens a
  tab and presses Ctrl+Enter. No modal for sorting — click the header. No
  dialog for filtering — inline popover. Every interaction saves a step
  compared to dialog-heavy alternatives like pgAdmin.
- **Sovereign posture**: The database IDE takes over the full workspace when
  active (panel + editor + grid + toolbar + status bar) but coexists with code
  editing in the same window. This mirrors DataGrip's sovereign posture within
  the IntelliJ platform.
- **Vocabulary of idioms**: Column header click-to-sort, drag-to-resize,
  double-click-to-auto-fit are idioms users have learned from Excel and will
  expect here. The split handle drag-to-resize is an idiom from every split
  pane editor.

**8. Krug's Self-Explanatory Design** (Steve Krug, *Don't Make Me Think*)

Applied to:
- **Error banner**: Shows the exact error message, the line number, a "Go to
  Error" action, and a dismiss button — all self-explanatory, no documentation
  needed.
- **Empty states**: "No Connections — Add a database connection to get started.
  [Add Connection]" — tells the user what happened, why, and what to do next.
- **Pending changes bar**: "3 pending changes (1 insert, 1 update, 1 delete)
  [Revert All] [Commit]" — immediately communicates state and available actions.
- **Connection dialog**: Labels next to every field, placeholder text showing
  expected format (e.g., "db.example.com"), test button with inline result
  (green check or red X + message) — zero guesswork.

**9. Progressive Disclosure** (JetBrains / Cooper / NNg)

Applied to:
- **Explorer lazy loading**: Level 0 → Level 1 → Level 2 → Level 3.
  Don't load columns until the user expands a table. Don't load DDL until
  explicitly requested. This matches DataGrip's introspection strategy.
- **Connection dialog**: SSH Tunnel and SSL/TLS sections collapsed by default.
- **Column filter**: Single-line input visible, advanced options (regex, IS NULL)
  revealed via radio buttons below.
- **ValueEditor panel**: Hidden by default, appears on-demand when the user
  needs to inspect a large value.
- **AggregateView**: Appears only when a numeric range is selected — not
  always visible.
- **Quick Actions toolbar**: Configurable visibility
  (`DatabaseSettings.show_quick_actions_toolbar`), disabled by default to
  avoid visual clutter for users who prefer keyboard-only workflows.

**10. NNg's Four Table Tasks** (Nielsen Norman Group, 2024)

Nielsen Norman Group identifies four fundamental user tasks for data tables.
Every feature in our ResultGrid maps to one or more of these tasks:

| NNg Task | Description | Our Features |
|---|---|---|
| **Find** | Locate specific records | Column filter, server-side WHERE, Ctrl+F12 column search, fuzzy filter in explorer |
| **Compare** | Compare values across rows/columns | Frozen headers, striped rows, zebra striping, column resize, multi-column sort |
| **View/Edit** | Read and modify individual values | Inline editing, Record View, Value Editor, type-aware cell rendering |
| **Act** | Take action on records | Delete rows, clone rows, export selection, FK navigation, Copy as SQL/JSON/CSV |

### Design Language: Consistency Principles

Following Krug's "consistency over cleverness" principle and Jakob's Law,
the database IDE reuses Zed's existing design language everywhere:

| Design Element | Zed Convention | Database IDE Usage |
|---|---|---|
| Panel toggle | Dock icon bar with tooltip | DatabaseExplorer, ValueEditor |
| Tab with close button | `Tab` component with `TabCloseSide::End` | QueryEditor tabs |
| Context menu | `ContextMenu::build()` with keyboard navigation | All right-click menus |
| Inline filter | `Entity<Editor>` single-line at panel top | Explorer filter, column filter |
| Split pane divider | `SplitEditorView` drag handle pattern | Query/Result vertical split |
| Loading indicator | `Spinner` component | Tree loading, query executing |
| Error notification | Notification bar with action buttons | `DatabaseError::notify()` |
| Empty state | Centered text + primary action button | No connections, no results |
| Modal with dimmed background | `ModalView` + `fade_out_background()` | ConnectionDialog, DML Preview |
| Breadcrumbs in toolbar | `ToolbarItemView` at `PrimaryLeft` | Connection > Schema path |

### 1. Overall Layout and Navigation Flow

> **Design rationale**: The three-panel layout (navigation + content + detail) follows
> Cooper's "sovereign application" model for tools users spend extended time in. The
> layout mirrors JetBrains IntelliJ/DataGrip (left tree + center editor + bottom/right
> panels) and Zed's own architecture, applying **Jakob's Law** — users already know
> this pattern from every modern IDE. The center pane dominates screen real estate,
> following **Tufte's data density** principle: maximize the space dedicated to actual
> data display.

```
+----------------------------------------------------------------------+
| Toolbar: [Connection > Schema > Table]      [Execute] [Cancel] [Fmt] |
+-------------------+--------------------------------+-----------------+
| Left Dock         | Center Pane(s)                 | Right Dock      |
|                   |                                |                 |
| ┌───────────────┐ | ┌────────────────────────────┐ | ┌─────────────┐ |
| │ Database      │ | │ QueryEditor tab            │ | │ Value       │ |
| │ Explorer      │ | │ (Item trait)               │ | │ Editor      │ |
| │ (Panel trait) │ | │                            │ | │ (Panel)     │ |
| │               │ | │ ┌────────────────────────┐ │ | │             │ |
| │ ▼ Production  │ | │ │ SQL Editor             │ │ | │ JSON/XML/   │ |
| │   ▼ public    │ | │ │ (Entity<Editor>)       │ │ | │ Hex/Image   │ |
| │     ▼ Tables  │ | │ └────────────────────────┘ │ | │ preview     │ |
| │       users   │ | │ ═══════════╤═══════════════ │ | │             │ |
| │       orders  │ | │ ┌──────────┴───────────────┐│ | │             │ |
| │     ▼ Views   │ | │ │ ResultGrid              ││ | │             │ |
| │     ▼ Funcs   │ | │ │ (Table + uniform_list)  ││ | │             │ |
| │               │ | │ │                         ││ | │             │ |
| │ ▼ Staging     │ | │ └─────────────────────────┘│ | │             │ |
| └───────────────┘ | └────────────────────────────┘ | └─────────────┘ |
+-------------------+--------------------------------+-----------------+
| Status Bar: [● Production] [243 rows / 0.12s] [Ctrl+Enter to run]   |
+----------------------------------------------------------------------+
```

#### Component Placement Decisions

| Component | Placement | Zed Trait | Priority | Icon |
|---|---|---|---|---|
| **DatabaseExplorer** | Left dock (default, movable) | `Panel` | 15 | `IconName::DatabaseZap` |
| **QueryEditor** | Center pane tab | `Item` | — | `IconName::Database` + connection color |
| **ResultGrid** | Embedded below QueryEditor (split) | Part of QueryEditor `Render` | — | — |
| **ValueEditor** | Right dock (on-demand) | `Panel` | 16 | `IconName::TextSelect` |
| **ConnectionDialog** | Modal overlay (centered) | `ModalView` | — | — |
| **QueryBreadcrumbs** | Toolbar (PrimaryLeft) | `ToolbarItemView` | — | — |
| **QueryActions** | Toolbar (PrimaryRight) | `ToolbarItemView` | — | — |
| **DatabaseStatusItem** | Status bar (right section) | `StatusItemView` | — | — |
| **AggregateView** | Floating popover on selection | `Popover` (anchored) | — | — |

#### Coexistence with Code Editing

DatabaseExplorer is a dock panel just like ProjectPanel and TerminalPanel. It coexists
in the left dock — users toggle between them via the dock icon bar. QueryEditor tabs
appear in standard Pane tabs alongside code editor tabs, so users can have SQL and Rust
files open simultaneously. The ResultGrid is embedded within the QueryEditor item
(vertical split), not a separate panel, to avoid cluttering the dock.

Panel registration follows the existing pattern in `crates/zed/src/zed.rs`:

```rust
// In initialize_panels(), alongside existing panel registrations
workspace.register_panel::<DatabaseExplorer>(window, cx);
workspace.register_panel::<ValueEditor>(window, cx);
```

### 2. Database Explorer Panel

> **Design rationale**: The tree explorer applies **Gestalt proximity** (nested indentation
> groups related objects), **Gestalt similarity** (consistent icon shapes per type), and
> **progressive disclosure** (NNg / Cooper) — children load only on expand. The 4-level
> lazy loading (L0→L3) mirrors DataGrip's introspection strategy, validated by JetBrains'
> 2025 UX survey as a key performance enabler. The connection color coding uses **Gestalt
> similarity** across all touchpoints (explorer, tabs, status bar) — a single visual cue
> consistently applied, per Krug's "consistency over cleverness."

#### Entity Structure

```rust
pub struct DatabaseExplorer {
    connections: Vec<ConnectionNode>,
    filter_editor: Entity<Editor>,           // Single-line fuzzy filter
    filter_text: String,
    selected_index: Option<usize>,
    scroll_handle: UniformListScrollHandle,
    focus_handle: FocusHandle,
    context_menu: Option<Entity<ContextMenu>>,
    width: Option<Pixels>,
    pending_serialization: Task<()>,
    _subscriptions: Vec<Subscription>,
}
```

#### Tree Hierarchy (4 Levels)

```
▼ 🔌 Production (green border-left 2px)        ← ConnectionNode
  ▼ 📦 mydb                                     ← DatabaseNode
    ▼ 📐 public                                  ← SchemaNode
      ▼ 📋 Tables (3)                            ← CategoryNode
        ▼ users                                   ← TableNode
            id (PK, int4)                          ← ColumnNode
            email (varchar, NOT NULL)              ← ColumnNode
            created_at (timestamptz)               ← ColumnNode
        ▶ orders                                   ← TableNode (collapsed)
        ▶ products                                 ← TableNode (collapsed)
      ▶ 👁 Views (1)                               ← CategoryNode
      ▶ ƒ Functions (5)                            ← CategoryNode
      ▶ 📊 Sequences (2)                           ← CategoryNode
  ▼ 📐 information_schema                        ← SchemaNode
    ...
▶ 🔌 Staging (blue border-left 2px)              ← ConnectionNode (collapsed)
```

#### Rendering Pattern (ProjectPanel-style)

Each tree node renders as a `ListItem` with the following customization:

```rust
// Pseudocode for rendering a tree node
fn render_tree_node(&self, node: &TreeNode, window: &mut Window, cx: &mut App) -> AnyElement {
    ListItem::new(node.id())
        .indent_level(node.depth())
        .indent_step_size(px(20.))
        .toggle(if node.has_children() { Some(node.is_expanded()) } else { None })
        .on_toggle(cx.listener(move |this, _, window, cx| {
            this.toggle_expanded(node_id, window, cx);
        }))
        .start_slot(
            Icon::new(node.icon_name())
                .size(IconSize::Small)
                .color(node.icon_color())
        )
        .end_hover_slot(self.render_hover_actions(node, cx))
        .on_click(cx.listener(move |this, _, window, cx| {
            this.select_node(node_id, window, cx);
        }))
        .on_secondary_mouse_down(cx.listener(move |this, event, window, cx| {
            this.show_context_menu(node_id, event.position, window, cx);
        }))
        .when(node.is_connection(), |item| {
            item.child(
                div()
                    .absolute().left_0().top_0().bottom_0()
                    .w(px(2.))
                    .bg(node.connection_color())
            )
        })
        .child(Label::new(node.display_name()).size(LabelSize::Small))
        .into_any_element()
}
```

#### Node Icons

| Node Type | Icon | Color |
|---|---|---|
| Connection (connected) | `DatabaseZap` | `Color::Success` |
| Connection (disconnected) | `Database` | `Color::Muted` |
| Database | `Package` | `Color::Default` |
| Schema | `Layers` | `Color::Default` |
| Tables (category) | `Table` (new) | `Color::Accent` |
| Views (category) | `Eye` | `Color::Accent` |
| Functions (category) | `Code` | `Color::Accent` |
| Sequences (category) | `Hash` | `Color::Accent` |
| Table | `Table` (new) | `Color::Default` |
| View | `Eye` | `Color::Default` |
| Column (PK) | `Key` (new) | `Color::Warning` |
| Column (FK) | `ArrowUpRight` | `Color::Info` |
| Column (regular) | `Minus` | `Color::Muted` |
| Index | `ListFilter` | `Color::Muted` |

#### Connection Color Coding

Each connection is assigned a color from a palette of 8:

```rust
pub const CONNECTION_COLORS: [Hsla; 8] = [
    hsla(0.0, 0.7, 0.5, 1.0),    // Red — production
    hsla(0.33, 0.7, 0.4, 1.0),   // Green — development
    hsla(0.58, 0.7, 0.5, 1.0),   // Blue — staging
    hsla(0.08, 0.8, 0.5, 1.0),   // Orange — QA
    hsla(0.75, 0.6, 0.5, 1.0),   // Purple — analytics
    hsla(0.47, 0.7, 0.4, 1.0),   // Teal — replica
    hsla(0.89, 0.6, 0.5, 1.0),   // Pink — test
    hsla(0.14, 0.8, 0.5, 1.0),   // Yellow — local
];
```

Color is applied to:
- 2px left border on the connection node in the explorer
- Tab border-bottom on QueryEditor tabs
- Dot indicator in the status bar
- Top border on the ResultGrid header row

#### Fuzzy Filter

- Single-line `Entity<Editor>` at the top of the panel (visible on `Ctrl+F` or always visible)
- Filters all visible nodes using fuzzy matching (reuse `fuzzy` crate)
- Matching ranges highlighted with `Color::Accent` on the label
- Empty state: "No matching objects" with muted text

#### Lazy Loading Strategy

| Level | Trigger | Data Loaded |
|---|---|---|
| L0 | Connection established | Database names only |
| L1 | Expand database/schema | Object names + types (tables, views, functions) |
| L2 | Expand table/view | Column names, types, PK/FK, NOT NULL |
| L3 | Explicit "Load DDL" action | Full DDL / source code |

Loading indicator: `ListItem` with a `Spinner` element in the start slot while loading.

#### Context Menus (Per Node Type)

**Connection node:**
- New Query Console → opens empty QueryEditor tab
- Refresh → re-introspects all schemas
- Disconnect / Reconnect
- Edit Connection... → opens ConnectionDialog
- Duplicate Connection
- Remove Connection

**Table node:**
- Open Table Data → opens QueryEditor with `SELECT * FROM ...` + executes
- Edit Table Data → same but with auto-commit off
- New Query on Table → opens QueryEditor with `SELECT * FROM table`
- Copy Qualified Name → clipboard
- Generate: INSERT / SELECT / UPDATE / CREATE → clipboard
- Drop Table... → confirmation modal

**Column node:**
- Filter by this Column → adds WHERE clause in active QueryEditor
- Copy Column Name
- Sort by this Column

#### Drag-and-Drop

Tables and columns are draggable (using GPUI's `.on_drag()` / `.on_drop()` pattern from
`SplitEditorView`). Dropping onto a QueryEditor inserts the fully-qualified name at cursor.

```rust
// On the tree ListItem
.on_drag(DraggedDatabaseObject { qualified_name, object_type }, |_, _, _, cx| {
    cx.new(|_| Label::new(qualified_name.clone()))
})

// On the QueryEditor
.on_drop::<DraggedDatabaseObject>(cx.listener(|this, payload, window, cx| {
    this.editor.update(cx, |editor, cx| {
        editor.insert(&payload.qualified_name, window, cx);
    });
}))
```

### 3. Connection Dialog

> **Design rationale**: The dialog applies **Hick's Law** aggressively — collapsing SSH
> and SSL sections reduces visible fields from 16 to 6, cutting decision time. The driver
> tab strip applies **Miller's Law** (5 tabs ≤ 7±2). The inline test result (green
> check / red X with latency) follows Krug's self-explanatory principle — the user never
> has to navigate elsewhere to see if the connection works. The 8-color picker follows
> **Miller's Law** (8 ≤ 7+2) and **Gestalt similarity** — the same dot appears everywhere.

#### Entity Structure

```rust
pub struct ConnectionDialog {
    mode: DialogMode,                        // New | Edit(ConnectionId)
    driver_tab: DriverType,                  // PostgreSQL | MySQL | SQLite | ...
    name_editor: Entity<Editor>,
    host_editor: Entity<Editor>,
    port_editor: Entity<Editor>,
    database_editor: Entity<Editor>,
    user_editor: Entity<Editor>,
    password_editor: Entity<Editor>,
    selected_color: usize,                   // Index into CONNECTION_COLORS
    ssh_enabled: bool,
    ssh_host_editor: Entity<Editor>,
    ssh_port_editor: Entity<Editor>,
    ssh_user_editor: Entity<Editor>,
    ssh_key_path_editor: Entity<Editor>,
    ssl_mode: SslMode,                       // Disable | Prefer | Require | VerifyCA | VerifyFull
    ssl_ca_path_editor: Entity<Editor>,
    ssl_cert_path_editor: Entity<Editor>,
    ssl_key_path_editor: Entity<Editor>,
    test_status: TestConnectionStatus,       // Idle | Testing | Success(Duration) | Failed(String)
    focus_handle: FocusHandle,
}

pub enum DialogMode { New, Edit(ConnectionId) }
pub enum TestConnectionStatus {
    Idle,
    Testing,
    Success(Duration),
    Failed(String),
}
```

#### Layout (ModalView)

```
┌──────────────────────────────────────────────────────┐
│  New Connection                                   ✕  │
├──────────────────────────────────────────────────────┤
│  [PostgreSQL] [MySQL] [SQLite] [DuckDB] [MSSQL]     │  ← Driver tabs
│                                                      │
│  Name:     [Production DB_____________________________]│
│  Color:    ● ● ● ● ● ● ● ●                         │  ← 8 color pastilles
│                                                      │
│  Host:     [db.example.com____________________________]│
│  Port:     [5432__]  Database: [myapp________________]│
│  User:     [admin_____________________________________]│
│  Password: [••••••••__________________________________]│
│                                                      │
│  ▶ SSH Tunnel                                        │  ← Disclosure (collapsed)
│  ▶ SSL / TLS                                         │  ← Disclosure (collapsed)
│                                                      │
│  [Test Connection]  ✓ Connected (45ms)               │  ← Test result inline
│                                                      │
├──────────────────────────────────────────────────────┤
│                           [Cancel]  [Save Connection]│
└──────────────────────────────────────────────────────┘
```

#### Implementation Details

- Implements `ModalView` + `EventEmitter<DismissEvent>` + `Focusable`
- `fade_out_background() -> true` for dimmed background overlay
- Driver tabs rendered as a `TabBar` with custom icons per driver
- Each form field is an `Entity<Editor>` in single-line mode
- Password field uses a custom `Editor` with character masking
- Color picker: 8 `div()` circles with `.rounded_full().w(px(16.)).h(px(16.)).bg(color)`,
  selected one gets a `border_2()` ring
- SSH/SSL sections use `Disclosure` components, collapsed by default
- "Test Connection" spawns a background task via `cx.spawn()`, updates `test_status`
- Result shown inline: green checkmark + latency, or red X + error message
- Footer: `ModalFooter` with Cancel (dismisses) and Save (validates + persists)
- SQLite driver tab hides Host/Port/User/Password, shows only file path picker
- Tab key navigates between form fields (standard GPUI focus chain)

### 4. Query Editor (Workspace Item)

> **Design rationale**: The inline split layout (editor above, results below) follows
> Cooper's **excise reduction** — no tab switch needed to see results, no dialog to
> configure output location. The draggable divider is a **learned idiom** (Cooper) from
> every split-pane editor. The error banner follows Krug's principle: "Make it
> self-explanatory" — it shows *what* failed, *where* (line number), and *what to do*
> (Go to Error / Dismiss) without requiring the user to read documentation.

#### Entity Structure

```rust
pub struct QueryEditor {
    editor: Entity<Editor>,                  // SQL editor (full Editor with language server)
    connection_id: Option<ConnectionId>,
    schema: Option<String>,
    result_grid: Option<Entity<ResultGrid>>,
    split_state: Entity<SplitState>,         // Manages vertical split ratio
    execution_state: ExecutionState,
    tab_name: Option<String>,                // From `-- @name Foo` comment
    focus_handle: FocusHandle,
    _subscriptions: Vec<Subscription>,
}

pub struct SplitState {
    ratio: f32,                              // 0.0-1.0, default 0.5
    visible_ratio: f32,                      // During drag
    cached_height: Pixels,
    is_dragging: bool,
}

pub enum ExecutionState {
    Idle,
    Executing { task: Task<()>, started_at: Instant },
    Completed { duration: Duration, row_count: usize },
    Failed { error: DatabaseError, duration: Duration },
}
```

#### Render Layout (Vertical Split)

```rust
impl Render for QueryEditor {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let ratio = self.split_state.read(cx).visible_ratio;

        v_flex()
            .size_full()
            .child(
                // SQL Editor (top section)
                div()
                    .flex_grow()
                    .flex_basis(relative(ratio))
                    .min_h(px(60.))
                    .child(self.editor.clone())
            )
            .when_some(self.result_grid.as_ref(), |this, grid| {
                this
                    .child(self.render_split_handle(window, cx))  // Draggable divider
                    .child(
                        // Result Grid (bottom section)
                        div()
                            .flex_grow()
                            .flex_basis(relative(1.0 - ratio))
                            .min_h(px(60.))
                            .child(grid.clone())
                    )
            })
            .when(matches!(self.execution_state, ExecutionState::Failed { .. }), |this| {
                this.child(self.render_error_banner(window, cx))
            })
    }
}
```

#### Split Handle (Draggable Divider)

Follows the exact pattern from `crates/editor/src/split_editor_view.rs`:

```rust
fn render_split_handle(&self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    let split_state = self.split_state.clone();
    div()
        .h(px(6.))
        .w_full()
        .cursor_row_resize()
        .bg(cx.theme().colors().border)
        .hover(|style| style.bg(cx.theme().colors().border_focused))
        .on_drag(DraggedSplitHandle, |_, _, _, cx| cx.new(|_| gpui::Empty))
        .on_drag_move::<DraggedSplitHandle>(
            cx.listener(move |this, event: &DragMoveEvent<DraggedSplitHandle>, window, cx| {
                this.split_state.update(cx, |state, cx| {
                    state.on_drag_move(event, window, cx);
                });
            })
        )
        .on_click(cx.listener(move |this, event: &ClickEvent, _, cx| {
            if event.click_count() >= 2 {
                // Double-click resets to 50/50
                this.split_state.update(cx, |state, cx| {
                    state.ratio = 0.5;
                    state.visible_ratio = 0.5;
                    cx.notify();
                });
            }
        }))
}
```

#### Item Trait Implementation

```rust
impl Item for QueryEditor {
    type Event = QueryEditorEvent;

    fn tab_content(&self, params: TabContentParams, window: &Window, cx: &App) -> AnyElement {
        let connection_color = self.connection_color(cx);
        h_flex()
            .gap_1()
            .when_some(connection_color, |this, color| {
                // Colored dot for connection identity
                this.child(div().w(px(6.)).h(px(6.)).rounded_full().bg(color))
            })
            .child(Label::new(self.tab_name(cx)).color(params.text_color()))
            .into_any_element()
    }

    fn tab_icon(&self, _: &Window, _: &App) -> Option<Icon> {
        Some(Icon::new(IconName::Database))
    }

    fn breadcrumb_location(&self, _: &App) -> ToolbarItemLocation {
        ToolbarItemLocation::PrimaryLeft
    }

    fn breadcrumbs(&self, cx: &App) -> Option<Vec<BreadcrumbText>> {
        // Connection > Schema path shown in breadcrumbs
        let mut crumbs = vec![];
        if let Some(conn) = &self.connection_id {
            crumbs.push(BreadcrumbText {
                text: conn.display_name().into(),
                highlights: None, font: None,
            });
        }
        if let Some(schema) = &self.schema {
            crumbs.push(BreadcrumbText {
                text: schema.clone().into(),
                highlights: None, font: None,
            });
        }
        Some(crumbs)
    }

    fn show_toolbar(&self) -> bool { true }
    fn is_dirty(&self, cx: &App) -> bool { self.editor.read(cx).is_dirty(cx) }
}
```

#### Tab Naming Convention

Priority order:
1. `-- @name My Query` comment in first 5 lines → "My Query"
2. Saved file name → "users_report.sql"
3. Auto-generated → "Query 1", "Query 2", etc.

#### Error Banner (Between Editor and Grid)

```
┌──────────────────────────────────────────────────────────┐
│ ✕  ERROR at line 3: column "emial" does not exist        │
│    Hint: Perhaps you meant "email"?                      │
│    [Go to Error]  [Dismiss]                              │
└──────────────────────────────────────────────────────────┘
```

- Red left border (4px)
- `Icon::new(IconName::XCircle).color(Color::Error)`
- Error text selectable (wrapped in an `Entity<Editor>` read-only)
- "Go to Error" jumps cursor to the SQL position if available
- Dismissible with Escape or X button
- Replaces previous error on re-execution

### 5. Result Grid

> **Design rationale**: The grid is the heart of the database IDE and draws heavily from
> multiple UX frameworks:
>
> - **NNg's four table tasks** (find, compare, view/edit, act) — every feature maps to
>   at least one task (see mapping table in Design Foundations).
> - **Tufte's data-ink ratio** — cells contain data only, no decorative borders between
>   cells. Striped rows (`Table::striped()`) provide scanning support without adding
>   non-data ink. Column resize handles are invisible until hover (Tufte: erase non-data
>   ink that is redundant).
> - **Tufte's data density** — `uniform_list` with 28px row height displays ~25 rows
>   in a typical viewport. Virtual scrolling removes pagination as the primary navigation
>   method (pagination exists but as a secondary mechanism for server-side limits).
> - **Gestalt figure/ground** — selected cells use sufficient contrast (`element_selected`)
>   against both light and dark themes. Pending changes use low-opacity color overlays
>   (0.12 alpha) so the data remains readable as the figure.
> - **Jakob's Law** — click-to-sort, double-click-to-edit, right-click-for-menu,
>   drag-to-resize are idioms from Excel/Google Sheets that users already know.
> - **Fitts's Law** — context menus appear at cursor position; the pagination bar is
>   a persistent strip (infinite width target at the bottom edge of the grid).
> - **Pencil & Paper (enterprise data table patterns)** — right-aligned numbers with
>   tabular figures, left-aligned strings, centered booleans.

#### Entity Structure

```rust
pub struct ResultGrid {
    columns: Vec<ColumnDef>,
    rows: Arc<Vec<Row>>,
    total_row_count: Option<usize>,          // Server-side total if known
    page: usize,
    page_size: usize,
    sort_state: Vec<SortColumn>,             // Multi-column sort stack
    filters: Vec<ColumnFilter>,
    selection: GridSelection,
    pending_edits: IndexMap<CellAddress, PendingEdit>,
    view_mode: ViewMode,                     // Grid | Record
    table_interaction_state: Entity<TableInteractionState>,
    column_widths: Entity<TableColumnWidths>,
    focus_handle: FocusHandle,
    _subscriptions: Vec<Subscription>,
}

pub enum GridSelection {
    None,
    Cell { row: usize, col: usize },
    Range { start: (usize, usize), end: (usize, usize) },
    Rows(Vec<usize>),
    Columns(Vec<usize>),
    All,
}

pub enum ViewMode { Grid, Record }

pub struct PendingEdit {
    original: CellValue,
    current: CellValue,
    kind: EditKind,                          // Insert | Update | Delete
}
```

#### Table Rendering

Built on `Table` from `crates/ui/src/components/data_table.rs`:

```rust
fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
    let connection_color = self.connection_color(cx);

    v_flex()
        .size_full()
        .child(
            Table::new(self.columns.len())
                .header(self.render_column_headers(window, cx))
                .uniform_list(
                    "result-grid",
                    self.rows.len(),
                    cx.listener(Self::render_rows),
                )
                .interactable(&self.table_interaction_state)
                .resizable_columns(
                    TableResizeBehavior::Resizable,
                    &self.column_widths,
                    cx,
                )
                .striped()
                .map_row(cx.listener(Self::style_row))
                .when_some(connection_color, |table, color| {
                    // Colored top border on header
                    table  // Applied via map_row on row 0
                })
        )
        .child(self.render_pagination_bar(window, cx))
}
```

#### Column Headers

Each column header is interactive:

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ id ▲1  🔍    │ name    🔍   │ email ▼2 🔍  │ created_at   │
├──────────────┼──────────────┼──────────────┼──────────────┤
```

- **Click** on column name → toggle sort (None → ASC → DESC → None)
- **Shift+Click** → add to multi-column sort (shows priority number)
- **Sort indicator**: `▲` / `▼` arrow + sort priority number for multi-sort
- **Filter icon** (`🔍`): opens `PopoverMenu` with filter options:

```
┌─────────────────────────────┐
│ Filter: email               │
│ ┌─────────────────────────┐ │
│ │ Contains...             │ │  ← Entity<Editor> single-line
│ └─────────────────────────┘ │
│ ○ Contains    ○ Equals      │
│ ○ Starts with ○ Regex       │
│ ○ Is NULL     ○ Is NOT NULL │
│ [Clear]           [Apply]   │
└─────────────────────────────┘
```

#### Cell Rendering by Data Type

| Type | Rendering | Alignment | Font |
|---|---|---|---|
| `NULL` | `"NULL"` in italic, `Color::Muted`, `opacity(0.5)` | — | UI font |
| `Integer` | Formatted with grouping separators (settings) | Right | Monospace |
| `Float` | Formatted with decimal + grouping (settings) | Right | Monospace |
| `String` | Text, truncated with `…` at cell boundary | Left | UI font |
| `Boolean` | `"true"` in green / `"false"` in red + checkbox icon | Center | UI font |
| `Date` | ISO 8601 format (configurable) | Left | Monospace |
| `Timestamp` | ISO 8601 with timezone (configurable) | Left | Monospace |
| `JSON` | First-line preview + `Icon::new(IconName::Braces)` | Left | Monospace |
| `BLOB` | Formatted size (e.g. "4.2 KB") + `Icon::new(IconName::Binary)` | Left | UI font |
| `UUID` | Full UUID string, monospace | Left | Monospace |
| `Array` | `"{1,2,3}"` formatted, truncated | Left | Monospace |

#### Cell Selection Visual States

```rust
fn style_row(&self, (row_index, row_div): (usize, Stateful<Div>), window: &mut Window, cx: &mut App) -> AnyElement {
    let is_selected = self.selection.contains_row(row_index);
    let has_pending_edit = self.pending_edits.values().any(|e| e.row == row_index);

    row_div
        .when(is_selected, |div| {
            div.bg(cx.theme().colors().element_selected)
        })
        .when(has_pending_edit, |div| {
            let edit = self.pending_edit_for_row(row_index);
            match edit.kind {
                EditKind::Insert => div.bg(hsla(0.33, 0.3, 0.5, 0.12)),  // Green tint
                EditKind::Update => div.bg(hsla(0.14, 0.3, 0.5, 0.12)), // Yellow tint
                EditKind::Delete => div.bg(hsla(0.0, 0.3, 0.5, 0.12))   // Red tint
                    .child(div().absolute().inset_0().bg(hsla(0.0, 0.0, 0.5, 0.3))), // Strikethrough overlay
            }
        })
        .into_any_element()
}
```

#### Inline Cell Editing

- **Trigger**: Double-click or Enter/F2 on selected cell
- **Mechanism**: Replace cell element with an `Entity<Editor>` single-line
- **Confirm**: Enter (commits edit to pending_edits), Tab (commits + moves right)
- **Cancel**: Escape (reverts to original)
- **Boolean cells**: Space toggles directly (no editor needed)
- **NULL cells**: Ctrl+Delete or Delete sets cell to NULL
- **Visual feedback**: Edited cell gets a small colored triangle in top-left corner

```rust
fn render_cell(&self, row: usize, col: usize, window: &mut Window, cx: &mut Context<Self>) -> AnyElement {
    if self.editing_cell == Some((row, col)) {
        // Render inline editor
        div()
            .size_full()
            .child(self.inline_editor.clone())
            .into_any_element()
    } else {
        let value = &self.rows[row][col];
        let pending = self.pending_edits.get(&CellAddress { row, col });

        div()
            .size_full()
            .when_some(pending, |div, edit| {
                // Edited indicator triangle
                div.child(
                    div().absolute().top_0().left_0()
                        .w(px(6.)).h(px(6.))
                        .bg(edit.kind.indicator_color())
                        .clip_path("polygon(0 0, 100% 0, 0 100%)")
                )
            })
            .child(self.render_typed_value(value, &self.columns[col], cx))
            .into_any_element()
    }
}
```

#### Pagination Bar

```
┌──────────────────────────────────────────────────────────────────┐
│ 243 rows (0.12s) │ ◀ 1 2 [3] 4 5 ▶ │ Page size: [500 ▾] │ ⟳  │
└──────────────────────────────────────────────────────────────────┘
```

- **Left section**: Row count + query execution time
- **Center**: Page navigation (when total > page_size)
- **Right**: Page size dropdown (`PopoverMenu` with [100, 500, 1000, 5000]) + Refresh button
- **Behavior**: Page changes re-execute query with `LIMIT/OFFSET`

#### Context Menus

**Cell context menu** (right-click on data cell):
- Copy Cell Value (Ctrl+C)
- Copy Row as: INSERT | CSV | JSON | Tab-separated
- Edit Cell (F2)
- Set to NULL (Delete)
- Filter by This Value
- ─── (separator)
- Add Row
- Clone Row
- Delete Row(s)

**Column header context menu:**
- Sort Ascending / Descending / Clear Sort
- Filter Column...
- Hide Column
- Show All Columns
- ─── (separator)
- Resize to Fit Content
- Copy Column Name

**Row header context menu** (when clicking row numbers, if shown):
- Copy Selected Rows
- Delete Selected Rows
- Clone Row

### 6. Toolbar and Status Bar

> **Design rationale**: The toolbar applies **Fitts's Law** — the most frequent action
> (Execute) is placed leftmost in the action group (shortest distance from the editor).
> The toolbar is hidden when no QueryEditor is active, following **progressive disclosure**
> (NNg) — don't show controls that have no effect. The status bar provides **system
> visibility** (Nielsen's heuristic #1) — connection state, row count, and execution
> time are always visible. The connection dropdown in breadcrumbs applies **Hick's Law**
> — only showing the current connection + schema, expanding to the full list on click.

#### Toolbar Components

The toolbar is visible whenever a `QueryEditor` or detached `ResultGrid` is the active pane item.

**PrimaryLeft — QueryBreadcrumbs:**

```rust
pub struct QueryBreadcrumbs {
    active_query_editor: Option<WeakEntity<QueryEditor>>,
}

impl ToolbarItemView for QueryBreadcrumbs {
    fn set_active_pane_item(
        &mut self,
        active_pane_item: Option<&dyn ItemHandle>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> ToolbarItemLocation {
        self.active_query_editor = active_pane_item
            .and_then(|item| item.act_as::<QueryEditor>(cx))
            .map(|entity| entity.downgrade());

        if self.active_query_editor.is_some() {
            ToolbarItemLocation::PrimaryLeft
        } else {
            ToolbarItemLocation::Hidden
        }
    }
}
```

Renders as: `[● Connection Name ▾]` > `[Schema Name ▾]`

Each segment is a `PopoverMenu` trigger that opens a `ContextMenu` listing available
connections / schemas respectively. Selecting one switches the active QueryEditor's
connection or schema.

**PrimaryRight — QueryActions:**

| Button | Icon | Action | Shortcut | Visibility |
|---|---|---|---|---|
| Execute | `Play` | `ExecuteQuery` | `Ctrl+Enter` | Always |
| Cancel | `Square` | `CancelQuery` | `Escape` | While executing |
| Explain | `ChartLine` | `ExplainQuery` | `Ctrl+Shift+E` | Always |
| Format | `Paintbrush` | `FormatQuery` | `Ctrl+Shift+F` | Always |
| Record View | `Table2` | `ToggleRecordView` | `Ctrl+Shift+R` | When results exist |
| Detach Grid | `PanelBottom` | `DetachResultGrid` | — | When results exist |

Execute button shows a `Spinner` icon while query is running.

#### Status Bar — DatabaseStatusItem

```rust
pub struct DatabaseStatusItem {
    active_connection: Option<ConnectionInfo>,
    execution_state: Option<ExecutionStateSnapshot>,
}

impl StatusItemView for DatabaseStatusItem {
    fn set_active_pane_item(&mut self, item: Option<&dyn ItemHandle>, window: &mut Window, cx: &mut Context<Self>) {
        self.active_connection = item
            .and_then(|i| i.act_as::<QueryEditor>(cx))
            .and_then(|qe| qe.read(cx).connection_info(cx));
    }
}
```

Renders as:

```
[● Production] [🔒 Read-only] [243 rows / 0.12s] [⟳ Executing...]
```

- Colored dot matches connection color
- Lock icon for read-only connections
- Row count + execution time from last query
- Spinner + "Executing..." during query execution
- Click on connection name opens connection switcher
- Hidden when no QueryEditor is active

### 7. Tab Management

> **Design rationale**: Tab behavior follows **Jakob's Law** — preview tabs (italic,
> replaced by next preview) and pinned tabs are conventions established by VS Code
> and Zed itself. Users already understand this pattern. The connection color indicator
> in tabs applies **Gestalt similarity** — the same color dot seen in the explorer
> and status bar appears in the tab, creating a unified visual thread per connection.
> Following Krug's "don't make me think," the tab title shows the most meaningful
> identifier (custom `@name` > filename > auto-number) without requiring user
> configuration.

#### Tab Behavior for QueryEditor

| Action | Tab Behavior |
|---|---|
| Double-click table in Explorer | Opens as **preview tab** (italic title), replaced by next preview |
| Start editing SQL | Preview tab becomes **permanent** (regular title) |
| "New Query" from Explorer | Opens as permanent tab |
| Execute query | Tab stays (results appear in split below) |
| Pin tab (`TogglePinTab`) | Tab cannot be replaced or auto-closed |

#### Tab Indicators

```
┌─────────────────────────────────────────────────────┐
│ [● Query 1] [● users ⟳] [📌 orders] [+ migrations] │
└─────────────────────────────────────────────────────┘
```

- `●` colored dot = connection color
- `⟳` = query currently executing
- `📌` = pinned tab
- Italic title = preview tab
- `•` modified indicator = has unsaved edits (pending changes to commit)

#### Tab Extra Context Menu

```rust
impl Item for QueryEditor {
    fn tab_extra_context_menu_actions(&self, _: &mut Window, _: &mut Context<Self>) -> Vec<(SharedString, Box<dyn Action>)> {
        vec![
            ("Copy as SQL".into(), Box::new(CopyAsSql)),
            ("Export Results...".into(), Box::new(ExportResults)),
            ("Change Connection...".into(), Box::new(ChangeConnection)),
        ]
    }
}
```

### 8. Side Panels

> **Design rationale**: Side panels follow **progressive disclosure** (Cooper / NNg) —
> the ValueEditor is hidden by default and only appears when the user needs to inspect
> a large value. The AggregateView uses a floating popover rather than a permanent panel,
> following **Tufte's data-ink ratio** — show aggregates only when relevant (numeric
> selection exists). The RecordView is a mode toggle within the grid, not a separate
> panel, following **Miller's Law** — don't add another item to the user's mental model
> of panel locations. JetBrains' DataGrip uses the same pattern: record view is a toggle
> within the result tab, not a separate window.

#### Value Editor Panel (Right Dock)

**Purpose**: Full-content editor for large cell values (JSON, XML, BLOB, long text).

```rust
pub struct ValueEditor {
    content_editor: Option<Entity<Editor>>,
    content_type: ContentType,
    source_cell: Option<CellAddress>,
    is_read_only: bool,
    focus_handle: FocusHandle,
    width: Option<Pixels>,
}

pub enum ContentType {
    Json,
    Xml,
    PlainText,
    Hex(Vec<u8>),
    Image(Arc<[u8]>),
}
```

**Trigger**: Double-click on JSON/BLOB/XML cell, or `Ctrl+Shift+V` on selected cell.

**Rendering by type**:
- **JSON**: `Entity<Editor>` with `LanguageId::Json` for syntax highlighting + folding
- **XML**: `Entity<Editor>` with `LanguageId::Xml`
- **Plain text**: `Entity<Editor>` with word wrap enabled
- **Hex (binary)**: Custom hex view — 16 bytes per line, offset + hex + ASCII columns
- **Image (detected BLOB)**: `gpui::img(data)` centered in panel

**Panel implementation**:
```rust
impl Panel for ValueEditor {
    fn persistent_name() -> &'static str { "ValueEditor" }
    fn panel_key() -> &'static str { "ValueEditor" }
    fn position(&self, ..) -> DockPosition { DockPosition::Right }
    fn icon(&self, ..) -> Option<IconName> { Some(IconName::TextSelect) }
    fn activation_priority(&self) -> u32 { 16 }
    fn starts_open(&self, ..) -> bool { false }  // Hidden by default
    fn toggle_action(&self) -> Box<dyn Action> { Box::new(ToggleValueEditor) }
}
```

#### Record View (Alternate Mode in ResultGrid)

Not a separate panel — a mode toggle within `ResultGrid` that switches from grid to
a vertical single-row layout:

```
┌──────────────────────────────────────────────────┐
│  ◀ Row 42 of 243 ▶                              │
├──────────────────────────────────────────────────┤
│  id            │ 42                              │
│  name          │ John Doe                        │
│  email         │ john@example.com                │
│  bio           │ Software engineer living in...  │
│  avatar_url    │ https://cdn.example.com/...     │
│  created_at    │ 2024-01-15T10:30:00Z            │
│  updated_at    │ 2024-03-01T14:22:00Z            │
│  is_active     │ ✓ true                          │
│  metadata      │ {"role":"admin","permi...  [▸]  │
│  profile_image │ 12.4 KB                    [▸]  │
└──────────────────────────────────────────────────┘
```

- Rendered as a 2-column `Table` (field name | value)
- Navigation: `◀ ▶` or Up/Down arrows move between rows
- `[▸]` on JSON/BLOB cells opens ValueEditor panel
- Editable: clicking on a value cell enters edit mode
- Toggle: `Ctrl+Shift+R` or toolbar button

#### Aggregate View (Floating Popover)

Appears automatically when selecting multiple numeric cells (range selection):

```
┌─────────────────────────┐
│ Count: 15               │
│ Sum:   4,523.50         │
│ Avg:   301.57           │
│ Min:   12.00            │
│ Max:   892.00           │
└─────────────────────────┘
```

- Uses `Popover` component anchored to bottom-right of selection area
- Updates dynamically as selection changes
- Only appears for columns with numeric types
- Dismisses when selection is cleared
- Can be pinned (click keeps it visible until explicit close)

### 9. Keyboard Navigation and Shortcuts

> **Design rationale**: Cooper's *About Face* argues that **perpetual intermediates**
> are the largest user segment for complex applications. The keyboard system is designed
> in three layers for this spectrum:
>
> 1. **Discoverable** (beginners): All actions available via mouse and context menus,
>    shortcuts shown in tooltips and menu items.
> 2. **Efficient** (intermediates): Standard shortcuts (`Ctrl+Enter`, arrow keys, `Tab`)
>    match conventions from DataGrip, Excel, and VS Code — **Jakob's Law** ensures
>    zero learning curve for common actions.
> 3. **Expert** (power users): Vim mode bindings (`h/j/k/l`, `gg`, `G`, `dd`, `/`)
>    provide maximum throughput for keyboard-centric users, following Cooper's principle
>    of not penalizing experts for the sake of beginners.
>
> The Vim integration uses GPUI's `KeyContext` system to scope bindings — vim keys
> only activate in "ResultGrid" context, never conflicting with the SQL editor's own
> vim mode. This follows **Hick's Law**: the available actions are scoped to the
> current context, reducing the decision space.

#### Grid Navigation

| Key | Action | Context |
|---|---|---|
| `Arrow keys` | Move selection by one cell | Grid focused |
| `Shift+Arrow` | Extend selection range | Grid focused |
| `Ctrl+Arrow` | Jump to edge of data region | Grid focused |
| `Ctrl+Shift+Arrow` | Extend selection to edge | Grid focused |
| `Home` / `End` | First / last column in row | Grid focused |
| `Ctrl+Home` / `Ctrl+End` | First / last cell in grid | Grid focused |
| `Tab` / `Shift+Tab` | Next / previous cell | Grid or editing |
| `Enter` / `F2` | Start editing cell | Cell selected |
| `Escape` | Cancel edit / clear selection | Editing or selected |
| `Space` | Toggle boolean cell | Boolean cell selected |
| `Delete` | Set cell to NULL | Cell selected |
| `Ctrl+A` | Select all cells | Grid focused |
| `Ctrl+C` | Copy selection | Any selection |
| `Ctrl+V` | Paste into selection | Cell selected |
| `Ctrl+Z` | Undo pending edit | Has pending edits |
| `Ctrl+Shift+Z` | Redo pending edit | Has undone edits |
| `Ctrl+D` | Duplicate selected row(s) | Row(s) selected |
| `Ctrl+Minus` | Delete selected row(s) | Row(s) selected |
| `Ctrl+Plus` | Insert new row | Grid focused |
| `PageDown` / `PageUp` | Next / previous page | Grid focused |

#### Query Execution

| Key | Action | Context |
|---|---|---|
| `Ctrl+Enter` | Execute query (or selected text) | QueryEditor focused |
| `Escape` | Cancel running query | Query executing |
| `Ctrl+Shift+E` | Explain query plan | QueryEditor focused |
| `Ctrl+Shift+F` | Format/beautify SQL | QueryEditor focused |
| `Ctrl+Shift+R` | Toggle Record View | Results exist |
| `Ctrl+Shift+V` | Open Value Editor for cell | Cell selected |

#### Panel Navigation

| Key | Action | Context |
|---|---|---|
| `Ctrl+Shift+D` | Toggle Database Explorer | Global |
| `Ctrl+Shift+B` | Toggle Value Editor | Global |
| `F5` | Refresh Explorer tree | Explorer focused |
| `Ctrl+F` | Focus filter in Explorer | Explorer focused |

#### Vim Mode Integration

When `vim_mode` setting is enabled, the ResultGrid registers an additional `KeyContext`
to support vim-style navigation:

```rust
// In ResultGrid's key_context contribution
fn contribute_key_context(&self, context: &mut KeyContext, cx: &App) {
    context.add("ResultGrid");
    if self.is_editing() {
        context.set("grid_mode", "editing");
    } else {
        context.set("grid_mode", "normal");
    }
}
```

**Vim normal mode bindings** (context: `ResultGrid && grid_mode == normal`):

| Key | Action | Equivalent |
|---|---|---|
| `h` / `j` / `k` / `l` | Move cell | Arrow keys |
| `g g` | Go to first row | Ctrl+Home |
| `G` | Go to last row | Ctrl+End |
| `0` / `$` | First / last column | Home / End |
| `i` | Edit cell | Enter/F2 |
| `/` | Open column filter | Ctrl+F on column |
| `y y` | Copy row | Ctrl+C on row |
| `d d` | Delete row | Ctrl+Minus |
| `o` | Insert row below | Ctrl+Plus |
| `v` | Start visual selection | Shift+Arrow |
| `V` | Select entire row | Click row header |
| `:w` | Commit pending changes | Submit button |
| `:q` | Close tab | Standard close |

### 10. Visual Design Language

> **Design rationale**: The visual language follows three core principles:
>
> 1. **Tufte's data-ink ratio**: Every visual element must earn its place. No decorative
>    borders, gradients, or shadows on data cells. The striped pattern is the minimum
>    ink needed for horizontal scanning (NNg recommendation for dense tables).
>
> 2. **Gestalt similarity for states**: Pending changes use a consistent color language
>    (green = insert, yellow = update, red = delete) across all contexts — cell
>    indicators, row backgrounds, commit bar summary, and DML preview. This follows
>    Tufte's "Show data variation, not design variation" principle.
>
> 3. **Accessibility (WCAG 2.1 AA)**: Color is never the sole indicator — always
>    paired with text or icon (Krug: "Don't rely on a single channel"). The muted
>    NULL styling uses italic *and* reduced opacity. Error banners have icon *and*
>    red border *and* text. This ensures the interface works for colorblind users
>    and follows NNg's inclusive design guidelines.

#### Color Palette for Data States

| State | Background | Border | Use |
|---|---|---|---|
| Selected cell | `theme.colors().element_selected` | — | Current selection |
| Selected range | `theme.colors().element_selected` at 60% | — | Multi-cell selection |
| Pending INSERT | `hsla(0.33, 0.3, 0.5, 0.12)` | Green left 2px | New row |
| Pending UPDATE | `hsla(0.14, 0.3, 0.5, 0.12)` | Yellow left 2px | Modified cell/row |
| Pending DELETE | `hsla(0.0, 0.3, 0.5, 0.12)` + strikethrough | Red left 2px | Deleted row |
| Hover row | `theme.colors().element_hover` | — | Mouse hover |
| Error | `hsla(0.0, 0.4, 0.5, 0.08)` | Red left 4px | Error banner |
| Success | `hsla(0.33, 0.4, 0.5, 0.08)` | Green left 4px | Success banner |

#### Typography

| Element | Font | Size | Weight | Notes |
|---|---|---|---|---|
| Column headers | UI font | `ui_sm` | `FontWeight::SEMIBOLD` | Uppercase optional |
| Cell data | Grid font (setting) or buffer font | `ui_sm` | `FontWeight::NORMAL` | Monospace for numbers |
| NULL values | UI font | `ui_sm` | `FontWeight::NORMAL` | Italic, muted |
| Boolean values | UI font | `ui_sm` | `FontWeight::MEDIUM` | Colored |
| Tree labels | UI font | `ui_sm` | `FontWeight::NORMAL` | — |
| Connection names | UI font | `ui_sm` | `FontWeight::SEMIBOLD` | — |
| Status bar | UI font | `ui_xs` | `FontWeight::NORMAL` | — |
| Pagination | UI font | `ui_xs` | `FontWeight::NORMAL` | — |

#### Spacing and Sizing

| Element | Value | Notes |
|---|---|---|
| Grid cell padding | `px(6.)` horizontal, `px(4.)` vertical | Compact by default |
| Grid row height | `px(28.)` | Uniform for virtual scrolling |
| Column min-width | `px(60.)` | Prevents columns from disappearing |
| Column default-width | Content-fitted or equal distribution | |
| Explorer indent step | `px(20.)` per level | Matches ProjectPanel |
| Split handle height | `px(6.)` | Draggable area |
| Panel min-width | `px(150.)` | For Explorer/ValueEditor |
| Tab bar height | `Tab::container_height(cx)` | Standard Zed tab height |
| Connection color dot | `px(6.)` diameter | In tabs and status bar |
| Edit indicator triangle | `px(6.)` side | Top-left corner of edited cell |

#### Empty States

**No connections:**
```
┌─────────────────────────────┐
│                             │
│    🔌 No Connections        │
│                             │
│    Add a database           │
│    connection to get        │
│    started.                 │
│                             │
│    [Add Connection]         │
│                             │
└─────────────────────────────┘
```

**No results:**
```
┌─────────────────────────────┐
│                             │
│    Execute a query to       │
│    see results here.        │
│                             │
│    Ctrl+Enter to run        │
│                             │
└─────────────────────────────┘
```

Uses `Table::empty_table_callback()` for the grid empty state.

### 11. Responsive Behavior

> **Design rationale**: NNg's article *How to Fit Big Tables on Small Screens*
> recommends locked headers and progressive content reduction. Our approach follows
> three tiers of degradation: at full width, all features are visible; at medium width,
> labels truncate; at minimum width, icons replace labels. This matches JetBrains'
> DataGrip "Compact Mode" philosophy. The split ratio constraints (15%/85% min/max)
> ensure neither the editor nor the grid becomes unusably small — following Cooper's
> principle that controls should never disappear entirely.

#### Grid Column Layout

- Columns distribute available width proportionally to content
- Minimum column width: `px(60.)` — below this, horizontal scroll activates
- Horizontal scrolling managed by `TableInteractionState::scroll_handle`
- Column resize handles are `px(4.)` wide, visible on hover
- Double-click resize handle → auto-fit column to content width

#### Explorer Panel Width

| Width Range | Behavior |
|---|---|
| > 200px | Full labels + icons + hover actions |
| 150–200px | Labels truncated with ellipsis |
| < 150px | Icons only (labels hidden), indent reduced to `px(12.)` |
| < 100px | Minimum width, panel cannot shrink further |

#### Split Ratio Constraints

- Minimum: 15% for either section (editor or grid)
- Maximum: 85% for either section
- Default: 50/50
- Double-click handle: reset to 50/50
- Ratio persisted in workspace serialization

#### Window Size Adaptation

- Panels follow Zed's standard dock resize behavior (6px drag handle)
- At very narrow window widths, panels can be collapsed to icon-only in dock bar
- Grid pagination bar wraps: info on first line, navigation on second when narrow
- Modal dialogs have `max_w(px(560.))` and `max_h(vh(0.85))` with scroll

### 12. Commit/Submit Flow for Data Modifications

> **Design rationale**: The explicit commit flow is a direct application of Cooper's
> **"no data loss" principle** — users should never lose work due to an accidental
> action. Staging changes locally before committing them is a **recognition over recall**
> pattern (NNg) — the user can see exactly what will change before it happens. The DML
> preview dialog showing exact SQL statements follows **Tufte's "show the data"** —
> don't hide the underlying truth behind abstractions. Auto-commit mode is opt-in
> and signaled with a visible `[⚡ Auto-commit]` indicator, following Krug's principle
> of making system state visible.

Since data modifications (INSERT, UPDATE, DELETE) are staged as pending edits,
the user must explicitly commit them:

#### Pending Changes Indicator

When `pending_edits` is non-empty, show a commit bar above the pagination:

```
┌──────────────────────────────────────────────────────────────────┐
│ 3 pending changes (1 insert, 1 update, 1 delete)  [Revert All] [Commit] │
└──────────────────────────────────────────────────────────────────┘
```

- "Commit" sends all pending changes as SQL (within a transaction if auto-commit is off)
- "Revert All" clears all pending edits
- Individual row revert via context menu
- If the user tries to close a tab with pending changes, show a confirmation modal
  (implementing `on_before_dismiss` pattern)

#### Auto-commit Mode

When `DatabaseSettings.auto_commit` is true:
- Each edit immediately executes the corresponding SQL
- No commit bar shown
- Undo sends a reverse SQL statement
- Setting shown as `[⚡ Auto-commit]` indicator in status bar

### 13. Accessibility Considerations

> **Design rationale**: Accessibility has evolved from compliance to competitive advantage
> (devPulse, 2025). Following WCAG 2.1 AA guidelines and NNg's inclusive design
> research, every interaction is available through at least two modalities (mouse +
> keyboard, color + text/icon). Cooper's *About Face* emphasizes that accessible
> constraints often produce better designs for everyone — the keyboard navigation
> system benefits power users as much as users who cannot use a mouse.

- All interactive elements have ARIA-equivalent focus management via GPUI's `FocusHandle`
- Grid cells are navigable via keyboard (never mouse-only interactions) — following
  Cooper's axiom: "every mouse action must have a keyboard equivalent"
- Color is never the sole indicator — always paired with icon or text
  (e.g., NULL is italic + muted, not just grey; errors have icon + border + text).
  This follows **Gestalt figure/ground** — even without color, the visual encoding
  (italic, border, icon shape) carries the meaning
- Screen reader support: column headers announce type, sort state; cells announce
  value + column name. Focus indicators follow WCAG 2.4.7 (visible focus)
- High contrast theme support: all custom colors use theme tokens where possible,
  fall back to hardcoded hsla only for data-state backgrounds (which use low opacity
  overlays on top of theme background)
- **Reduced motion**: Animations (loading spinners, popover transitions) respect
  the OS-level `prefers-reduced-motion` setting
- **Font scaling**: Grid font size is independently configurable
  (`DatabaseSettings.grid_font_size`), supporting users who need larger text
  without scaling the entire UI

### GPUI Components Reuse Summary

| Zed Component | File | Usage in Database IDE |
|---|---|---|
| `Table` + `uniform_list` | `crates/ui/src/components/data_table.rs` | ResultGrid core rendering |
| `TableInteractionState` | same file | Scroll + focus for grid |
| `TableColumnWidths` | same file | Resizable columns |
| `ListItem` | `crates/ui/src/components/list/list_item.rs` | Explorer tree nodes |
| `ListHeader` | `crates/ui/src/components/list/list_header.rs` | Explorer category headers |
| `ContextMenu::build()` | `crates/ui/src/components/context_menu.rs` | All right-click menus |
| `PopoverMenu` | `crates/ui/src/components/popover_menu.rs` | Dropdowns, filter popups |
| `Popover` | `crates/ui/src/components/popover.rs` | AggregateView float |
| `Disclosure` | `crates/ui/src/components/disclosure.rs` | SSH/SSL sections in dialog |
| `Modal` + `ModalHeader` + `ModalFooter` | `crates/ui/src/components/modal.rs` | ConnectionDialog shell |
| `Tab` + `TabBar` | `crates/ui/src/components/tab.rs` | Driver selector in dialog |
| `Icon` + `IconName` | `crates/ui/src/components/icon.rs` | All iconography |
| `Label` | `crates/ui/src/components/label.rs` | All text rendering |
| `Spinner` | `crates/ui/src/components/spinner.rs` | Loading states |
| `Panel` trait | `crates/workspace/src/dock.rs` | DatabaseExplorer, ValueEditor |
| `Item` trait | `crates/workspace/src/item.rs` | QueryEditor |
| `ToolbarItemView` | `crates/workspace/src/toolbar.rs` | QueryBreadcrumbs, QueryActions |
| `StatusItemView` | `crates/workspace/src/status_bar.rs` | DatabaseStatusItem |
| `ModalView` | `crates/workspace/src/modal_layer.rs` | ConnectionDialog |
| `Entity<Editor>` | `crates/editor/` | SQL editing, inline cells, filters |
| `SplitEditorView` pattern | `crates/editor/src/split_editor_view.rs` | Drag-to-resize split handle |
| `initialize_panels()` | `crates/zed/src/zed.rs` (line ~619) | Panel registration |

---

## Security Architecture

This section defines the security model for the database IDE experience. Every design
decision is grounded in established security standards and references from recognized
authorities. Security is not an afterthought — it is integrated from the design phase
following **Microsoft's Security Development Lifecycle (SDL)** methodology.

### Reference Standards and Authorities

| Standard / Work | Authority | Year | Applied To |
|---|---|---|---|
| **OWASP Top 10:2025** | Open Web Application Security Project | 2025 | Threat prioritization across all phases |
| **OWASP SQL Injection Prevention Cheat Sheet** | OWASP | 2024 | Query execution (parameterized queries) |
| **OWASP Database Security Cheat Sheet** | OWASP | 2024 | Connection management, least privilege |
| **OWASP Secrets Management Cheat Sheet** | OWASP | 2024 | Credential storage, key rotation |
| **OWASP Transport Layer Security Cheat Sheet** | OWASP | 2024 | SSL/TLS for database connections |
| **CWE-89** (SQL Injection) | MITRE Corporation | 2006–2025 | Query construction hardening |
| **CWE-522** (Insufficiently Protected Credentials) | MITRE Corporation | 2006–2025 | Credential lifecycle |
| **CWE-311** (Missing Encryption of Sensitive Data) | MITRE Corporation | 2006–2025 | Data in transit/at rest |
| **NIST SP 800-63B-4** | National Institute of Standards and Technology | 2024 | Password/credential guidelines |
| **NIST SP 800-53 Rev. 5** (AC, IA, SC families) | NIST | 2020 | Access control, identification, system comm. |
| **Microsoft SDL** | Microsoft Security Engineering | 2004–2025 | Secure development lifecycle methodology |
| **STRIDE Threat Model** | Microsoft (Loren Kohnfelder, Praerit Garg) | 1999 | Systematic threat identification |
| **Rust Security Best Practices** | Rust Security Response WG | Ongoing | Memory safety, safe concurrency |
| *The Web Application Hacker's Handbook* (2nd ed.) | **Dafydd Stuttard**, Marcus Pinto | 2011 | Input validation, session management |
| *Threat Modeling: Designing for Security* | **Adam Shostack** (Microsoft) | 2014 | Threat modeling methodology |

### STRIDE Threat Model for the Database IDE

Following Adam Shostack's methodology and Microsoft's STRIDE framework, we identify
the six categories of threats applied to our specific system components:

```
+------------------+                  +-------------------+
|  User (Zed)      |  SQL + creds     |  Database Server  |
|  +-----------+   | ────────────────>|  (PostgreSQL,     |
|  | QueryEditor|  |  TLS / SSH       |   MySQL, SQLite)  |
|  +-----------+   |<────────────────-|                   |
|  | ResultGrid |  |  Query Results   |                   |
|  +-----------+   |                  +-------------------+
|  | Explorer   |  |                         ↑
|  +-----------+   |                         |
|  | Settings   |  |     +-------------------+
|  +-----------+   |     | Platform Keychain  |
+------------------+     | (macOS/Linux/Win)  |
         ↑               +-------------------+
         |
+------------------+
|  settings.json   |  ← Connection configs (NO passwords)
|  (local file)    |
+------------------+
```

| STRIDE Category | Threat | Component | Mitigation | Reference |
|---|---|---|---|---|
| **S — Spoofing** | Attacker impersonates a database server (MITM) | Connection layer | TLS certificate verification (`SslMode::VerifyFull`), SSH host key fingerprint validation | OWASP TLS Cheat Sheet |
| **S — Spoofing** | Attacker uses stolen credentials | Credential storage | Platform keychain (never plaintext), memory zeroization after use, env var priority | NIST SP 800-63B-4, OWASP Secrets Mgmt |
| **T — Tampering** | Attacker modifies SQL in transit | Connection layer | TLS encryption for all connections, SSH tunnel for untrusted networks | CWE-319, OWASP TLS CS |
| **T — Tampering** | Attacker modifies settings.json to inject malicious connection | Settings | Settings file permissions (0600), no passwords in settings, validate config on load | OWASP Secrets Mgmt CS |
| **R — Repudiation** | User denies executing a destructive query | Query history | Immutable query history log with timestamps, DML preview before commit | OWASP Logging CS |
| **I — Info Disclosure** | Credentials leaked via logs, crash reports, or memory dumps | Credential lifecycle | Redact passwords from all logs, `Debug` impl omits secrets, zeroize on drop | CWE-532, CWE-522 |
| **I — Info Disclosure** | Query results containing PII visible in process memory | Result grid | No result caching to disk, clear results on tab close, no swap/core dump of passwords | CWE-311 |
| **D — Denial of Service** | Malicious query consumes all server resources | Query execution | Query timeout (configurable), cancel mechanism per driver, connection pool limits | OWASP Top 10 A10:2025 |
| **D — Denial of Service** | Huge result set exhausts client memory | Result grid | Server-side pagination (`LIMIT/OFFSET`), configurable `page_size`, streaming with backpressure | CWE-400 |
| **E — Elevation of Privilege** | Application account used for DDL when only SELECT needed | Connection config | Read-only mode toggle per connection, statement type validation before execution | OWASP DB Security CS, Least Privilege |
| **E — Elevation of Privilege** | AI agent executes destructive queries without consent | AI tools | `ActionAllowlist`, `ToolKind::Write` requires confirmation, max_rows_affected limit | OWASP Top 10 A01:2025 |

### 1. SQL Injection Prevention (OWASP A05:2025, CWE-89)

> **Reference**: OWASP ranks injection as **A05:2025** with over 14,000 CVEs for SQL
> injection alone. CWE-89 is classified as a "weakness that is still very frequent"
> by MITRE. The OWASP SQL Injection Prevention Cheat Sheet states: *"The #1 recommended
> defense is the use of parameterized queries (prepared statements)."*

#### Principle: Never Build SQL from User Input by String Concatenation

All user-supplied data must be bound via parameterized queries. The Rust ecosystem
enforces this through `sqlx`'s compile-time checked query macros.

**Hardened zones** (where user input touches SQL):

| Zone | User Input | SQL Context | Defense |
|---|---|---|---|
| QueryEditor execute | Full SQL text | Direct execution | Statement type validation + read-only guard |
| Server-side filter | WHERE clause text | Appended to query | Parse and validate AST, reject multi-statement |
| Sort columns | Column name | ORDER BY clause | Allow-list validation against schema cache |
| Pagination | Page number, page size | LIMIT/OFFSET values | Integer parsing only, max cap from settings |
| FK navigation | Cell value | WHERE clause | Parameterized bind (`$1` / `?`) |
| AI tools | Generated SQL | Direct execution | Human confirmation for writes, `ActionAllowlist` |
| DDL generator | Object names | DDL statements | Quote identifiers, validate against schema |

```rust
// crates/database_core/src/query_executor.rs

/// SECURE: Execute a user-written SQL query with safety checks.
/// This is the ONLY entry point for executing user-supplied SQL.
pub async fn execute_user_query(
    connection: &dyn DatabaseConnection,
    sql: &str,
    config: &ConnectionConfig,
) -> Result<QueryResult> {
    // 1. Validate statement type against read-only mode
    if config.read_only {
        let statement_type = classify_statement(sql)?;
        if statement_type.is_mutating() {
            return Err(DatabaseError::ReadOnlyViolation {
                statement_type: statement_type.to_string(),
            }.into());
        }
    }

    // 2. Reject multi-statement batches from untrusted input
    //    (user QueryEditor is trusted, but server-side filters are not)
    // 3. Execute
    connection.execute_query(sql).await
}

/// SECURE: Build a parameterized filter query — never concatenate user values.
/// Used by server-side filtering and FK navigation.
pub fn build_filtered_query(
    base_table: &TableRef,
    columns: &[ColumnInfo],
    filters: &[ColumnFilter],
    sort: &[SortColumn],
    page: usize,
    page_size: usize,
) -> (String, Vec<CellValue>) {
    let mut sql = format!(
        "SELECT * FROM {}",
        quote_identifier(&base_table.qualified_name()),
    );
    let mut params: Vec<CellValue> = Vec::new();
    let mut param_index = 1;

    // WHERE clause — all values are parameterized
    if !filters.is_empty() {
        sql.push_str(" WHERE ");
        let conditions: Vec<String> = filters.iter().map(|f| {
            let col_name = quote_identifier(&columns[f.column_index].name);
            let condition = match f.filter_type {
                FilterType::Equals => {
                    params.push(CellValue::String(f.value.clone()));
                    format!("{} = ${}", col_name, param_index)
                }
                FilterType::Contains => {
                    params.push(CellValue::String(format!("%{}%", f.value)));
                    format!("{} LIKE ${}", col_name, param_index)
                }
                FilterType::IsNull => {
                    return format!("{} IS NULL", col_name);
                    // No param needed — no user value in SQL
                }
                FilterType::IsNotNull => {
                    return format!("{} IS NOT NULL", col_name);
                }
                _ => {
                    params.push(CellValue::String(f.value.clone()));
                    format!("{} = ${}", col_name, param_index)
                }
            };
            param_index += 1;
            condition
        }).collect();
        sql.push_str(&conditions.join(" AND "));
    }

    // ORDER BY — column names validated against schema (allow-list)
    if !sort.is_empty() {
        let order_clauses: Vec<String> = sort.iter()
            .filter(|s| s.column_index < columns.len()) // Bounds check
            .map(|s| {
                let col_name = quote_identifier(&columns[s.column_index].name);
                let direction = match s.direction {
                    SortDirection::Ascending => "ASC",
                    SortDirection::Descending => "DESC",
                };
                format!("{} {}", col_name, direction)
            })
            .collect();
        if !order_clauses.is_empty() {
            sql.push_str(" ORDER BY ");
            sql.push_str(&order_clauses.join(", "));
        }
    }

    // LIMIT/OFFSET — integers only, capped by settings
    let safe_page_size = page_size.min(10_000); // Hard cap
    sql.push_str(&format!(" LIMIT {} OFFSET {}", safe_page_size, page * safe_page_size));

    (sql, params)
}

/// Quote a SQL identifier to prevent injection through object names.
/// Follows ANSI SQL double-quote escaping: `"table""name"` escapes embedded quotes.
fn quote_identifier(name: &str) -> String {
    format!("\"{}\"", name.replace('"', "\"\""))
}

/// Classify a SQL statement to determine if it's read-only or mutating.
/// Used by the read-only guard. Does NOT attempt to parse full SQL —
/// uses a conservative keyword-based heuristic.
fn classify_statement(sql: &str) -> Result<StatementType> {
    let trimmed = sql.trim_start();
    // Skip leading comments
    let normalized = strip_sql_comments(trimmed).to_uppercase();
    let first_keyword = normalized.split_whitespace().next().unwrap_or("");

    match first_keyword {
        "SELECT" | "EXPLAIN" | "SHOW" | "DESCRIBE" | "WITH" => Ok(StatementType::ReadOnly),
        "INSERT" => Ok(StatementType::Insert),
        "UPDATE" => Ok(StatementType::Update),
        "DELETE" => Ok(StatementType::Delete),
        "CREATE" | "ALTER" | "DROP" | "TRUNCATE" | "RENAME" => Ok(StatementType::Ddl),
        "GRANT" | "REVOKE" => Ok(StatementType::Dcl),
        "BEGIN" | "COMMIT" | "ROLLBACK" | "SAVEPOINT" => Ok(StatementType::Transaction),
        _ => Ok(StatementType::Unknown),
    }
}

#[derive(Debug, Clone, PartialEq)]
pub enum StatementType {
    ReadOnly,
    Insert,
    Update,
    Delete,
    Ddl,
    Dcl,
    Transaction,
    Unknown,
}

impl StatementType {
    pub fn is_mutating(&self) -> bool {
        matches!(self, Self::Insert | Self::Update | Self::Delete | Self::Ddl | Self::Dcl)
    }
}
```

### 2. Credential Security (OWASP A04:2025, CWE-522, NIST SP 800-63B-4)

> **Reference**: OWASP Secrets Management Cheat Sheet: *"Engineers should not have
> access to all secrets. Apply the Least Privilege principle."* NIST SP 800-63B-4
> eliminates mandatory password rotation and complexity rules, focusing instead on
> longer passwords and blocklist screening. CWE-522 addresses insufficiently protected
> credentials.

#### Principle: Passwords Never Exist in Plaintext Outside Secure Storage

```
Credential Lifecycle:
                                                              ┌────────────────┐
                                                              │ Platform       │
  User types password ──→ ConnectionDialog ──→ Save ──→       │ Keychain       │
  in modal                (in-memory only)    (encrypt)       │ (macOS/Linux/  │
                                                              │  Windows)      │
                                                              └───────┬────────┘
                                                                      │
  Connection needed ──→ CredentialProvider ──→ Load ──→ Memory ──→ Driver
                         Priority:                    (zeroized     (TLS
                         1. Env var                    after use)    protected)
                         2. Keychain
                         3. Prompt user
```

**Rules enforced in code:**

| Rule | Enforcement | CWE / Reference |
|---|---|---|
| No passwords in `settings.json` | `ConnectionConfig` has no `password` field — only `id`, `name`, `host`, etc. | CWE-256 (Plaintext Storage) |
| No passwords in logs | `DatabaseError::fmt()` redacts any string matching password patterns. `Debug` impls for credential types print `"[REDACTED]"` | CWE-532 (Info Exposure via Log) |
| No passwords in crash reports | `Sentry` scrubbing rules exclude `password`, `credential`, `secret` fields | CWE-209 (Info Exposure via Error) |
| Memory zeroization | `zeroize` crate on password `String` fields — cleared on drop | CWE-316 (Cleartext in Memory) |
| Env var priority | `DB_PASSWORD_<NAME>` env var checked first — enables CI/CD without keychain | OWASP Secrets Mgmt CS |
| No clipboard leaks | Password fields in ConnectionDialog prevent copy/paste out | CWE-200 (Exposure of Sensitive Info) |

```rust
// crates/database_core/src/credential.rs

use zeroize::Zeroize;

/// A password that is zeroized when dropped.
/// NEVER implements Display or Debug with the actual value.
#[derive(Clone, Zeroize)]
#[zeroize(drop)]
pub struct SecurePassword(String);

impl SecurePassword {
    pub fn new(password: String) -> Self {
        Self(password)
    }

    /// Expose the password for driver consumption.
    /// The caller must not store or log the returned value.
    pub fn expose_secret(&self) -> &str {
        &self.0
    }
}

impl std::fmt::Debug for SecurePassword {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str("[REDACTED]")
    }
}

impl std::fmt::Display for SecurePassword {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str("[REDACTED]")
    }
}

/// Credential loading with priority chain.
/// Follows OWASP Secrets Management Cheat Sheet recommendations.
pub struct CredentialProvider {
    keychain: Box<dyn CredentialsProvider>,
}

impl CredentialProvider {
    /// Load credential with defense-in-depth priority chain:
    /// 1. Environment variable (for CI/CD and containers)
    /// 2. Platform keychain (macOS Keychain, Linux Secret Service, Windows Credential Manager)
    /// 3. Prompt user via modal dialog
    pub async fn load(
        &self,
        connection_id: &ConnectionId,
        connection_name: &str,
        cx: &mut AsyncApp,
    ) -> Result<SecurePassword> {
        // Priority 1: Environment variable
        let env_key = format!(
            "DB_PASSWORD_{}",
            connection_name.to_uppercase().replace(|c: char| !c.is_alphanumeric(), "_")
        );
        if let Ok(password) = std::env::var(&env_key) {
            return Ok(SecurePassword::new(password));
        }

        // Priority 2: Platform keychain
        let keychain_url = format!("zed-db://{}", connection_id.0);
        match self.keychain.read_credentials(&keychain_url).await {
            Ok(Some(credentials)) => {
                let password = String::from_utf8(credentials.password)
                    .map_err(|_| anyhow::anyhow!("Invalid UTF-8 in stored password"))?;
                return Ok(SecurePassword::new(password));
            }
            Ok(None) => { /* Not stored, fall through to prompt */ }
            Err(error) => {
                log::warn!("Keychain read failed for {}: {}", connection_name, error);
                // Fall through to prompt — don't block on keychain errors
            }
        }

        // Priority 3: Prompt user
        Err(anyhow::anyhow!("Credential not found — prompt user"))
    }

    /// Save credential to platform keychain.
    /// Per NIST SP 800-63B-4: no mandatory rotation, no complexity rules.
    pub async fn save(
        &self,
        connection_id: &ConnectionId,
        password: &SecurePassword,
    ) -> Result<()> {
        let keychain_url = format!("zed-db://{}", connection_id.0);
        self.keychain.write_credentials(
            &keychain_url,
            "zed-database",
            password.expose_secret().as_bytes(),
        ).await
    }

    /// Delete credential from keychain (on connection removal).
    pub async fn delete(&self, connection_id: &ConnectionId) -> Result<()> {
        let keychain_url = format!("zed-db://{}", connection_id.0);
        self.keychain.delete_credentials(&keychain_url).await
    }
}
```

### 3. Transport Security (OWASP A04:2025, CWE-319, CWE-311)

> **Reference**: OWASP Transport Layer Security Cheat Sheet: *"When correctly implemented,
> TLS can provide confidentiality, integrity, and authentication."* The OWASP Database
> Security Cheat Sheet states: *"Most database default configurations start with
> unencrypted network connections. Configure the database to only allow encrypted
> connections."*

#### Principle: All Database Connections Must Be Encrypted in Transit

| Connection Type | Encryption Mechanism | Minimum Standard |
|---|---|---|
| Direct to PostgreSQL | TLS 1.2+ via `native-tls` or `rustls` | `SslMode::Prefer` default, `VerifyFull` recommended |
| Direct to MySQL | TLS 1.2+ via `mysql_async` TLS support | Required for non-localhost connections |
| Direct to MSSQL | TDS protocol encryption via `tiberius` | `Encryption::Required` default |
| SQLite (local) | N/A (local file access) | File permissions (0600) |
| SSH Tunnel | SSH encryption (AES-256) | Host key fingerprint verification |

```rust
// crates/database_core/src/driver.rs (TLS section)

/// Build TLS configuration per OWASP TLS Cheat Sheet recommendations.
/// Minimum: TLS 1.2, SHA-256 certificates, 2048-bit keys.
pub fn build_tls_config(ssl_config: &SslConfig) -> Result<TlsConnector> {
    let mut builder = native_tls::TlsConnector::builder();

    // Enforce minimum TLS 1.2 (OWASP TLS CS requirement)
    builder.min_protocol_version(Some(native_tls::Protocol::Tlsv12));

    match ssl_config.mode {
        SslMode::Disable => {
            // Log a warning — unencrypted connections are a risk
            log::warn!("TLS disabled for database connection — data transmitted in cleartext");
            return Err(anyhow::anyhow!("TLS disabled"));
        }
        SslMode::Prefer | SslMode::Require => {
            // Connect with TLS but don't verify server certificate
            builder.danger_accept_invalid_certs(true);
        }
        SslMode::VerifyCa => {
            // Verify server certificate against CA but don't check hostname
            if let Some(ca_path) = &ssl_config.ca_cert_path {
                let ca_cert = std::fs::read(ca_path)?;
                let cert = native_tls::Certificate::from_pem(&ca_cert)?;
                builder.add_root_certificate(cert);
            }
        }
        SslMode::VerifyFull => {
            // Full verification: CA chain + hostname match (RECOMMENDED)
            if let Some(ca_path) = &ssl_config.ca_cert_path {
                let ca_cert = std::fs::read(ca_path)?;
                let cert = native_tls::Certificate::from_pem(&ca_cert)?;
                builder.add_root_certificate(cert);
            }
            // Hostname verification is enabled by default in native-tls
        }
    }

    // Client certificate authentication (mTLS)
    if let (Some(cert_path), Some(key_path)) = (&ssl_config.client_cert_path, &ssl_config.client_key_path) {
        let cert = std::fs::read(cert_path)?;
        let key = std::fs::read(key_path)?;
        let identity = native_tls::Identity::from_pkcs8(&cert, &key)?;
        builder.identity(identity);
    }

    Ok(builder.build()?)
}
```

**SSH tunnel host key verification:**

```rust
// crates/database_core/src/ssh_tunnel.rs (security section)

/// SSH host key verification strategy.
/// Per OWASP: always verify host keys to prevent MITM attacks.
pub enum HostKeyVerification {
    /// Reject unknown hosts — most secure, requires known_hosts file
    Strict,
    /// Accept on first connection, reject on change (TOFU — Trust On First Use)
    TrustOnFirstUse { known_hosts_path: PathBuf },
    /// Accept all keys — INSECURE, only for development
    AcceptAll,
}

impl HostKeyVerification {
    pub fn verify(&self, host: &str, key: &ssh_key::PublicKey) -> Result<()> {
        match self {
            Self::Strict => {
                // Look up host in known_hosts, reject if not found or mismatch
                todo!()
            }
            Self::TrustOnFirstUse { known_hosts_path } => {
                // Accept if not seen before (and store), reject if key changed
                // Show warning to user on first connection
                todo!()
            }
            Self::AcceptAll => {
                log::warn!("SSH host key verification disabled — vulnerable to MITM");
                Ok(())
            }
        }
    }
}
```

### 4. Access Control and Least Privilege (OWASP A01:2025)

> **Reference**: OWASP ranks Broken Access Control as **A01:2025** (#1 risk). 100%
> of tested applications had some form of broken access control. The OWASP Database
> Security Cheat Sheet states: *"DO NOT ASSIGN DBA OR ADMIN TYPE ACCESS TO YOUR
> APPLICATION ACCOUNTS."* The principle of least privilege (PoLP) dictates that each
> connection should have only the minimum permissions required.

#### Principle: Enforce Least Privilege at Every Layer

**Layer 1 — Connection-level read-only mode:**

```rust
// crates/database_core/src/query_executor.rs (access control section)

/// Guard that enforces read-only mode per connection.
/// This is the first line of defense — even before SQL reaches the driver.
pub struct ReadOnlyGuard;

impl ReadOnlyGuard {
    pub fn check(sql: &str, config: &ConnectionConfig) -> Result<()> {
        if !config.read_only {
            return Ok(()); // No restriction
        }

        let statement_type = classify_statement(sql)?;
        if statement_type.is_mutating() {
            return Err(DatabaseError::ReadOnlyViolation {
                statement_type: format!("{:?}", statement_type),
            }.into());
        }
        Ok(())
    }
}
```

**Layer 2 — AI agent action allowlist:**

```rust
// crates/database_ai/src/autonomous_agent.rs (access control section)

/// Controls what operations the AI agent can perform.
/// Follows OWASP principle: "minimize privileges assigned to every account."
pub struct ActionAllowlist {
    pub allow_select: bool,        // Default: true
    pub allow_insert: bool,        // Default: false
    pub allow_update: bool,        // Default: false
    pub allow_delete: bool,        // Default: false
    pub allow_ddl: bool,           // Default: false
    pub allow_explain: bool,       // Default: true
    pub max_rows_affected: usize,  // Default: 100
    pub require_confirmation: bool, // Default: true for write operations
}

impl ActionAllowlist {
    pub fn check_permission(&self, statement_type: &StatementType) -> Result<()> {
        match statement_type {
            StatementType::ReadOnly => {
                if !self.allow_select {
                    return Err(anyhow::anyhow!("SELECT not allowed by agent policy"));
                }
            }
            StatementType::Insert => {
                if !self.allow_insert {
                    return Err(anyhow::anyhow!("INSERT not allowed by agent policy"));
                }
            }
            StatementType::Update => {
                if !self.allow_update {
                    return Err(anyhow::anyhow!("UPDATE not allowed by agent policy"));
                }
            }
            StatementType::Delete => {
                if !self.allow_delete {
                    return Err(anyhow::anyhow!("DELETE not allowed by agent policy"));
                }
            }
            StatementType::Ddl | StatementType::Dcl => {
                if !self.allow_ddl {
                    return Err(anyhow::anyhow!("DDL/DCL not allowed by agent policy"));
                }
            }
            _ => {}
        }
        Ok(())
    }

    /// Default allowlist: read-only, safe for autonomous operation
    pub fn default_safe() -> Self {
        Self {
            allow_select: true,
            allow_insert: false,
            allow_update: false,
            allow_delete: false,
            allow_ddl: false,
            allow_explain: true,
            max_rows_affected: 100,
            require_confirmation: true,
        }
    }
}
```

**Layer 3 — UI-level confirmation for destructive operations:**

All data modifications go through the DML preview dialog (Section 3.16). This
ensures the user sees the exact SQL before it executes. The dialog follows Cooper's
"no data loss" principle and provides a last line of defense.

### 5. Input Validation and Sanitization (OWASP A05:2025)

> **Reference**: OWASP Injection Prevention: *"Assume all input is malicious. Use
> an 'accept known good' input validation strategy."* CWE-89: *"Apply allow-listing
> rather than block-lists."*

#### Validation Layers

| Input Source | Validation Rule | Implementation |
|---|---|---|
| Column names in sort/filter | Allow-list: must exist in `schema_cache.columns` | `build_filtered_query()` checks `column_index < columns.len()` |
| Table names from explorer | Allow-list: must exist in `schema_cache.objects` | `quote_identifier()` + schema validation |
| Page size | Integer, capped: 1 ≤ n ≤ 10,000 | `page_size.clamp(1, 10_000)` |
| Page number | Non-negative integer | `page.max(0)` |
| Filter values | Parameterized bind (never concatenated) | `$1` / `?` placeholders in SQL |
| Connection form fields | Type-specific: port is u16, host is non-empty string | `ConnectionConfig::validate()` |
| SQL from QueryEditor | User-trusted (full SQL editor) but checked for read-only | `ReadOnlyGuard::check()` |
| SQL from AI agent | Untrusted: classified + allowlist checked + confirmation | `ActionAllowlist::check_permission()` |

```rust
// crates/database_core/src/schema.rs (validation section)

impl ConnectionConfig {
    /// Validate all fields before attempting connection.
    /// Applies input validation per OWASP guidelines.
    pub fn validate(&self) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();

        if self.name.trim().is_empty() {
            errors.push("Connection name is required".to_string());
        }
        if self.name.len() > 255 {
            errors.push("Connection name must be 255 characters or less".to_string());
        }

        match self.driver {
            DriverType::Sqlite => {
                if self.sqlite_path.is_none() {
                    errors.push("SQLite file path is required".to_string());
                }
            }
            _ => {
                if self.host.as_ref().map_or(true, |h| h.trim().is_empty()) {
                    errors.push("Host is required".to_string());
                }
                if let Some(host) = &self.host {
                    if host.len() > 253 {
                        errors.push("Host must be 253 characters or less".to_string());
                    }
                    // Reject hosts with SQL injection patterns
                    if host.contains(';') || host.contains('\'') || host.contains('"') {
                        errors.push("Host contains invalid characters".to_string());
                    }
                }
                if let Some(port) = self.port {
                    if port == 0 {
                        errors.push("Port must be between 1 and 65535".to_string());
                    }
                }
                if let Some(user) = &self.user {
                    if user.len() > 128 {
                        errors.push("Username must be 128 characters or less".to_string());
                    }
                }
            }
        }

        if self.color_index >= 8 {
            errors.push("Color index must be 0-7".to_string());
        }

        if errors.is_empty() { Ok(()) } else { Err(errors) }
    }
}
```

### 6. Denial of Service Protection (OWASP A10:2025, CWE-400)

> **Reference**: OWASP A10:2025 addresses *"Mishandling of Exceptional Conditions"*
> — this includes resource exhaustion. CWE-400 covers uncontrolled resource consumption.

#### Protections Implemented

| Threat | Mitigation | Setting |
|---|---|---|
| Infinite/huge result set | Server-side `LIMIT` on all queries | `DatabaseSettings.page_size` (default: 500, max: 10,000) |
| Runaway query | Configurable query timeout per connection | `ConnectionConfig.query_timeout` (default: 30s) |
| Memory exhaustion from LOBs | LOB size cap with interactive "load more" | `DatabaseSettings.max_lob_size` (default: 1 KB) |
| Connection pool exhaustion | Max connections per pool with queue timeout | `ConnectionPoolConfig.max_connections` (default: 5) |
| Too many open tabs | No hard limit, but lazy result cleanup on tab close | Results freed on `on_removed()` |
| Malicious SSH server | Connection timeout + host key verification | SSH connect timeout: 10s |

```rust
// crates/database_core/src/connection_pool.rs (DoS protection section)

pub struct ConnectionPoolConfig {
    pub max_connections: usize,      // Default: 5
    pub idle_timeout: Duration,      // Default: 5 minutes
    pub connection_timeout: Duration, // Default: 10 seconds
    pub query_timeout: Duration,     // Default: 30 seconds
    pub max_query_size: usize,       // Default: 1 MB (SQL text size)
}

impl Default for ConnectionPoolConfig {
    fn default() -> Self {
        Self {
            max_connections: 5,
            idle_timeout: Duration::from_secs(300),
            connection_timeout: Duration::from_secs(10),
            query_timeout: Duration::from_secs(30),
            max_query_size: 1_048_576, // 1 MB
        }
    }
}

/// Validate query size before execution (defense against oversized payloads).
pub fn validate_query_size(sql: &str, max_size: usize) -> Result<()> {
    if sql.len() > max_size {
        return Err(DatabaseError::QueryTooLarge {
            actual: sql.len(),
            limit: max_size,
        }.into());
    }
    Ok(())
}
```

### 7. Logging and Audit Trail (OWASP A09:2025)

> **Reference**: OWASP A09:2025 (*"Security Logging and Alerting Failures"*): Applications
> must log security-relevant events with enough detail to reconstruct what happened, but
> without exposing sensitive data. CWE-532 addresses information exposure through log files.

#### Logging Rules

| Event | Logged Data | Redacted Data |
|---|---|---|
| Connection established | Connection name, host, port, driver, timestamp | Password, SSH key |
| Connection failed | Connection name, error type, timestamp | Password, detailed error if auth-related |
| Query executed | SQL (first 500 chars), duration, row count, connection name | Full SQL if > 500 chars (truncated) |
| Query failed | SQL (first 500 chars), error message, connection name | Sensitive error details |
| Data modification committed | Table name, operation count (N inserts, N updates, N deletes) | Actual data values |
| Credential access | Connection name, source (keychain/env/prompt) | Password value |
| SSL/TLS connection | Mode, cipher suite, certificate info | Private key material |

```rust
// crates/database_core/src/errors.rs (logging section)

impl DatabaseError {
    /// Log the error with appropriate redaction.
    /// Per OWASP: log enough to diagnose, never log secrets.
    pub fn log_securely(&self) {
        match self {
            Self::AuthenticationFailed { user } => {
                // Log the attempt but not the password
                log::warn!("Authentication failed for user '{}' — check credentials", user);
            }
            Self::QueryFailed { sql_preview, db_error, .. } => {
                // Truncate SQL to avoid logging sensitive data in queries
                let safe_preview = if sql_preview.len() > 500 {
                    format!("{}...[truncated]", &sql_preview[..500])
                } else {
                    sql_preview.clone()
                };
                log::error!("Query failed: {} — SQL: {}", db_error, safe_preview);
            }
            Self::ConnectionFailed { host, port, cause } => {
                log::error!("Connection failed to {}:{} — {}", host, port, cause);
            }
            _ => {
                log::error!("{}", self);
            }
        }
    }
}

/// Redact sensitive patterns from arbitrary strings (for Sentry, crash reports).
pub fn redact_sensitive(text: &str) -> String {
    let patterns = [
        (r"(?i)password\s*=\s*'[^']*'", "password='[REDACTED]'"),
        (r"(?i)password\s*=\s*\S+", "password=[REDACTED]"),
        (r"(?i)secret\s*=\s*'[^']*'", "secret='[REDACTED]'"),
        (r"(?i)token\s*=\s*\S+", "token=[REDACTED]"),
    ];
    let mut result = text.to_string();
    for (pattern, replacement) in &patterns {
        if let Ok(regex) = regex::Regex::new(pattern) {
            result = regex.replace_all(&result, *replacement).to_string();
        }
    }
    result
}
```

### 8. Rust-Specific Security Advantages

> **Reference**: The Rust Security Response Working Group maintains ongoing security
> advisories. SQLx uses `#![forbid(unsafe_code)]` for PostgreSQL and MySQL drivers.
> Rust's ownership model eliminates use-after-free, buffer overflows, and data races
> at compile time.

| Rust Feature | Security Benefit | Relevant CWE |
|---|---|---|
| Ownership + borrowing | No use-after-free, no double-free | CWE-416, CWE-415 |
| No null pointers | No null dereference crashes | CWE-476 |
| Bounds checking | No buffer overflow on indexing | CWE-120, CWE-787 |
| `Send` + `Sync` traits | No data races in concurrent code | CWE-362 |
| `sqlx::query!` macro | Compile-time SQL type checking | CWE-89 (partial) |
| `#![forbid(unsafe_code)]` | No unsafe memory operations in driver code | CWE-119 |
| `zeroize` crate | Guaranteed memory clearing of secrets | CWE-316 |
| `Result<T, E>` everywhere | Forced error handling, no silent failures | CWE-252, CWE-391 |

**Crate-level safety declarations:**

```rust
// crates/database_core/src/database_core.rs (top of file)

// Forbid unsafe code in the entire crate — all memory safety
// guarantees are enforced by the Rust compiler.
// Exception: SQLite driver uses rusqlite which contains unsafe internally,
// but our code wrapping it does not.
#![deny(unsafe_code)]
#![deny(clippy::unwrap_used)]   // Per CLAUDE.md: no panicking operations
#![deny(clippy::expect_used)]
#![warn(clippy::pedantic)]
```

### 9. Supply Chain Security (OWASP A03:2025)

> **Reference**: OWASP A03:2025 (*"Software Supply Chain Failures"*) is a **new
> category** in the 2025 edition, reflecting the growing risk of compromised dependencies.

#### Dependency Audit Requirements

| Dependency | Purpose | Audit Status | Notes |
|---|---|---|---|
| `sqlx` | PostgreSQL/MySQL/SQLite driver | Widely audited, `#![forbid(unsafe_code)]` for PG/MySQL | Pure Rust, no C bindings for PG/MySQL |
| `rusqlite` | SQLite bindings | Contains unsafe (C FFI) | Mature, well-maintained |
| `tokio-postgres` | Alternative PG driver | Widely used | Consider vs sqlx |
| `mysql_async` | MySQL driver | | |
| `tiberius` | MSSQL driver | | TDS protocol implementation |
| `russh` | SSH tunneling | | Pure Rust SSH |
| `native-tls` | TLS | Platform TLS backend | Uses OS TLS libraries |
| `zeroize` | Secret memory clearing | RustCrypto project, audited | |
| `csv` | CSV export | No security-sensitive code | Data processing only |

**Verification steps (to be run in CI):**

```bash
# Audit all dependencies for known vulnerabilities (RUSTSEC advisories)
cargo audit

# Check for unmaintained crates
cargo audit --deny unmaintained

# Verify no new unsafe code introduced
cargo clippy -- -D unsafe_code

# Check for dependency confusion (ensure all deps are from crates.io)
cargo deny check sources
```

### 10. Security Testing Plan

| Test Category | Tool / Method | Phase | Frequency |
|---|---|---|---|
| SQL injection fuzzing | Custom test harness with malicious inputs | Phase 0 | Every PR |
| Credential leak detection | `grep` for password patterns in logs, settings | Phase 0 | Every PR |
| TLS verification | Integration test with invalid certs | Phase 1 | Every PR |
| Read-only enforcement | Unit tests for `ReadOnlyGuard` with all statement types | Phase 0 | Every PR |
| AI allowlist enforcement | Unit tests for `ActionAllowlist` with all statement types | Phase 5 | Every PR |
| Dependency audit | `cargo audit` | Phase 0 | Weekly CI + every PR |
| Memory safety (secrets) | Test `zeroize` behavior with `miri` | Phase 0 | Monthly |
| Input validation | Fuzz `ConnectionConfig::validate()` with arbitrary strings | Phase 1 | Every PR |
| Statement classification | Unit tests for `classify_statement()` with edge cases | Phase 0 | Every PR |

```rust
// crates/database_core/src/tests/security_tests.rs

#[cfg(test)]
mod sql_injection_tests {
    use super::*;

    #[test]
    fn test_quote_identifier_prevents_injection() {
        // Double-quote escaping prevents breaking out of identifiers
        assert_eq!(quote_identifier("users"), "\"users\"");
        assert_eq!(quote_identifier("users\"--"), "\"users\"\"--\"");
        assert_eq!(
            quote_identifier("a\"; DROP TABLE users; --"),
            "\"a\"\"; DROP TABLE users; --\""
        );
    }

    #[test]
    fn test_classify_statement_read_only() {
        assert_eq!(classify_statement("SELECT * FROM users").unwrap(), StatementType::ReadOnly);
        assert_eq!(classify_statement("  SELECT 1").unwrap(), StatementType::ReadOnly);
        assert_eq!(classify_statement("EXPLAIN SELECT 1").unwrap(), StatementType::ReadOnly);
        assert_eq!(classify_statement("WITH cte AS (SELECT 1) SELECT * FROM cte").unwrap(), StatementType::ReadOnly);
    }

    #[test]
    fn test_classify_statement_mutating() {
        assert!(classify_statement("INSERT INTO users VALUES (1)").unwrap().is_mutating());
        assert!(classify_statement("UPDATE users SET name = 'x'").unwrap().is_mutating());
        assert!(classify_statement("DELETE FROM users").unwrap().is_mutating());
        assert!(classify_statement("DROP TABLE users").unwrap().is_mutating());
        assert!(classify_statement("ALTER TABLE users ADD COLUMN x INT").unwrap().is_mutating());
        assert!(classify_statement("TRUNCATE users").unwrap().is_mutating());
        assert!(classify_statement("GRANT SELECT ON users TO role").unwrap().is_mutating());
    }

    #[test]
    fn test_classify_statement_with_comments() {
        // SQL comments should not bypass classification
        assert!(classify_statement("-- SELECT\nDELETE FROM users").unwrap().is_mutating());
        assert!(classify_statement("/* comment */ DROP TABLE x").unwrap().is_mutating());
    }

    #[test]
    fn test_read_only_guard_blocks_mutations() {
        let config = ConnectionConfig {
            read_only: true,
            ..test_config()
        };
        assert!(ReadOnlyGuard::check("SELECT 1", &config).is_ok());
        assert!(ReadOnlyGuard::check("DELETE FROM users", &config).is_err());
        assert!(ReadOnlyGuard::check("DROP TABLE users", &config).is_err());
    }

    #[test]
    fn test_build_filtered_query_parameterizes_values() {
        let columns = vec![test_column("name"), test_column("age")];
        let filters = vec![ColumnFilter {
            column_index: 0,
            filter_type: FilterType::Equals,
            value: "'; DROP TABLE users; --".to_string(), // Injection attempt
        }];
        let (sql, params) = build_filtered_query(
            &TableRef { catalog: None, schema: None, name: "users".into() },
            &columns, &filters, &[], 0, 500,
        );
        // Value must be in params, NOT in the SQL string
        assert!(!sql.contains("DROP TABLE"));
        assert!(sql.contains("$1"));
        assert_eq!(params.len(), 1);
    }

    #[test]
    fn test_page_size_capped() {
        let (sql, _) = build_filtered_query(
            &TableRef { catalog: None, schema: None, name: "t".into() },
            &[], &[], &[], 0, 999_999, // Unreasonable page size
        );
        assert!(sql.contains("LIMIT 10000")); // Capped to max
    }
}

#[cfg(test)]
mod credential_tests {
    use super::*;

    #[test]
    fn test_secure_password_debug_is_redacted() {
        let password = SecurePassword::new("my_secret_password".to_string());
        let debug_output = format!("{:?}", password);
        assert_eq!(debug_output, "[REDACTED]");
        assert!(!debug_output.contains("my_secret"));
    }

    #[test]
    fn test_secure_password_display_is_redacted() {
        let password = SecurePassword::new("my_secret_password".to_string());
        let display_output = format!("{}", password);
        assert_eq!(display_output, "[REDACTED]");
    }

    #[test]
    fn test_redact_sensitive_patterns() {
        let input = "host=db.example.com password='secret123' port=5432";
        let redacted = redact_sensitive(input);
        assert!(!redacted.contains("secret123"));
        assert!(redacted.contains("[REDACTED]"));
        assert!(redacted.contains("host=db.example.com"));
    }

    #[test]
    fn test_connection_config_validation_rejects_injection() {
        let config = ConnectionConfig {
            host: Some("db.example.com'; DROP TABLE--".to_string()),
            ..test_config()
        };
        assert!(config.validate().is_err());
    }
}

#[cfg(test)]
mod ai_allowlist_tests {
    use super::*;

    #[test]
    fn test_default_safe_allowlist_blocks_writes() {
        let allowlist = ActionAllowlist::default_safe();
        assert!(allowlist.check_permission(&StatementType::ReadOnly).is_ok());
        assert!(allowlist.check_permission(&StatementType::Insert).is_err());
        assert!(allowlist.check_permission(&StatementType::Update).is_err());
        assert!(allowlist.check_permission(&StatementType::Delete).is_err());
        assert!(allowlist.check_permission(&StatementType::Ddl).is_err());
    }
}
```

### Security Compliance Summary

| OWASP Top 10:2025 Category | Relevance | Our Mitigation | Section |
|---|---|---|---|
| **A01** Broken Access Control | Read-only bypass, AI privilege escalation | ReadOnlyGuard, ActionAllowlist, DML preview | §4 |
| **A02** Security Misconfiguration | Default insecure connection settings | `SslMode::Prefer` default, SSH host key verification | §3 |
| **A03** Supply Chain Failures | Compromised Rust crates | `cargo audit`, `cargo deny`, dependency pinning | §9 |
| **A04** Cryptographic Failures | Cleartext credentials, unencrypted connections | Platform keychain, TLS enforcement, `zeroize` | §2, §3 |
| **A05** Injection (SQL) | User input in SQL queries | Parameterized queries, identifier quoting, allow-lists | §1 |
| **A06** Insecure Design | Missing security controls by design | STRIDE threat model, defense-in-depth layers | All |
| **A07** Authentication Failures | Weak credential management | NIST SP 800-63B-4 compliance, env var chain | §2 |
| **A08** Software/Data Integrity | Modified query results, tampered settings | TLS integrity, settings file permissions | §3 |
| **A09** Logging Failures | Missing audit trail, sensitive data in logs | Structured logging with redaction | §7 |
| **A10** Exceptional Conditions | Resource exhaustion, unhandled errors | Query timeout, page size cap, `Result<T,E>` everywhere | §6 |

---

## Phase 0 — Infrastructure

Foundation work required before any feature development.

### 0.1 DatabaseSettings type + registration

Register a new settings type following Zed's hierarchical settings system.
Supports both global (`~/.config/zed/settings.json`) and project-level
(`.zed/settings.json`) configuration.

**Steps:**
1. Create `crates/database_core/src/settings.rs`
2. Define `DatabaseSettings` struct with `#[derive(RegisterSetting)]`
3. Create `crates/database_core/src/settings/default.json` with default values
4. Register in `database_ui.rs` init via `SettingsStore::register::<DatabaseSettings>()`
5. Add JSON schema entries for `settings.json` documentation

```rust
// crates/database_core/src/settings.rs

#[derive(Clone, Debug, Serialize, Deserialize, JsonSchema, RegisterSetting)]
#[setting(schema_only)]
pub struct DatabaseSettings {
    pub page_size: usize,                              // Rows per page (default: 500)
    pub default_sort_mode: SortMode,                   // Server or Client
    pub dock_position: DockPosition,                   // Left, Right, or Bottom
    pub max_lob_size: usize,                           // Max LOB bytes to load (default: 1024)
    pub default_export_format: ExportFormat,            // CSV, JSON, SQL, etc.
    pub show_null_indicator: bool,                      // Render NULL distinctly
    pub grid_font_family: Option<String>,               // Dedicated grid font
    pub grid_font_size: Option<f32>,                    // Dedicated grid font size
    pub number_format: NumberFormatSettings,            // Decimal/grouping separators
    pub auto_commit: bool,                              // Auto-commit or manual
    pub default_introspection_level: IntrospectionLevel, // 1, 2, or 3
    pub connections: Vec<ConnectionConfig>,             // Saved connections (NO passwords)
    pub show_quick_actions_toolbar: bool,               // Floating toolbar on cell select
    pub tab_naming_prefix: Option<String>,              // Comment prefix for tab naming
}

#[derive(Clone, Debug, Serialize, Deserialize, JsonSchema)]
pub struct NumberFormatSettings {
    pub decimal_separator: char,      // '.' or ','
    pub grouping_separator: Option<char>, // ',' or ' ' or None
    pub grouping_size: usize,         // Usually 3
}

#[derive(Clone, Debug, Serialize, Deserialize, JsonSchema)]
pub enum SortMode { Server, Client }

#[derive(Clone, Debug, Serialize, Deserialize, JsonSchema)]
pub enum ExportFormat { Csv, Tsv, Json, SqlInsert, SqlDdlDml, Html, Markdown, Excel }

#[derive(Clone, Copy, Debug, Serialize, Deserialize, JsonSchema)]
pub enum IntrospectionLevel { Names = 1, Columns = 2, FullDdl = 3 }

#[derive(Clone, Debug, Serialize, Deserialize, JsonSchema)]
pub struct ConnectionConfig {
    pub id: String,                   // UUID
    pub name: String,
    pub driver: DriverType,
    pub host: Option<String>,
    pub port: Option<u16>,
    pub database: Option<String>,
    pub user: Option<String>,
    pub color_index: usize,           // Index into CONNECTION_COLORS
    pub read_only: bool,
    pub ssh: Option<SshTunnelConfig>,
    pub ssl: Option<SslConfig>,
    pub sqlite_path: Option<String>,
}
```

### 0.2 Credential storage via CredentialsProvider

Database passwords stored securely via platform keychain, following the pattern
in `crates/language_model/src/api_key.rs`.

**Steps:**
1. Create `crates/database_core/src/credential.rs`
2. Implement `DatabaseCredential` with `CredentialsProvider` integration
3. Implement priority chain: env var > keychain > prompt
4. Add password caching in memory (cleared on disconnect)

```rust
// crates/database_core/src/credential.rs

pub struct DatabaseCredential {
    pub connection_id: ConnectionId,
    url: SharedString,               // "zed-db://<connection-name>" as keychain key
    load_status: CredentialLoadStatus,
}

pub enum CredentialLoadStatus {
    NotLoaded,
    Loading(Task<()>),
    Loaded(String),              // Password in memory
    Failed(String),              // Error message
}

impl DatabaseCredential {
    pub fn new(connection_id: ConnectionId) -> Self { /* ... */ }

    /// Load credential with priority: env var > keychain > prompt user
    pub async fn load(&mut self, cx: &mut AsyncApp) -> Result<String> {
        // 1. Check env var DB_PASSWORD_<CONNECTION_NAME>
        // 2. Check platform keychain via CredentialsProvider
        // 3. If not found, prompt user via modal
        todo!()
    }

    /// Save to platform keychain
    pub async fn save(&self, password: &str, cx: &mut AsyncApp) -> Result<()> {
        todo!()
    }

    /// Delete from keychain
    pub async fn delete(&self, cx: &mut AsyncApp) -> Result<()> {
        todo!()
    }
}
```

### 0.3 DatabaseError types + notification integration

**Steps:**
1. Create `crates/database_core/src/errors.rs`
2. Define all error variants with structured data
3. Implement `Display`, `Error`, conversion traits
4. Implement `DatabaseError::notify()` for UI notification with actions

```rust
// crates/database_core/src/errors.rs

#[derive(Debug, thiserror::Error)]
pub enum DatabaseError {
    #[error("Connection failed to {host}:{port}: {cause}")]
    ConnectionFailed { host: String, port: u16, cause: String },

    #[error("Authentication failed for user '{user}'")]
    AuthenticationFailed { user: String },

    #[error("Query timed out after {duration:?}")]
    QueryTimeout { sql_preview: String, duration: Duration },

    #[error("Query failed: {db_error}")]
    QueryFailed {
        sql_preview: String,
        db_error: String,
        position: Option<usize>,  // Byte offset in SQL for cursor positioning
    },

    #[error("Connection lost{}", if *.was_in_transaction { " (transaction in progress)" } else { "" })]
    ConnectionLost { was_in_transaction: bool },

    #[error("SSL error: {cause}")]
    SslError { cause: String },

    #[error("SSH tunnel failed: {cause}")]
    SshTunnelFailed { cause: String },

    #[error("LOB size exceeded: {actual} bytes (limit: {limit}) in column '{column}'")]
    LobSizeExceeded { actual: usize, limit: usize, column: String },

    #[error("Driver not found: {driver}")]
    DriverNotFound { driver: String },

    #[error("Operation cancelled")]
    Cancelled,
}

impl DatabaseError {
    /// Show error in Zed's notification system with contextual actions
    pub fn notify(&self, cx: &mut App) {
        match self {
            Self::ConnectionFailed { .. } => {
                // Notification with "Retry" and "Edit Connection" buttons
            }
            Self::QueryFailed { position, .. } => {
                // Notification with "Go to Error" action if position is Some
            }
            Self::ConnectionLost { was_in_transaction } => {
                // Notification with "Reconnect" action
                // Warning about pending transaction if applicable
            }
            Self::LobSizeExceeded { .. } => {
                // Notification with "Load Full Content" action
            }
            _ => { /* Standard error notification */ }
        }
    }

    /// Convert to a user-facing error banner element for inline display
    pub fn to_error_banner(&self, cx: &App) -> AnyElement {
        todo!()
    }
}
```

### 0.4 Database icon in Zed icon system

**Steps:**
1. Add SVG icons to `assets/icons/` for each database object type
2. Register new `IconName` variants in `crates/ui/src/components/icon.rs`
3. Test rendering at all icon sizes (`XSmall`, `Small`, `Medium`, `Large`)

New `IconName` variants needed:

```rust
// Added to the IconName enum in crates/ui/src/components/icon.rs
DatabaseZap,        // Connected database (existing)
Database,           // Disconnected database (existing)
Table,              // Table object (new SVG)
Column,             // Column (new SVG, or reuse Minus)
KeyRound,           // Primary key column (new SVG)
ArrowUpRight,       // Foreign key column (existing)
Schema,             // Schema (new SVG, or reuse Layers)
Sequence,           // Sequence (new SVG, or reuse Hash)
StoredProcedure,    // Function/procedure (reuse Code)
```

### 0.5 Test infrastructure

**Steps:**
1. Create `crates/database_core/src/tests/` directory
2. Implement `MockDatabaseConnection` for unit tests
3. Create `TestConnectionFactory` for SQLite in-memory
4. Create GPUI test helpers in `crates/database_ui/src/tests/`

```rust
// crates/database_core/src/tests/mock_connection.rs

pub struct MockDatabaseConnection {
    pub query_results: HashMap<String, QueryResult>,
    pub schema: Vec<SchemaObject>,
    pub execute_log: Arc<Mutex<Vec<String>>>,
    pub should_fail: Option<DatabaseError>,
    pub latency: Option<Duration>,
}

#[async_trait]
impl DatabaseConnection for MockDatabaseConnection {
    async fn execute_query(&self, sql: &str) -> Result<QueryResult> {
        if let Some(latency) = self.latency {
            smol::Timer::after(latency).await;
        }
        if let Some(error) = &self.should_fail {
            return Err(anyhow::anyhow!("{}", error));
        }
        self.execute_log.lock().push(sql.to_string());
        self.query_results.get(sql)
            .cloned()
            .ok_or_else(|| anyhow::anyhow!("No mock result for: {}", sql))
    }

    async fn execute_statement(&self, sql: &str) -> Result<u64> {
        self.execute_log.lock().push(sql.to_string());
        Ok(1)
    }

    async fn cancel(&self) -> Result<()> { Ok(()) }

    async fn introspect_names(&self) -> Result<Vec<SchemaObject>> {
        Ok(self.schema.clone())
    }

    async fn introspect_columns(&self, table: &TableRef) -> Result<Vec<ColumnInfo>> {
        todo!()
    }

    async fn introspect_ddl(&self, object: &SchemaObject) -> Result<String> {
        todo!()
    }

    async fn foreign_keys(&self, table: &TableRef) -> Result<Vec<ForeignKey>> {
        Ok(vec![])
    }

    async fn indexes(&self, table: &TableRef) -> Result<Vec<IndexInfo>> {
        Ok(vec![])
    }

    async fn explain(&self, sql: &str) -> Result<ExplainPlan> {
        todo!()
    }

    async fn execute_ddl(&self, sql: &str) -> Result<()> {
        self.execute_log.lock().push(sql.to_string());
        Ok(())
    }
}

// Helper to build mock connections quickly in tests
pub struct MockConnectionBuilder {
    connection: MockDatabaseConnection,
}

impl MockConnectionBuilder {
    pub fn new() -> Self { /* ... */ }
    pub fn with_result(mut self, sql: &str, result: QueryResult) -> Self { /* ... */ }
    pub fn with_schema(mut self, objects: Vec<SchemaObject>) -> Self { /* ... */ }
    pub fn with_latency(mut self, duration: Duration) -> Self { /* ... */ }
    pub fn with_failure(mut self, error: DatabaseError) -> Self { /* ... */ }
    pub fn build(self) -> Box<dyn DatabaseConnection> { /* ... */ }
}
```

```rust
// crates/database_ui/src/tests/helpers.rs

/// Create a test workspace with database panels registered
pub fn init_test_workspace(cx: &mut VisualTestContext) -> Entity<Workspace> {
    // 1. Build test project
    // 2. Create workspace
    // 3. Register DatabaseExplorer and ValueEditor panels
    // 4. Register toolbar items
    // 5. Return workspace entity
    todo!()
}

/// Create a QueryEditor with a mock connection for testing
pub fn create_test_query_editor(
    workspace: &Entity<Workspace>,
    connection: Box<dyn DatabaseConnection>,
    cx: &mut VisualTestContext,
) -> Entity<QueryEditor> {
    todo!()
}

/// Assert that a ResultGrid displays the expected data
pub fn assert_grid_contents(
    grid: &Entity<ResultGrid>,
    expected_columns: &[&str],
    expected_rows: &[Vec<&str>],
    cx: &App,
) {
    todo!()
}
```

### 0.6 Action definitions

**Steps:**
1. Create `crates/database_ui/src/actions.rs`
2. Define all actions using `actions!()` macro and `#[derive(Action)]`
3. Register default keybindings in `crates/database_ui/src/key_bindings.rs`

```rust
// crates/database_ui/src/actions.rs

actions!(
    database,
    [
        // Panel toggles
        ToggleDatabaseExplorer,
        ToggleValueEditor,

        // Query execution
        ExecuteQuery,
        CancelQuery,
        ExplainQuery,
        FormatQuery,

        // Grid navigation
        ToggleRecordView,
        DetachResultGrid,
        NextPage,
        PreviousPage,
        SelectAll,
        ExpandSelection,
        ShrinkSelection,

        // Data editing
        EditCell,
        SetCellNull,
        AddRow,
        CloneRow,
        DeleteSelectedRows,
        CommitPendingChanges,
        RevertPendingChanges,
        UndoDataEdit,
        RedoDataEdit,

        // Explorer actions
        RefreshExplorer,
        NewQueryConsole,
        OpenTableData,
        EditTableData,
        CopyQualifiedName,
        FilterExplorerObjects,

        // Connection
        NewConnection,
        EditConnection,
        DisconnectConnection,
        ReconnectConnection,
        TestConnection,

        // Export
        ExportResults,
        CopyAsSql,
        CopyAsCsv,
        CopyAsJson,

        // Column actions
        SortAscending,
        SortDescending,
        ClearSort,
        FilterColumn,
        HideColumn,
        ShowAllColumns,
        ResizeColumnToFit,
        ColumnListSearch,
    ]
);

/// Action with data: sort by specific column
#[derive(Clone, PartialEq, Debug, Deserialize, Action)]
pub struct SortByColumn {
    pub column_index: usize,
    pub direction: SortDirection,
}

/// Action with data: navigate to foreign key target
#[derive(Clone, PartialEq, Debug, Deserialize, Action)]
pub struct NavigateToForeignKey {
    pub row_index: usize,
    pub column_index: usize,
}
```

```rust
// crates/database_ui/src/key_bindings.rs

pub fn register_key_bindings(cx: &mut App) {
    cx.bind_keys([
        // Panel toggles
        KeyBinding::new("ctrl-shift-d", ToggleDatabaseExplorer, None),
        KeyBinding::new("ctrl-shift-b", ToggleValueEditor, None),

        // Query execution (context: QueryEditor)
        KeyBinding::new("ctrl-enter", ExecuteQuery, Some("QueryEditor")),
        KeyBinding::new("escape", CancelQuery, Some("QueryEditor && is_executing")),
        KeyBinding::new("ctrl-shift-e", ExplainQuery, Some("QueryEditor")),
        KeyBinding::new("ctrl-shift-f", FormatQuery, Some("QueryEditor")),

        // Grid navigation (context: ResultGrid)
        KeyBinding::new("enter", EditCell, Some("ResultGrid && !is_editing")),
        KeyBinding::new("f2", EditCell, Some("ResultGrid && !is_editing")),
        KeyBinding::new("escape", CancelQuery, Some("ResultGrid && is_editing")),
        KeyBinding::new("delete", SetCellNull, Some("ResultGrid")),
        KeyBinding::new("ctrl-a", SelectAll, Some("ResultGrid")),
        KeyBinding::new("ctrl-d", CloneRow, Some("ResultGrid")),
        KeyBinding::new("ctrl-minus", DeleteSelectedRows, Some("ResultGrid")),
        KeyBinding::new("ctrl-=", AddRow, Some("ResultGrid")),
        KeyBinding::new("ctrl-z", UndoDataEdit, Some("ResultGrid")),
        KeyBinding::new("ctrl-shift-z", RedoDataEdit, Some("ResultGrid")),
        KeyBinding::new("ctrl-shift-r", ToggleRecordView, Some("ResultGrid")),
        KeyBinding::new("ctrl-shift-v", ToggleValueEditor, Some("ResultGrid")),
        KeyBinding::new("ctrl-pagedown", NextPage, Some("ResultGrid")),
        KeyBinding::new("ctrl-pageup", PreviousPage, Some("ResultGrid")),
        KeyBinding::new("ctrl-f12", ColumnListSearch, Some("ResultGrid")),

        // Explorer (context: DatabaseExplorer)
        KeyBinding::new("f5", RefreshExplorer, Some("DatabaseExplorer")),
        KeyBinding::new("ctrl-f", FilterExplorerObjects, Some("DatabaseExplorer")),
    ]);
}
```

### 0.7 Core data types

**Steps:**
1. Create `crates/database_core/src/schema.rs`
2. Define all schema types used across the application
3. Implement Display, serialization, comparison traits

```rust
// crates/database_core/src/schema.rs

/// Unique identifier for a connection across the application
#[derive(Clone, Debug, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct ConnectionId(pub String);

/// Reference to a table or view in a specific schema
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct TableRef {
    pub catalog: Option<String>,
    pub schema: Option<String>,
    pub name: String,
}

impl TableRef {
    pub fn qualified_name(&self) -> String {
        let mut parts = Vec::new();
        if let Some(catalog) = &self.catalog { parts.push(catalog.as_str()); }
        if let Some(schema) = &self.schema { parts.push(schema.as_str()); }
        parts.push(&self.name);
        parts.join(".")
    }
}

/// A database object discovered via introspection
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct SchemaObject {
    pub object_type: SchemaObjectType,
    pub catalog: Option<String>,
    pub schema: Option<String>,
    pub name: String,
}

#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub enum SchemaObjectType {
    Database,
    Schema,
    Table,
    View,
    MaterializedView,
    Function,
    Procedure,
    Sequence,
    Index,
    Trigger,
}

/// Column metadata from introspection
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct ColumnInfo {
    pub name: String,
    pub data_type: String,          // Raw SQL type string
    pub normalized_type: DataType,  // Normalized enum for cell rendering
    pub is_nullable: bool,
    pub is_primary_key: bool,
    pub is_foreign_key: bool,
    pub default_value: Option<String>,
    pub ordinal_position: usize,
    pub foreign_key_ref: Option<ForeignKeyRef>,
}

/// Normalized data types for type-aware rendering and editing
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub enum DataType {
    Boolean,
    SmallInt,
    Integer,
    BigInt,
    Float,
    Double,
    Decimal { precision: Option<u32>, scale: Option<u32> },
    Varchar { max_length: Option<u32> },
    Text,
    Char { length: u32 },
    Date,
    Time,
    Timestamp,
    TimestampTz,
    Interval,
    Uuid,
    Json,
    Jsonb,
    Xml,
    ByteArray,
    Blob,
    Array { element_type: Box<DataType> },
    Enum { variants: Vec<String> },
    Point,
    Geometry,
    Other(String),
}

/// A cell value in a query result
#[derive(Clone, Debug, Serialize, Deserialize)]
pub enum CellValue {
    Null,
    Boolean(bool),
    Integer(i64),
    Float(f64),
    String(String),
    Bytes(Vec<u8>),
    Date(chrono::NaiveDate),
    Time(chrono::NaiveTime),
    Timestamp(chrono::NaiveDateTime),
    TimestampTz(chrono::DateTime<chrono::Utc>),
    Json(serde_json::Value),
    Uuid(uuid::Uuid),
    Array(Vec<CellValue>),
}

impl CellValue {
    pub fn is_null(&self) -> bool { matches!(self, CellValue::Null) }

    pub fn display_string(&self, settings: &NumberFormatSettings) -> String {
        match self {
            CellValue::Null => "NULL".to_string(),
            CellValue::Boolean(b) => b.to_string(),
            CellValue::Integer(i) => format_number(*i as f64, settings),
            CellValue::Float(f) => format_number(*f, settings),
            CellValue::String(s) => s.clone(),
            CellValue::Bytes(b) => format!("{} bytes", b.len()),
            CellValue::Json(v) => serde_json::to_string(v).unwrap_or_default(),
            // ... other variants
            _ => format!("{:?}", self),
        }
    }
}

/// Result of a query execution
#[derive(Clone, Debug)]
pub struct QueryResult {
    pub columns: Vec<ColumnInfo>,
    pub rows: Vec<Vec<CellValue>>,
    pub rows_affected: Option<u64>,
    pub execution_time: Duration,
    pub has_more_rows: bool,         // True if server-side pagination truncated
    pub total_row_count: Option<u64>, // If known (e.g., from COUNT(*))
    pub warnings: Vec<String>,
}

/// Foreign key reference
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct ForeignKeyRef {
    pub referenced_table: TableRef,
    pub referenced_column: String,
}

/// Foreign key metadata
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct ForeignKey {
    pub name: String,
    pub columns: Vec<String>,
    pub referenced_table: TableRef,
    pub referenced_columns: Vec<String>,
    pub on_delete: ForeignKeyAction,
    pub on_update: ForeignKeyAction,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub enum ForeignKeyAction { NoAction, Restrict, Cascade, SetNull, SetDefault }

/// Index metadata
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct IndexInfo {
    pub name: String,
    pub columns: Vec<String>,
    pub is_unique: bool,
    pub is_primary: bool,
    pub index_type: String,         // btree, hash, gin, gist, etc.
}

/// Execution plan from EXPLAIN
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct ExplainPlan {
    pub raw_text: String,
    pub nodes: Vec<ExplainNode>,
    pub total_cost: Option<f64>,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct ExplainNode {
    pub operation: String,
    pub object: Option<String>,
    pub estimated_rows: Option<u64>,
    pub actual_rows: Option<u64>,
    pub cost: Option<f64>,
    pub children: Vec<ExplainNode>,
}
```

---

## Phase 1 — Connection & Schema

### 1.1 DatabaseDriver trait + SQLite driver

Implement the core trait and the SQLite driver first (simplest for testing).
Use `rusqlite` (already a dependency via `sqlez`).

**Steps:**
1. Create `crates/database_core/src/driver.rs` with trait definitions
2. Create `crates/database_core/src/drivers/sqlite.rs`
3. Implement `DatabaseDriver` and `DatabaseConnection` for SQLite
4. Write integration tests with in-memory SQLite
5. Implement type mapping from SQLite types to `DataType` enum

```rust
// crates/database_core/src/driver.rs

/// Identifies a supported database driver
#[derive(Clone, Debug, PartialEq, Eq, Hash, Serialize, Deserialize, JsonSchema)]
pub enum DriverType {
    Sqlite,
    Postgres,
    Mysql,
    Mssql,
    Duckdb,
}

impl DriverType {
    pub fn default_port(&self) -> Option<u16> {
        match self {
            Self::Postgres => Some(5432),
            Self::Mysql => Some(3306),
            Self::Mssql => Some(1433),
            _ => None,
        }
    }

    pub fn display_name(&self) -> &'static str {
        match self {
            Self::Sqlite => "SQLite",
            Self::Postgres => "PostgreSQL",
            Self::Mysql => "MySQL",
            Self::Mssql => "SQL Server",
            Self::Duckdb => "DuckDB",
        }
    }
}

#[async_trait]
pub trait DatabaseDriver: Send + Sync {
    fn driver_type(&self) -> DriverType;
    fn supported_types(&self) -> Vec<DataType>;

    /// Connect to the database. The connection owns its resources
    /// and should be held in an Entity for lifecycle management.
    async fn connect(&self, config: &ConnectionConfig) -> Result<Box<dyn DatabaseConnection>>;

    /// Validate a config without connecting (check required fields, etc.)
    fn validate_config(&self, config: &ConnectionConfig) -> Result<()>;
}

#[async_trait]
pub trait DatabaseConnection: Send + Sync {
    // --- Query execution ---
    async fn execute_query(&self, sql: &str) -> Result<QueryResult>;
    async fn execute_statement(&self, sql: &str) -> Result<u64>;
    async fn cancel(&self) -> Result<()>;

    // --- Introspection (3 levels) ---
    async fn introspect_names(&self) -> Result<Vec<SchemaObject>>;
    async fn introspect_columns(&self, table: &TableRef) -> Result<Vec<ColumnInfo>>;
    async fn introspect_ddl(&self, object: &SchemaObject) -> Result<String>;

    // --- Metadata ---
    async fn foreign_keys(&self, table: &TableRef) -> Result<Vec<ForeignKey>>;
    async fn indexes(&self, table: &TableRef) -> Result<Vec<IndexInfo>>;
    async fn explain(&self, sql: &str) -> Result<ExplainPlan>;

    // --- Schema modification ---
    async fn execute_ddl(&self, sql: &str) -> Result<()>;

    // --- Connection state ---
    fn is_alive(&self) -> bool;
    fn server_version(&self) -> Option<String>;
    fn current_database(&self) -> Option<String>;
    fn current_schema(&self) -> Option<String>;
}

/// Registry of available drivers
pub struct DriverRegistry {
    drivers: HashMap<DriverType, Arc<dyn DatabaseDriver>>,
}

impl DriverRegistry {
    pub fn new() -> Self {
        let mut registry = Self { drivers: HashMap::new() };
        registry.register(Arc::new(SqliteDriver));
        // Other drivers registered in later phases
        registry
    }

    pub fn register(&mut self, driver: Arc<dyn DatabaseDriver>) {
        self.drivers.insert(driver.driver_type(), driver);
    }

    pub fn get(&self, driver_type: &DriverType) -> Result<Arc<dyn DatabaseDriver>> {
        self.drivers.get(driver_type)
            .cloned()
            .ok_or_else(|| anyhow::anyhow!("Driver not found: {:?}", driver_type))
    }
}
```

```rust
// crates/database_core/src/drivers/sqlite.rs

pub struct SqliteDriver;

#[async_trait]
impl DatabaseDriver for SqliteDriver {
    fn driver_type(&self) -> DriverType { DriverType::Sqlite }

    fn supported_types(&self) -> Vec<DataType> {
        vec![
            DataType::Integer, DataType::Float, DataType::Text,
            DataType::Blob, DataType::Boolean,
        ]
    }

    async fn connect(&self, config: &ConnectionConfig) -> Result<Box<dyn DatabaseConnection>> {
        let path = config.sqlite_path.as_ref()
            .ok_or_else(|| anyhow::anyhow!("SQLite requires a file path"))?;
        let connection = rusqlite::Connection::open(path)?;
        Ok(Box::new(SqliteConnection { connection: Mutex::new(connection) }))
    }

    fn validate_config(&self, config: &ConnectionConfig) -> Result<()> {
        if config.sqlite_path.is_none() {
            return Err(anyhow::anyhow!("SQLite path is required"));
        }
        Ok(())
    }
}

pub struct SqliteConnection {
    connection: Mutex<rusqlite::Connection>,
}

#[async_trait]
impl DatabaseConnection for SqliteConnection {
    async fn execute_query(&self, sql: &str) -> Result<QueryResult> {
        let conn = self.connection.lock();
        let mut stmt = conn.prepare(sql)?;
        let column_count = stmt.column_count();
        let columns: Vec<ColumnInfo> = (0..column_count)
            .map(|i| ColumnInfo {
                name: stmt.column_name(i).unwrap_or("?").to_string(),
                data_type: stmt.column_decltype(i)
                    .unwrap_or_default()
                    .to_string(),
                normalized_type: sqlite_type_to_data_type(
                    stmt.column_decltype(i).unwrap_or_default()
                ),
                is_nullable: true,
                is_primary_key: false,
                is_foreign_key: false,
                default_value: None,
                ordinal_position: i,
                foreign_key_ref: None,
            })
            .collect();

        let started_at = Instant::now();
        let rows = stmt.query_map([], |row| {
            let values: Vec<CellValue> = (0..column_count)
                .map(|i| sqlite_value_to_cell(row, i))
                .collect();
            Ok(values)
        })?
        .collect::<Result<Vec<_>, _>>()?;

        Ok(QueryResult {
            columns,
            rows_affected: None,
            execution_time: started_at.elapsed(),
            has_more_rows: false,
            total_row_count: Some(rows.len() as u64),
            warnings: vec![],
            rows,
        })
    }

    async fn execute_statement(&self, sql: &str) -> Result<u64> {
        let conn = self.connection.lock();
        let affected = conn.execute(sql, [])?;
        Ok(affected as u64)
    }

    async fn cancel(&self) -> Result<()> {
        self.connection.lock().interrupt();
        Ok(())
    }

    async fn introspect_names(&self) -> Result<Vec<SchemaObject>> {
        let conn = self.connection.lock();
        let mut stmt = conn.prepare(
            "SELECT type, name FROM sqlite_master WHERE type IN ('table', 'view') ORDER BY name"
        )?;
        let objects = stmt.query_map([], |row| {
            let obj_type: String = row.get(0)?;
            let name: String = row.get(1)?;
            Ok(SchemaObject {
                object_type: match obj_type.as_str() {
                    "table" => SchemaObjectType::Table,
                    "view" => SchemaObjectType::View,
                    _ => SchemaObjectType::Table,
                },
                catalog: None,
                schema: None,
                name,
            })
        })?
        .collect::<Result<Vec<_>, _>>()?;
        Ok(objects)
    }

    async fn introspect_columns(&self, table: &TableRef) -> Result<Vec<ColumnInfo>> {
        let conn = self.connection.lock();
        let mut stmt = conn.prepare(&format!("PRAGMA table_info('{}')", table.name))?;
        let columns = stmt.query_map([], |row| {
            let name: String = row.get(1)?;
            let type_name: String = row.get(2)?;
            let not_null: bool = row.get(3)?;
            let default_value: Option<String> = row.get(4)?;
            let is_pk: bool = row.get(5)?;
            Ok(ColumnInfo {
                ordinal_position: row.get::<_, usize>(0)?,
                name,
                data_type: type_name.clone(),
                normalized_type: sqlite_type_to_data_type(&type_name),
                is_nullable: !not_null,
                is_primary_key: is_pk,
                is_foreign_key: false,
                default_value,
                foreign_key_ref: None,
            })
        })?
        .collect::<Result<Vec<_>, _>>()?;
        Ok(columns)
    }

    async fn introspect_ddl(&self, object: &SchemaObject) -> Result<String> {
        let conn = self.connection.lock();
        let sql = conn.query_row(
            "SELECT sql FROM sqlite_master WHERE name = ?",
            [&object.name],
            |row| row.get::<_, String>(0),
        )?;
        Ok(sql)
    }

    async fn foreign_keys(&self, table: &TableRef) -> Result<Vec<ForeignKey>> {
        let conn = self.connection.lock();
        let mut stmt = conn.prepare(&format!("PRAGMA foreign_key_list('{}')", table.name))?;
        let fks = stmt.query_map([], |row| {
            Ok(ForeignKey {
                name: format!("fk_{}", row.get::<_, usize>(0)?),
                columns: vec![row.get::<_, String>(3)?],
                referenced_table: TableRef {
                    catalog: None,
                    schema: None,
                    name: row.get::<_, String>(2)?,
                },
                referenced_columns: vec![row.get::<_, String>(4)?],
                on_update: parse_fk_action(row.get::<_, String>(5)?),
                on_delete: parse_fk_action(row.get::<_, String>(6)?),
            })
        })?
        .collect::<Result<Vec<_>, _>>()?;
        Ok(fks)
    }

    async fn indexes(&self, table: &TableRef) -> Result<Vec<IndexInfo>> {
        let conn = self.connection.lock();
        let mut stmt = conn.prepare(&format!("PRAGMA index_list('{}')", table.name))?;
        let indexes = stmt.query_map([], |row| {
            Ok(IndexInfo {
                name: row.get::<_, String>(1)?,
                columns: vec![], // Filled by PRAGMA index_info
                is_unique: row.get::<_, bool>(2)?,
                is_primary: row.get::<_, String>(3)? == "pk",
                index_type: "btree".to_string(),
            })
        })?
        .collect::<Result<Vec<_>, _>>()?;
        Ok(indexes)
    }

    async fn explain(&self, sql: &str) -> Result<ExplainPlan> {
        let conn = self.connection.lock();
        let mut stmt = conn.prepare(&format!("EXPLAIN QUERY PLAN {}", sql))?;
        let mut raw_lines = Vec::new();
        stmt.query_map([], |row| {
            let detail: String = row.get(3)?;
            raw_lines.push(detail);
            Ok(())
        })?.collect::<Result<Vec<_>, _>>()?;

        Ok(ExplainPlan {
            raw_text: raw_lines.join("\n"),
            nodes: vec![], // Parsed in a later step
            total_cost: None,
        })
    }

    async fn execute_ddl(&self, sql: &str) -> Result<()> {
        let conn = self.connection.lock();
        conn.execute_batch(sql)?;
        Ok(())
    }

    fn is_alive(&self) -> bool { true }
    fn server_version(&self) -> Option<String> { Some(rusqlite::version().to_string()) }
    fn current_database(&self) -> Option<String> { Some("main".to_string()) }
    fn current_schema(&self) -> Option<String> { None }
}

/// Map SQLite type declarations to normalized DataType
fn sqlite_type_to_data_type(type_name: &str) -> DataType {
    let upper = type_name.to_uppercase();
    match upper.as_str() {
        "INTEGER" | "INT" | "BIGINT" | "SMALLINT" | "TINYINT" => DataType::Integer,
        "REAL" | "DOUBLE" | "FLOAT" => DataType::Double,
        "TEXT" | "VARCHAR" | "CHAR" | "CLOB" => DataType::Text,
        "BLOB" => DataType::Blob,
        "BOOLEAN" | "BOOL" => DataType::Boolean,
        "DATE" => DataType::Date,
        "DATETIME" | "TIMESTAMP" => DataType::Timestamp,
        "JSON" => DataType::Json,
        _ => DataType::Other(type_name.to_string()),
    }
}
```

### 1.2 ConnectionManager entity with pooling

**Steps:**
1. Create `crates/database_core/src/connection_pool.rs`
2. Define `ConnectionManager` as a GPUI `Entity`
3. Implement connect/disconnect lifecycle
4. Implement connection pooling with idle timeout
5. Emit events on connection state changes

```rust
// crates/database_core/src/connection_pool.rs

pub struct ConnectionManager {
    driver_registry: Arc<DriverRegistry>,
    connections: HashMap<ConnectionId, Entity<DatabaseConnectionState>>,
    active_connection: Option<ConnectionId>,
    credential_cache: HashMap<ConnectionId, String>,
    _subscriptions: Vec<Subscription>,
}

pub struct DatabaseConnectionState {
    pub config: ConnectionConfig,
    pub status: ConnectionStatus,
    pub connection: Option<Box<dyn DatabaseConnection>>,
    pub schema_cache: SchemaCache,
    pub watchdog_task: Option<Task<()>>,
}

#[derive(Clone, Debug, PartialEq)]
pub enum ConnectionStatus {
    Disconnected,
    Connecting,
    Connected { since: Instant, server_version: Option<String> },
    Reconnecting { attempt: usize, next_retry: Instant },
    Failed(String),
}

impl EventEmitter<ConnectionEvent> for ConnectionManager {}

pub enum ConnectionEvent {
    Connected(ConnectionId),
    Disconnected(ConnectionId),
    ConnectionFailed { id: ConnectionId, error: String },
    SchemaRefreshed(ConnectionId),
    ActiveConnectionChanged(Option<ConnectionId>),
}

impl ConnectionManager {
    pub fn new(driver_registry: Arc<DriverRegistry>, cx: &mut Context<Self>) -> Self {
        Self {
            driver_registry,
            connections: HashMap::new(),
            active_connection: None,
            credential_cache: HashMap::new(),
            _subscriptions: vec![],
        }
    }

    /// Connect to a data source. Returns the connection id.
    pub fn connect(
        &mut self,
        config: ConnectionConfig,
        cx: &mut Context<Self>,
    ) -> Task<Result<ConnectionId>> {
        let connection_id = ConnectionId(config.id.clone());
        let driver = self.driver_registry.get(&config.driver);

        cx.spawn({
            let connection_id = connection_id.clone();
            async move |this, cx| {
                let driver = driver?;
                let connection = driver.connect(&config).await?;

                this.update(cx, |this, cx| {
                    let state = cx.new(|_| DatabaseConnectionState {
                        config: config.clone(),
                        status: ConnectionStatus::Connected {
                            since: Instant::now(),
                            server_version: connection.server_version(),
                        },
                        connection: Some(connection),
                        schema_cache: SchemaCache::new(),
                        watchdog_task: None,
                    });
                    this.connections.insert(connection_id.clone(), state);
                    this.active_connection = Some(connection_id.clone());
                    cx.emit(ConnectionEvent::Connected(connection_id.clone()));
                    cx.notify();
                })?;

                Ok(connection_id)
            }
        })
    }

    /// Disconnect from a data source
    pub fn disconnect(&mut self, id: &ConnectionId, cx: &mut Context<Self>) {
        if let Some(state) = self.connections.remove(id) {
            state.update(cx, |state, _| {
                state.connection = None;
                state.status = ConnectionStatus::Disconnected;
                state.watchdog_task = None;
            });
        }
        if self.active_connection.as_ref() == Some(id) {
            self.active_connection = self.connections.keys().next().cloned();
        }
        cx.emit(ConnectionEvent::Disconnected(id.clone()));
        cx.notify();
    }

    /// Get the active connection for executing queries
    pub fn active_connection(&self) -> Option<&Entity<DatabaseConnectionState>> {
        self.active_connection.as_ref()
            .and_then(|id| self.connections.get(id))
    }

    /// Set active connection
    pub fn set_active(&mut self, id: ConnectionId, cx: &mut Context<Self>) {
        self.active_connection = Some(id.clone());
        cx.emit(ConnectionEvent::ActiveConnectionChanged(Some(id)));
        cx.notify();
    }

    /// Execute a query on a specific connection
    pub fn execute_query(
        &self,
        connection_id: &ConnectionId,
        sql: String,
        cx: &mut Context<Self>,
    ) -> Task<Result<QueryResult>> {
        let state = self.connections.get(connection_id).cloned();
        cx.background_spawn(async move {
            let state = state.ok_or_else(|| anyhow::anyhow!("Connection not found"))?;
            // Read connection from state (needs careful borrow management)
            // Execute query on background thread
            todo!()
        })
    }

    /// List all connections
    pub fn connections(&self) -> Vec<(ConnectionId, ConnectionStatus)> {
        self.connections.iter().map(|(id, state)| {
            // Read status from state
            (id.clone(), ConnectionStatus::Disconnected) // placeholder
        }).collect()
    }
}
```

### 1.3 SSH tunnel support

Use `russh` or `async-ssh2-lite` for SSH tunneling. Configuration:

**Steps:**
1. Create `crates/database_core/src/ssh_tunnel.rs`
2. Implement SSH tunnel lifecycle (open, monitor, close)
3. Support password, private key, and SSH agent auth
4. Integrate with `ConnectionManager` connect flow

```rust
// crates/database_core/src/ssh_tunnel.rs

pub struct SshTunnelConfig {
    pub host: String,
    pub port: u16,
    pub username: String,
    pub auth: SshAuth,            // Password, PrivateKey, or Agent
    pub local_port: Option<u16>,  // Auto-assign if None
}

pub enum SshAuth {
    Password(String),
    PrivateKey { path: PathBuf, passphrase: Option<String> },
    Agent,
}

pub struct SshTunnel {
    config: SshTunnelConfig,
    local_port: u16,
    session: Option<russh::client::Handle<SshHandler>>,
    monitor_task: Option<Task<()>>,
}

impl SshTunnel {
    /// Open the SSH tunnel and return the local port to connect through
    pub async fn open(config: SshTunnelConfig, remote_host: &str, remote_port: u16) -> Result<Self> {
        // 1. Connect to SSH server
        // 2. Authenticate
        // 3. Open port forwarding (local_port -> remote_host:remote_port)
        // 4. Start health monitor task
        todo!()
    }

    pub fn local_port(&self) -> u16 { self.local_port }

    pub async fn close(&mut self) -> Result<()> {
        self.monitor_task = None;
        if let Some(session) = self.session.take() {
            session.disconnect(russh::Disconnect::ByApplication, "", "en").await?;
        }
        Ok(())
    }
}
```

### 1.4 SSL/TLS configuration

Certificate management: CA cert, client cert, verify-full vs verify-ca mode.

**Steps:**
1. Add `SslConfig` to `crates/database_core/src/schema.rs`
2. Implement TLS connector building per driver
3. File picker integration in connection dialog for cert paths

```rust
// Added to crates/database_core/src/schema.rs

#[derive(Clone, Debug, Serialize, Deserialize, JsonSchema)]
pub struct SslConfig {
    pub mode: SslMode,
    pub ca_cert_path: Option<PathBuf>,
    pub client_cert_path: Option<PathBuf>,
    pub client_key_path: Option<PathBuf>,
}

#[derive(Clone, Debug, Serialize, Deserialize, JsonSchema)]
pub enum SslMode {
    Disable,
    Prefer,
    Require,
    VerifyCa,
    VerifyFull,
}
```

### 1.5 Connection dialog UI

`Entity<ConnectionDialog>` — modal dialog for configuring a data source:
host, port, database, user, password, SSL, SSH, test connection button.

**Steps:**
1. Create `crates/database_ui/src/connection_dialog.rs`
2. Implement `ModalView` + `EventEmitter<DismissEvent>` + `Focusable` + `Render`
3. Create form layout with `Entity<Editor>` fields
4. Implement driver tab switching
5. Implement test connection flow with spinner
6. Implement save/cancel actions
7. Wire to `ConnectionManager::connect()`

(See detailed render layout and entity structure in the
[UI/Interface Specification — Section 3](#3-connection-dialog).)

**Color coding per connection** — Each connection can be assigned a color
(red for production, green for dev, blue for staging). This color tints:
- The Database Explorer header for that connection
- Query editor tab borders
- Result grid tab borders
- Status bar indicator

This prevents accidentally running queries on production.

### 1.6 Database Explorer panel

Tree panel registered as a Zed Panel with `activation_priority: 15`.

**Steps:**
1. Create `crates/database_ui/src/database_explorer.rs`
2. Implement `Panel` + `Focusable` + `EventEmitter<PanelEvent>` + `Render`
3. Implement tree node model (`TreeNode` enum with variants per object type)
4. Implement tree rendering with `uniform_list` + `ListItem`
5. Implement expand/collapse with lazy loading via `ConnectionManager`
6. Implement context menus per node type
7. Implement drag-and-drop for tables/columns
8. Implement fuzzy filter
9. Register panel in `initialize_panels()`

(See detailed entity structure, tree hierarchy, and rendering pattern in the
[UI/Interface Specification — Section 2](#2-database-explorer-panel).)

```rust
// crates/database_ui/src/database_explorer.rs

/// Tree node model for the explorer hierarchy
pub enum TreeNode {
    Connection {
        id: ConnectionId,
        name: String,
        color_index: usize,
        status: ConnectionStatus,
        children: Vec<TreeNode>,
        is_expanded: bool,
    },
    Database {
        name: String,
        children: Vec<TreeNode>,
        is_expanded: bool,
    },
    Schema {
        name: String,
        children: Vec<TreeNode>,
        is_expanded: bool,
    },
    Category {
        kind: CategoryKind,
        children: Vec<TreeNode>,
        count: usize,
        is_expanded: bool,
    },
    Table {
        table_ref: TableRef,
        children: Vec<TreeNode>,  // Columns, loaded on expand
        is_expanded: bool,
    },
    View {
        table_ref: TableRef,
        is_expanded: bool,
    },
    Function {
        name: String,
        signature: Option<String>,
    },
    Sequence {
        name: String,
    },
    Column {
        info: ColumnInfo,
    },
    Loading,  // Placeholder while loading children
}

#[derive(Clone, Debug, PartialEq)]
pub enum CategoryKind { Tables, Views, Functions, Sequences }

impl TreeNode {
    pub fn id(&self) -> ElementId { /* unique ID from path */ todo!() }
    pub fn depth(&self) -> usize { /* computed from parent chain */ todo!() }
    pub fn has_children(&self) -> bool { /* depends on variant */ todo!() }
    pub fn is_expanded(&self) -> bool { /* depends on variant */ todo!() }
    pub fn icon_name(&self) -> IconName { /* per variant */ todo!() }
    pub fn icon_color(&self) -> Color { /* per variant */ todo!() }
    pub fn display_name(&self) -> SharedString { /* per variant */ todo!() }
    pub fn connection_color(&self) -> Option<Hsla> { /* from connection ancestor */ todo!() }
}

impl Panel for DatabaseExplorer {
    fn persistent_name() -> &'static str { "DatabaseExplorer" }
    fn panel_key() -> &'static str { "DatabaseExplorer" }
    fn position(&self, _window: &Window, _cx: &App) -> DockPosition { self.position }
    fn position_is_valid(&self, position: DockPosition) -> bool {
        matches!(position, DockPosition::Left | DockPosition::Right)
    }
    fn set_position(&mut self, position: DockPosition, _window: &mut Window, cx: &mut Context<Self>) {
        self.position = position;
        cx.notify();
    }
    fn size(&self, _window: &Window, _cx: &App) -> Pixels { self.width.unwrap_or(px(260.)) }
    fn set_size(&mut self, size: Option<Pixels>, _window: &mut Window, cx: &mut Context<Self>) {
        self.width = size;
        cx.notify();
    }
    fn icon(&self, _window: &Window, _cx: &App) -> Option<IconName> {
        Some(IconName::DatabaseZap)
    }
    fn icon_tooltip(&self, _window: &Window, _cx: &App) -> Option<&'static str> {
        Some("Database Explorer")
    }
    fn toggle_action(&self) -> Box<dyn Action> {
        Box::new(ToggleDatabaseExplorer)
    }
    fn activation_priority(&self) -> u32 { 15 }
    fn starts_open(&self, _window: &Window, _cx: &App) -> bool { false }
}

impl Focusable for DatabaseExplorer {
    fn focus_handle(&self, _cx: &App) -> FocusHandle { self.focus_handle.clone() }
}

impl EventEmitter<PanelEvent> for DatabaseExplorer {}
```

Features:
- Hierarchical tree: Data Source > Database > Schema > Tables/Views/Functions/Sequences
- Each node shows icon + name + object count badge
- Double-click table opens data grid; double-click view opens DDL
- **Object filtering by pattern** — Right-click a schema node >
  "Filter Objects" > enter regex (e.g., `^(?!_tmp).*`) to hide matching objects

### 1.7 Introspection by levels with cache

- Level 1: Object names only (instant, for large databases)
- Level 2: Columns, types, constraints (default)
- Level 3: Full DDL source code

Smart refresh: after DDL execution, only re-introspect affected objects.
Schema cache stored in-memory with `Arc<RwLock<SchemaCache>>`.

**Steps:**
1. Create `crates/database_core/src/introspection.rs`
2. Implement `SchemaCache` with TTL and invalidation
3. Implement progressive loading (Level 1 on connect, Level 2 on expand)
4. Implement smart refresh (invalidate only changed objects after DDL)

```rust
// crates/database_core/src/introspection.rs

pub struct SchemaCache {
    /// Level 1: Object names
    objects: HashMap<String, Vec<SchemaObject>>,  // schema_name -> objects
    /// Level 2: Column metadata per table
    columns: HashMap<TableRef, Vec<ColumnInfo>>,
    /// Level 3: DDL source
    ddl: HashMap<SchemaObject, String>,
    /// Timestamps for cache invalidation
    last_refreshed: HashMap<String, Instant>,
    /// TTL for cache entries
    ttl: Duration,
}

impl SchemaCache {
    pub fn new() -> Self {
        Self {
            objects: HashMap::new(),
            columns: HashMap::new(),
            ddl: HashMap::new(),
            last_refreshed: HashMap::new(),
            ttl: Duration::from_secs(300), // 5 min default
        }
    }

    pub fn get_objects(&self, schema: &str) -> Option<&Vec<SchemaObject>> {
        if self.is_stale(schema) { return None; }
        self.objects.get(schema)
    }

    pub fn get_columns(&self, table: &TableRef) -> Option<&Vec<ColumnInfo>> {
        self.columns.get(table)
    }

    pub fn get_ddl(&self, object: &SchemaObject) -> Option<&String> {
        self.ddl.get(object)
    }

    pub fn set_objects(&mut self, schema: String, objects: Vec<SchemaObject>) {
        self.last_refreshed.insert(schema.clone(), Instant::now());
        self.objects.insert(schema, objects);
    }

    pub fn set_columns(&mut self, table: TableRef, columns: Vec<ColumnInfo>) {
        self.columns.insert(table, columns);
    }

    /// Invalidate a specific object (after DDL execution)
    pub fn invalidate(&mut self, object: &SchemaObject) {
        let table_ref = TableRef {
            catalog: object.catalog.clone(),
            schema: object.schema.clone(),
            name: object.name.clone(),
        };
        self.columns.remove(&table_ref);
        self.ddl.remove(object);
        // Don't remove from objects — will be refreshed on next access
        if let Some(schema) = &object.schema {
            self.last_refreshed.remove(schema);
        }
    }

    /// Invalidate all cache for a schema
    pub fn invalidate_schema(&mut self, schema: &str) {
        self.objects.remove(schema);
        self.last_refreshed.remove(schema);
        self.columns.retain(|table_ref, _| table_ref.schema.as_deref() != Some(schema));
    }

    fn is_stale(&self, schema: &str) -> bool {
        self.last_refreshed.get(schema)
            .map_or(true, |t| t.elapsed() > self.ttl)
    }
}
```

### 1.8 Workspace serialization

Persist panel state across Zed restarts using KVP store (same pattern as
ProjectPanel).

**Steps:**
1. Define `SerializedDatabasePanel` with all state to persist
2. Implement `SerializableItem` (if using Item) or custom KVP serialization
3. Save on panel close / workspace deactivate
4. Restore on workspace open

```rust
// crates/database_ui/src/database_explorer.rs (serialization section)

#[derive(Serialize, Deserialize)]
struct SerializedDatabasePanel {
    width: Option<f32>,
    position: DockPosition,
    active_connection_id: Option<String>,
    expanded_nodes: Vec<String>,         // Serialized paths like "conn/db/schema/table"
    open_query_tabs: Vec<SerializedQueryTab>,
    pinned_result_tabs: Vec<SerializedResultTab>,
}

#[derive(Serialize, Deserialize)]
struct SerializedQueryTab {
    connection_id: String,
    schema: Option<String>,
    sql_content: String,
    tab_name: Option<String>,
    split_ratio: f32,
    is_pinned: bool,
}

#[derive(Serialize, Deserialize)]
struct SerializedResultTab {
    connection_id: String,
    sql: String,
    tab_name: String,
}

impl DatabaseExplorer {
    fn serialize(&self, cx: &App) -> SerializedDatabasePanel {
        SerializedDatabasePanel {
            width: self.width.map(|px| px.0),
            position: self.position,
            active_connection_id: self.active_connection_id().map(|id| id.0.clone()),
            expanded_nodes: self.collect_expanded_paths(),
            open_query_tabs: vec![], // Collected from workspace pane items
            pinned_result_tabs: vec![],
        }
    }

    fn deserialize(
        serialized: SerializedDatabasePanel,
        connection_manager: Entity<ConnectionManager>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Self {
        let mut explorer = Self::new(connection_manager, window, cx);
        explorer.width = serialized.width.map(px);
        explorer.position = serialized.position;
        // Restore expanded state after connections are re-established
        explorer.pending_restore = Some(serialized);
        explorer
    }
}
```

### 1.9 Auto-reconnection with backoff

`ConnectionWatchdog` monitors connection health with periodic pings.
On connection loss: notify user, retry with exponential backoff,
preserve pending changes if in a transaction.

**Steps:**
1. Create watchdog task in `DatabaseConnectionState`
2. Implement health check (ping) per driver
3. Implement exponential backoff (1s, 2s, 4s, 8s, max 30s)
4. Emit `ConnectionEvent::ConnectionFailed` / `ConnectionEvent::Connected` on state changes

```rust
// crates/database_core/src/connection_pool.rs (watchdog section)

impl DatabaseConnectionState {
    /// Start a watchdog task that monitors connection health
    fn start_watchdog(&mut self, cx: &mut Context<Self>) {
        self.watchdog_task = Some(cx.spawn(async move |this, cx| {
            let ping_interval = Duration::from_secs(30);
            let mut backoff = ExponentialBackoff::new(
                Duration::from_secs(1),
                Duration::from_secs(30),
                2.0,
            );

            loop {
                cx.background_executor().timer(ping_interval).await;

                let is_alive = this.update(cx, |state, _| {
                    state.connection.as_ref().map_or(false, |c| c.is_alive())
                }).unwrap_or(false);

                if !is_alive {
                    // Try to reconnect with exponential backoff
                    loop {
                        let delay = backoff.next_delay();
                        cx.background_executor().timer(delay).await;

                        match this.update(cx, |state, cx| {
                            state.attempt_reconnect(cx)
                        }) {
                            Ok(task) => match task.await {
                                Ok(()) => {
                                    backoff.reset();
                                    break; // Successfully reconnected
                                }
                                Err(_) => continue, // Retry
                            },
                            Err(_) => return, // Entity dropped
                        }
                    }
                }
            }
        }));
    }
}

pub struct ExponentialBackoff {
    initial: Duration,
    max: Duration,
    factor: f64,
    current: Duration,
}

impl ExponentialBackoff {
    pub fn new(initial: Duration, max: Duration, factor: f64) -> Self {
        Self { initial, max, factor, current: initial }
    }

    pub fn next_delay(&mut self) -> Duration {
        let delay = self.current;
        self.current = Duration::from_secs_f64(
            (self.current.as_secs_f64() * self.factor).min(self.max.as_secs_f64())
        );
        delay
    }

    pub fn reset(&mut self) { self.current = self.initial; }
}
```

### 1.10 Read-only mode per connection

Toggle per data source that prevents any INSERT/UPDATE/DELETE/DDL.
Visual indicator in the connection's color bar.

**Steps:**
1. Add `read_only: bool` to `ConnectionConfig` (already in 0.1)
2. Implement write guard in `ConnectionManager::execute_statement()`
3. Add lock icon to explorer node, tab, and status bar when read-only
4. Allow toggling via connection context menu

```rust
// In ConnectionManager
pub fn execute_statement(
    &self,
    connection_id: &ConnectionId,
    sql: String,
    cx: &mut Context<Self>,
) -> Task<Result<u64>> {
    let state = self.connections.get(connection_id).cloned();
    cx.background_spawn(async move {
        let state = state.ok_or_else(|| anyhow::anyhow!("Connection not found"))?;
        // Check read-only guard
        if state.read(cx).config.read_only {
            return Err(DatabaseError::ReadOnlyConnection.into());
        }
        // Execute on background thread
        todo!()
    })
}
```

---

## Phase 2 — Query Editor & Execution

### 2.1 SQL buffer with dialect detection

Specialized buffer with SQL language activated, bound to a connection/schema.
Uses Zed's standard editor infrastructure (multicursor, vim mode, etc.).

**Steps:**
1. Create `crates/database_ui/src/query_editor.rs`
2. Implement `Item` + `Focusable` + `EventEmitter<QueryEditorEvent>` + `Render`
3. Wrap `Entity<Editor>` with SQL language mode
4. Implement vertical split layout with ResultGrid
5. Implement `-- @name` comment parsing for tab naming
6. Register toolbar items

(See detailed entity structure and render implementation in the
[UI/Interface Specification — Section 4](#4-query-editor-workspace-item).)

```rust
// crates/database_ui/src/query_editor.rs

pub struct QueryEditor {
    editor: Entity<Editor>,
    connection_manager: Entity<ConnectionManager>,
    connection_id: Option<ConnectionId>,
    schema: Option<String>,
    result_grid: Option<Entity<ResultGrid>>,
    split_state: Entity<SplitState>,
    execution_state: ExecutionState,
    execution_task: Option<Task<()>>,
    tab_name: Option<String>,
    query_counter: usize,           // For auto-naming "Query N"
    focus_handle: FocusHandle,
    _subscriptions: Vec<Subscription>,
}

pub enum QueryEditorEvent {
    ExecutionStarted,
    ExecutionCompleted { duration: Duration, row_count: usize },
    ExecutionFailed(DatabaseError),
    ConnectionChanged(Option<ConnectionId>),
    Edited,
}

impl EventEmitter<QueryEditorEvent> for QueryEditor {}

impl QueryEditor {
    pub fn new(
        connection_manager: Entity<ConnectionManager>,
        connection_id: Option<ConnectionId>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Self {
        let editor = cx.new(|cx| {
            let mut editor = Editor::multi_line(window, cx);
            // Set SQL language via language registry
            // editor.set_language(sql_language, cx);
            editor
        });

        let split_state = cx.new(|_| SplitState {
            ratio: 0.5,
            visible_ratio: 0.5,
            cached_height: px(0.),
            is_dragging: false,
        });

        let _subscriptions = vec![
            cx.subscribe(&editor, Self::on_editor_event),
        ];

        Self {
            editor,
            connection_manager,
            connection_id,
            schema: None,
            result_grid: None,
            split_state,
            execution_state: ExecutionState::Idle,
            execution_task: None,
            tab_name: None,
            query_counter: 0,
            focus_handle: cx.focus_handle(),
            _subscriptions,
        }
    }

    /// Execute the current query (or selected text if any)
    pub fn execute(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        let connection_id = match &self.connection_id {
            Some(id) => id.clone(),
            None => {
                // Show connection picker
                return;
            }
        };

        let sql = self.get_sql_to_execute(cx);
        if sql.trim().is_empty() { return; }

        self.execution_state = ExecutionState::Executing {
            task: Task::ready(()),
            started_at: Instant::now(),
        };
        cx.emit(QueryEditorEvent::ExecutionStarted);
        cx.notify();

        let connection_manager = self.connection_manager.clone();
        self.execution_task = Some(cx.spawn({
            let connection_id = connection_id.clone();
            async move |this, cx| {
                let result = connection_manager.update(cx, |manager, cx| {
                    manager.execute_query(&connection_id, sql.clone(), cx)
                })?.await;

                this.update(cx, |this, cx| {
                    match result {
                        Ok(query_result) => {
                            this.execution_state = ExecutionState::Completed {
                                duration: query_result.execution_time,
                                row_count: query_result.rows.len(),
                            };

                            // Create or update the result grid
                            if let Some(grid) = &this.result_grid {
                                grid.update(cx, |grid, cx| {
                                    grid.set_data(query_result, cx);
                                });
                            } else {
                                let grid = cx.new(|cx| {
                                    ResultGrid::new(query_result, cx)
                                });
                                this.result_grid = Some(grid);
                            }

                            cx.emit(QueryEditorEvent::ExecutionCompleted {
                                duration: query_result.execution_time,
                                row_count: query_result.rows.len(),
                            });

                            // Save to query history
                            this.save_to_history(&sql, &query_result, cx);
                        }
                        Err(error) => {
                            let db_error = DatabaseError::from(error);
                            this.execution_state = ExecutionState::Failed {
                                error: db_error.clone(),
                                duration: Duration::default(),
                            };
                            cx.emit(QueryEditorEvent::ExecutionFailed(db_error));
                        }
                    }
                    cx.notify();
                }).log_err();
            }
        }));
    }

    /// Cancel running query
    pub fn cancel(&mut self, cx: &mut Context<Self>) {
        self.execution_task = None;
        // Also send cancel to the server via ConnectionManager
        if let Some(connection_id) = &self.connection_id {
            // connection_manager.cancel_query(connection_id, cx);
        }
        self.execution_state = ExecutionState::Idle;
        cx.notify();
    }

    /// Get the SQL to execute: selected text if any, otherwise full buffer
    fn get_sql_to_execute(&self, cx: &App) -> String {
        let editor = self.editor.read(cx);
        let selections = editor.selections.all::<usize>(cx);
        if selections.len() == 1 && !selections[0].is_empty() {
            let range = selections[0].range();
            editor.text_for_range(range, cx).collect::<String>()
        } else {
            editor.text(cx).to_string()
        }
    }

    /// Parse `-- @name MyQuery` from first 5 lines
    fn parse_tab_name(&self, cx: &App) -> Option<String> {
        let text = self.editor.read(cx).text(cx);
        let prefix = "-- @name ";
        for line in text.lines().take(5) {
            let trimmed = line.trim();
            if let Some(name) = trimmed.strip_prefix(prefix) {
                return Some(name.trim().to_string());
            }
        }
        None
    }

    fn tab_name(&self, cx: &App) -> SharedString {
        if let Some(name) = self.parse_tab_name(cx) {
            return name.into();
        }
        if let Some(name) = &self.tab_name {
            return name.clone().into();
        }
        format!("Query {}", self.query_counter).into()
    }

    fn connection_color(&self, cx: &App) -> Option<Hsla> {
        self.connection_id.as_ref().and_then(|id| {
            self.connection_manager.read(cx)
                .connection_config(id)
                .map(|config| CONNECTION_COLORS[config.color_index % CONNECTION_COLORS.len()])
        })
    }

    fn on_editor_event(
        &mut self,
        _editor: &Entity<Editor>,
        event: &EditorEvent,
        _window: &mut Window,
        cx: &mut Context<Self>,
    ) {
        match event {
            EditorEvent::Edited { .. } => {
                cx.emit(QueryEditorEvent::Edited);
            }
            _ => {}
        }
    }

    fn save_to_history(&self, sql: &str, result: &QueryResult, cx: &mut Context<Self>) {
        // Save to QueryHistory via KVP store
        todo!()
    }
}

// Item trait implementation: see UI Specification Section 4 for full details
```

### 2.2 SQL LSP integration

Integrate an SQL Language Server (`sqls` or `sql-language-server`) with
injected schema metadata for accurate completion.

**Steps:**
1. Add SQL language definition to Zed's language registry
2. Configure `sqls` or similar LSP for SQL completion
3. Inject schema metadata (table/column names) into LSP config
4. Wire connection switching to LSP config update

Features: completion, formatting, diagnostics (syntax errors, unknown tables).

```rust
// crates/database_ui/src/sql_language.rs

/// Configure SQL LSP with schema metadata from active connection
pub fn configure_sql_lsp(
    connection_id: &ConnectionId,
    schema_cache: &SchemaCache,
    cx: &mut App,
) -> LspAdapterConfig {
    // Build sqls configuration with connection and schema info
    // See: https://github.com/sqls-server/sqls
    let tables = schema_cache.all_tables()
        .map(|t| serde_json::json!({
            "name": t.name,
            "columns": schema_cache.get_columns(&t.to_table_ref())
                .unwrap_or(&vec![])
                .iter()
                .map(|c| serde_json::json!({
                    "columnName": c.name,
                    "dataType": c.data_type,
                }))
                .collect::<Vec<_>>(),
        }))
        .collect::<Vec<_>>();

    // Return LSP adapter config
    todo!()
}
```

### 2.3 Query execution (background_spawn)

Execute queries on `cx.background_spawn()`. Return results via channel to
foreground for UI update.

**Steps:**
1. Execute SQL on background thread via `cx.background_spawn()`
2. Stream results row-by-row for large result sets
3. Update foreground UI progressively
4. Handle cancellation via `Task` drop

```rust
// In ConnectionManager
pub fn execute_query_streaming(
    &self,
    connection_id: &ConnectionId,
    sql: String,
    page_size: usize,
    cx: &mut Context<Self>,
) -> Task<Result<QueryResult>> {
    let connection = self.get_connection(connection_id);

    cx.background_spawn(async move {
        let connection = connection?;

        // Add LIMIT/OFFSET for pagination
        let paginated_sql = if !sql.to_uppercase().contains("LIMIT") {
            format!("{} LIMIT {}", sql.trim_end_matches(';'), page_size)
        } else {
            sql
        };

        connection.execute_query(&paginated_sql).await
    })
}
```

### 2.4 Results in extended DataTable

Build on existing `crates/ui/src/components/data_table.rs`.

**Steps:**
1. Create `crates/database_ui/src/result_grid.rs`
2. Implement `Render` + `Focusable` + `EventEmitter<ResultGridEvent>`
3. Build on `Table::new().uniform_list().interactable().resizable_columns()`
4. Implement type-aware cell rendering (see UI Spec Section 5)
5. Implement selection model (`GridSelection` enum)
6. Implement inline editing flow
7. Implement pending changes tracking with color coding

(See detailed entity structure, rendering, and interaction model in the
[UI/Interface Specification — Section 5](#5-result-grid).)

```rust
// crates/database_ui/src/result_grid.rs

pub struct ResultGrid {
    columns: Vec<ColumnDef>,
    rows: Arc<Vec<Vec<CellValue>>>,
    total_row_count: Option<usize>,
    page: usize,
    page_size: usize,
    sort_state: Vec<SortColumn>,
    filters: Vec<ColumnFilter>,
    selection: GridSelection,
    pending_edits: IndexMap<CellAddress, PendingEdit>,
    edit_history: DataEditHistory,
    view_mode: ViewMode,
    inline_editor: Option<Entity<Editor>>,
    editing_cell: Option<(usize, usize)>,
    table_interaction_state: Entity<TableInteractionState>,
    column_widths: Entity<TableColumnWidths>,
    connection_color: Option<Hsla>,
    focus_handle: FocusHandle,
    _subscriptions: Vec<Subscription>,
}

#[derive(Clone, Debug)]
pub struct ColumnDef {
    pub info: ColumnInfo,
    pub visible: bool,
    pub display_format: Option<DisplayFormat>,
}

#[derive(Clone, Debug)]
pub struct SortColumn {
    pub column_index: usize,
    pub direction: SortDirection,
    pub priority: usize,
}

#[derive(Clone, Debug, PartialEq)]
pub enum SortDirection { Ascending, Descending }

#[derive(Clone, Debug)]
pub struct ColumnFilter {
    pub column_index: usize,
    pub filter_type: FilterType,
    pub value: String,
}

#[derive(Clone, Debug)]
pub enum FilterType {
    Contains,
    Equals,
    StartsWith,
    Regex,
    IsNull,
    IsNotNull,
    GreaterThan,
    LessThan,
}

#[derive(Clone, Debug, Hash, PartialEq, Eq)]
pub struct CellAddress {
    pub row: usize,
    pub col: usize,
}

pub struct PendingEdit {
    pub original: CellValue,
    pub current: CellValue,
    pub kind: EditKind,
}

#[derive(Clone, Debug, PartialEq)]
pub enum EditKind { Insert, Update, Delete }

impl EditKind {
    pub fn indicator_color(&self) -> Hsla {
        match self {
            EditKind::Insert => hsla(0.33, 0.7, 0.5, 1.0),  // Green
            EditKind::Update => hsla(0.14, 0.7, 0.5, 1.0),  // Yellow
            EditKind::Delete => hsla(0.0, 0.7, 0.5, 1.0),   // Red
        }
    }

    pub fn background_color(&self) -> Hsla {
        match self {
            EditKind::Insert => hsla(0.33, 0.3, 0.5, 0.12),
            EditKind::Update => hsla(0.14, 0.3, 0.5, 0.12),
            EditKind::Delete => hsla(0.0, 0.3, 0.5, 0.12),
        }
    }
}

pub enum ResultGridEvent {
    SelectionChanged(GridSelection),
    CellEdited { address: CellAddress, value: CellValue },
    PendingChangesUpdated { count: usize },
    CommitRequested,
    PageChanged(usize),
}

impl EventEmitter<ResultGridEvent> for ResultGrid {}

impl ResultGrid {
    pub fn new(query_result: QueryResult, cx: &mut Context<Self>) -> Self {
        let columns = query_result.columns.iter().map(|c| ColumnDef {
            info: c.clone(),
            visible: true,
            display_format: None,
        }).collect();

        Self {
            columns,
            rows: Arc::new(query_result.rows),
            total_row_count: query_result.total_row_count.map(|n| n as usize),
            page: 0,
            page_size: 500,
            sort_state: vec![],
            filters: vec![],
            selection: GridSelection::None,
            pending_edits: IndexMap::new(),
            edit_history: DataEditHistory::new(),
            view_mode: ViewMode::Grid,
            inline_editor: None,
            editing_cell: None,
            table_interaction_state: cx.new(|cx| TableInteractionState::new(cx)),
            column_widths: cx.new(|_| TableColumnWidths::default()),
            connection_color: None,
            focus_handle: cx.focus_handle(),
            _subscriptions: vec![],
        }
    }

    pub fn set_data(&mut self, query_result: QueryResult, cx: &mut Context<Self>) {
        self.columns = query_result.columns.iter().map(|c| ColumnDef {
            info: c.clone(),
            visible: true,
            display_format: None,
        }).collect();
        self.rows = Arc::new(query_result.rows);
        self.total_row_count = query_result.total_row_count.map(|n| n as usize);
        self.selection = GridSelection::None;
        self.pending_edits.clear();
        self.edit_history = DataEditHistory::new();
        cx.notify();
    }

    /// Start inline editing a cell
    pub fn start_editing(&mut self, row: usize, col: usize, window: &mut Window, cx: &mut Context<Self>) {
        let value = &self.rows[row][col];
        let text = value.display_string(&NumberFormatSettings::default());

        let editor = cx.new(|cx| {
            let mut editor = Editor::single_line(window, cx);
            editor.set_text(&text, window, cx);
            editor.select_all(&Default::default(), window, cx);
            editor
        });

        self.inline_editor = Some(editor);
        self.editing_cell = Some((row, col));
        cx.notify();
    }

    /// Commit inline edit
    pub fn commit_edit(&mut self, cx: &mut Context<Self>) {
        if let (Some((row, col)), Some(editor)) = (self.editing_cell, &self.inline_editor) {
            let text = editor.read(cx).text(cx).to_string();
            let new_value = self.parse_cell_value(&text, &self.columns[col].info);
            let original = self.rows[row][col].clone();

            if new_value != original {
                let operation = DataOperation::UpdateCell {
                    row, col,
                    old_value: original.clone(),
                    new_value: new_value.clone(),
                };
                self.edit_history.push(operation);
                self.pending_edits.insert(
                    CellAddress { row, col },
                    PendingEdit {
                        original,
                        current: new_value,
                        kind: EditKind::Update,
                    },
                );
                cx.emit(ResultGridEvent::PendingChangesUpdated {
                    count: self.pending_edits.len(),
                });
            }
        }
        self.inline_editor = None;
        self.editing_cell = None;
        cx.notify();
    }

    /// Cancel inline edit
    pub fn cancel_edit(&mut self, cx: &mut Context<Self>) {
        self.inline_editor = None;
        self.editing_cell = None;
        cx.notify();
    }

    /// Toggle sort on a column
    pub fn toggle_sort(&mut self, column_index: usize, multi: bool, cx: &mut Context<Self>) {
        if !multi {
            // Single column sort: cycle None -> ASC -> DESC -> None
            if let Some(existing) = self.sort_state.iter().position(|s| s.column_index == column_index) {
                match self.sort_state[existing].direction {
                    SortDirection::Ascending => {
                        self.sort_state[existing].direction = SortDirection::Descending;
                    }
                    SortDirection::Descending => {
                        self.sort_state.remove(existing);
                    }
                }
            } else {
                self.sort_state = vec![SortColumn {
                    column_index,
                    direction: SortDirection::Ascending,
                    priority: 1,
                }];
            }
        } else {
            // Multi-column: add to stack
            let priority = self.sort_state.len() + 1;
            if let Some(existing) = self.sort_state.iter().position(|s| s.column_index == column_index) {
                self.sort_state.remove(existing);
            } else {
                self.sort_state.push(SortColumn {
                    column_index,
                    direction: SortDirection::Ascending,
                    priority,
                });
            }
        }
        // Re-sort rows (client-side) or re-execute query (server-side)
        cx.notify();
    }

    /// Generate SQL DML for all pending changes
    pub fn generate_dml(&self, table_ref: &TableRef) -> Vec<String> {
        let mut statements = Vec::new();
        for (address, edit) in &self.pending_edits {
            match edit.kind {
                EditKind::Insert => {
                    // Generate INSERT statement
                    let columns: Vec<&str> = self.columns.iter()
                        .map(|c| c.info.name.as_str())
                        .collect();
                    let values: Vec<String> = self.rows[address.row].iter()
                        .map(|v| v.to_sql_literal())
                        .collect();
                    statements.push(format!(
                        "INSERT INTO {} ({}) VALUES ({})",
                        table_ref.qualified_name(),
                        columns.join(", "),
                        values.join(", "),
                    ));
                }
                EditKind::Update => {
                    // Generate UPDATE with WHERE on primary key
                    let pk_columns = self.primary_key_where_clause(address.row);
                    statements.push(format!(
                        "UPDATE {} SET {} = {} WHERE {}",
                        table_ref.qualified_name(),
                        self.columns[address.col].info.name,
                        edit.current.to_sql_literal(),
                        pk_columns,
                    ));
                }
                EditKind::Delete => {
                    let pk_columns = self.primary_key_where_clause(address.row);
                    statements.push(format!(
                        "DELETE FROM {} WHERE {}",
                        table_ref.qualified_name(),
                        pk_columns,
                    ));
                }
            }
        }
        statements
    }

    fn primary_key_where_clause(&self, row: usize) -> String {
        self.columns.iter().enumerate()
            .filter(|(_, c)| c.info.is_primary_key)
            .map(|(i, c)| format!("{} = {}", c.info.name, self.rows[row][i].to_sql_literal()))
            .collect::<Vec<_>>()
            .join(" AND ")
    }

    fn parse_cell_value(&self, text: &str, column: &ColumnInfo) -> CellValue {
        if text.eq_ignore_ascii_case("null") {
            return CellValue::Null;
        }
        match &column.normalized_type {
            DataType::Boolean => CellValue::Boolean(text.eq_ignore_ascii_case("true")),
            DataType::Integer | DataType::BigInt | DataType::SmallInt => {
                text.parse::<i64>().map(CellValue::Integer).unwrap_or(CellValue::String(text.to_string()))
            }
            DataType::Float | DataType::Double => {
                text.parse::<f64>().map(CellValue::Float).unwrap_or(CellValue::String(text.to_string()))
            }
            _ => CellValue::String(text.to_string()),
        }
    }
}
```

### 2.5 Multi-tab results + tab pinning

Each query execution opens a result tab. Tabs can be pinned to prevent
replacement.

**Steps:**
1. Implement result tab creation in `QueryEditor::execute()`
2. Non-pinned result tabs are replaced by default on re-execution
3. Pinned tabs create a new result tab on re-execution
4. Tab naming via `-- @name Monthly Sales` SQL comments

**Tab naming via SQL comments** — A comment like `-- @name Monthly Sales`
above a query names the result tab "Monthly Sales" instead of the default.
The prefix keyword (`@name`) is configurable in `DatabaseSettings.tab_naming_prefix`.

### 2.6 In-editor results (inline below query)

Display results directly below the query in the SQL buffer (like DataGrip's
in-editor results mode). Full-width grid that adjusts to editor width.

**Steps:**
1. Implement vertical split in `QueryEditor::render()` (see UI Spec Section 4)
2. Implement draggable split handle with ratio state
3. Implement independent grid instances per split

**Independent split data grids** — When the editor is split, each split
gets its own independent data grid instance with separate filter/sort state.
This is managed by giving each split its own `Entity<ResultGrid>`.

```rust
impl Item for QueryEditor {
    fn can_split(&self) -> bool { true }

    fn clone_on_split(
        &self,
        _workspace_id: Option<WorkspaceId>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Task<Option<Entity<Self>>> {
        let editor_clone = self.editor.update(cx, |editor, cx| {
            editor.clone_on_split(None, window, cx)
        });
        let connection_manager = self.connection_manager.clone();
        let connection_id = self.connection_id.clone();

        Task::ready(Some(cx.new(|cx| {
            let mut cloned = QueryEditor::new(connection_manager, connection_id, window, cx);
            // Clone gets its own independent ResultGrid (not shared)
            cloned
        })))
    }
}
```

### 2.7 Query history per connection

Save all executed queries with timestamp, duration, row count, and error status.

**Steps:**
1. Create `crates/database_core/src/history.rs`
2. Define `QueryHistoryEntry` struct
3. Store in `db` crate's KVP store, scoped by connection ID
4. Implement searchable history panel (optional, can be a `ModalView`)

```rust
// crates/database_core/src/history.rs

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct QueryHistoryEntry {
    pub id: usize,
    pub connection_id: ConnectionId,
    pub sql: String,
    pub executed_at: chrono::DateTime<chrono::Utc>,
    pub duration: Duration,
    pub row_count: Option<usize>,
    pub error: Option<String>,
    pub schema: Option<String>,
}

pub struct QueryHistory {
    entries: Vec<QueryHistoryEntry>,
    max_entries: usize,
    next_id: usize,
}

impl QueryHistory {
    pub fn new(max_entries: usize) -> Self {
        Self { entries: Vec::new(), max_entries, next_id: 0 }
    }

    pub fn add(&mut self, entry: QueryHistoryEntry) {
        self.entries.push(entry);
        if self.entries.len() > self.max_entries {
            self.entries.remove(0);
        }
    }

    pub fn search(&self, query: &str) -> Vec<&QueryHistoryEntry> {
        self.entries.iter()
            .filter(|e| e.sql.to_lowercase().contains(&query.to_lowercase()))
            .rev()
            .collect()
    }

    pub fn for_connection(&self, connection_id: &ConnectionId) -> Vec<&QueryHistoryEntry> {
        self.entries.iter()
            .filter(|e| &e.connection_id == connection_id)
            .rev()
            .collect()
    }

    /// Persist to KVP store
    pub fn save(&self, cx: &App) -> Result<()> {
        // Use db::kvp::KeyValueStore
        todo!()
    }

    /// Load from KVP store
    pub fn load(cx: &App) -> Result<Self> {
        todo!()
    }
}
```

### 2.8 Query cancellation

Cancel a running query via driver-specific mechanisms.

**Steps:**
1. Implement `cancel()` on each `DatabaseConnection` implementation
2. Wire `Escape` keybinding to `CancelQuery` action
3. Show cancel button in toolbar while executing
4. Emit `ExecutionFailed(DatabaseError::Cancelled)` on cancel

```rust
// Driver-specific cancellation implementations:
//
// PostgreSQL: pg_cancel_backend(pid)
//   - Requires a separate connection to send the cancel signal
//   - PID obtained from pg_stat_activity after query starts
//
// MySQL: KILL QUERY <id>
//   - Requires a separate connection
//   - Thread ID obtained from SHOW PROCESSLIST
//
// SQLite: sqlite3_interrupt()
//   - Called on the same connection handle
//   - Already implemented in SqliteConnection::cancel()
```

### 2.9 Explain Plan (text + diagram)

**Steps:**
1. Create `crates/database_ui/src/explain_plan_viewer.rs`
2. Implement text view (table of operations, costs, row estimates)
3. Implement diagram view (tree rendering of plan nodes)
4. Parse EXPLAIN output per driver into `ExplainPlan` struct

Two views:
1. **Table view**: Operations, object names, row estimates, costs
2. **Diagram view**: Graphical execution plan with node graph

```rust
// crates/database_ui/src/explain_plan_viewer.rs

pub struct ExplainPlanViewer {
    plan: ExplainPlan,
    view_mode: ExplainViewMode,
    focus_handle: FocusHandle,
}

pub enum ExplainViewMode { Table, Diagram, RawText }

impl Render for ExplainPlanViewer {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        match self.view_mode {
            ExplainViewMode::Table => self.render_table_view(window, cx),
            ExplainViewMode::Diagram => self.render_diagram_view(window, cx),
            ExplainViewMode::RawText => self.render_raw_text(window, cx),
        }
    }
}

impl ExplainPlanViewer {
    fn render_table_view(&self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // Table with columns: Operation | Object | Rows | Cost | Time
        let headers = vec!["Operation", "Object", "Est. Rows", "Cost", "Actual Rows"]
            .into_iter()
            .map(|h| Label::new(h).into_any_element())
            .collect::<Vec<_>>();

        Table::new(5)
            .header(headers.into_table_row(5))
            .uniform_list("explain-nodes", self.plan.nodes.len(), |range, window, cx| {
                // Render each plan node as a row with indentation
                todo!()
            })
            .striped()
    }

    fn render_diagram_view(&self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // Tree visualization of plan nodes
        // Each node shows: operation name, cost bar, row count
        // Lines connect parent to child nodes
        todo!()
    }

    fn render_raw_text(&self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // Read-only editor with the raw EXPLAIN output
        todo!()
    }
}
```

### 2.10 Execute to file

Run a query and write results directly to a file in a chosen format,
without loading them into the grid. Useful for large exports.

**Steps:**
1. Add `ExecuteToFile` action
2. Show file picker dialog with format selection
3. Stream results directly to file (no in-memory buffering)
4. Show progress indicator during export

### 2.11 SQL formatting

Integrate a SQL formatter (via LSP or dedicated formatter like `sqlformat`).

**Steps:**
1. Add `FormatQuery` action (Ctrl+Shift+F)
2. Use LSP formatting if available
3. Fallback to `sqlformat` crate for local formatting
4. Format selected text only if selection exists

```rust
impl QueryEditor {
    pub fn format_sql(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        let sql = self.get_sql_to_execute(cx);
        // Try LSP formatting first, fallback to sqlformat crate
        let formatted = sqlformat::format(
            &sql,
            &sqlformat::QueryParams::None,
            &sqlformat::FormatOptions {
                indent: sqlformat::Indent::Spaces(2),
                uppercase: Some(true),
                lines_between_queries: 2,
            },
        );
        self.editor.update(cx, |editor, cx| {
            editor.set_text(&formatted, window, cx);
        });
    }
}
```

### 2.12 SQL Generator (DDL export)

Generate the complete DDL for an entire database or schema in one click.
Right-click a schema in the Database Explorer > "Generate DDL" > choose output
(clipboard, new buffer, or file).

**Steps:**
1. Implement `generate_ddl()` on `DatabaseConnection` (introspect_ddl for each object)
2. Add "Generate DDL" to schema node context menu
3. Show output picker: clipboard, new QueryEditor tab, or file
4. Include options: DROP IF EXISTS, IF NOT EXISTS, comments

```rust
// crates/database_core/src/ddl_generator.rs

pub struct DdlGeneratorOptions {
    pub include_drop: bool,
    pub if_not_exists: bool,
    pub include_comments: bool,
    pub include_indexes: bool,
    pub include_constraints: bool,
    pub object_filter: Option<String>,  // Regex filter
}

pub async fn generate_schema_ddl(
    connection: &dyn DatabaseConnection,
    schema: &str,
    options: &DdlGeneratorOptions,
) -> Result<String> {
    let objects = connection.introspect_names().await?;
    let mut ddl = String::new();

    // Order: sequences, types, tables, views, functions, indexes
    let ordered = order_by_dependencies(&objects);

    for object in &ordered {
        if let Some(filter) = &options.object_filter {
            let regex = regex::Regex::new(filter)?;
            if !regex.is_match(&object.name) { continue; }
        }

        if options.include_drop {
            ddl.push_str(&format!("DROP {} IF EXISTS {};\n",
                object.object_type.sql_keyword(),
                object.name,
            ));
        }

        let object_ddl = connection.introspect_ddl(object).await?;
        ddl.push_str(&object_ddl);
        ddl.push_str(";\n\n");
    }

    Ok(ddl)
}
```

---

## Phase 3 — Interactive Data Grid

### 3.1 Inline editing + type-aware cell editors

Double-click a cell to edit. Cell editor adapts to column type.

**Steps:**
1. Implement `ResultGrid::start_editing()` (already sketched in 2.4)
2. Create type-specific editor configurations
3. Implement validation per type before committing
4. Handle Tab navigation between cells while editing

```rust
// crates/database_ui/src/result_grid.rs (cell editor section)

/// Create the appropriate inline editor for a column type
fn create_cell_editor(
    column: &ColumnInfo,
    current_value: &CellValue,
    window: &mut Window,
    cx: &mut Context<ResultGrid>,
) -> Entity<Editor> {
    match &column.normalized_type {
        DataType::Boolean => {
            // Boolean cells don't use an editor — Space toggles directly
            unreachable!("Boolean cells use toggle, not editor")
        }
        DataType::Integer | DataType::BigInt | DataType::SmallInt => {
            let editor = cx.new(|cx| {
                let mut editor = Editor::single_line(window, cx);
                editor.set_text(&current_value.display_string(&NumberFormatSettings::default()), window, cx);
                editor.select_all(&Default::default(), window, cx);
                editor
            });
            // Add numeric validation on input
            editor
        }
        DataType::Json | DataType::Jsonb => {
            // Large JSON opens ValueEditor panel instead
            // Small JSON gets inline editor
            let text = match current_value {
                CellValue::Json(v) => serde_json::to_string_pretty(v).unwrap_or_default(),
                _ => current_value.display_string(&NumberFormatSettings::default()),
            };
            cx.new(|cx| {
                let mut editor = Editor::single_line(window, cx);
                editor.set_text(&text, window, cx);
                editor
            })
        }
        DataType::Date | DataType::Timestamp | DataType::TimestampTz => {
            cx.new(|cx| {
                let mut editor = Editor::single_line(window, cx);
                editor.set_text(&current_value.display_string(&NumberFormatSettings::default()), window, cx);
                editor.select_all(&Default::default(), window, cx);
                // Placeholder hint for date format
                editor
            })
        }
        _ => {
            // Default text editor
            cx.new(|cx| {
                let mut editor = Editor::single_line(window, cx);
                editor.set_text(&current_value.display_string(&NumberFormatSettings::default()), window, cx);
                editor.select_all(&Default::default(), window, cx);
                editor
            })
        }
    }
}

/// Validate edited value before committing
fn validate_cell_value(text: &str, column: &ColumnInfo) -> Result<CellValue, String> {
    if text.eq_ignore_ascii_case("null") {
        if column.is_nullable {
            return Ok(CellValue::Null);
        } else {
            return Err(format!("Column '{}' does not allow NULL", column.name));
        }
    }
    match &column.normalized_type {
        DataType::Integer | DataType::BigInt | DataType::SmallInt => {
            text.parse::<i64>()
                .map(CellValue::Integer)
                .map_err(|_| format!("'{}' is not a valid integer", text))
        }
        DataType::Float | DataType::Double => {
            text.parse::<f64>()
                .map(CellValue::Float)
                .map_err(|_| format!("'{}' is not a valid number", text))
        }
        DataType::Boolean => {
            match text.to_lowercase().as_str() {
                "true" | "t" | "1" | "yes" => Ok(CellValue::Boolean(true)),
                "false" | "f" | "0" | "no" => Ok(CellValue::Boolean(false)),
                _ => Err(format!("'{}' is not a valid boolean", text)),
            }
        }
        DataType::Json | DataType::Jsonb => {
            serde_json::from_str::<serde_json::Value>(text)
                .map(CellValue::Json)
                .map_err(|e| format!("Invalid JSON: {}", e))
        }
        _ => Ok(CellValue::String(text.to_string())),
    }
}
```

Cell editor types:
- Text: inline text input
- Integer/Float: numeric input with validation
- Date/Time: formatted input with placeholder hint
- JSON: inline for small values, opens Value Editor for large
- Boolean: toggle (see 3.2)
- BLOB: "Open in Value Editor" button

### 3.2 Boolean toggle

`Space` key toggles boolean values. Single-key shortcuts:
- `t` → true, `f` → false, `n` → null, `d` → default
- Dropdown of possible values on `Enter`

**Steps:**
1. Register Space as toggle action in `ResultGrid` for boolean columns
2. Implement t/f/n/d single-key shortcuts in grid normal mode
3. Create pending edit on toggle

```rust
impl ResultGrid {
    fn toggle_boolean(&mut self, row: usize, col: usize, cx: &mut Context<Self>) {
        let current = &self.rows[row][col];
        let new_value = match current {
            CellValue::Boolean(true) => CellValue::Boolean(false),
            CellValue::Boolean(false) => CellValue::Null,
            CellValue::Null => CellValue::Boolean(true),
            _ => return,
        };
        self.apply_edit(row, col, new_value, cx);
    }

    fn set_boolean(&mut self, row: usize, col: usize, value: CellValue, cx: &mut Context<Self>) {
        if !matches!(self.columns[col].info.normalized_type, DataType::Boolean) {
            return;
        }
        self.apply_edit(row, col, value, cx);
    }

    fn apply_edit(&mut self, row: usize, col: usize, new_value: CellValue, cx: &mut Context<Self>) {
        let original = self.rows[row][col].clone();
        if new_value != original {
            let operation = DataOperation::UpdateCell {
                row, col,
                old_value: original.clone(),
                new_value: new_value.clone(),
            };
            self.edit_history.push(operation);
            self.pending_edits.insert(
                CellAddress { row, col },
                PendingEdit { original, current: new_value, kind: EditKind::Update },
            );
            cx.emit(ResultGridEvent::PendingChangesUpdated {
                count: self.pending_edits.len(),
            });
            cx.notify();
        }
    }
}
```

### 3.3 Value Editor panel

Side panel for editing large or complex values.

**Steps:**
1. Create `crates/database_ui/src/value_editor.rs`
2. Implement `Panel` + `Focusable` + `EventEmitter<PanelEvent>` + `Render`
3. Implement content type detection (JSON, XML, image, hex)
4. Create `Entity<Editor>` with appropriate language mode per type
5. Implement hex view for binary data
6. Implement image preview for detected image BLOBs

(See detailed structure in [UI/Interface Specification — Section 8](#8-side-panels).)

```rust
// crates/database_ui/src/value_editor.rs

pub struct ValueEditor {
    content_editor: Option<Entity<Editor>>,
    content_type: ContentType,
    source_cell: Option<CellAddress>,
    source_grid: Option<WeakEntity<ResultGrid>>,
    is_read_only: bool,
    focus_handle: FocusHandle,
    width: Option<Pixels>,
    position: DockPosition,
}

pub enum ContentType {
    Json,
    Xml,
    PlainText,
    Hex { data: Vec<u8> },
    Image { data: Arc<[u8]>, format: ImageFormat },
}

pub enum ImageFormat { Png, Jpeg, Gif, Webp, Unknown }

impl ValueEditor {
    pub fn set_content(
        &mut self,
        value: &CellValue,
        column: &ColumnInfo,
        source_cell: CellAddress,
        source_grid: WeakEntity<ResultGrid>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) {
        self.source_cell = Some(source_cell);
        self.source_grid = Some(source_grid);

        match value {
            CellValue::Json(json) => {
                let text = serde_json::to_string_pretty(json).unwrap_or_default();
                self.content_type = ContentType::Json;
                self.create_editor_with_language(&text, "JSON", window, cx);
            }
            CellValue::String(s) if looks_like_xml(s) => {
                self.content_type = ContentType::Xml;
                self.create_editor_with_language(s, "XML", window, cx);
            }
            CellValue::String(s) => {
                self.content_type = ContentType::PlainText;
                self.create_editor_with_language(s, "Plain Text", window, cx);
            }
            CellValue::Bytes(bytes) => {
                if let Some(format) = detect_image_format(bytes) {
                    self.content_type = ContentType::Image {
                        data: bytes.clone().into(),
                        format,
                    };
                    self.content_editor = None; // Image uses gpui::img() instead
                } else {
                    self.content_type = ContentType::Hex { data: bytes.clone() };
                    let hex_text = format_hex_view(bytes);
                    self.create_editor_with_language(&hex_text, "Plain Text", window, cx);
                }
            }
            _ => {
                let text = value.display_string(&NumberFormatSettings::default());
                self.content_type = ContentType::PlainText;
                self.create_editor_with_language(&text, "Plain Text", window, cx);
            }
        }
        cx.notify();
    }

    fn create_editor_with_language(
        &mut self,
        text: &str,
        language: &str,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) {
        let editor = cx.new(|cx| {
            let mut editor = Editor::multi_line(window, cx);
            editor.set_text(text, window, cx);
            if self.is_read_only {
                editor.set_read_only(true);
            }
            // Set language mode via language registry
            editor
        });
        self.content_editor = Some(editor);
    }
}

impl Render for ValueEditor {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        v_flex()
            .size_full()
            .child(self.render_header(window, cx))
            .child(match &self.content_type {
                ContentType::Image { data, .. } => {
                    div()
                        .flex_1()
                        .items_center()
                        .justify_center()
                        .child(gpui::img(data.clone()))
                        .into_any_element()
                }
                _ => {
                    if let Some(editor) = &self.content_editor {
                        div().flex_1().child(editor.clone()).into_any_element()
                    } else {
                        div().flex_1().into_any_element()
                    }
                }
            })
    }
}

/// Format bytes as hex view: offset | hex bytes | ASCII
fn format_hex_view(data: &[u8]) -> String {
    let mut output = String::new();
    for (offset, chunk) in data.chunks(16).enumerate() {
        // Offset column
        output.push_str(&format!("{:08x}  ", offset * 16));
        // Hex columns
        for (i, byte) in chunk.iter().enumerate() {
            output.push_str(&format!("{:02x} ", byte));
            if i == 7 { output.push(' '); }
        }
        // Pad if last line is short
        for _ in chunk.len()..16 {
            output.push_str("   ");
        }
        output.push_str(" |");
        // ASCII column
        for byte in chunk {
            let c = if byte.is_ascii_graphic() || *byte == b' ' {
                *byte as char
            } else {
                '.'
            };
            output.push(c);
        }
        output.push_str("|\n");
    }
    output
}

fn detect_image_format(data: &[u8]) -> Option<ImageFormat> {
    if data.starts_with(b"\x89PNG") { Some(ImageFormat::Png) }
    else if data.starts_with(b"\xFF\xD8\xFF") { Some(ImageFormat::Jpeg) }
    else if data.starts_with(b"GIF8") { Some(ImageFormat::Gif) }
    else if data.starts_with(b"RIFF") && data.get(8..12) == Some(b"WEBP") { Some(ImageFormat::Webp) }
    else { None }
}
```

**LOB size hint** — When a value exceeds `max_lob_size`, display an
interactive hint: "Value truncated (2.4 MB). Click to load full content."
Clicking loads the full value into the Value Editor.

### 3.4 Multi-cell editing

Select multiple cells of the same type in non-unique columns. Start typing to
edit all selected cells simultaneously.

### 3.5 Record View

Vertical view of a single row: field names on the left, values on the right.
Toggle via toolbar button or `Ctrl+Shift+R`. Navigate between rows with
arrow keys.

### 3.6 Sorting (click header, multi-column, server/client)

- Click column header: cycle ASC > DESC > unsorted
- `Alt+Click`: add to existing sort (multi-column stacking)
- Sort indicator showing direction and priority number
- Toggle server-side (`ORDER BY`) vs client-side sorting
- Dedicated `ORDER BY` text field under toolbar

### 3.7 Local filtering (checkbox per column)

Click filter icon in column header > checkbox list of distinct values.
Select/deselect values to filter. "Clear Local Filter For All Columns" action
in toolbar.

### 3.8 Server-side filtering (WHERE clause + quick filters)

- `WHERE` text field under toolbar (e.g., `age > 30 AND name LIKE 'A%'`)
- Right-click cell > "Filter by" submenu:
  - Equals / Not Equals
  - Greater Than / Less Than
  - LIKE / NOT LIKE
  - IS NULL / IS NOT NULL

### 3.9 Column list search (Ctrl+F12)

Type-ahead popup to jump to a column by name in wide tables.
Uses fuzzy matching.

### 3.10 Expand/Shrink selection

`Ctrl+W` / `Ctrl+Shift+W` for progressive selection expansion in the grid:
cell > row > column > all. Similar to structural selection in the code editor.

### 3.11 Grid paging control

Pagination bar at bottom: page number, total rows, page size selector.
Keyboard shortcuts: `Ctrl+Page Down` / `Ctrl+Page Up`.

### 3.12 View modes (Table, Tree, Text, Transpose)

- **Table**: Default grid view
- **Tree**: Key-value pairs with expandable nodes (useful for JSON columns)
- **Text**: Raw text representation
- **Transpose**: Rows and columns swapped

### 3.13 FK navigation + Virtual FK

- Click a FK value > navigate to referenced row in referenced table
- Find usages: right-click a row > find rows referencing it via FK
- **Virtual Foreign Keys**: User-defined FK relationships for legacy databases
  that lack proper constraints. Defined via regex or explicit column mapping.
  Stored in project settings.

**Steps:**
1. Detect FK columns via introspection metadata
2. Render FK values as clickable links (underlined, accent color)
3. On click: open referenced table in new tab with WHERE filter
4. Implement "Find Usages" reverse lookup
5. Add virtual FK definition UI

```rust
// crates/database_ui/src/result_grid.rs (FK navigation section)

impl ResultGrid {
    /// Navigate to the referenced row via FK
    pub fn navigate_to_foreign_key(
        &self,
        row: usize,
        col: usize,
        workspace: &mut Workspace,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) {
        let column = &self.columns[col];
        let fk_ref = match &column.info.foreign_key_ref {
            Some(fk) => fk.clone(),
            None => return,
        };

        let value = &self.rows[row][col];
        let sql = format!(
            "SELECT * FROM {} WHERE {} = {}",
            fk_ref.referenced_table.qualified_name(),
            fk_ref.referenced_column,
            value.to_sql_literal(),
        );

        // Open a new QueryEditor tab with this query and execute it
        // workspace.open_query_editor(sql, connection_id, window, cx);
    }

    /// Find all rows in other tables that reference this row
    pub fn find_fk_usages(
        &self,
        row: usize,
        connection: &dyn DatabaseConnection,
        cx: &mut Context<Self>,
    ) -> Task<Result<Vec<FkUsage>>> {
        // Query information_schema for all tables with FKs pointing to this table
        // For each, execute a count query
        todo!()
    }
}

pub struct FkUsage {
    pub referencing_table: TableRef,
    pub referencing_column: String,
    pub count: usize,
}

/// Virtual FK definition (stored in project settings)
#[derive(Clone, Debug, Serialize, Deserialize, JsonSchema)]
pub struct VirtualForeignKey {
    pub name: String,
    pub source_table: String,
    pub source_column: String,
    pub target_table: String,
    pub target_column: String,
}
```

### 3.14 Aggregate View

Select multiple cells > floating popover shows aggregated values.

**Steps:**
1. Create `crates/database_ui/src/aggregate_view.rs`
2. Calculate aggregates on selection change
3. Render as `Popover` anchored to selection
4. Support pinning (click keeps visible)

(See [UI/Interface Specification — Section 8, AggregateView](#8-side-panels).)

```rust
// crates/database_ui/src/aggregate_view.rs

pub struct AggregateView {
    values: Vec<AggregateResult>,
    is_pinned: bool,
    anchor_position: Point<Pixels>,
}

pub struct AggregateResult {
    pub name: SharedString,
    pub value: String,
}

impl AggregateView {
    pub fn calculate(cells: &[(usize, usize)], rows: &[Vec<CellValue>], columns: &[ColumnDef]) -> Vec<AggregateResult> {
        let numeric_values: Vec<f64> = cells.iter()
            .filter_map(|(row, col)| match &rows[*row][*col] {
                CellValue::Integer(i) => Some(*i as f64),
                CellValue::Float(f) => Some(*f),
                _ => None,
            })
            .collect();

        if numeric_values.is_empty() {
            return vec![
                AggregateResult { name: "Count".into(), value: cells.len().to_string() },
            ];
        }

        let count = numeric_values.len();
        let sum: f64 = numeric_values.iter().sum();
        let avg = sum / count as f64;
        let min = numeric_values.iter().cloned().fold(f64::INFINITY, f64::min);
        let max = numeric_values.iter().cloned().fold(f64::NEG_INFINITY, f64::max);

        vec![
            AggregateResult { name: "Count".into(), value: count.to_string() },
            AggregateResult { name: "Sum".into(), value: format!("{:.2}", sum) },
            AggregateResult { name: "Avg".into(), value: format!("{:.2}", avg) },
            AggregateResult { name: "Min".into(), value: format!("{:.2}", min) },
            AggregateResult { name: "Max".into(), value: format!("{:.2}", max) },
        ]
    }
}

impl Render for AggregateView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        Popover::new()
            .child(
                v_flex()
                    .p_2()
                    .gap_1()
                    .children(self.values.iter().map(|v| {
                        h_flex()
                            .justify_between()
                            .gap_4()
                            .child(Label::new(v.name.clone()).size(LabelSize::Small).color(Color::Muted))
                            .child(Label::new(v.value.clone()).size(LabelSize::Small))
                    }))
            )
    }
}
```

Built-in aggregators: Count, Sum, Average, Min, Max, Median, StdDev, Variance, Distinct Count.
Custom aggregator scripts (Lua or Rhai, stored in config dir).
Dynamic update as selection changes.

### 3.15 CRUD operations (add/delete/clone row)

- Add row: inserts empty row at bottom with default values
- Delete row: marks row for deletion (red highlight)
- Clone row: duplicates an existing row for easy insertion
- All changes are local until submitted

**Steps:**
1. Implement `AddRow` action → append row with default values
2. Implement `DeleteSelectedRows` → mark rows as deleted
3. Implement `CloneRow` → duplicate row data as insert
4. Track all operations in `DataEditHistory`

```rust
impl ResultGrid {
    pub fn add_row(&mut self, cx: &mut Context<Self>) {
        let default_values: Vec<CellValue> = self.columns.iter()
            .map(|col| {
                if let Some(default) = &col.info.default_value {
                    CellValue::String(default.clone()) // Will be parsed on commit
                } else if col.info.is_nullable {
                    CellValue::Null
                } else {
                    CellValue::String(String::new())
                }
            })
            .collect();

        let new_row_index = self.rows.len();
        let mut rows = Arc::make_mut(&mut self.rows);
        rows.push(default_values.clone());

        self.edit_history.push(DataOperation::InsertRow {
            row: new_row_index,
            data: default_values,
        });
        self.pending_edits.insert(
            CellAddress { row: new_row_index, col: 0 },
            PendingEdit {
                original: CellValue::Null,
                current: CellValue::Null,
                kind: EditKind::Insert,
            },
        );
        self.selection = GridSelection::Cell { row: new_row_index, col: 0 };
        cx.emit(ResultGridEvent::PendingChangesUpdated { count: self.pending_edits.len() });
        cx.notify();
    }

    pub fn delete_selected_rows(&mut self, cx: &mut Context<Self>) {
        let rows_to_delete = match &self.selection {
            GridSelection::Rows(rows) => rows.clone(),
            GridSelection::Cell { row, .. } => vec![*row],
            GridSelection::Range { start, end } => (start.0..=end.0).collect(),
            _ => return,
        };

        for row_index in &rows_to_delete {
            let row_data = self.rows[*row_index].clone();
            self.edit_history.push(DataOperation::DeleteRow {
                row: *row_index,
                data: row_data,
            });
            // Mark first cell of row as deleted (the row styling handles the rest)
            self.pending_edits.insert(
                CellAddress { row: *row_index, col: 0 },
                PendingEdit {
                    original: self.rows[*row_index][0].clone(),
                    current: CellValue::Null,
                    kind: EditKind::Delete,
                },
            );
        }
        cx.emit(ResultGridEvent::PendingChangesUpdated { count: self.pending_edits.len() });
        cx.notify();
    }

    pub fn clone_row(&mut self, row_index: usize, cx: &mut Context<Self>) {
        let cloned_data = self.rows[row_index].clone();
        let new_row_index = self.rows.len();
        let mut rows = Arc::make_mut(&mut self.rows);
        rows.push(cloned_data.clone());

        self.edit_history.push(DataOperation::CloneRow {
            source_row: row_index,
            new_row: new_row_index,
        });
        self.pending_edits.insert(
            CellAddress { row: new_row_index, col: 0 },
            PendingEdit {
                original: CellValue::Null,
                current: cloned_data[0].clone(),
                kind: EditKind::Insert,
            },
        );
        cx.emit(ResultGridEvent::PendingChangesUpdated { count: self.pending_edits.len() });
        cx.notify();
    }
}
```

### 3.16 DML Preview before commit

Before submitting changes, display a dialog showing the exact SQL that will
be executed (INSERT/UPDATE/DELETE statements). User can review and confirm.

**Steps:**
1. Generate DML from `pending_edits` via `ResultGrid::generate_dml()`
2. Show in a `ModalView` with syntax-highlighted SQL
3. Confirm button executes all statements in a transaction
4. Cancel reverts to the pending state (no changes lost)

```rust
// crates/database_ui/src/dml_preview.rs

pub struct DmlPreviewDialog {
    statements: Vec<String>,
    preview_editor: Entity<Editor>,
    table_ref: TableRef,
    connection_id: ConnectionId,
    focus_handle: FocusHandle,
}

impl ModalView for DmlPreviewDialog {
    fn fade_out_background(&self) -> bool { true }
}

impl Render for DmlPreviewDialog {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        Modal::new("dml-preview", ScrollHandle::new())
            .header(ModalHeader::new().headline("Review Pending Changes"))
            .section(
                div()
                    .p_2()
                    .child(Label::new(format!("{} statements", self.statements.len()))
                        .size(LabelSize::Small).color(Color::Muted))
                    .child(self.preview_editor.clone())
            )
            .footer(
                ModalFooter::new()
                    .start_slot(
                        Button::new("cancel", "Cancel")
                            .on_click(cx.listener(|this, _, window, cx| {
                                cx.emit(DismissEvent);
                            }))
                    )
                    .end_slot(
                        Button::new("commit", "Commit Changes")
                            .style(ButtonStyle::Filled)
                            .on_click(cx.listener(|this, _, window, cx| {
                                this.execute_commit(window, cx);
                            }))
                    )
            )
    }
}
```

### 3.17 Undo/Redo transactional

**Steps:**
1. Create `crates/database_core/src/data_edit_history.rs`
2. Implement operation stack with cursor
3. Wire `Ctrl+Z` / `Ctrl+Shift+Z` to undo/redo in grid context
4. Apply undo/redo to both `pending_edits` map and display

```rust
// crates/database_core/src/data_edit_history.rs

pub struct DataEditHistory {
    operations: Vec<DataOperation>,
    cursor: usize,  // Points to next operation slot
}

pub enum DataOperation {
    UpdateCell { row: usize, col: usize, old_value: CellValue, new_value: CellValue },
    InsertRow { row: usize, data: Vec<CellValue> },
    DeleteRow { row: usize, data: Vec<CellValue> },
    CloneRow { source_row: usize, new_row: usize },
}

impl DataEditHistory {
    pub fn new() -> Self {
        Self { operations: Vec::new(), cursor: 0 }
    }

    pub fn push(&mut self, operation: DataOperation) {
        // Truncate any redo history
        self.operations.truncate(self.cursor);
        self.operations.push(operation);
        self.cursor += 1;
    }

    pub fn undo(&mut self) -> Option<&DataOperation> {
        if self.cursor == 0 { return None; }
        self.cursor -= 1;
        Some(&self.operations[self.cursor])
    }

    pub fn redo(&mut self) -> Option<&DataOperation> {
        if self.cursor >= self.operations.len() { return None; }
        let operation = &self.operations[self.cursor];
        self.cursor += 1;
        Some(operation)
    }

    pub fn can_undo(&self) -> bool { self.cursor > 0 }
    pub fn can_redo(&self) -> bool { self.cursor < self.operations.len() }
    pub fn operation_count(&self) -> usize { self.operations.len() }

    /// Summary of pending changes for UI display
    pub fn pending_summary(&self) -> (usize, usize, usize) {
        let mut inserts = 0;
        let mut updates = 0;
        let mut deletes = 0;
        for op in &self.operations[..self.cursor] {
            match op {
                DataOperation::InsertRow { .. } | DataOperation::CloneRow { .. } => inserts += 1,
                DataOperation::UpdateCell { .. } => updates += 1,
                DataOperation::DeleteRow { .. } => deletes += 1,
            }
        }
        (inserts, updates, deletes)
    }
}

impl ResultGrid {
    pub fn undo_edit(&mut self, cx: &mut Context<Self>) {
        if let Some(operation) = self.edit_history.undo() {
            match operation {
                DataOperation::UpdateCell { row, col, old_value, .. } => {
                    self.pending_edits.remove(&CellAddress { row: *row, col: *col });
                    // Restore original value in display
                }
                DataOperation::InsertRow { row, .. } => {
                    let mut rows = Arc::make_mut(&mut self.rows);
                    if *row < rows.len() { rows.remove(*row); }
                    self.pending_edits.remove(&CellAddress { row: *row, col: 0 });
                }
                DataOperation::DeleteRow { row, .. } => {
                    self.pending_edits.remove(&CellAddress { row: *row, col: 0 });
                }
                DataOperation::CloneRow { new_row, .. } => {
                    let mut rows = Arc::make_mut(&mut self.rows);
                    if *new_row < rows.len() { rows.remove(*new_row); }
                    self.pending_edits.remove(&CellAddress { row: *new_row, col: 0 });
                }
            }
            cx.emit(ResultGridEvent::PendingChangesUpdated { count: self.pending_edits.len() });
            cx.notify();
        }
    }

    pub fn redo_edit(&mut self, cx: &mut Context<Self>) {
        if let Some(operation) = self.edit_history.redo() {
            // Re-apply the operation (inverse of undo)
            match operation {
                DataOperation::UpdateCell { row, col, new_value, old_value } => {
                    self.pending_edits.insert(
                        CellAddress { row: *row, col: *col },
                        PendingEdit {
                            original: old_value.clone(),
                            current: new_value.clone(),
                            kind: EditKind::Update,
                        },
                    );
                }
                // Similar for other variants...
                _ => {}
            }
            cx.emit(ResultGridEvent::PendingChangesUpdated { count: self.pending_edits.len() });
            cx.notify();
        }
    }
}
```

`Ctrl+Z` / `Ctrl+Shift+Z` in the data grid operates on the data edit stack,
separate from the text editor undo.

### 3.18 NULL indicator + display options

- NULL cells show a distinct visual indicator (dimmed "NULL" text)
- Configurable decimal/grouping separators for numbers
- Infinity and NaN rendering options
- Timestamp display precision (seconds, milliseconds, microseconds)
- **NEW: Change Display Type per column** — Right-click column header >
  "Change Display Type" > choose from available format options for that
  data type (e.g., timestamp as ISO 8601, Unix epoch, or relative time)

### 3.19 Grid heatmaps

Color-code numeric cells:
- **Diverging**: Two contrasting colors from a central value
- **Sequential**: Single color varying in intensity
- Applicable to whole table, individual columns, or boolean-only

### 3.20 NEW: Quick Actions floating toolbar

When a cell is selected, a floating toolbar appears above/below the cell with
context-sensitive actions based on the data type:

| Data Type | Actions |
|---|---|
| Text | Copy, Edit, Open in Value Editor |
| FK value | Navigate to Referenced Row, Filter by Value |
| JSON | Expand in Value Editor, Copy as JSON |
| Boolean | Toggle True/False/Null |
| NULL | Set Default, Set to... |
| Image/BLOB | Preview, Save to File |
| Number | Copy, Show in Aggregate View |

Can be disabled in `DatabaseSettings.show_quick_actions_toolbar`.

### 3.21 NEW: Value completion in cells

During inline cell editing, `Ctrl+Space` triggers local completion from
other values already loaded in the same column. Useful for filling in
repetitive categorical data.

### 3.22 NEW: Quick documentation hover on cells

Hover on a cell > popup showing:
- Full value (for truncated cells)
- Column type and constraints
- If FK: referenced table, referenced row preview
- If indexed: index name and type

---

## Phase 4 — Export/Import & Comparison

### 4.1 Export multi-format

Export from any data grid (table, query result, CSV file).

**Steps:**
1. Create `crates/database_ui/src/export.rs`
2. Implement `ExportFormat` trait with format-specific serializers
3. Create export dialog (`ModalView`) with format selection + options
4. Implement streaming export for large datasets (no full in-memory buffer)

| Format | Implementation |
|---|---|
| CSV | `csv` crate |
| TSV | `csv` crate with tab delimiter |
| JSON | `serde_json` |
| SQL (INSERT) | Custom SQL generator |
| SQL (DDL+DML) | Custom SQL generator |
| HTML | Template-based |
| Markdown | Custom formatter |
| Excel (xlsx) | `xlsxwriter` or `calamine` |

```rust
// crates/database_ui/src/export.rs

pub trait DataExporter: Send + Sync {
    fn format_name(&self) -> &str;
    fn file_extension(&self) -> &str;
    fn export_header(&self, columns: &[ColumnDef]) -> Result<String>;
    fn export_row(&self, row: &[CellValue], columns: &[ColumnDef]) -> Result<String>;
    fn export_footer(&self) -> Result<String> { Ok(String::new()) }
}

pub struct CsvExporter {
    pub delimiter: u8,
    pub quote: bool,
    pub include_header: bool,
}

impl DataExporter for CsvExporter {
    fn format_name(&self) -> &str { "CSV" }
    fn file_extension(&self) -> &str { "csv" }

    fn export_header(&self, columns: &[ColumnDef]) -> Result<String> {
        if !self.include_header { return Ok(String::new()); }
        let names: Vec<&str> = columns.iter().map(|c| c.info.name.as_str()).collect();
        Ok(names.join(&String::from(self.delimiter as char)) + "\n")
    }

    fn export_row(&self, row: &[CellValue], _columns: &[ColumnDef]) -> Result<String> {
        let values: Vec<String> = row.iter()
            .map(|v| v.display_string(&NumberFormatSettings::default()))
            .collect();
        Ok(values.join(&String::from(self.delimiter as char)) + "\n")
    }
}

pub struct SqlInsertExporter {
    pub table_name: String,
    pub batch_size: usize,
}

impl DataExporter for SqlInsertExporter {
    fn format_name(&self) -> &str { "SQL INSERT" }
    fn file_extension(&self) -> &str { "sql" }

    fn export_header(&self, columns: &[ColumnDef]) -> Result<String> {
        Ok(String::new())
    }

    fn export_row(&self, row: &[CellValue], columns: &[ColumnDef]) -> Result<String> {
        let col_names: Vec<&str> = columns.iter().map(|c| c.info.name.as_str()).collect();
        let values: Vec<String> = row.iter().map(|v| v.to_sql_literal()).collect();
        Ok(format!(
            "INSERT INTO {} ({}) VALUES ({});\n",
            self.table_name,
            col_names.join(", "),
            values.join(", "),
        ))
    }
}

pub struct JsonExporter {
    pub pretty: bool,
    pub array_format: bool,  // true: [{...},...], false: one object per line
}

pub struct MarkdownExporter;
pub struct HtmlExporter;
pub struct ExcelExporter;

/// Export dialog for selecting format and options
pub struct ExportDialog {
    format: ExportFormat,
    destination: ExportDestination,
    include_header: bool,
    selection_only: bool,
    focus_handle: FocusHandle,
}

pub enum ExportDestination {
    File(PathBuf),
    Clipboard,
    NewBuffer,
}

/// Main export function
pub async fn export_data(
    exporter: &dyn DataExporter,
    columns: &[ColumnDef],
    rows: &[Vec<CellValue>],
    destination: &ExportDestination,
) -> Result<usize> {
    let mut output = exporter.export_header(columns)?;
    for row in rows {
        output.push_str(&exporter.export_row(row, columns)?);
    }
    output.push_str(&exporter.export_footer()?);

    match destination {
        ExportDestination::File(path) => {
            std::fs::write(path, &output)?;
        }
        ExportDestination::Clipboard => {
            // Use GPUI clipboard API
        }
        ExportDestination::NewBuffer => {
            // Open in new editor tab
        }
    }
    Ok(rows.len())
}
```

### 4.2 Clipboard export

`Ctrl+C` copies selection in the currently active extractor format.
Configurable default format (CSV, JSON, SQL INSERT, Markdown).

**Steps:**
1. Implement `copy_selection()` on `ResultGrid`
2. Use default format from `DatabaseSettings.default_export_format`
3. Support Ctrl+Shift+C for "Copy As..." format picker

```rust
impl ResultGrid {
    pub fn copy_selection(&self, format: ExportFormat, cx: &mut App) {
        let (columns, rows) = self.selected_data();
        let exporter = create_exporter(format, &self.table_ref_name());
        let mut output = exporter.export_header(&columns).unwrap_or_default();
        for row in &rows {
            output.push_str(&exporter.export_row(row, &columns).unwrap_or_default());
        }
        cx.write_to_clipboard(ClipboardItem::new_string(output));
    }

    fn selected_data(&self) -> (Vec<ColumnDef>, Vec<Vec<CellValue>>) {
        match &self.selection {
            GridSelection::Cell { row, col } => {
                (vec![self.columns[*col].clone()], vec![vec![self.rows[*row][*col].clone()]])
            }
            GridSelection::Range { start, end } => {
                let cols: Vec<ColumnDef> = (start.1..=end.1)
                    .map(|c| self.columns[c].clone())
                    .collect();
                let rows: Vec<Vec<CellValue>> = (start.0..=end.0)
                    .map(|r| (start.1..=end.1).map(|c| self.rows[r][c].clone()).collect())
                    .collect();
                (cols, rows)
            }
            GridSelection::Rows(indices) => {
                let cols = self.columns.clone();
                let rows: Vec<Vec<CellValue>> = indices.iter()
                    .map(|r| self.rows[*r].clone())
                    .collect();
                (cols, rows)
            }
            GridSelection::All => {
                (self.columns.clone(), self.rows.as_ref().clone())
            }
            _ => (vec![], vec![]),
        }
    }
}
```

### 4.3 Custom extractors (Lua or Rhai scripts)

Community-creatable export scripts stored in config directory.
Extractors and aggregators are interchangeable.

### 4.4 Import CSV wizard

Dedicated UI: file selector, delimiter config, header row toggle,
column mapping, data preview, target table selection.

### 4.5 Edit CSV as table

Open a `.csv` file in Zed > "Edit As Table" action > full data grid editor
with configurable parsing options (delimiter, quote char, encoding).

### 4.6 pg_dump / mysqldump integration

Right-click a database/schema in Explorer > "Dump" / "Restore".
Calls platform CLI tools with appropriate arguments.

### 4.7 Data diff viewer

Compare two tables or query results side by side:
- Highlighted differences (added, removed, changed rows)
- Configurable tolerance (number of column differences for row matching)
- Column exclusion from comparison
- Detect Column Insertion option
- Configurable row limit (default: 500)

### 4.8 Schema diff + migration DDL generation

Compare two schemas > generate migration DDL to transform one into the other.
Supports cross-database comparison.

### 4.9 Create table from grid result

Any data grid result can create a new table:
- Right-click grid > "Create Table From Result"
- Choose target connection and schema
- Configure column types (auto-detected from data)
- Works cross-vendor (e.g., create MySQL table from PostgreSQL query result)

### 4.10 Copy table cross-vendor

Drag & drop a table from one connection to another in the Database Explorer,
or right-click > "Copy To" > select target. Handles type mapping between
database vendors.

### 4.11 NEW: Paste from Excel / clipboard detection

Paste tabular data from clipboard into a data grid or new table.
Auto-detect format (TSV from Excel, CSV, etc.) or let user configure.
Shows preview before inserting.

### 4.12 NEW: SQL Generator (full DDL export)

Generate complete DDL for an entire database or schema:
- Right-click schema > "Generate DDL"
- Includes: tables, columns, constraints, indexes, views, functions, sequences
- Output to: new buffer, clipboard, or file
- Options: include/exclude DROP statements, IF NOT EXISTS

---

## Phase 5 — AI & Agentic Integration

### 5.1 AgentTool: execute_query

```rust
#[derive(Debug, Serialize, Deserialize, JsonSchema)]
pub struct ExecuteQueryToolInput {
    /// The SQL query to execute.
    pub sql: String,
    /// Name of the database connection to use.
    pub connection: String,
    /// Max rows to return (default: 100).
    #[serde(default = "default_limit")]
    pub limit: usize,
}

impl AgentTool for ExecuteQueryTool {
    const NAME: &'static str = "execute_query";
    fn kind() -> ToolKind { ToolKind::Other }
    // ...
}
```

### 5.2 AgentTool: describe_database_object

Returns DDL + columns + FK + indexes for a database object.

```rust
#[derive(Debug, Serialize, Deserialize, JsonSchema)]
pub struct DescribeObjectToolInput {
    /// The name of the database object (table, view, function).
    pub object_name: String,
    /// Name of the database connection.
    pub connection: String,
    /// Schema name (optional, uses default if omitted).
    pub schema: Option<String>,
    /// Level of detail: "summary" (columns only), "full" (DDL + FK + indexes).
    #[serde(default = "default_detail_level")]
    pub detail: String,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct DescribeObjectToolOutput {
    pub object_name: String,
    pub object_type: String,
    pub columns: Vec<ColumnSummary>,
    pub primary_key: Option<Vec<String>>,
    pub foreign_keys: Vec<ForeignKeySummary>,
    pub indexes: Vec<IndexSummary>,
    pub ddl: Option<String>,
    pub row_count_estimate: Option<u64>,
}

impl AgentTool for DescribeObjectTool {
    const NAME: &'static str = "describe_database_object";
    fn kind() -> ToolKind { ToolKind::Other }
    // Returns structured JSON with schema information
}
```

### 5.3 AgentTool: list_database_objects

Lists tables, views, functions in a schema. Supports filtering by type.

```rust
#[derive(Debug, Serialize, Deserialize, JsonSchema)]
pub struct ListObjectsToolInput {
    /// Name of the database connection.
    pub connection: String,
    /// Schema name (optional).
    pub schema: Option<String>,
    /// Filter by type: "tables", "views", "functions", "all".
    #[serde(default = "default_all")]
    pub object_type: String,
    /// Optional name pattern (SQL LIKE syntax).
    pub pattern: Option<String>,
}

impl AgentTool for ListObjectsTool {
    const NAME: &'static str = "list_database_objects";
    fn kind() -> ToolKind { ToolKind::Other }
}
```

### 5.4 AgentTool: explain_query

Runs EXPLAIN on a query and returns the execution plan.

```rust
#[derive(Debug, Serialize, Deserialize, JsonSchema)]
pub struct ExplainQueryToolInput {
    /// The SQL query to explain.
    pub sql: String,
    /// Name of the database connection.
    pub connection: String,
    /// Whether to run EXPLAIN ANALYZE (actually executes the query).
    #[serde(default)]
    pub analyze: bool,
}

impl AgentTool for ExplainQueryTool {
    const NAME: &'static str = "explain_query";
    fn kind() -> ToolKind { ToolKind::Other }
}
```

### 5.5 AgentTool: modify_data (with confirmation)

Executes INSERT/UPDATE/DELETE. Requires user confirmation (`ToolKind::Write`).
Shows DML preview before execution.

```rust
#[derive(Debug, Serialize, Deserialize, JsonSchema)]
pub struct ModifyDataToolInput {
    /// The SQL statement to execute (INSERT/UPDATE/DELETE/DDL).
    pub sql: String,
    /// Name of the database connection.
    pub connection: String,
    /// Human-readable description of what this modification does.
    pub description: String,
}

impl AgentTool for ModifyDataTool {
    const NAME: &'static str = "modify_data";
    fn kind() -> ToolKind { ToolKind::Write } // Requires user confirmation
}
```

### 5.6 Schema context in AI chat (@db:, @table:, @schema:)

New `MentionUri` variants:
- `@db:my_connection` — injects connection metadata
- `@table:users` — injects DDL of the `users` table
- `@schema:public` — injects list of all objects in the `public` schema

### 5.7 MCP server for DB resources

Expose database metadata as MCP resources:
- `database://conn/schema/public` — schema metadata
- `database://conn/table/users` — table DDL
- Tools: `execute_query`, `describe_table`, `list_objects`

### 5.8 Natural language to SQL generation

User types in plain language > AI generates SQL with schema context.
Works both in the chat panel and inline in the query editor.

### 5.9 Complex query explanation

AI explains what a complex query does in natural language, including
join logic, aggregation, and subquery behavior.

### 5.10 Query optimization (EXPLAIN analysis + suggestions)

AI analyzes the execution plan and suggests:
- Missing indexes
- Query rewrites (subquery to JOIN, etc.)
- Redundant operations
- Generates CREATE INDEX DDL

### 5.11 SQL error debugging with schema context

When a query fails, AI proposes corrections using the full schema context:
- Typo detection (column/table name suggestions)
- Type mismatch identification
- Missing JOIN conditions

### 5.12 Migration DDL generation

AI generates migration scripts from natural language descriptions:
"Add an email verification column to the users table" > ALTER TABLE DDL.

### 5.13 Data insights (anomalies, trends)

AI analyzes a query result and provides observations:
- Statistical anomalies
- Null percentage warnings
- Data distribution insights
- Trend identification in time-series data

### 5.14 NEW: Autonomous multi-step agent with allowlist

A Junie-style autonomous agent that can plan and execute multi-step database
tasks:

```rust
pub struct AutonomousDatabaseAgent {
    thread: WeakEntity<Thread>,
    connection_manager: Entity<ConnectionManager>,
    allowlist: ActionAllowlist,
    plan: Vec<AgentStep>,
    current_step: usize,
}

pub struct ActionAllowlist {
    pub allow_select: bool,        // Default: true
    pub allow_insert: bool,        // Default: false
    pub allow_update: bool,        // Default: false
    pub allow_delete: bool,        // Default: false
    pub allow_ddl: bool,           // Default: false
    pub allow_explain: bool,       // Default: true
    pub max_rows_affected: usize,  // Default: 100
    pub require_confirmation: bool, // Default: true for write ops
}
```

Workflow:
1. User: "Why is the orders query slow?"
2. Agent reads schema (tool: `describe_database_object`)
3. Agent runs EXPLAIN (tool: `explain_query`)
4. Agent identifies full table scan
5. Agent proposes CREATE INDEX and generates DDL
6. User approves > agent executes (tool: `modify_data`)

The allowlist controls which operations the agent can perform without
asking for permission.

---

## Phase 6 — Additional Drivers

### 6.1 PostgreSQL driver

Full implementation using `tokio-postgres` or `sqlx`. Supports:
- All PostgreSQL types including arrays, JSON, geometry
- LISTEN/NOTIFY for real-time updates
- Advisory locks
- pg_cancel_backend for cancellation

**Steps:**
1. Create `crates/database_core/src/drivers/postgres.rs`
2. Implement `DatabaseDriver` + `DatabaseConnection`
3. Implement type mapping from PostgreSQL OIDs to `DataType`
4. Implement cancel via separate connection + `pg_cancel_backend()`
5. Implement LISTEN/NOTIFY integration for live schema updates

```rust
// crates/database_core/src/drivers/postgres.rs

pub struct PostgresDriver;

impl DatabaseDriver for PostgresDriver {
    fn driver_type(&self) -> DriverType { DriverType::Postgres }

    fn supported_types(&self) -> Vec<DataType> {
        vec![
            DataType::Boolean, DataType::SmallInt, DataType::Integer,
            DataType::BigInt, DataType::Float, DataType::Double,
            DataType::Decimal { precision: None, scale: None },
            DataType::Varchar { max_length: None }, DataType::Text,
            DataType::Date, DataType::Time, DataType::Timestamp, DataType::TimestampTz,
            DataType::Uuid, DataType::Json, DataType::Jsonb, DataType::Xml,
            DataType::ByteArray, DataType::Array { element_type: Box::new(DataType::Text) },
            DataType::Point, DataType::Geometry,
        ]
    }

    async fn connect(&self, config: &ConnectionConfig) -> Result<Box<dyn DatabaseConnection>> {
        let host = config.host.as_deref().unwrap_or("localhost");
        let port = config.port.unwrap_or(5432);
        let database = config.database.as_deref().unwrap_or("postgres");
        let user = config.user.as_deref().unwrap_or("postgres");
        // Build connection string, apply SSL config, connect via SSH tunnel if configured
        todo!()
    }
}

pub struct PostgresConnection {
    client: tokio_postgres::Client,
    cancel_token: tokio_postgres::CancelToken,
    server_version: String,
}

#[async_trait]
impl DatabaseConnection for PostgresConnection {
    async fn cancel(&self) -> Result<()> {
        self.cancel_token.cancel_query(tokio_postgres::NoTls).await?;
        Ok(())
    }

    async fn introspect_names(&self) -> Result<Vec<SchemaObject>> {
        let rows = self.client.query(
            "SELECT schemaname, tablename, 'table' as type FROM pg_tables
             WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
             UNION ALL
             SELECT schemaname, viewname, 'view' FROM pg_views
             WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
             ORDER BY schemaname, tablename",
            &[],
        ).await?;
        // Map to SchemaObject vec
        todo!()
    }

    // ... other trait methods
}
```

### 6.2 MySQL / MariaDB driver

Implementation using `mysql_async` or `sqlx`. Supports:
- MySQL and MariaDB dialects
- KILL QUERY for cancellation
- Multiple result sets from stored procedures

**Steps:**
1. Create `crates/database_core/src/drivers/mysql.rs`
2. Implement `DatabaseDriver` + `DatabaseConnection`
3. Implement type mapping from MySQL types
4. Implement cancel via separate connection + `KILL QUERY`
5. Handle multiple result sets from procedures

```rust
// crates/database_core/src/drivers/mysql.rs

pub struct MysqlDriver;
pub struct MysqlConnection {
    pool: mysql_async::Pool,
    thread_id: u32,  // For KILL QUERY
}

#[async_trait]
impl DatabaseConnection for MysqlConnection {
    async fn cancel(&self) -> Result<()> {
        let mut conn = self.pool.get_conn().await?;
        conn.exec_drop(format!("KILL QUERY {}", self.thread_id), ()).await?;
        Ok(())
    }

    async fn introspect_names(&self) -> Result<Vec<SchemaObject>> {
        let mut conn = self.pool.get_conn().await?;
        let rows: Vec<(String, String)> = conn.exec(
            "SELECT TABLE_NAME, TABLE_TYPE FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = DATABASE()",
            (),
        ).await?;
        // Map to SchemaObject
        todo!()
    }
}
```

### 6.3 Microsoft SQL Server driver

Implementation using `tiberius`. Supports:
- Windows Authentication and SQL Authentication
- TDS protocol
- KILL for cancellation

**Steps:**
1. Create `crates/database_core/src/drivers/mssql.rs`
2. Implement `DatabaseDriver` + `DatabaseConnection`
3. Implement type mapping from SQL Server types
4. Support both Windows and SQL authentication modes

### 6.4 Cloud connectivity (AWS RDS, GCP Cloud SQL, Azure SQL)

- AWS RDS: IAM authentication, RDS Proxy support
- GCP Cloud SQL: Cloud SQL Auth Proxy integration
- Azure SQL: Azure AD authentication

### 6.5 DuckDB driver

For local analytics. Useful for querying Parquet, CSV, and JSON files
as SQL tables without a database server.

---

## Phase 7 — Advanced & Extensibility

### 7.1 Extension API (WASM) for community drivers

Since WASM cannot make direct TCP connections, community database drivers
use the **Agent Server** pattern:

```toml
# extension.toml
[agent_servers.oracle-driver]
name = "Oracle Database Driver"

[agent_servers.oracle-driver.targets.linux-x86_64]
archive = "https://github.com/example/zed-oracle/releases/download/v1.0/oracle-linux-x64.tar.gz"
cmd = "./oracle-bridge"
args = ["--stdio"]
```

The bridge binary communicates via stdio with Zed, handling the actual
database connection natively.

### 7.2 Charts / Data visualization

Integrate `plotters` crate for building charts from query results:
- Bar, Line, Scatter, Pie charts
- Configurable X/Y axis mapping
- Export to PNG
- Interactive tooltip on hover

### 7.3 Geo Viewer for spatial data

Visualize PostGIS/MySQL spatial data on a map view.
Support for: POINT, LINESTRING, POLYGON, MULTIPOLYGON.

### 7.4 Real-time collaboration (result sharing)

- Share query results with collab participants
- Collaborative SQL editing (like shared document editing)
- Follow mode in Database Explorer
- Integration with `crates/collab/` protocol

### 7.5 Bookmarks on DB objects

Mark favorite tables/views/connections with `F11`. Navigate via `Shift+F11`.

**NEW: Bookmarks on rows/cells** — In a data grid, bookmark a specific row
by its primary key. Bookmarks persist across sessions and can be navigated
even after re-executing the query.

### 7.6 Stored procedure execution with parameter prompts

- Execute stored procedures/functions from the Explorer
- Parameter dialog with type-aware inputs
- Multiple result set display
- Output parameter display

### 7.7 ER Diagram viewer

Visual entity-relationship diagram generated from introspected schema.

**NEW: Bidirectional navigation** — Diagram elements are fully navigable:
- Double-click table > opens data grid (`F4`)
- `Ctrl+B` > opens DDL in editor
- `Alt+Shift+B` > selects in Database Explorer
- `Ctrl+F6` > opens Modify Table dialog
- Selecting a table in Explorer highlights it in the diagram

### 7.8 Long-running query notifications

When a query exceeds a configurable threshold (default: 5s):
- Progress indicator in status bar with elapsed time
- Desktop notification when query completes (if editor not focused)
- Cancel button (`Escape` or click)
- Sound notification option

### 7.9 Data generation tools

Generate test data for tables:
- Faker-style generators (names, emails, addresses, dates, etc.)
- Configurable row count
- Respect FK constraints (generate parent rows first)
- Preview before insertion

### 7.10 Monitoring (server stats, active queries)

Dashboard panel showing:
- Active connections and their status
- Running queries with duration
- Server statistics (PostgreSQL: pg_stat_activity, MySQL: SHOW PROCESSLIST)
- Kill query from monitoring view

### 7.11 NEW: Create/Alter Table GUI dialog

Visual dialog for table design without writing DDL:
- Add/remove/reorder columns
- Set types, defaults, constraints
- Define primary key and indexes
- Add foreign keys (with autocomplete from other tables)
- Preview generated DDL before execution
- Support both CREATE TABLE and ALTER TABLE

### 7.12 NEW: Dedicated grid font (complete feature)

Full font configuration for the data grid:
- Font family picker (separate from editor font)
- Font size slider
- Preview in settings UI
- Applies only to data grid cells, not to the query editor

---

## Key Dependencies

```toml
[dependencies]
# Database drivers
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "mysql", "sqlite"] }
# OR individual drivers:
tokio-postgres = "0.7"       # PostgreSQL
mysql_async = "0.34"         # MySQL / MariaDB
tiberius = "0.12"            # MS SQL Server
duckdb = "1.0"               # DuckDB

# SSH tunneling
russh = "0.46"

# Export
csv = "1.3"
xlsxwriter = "0.6"          # Excel export
calamine = "0.26"            # Excel import

# Visualization (Phase 7)
plotters = "0.3"

# Data generation (Phase 7)
fake = "3.0"

# Already present in Zed
serde_json = "1.0"
rusqlite = "*"               # Via sqlez
schemars = "*"               # For AgentTool JSON schemas
```

---

## Feature Coverage Matrix

Complete mapping of every DataGrip feature to a plan phase.

### View Modes & Navigation

| DataGrip Feature | Phase | Item |
|---|---|---|
| Table mode (default grid) | 3 | 3.12 |
| Tree mode (expandable nodes) | 3 | 3.12 |
| Text mode (raw text) | 3 | 3.12 |
| Transpose mode (rows↔columns) | 3 | 3.12 |
| Column List search (Ctrl+F12) | 3 | 3.9 |
| Expand/Shrink Selection | 3 | 3.10 |
| Grid paging | 3 | 3.11 |
| Full-width in-editor results | 2 | 2.6 |
| In-editor results | 2 | 2.6 |
| Independent split editors | 2 | 2.6 |
| Quick documentation on cells | 3 | 3.22 |
| Record View | 3 | 3.5 |
| Bookmarks on objects | 7 | 7.5 |
| Bookmarks on rows/cells | 7 | 7.5 |

### Data Editing

| DataGrip Feature | Phase | Item |
|---|---|---|
| Inline cell editing | 3 | 3.1 |
| Multi-cell editing | 3 | 3.4 |
| Value Editor (JSON/XML/images) | 3 | 3.3 |
| Boolean toggle (Space, t/f/n) | 3 | 3.2 |
| Editable JOIN result sets | 3 | 3.1 |
| Local change tracking (color-coded) | 2 | 2.4 |
| DML Preview before submit | 3 | 3.16 |
| Batch submission | 3 | 3.16 |
| Add/Delete/Clone rows | 3 | 3.15 |
| Create table from grid | 4 | 4.9 |
| Copy table to another datasource | 4 | 4.10 |
| Undo/Redo transactional | 3 | 3.17 |
| Value completion in cells | 3 | 3.21 |
| LOB size hint (interactive) | 3 | 3.3 |
| Quick Actions floating toolbar | 3 | 3.20 |

### Filtering & Sorting

| DataGrip Feature | Phase | Item |
|---|---|---|
| Text search (Ctrl+F) | 3 | 3.7 |
| Local column filters (checkboxes) | 3 | 3.7 |
| Clear Local Filter For All Columns | 3 | 3.7 |
| Quick filters (context menu) | 3 | 3.8 |
| WHERE clause field | 3 | 3.8 |
| Server-side vs client-side filtering | 3 | 3.8 |
| Column header sorting | 3 | 3.6 |
| Multi-column stacking (Alt+click) | 3 | 3.6 |
| ORDER BY clause field | 3 | 3.6 |
| Server-side vs client-side sorting | 3 | 3.6 |

### Export / Import

| DataGrip Feature | Phase | Item |
|---|---|---|
| CSV/TSV/JSON/XML/HTML/Markdown export | 4 | 4.1 |
| Excel export | 4 | 4.1 |
| SQL export (DML/DDL) | 4 | 4.1 |
| Custom DSV | 4 | 4.1 |
| Custom extractor scripts | 4 | 4.3 |
| Clipboard export (active format) | 4 | 4.2 |
| pg_dump / mysqldump | 4 | 4.6 |
| DDL/structure export (SQL Generator) | 4 | 4.12 |
| CSV import wizard | 4 | 4.4 |
| Paste from Excel/clipboard | 4 | 4.11 |
| Edit CSV as table | 4 | 4.5 |
| Execute to file | 2 | 2.10 |

### Schema Browsing

| DataGrip Feature | Phase | Item |
|---|---|---|
| Hierarchical tree | 1 | 1.6 |
| Search Everywhere | 1 | 1.6 |
| Object filtering by pattern | 1 | 1.6 |
| Color coding per data source | 1 | 1.5 |
| Introspection by levels | 1 | 1.7 |
| Smart refresh | 1 | 1.7 |
| Schema comparison + migration DDL | 4 | 4.8 |
| Show All Namespaces | 1 | 1.6 |
| Create/Modify/Drop via GUI | 7 | 7.11 |

### Query Execution

| DataGrip Feature | Phase | Item |
|---|---|---|
| Multiple query consoles | 2 | 2.1 |
| Query history per console | 2 | 2.7 |
| Read-only mode | 1 | 1.10 |
| Execute to file | 2 | 2.10 |
| Multi-tab results | 2 | 2.5 |
| Tab pinning | 2 | 2.5 |
| Tab naming via comments | 2 | 2.5 |
| Explain Plan (table) | 2 | 2.9 |
| Explain Plan (raw text) | 2 | 2.9 |
| Explain Plan (diagram) | 2 | 2.9 |
| Result set comparison | 4 | 4.7 |

### Cell Editors by Type

| DataGrip Feature | Phase | Item |
|---|---|---|
| Text/VARCHAR editor | 3 | 3.1 |
| JSON editor (pretty-print, tree) | 3 | 3.3 |
| XML editor (formatted view) | 3 | 3.3 |
| Boolean editor (toggle/shortcuts) | 3 | 3.2 |
| BLOB/Binary (hex display, UUID) | 3 | 3.3 |
| Image viewer | 3 | 3.3 |
| LOB with size hint | 3 | 3.3 |
| Spatial/Geo viewer | 7 | 7.3 |
| Date/Time with format awareness | 3 | 3.1 |
| NULL indicator | 3 | 3.18 |
| Change Display Type per column | 3 | 3.18 |

### Aggregation

| DataGrip Feature | Phase | Item |
|---|---|---|
| Aggregate View (9 built-in) | 3 | 3.14 |
| Custom aggregator scripts | 3 | 3.14 |
| Dynamic updates on selection | 3 | 3.14 |

### Foreign Key Navigation

| DataGrip Feature | Phase | Item |
|---|---|---|
| Navigate to referenced row | 3 | 3.13 |
| Find usages (reverse FK) | 3 | 3.13 |
| Virtual Foreign Keys | 3 | 3.13 |
| JOIN clause generation | 2 | 2.2 |

### Data Comparison

| DataGrip Feature | Phase | Item |
|---|---|---|
| Compare two tables/views | 4 | 4.7 |
| Tolerance parameter | 4 | 4.7 |
| Column exclusion | 4 | 4.7 |
| Detect Column Insertion | 4 | 4.7 |

### AI / Agentic

| DataGrip Feature | Phase | Item |
|---|---|---|
| Natural language to SQL | 5 | 5.8 |
| Query optimization (AI) | 5 | 5.10 |
| Query explanation (AI) | 5 | 5.9 |
| Error explanations with schema | 5 | 5.11 |
| DB object context in AI chat | 5 | 5.6 |
| Execution plan analysis (AI) | 5 | 5.10 |
| Cloud-based code completion | 2 | 2.2 |
| Autonomous agent (Junie-style) | 5 | 5.14 |

### Additional

| DataGrip Feature | Phase | Item |
|---|---|---|
| Grid heatmaps | 3 | 3.19 |
| Charts/Data visualization | 7 | 7.2 |
| Dedicated data font | 7 | 7.12 |
| Cloud DB connectivity | 6 | 6.4 |
| Stored procedure execution | 7 | 7.6 |
| Server monitoring | 7 | 7.10 |
| Long-running query notifications | 7 | 7.8 |

---

**Coverage: 100% of identified DataGrip features are mapped to a plan phase.**
