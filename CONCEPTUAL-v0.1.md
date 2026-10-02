## Proposed name: **Caravan**

The strongest fit is **Caravan**.

The Silk Road was not one continuous road. Goods moved between places through **caravans**, with goods being carried, exchanged, stored, and transferred across different parts of the route.

That maps surprisingly well to the abstraction wanted:

```text
POCO
  │
  ▼
Caravan
  │
  ├── Repository
  ├── Mapper
  ├── Transaction
  └── Connection
        │
        ▼
   PostgreSQL / MySQL / ...
```

The important semantic idea is:

> **Caravan carries application objects across the persistence boundary without making the objects themselves aware of the underlying store.**

It also avoids names such as `persistence`, `database`, `storage`, or `orm`, which unnecessarily constrain the project.

---

# Caravan — Conceptual Specification

## 1. Purpose

**Caravan** is a small persistence abstraction for ordinary application objects.

Its purpose is to allow an application to:

* persist POCOs;
* retrieve POCOs;
* update and delete POCOs;
* group persistence operations into transactions;
* remain independent of a particular database technology;
* receive uniform persistence-level exceptions;
* retain access to the original underlying database exception when required.

Caravan is **not an ORM**.

It does not attempt to make a database look like an object graph.

Its purpose is much narrower:

> **Provide a stable persistence boundary between application objects and database implementations.**

---

# 2. Fundamental Principle

The application owns the meaning of its objects.

Caravan owns the mechanics of persistence.

Therefore:

```text
Application
    │
    │ semantic objects
    ▼
  POCOs
    │
    │ persistence boundary
    ▼
 Caravan
    │
    │ database-specific implementation
    ▼
Database
```

A POCO does not know that Caravan exists.

A POCO does not know:

* PostgreSQL;
* MySQL;
* SQL;
* connections;
* transactions;
* database drivers.

Likewise, the database implementation does not define the meaning of the POCO.

---

# 3. Separation of Concerns

Caravan has four principal responsibilities.

### 3.1 Mapping

Translate between:

```text
POCO ↔ database representation
```

### 3.2 Repository

Provide operations for a particular persisted object or aggregate.

```text
Repository ↔ persistence operations
```

### 3.3 Connection

Provide access to a database implementation.

```text
Connection ↔ database communication
```

### 3.4 Transaction

Define atomicity across multiple persistence operations.

```text
Transaction ↔ commit / rollback
```

These are deliberately separate concepts.

---

# 4. What Caravan Does Not Do

Caravan deliberately does **not** provide:

* object-relational mapping;
* automatic schema generation;
* automatic migrations;
* lazy loading;
* relationship traversal;
* identity maps;
* change tracking;
* automatic query generation;
* domain validation;
* business rules;
* caching;
* event sourcing;
* message persistence;
* workflow orchestration.

These may be useful elsewhere, but they are outside Caravan's conceptual boundary.

This is important.

The project should remain small enough that its complete semantic model can be understood without learning an ORM.

---

# 5. POCO

A persisted object is an ordinary application object.

For example:

```python
@dataclass
class Phase:
    instrument: str
    timeframe: str
    bar_time: datetime
    value: float
```

There is no requirement for:

```python
class Phase(CaravanObject):
```

and no requirement for persistence annotations.

The object remains a normal object.

---

# 6. Mapper

A `Mapper` defines how a particular POCO is represented in persistence.

Conceptually:

```text
POCO
  │
  │ to_record
  ▼
Record
```

and:

```text
Record
  │
  │ from_record
  ▼
POCO
```

A mapper therefore owns representation, not semantics.

For example:

```text
Phase
 ├── instrument
 ├── timeframe
 ├── bar_time
 └── value

        ↓ Mapper

phase
 ├── instrument
 ├── timeframe
 ├── bar_time
 └── value
```

The database representation may differ from the POCO representation without changing the POCO.

---

# 7. Repository

A repository provides the persistence operations required by an application.

Conceptually:

```text
Repository[T]
```

where `T` is the POCO type.

For example:

```python
phases.save(phase)
phases.get(identity)
phases.delete(identity)
```

But Caravan should **not** force every repository into an artificial universal CRUD interface.

A repository may expose operations appropriate to its object:

```python
phases.find_for_bar(...)
phases.find_latest(...)
phases.save(...)
```

The repository therefore expresses **persistence operations**, while the domain object expresses **domain semantics**.

---

# 8. SQL Is Not Abstracted

This is one of the most important decisions.

Caravan should abstract **persistence**, not attempt to create a universal SQL language.

A PostgreSQL repository may contain PostgreSQL-oriented SQL.

Later, a MySQL repository may contain MySQL-oriented SQL.

For example:

```text
PostgreSQLRepository
       │
       └── PostgreSQL SQL

MySQLRepository
       │
       └── MySQL SQL
```

The application sees:

```text
Repository
```

not:

```text
PostgreSQL SQL
```

This keeps the abstraction honest.

---

# 9. Database Implementations

The first implementation is PostgreSQL using `psycopg3`.

Conceptually:

```text
                 Caravan
                    │
             Database Contract
                    │
          ┌─────────┴─────────┐
          │                   │
    PostgreSQL             MySQL
          │                   │
      psycopg3             MySQL driver
```

PostgreSQL is therefore an **implementation**, not part of Caravan's identity.

That means the public API should not contain things such as:

```python
PsycopgRepository
PsycopgTransaction
PostgresPOCO
```

except inside the PostgreSQL implementation.

---

# 10. Transactions

A transaction is an explicit persistence boundary.

For example:

```python
with transaction:
    phases.save(phase)
    levels.save(level)
    zones.save(zone)
```

The semantic rule is:

> Operations within a transaction either become durable together or are rolled back together.

Caravan itself does not decide **when** an application should use a transaction.

The application or higher-level infrastructure defines that boundary.

This also allows Bactrian to use Caravan later without giving Caravan knowledge of Bactrian.

---

# 11. Exception Model

This should be a first-class part of the specification.

The database implementation may generate completely different exceptions:

```text
psycopg.errors.UniqueViolation
mysql.connector.errors.IntegrityError
...
```

Caravan translates these into a uniform abstraction.

Conceptually:

```text
                    PersistenceError
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
 ConnectionError     TransactionError    OperationError
        │
        └── ConflictError
        └── NotFoundError
        └── MappingError
```

The exact hierarchy can be refined during API design.

The critical rule is:

> **Every exception crossing the Caravan public boundary has Caravan semantics.**

---

# 12. Exception Chaining

Normalization must never destroy diagnostic information.

If PostgreSQL produces:

```python
psycopg.errors.UniqueViolation
```

Caravan raises:

```python
PersistenceConflictError(...)
```

but retains:

```python
PersistenceConflictError.__cause__
```

as the original exception.

Thus:

```text
Application
    │
    ▼
PersistenceConflictError
    │
    └── __cause__
            │
            ▼
      psycopg.errors.UniqueViolation
```

The application can therefore operate at the abstraction level:

```python
except PersistenceConflictError:
    ...
```

while diagnostics can descend to the implementation level:

```python
except PersistenceError as exc:
    inspect(exc.__cause__)
```

This gives us **uniform semantics without loss of provenance**.

---

# 13. Error Ownership

Caravan should only translate errors that belong to persistence.

For example:

```text
Invalid Phase
```

is an application/domain error.

It should not become:

```text
PersistenceError
```

But:

```text
PostgreSQL connection refused
```

is a persistence error.

Likewise:

```text
duplicate key
```

is a persistence-level conflict.

Therefore:

```text
Domain error
    → remains domain error

Persistence failure
    → becomes Caravan exception
       + original cause
```

---

# 14. Dependency Direction

The dependency structure should be:

```text
                 Application
                      │
             ┌────────┴────────┐
             │                 │
            POCO            Caravan
                               │
                    ┌──────────┴──────────┐
                    │                     │
               PostgreSQL               MySQL
                    │                     │
                 psycopg3              driver
                    │                     │
                    └──────────┬──────────┘
                               │
                           Database
```

The critical rule is:

> **Caravan depends on database implementations; application POCOs do not depend on Caravan.**

And:

> **Bactrian depends on Caravan; Caravan does not depend on Bactrian.**

---

# 15. Bactrian's Relationship to Caravan

This gives us a very clean architecture:

```text
                  Bactrian
                     │
                     │ uses
                     ▼
                  Caravan
                     │
                     ▼
              PostgreSQL/MySQL
```

Bactrian can therefore persist things such as:

```text
Exchange
ExecutionResult
Phase
Level
Zone
...
```

without Caravan knowing what any of those things mean.

And another application could use exactly the same Caravan library for:

```text
Customer
Invoice
Document
Configuration
...
```

That is the real test of whether the abstraction is independent.

---

# 16. Core Vocabulary

I would initially restrict Caravan's conceptual vocabulary to:

| Concept               | Meaning                                     |
| --------------------- | ------------------------------------------- |
| **POCO**              | Ordinary application object                 |
| **Mapper**            | Converts POCO ↔ persistence representation  |
| **Repository**        | Performs persistence operations for a POCO  |
| **Connection**        | Provides database communication             |
| **Transaction**       | Defines atomicity of persistence operations |
| **Persistence Error** | Uniform error abstraction                   |
| **Provider**          | Database-specific implementation            |

That's enough.

No `EntityManager`, `Session`, `UnitOfWork`, `Model`, `Context`, `Store`, etc. unless a concrete requirement eventually justifies one.

---

# 17. The Silk Road Metaphor

The terminology becomes quite coherent:

```text
                         CARAVAN

    Application                              Database
        │                                        │
        │ POCO                                   │
        ▼                                        │
   ┌───────────┐                                 │
   │  Caravan  │─────────────────────────────────┤
   │           │                                 │
   │ Repository│                                 │
   │ Mapper    │                                 │
   │ Transaction│                                │
   │ Connection│                                 │
   └───────────┘                                 │
        │                                        │
        └──────────── Provider ──────────────────┘
```

A caravan does not change the identity of the goods it carries.

It merely provides the means for them to travel between places.

That is almost exactly the abstraction we want.

### Project identity

I would therefore use:

> **Caravan** — Lightweight POCO Persistence

with a deliberately plain description such as:

> **Caravan is a small database-independent persistence library for ordinary Python objects. It provides explicit mapping, repositories, transactions, database providers, and uniform persistence exceptions while keeping application objects independent of database technology.**

Then the first implementation can be:

> **Caravan PostgreSQL Provider — psycopg3**

and later:

> **Caravan MySQL Provider**

without changing the conceptual identity of the project.

