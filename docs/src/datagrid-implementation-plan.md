# DataGrid Implementation Plan — Full DataGrip Feature Parity for Zed

This document describes the complete implementation plan for adding a DataGrip-equivalent
data grid and database IDE experience to Zed, with agentic AI integration.

## Table of Contents

- [Existing Foundation](#existing-foundation)
- [Architecture Overview](#architecture-overview)
- [Folder Structure Conventions](#folder-structure-conventions)
- [UI/Interface Specification](#uiinterface-specification)
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

### 1. Overall Layout and Navigation Flow

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

- All interactive elements have ARIA-equivalent focus management via GPUI's `FocusHandle`
- Grid cells are navigable via keyboard (never mouse-only interactions)
- Color is never the sole indicator — always paired with icon or text
  (e.g., NULL is italic + muted, not just grey; errors have icon + border + text)
- Screen reader support: column headers announce type, sort state; cells announce value + column name
- High contrast theme support: all custom colors use theme tokens where possible,
  fall back to hardcoded hsla only for data-state backgrounds (which use low opacity
  overlays on top of theme background)

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

## Phase 0 — Infrastructure

Foundation work required before any feature development.

### 0.1 DatabaseSettings type + registration

Register a new settings type following Zed's hierarchical settings system.
Supports both global (`~/.config/zed/settings.json`) and project-level
(`.zed/settings.json`) configuration.

```rust
#[derive(Clone, Debug, Serialize, Deserialize, JsonSchema, RegisterSetting)]
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
```

### 0.2 Credential storage via CredentialsProvider

Database passwords stored securely via platform keychain, following the pattern
in `crates/language_model/src/api_key.rs`.

```rust
pub struct DatabaseCredential {
    pub connection_id: ConnectionId,
    url: SharedString,               // "zed-db://<connection-name>" as keychain key
    load_status: CredentialLoadStatus,
}
```

Priority: env var `DB_PASSWORD_<CONNECTION_NAME>` > keychain > prompt user.

### 0.3 DatabaseError types + notification integration

```rust
pub enum DatabaseError {
    ConnectionFailed { host: String, port: u16, cause: String },
    AuthenticationFailed { user: String },
    QueryTimeout { sql_preview: String, duration: Duration },
    QueryFailed { sql_preview: String, db_error: String, position: Option<usize> },
    ConnectionLost { was_in_transaction: bool },
    SslError { cause: String },
    SshTunnelFailed { cause: String },
    LobSizeExceeded { actual: usize, limit: usize, column: String },
}
```

Each error displays in Zed's notification bar with contextual actions
("Retry", "Edit Connection", "Show Details", "Load Full Content" for LOB).

### 0.4 Database icon in Zed icon system

Add `IconName::Database` and related icons (Table, Column, Key, Index, View,
Function, Schema, Connection) to `crates/ui/src/components/icon.rs`.

### 0.5 Test infrastructure

- Mock `DatabaseConnection` trait implementation for unit tests
- SQLite in-memory connections for integration tests
- GPUI test helpers for panel and grid visual tests
- `cx.background_executor().timer()` for async test timeouts (per CLAUDE.md)

---

## Phase 1 — Connection & Schema

### 1.1 DatabaseDriver trait + SQLite driver

Implement the core trait and the SQLite driver first (simplest for testing).
Use `rusqlite` (already a dependency via `sqlez`).

### 1.2 ConnectionManager entity with pooling

```rust
pub struct ConnectionManager {
    connections: Vec<Entity<DatabaseConnectionState>>,
    active_connection: Option<ConnectionId>,
    _subscriptions: Vec<Subscription>,
}
```

Connection pooling with configurable max connections, idle timeout,
and automatic reconnection with exponential backoff (1s, 2s, 4s, 8s, max 30s).

### 1.3 SSH tunnel support

Use `russh` or `async-ssh2-lite` for SSH tunneling. Configuration:

```rust
pub struct SshTunnelConfig {
    pub host: String,
    pub port: u16,
    pub username: String,
    pub auth: SshAuth,            // Password, PrivateKey, or Agent
    pub local_port: Option<u16>,  // Auto-assign if None
}
```

### 1.4 SSL/TLS configuration

Certificate management: CA cert, client cert, verify-full vs verify-ca mode.

### 1.5 Connection dialog UI

`Entity<ConnectionDialog>` — modal dialog for configuring a data source:
host, port, database, user, password, SSL, SSH, test connection button.

**NEW: Color coding per connection** — Each connection can be assigned a color
(red for production, green for dev, blue for staging). This color tints:
- The Database Explorer header for that connection
- Query editor tab borders
- Result grid tab borders
- Status bar indicator

This prevents accidentally running queries on production.

### 1.6 Database Explorer panel

Tree panel registered as a Zed Panel with `activation_priority: 15`.

Features:
- Hierarchical tree: Data Source > Database > Schema > Tables/Views/Functions/Sequences
- Each node shows icon + name + object count badge
- Double-click table opens data grid; double-click view opens DDL
- **NEW: Object filtering by pattern** — Right-click a schema node >
  "Filter Objects" > enter regex (e.g., `^(?!_tmp).*`) to hide matching objects

### 1.7 Introspection by levels with cache

- Level 1: Object names only (instant, for large databases)
- Level 2: Columns, types, constraints (default)
- Level 3: Full DDL source code

Smart refresh: after DDL execution, only re-introspect affected objects.
Schema cache stored in-memory with `Arc<RwLock<SchemaCache>>`.

### 1.8 Workspace serialization

Persist panel state across Zed restarts using KVP store (same pattern as
ProjectPanel):

```rust
#[derive(Serialize, Deserialize)]
struct SerializedDatabasePanel {
    width: Option<f32>,
    active_connection_id: Option<String>,
    expanded_nodes: Vec<SchemaObjectPath>,
    open_query_tabs: Vec<SerializedQueryTab>,
    pinned_result_tabs: Vec<SerializedResultTab>,
}
```

### 1.9 Auto-reconnection with backoff

`ConnectionWatchdog` monitors connection health with periodic pings.
On connection loss: notify user, retry with exponential backoff,
preserve pending changes if in a transaction.

### 1.10 Read-only mode per connection

Toggle per data source that prevents any INSERT/UPDATE/DELETE/DDL.
Visual indicator in the connection's color bar.

---

## Phase 2 — Query Editor & Execution

### 2.1 SQL buffer with dialect detection

Specialized buffer with SQL language activated, bound to a connection/schema.
Uses Zed's standard editor infrastructure (multicursor, vim mode, etc.).

### 2.2 SQL LSP integration

Integrate an SQL Language Server (`sqls` or `sql-language-server`) with
injected schema metadata for accurate completion.

Features: completion, formatting, diagnostics (syntax errors, unknown tables).

### 2.3 Query execution (background_spawn)

Execute queries on `cx.background_spawn()`. Return results via channel to
foreground for UI update.

### 2.4 Results in extended DataTable

Build on existing `crates/ui/src/components/data_table.rs` with:
- Type-aware cell rendering (numbers right-aligned, strings left-aligned)
- Color-coded pending changes (green=insert, yellow=update, red=delete)
- Configurable page size with paging control at bottom

### 2.5 Multi-tab results + tab pinning

Each query execution opens a result tab. Tabs can be pinned to prevent
replacement.

**NEW: Tab naming via SQL comments** — A comment like `-- @name Monthly Sales`
above a query names the result tab "Monthly Sales" instead of the default.
The prefix keyword (`@name`) is configurable in `DatabaseSettings.tab_naming_prefix`.

### 2.6 In-editor results (inline below query)

Display results directly below the query in the SQL buffer (like DataGrip's
in-editor results mode). Full-width grid that adjusts to editor width.

**NEW: Independent split data grids** — When the editor is split, each split
gets its own independent data grid instance with separate filter/sort state.
This is managed by giving each split its own `Entity<ResultGrid>`.

### 2.7 Query history per connection

Save all executed queries with timestamp, duration, row count, and error status.
Stored in the `db` crate's KVP store, scoped by connection ID.
Searchable via a history panel.

### 2.8 Query cancellation

Cancel a running query via:
- PostgreSQL: `pg_cancel_backend(pid)`
- MySQL: `KILL QUERY <id>`
- SQLite: `sqlite3_interrupt()`

Cancel button in the status bar and `Escape` keybinding.

### 2.9 Explain Plan (text + diagram)

Two views:
1. **Table view**: Operations, object names, row estimates, costs
2. **Diagram view**: Graphical execution plan with node graph

### 2.10 Execute to file

Run a query and write results directly to a file in a chosen format,
without loading them into the grid. Useful for large exports.

### 2.11 SQL formatting

Integrate a SQL formatter (via LSP or dedicated formatter like `sqlformat`).
Accessible via `Ctrl+Shift+I` or "Format Document" action.

### 2.12 NEW: SQL Generator (DDL export)

Generate the complete DDL for an entire database or schema in one click.
Right-click a schema in the Database Explorer > "Generate DDL" > choose output
(clipboard, new buffer, or file).

---

## Phase 3 — Interactive Data Grid

### 3.1 Inline editing + type-aware cell editors

Double-click a cell to edit. Cell editor adapts to column type:
- Text: inline text input
- Integer/Float: numeric input with validation
- Date/Time: date picker or formatted input
- JSON: opens Value Editor with syntax highlighting
- Boolean: toggle (see 3.2)
- BLOB: hex editor or "Open in Value Editor"

### 3.2 Boolean toggle

`Space` key toggles boolean values. Single-key shortcuts:
- `t` → true, `f` → false, `n` → null, `d` → default
- Dropdown of possible values on `Enter`

### 3.3 Value Editor panel

Side panel for editing large or complex values:
- JSON/XML: pretty-printed with syntax highlighting, collapsible nodes
- Text: multiline editor
- Images: preview display (PNG, JPEG, etc.)
- Hex: binary data viewer

**NEW: LOB size hint** — When a value exceeds `max_lob_size`, display an
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

### 3.14 Aggregate View

Select multiple cells > panel on right shows:
- Count, Sum, Average, Min, Max (built-in, 9 aggregators)
- Custom aggregator scripts (Lua or Rhai, stored in config dir)
- Dynamic update as selection changes

### 3.15 CRUD operations (add/delete/clone row)

- Add row: inserts empty row at bottom with default values
- Delete row: marks row for deletion (red highlight)
- Clone row: duplicates an existing row for easy insertion
- All changes are local until submitted

### 3.16 DML Preview before commit

Before submitting changes, display a dialog showing the exact SQL that will
be executed (INSERT/UPDATE/DELETE statements). User can review and confirm.

### 3.17 Undo/Redo transactional

```rust
pub struct DataEditHistory {
    operations: Vec<DataOperation>,
    cursor: usize,
}

pub enum DataOperation {
    UpdateCell { row: usize, col: usize, old_value: Value, new_value: Value },
    InsertRow { row: usize, data: Vec<Value> },
    DeleteRow { row: usize, data: Vec<Value> },
    CloneRow { source_row: usize, new_row: usize },
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

Export from any data grid (table, query result, CSV file):

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

### 4.2 Clipboard export

`Ctrl+C` copies selection in the currently active extractor format.
Configurable default format (CSV, JSON, SQL INSERT, Markdown).

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

### 5.3 AgentTool: list_database_objects

Lists tables, views, functions in a schema. Supports filtering by type.

### 5.4 AgentTool: explain_query

Runs EXPLAIN on a query and returns the execution plan.

### 5.5 AgentTool: modify_data (with confirmation)

Executes INSERT/UPDATE/DELETE. Requires user confirmation (`ToolKind::Write`).
Shows DML preview before execution.

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

### 6.2 MySQL / MariaDB driver

Implementation using `mysql_async` or `sqlx`. Supports:
- MySQL and MariaDB dialects
- KILL QUERY for cancellation
- Multiple result sets from stored procedures

### 6.3 Microsoft SQL Server driver

Implementation using `tiberius`. Supports:
- Windows Authentication and SQL Authentication
- TDS protocol
- KILL for cancellation

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
