# Oracle EF Core — orphaned ref cursors when a non-first statement in a batch fails

Minimal repro for a cursor leak in `Oracle.EntityFrameworkCore` that leads to
`ORA-01000: maximum open cursors exceeded` in long-running services using
connection pooling.

Reproduces on **both** provider lines:
- `Oracle.EntityFrameworkCore` **8.23.90** (EF Core 8, .NET 8)
- `Oracle.EntityFrameworkCore` **10.23.26200** (EF Core 10, .NET 10) — the
  version this project currently targets

## Summary

When `SaveChanges` batches multiple DML statements into an anonymous PL/SQL block
(entities with `ValueGeneratedOnAdd` / `HasDefaultValueSql`, so generated values are
propagated back via `OPEN <refcursor> FOR SELECT :B1 FROM DUAL`), and a statement
**after the first one** raises an error (e.g. `ORA-12899: value too large for column`):

- the ref cursors already opened by the **preceding** statements of the block are
  never received by the client and remain **open on the session indefinitely**;
- they are NOT released by: transaction rollback, 
  `DbContext.Dispose()`, `OracleConnection.PurgeStatementCache()`;
- they ARE released only by closing the physical session (`OracleConnection.ClearPool`);
- with pooling, the session is reused forever, so every failed batch adds cursors
  until the session hits `open_cursors` → `ORA-01000`.

`v$open_cursor` for the affected SID shows the leaked cursors all with
`sql_text = 'SELECT :B1 FROM DUAL'`, one more per failed `SaveChanges`.

**Position-dependence (key evidence):** if the *failing* statement is the FIRST of
the block, nothing leaks (no ref cursor was opened yet). If it comes later, exactly
the cursors of the preceding statements leak. `MaxBatchSize(1)` makes the leak
structurally impossible (at the cost of batching).

**Provider version difference (batch composition only):** on 8.23.x a HEAD insert
preceding the failing DETAIL is enough to leak. On 10.23.26200 the provider
composes batches differently (commands appear to be grouped per table), so the
leak requires a *successful row of the same table* before the failing one in the
block — the repro therefore inserts one good DETAIL row before the bad one, which
makes it leak on **both** provider lines. The leak mechanism itself is identical.

**Saturation behavior:** once the session reaches `open_cursors`, every subsequent
`SaveChanges` on that session fails with `ORA-01000` *instead of* the original
`ORA-12899` — i.e. the poisoned pooled session starts failing statements that
would otherwise succeed, which is how the bug surfaces in production services.

## How to run

```bash
dotnet run -- "User Id=...;Password=...;Data Source=..." 1 200   # mode 1: LEAK
dotnet run -- "User Id=...;Password=...;Data Source=..." 2 200   # mode 2: flat (MaxBatchSize=1)
```

The program creates two throw-away tables (`CURSOR_LEAK_HEAD`, `CURSOR_LEAK_DETAIL`,
the latter with a `VARCHAR2(10)` column). On every iteration it creates a **fresh
`DbContext`**, saves a batch where the DETAIL insert fails with `ORA-12899` (50 chars
into `VARCHAR2(10)`), and disposes the context so the connection is **returned to the
pool**. With **default pooling** (no connection-string changes) the pool hands the
**same physical session** back every iteration — the printed `SID` stays constant — so
the leaked cursors accumulate on it. This proves the leak **survives returning the
connection to the pool**: it is the realistic mechanism for a long-running service,
not an artefact of holding one connection open.

| Mode | SaveChanges content | Failing stmt position | Result |
|------|--------------------|----------------------|--------|
| 1 | HEAD (ok) + DETAIL (ok) + DETAIL (fails) | after a successful stmt | **leaks on every iteration** |
| 2 | same as 1, `MaxBatchSize(1)` | always 1st of its block | flat |

Per-SID monitoring uses `v$open_cursor` (needs a grant); without it, mode 1 still
demonstrates the bug by eventually crashing the session with `ORA-01000`.

## Cleanup

```sql
DROP TABLE CURSOR_LEAK_HEAD PURGE;
DROP TABLE CURSOR_LEAK_DETAIL PURGE;
```

## Environment

- Oracle.EntityFrameworkCore 10.23.26200 (NuGet) — also reproduced on 8.23.90
  (to test the 8.x line: set `<TargetFramework>net8.0</TargetFramework>` and
  `Oracle.EntityFrameworkCore` 8.23.90 in the csproj)
- Microsoft.EntityFrameworkCore 10.x / .NET 10 (also 8.x / .NET 8)
- Oracle Database Oracle Database 21c Express Edition Release 21.0.0.0.0 - Production
Version 21.3.0.0.0
- Default ODP.NET pooling and statement caching (no custom connection string params)

## Expected behavior

The provider should close (or never abandon) ref cursors already opened inside the
PL/SQL block when a later statement in the same block raises — e.g. wrap the block
body in an exception handler that closes opened cursors before re-raising, or open
the ref cursors only after all DML completed successfully.
