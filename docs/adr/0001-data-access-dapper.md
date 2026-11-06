# 0001. Use Dapper for data access instead of EF Core

- **Status:** Proposed (awaiting Product Owner / developer approval)
- **Date:** 2026-11-06

## Context

The project owner asked for Dapper in place of EF Core entities. CLAUDE.md
section 6 previously named EF Core as the data-access layer, with EF Core
migrations as the only way to change the schema. This ADR records the change
and the consequences for the rest of the standards.

## Decision

- Use **Dapper** (micro-ORM) for all database reads and writes.
- Use **ASP.NET Core** with repositories in `Infrastructure` that run hand-written
  parameterised SQL.
- Repositories map rows to Domain entities or DTOs. Row types never leave the
  repository layer.
- Schema changes use **versioned SQL scripts** applied by a migration runner.
  **Recommended: DbUp.** This is still to be confirmed.

## Alternatives

| Option | Pros | Cons |
|---|---|---|
| EF Core (previous plan) | Generated SQL, built-in migrations, change tracking | Entity mapping overhead; the owner asked against it |
| **Dapper + DbUp (chosen)** | Explicit SQL, small surface, fast, easy to read SQL performance | Hand-written SQL and scripts; no change tracking |
| Dapper + FluentMigrator | C# migration classes, versioned | Another abstraction to learn |

## Consequences

- No change tracking. Updates must be written as explicit SQL statements.
- Every query needs an explicit column list. `SELECT *` stays banned.
- Migration scripts are reviewed by hand. They are never edited after they are applied.
- Integration tests run against a real SQL Server (Testcontainers or LocalDB) and
  cover the scripts and repository queries.
- Soft-delete, audit columns (`CreatedAtUtc`, `UpdatedAtUtc`), and the
  appointment-status CHECK constraint must be written into the scripts by hand.
