# Persistence options for Fenro: SwiftData vs GRDB vs Core Data

Research for the wayfinder ticket "Research persistence options". Facts were gathered on 2026-09-12 from Apple documentation and WWDC sessions (SwiftData, Core Data) and from the GRDB repository and its DocC guides. Anything not confirmed by a primary source is marked UNVERIFIED. This document surfaces facts and a recommendation; the decision is made in the ticket "Choose the persistence layer".

## Summary

- **SwiftData** is Apple's current framework, built on Core Data's SQLite store. On macOS 26 it has unique constraints, versioned migrations, background access through `@ModelActor`, and a History API useful for sync. Its weak spots for Fenro are: no raw SQL or aggregates, cross-context observation that is documented as buggy on macOS 26 and only properly fixed in macOS 27 (`ResultsObserver`), and predicates limited to one model root.
- **GRDB** is a mature third-party SQLite toolkit, fully Swift 6, with explicit SQL migrations, a one-writer-many-readers `DatabasePool`, and `ValueObservation` that reliably notifies the UI of any committed change in the same process, including writes from a background actor such as an in-process MCP server. It costs one dependency and asks you to write SQL for the schema, which the owner already knows.
- **Core Data** is not deprecated but is the older API. It shares the store format with SwiftData and can coexist with it. On its own it adds Objective-C era ceremony with no advantage over the other two for a new app.
- **Recommendation**: GRDB, unless the owner values learning Apple's newest framework more than the in-process MCP observation story. Conditions that flip the recommendation are listed at the end.

## 1. SwiftData on macOS 26

### Maturity

- Available since macOS 14 (WWDC 2023). Apple's own "SwiftData updates" page lists yearly additions: June 2024 added `#Index`, `#Unique`, unidirectional relationships, history fetching and custom `DataStore`; June 2025 added class inheritance; June 2026 (macOS 27) added sectioned `@Query`, `@Attribute(.codable)`, `ResultsObserver` and `HistoryObserver`. Source: https://developer.apple.com/documentation/updates/swiftdata
- The macOS 27 additions are stamped iOS 27 / macOS 27, so they are **not available on macOS 26**, which is the deployment target. Sources: https://developer.apple.com/documentation/swiftdata/resultsobserver and https://developer.apple.com/documentation/swiftdata/historyobserver
- Xcode 16 through 27 release notes contain no SwiftData "known issues" sections; the only listed fixes were in Xcode 15.x. Source: https://developer.apple.com/documentation/xcode-release-notes/xcode-26-release-notes

### Unique constraints

- `@Attribute(.unique)` on a primitive or a to-one relationship makes inserts collide into an upsert. `#Unique<T>([\.a], [\.b, \.c])` (macOS 15+) supports compound constraints. Sources: https://developer.apple.com/documentation/swiftdata/unique(_:) and WWDC23 session 10195.
- Swift idiom note: an "upsert" means an insert that becomes an update when it collides with an existing row. For Fenro this matters for Keys (FEN-12 must be unique per Project).
- UNVERIFIED: community forum reports of crashes on `save()` instead of a silent upsert in some iOS 17 builds. https://developer.apple.com/forums/thread/756099

### Migrations

- `VersionedSchema`, `SchemaMigrationPlan` and `MigrationStage.lightweight` / `.custom(willMigrate:didMigrate:)` exist since macOS 14. `ModelContainer` migrates automatically when the change is lightweight. Source: https://developer.apple.com/documentation/swiftdata/migrationstage
- Apple gives no exhaustive list of what counts as lightweight for SwiftData. Known: renames with `originalName` and delete-rule changes are lightweight; adding a unique constraint is not (WWDC23 10195); adding subclasses is (WWDC25 291).
- `.codable` attributes (macOS 27 only) are opaque blobs that never trigger migrations but cannot be used in predicates.

### Background access and concurrency

- `ModelContext` is not `Sendable`. `ModelContainer` is `Sendable`. `PersistentIdentifier` is `Sendable`. The supported pattern for non-UI work is an actor annotated with `@ModelActor`, which owns its own context; pass identifiers or plain value types between actors, never model objects. Sources: https://developer.apple.com/documentation/swiftdata/modelactor() and the WWDC26 SwiftData Group Lab (8017).
- Swift idiom note: an actor is a type whose state can only be touched from one task at a time; `@ModelActor` generates the boilerplate so the actor holds a private `ModelContext`.
- Background contexts do not autosave; `autosaveEnabled` is true only for `mainContext`. Source: https://developer.apple.com/documentation/swiftdata/modelcontext/autosaveenabled

### Cross-context observation, the important gap

- Apple distinguishes "local" changes (same container, same context) from "remote" changes (another context or process). A DTS engineer stated that `@Query` is supposed to refresh on remote changes, and called non-refresh after a `@ModelActor` write "a bug on SwiftData + SwiftUI side", with a workaround of observing `NSManagedObjectContextDidSave` and bumping a view `id`. Sources: https://developer.apple.com/forums/thread/791794 and https://developer.apple.com/forums/thread/759364?page=2
- Community feedback reports (FB12689036, FB14750050) say this worked in iOS 17 and regressed in iOS 18 betas. UNVERIFIED for macOS 26 specifically; no Apple statement either way.
- macOS 27 adds `ResultsObserver`, explicitly documented to respond to local, remote-context and cross-process changes. That is the proper fix and it is one OS version away from the deployment target.
- **Why this matters for Fenro**: the MCP server will write from a background actor while the Dashboard is on screen. On macOS 26 the UI refresh after an agent edit relies on behaviour Apple's own engineers call buggy, with a workaround.

### Query expressiveness

- `#Predicate` supports comparisons, boolean logic, optionals, string operations (`contains`, `localizedStandardContains`), and sequence operations across relationships. No control flow, no nested declarations. Source: https://developer.apple.com/documentation/foundation/predicate
- Predicates are typed to one model root. Aggregates beyond `fetchCount` are a "known gap" per the Group Lab; the suggested workaround is dropping into Core Data on the same store.
- `FetchDescriptor` has `fetchLimit`, `fetchOffset`, `sortBy`, prefetching and `propertiesToFetch`.
- No raw SQL. The store file is a Core Data SQLite database (`.store` in samples, with WAL sidecars), so it can be inspected with `sqlite3` for debugging, but that is community knowledge, UNVERIFIED as an Apple statement.

### Sync readiness

- Stable UUIDs, timestamps and soft-delete flags are plain attributes you add yourself; nothing automatic.
- SwiftData History (`HistoryDescriptor`, `fetchHistory`, macOS 15+) records inserts, updates and deletes with tokens, and `.preserveValueOnDeletion` keeps values of deleted rows. This is Apple's recommended way to compute "what changed since token X", which is exactly what a future backend sync needs. Source: https://developer.apple.com/documentation/swiftdata/fetching-and-filtering-time-based-model-changes
- CloudKit sync is built in but cannot enforce `unique` and requires optional relationships. Fenro plans its own backend, so this is not a factor.

## 2. GRDB (SQLite)

### Current state

- Latest release v7.11.1 on 2026-06-18; GRDB 7.0 shipped January 2025 with "full support for Swift 6". Requirements: macOS 10.15+, Swift 6.1+, Xcode 16.3+. Uses the system SQLite. Sources: https://github.com/groue/GRDB.swift/releases and https://github.com/groue/GRDB.swift/blob/master/README.md
- Maintenance: last push August 2026, 7 open issues, one primary maintainer (groue) with outside contributors; no GRDB 8 roadmap published. 8.6k stars. The single-maintainer risk is real but the project has been stable for a decade.

### Swift 6 concurrency

- Package is built in Swift 6 language mode. `DatabaseQueue` (one connection, serialized) or `DatabasePool` (WAL mode, parallel reads, one writer). `try await dbPool.write { db in ... }` and `read { }` honour task cancellation. Values crossing the closure boundary must be `Sendable`, which pushes you toward struct records, the recommended style. Source: https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/swiftconcurrency
- Rule: open exactly one `DatabaseQueue` or `DatabasePool` per file for the app's lifetime and share it. An in-process MCP server and the UI simply share that one pool.

### Migrations

- `DatabaseMigrator` with `registerMigration("name") { db in ... }`, run in order, each in its own transaction. Schema is written in SQL or a Swift DSL (`db.create(table:)`, `t.column(...)`, `t.belongsTo(...)`). `eraseDatabaseOnSchemaChange` for DEBUG builds. Source: https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/migrations
- Maintainer guidance: migrations must not depend on application types, so they stay valid forever.
- SQLite limitation applies: only add, rename and drop column are cheap; other column changes need the documented table-recreation sequence.

### SwiftUI glue

- `ValueObservation.tracking { db in ... }` notifies on every committed transaction touching the observed region, including writes via raw SQL or from another actor in the same process. Delivered on the main actor by default. It does not see writes from other processes. Source: https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/valueobservation
- GRDBQuery (companion package, v0.11.0, March 2025) adds `@Query(SomeRequest())` for views, roughly six lines of boilerplate per request. Or drive an `@Observable` model from `ValueObservation` as Apple's demo app pattern does. Source: https://github.com/groue/GRDBQuery
- Swift idiom note: `@Observable` is Apple's macro that lets SwiftUI re-render when a class's properties change; GRDB feeds such a class from the database.

### Records and SQL

- Protocols `FetchableRecord`, `TableRecord`, `PersistableRecord` combined with `Codable` structs. Query interface (`Task.filter { $0.status == "done" }.order(\.createdAt)`) plus type-safe SQL interpolation (`"SELECT * FROM task WHERE key = \(key)"`). Associations (`belongsTo`, `hasMany`) for joins. JSON columns supported. Source: https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/queryinterface
- Full SQL is always available, so aggregates, full-text search (FTS5), window functions and any filter grammar the MCP `list_tasks` tool needs map directly to SQL.

### Sync readiness

- UUIDs stored as blobs or text; timestamps via `willInsert`/`willSave` hooks using `db.transactionDate`; no built-in soft delete or change log. A change log is a few SQL triggers or a `TransactionObserver`. Source: https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb/recordtimestamps
- Because the schema is explicit SQL, a backend can share the same table definitions and migration numbering, which simplifies a later sync design.

### Learning cost

- For someone fluent in SQL and new to Swift, GRDB is close to the mental model already held: tables, migrations, queries. The Swift-specific parts (structs, `Codable`, actors, `Sendable`) are the same ones SwiftData requires anyway.

## 3. Core Data

- Not deprecated; Apple documents SwiftData and Core Data coexistence on one store file with `NSPersistentHistoryTrackingKey` set. Source: https://developer.apple.com/documentation/coredata/adopting-swiftdata-for-a-core-data-app
- For a new Swift-only app it offers nothing SwiftData or GRDB lack, and it carries `NSManagedObject` subclasses, a visual model editor and Objective-C conventions. Its only role in this decision is as an escape hatch inside a SwiftData app for aggregates, which is awkward.
- Verdict: rule it out as a primary choice.

## 4. Comparison against Fenro's needs

| Need | SwiftData (macOS 26) | GRDB 7 |
|---|---|---|
| In-process MCP writes refresh the UI | Documented as buggy, workaround needed; fixed in macOS 27 | Reliable via `ValueObservation` |
| Custom filter grammar for Dashboard and MCP | `#Predicate`, one root model, no aggregates | Full SQL |
| Unique Key per Project | `#Unique` compound constraint, upsert semantics | SQL `UNIQUE(project_id, sequence)` |
| Migrations | Versioned schemas, automatic when lightweight, rules not fully documented | Explicit numbered SQL migrations |
| Sync change tracking | Built-in History API (macOS 15+) | Triggers or `TransactionObserver`, written by you |
| Dependencies | None | GRDB (plus optional GRDBQuery) |
| Apple-idiomatic learning | Highest | High (Swift 6, actors, structs) but third-party |
| Maintenance risk | Apple | One maintainer, healthy cadence |
| Debuggability | Inspect the SQLite file (unofficial) | Inspect the SQLite file, run SQL directly |

## 5. Implications for the "Choose the persistence layer" decision

**Recommendation: GRDB.** The deciding factor is the in-process MCP server. Fenro's core promise is that an agent updates a Task and the developer sees it immediately. GRDB makes that a documented, first-class path on macOS 26; SwiftData makes it depend on behaviour Apple's own engineers call a bug until macOS 27. The secondary factors all lean the same way: the owner knows SQL, the shared filter grammar maps to SQL, and migrations are explicit.

The cost is one well-maintained dependency and giving up Apple's History API, which a few triggers replace.

**Conditions that flip the recommendation to SwiftData:**

- The deployment target moves to macOS 27, where `ResultsObserver` fixes cross-context observation and sectioned queries arrive.
- The owner decides the learning goal is specifically Apple's newest frameworks, and accepts the `NSManagedObjectContextDidSave` workaround for agent edits in the meantime.
- The future backend turns out to be CloudKit rather than a custom server.

**Either way**, shape the model for sync now: UUID primary keys, `createdAt` and `updatedAt`, a `deletedAt` soft-delete column, and a Key sequence per Project enforced by a unique constraint.
