# DB Wrangler — Language & Runtime Specification (Draft)

DB Wrangler is a language, runtime, and workbench for database development. It combines the immediacy of interactive SQL with the maintainability of reusable, version-controlled database functions.

Unlike traditional database tools, DB Wrangler treats database operations as software — organizing diagnostics, reports, repairs, and automation into packages and repositories, while still supporting free-form SQL exploration.

This document defines the DB Wrangler language and execution model. Runtime internals, compiler architecture, and provider APIs are specified separately.

> **Document scope:** this is one of four planned documents — **Language Specification** (this doc, syntax/semantics/execution model), **Runtime Specification** (execution internals), **Provider Specification** (implementation SPI), **Compiler Specification** (parser/AST/pipeline). See `db-wrangler-compiler-provider-architecture-v2.md` for compiler/provider-level detail; terminology is reconciled between the two at the end of this document.

---

## Design Principles

- Interactive SQL is a first-class language feature.
- Reusable behaviour is implemented as Wrangler Functions.
- Functions are independent of their implementation language.
- Packages provide globally unique namespaces.
- All SQL execution is parameterised.
- Functions compose naturally through nested calls.
- The shell evaluates expressions rather than commands.
- The same language is used by the REPL, workbench, and IDE.

---

## Execution Models

### Interactive SQL

For exploration, diagnostics, and administration.

```
select *
from customer
where status = 'ACTIVE';
```

Results may be assigned:

```
customers =
    select *
    from customer
    where status = 'ACTIVE';
```

Shell variables are automatically available as SQL parameters:

```
status = "FAILED"
failed =
    select *
    from payment
    where status = :status;
```

Interactive SQL replaces the traditional query window found in tools such as pgAdmin and DBeaver.

### Wrangler Functions

Reusable, version-controlled database behaviour:

```
payments.findFailed(Duration.ofHours(2))
payments.repair(failed)
reports.generate(failed)
```

Callable from: the interactive shell, other Wrangler Functions, the workbench, scheduled automation, and external APIs.

### Relationship Between Interactive SQL and FSQL

FSQL is the reusable SQL implementation language. Interactive SQL and `[fsql]` functions are executed by the **same provider and SQL engine** — the difference lies only in **parameter resolution**:

- **Interactive SQL** — parameters resolve against shell variables:

  ```
  status = "FAILED"
  select *
  from payment
  where status = :status;
  ```

- **FSQL functions** — parameters resolve only against declared function parameters:

  ```
  function findFailed(String status) [fsql] : Table {
      select *
      from payment
      where status = :status;
  }
  ```

  Function bodies have no visibility of shell variables. This means an unresolved `:param` inside a function body (not matching any declared parameter) is statically checkable at compile time, unlike the shell case, which is inherently a runtime lookup against caller state.

---

## Packages

Every Wrangler module belongs to a package:

```
package com.company.payments;
```

Packages define globally unique namespaces.

---

## Imports

```
import qualified com.company.payments;                 // usage: payments.findFailed()
import qualified com.company.payments as pay;           // aliased
import findFailed, repair from com.company.payments;    // selective
```

Only explicitly imported functions become visible in the local namespace.

---

## Wrangler Modules

A Wrangler module is the primary development unit, containing a package declaration, imports, and Wrangler Functions. Unlike notebooks, every executable block is a named, reusable function. A module may contain functions implemented in different languages:

```
package com.company.payments;

function findFailed() [fsql] : Table {
    select *
    from payment
    where status = 'FAILED';
}

function repair(Table failed) [groovy] : Integer {
    ...
}

function report() [groovy] : Binary {
    failed = findFailed()
    repaired = repair(failed)
    return reports.generate(failed, repaired)
}
```

---

## Function Declaration

```
function <name>(<parameters>) [<language>] : <return-type> {
    implementation
}
```

Examples:

```
function findFailed(Duration since) [fsql] : Table
function repair(Table rows) [groovy] : Integer
function generate() [java] : Binary
```

---

## Expression Language

The shell evaluates expressions:

```
payments.findFailed()
report.generate(customer.findActive())
repair(findFailed())
```

Evaluation follows standard programming language rules. Nested calls are evaluated from the inside out:

```
repair(findFailed())
```

becomes

```
findFailed()
  ↓
Table
  ↓
repair(Table)
```

---

## Variables

Any expression may be assigned:

```
failed = findFailed()
customers =
    select *
    from customer
```

Variables reference the value returned by a specific execution. **Variables never change automatically** — see the note on refresh semantics in Execution Panels below, which is not yet fully reconciled with this rule.

---

## Runtime Values

Every expression returns a **Wrangler Value**. Core types:

- `Table`
- `Row`
- `Scalar`
- `Object`
- `List`
- `Binary`
- `Null`

### `Table`

The primary runtime type. Conceptually:

```
Table implements List<Row>
Row implements Map<String, Object>
```

A `Table` also carries **schema metadata**:

- Column name
- Type
- Nullable
- Precision
- Scale

This metadata supports charts, editors, validation, export, and auto-complete — resolving the earlier open question of where chart-view suggestions get their type information from: it's carried directly on the `Table` value itself, not re-derived by the workbench.

### `Binary`

Represents arbitrary content via a MIME-type discriminator rather than a closed set of format-specific types:

```
Binary
    contentType
    data
```

Examples: `application/pdf`, `text/html`, `image/png`, `application/json`. Rendering is determined entirely by the workbench, dispatching on `contentType` — the runtime type system carries no presentation opinion.

---

## Execution Model

Functions are stateless. **Only executions produce values.** Two execution types exist.

### Interactive Execution

Started directly by the user:

```
findFailed()
customer.find(123)
select *
from customer
```

Interactive execution creates an **Execution Panel**, which stores:

- Expression
- Bound argument values
- Connection
- Last execution time
- Duration
- Returned value
- Result views

The stored result is a snapshot of that execution.

### Nested Execution

Nested calls execute normally but do **not** create or update execution panels:

```
repair(findFailed())
```

Execution order:

```
findFailed()
  ↓
Table
  ↓
repair(Table)
  ↓
Integer
```

Only the outer interactive execution updates its panel. The nested execution of `findFailed()` remains an implementation detail, invisible to the workbench as its own panel.

---

## Execution Panels

The workbench manages **Execution Panels** rather than function panels or result tabs. Each panel represents one interactive expression.

```
findFailed(Duration.ofHours(2))
```

and

```
findFailed(Duration.ofDays(1))
```

create two independent panels, because the expressions (including bound arguments) differ.

Each panel remembers: expression, bound argument values, last result, and last execution metadata. Selecting **Refresh** re-executes the original expression. Refreshing one panel never affects any other panel.

> **Open question — not yet resolved by this document:** if a variable was assigned from a panel's execution (`failed = findFailed()`) and that panel is later refreshed, does `failed` update to the new snapshot, or remain frozen to the original value per the "variables never change automatically" rule? The two statements aren't currently reconciled — needs an explicit answer (most likely: refresh updates only the panel's own stored result, and the variable stays bound to its original snapshot unless re-assigned, but this should be stated directly rather than inferred).

> **Open question — not yet resolved by this document:** re-executing the *identical* expression (same arguments) — does it create a new panel each time, or update the existing panel for that expression in place? The doc only specifies behavior for *differing* expressions.

---

## Result Views

Every execution may be viewed multiple ways — Grid, Chart, JSON, Statistics, Execution Plan, Export — as different presentations of the same Wrangler Value, surfaced through its owning Execution Panel.

### Charts

Charts are **not** runtime values — they are workbench views over a `Table`, with suggestions derived from the table's schema metadata (see `Table` above):

```
select status, count(*) from payment group by status;
```
→ suggested view: bar chart

```
select created_date, count(*) from payment group by created_date order by created_date;
```
→ suggested view: line chart

---

## Function Providers

Functions are implemented by **Function Providers** — the doc-level term aligned with the Compiler Specification's `FunctionProvider` interface (this document previously used "Language Provider"; "Function Provider" is the settled term going forward, resolving the earlier terminology-overload open question). Initial providers:

- FSQL
- Groovy
- Kotlin
- Java

Each provider is responsible for compiling functions, caching compiled artefacts, executing implementations, and marshaling Wrangler Values. The runtime itself is implementation-language independent — see the Compiler Specification for how a provider fits into the `FunctionDescriptor → CompiledFunction → WranglerFunction` lifecycle.

---

## Execution Context

Every execution receives an Execution Context, containing:

- Database connection
- Transaction
- Configuration
- Logger
- Repository manager
- Shell variables
- Output services

Nested executions inherit the same execution context — including the active transaction — regardless of how many providers a call chain crosses.

---

## Runtime Architecture

```
                 Wrangler Language
                         │
                    Parser & AST
                         │
              Expression Evaluator
                         │
              Function Dispatcher
                         │
               Wrangler Function
          ┌──────────────┼──────────────┐
          │              │              │
       [fsql]        [groovy]    [kotlin/java]
          │              │              │
      SQL Engine    JVM Runtime    JVM Runtime
                         │
                Execution Context
                         │
             JDBC • Git • File System
```

---

## Reconciliation with the Compiler & Provider Architecture document

| This document | Compiler/Provider document | Notes |
|---|---|---|
| Function Provider | `FunctionProvider` | Now the same term at both the doc-prose and interface level — earlier "Language Provider" wording is retired. |
| Wrangler Value | `WranglerValue` | Canonical member list: `Table`/`Row`/`Scalar`/`Object`/`List`/`Binary`/`Null`. `Document` was considered and dropped in favor of `Binary` + `contentType`. |
| Execution Context | *(previously an open question)* | Confirms transaction-scoping: context, including transaction, flows through the Dispatcher into every provider call in a chain. |
| Execution Panel / Interactive vs. Nested Execution | *(not previously modeled)* | New concept at this layer; the Compiler Specification's Dispatcher is the natural place that would need to know whether a given call is the outer (panel-creating) execution or a nested one — worth cross-referencing once that distinction is added to the dispatcher's contract. |
| FSQL as one provider for both execution models | Flagged as an open question in the compiler notes | Confirmed at the **provider/engine level** only. Parameter binding is explicitly *not* unified — shell-scoped for ad hoc SQL, function-scoped (statically checkable) for `[fsql]` functions. |
| *(not covered here)* | `FunctionDescriptor` / `CompiledFunction` / `WranglerFunction` lifecycle | Internal compiler-stage detail; this document operates one level up. |

---

## Vision

DB Wrangler unifies two traditionally separate workflows:

1. Interactive SQL for exploration and administration.
2. Reusable Wrangler Functions for production-quality database software.

The workbench elevates both by treating every interactive execution as a first-class, refreshable Execution Panel, while preserving conventional programming semantics for nested function calls. Developers can begin with exploratory SQL, evolve it into reusable functions, compose those functions across multiple implementation languages, and execute everything through a single language, runtime, and workbench.

---

## Open Questions (Updated)

- **Variable/panel refresh relationship** — does refreshing a panel update a variable that was assigned from it, or does the variable stay frozen to its original snapshot? (New — see Execution Panels section.)
- **Panel identity on identical re-execution** — does re-running the exact same expression create a new panel or update the existing one in place? (New — see Execution Panels section.)
- Whether an unresolved `:param` inside a `[fsql]` function should be enforced as a compile-time error, now that it's statically checkable.
- Native type mapping for `Binary` (with `contentType`) across all providers.
- Whether the workbench's per-`contentType` rendering rules should be a fixed built-in table or extensible/pluggable.
- Function overloading vs. default-parameter semantics.
- Aliasing for unqualified imports (`import findFailed as ff from ...`).
