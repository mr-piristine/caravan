
# Caravan — Normative API Specification

**Status:** Draft
**Version:** 0.1
**Purpose:** Define the minimal public contract of Caravan independently of database technology.

---

## 1. Scope

Caravan defines a persistence boundary between application objects and database implementations.

The API consists of five principal abstractions:

```text
POCO
  │
  ├── Mapper
  │
  └── Repository
          │
          └── Connection
                  │
                  └── Transaction
```

and one cross-cutting abstraction:

```text
PersistenceException
```

A conforming implementation MAY use PostgreSQL, MySQL, or another database technology.

The API MUST NOT require application code to depend on a specific database driver.

---

# 2. Design Rules

### C-01 — POCO independence

A persisted POCO MUST NOT require inheritance from, or dependency on, Caravan.

### C-02 — Explicit persistence

Persistence operations MUST be explicit.

Caravan MUST NOT implicitly persist objects through attribute mutation, garbage collection, property access, or similar mechanisms.

### C-03 — No ORM semantics

Caravan MUST NOT require:

* identity maps;
* lazy loading;
* change tracking;
* automatic relationship loading;
* automatic schema generation;
* automatic query generation.

### C-04 — Provider independence

The Caravan public API MUST NOT expose PostgreSQL- or MySQL-specific types.

### C-05 — Explicit mapping

The relationship between a POCO and its persistence representation MUST be defined by a `Mapper`.

### C-06 — Explicit transactions

Transaction boundaries MUST be explicit.

### C-07 — Uniform errors

Exceptions crossing the Caravan public API boundary MUST use Caravan exception types.

The underlying provider exception MUST remain available as the exception cause where one exists.

---

# 3. `Mapper[T]`

A `Mapper` defines the transformation between a POCO and its persistence representation.

Conceptually:

```python
Mapper[T]
```

### Required operations

```python
to_record(value: T) -> Record
from_record(record: Record) -> T
```

Where `Record` is an implementation-independent persistence representation.

The specification deliberately does not prescribe whether `Record` is:

* a dictionary;
* a tuple;
* a custom object;
* another structure.

That is an API-design detail to settle during implementation.

### Semantics

`to_record()`:

* MUST NOT mutate the POCO.
* MUST produce a representation suitable for persistence.
* MUST raise a Caravan mapping exception if conversion cannot be performed.

`from_record()`:

* MUST construct a new POCO.
* MUST NOT mutate the supplied record.
* MUST raise a Caravan mapping exception if conversion cannot be performed.

A mapper MUST NOT perform database operations.

---

# 4. `Repository[T]`

A repository provides persistence operations for a particular POCO type.

The minimal normative contract is:

```python
class Repository[T]:

    def save(self, value: T) -> None:
        ...

    def get(self, identity) -> T | None:
        ...

    def delete(self, identity) -> None:
        ...
```

The exact identity type is intentionally unspecified.

It MAY be:

```text
scalar
tuple
dataclass
named object
domain-specific identity
```

### `save()`

`save(value)` MUST persist the supplied POCO according to the repository's persistence semantics.

It MAY implement:

* insert;
* update;
* upsert;

but the repository MUST document which semantics apply.

`save()` MUST NOT silently modify the POCO.

### `get()`

`get(identity)` MUST return:

```text
POCO
```

when the object exists, or:

```text
None
```

when it does not.

A missing object MUST NOT automatically be treated as an exception.

If a repository needs strict retrieval semantics, it MAY additionally provide:

```python
require(identity) -> T
```

with `PersistenceNotFoundError`.

### `delete()`

`delete(identity)` MUST remove the identified persistent object.

Deleting an object that does not exist SHOULD be idempotent unless the repository explicitly documents otherwise.

---

# 5. Repository Specialization

`Repository[T]` is a minimal common contract, not a restriction against specialized operations.

For example:

```python
class PhaseRepository(Repository[Phase]):

    def find_for_bar(self, bar_time):
        ...

    def find_latest(self, instrument, timeframe):
        ...
```

Such methods are legitimate.

Caravan MUST NOT attempt to create a universal query API merely to avoid repository-specific operations.

This is important to preserving the small abstraction.

---

# 6. `Connection`

`Connection` represents an active connection to a persistence provider.

Its normative responsibility is:

> Provide the execution context in which repositories perform persistence operations.

The public abstraction MUST provide transaction capability.

Conceptually:

```python
connection.transaction()
```

The specification does **not** require a particular low-level method such as:

```python
execute()
cursor()
commit()
rollback()
```

Those are provider-level concerns unless required by the higher-level API.

This keeps SQL-driver concepts from leaking into Caravan.

---

# 7. `Transaction`

A transaction represents an atomic persistence boundary.

Conceptually:

```python
with connection.transaction():
    repository_a.save(a)
    repository_b.save(b)
```

### Successful completion

If the transaction scope completes normally:

```text
COMMIT
```

MUST occur.

### Exceptional completion

If an exception escapes the transaction scope:

```text
ROLLBACK
```

MUST occur.

The exception MUST then propagate.

### Atomicity

For a successful transaction:

```text
all operations become durable
```

For an unsuccessful transaction:

```text
none of the transaction's operations become durable
```

subject to the guarantees and limitations of the underlying database.

---

# 8. Nested Transactions

Caravan 0.1 SHOULD NOT define nested transaction semantics.

In particular, it SHOULD NOT introduce a generic `NestedTransaction` abstraction at this stage.

If an implementation supports database savepoints, that is a provider capability rather than a required Caravan 0.1 feature.

This keeps the first contract small.

---

# 9. Repository and Transaction Relationship

Repositories MUST participate in the transaction associated with their `Connection`.

For example:

```text
Connection
    │
    └── Transaction
           │
           ├── PhaseRepository
           ├── LevelRepository
           └── ZoneRepository
```

Repositories MUST NOT independently commit an operation when participating in an active transaction.

Therefore:

```python
with connection.transaction():
    phases.save(phase)
    levels.save(level)
```

is one atomic operation.

Not:

```text
save phase → COMMIT
save level → COMMIT
```

---

# 10. Exception Hierarchy

The public exception hierarchy should remain small.

```text
PersistenceException
│
├── PersistenceConnectionError
│
├── PersistenceTransactionError
│
├── PersistenceMappingError
│
├── PersistenceNotFoundError
│
├── PersistenceConflictError
│
└── PersistenceOperationError
```

All Caravan exceptions MUST derive from:

```python
PersistenceException
```

---

## 10.1 `PersistenceConnectionError`

The provider could not establish, maintain, or use a required database connection.

Examples:

```text
connection refused
authentication failure
connection lost
network failure
```

---

## 10.2 `PersistenceTransactionError`

The transaction mechanism itself failed.

Examples:

```text
commit failure
rollback failure
transaction state failure
```

A failure of an operation *inside* a transaction SHOULD retain its more specific exception type where possible.

---

## 10.3 `PersistenceMappingError`

A POCO could not be converted to or from its persistence representation.

This is a Caravan-level mapping failure, not a database failure.

---

## 10.4 `PersistenceNotFoundError`

Used only by operations whose contract explicitly requires an object to exist.

`get()` does not require this exception; it returns `None`.

---

## 10.5 `PersistenceConflictError`

The requested persistence operation conflicts with an existing persistent state.

A typical example is a uniqueness violation.

The precise interpretation is provider/repository dependent.

---

## 10.6 `PersistenceOperationError`

A persistence operation failed for a reason not represented by a more specific Caravan exception.

Examples might include:

```text
SQL execution failure
constraint failure
data conversion failure at provider level
unexpected database error
```

---

# 11. Exception Chaining

Every Caravan exception caused by an underlying provider exception MUST preserve that exception through Python exception chaining.

Conceptually:

```python
raise PersistenceOperationError(...) from provider_exception
```

Therefore:

```text
PersistenceOperationError
        │
        └── __cause__
                │
                ▼
        provider exception
```

The public application normally handles:

```python
except PersistenceException:
```

while diagnostic code can inspect:

```python
exception.__cause__
```

### Requirement

Caravan MUST NOT stringify and discard the provider exception.

This is a fundamental part of the API contract.

---

# 12. Exception Translation Boundary

Provider-specific exceptions MUST be translated at the provider boundary.

```text
Repository
    │
    ▼
Provider
    │
    ├── catches provider exception
    │
    └── translates
             │
             ▼
      Caravan exception
```

Application code MUST NOT need:

```python
except psycopg.Error:
```

when using Caravan.

Instead:

```python
except PersistenceException:
```

---

# 13. PostgreSQL Provider Boundary

PostgreSQL is the first Caravan provider.

Its responsibility is to translate between:

```text
Caravan contract
        ↕
PostgreSQL / psycopg3
```

The PostgreSQL provider MAY know about:

* psycopg3;
* PostgreSQL SQL;
* PostgreSQL transaction semantics;
* PostgreSQL-specific error codes;
* PostgreSQL connection configuration.

The core Caravan API MUST NOT know about any of them.

Therefore:

```text
caravan.core
       │
       │ no psycopg dependency
       ▼
caravan.postgres
       │
       ▼
    psycopg3
```

---

# 14. PostgreSQL Provider Responsibilities

The PostgreSQL provider MUST:

1. establish PostgreSQL connections;
2. implement Caravan's `Connection` contract;
3. implement Caravan transaction semantics;
4. execute repository operations;
5. translate relevant psycopg3 exceptions;
6. preserve the original exception as `__cause__`.

It MUST NOT:

* modify POCO semantics;
* introduce domain validation;
* require Bactrian;
* define application business rules.

---

# 15. MySQL Provider

MySQL is not part of Caravan 0.1 implementation.

However, Caravan's contract MUST be designed so that a future provider can implement:

```text
Caravan Connection
Caravan Transaction
Caravan Repository
Caravan Exception translation
```

using MySQL without changing application-level persistence code.

Conceptually:

```text
                   Caravan API
                       │
             ┌─────────┴─────────┐
             │                   │
        PostgreSQL             MySQL
             │                   │
          psycopg3          mysql driver
```

The two providers need not have identical internal implementations.

They only need to satisfy the same Caravan contracts.

---

# 16. Provider-Specific Capabilities

Caravan 0.1 MUST NOT pretend that every database has identical capabilities.

If PostgreSQL provides something that MySQL does not, that difference may exist inside the provider.

The common API should expose only semantics that can reasonably be shared.

Provider-specific functionality MAY exist behind a provider-specific API, but application code using the portable Caravan contract MUST NOT depend on it.

---

# 17. Persistence Lifecycle

The normative lifecycle is:

```text
Create Connection
       │
       ▼
Create Repository
       │
       ▼
Begin Transaction
       │
       ▼
Repository operations
       │
       ├──── success ────► Commit
       │
       └──── failure ────► Rollback
                                │
                                ▼
                           Exception
```

A connection MAY be reused for multiple transactions.

A repository MAY be reused while its associated connection remains valid.

---

# 18. What Is Deliberately Undefined

Caravan 0.1 does **not** define:

* connection pooling;
* asynchronous APIs;
* migrations;
* schema discovery;
* SQL generation;
* repository factories;
* dependency injection;
* caching;
* retry policies;
* optimistic locking;
* pagination;
* bulk operations;
* streaming;
* savepoints;
* distributed transactions.

These are deliberately deferred.

That is not an omission in the specification; it is part of the design.

---

# 19. Minimal Public Surface

If we compress the entire specification into its essential API:

```python
Mapper[T]
    to_record(value: T) -> Record
    from_record(record: Record) -> T


Repository[T]
    save(value: T) -> None
    get(identity) -> T | None
    delete(identity) -> None


Connection
    transaction() -> Transaction


Transaction
    enter()
    exit(...)


PersistenceException
    PersistenceConnectionError
    PersistenceTransactionError
    PersistenceMappingError
    PersistenceNotFoundError
    PersistenceConflictError
    PersistenceOperationError
```

Everything else can be built around this.

---

# 20. The Core Contract

I would put this sentence near the beginning of the actual Caravan specification:

> **Caravan defines a database-independent persistence boundary for ordinary application objects. It standardizes mapping, repositories, transactions, and persistence exceptions while leaving database representation, SQL, and provider mechanics to implementations.**

And the corresponding architectural rule:

```text
                MEANING
                   │
                   ▼
                 POCO
                   │
          ┌────────┴────────┐
          │                 │
       Mapper           Repository
          │                 │
          └────────┬────────┘
                   │
               Connection
                   │
               Transaction
                   │
               Provider
                   │
          ┌────────┴────────┐
          │                 │
      PostgreSQL          MySQL
```

