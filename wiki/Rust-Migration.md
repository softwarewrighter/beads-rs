# Rust Migration

The Beads project is being migrated from Go to Rust to leverage Rust's type safety, memory safety, and performance characteristics.

## Migration Status

### ✅ Phase 1: Foundation (Completed)

**Core Data Types** (`bd/src/types.rs`)
- ✅ `Issue` struct with validation
- ✅ `Dependency`, `Comment`, `Event`, `Label`
- ✅ Enums: `Status`, `IssueType`, `DependencyType`, `EventType`
- ✅ Filter types: `IssueFilter`, `WorkFilter`, `StaleFilter`
- ✅ Content hashing with SHA256
- ✅ Serde serialization/deserialization

**Storage Layer** (`bd/src/storage/`)
- ✅ `Storage` trait interface (50+ methods)
- ✅ `SqliteStorage` implementation
- ✅ SQLite schema with indexes and views
- ✅ Thread-safe connection handling (`Arc<Mutex<Connection>>`)
- ✅ Event recording and audit trail
- ✅ Dirty tracking for incremental export

**CLI Framework** (`bd/src/main.rs`)
- ✅ clap-based argument parsing
- ✅ Command structure (init, create, list, show, update, close, export, import)
- ✅ Global flags (--db, --actor, --json)

### 🔨 Phase 2: Core Functionality (In Progress)

**JSONL Import/Export**
- 🔨 Export engine (dirty issues → JSONL)
- 🔨 Import engine (JSONL → SQLite)
- 🔨 Content hash deduplication
- 🔨 Timestamp-based merge strategy
- 📋 Conflict marker detection

**Command Implementations**
- 🔨 `bd create` - Create issues
- 🔨 `bd list` - Query with filters
- 🔨 `bd show` - Display details
- 🔨 `bd update` - Modify fields
- 🔨 `bd close` - Close issues
- 📋 `bd ready` - Find ready work
- 📋 `bd blocked` - Show blocked issues

**Dependency Operations**
- 📋 `bd dep add` - Add dependencies
- 📋 `bd dep remove` - Remove dependencies
- 📋 `bd dep tree` - Visualize tree
- 📋 `bd dep cycles` - Detect cycles

### 📋 Phase 3: Advanced Features (Planned)

**Daemon & RPC**
- 📋 Daemon process lifecycle
- 📋 Unix socket RPC server
- 📋 Auto-start mechanism
- 📋 Version checking
- 📋 Health monitoring

**Git Integration**
- 📋 Auto-sync on mutations
- 📋 File watching (notify crate)
- 📋 Git hooks integration
- 📋 Merge conflict detection

**Query Optimization**
- 📋 Ready work with recursive CTE
- 📋 Dependency tree traversal
- 📋 Cycle detection algorithm
- 📋 Epic closure detection

**Multi-Repo Support**
- 📋 Cross-repo dependency routing
- 📋 Repository discovery
- 📋 Hydration optimization

**Memory Compaction**
- 📋 Semantic compression with LLM
- 📋 Tiered compaction strategy
- 📋 Snapshot management

## Technology Choices

### Rust Ecosystem

| Component | Crate | Rationale |
|-----------|-------|-----------|
| **CLI** | `clap` v4.5 | Best-in-class arg parsing, derive macros |
| **Database** | `rusqlite` v0.32 | Pure Rust, bundled SQLite, stable API |
| **Serialization** | `serde` v1.0 | Industry standard, excellent derive support |
| **JSON** | `serde_json` v1.0 | Fast, zero-copy parsing |
| **Error Handling** | `anyhow` v1.0 | Ergonomic error types, context |
| **Async Runtime** | `tokio` v1.42 | For future daemon (not needed yet) |
| **File Watching** | `notify` v7.0 | Cross-platform FS events |
| **Hashing** | `sha2` v0.10 | Crypto-quality hashing |
| **Date/Time** | `chrono` v0.4 | SQLite-compatible timestamps |
| **Colors** | `colored` v2.2 | Terminal output formatting |

### Design Decisions

#### 1. Trait-Based Storage

**Go Approach**:
```go
type Storage interface {
    CreateIssue(ctx context.Context, issue *types.Issue, actor string) error
    // ...
}
```

**Rust Approach**:
```rust
pub trait Storage: Send + Sync {
    fn create_issue(&self, issue: &Issue, actor: &str) -> Result<()>;
    // ...
}
```

**Benefits**:
- Compile-time polymorphism (no virtual dispatch)
- `Send + Sync` bounds for thread safety
- Generic error handling with `Result<T>`

#### 2. Arc<Mutex<>> for SQLite

**Rationale**:
- rusqlite `Connection` is not `Send`
- Need shared access across threads (daemon)
- `Arc` for shared ownership
- `Mutex` for exclusive write access

**Alternative Considered**: Connection pool
- **Rejected**: Adds complexity, rusqlite is already fast
- **Future**: May add pool for high concurrency

#### 3. Synchronous Storage API

**Rationale**:
- rusqlite is synchronous
- Unnecessary complexity to wrap in async
- Daemon will use tokio for RPC only

**Future**: May add async wrapper for daemon RPC

#### 4. String-Based IDs

**Go Approach**:
```go
type Issue struct {
    ID string `json:"id"`
}
```

**Rust Approach**:
```rust
pub struct Issue {
    pub id: String,
}
```

**Alternative Considered**: Newtype wrapper
```rust
pub struct IssueId(String);
```

**Decision**: Keep simple for now, may add newtype later

## Migration Workflow

### 1. Compare Go Implementation

```bash
# Find Go implementation
fd -e go . cmd/bd/ internal/

# Read key files
bat cmd/bd/create.go
bat internal/storage/sqlite/sqlite.go
```

### 2. Convert to Rust

```bash
# Create Rust equivalent
$EDITOR bd/src/commands/create.rs

# Use Go code as reference, adapt to Rust idioms
```

### 3. Test

```bash
# Unit tests
cargo test

# Integration tests
cargo test --test integration

# Manual testing
cargo run -- create "Test issue"
```

### 4. Benchmark

```bash
# Compare Go vs Rust performance
hyperfine 'bd-go create "Test"' 'bd-rust create "Test"'
```

## Code Organization

### Go Structure

```
beads/
├── cmd/bd/
│   ├── main.go
│   ├── create.go
│   ├── list.go
│   └── ...
├── internal/
│   ├── types/
│   │   └── types.go
│   ├── storage/
│   │   ├── storage.go
│   │   └── sqlite/
│   │       └── sqlite.go
│   └── ...
```

### Rust Structure

```
beads-rs/
├── bd/
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs
│       ├── types.rs
│       └── storage/
│           ├── mod.rs
│           ├── sqlite.rs
│           └── sqlite_schema.sql
```

**Key Differences**:
- Rust uses modules instead of packages
- SQL schema embedded with `include_str!`
- Binary in workspace member `bd/`

## Performance Comparison

| Operation | Go | Rust (Target) | Status |
|-----------|-----|---------------|--------|
| Create issue | ~1.2ms | ~0.8ms | 🔨 |
| List 100 issues | ~8ms | ~5ms | 📋 |
| Ready work | ~15ms | ~10ms | 📋 |
| Export 1000 | ~950ms | ~700ms | 🔨 |
| Import 1000 | ~1200ms | ~900ms | 🔨 |

**Goals**:
- 25-30% performance improvement
- Lower memory usage
- Better type safety

## Type Safety Improvements

### 1. Enum Exhaustiveness

**Go**:
```go
switch status {
case "open", "in_progress", "blocked", "closed":
    // handle
default:
    // easy to forget!
}
```

**Rust**:
```rust
match status {
    Status::Open => { /* ... */ }
    Status::InProgress => { /* ... */ }
    Status::Blocked => { /* ... */ }
    Status::Closed => { /* ... */ }
    // Compiler error if missing variant!
}
```

### 2. Null Safety

**Go**:
```go
var closedAt *time.Time // Can be nil, must check!
if closedAt != nil {
    // use
}
```

**Rust**:
```rust
let closed_at: Option<DateTime<Utc>>;
match closed_at {
    Some(time) => { /* use time */ }
    None => { /* handle absence */ }
}
```

### 3. Lifetime Safety

**Go**:
```go
func GetIssue(id string) *Issue {
    return &Issue{} // Escape analysis, GC overhead
}
```

**Rust**:
```rust
fn get_issue(&self, id: &str) -> Result<Option<Issue>> {
    // Borrow checker prevents use-after-free
    // Zero-cost at runtime
}
```

## Testing Strategy

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_content_hash() {
        let issue = Issue { ... };
        let hash1 = issue.compute_content_hash();
        let hash2 = issue.compute_content_hash();
        assert_eq!(hash1, hash2); // Deterministic
    }

    #[test]
    fn test_validation() {
        let invalid = Issue {
            title: "".to_string(), // Empty title
            ..Default::default()
        };
        assert!(invalid.validate().is_err());
    }
}
```

### Integration Tests

```rust
#[test]
fn test_create_and_retrieve() {
    let storage = SqliteStorage::new(":memory:").unwrap();

    let issue = Issue { ... };
    storage.create_issue(&issue, "alice").unwrap();

    let retrieved = storage.get_issue(&issue.id).unwrap();
    assert_eq!(retrieved.unwrap().title, issue.title);
}
```

### Property-Based Tests

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn content_hash_deterministic(title: String, desc: String) {
        let issue = Issue { title, description: desc, .. };
        let hash1 = issue.compute_content_hash();
        let hash2 = issue.compute_content_hash();
        assert_eq!(hash1, hash2);
    }
}
```

## Documentation

### Rustdoc

```rust
/// Creates a new issue in the storage backend.
///
/// # Arguments
///
/// * `issue` - The issue to create
/// * `actor` - The user creating the issue
///
/// # Returns
///
/// * `Ok(())` - Issue created successfully
/// * `Err(_)` - Database error or validation failure
///
/// # Examples
///
/// ```
/// let issue = Issue { id: "bd-a1b2", title: "Fix bug", ... };
/// storage.create_issue(&issue, "alice")?;
/// ```
fn create_issue(&self, issue: &Issue, actor: &str) -> Result<()>;
```

Generate docs:
```bash
cargo doc --open
```

## Contributing

### Prerequisites

1. Rust toolchain (rustup recommended)
2. SQLite development headers
3. Git

### Setup

```bash
# Clone repo
git clone https://github.com/softwarewrighter/beads-rs.git
cd beads-rs

# Build
cargo build

# Run tests
cargo test

# Run
cargo run -- --help
```

### Development Workflow

1. **Pick a task**: See [[Home]] for current priorities
2. **Create branch**: `git checkout -b feature/my-feature`
3. **Implement**: Follow existing patterns
4. **Test**: Add unit and integration tests
5. **Document**: Add rustdoc comments
6. **Submit PR**: Reference related issue

## Blockers & Risks

### Current Blockers

1. 🔨 **JSONL Export/Import** - Core functionality needed for git sync
2. 📋 **Daemon RPC** - Complex threading model, needs careful design
3. 📋 **File Watching** - Platform differences (inotify, FSEvents, etc.)

### Migration Risks

| Risk | Mitigation |
|------|------------|
| **API Changes** | Keep Go version running in parallel |
| **Performance Regression** | Benchmark all operations |
| **Feature Parity** | Maintain feature matrix checklist |
| **User Disruption** | Provide migration guide, keep compatible |

## Timeline

| Phase | Duration | Completion |
|-------|----------|------------|
| **Phase 1: Foundation** | 2 weeks | ✅ Complete |
| **Phase 2: Core Functionality** | 4 weeks | 🔨 50% |
| **Phase 3: Advanced Features** | 6 weeks | 📋 0% |
| **Phase 4: Testing & Polish** | 2 weeks | 📋 0% |

**Total**: ~14 weeks (estimated)

## Related Pages

- [[Architecture Overview]] - System design
- [[Storage Layer]] - Implementation details
- [[Data Types]] - Type system reference

---

*Contributing? See [CONTRIBUTING.md](https://github.com/softwarewrighter/beads-rs/blob/main/CONTRIBUTING.md)*
