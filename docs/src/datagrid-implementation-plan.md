# DataGrid Implementation Plan — Full DataGrip Feature Parity for Zed

This document describes the complete implementation plan for adding a DataGrip-equivalent
data grid and database IDE experience to Zed, with agentic AI integration.

## Table of Contents

- [Existing Foundation](#existing-foundation)
- [Architecture Overview](#architecture-overview)
- [Folder Structure Conventions](#folder-structure-conventions)
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
