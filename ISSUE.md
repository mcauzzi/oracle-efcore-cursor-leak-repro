# `Oracle.EntityFrameworkCore` leaks ref cursors when a non-first statement in a `SaveChanges` batch fails → eventual `ORA-01000`

### Environment

- **Oracle.EntityFrameworkCore**: reproduced on both `10.23.26200` (EF Core 10 / .NET 10) and `8.23.90` (EF Core 8 / .NET 8)
- **ODP.NET**: managed, default pooling and statement caching (no custom connection-string parameters)
- **Oracle Database**: 21c Express Edition Release 21.0.0.0.0 - Production, Version 21.3.0.0.0

### Summary

When `SaveChanges` batches multiple DML statements into a single anonymous PL/SQL block (entities with
`ValueGeneratedOnAdd` / identity columns, so generated values are returned via `OPEN <refcursor> FOR SELECT :B1 FROM DUAL`)
and a statement **after the first one in the block** raises (e.g. `ORA-12899: value too large for column`):

- the ref cursors already opened by the **preceding** statements of the block are never received by the
  client and stay **open on the session indefinitely**;
- they are **not** released by rollback, `ChangeTracker.Clear()`, `DbContext.Dispose()`, or
  `OracleConnection.PurgeStatementCache()` — only by closing the physical session (`ClearPool`);
- with pooling the session is reused indefinitely, so each failed batch adds cursors until the session
  reaches `open_cursors` → **`ORA-01000: maximum open cursors exceeded`**.

`v$open_cursor` for the affected SID shows the leaked cursors, all with `sql_text = 'SELECT :B1 FROM DUAL'`,
growing by one more per failed `SaveChanges`.

### Why it matters

Once the pooled session hits `open_cursors`, **every subsequent `SaveChanges` on that session fails with
`ORA-01000` instead of the original error** — a single recurring data error (here `ORA-12899`) progressively
poisons a pooled session until it starts failing statements that would otherwise succeed. In long-running
services, unrelated requests begin failing on a "good" connection.

### Key evidence — position dependence

- Failing statement **first** in the block → nothing leaks (no ref cursor opened yet).
- Failing statement **later** → exactly the cursors of the preceding statements leak.
- `MaxBatchSize(1)` makes the leak structurally impossible (no predecessors in the block).

### Reproduction

Minimal, self-contained repro (two throw-away tables, one pinned session, per-SID cursor counting via
`v$open_cursor`). Full details, run modes, and expected output in the README:

https://github.com/mcauzzi/oracle-efcore-cursor-leak-repro

```bash
dotnet run -- "User Id=...;Password=...;Data Source=..." 1 200   # mode 1: LEAK
dotnet run -- "User Id=...;Password=...;Data Source=..." 2 200   # mode 2: control (MaxBatchSize=1)
```

Mode 1 leaks ~+1 cursor per iteration; mode 2 stays flat. Without a `SELECT` grant on `v$open_cursor`,
mode 1 still demonstrates the bug by eventually crashing the session with `ORA-01000`.

### Expected behavior

The provider should close (or never abandon) the ref cursors already opened inside the PL/SQL block when a
later statement in the same block raises — e.g. wrap the block body in an exception handler that closes the
opened cursors before re-raising, or open the ref cursors only after all DML in the block has completed
successfully.
