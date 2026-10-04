# Isolation Levels

*A deep-dive walkthrough of SQL transaction isolation levels — covering all five levels in precise, mechanical detail (Read Uncommitted, Read Committed, Repeatable Read, Serializable, and Snapshot as its own, genuinely distinct level), the crucial difference between Read Committed Snapshot Isolation and true Snapshot Isolation, the write skew anomaly that Snapshot permits but Serializable prevents, how each level's locking behavior maps onto what this series' Deadlocks guide covers, PostgreSQL's different implementation of the same ANSI standard, and how to choose the right level per workload rather than defaulting blindly.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Isolation Is a Dial, Not a Switch](#1-why-isolation-is-a-dial-not-a-switch)
3. [The Three Anomalies, Revisited Precisely](#2-the-three-anomalies-revisited-precisely)
4. [Read Uncommitted](#3-read-uncommitted)
5. [Read Committed](#4-read-committed)
6. [Repeatable Read](#5-repeatable-read)
7. [Serializable](#6-serializable)
8. [Snapshot: A Genuinely Different Mechanism, Not Just Another Level](#7-snapshot-a-genuinely-different-mechanism-not-just-another-level)
9. [Read Committed Snapshot Isolation (RCSI) vs. True Snapshot Isolation](#8-read-committed-snapshot-isolation-rcsi-vs-true-snapshot-isolation)
10. [Write Skew: The Anomaly Snapshot Permits That Serializable Prevents](#9-write-skew-the-anomaly-snapshot-permits-that-serializable-prevents)
11. [The Complete Anomaly Table](#10-the-complete-anomaly-table)
12. [How Isolation Level Choice Affects Deadlock Risk](#11-how-isolation-level-choice-affects-deadlock-risk)
13. [PostgreSQL: The Same Standard, a Different Implementation](#12-postgresql-the-same-standard-a-different-implementation)
14. [Setting Isolation Level in EF Core](#13-setting-isolation-level-in-ef-core)
15. [Choosing the Right Level Per Workload](#14-choosing-the-right-level-per-workload)
16. [Common Pitfalls](#15-common-pitfalls)
17. [Quick Reference Table](#quick-reference-table)
18. [Conclusion](#conclusion)

---

## Introduction

This series' Transactions guide introduces isolation levels as a trade-off between correctness and concurrency, and this series' Deadlocks guide's Section 10 shows the direct, quantitative link between isolation level and deadlock frequency — this guide is the dedicated, full treatment of the five levels themselves: what each one precisely guarantees and permits, the genuinely important distinction between Snapshot as a *level* and snapshot isolation as a *mechanism* (two related but different things, which SQL Server's own naming makes easy to conflate), and the write skew anomaly — a subtle, often-overlooked gap between what Snapshot isolation promises and what Serializable actually guarantees, which is exactly the kind of detail worth understanding precisely rather than assuming "Snapshot is basically Serializable but faster."

```plaintext
READ UNCOMMITTED  →  sees EVERYTHING, including other transactions' uncommitted work
READ COMMITTED    →  sees only COMMITTED data, but a re-read can see NEWER committed data
REPEATABLE READ   →  re-reads of the SAME rows are stable for the transaction's duration
SERIALIZABLE      →  behaves AS IF every transaction ran alone, one at a time
SNAPSHOT          →  a DIFFERENT mechanism entirely (versioning, not locking) giving a
                       transaction-consistent view — but NOT the same guarantee as Serializable
```

---

## 1. Why Isolation Is a Dial, Not a Switch

### The SQL standard defines isolation as a SPECTRUM, precisely because perfect isolation has a real, measurable cost

```plaintext
The theoretically "safest" behavior — every transaction behaving as if
  it ran completely alone, with no other transaction's existence even
  perceptible — is exactly what SERIALIZABLE provides. The SQL standard
  doesn't mandate this as the ONLY option specifically because achieving
  it requires real, costly machinery (extensive locking or conflict
  detection), and a great deal of real-world application logic genuinely
  doesn't need that strength of guarantee for every single operation.
```

This is the foundational framing this entire guide is built on — isolation level choice is a genuine, per-workload engineering decision, not a single "correct" setting to apply uniformly, and understanding each level precisely (rather than reaching for whichever one "sounds safest") is what lets you make that decision deliberately rather than by habit or folklore.

### Isolation level is typically set per-session or per-transaction, not globally fixed

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
    -- this transaction specifically runs under REPEATABLE READ
COMMIT;
-- subsequent transactions on this connection revert to whatever the session/database default is
```

Worth knowing this flexibility exists explicitly — a single application can, and often should, run most of its transactions at the database's default level while deliberately escalating specific, identified operations (Section 14) to a stronger guarantee, rather than treating isolation level as a single, application-wide constant.

---

## 2. The Three Anomalies, Revisited Precisely

### This series' Transactions guide's Section 6 introduces these; worth restating with complete precision as this guide's shared vocabulary

```plaintext
DIRTY READ:            reading another transaction's UNCOMMITTED data.
NON-REPEATABLE READ:    re-reading the SAME ROW within one transaction
                         returns a DIFFERENT value, because another
                         transaction committed a change to it in between.
PHANTOM READ:           re-running the SAME QUERY within one transaction
                         returns a DIFFERENT SET OF ROWS, because another
                         transaction committed an INSERT or DELETE
                         affecting which rows match the query's predicate.
```

Every level this guide covers is defined entirely in terms of which of these three anomalies it prevents and which it permits — this is genuinely the complete vocabulary needed to precisely characterize all five levels, and Section 10's table is this vocabulary applied systematically across every one of them.

---

## 3. Read Uncommitted

### The weakest level: NO isolation guarantee at all beyond basic row-level structural integrity

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT Balance FROM Accounts WHERE Id = 1;
-- can return a value from ANOTHER transaction's UPDATE that hasn't committed yet,
-- and might NEVER commit (if that other transaction rolls back)
```

Under Read Uncommitted, readers take **no shared locks at all**, and therefore never block on — or wait for — a writer's exclusive lock; they simply read whatever value is currently in the data page, committed or not. This permits all three anomalies (Section 10) and is, mechanically, the exact behavior this series' Threading guide's and Transactions guide's discussions of `WITH (NOLOCK)` describe in SQL Server specifically.

### Why it exists at all: a deliberate trade of correctness for zero read-blocking

```plaintext
The ONE genuine benefit: readers under READ UNCOMMITTED never wait on a
  writer's lock, and never cause a writer to wait on them either — for
  specific, narrow use cases (rough, approximate reporting dashboards
  where a transient, possibly-wrong value genuinely doesn't matter), this
  trade is occasionally defensible. For anything touching genuine
  business logic or user-facing correctness, it is almost never the right default.
```

---

## 4. Read Committed

### The default level in SQL Server, Oracle, and PostgreSQL — the practical, pragmatic baseline

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- SQL Server's DEFAULT if never explicitly set
SELECT Balance FROM Accounts WHERE Id = 1;  -- shared lock acquired for the DURATION of this statement only, then released
```

Read Committed guarantees you only ever see data that's genuinely been committed (eliminating dirty reads) — but it does this with the *shortest* possible lock-holding duration: a shared lock is taken just long enough to read a given row, then released immediately, **before** the statement (let alone the transaction) finishes. This is precisely why non-repeatable reads and phantom reads remain possible: nothing prevents another transaction from committing a change to that same row, or inserting a new matching row, the moment after your read released its lock.

### The precise mechanical trade-off this buys

```plaintext
SHORT lock duration → HIGH concurrency, LOW deadlock contribution (per
  this series' Deadlocks guide's Section 10) → but the SAME row, read
  TWICE in one transaction, can legitimately return TWO DIFFERENT values.
```

This is worth internalizing as the deliberate, reasonable design point most real application logic actually needs — most individual statements genuinely don't care whether a *different* query, five seconds later in the same transaction, would see updated data; what matters is that whatever you read was genuinely, actually committed when you read it, which Read Committed guarantees without the broader lock-retention cost of the stronger levels.

---

## 5. Repeatable Read

### Locks are held for the WHOLE transaction, not just each individual statement

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
    SELECT Balance FROM Accounts WHERE Id = 1;  -- shared lock acquired, and HELD until COMMIT/ROLLBACK
    -- ... other work ...
    SELECT Balance FROM Accounts WHERE Id = 1;  -- GUARANTEED to return the SAME value as the first read
COMMIT;
```

Repeatable Read extends Read Committed's shared lock from "just this statement" to "the entire remaining duration of this transaction" — any row your transaction has read is now genuinely protected from modification by anyone else until you finish, which is precisely what eliminates the non-repeatable read anomaly: a second read of a row you've already touched is *guaranteed* to match the first.

### Why phantom reads are STILL possible under Repeatable Read

```sql
SELECT COUNT(*) FROM Orders WHERE CustomerId = 42;  -- returns 5; locks the 5 EXISTING matching rows
-- another session: INSERT INTO Orders (CustomerId, ...) VALUES (42, ...); COMMIT;  -- a BRAND NEW row
SELECT COUNT(*) FROM Orders WHERE CustomerId = 42;  -- returns 6 — a PHANTOM appeared
```

This is a genuinely important, precise distinction worth getting exactly right: Repeatable Read's shared locks protect rows that your transaction has *already read* — they have no way to prevent an entirely *new* row from being inserted that would also match your query's predicate, since there was never a specific row to lock against that possibility. This is precisely why Serializable (Section 6) requires a fundamentally different locking mechanism to close this remaining gap.

---

## 6. Serializable

### Key-range locks: protecting the RANGE a predicate covers, not just the rows that currently exist within it

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
    SELECT COUNT(*) FROM Orders WHERE CustomerId = 42;  -- locks the ENTIRE RANGE of possible CustomerId=42
                                                           --  values, INCLUDING the "gaps" where a new row COULD be inserted
    -- another session attempting: INSERT INTO Orders (CustomerId, ...) VALUES (42, ...);
    -- BLOCKS, waiting on the key-range lock, until THIS transaction commits or rolls back
COMMIT;
```

This is the mechanism that closes Section 5's remaining gap — rather than locking only the specific rows that currently satisfy a predicate, a key-range lock locks the *entire logical range* the predicate covers, including the conceptual "gaps" between existing index entries where a new, matching row could be inserted — which is precisely what makes a phantom insert impossible: there's nowhere for the new row to go without colliding with the held range lock.

### The formal guarantee: AS IF every transaction ran serially, one at a time

```plaintext
Serializable's name describes its PRECISE guarantee: the END RESULT is
  always equivalent to SOME serial (one-at-a-time) execution order of
  the involved transactions — even though they genuinely ran
  concurrently. This is the STRONGEST isolation guarantee the SQL
  standard defines, and the only one of the four ANSI levels that
  eliminates all three anomalies (Section 10) completely.
```

### The real cost, restated with this guide's own precision

```plaintext
Key-range locks are held for the transaction's FULL duration (same as
  Repeatable Read's row locks) AND cover broader logical ranges than any
  weaker level — this is EXACTLY the combination this series' Deadlocks
  guide's Section 10 identifies as maximizing both blocking and deadlock
  probability, which is why Serializable is reserved for specific
  operations genuinely needing phantom protection, not applied broadly.
```

---

## 7. Snapshot: A Genuinely Different Mechanism, Not Just Another Level

### Why Snapshot doesn't fit neatly into the "stronger locking" progression the first four levels follow

```plaintext
Read Uncommitted → Read Committed → Repeatable Read → Serializable is a
  genuinely LINEAR progression — each level is strictly MORE locking,
  held LONGER and over a BROADER scope, than the one before it. SNAPSHOT
  isolation is NOT a further step in that same progression — it's a
  COMPLETELY DIFFERENT mechanism (row versioning/MVCC, per this series'
  Transactions guide's Section 9) that achieves a STRONG guarantee
  WITHOUT locking readers against writers AT ALL.
```

This is worth stating as plainly as possible, because SQL Server's naming (presenting `SNAPSHOT` as a sibling option alongside the four ANSI levels in the very same `SET TRANSACTION ISOLATION LEVEL` statement) genuinely invites the assumption that it's simply "a fifth rung on the same ladder" — it isn't; it's a structurally different approach to achieving isolation that happens to be exposed through the same syntax.

### How it actually works: your transaction sees a consistent snapshot of the database as of its own start

```sql
ALTER DATABASE MyDb SET ALLOW_SNAPSHOT_ISOLATION ON;

SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
    SELECT Balance FROM Accounts WHERE Id = 1;  -- sees the value as of THIS TRANSACTION'S START
    -- another session commits a change to this SAME row, in between...
    SELECT Balance FROM Accounts WHERE Id = 1;  -- STILL sees the ORIGINAL value — the transaction's
                                                   --  OWN snapshot never changes mid-transaction
COMMIT;
```

Per this series' Transactions guide's Section 9, the engine keeps previous versions of modified rows; your transaction is given a logically consistent view frozen at its own start time, and reads against it take **no locks at all** — this is what delivers Repeatable-Read-and-better guarantees for readers with zero reader/writer blocking, a genuinely different and often superior trade-off from the locking-based levels for read-heavy workloads.

---

## 8. Read Committed Snapshot Isolation (RCSI) vs. True Snapshot Isolation

### Two DIFFERENT database settings, both using the SAME underlying row-versioning mechanism, worth never confusing

```sql
-- RCSI: changes the BEHAVIOR of ordinary READ COMMITTED transactions
ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON;

-- True SNAPSHOT isolation: a SEPARATE, EXPLICIT isolation level a transaction OPTS INTO
ALTER DATABASE MyDb SET ALLOW_SNAPSHOT_ISOLATION ON;
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

This is a genuinely important, commonly conflated distinction worth getting exactly right — both settings use row versioning underneath, but they apply it at *different scopes*, with *different guarantees*:

```plaintext
RCSI: every STATEMENT gets its own snapshot, taken at the START OF THAT
  STATEMENT — this is "READ COMMITTED, but via versioning instead of
  locking" — it still permits non-repeatable reads WITHIN a transaction
  (a second SELECT in the same transaction gets a FRESH snapshot,
  potentially showing newer committed data), but readers never block
  writers and vice versa.
TRUE SNAPSHOT: the ENTIRE TRANSACTION shares ONE snapshot, taken at the
  transaction's START — this genuinely eliminates non-repeatable reads
  for the transaction's whole duration (and phantom reads too, per
  Section 10 — though see Section 9's write skew caveat), a strictly
  stronger guarantee than RCSI.
```

### Why RCSI is, in practice, the far more commonly deployed of the two

```plaintext
RCSI requires ONLY a single database-level setting, and APPLIES
  AUTOMATICALLY to every ordinary READ COMMITTED transaction (which is
  most of them, by default) — no application code changes are needed at
  all. TRUE Snapshot isolation requires EVERY transaction that wants it
  to EXPLICITLY opt in via SET TRANSACTION ISOLATION LEVEL SNAPSHOT,
  which is real, deliberate, per-transaction work.
```

This is worth knowing as the genuine, practical reason RCSI is the setting most production SQL Server databases actually enable (often as close to a default-recommended practice for reducing blocking), while true Snapshot isolation remains a more deliberately, narrowly applied tool for specific transactions that genuinely need a transaction-wide consistent view.

---

## 9. Write Skew: The Anomaly Snapshot Permits That Serializable Prevents

### The scenario: two transactions, each reading a CONSISTENT state, each making a DECISION based on it, that's only valid INDIVIDUALLY

```plaintext
A hospital rule: at least ONE doctor must always be on call. Two
  doctors, Alice and Bob, are both currently on call.

Transaction A (Alice wants to go off-call):
  SELECT COUNT(*) FROM OnCallDoctors;  -- reads 2 (Alice AND Bob) — the rule is satisfied, it's SAFE to go off-call
  UPDATE OnCallDoctors SET OnCall = 0 WHERE Name = 'Alice';
  COMMIT;

Transaction B (Bob ALSO wants to go off-call, running CONCURRENTLY):
  SELECT COUNT(*) FROM OnCallDoctors;  -- ALSO reads 2 — under THIS transaction's OWN consistent
                                          --  snapshot, Alice hasn't gone off-call YET — also looks SAFE
  UPDATE OnCallDoctors SET OnCall = 0 WHERE Name = 'Bob';
  COMMIT;

-- RESULT: BOTH commit successfully. ZERO doctors are now on call — the business rule is VIOLATED,
--          even though EACH transaction, taken individually, read a consistent, valid state and
--          made a correct decision based on exactly what it saw.
```

This is **write skew** — each transaction modifies a *different* row (Alice's row, Bob's row), so there's no direct row-level write conflict for Snapshot isolation's commit-time conflict detection (per this series' Transactions guide's Section 9) to catch — but the two transactions' combined effect violates an invariant that depends on *both* rows together, which neither transaction's own, individually-consistent snapshot ever reveals.

### Why Serializable specifically DOES prevent this, and Snapshot specifically does NOT

```plaintext
SERIALIZABLE'S key-range locking (Section 6) would cause Transaction B's
  read of OnCallDoctors to take a RANGE LOCK covering that table/predicate
  — Transaction A's UPDATE (modifying a row WITHIN that locked range) would
  then BLOCK until B commits or rolls back, at which point A's OWN read
  would need to be RE-VALIDATED against the now-changed data, correctly
  detecting the conflict. SNAPSHOT isolation's conflict detection ONLY
  catches two transactions writing the SAME row — it has NO mechanism
  to detect that two transactions' READS informed writes that TOGETHER
  violate an invariant neither write touches directly.
```

This is worth treating as the single most important, most easily overlooked fact in this entire guide — it is a completely reasonable, common assumption that "Snapshot isolation gives you Serializable-equivalent correctness, just implemented more efficiently via versioning instead of locking," and that assumption is **false**, precisely because of write skew; the two guarantees genuinely diverge for exactly this class of multi-row, invariant-dependent scenario, which is common enough in real business logic (capacity limits, "at least one of X" rules, mutual-exclusion-style constraints spanning multiple rows) to be a real, practical concern, not an academic curiosity.

### The practical implication: know which of your business invariants span more than one row

```plaintext
If a specific operation's CORRECTNESS depends on a constraint spanning
  MULTIPLE rows (not just a single row's own value), SNAPSHOT isolation
  alone does NOT protect it — this is precisely the kind of operation
  that genuinely warrants SERIALIZABLE (Section 6), explicitly, narrowly
  applied, rather than assuming Snapshot's generally-strong reputation
  covers this specific case too.
```

---

## 10. The Complete Anomaly Table

### Every level, every anomaly, stated precisely and completely

```plaintext
                     Dirty Read   Non-Repeatable Read   Phantom Read   Write Skew
READ UNCOMMITTED       possible       possible             possible      possible
READ COMMITTED         prevented      possible             possible      possible
REPEATABLE READ        prevented      prevented            possible      possible
SNAPSHOT                prevented      prevented            prevented*     possible   ← Section 9's key gap
SERIALIZABLE             prevented      prevented            prevented      prevented
```

*Snapshot isolation prevents phantom reads as this guide's earlier, standard definition describes them (a changed row *count* within your own transaction's consistent view) — but write skew (Section 9) is a related, distinct anomaly involving multiple rows and multiple transactions' combined effect, which Snapshot's single-row conflict detection does not catch.

This table is worth treating as the single, complete reference this entire guide builds toward — every section above exists to explain, precisely and mechanically, exactly one cell of this table, and Section 9's write skew row is the one cell this guide spends the most effort on specifically because it's the one most commonly assumed incorrectly.

---

## 11. How Isolation Level Choice Affects Deadlock Risk

### Restating this series' Deadlocks guide's Section 10 link, now with every level's mechanism in view

```plaintext
READ UNCOMMITTED: readers take NO locks → cannot participate in
  LOCK-based deadlocks at all (though can still be BLOCKED by, or
  contribute to, writer-vs-writer deadlocks).
READ COMMITTED: shortest lock duration among the locking-based levels →
  lowest deadlock contribution of the four ANSI levels.
REPEATABLE READ / SERIALIZABLE: locks held for the WHOLE transaction,
  SERIALIZABLE additionally over broader ranges → HIGHEST deadlock
  contribution, directly per this series' Deadlocks guide's Section 10.
SNAPSHOT / RCSI: readers take NO locks at all (row versioning instead)
  → structurally CANNOT participate in reader-vs-writer deadlock cycles,
  though writer-vs-writer conflicts (Section 9's related but distinct
  concern) are still possible, surfaced as an UPDATE CONFLICT error at
  commit time rather than a lock-based deadlock.
```

This is worth restating completely here, since it's the direct, practical payoff of understanding every level's mechanism precisely — this series' Deadlocks guide's Section 12 systematic elimination process explicitly includes "check whether the isolation level in use is genuinely needed," and this table is exactly the reference that check depends on.

---

## 12. PostgreSQL: The Same Standard, a Different Implementation

### PostgreSQL doesn't actually implement a distinct Read Uncommitted at all

```plaintext
PostgreSQL's documented behavior: requesting READ UNCOMMITTED is
  ACCEPTED syntactically, but internally treated IDENTICALLY to READ
  COMMITTED — PostgreSQL's MVCC architecture (row versioning, used for
  EVERY isolation level, not just an opt-in "Snapshot" level as in SQL
  Server) structurally never permits dirty reads in the first place,
  making a genuinely separate Read Uncommitted implementation unnecessary.
```

This is a genuinely important, concrete illustration of Section 1's "isolation is a spectrum defined by the standard, implemented differently by different engines" point — PostgreSQL's foundational architecture is MVCC-based for everything, which is precisely why it never needs SQL Server's separate "Snapshot" opt-in; PostgreSQL's ordinary Read Committed *already* behaves similarly to SQL Server's RCSI by default.

### PostgreSQL's REPEATABLE READ is actually closer to SQL Server's SNAPSHOT

```plaintext
PostgreSQL's REPEATABLE READ level uses a TRANSACTION-WIDE snapshot
  (taken at the transaction's first statement), via its MVCC mechanism —
  this is MECHANICALLY much closer to SQL Server's explicit SNAPSHOT
  level than to SQL Server's LOCK-based REPEATABLE READ (Section 5) —
  and critically, PostgreSQL's REPEATABLE READ ALSO prevents phantom
  reads as a direct, natural consequence of this mechanism, which makes
  it STRICTLY STRONGER than the ANSI standard technically requires for
  that level's name.
```

Worth knowing this specifically because it's a genuine, real trap for anyone moving between the two engines — "Repeatable Read" is not a portable guarantee with identical behavior across database systems, even though it's the same standardized name; always verify the *actual, specific* engine's documented behavior rather than assuming the ANSI standard's name alone fully specifies it.

### PostgreSQL's SERIALIZABLE: true serializability, via Serializable Snapshot Isolation (SSI)

```plaintext
PostgreSQL's SERIALIZABLE level is built on TOP of its snapshot
  mechanism, with ADDITIONAL conflict detection specifically designed to
  catch write skew (Section 9) and other serialization anomalies that a
  plain snapshot-based approach would otherwise miss — unlike SQL
  Server's SERIALIZABLE, which is LOCK-based (key-range locks), achieving
  the SAME formal guarantee through an entirely different mechanism.
```

---

## 13. Setting Isolation Level in EF Core

### Via `BeginTransactionAsync`, exactly as this series' Transactions guide's Section 13 and EF Core guide's Section 12 both cover

```csharp
await using var transaction = await context.Database.BeginTransactionAsync(IsolationLevel.Serializable);
try
{
    // the invariant-spanning, multi-row-dependent operation THIS specific guide's Section 9
    // identifies as genuinely needing Serializable, not just Snapshot
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

`System.Data.IsolationLevel` provides every level this guide covers (`ReadUncommitted`, `ReadCommitted`, `RepeatableRead`, `Serializable`, `Snapshot`) as a single enum, applied per-transaction via `BeginTransactionAsync`'s parameter — this is precisely the mechanism for applying Section 14's "narrow, deliberate escalation" principle in real application code, rather than changing the database's global default.

### Why RCSI (Section 8) is a database-level setting, not something EF Core toggles per-query

```plaintext
RCSI is enabled via ALTER DATABASE ... SET READ_COMMITTED_SNAPSHOT ON —
  a one-time, database-level configuration change, entirely OUTSIDE EF
  Core's own API surface — once enabled, EVERY ordinary READ COMMITTED
  transaction (EF Core's DEFAULT, per this series' EF Core guide)
  automatically benefits from row-versioning's reduced blocking, with NO
  application code changes required at all.
```

---

## 14. Choosing the Right Level Per Workload

### The practical decision framework this whole guide builds toward

```plaintext
DEFAULT, for the overwhelming majority of transactions: READ COMMITTED
  (ideally with RCSI enabled at the database level, per Section 8) —
  the right baseline for most everyday reads and single-row writes.

ESCALATE to REPEATABLE READ: when a transaction reads a row, makes a
  decision based on it, and later WRITES based on that same decision —
  and a non-repeatable change to that SPECIFIC row would genuinely
  produce an incorrect result.

ESCALATE to SERIALIZABLE: specifically when an operation's correctness
  depends on an INVARIANT SPANNING MULTIPLE ROWS (Section 9's write skew
  concern) — a capacity check, a uniqueness rule not already enforced by
  a database constraint, a "count of related rows must satisfy X" rule.

CONSIDER true SNAPSHOT: for a genuinely read-heavy, reporting-style
  transaction that needs a CONSISTENT, transaction-wide view across
  SEVERAL queries, without the write-side conflict-detection concerns
  (since it's read-only) — gets Repeatable-Read-and-better guarantees
  with zero reader/writer blocking.

NEVER default to READ UNCOMMITTED for genuine business logic — reserve
  it (if ever) for approximate, non-critical reporting only.
```

This is worth treating as the direct, actionable conclusion every other section in this guide has been building toward — and it's genuinely worth re-emphasizing the Section 9 caveat specifically here, since "just use Snapshot, it's basically as safe as Serializable" is exactly the plausible-sounding but incorrect shortcut this whole guide exists to correct.

---

## 15. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Assuming Snapshot isolation provides the same guarantee as Serializable | Write skew (Section 9) is a genuine, real gap — two transactions can each read a valid, consistent state and together violate a multi-row invariant | Use Serializable explicitly for operations whose correctness depends on an invariant spanning more than one row (Section 9, Section 14) |
| Confusing RCSI and true Snapshot isolation | They apply at different scopes (per-statement vs. per-transaction) with genuinely different guarantees, despite sharing the same underlying mechanism | Know precisely which one a given database/transaction is actually using before relying on either's specific guarantee (Section 8) |
| Assuming "Repeatable Read" means the same thing across different database engines | PostgreSQL's Repeatable Read is snapshot-based and stronger than SQL Server's lock-based implementation of the same standardized name | Verify the actual, documented behavior of the specific engine in use, not just the ANSI standard name (Section 12) |
| Defaulting to Serializable "to be safe" across an entire application | Maximizes both blocking and deadlock contribution, per this series' Deadlocks guide's Section 10, for the vast majority of operations that don't need it | Apply Read Committed as the default, escalating narrowly and deliberately only where Section 14's framework actually calls for it |
| Using `WITH (NOLOCK)`/Read Uncommitted for genuine business logic | Permits dirty reads of data that may never actually commit, and can even return duplicated or missing rows under concurrent page splits | Reserve Read Uncommitted for approximate, non-critical reporting only; never for correctness-sensitive logic (Section 3) |
| Assuming a database-level RCSI setting affects explicit `SNAPSHOT` transactions the same way | RCSI and Snapshot are separate, independently-configured features, even though both rely on row versioning | Enable `ALLOW_SNAPSHOT_ISOLATION` explicitly if true, transaction-wide Snapshot is needed; RCSI alone doesn't provide it (Section 8) |
| Not knowing which anomaly a specific business rule is actually vulnerable to | Leads to either under-protecting (a real correctness bug) or over-protecting (unnecessary blocking/deadlock cost) a given operation | Map the specific operation's correctness requirement onto Section 10's anomaly table before choosing a level |
| Assuming isolation level is a single, fixed, application-wide setting | Misses the real, available flexibility to apply different levels to different transactions based on their actual needs | Set isolation level per-transaction (Section 13), escalating only the specific operations that genuinely need it |

---

## Quick Reference Table

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew | Mechanism |
|---|---|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible | Possible | No locking at all |
| Read Committed | Prevented | Possible | Possible | Possible | Short-duration locks |
| Repeatable Read | Prevented | Prevented | Possible | Possible | Transaction-duration row locks |
| Snapshot | Prevented | Prevented | Prevented | **Possible** | Row versioning (MVCC) |
| Serializable | Prevented | Prevented | Prevented | Prevented | Key-range locks (SQL Server) / SSI (PostgreSQL) |

| Concept | Key Distinction |
|---|---|
| RCSI vs. Snapshot | Per-statement snapshot vs. per-transaction snapshot — different scope, different guarantee (Section 8) |
| Write skew | Multi-row invariant violated by two individually-valid, concurrently-committed transactions (Section 9) |
| SQL Server Serializable | Lock-based (key-range locks) |
| PostgreSQL Serializable | Snapshot-based with added conflict detection (SSI) |

---

## Conclusion

The five isolation levels this guide covers aren't five points on one simple ladder of "more safety, less speed" — four of them genuinely are exactly that linear progression, each buying a stronger guarantee through longer and broader locking, but Snapshot is a structurally different mechanism that happens to deliver most of that same progression's benefit through versioning instead, with its own specific, real gap at write skew that the locking-based Serializable closes and Snapshot does not. That gap is worth carrying as this guide's single most important takeaway, precisely because the two are so often treated as interchangeable in practice, and the difference only becomes visible for the specific, genuinely common class of business rule that depends on more than one row's state being considered together.

Understanding every level this precisely — not just "Serializable is strongest, Read Uncommitted is weakest," but the exact anomaly each one prevents, the exact mechanism it uses to prevent it, and the exact, quantified cost that mechanism imposes on concurrency and deadlock risk — is what turns isolation level choice from a setting copied out of a tutorial into a genuine, workload-specific engineering decision, made with the same deliberate, measured discipline this series' Deadlocks and SQL Indexes guides apply to their own respective corners of database performance and correctness.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the two-individually-correct-transactions-together-broke-an-invariant discovery that made write skew click far better than any anomaly table ever could on its own.*
