# Transactions, Concurrency Control and Recovery Systems

## What a transaction actually is

A transaction is a logical unit of work made up of several database operations that are meant to be treated as a single whole — either every part of it succeeds, or none of it takes effect at all.

A classic example is a money transfer from account A to account B: this requires debiting A and crediting B as two separate steps, but if either one fails, both need to roll back together. You can't end up in a world where money left A but never arrived at B.

## 7.1 Simple transaction models

In the simplest possible model, transactions run short, independently, and one after another in sequence.

The defining traits here: each transaction is treated as a single atomic unit, there's no concurrency happening at all, and if something fails, the transaction just restarts from scratch.

The catch is that this approach falls apart pretty quickly in any real multi-user environment — it just doesn't hold up to the demands of modern databases.

## 7.2 ACID properties

| Property | What it guarantees | How it's actually achieved |
|----------|------------------------|--------------------------------|
| Atomicity | Everything happens, or nothing does | Logs keep "before" images of data so changes can be undone |
| Consistency | The database's integrity rules stay intact | Enforced through constraints and triggers |
| Isolation | Concurrent transactions behave as though they ran one after another | Achieved using locks, timestamps, or MVCC |
| Durability | Once committed, changes survive even a crash | Changes are written to stable storage before the commit is confirmed |

## 7.3 Transaction isolation levels

### The problems isolation is trying to prevent

| Problem | What goes wrong |
|---------|---------------------|
| Dirty Read | Reading data that hasn't been committed yet, which might still get rolled back |
| Non-repeatable Read | Running the same query twice within a transaction and getting different results |
| Phantom Read | Rows appearing or disappearing between two reads within the same transaction |

### The levels themselves, from weakest to strongest

| Level | Allows Dirty Read | Allows Non-repeatable Read | Allows Phantom Read | Performance |
|-------|--------------------|------------------------------|--------------------------|----------------|
| READ UNCOMMITTED | Yes | Yes | Yes | Fastest |
| READ COMMITTED | No | Yes | Yes | Fast |
| REPEATABLE READ | No | No | Yes | Moderate |
| SERIALIZABLE | No | No | No | Slowest |

Most database systems default to READ COMMITTED — it's generally seen as a reasonable middle ground between safety and speed.

## 7.4 Serializability

A schedule earns the label **serializable** if running transactions concurrently produces the exact same effect as running them one after another in some order.

Two operations are said to "conflict" if they touch the same piece of data and at least one of them is a write — reads alone never conflict with each other.

The standard way to check for this is building a precedence graph, where each node represents a transaction and each edge represents a conflict between two transactions. If that graph comes out free of cycles, the schedule is conflict serializable.

This matters because it's the guarantee that lets a database stay consistent even while multiple transactions are running at the same time.

## 7.5 Lock-based protocols

### The two basic lock types

| Lock | What it's for | Compatibility with other locks |
|------|-------------------|-----------------------------------|
| Shared (S) | Reading only | Multiple shared locks can coexist fine |
| Exclusive (X) | Writing | Only one exclusive lock allowed at a time, and it can't coexist with a shared lock either |

### Two-Phase Locking (2PL)

This protocol splits a transaction's locking behavior into two distinct phases:

- **Growing phase:** The transaction can acquire new locks, but can't release any yet.
- **Shrinking phase:** The transaction starts releasing locks, but can no longer acquire new ones.

A stricter variant, **Strict 2PL**, holds onto every exclusive lock all the way until commit — this avoids a problem called cascading rollbacks, where one transaction's failure forces others to roll back too.

## 7.6 Handling deadlocks

A deadlock happens when, say, transaction T1 is holding resource A while waiting for resource B, and at the same time T2 is holding B while waiting for A — neither can move forward, and they're stuck waiting on each other indefinitely.

### Prevention — making deadlocks impossible in the first place

| Scheme | The rule it follows |
|--------|------------------------|
| Wait-Die | An older transaction is allowed to wait for a younger one; a younger transaction trying to wait for an older one just gets aborted instead |
| Wound-Wait | An older transaction "wounds" (aborts) a younger one that's in its way; a younger transaction is allowed to wait for an older one |

### Detection and recovery — catching deadlocks after they happen

One common tool here is a wait-for graph, where each node is a transaction and an edge from T1 to T2 means T1 is waiting on T2. If that graph contains a cycle, you've got a deadlock.

Once detected, recovery means picking a "victim" transaction to sacrifice — usually the youngest one, or whichever has done the least work or holds the fewest locks — then aborting it, rolling back its changes, and letting it restart.

## 7.7 Concurrency control mechanisms

| Mechanism | How it works | Prone to deadlocks? | Works best for |
|-----------|------------------|--------------------------|--------------------|
| Lock-based (2PL) | Locks get acquired before any data access happens | Yes | General-purpose workloads |
| Timestamp-based | Older transactions get priority over newer ones | No | Workloads with low conflict |
| Optimistic | Validation happens just before commit, rather than locking upfront | No | Workloads with very low conflict |
| MVCC | Keeps multiple versions of data around, so readers never block writers | Rarely | Read-heavy workloads |

MVCC is the mechanism behind PostgreSQL, Oracle, and MySQL's InnoDB engine — it's become something of an industry standard for handling concurrency without constant locking overhead.

## 7.8 Recovery algorithms

### Log records — the foundation of write-ahead logging

```
<START T>, <COMMIT T>, <ABORT T>
<UPDATE T, X, old_value, new_value>
```

The basic idea behind write-ahead logging is that every change gets written to the log *before* it's applied to the actual database, so there's always a record to fall back on if something goes wrong mid-transaction.

### The recovery protocols themselves

| Protocol | Needs Undo | Needs Redo | What it does |
|----------|----------------|----------------|------------------|
| Deferred Update | No | Yes | Only writes updates to the database once the transaction actually commits |
| Immediate Update | Yes | Yes | Allows updates to be written before commit, so both undo and redo become necessary |
| Shadow Paging | No | No | Maintains two separate page tables instead of relying on a log at all |

### ARIES — the industry-standard recovery algorithm (used in DB2, SQL Server)

ARIES works through three distinct phases:

1. **Analysis** — figures out which transactions were still active and which pages were left "dirty" (modified but not yet flushed to disk) at the time of the crash.
2. **Redo** — scans forward through the log, reapplying all the updates that were recorded.
3. **Undo** — scans backward through the log, reversing the work of any transactions that never committed.

A **checkpoint** is a periodic snapshot taken along the way, and it exists purely to cut down how much work recovery needs to do after a crash.

## 7.10 Remote backup systems

A remote backup is essentially a geographically separate copy of the database, kept around specifically for disaster recovery scenarios — fires, floods, hardware failures, anything that could take out the primary system entirely.

### The different flavors of standby systems

| Type | How long failover takes | Data loss risk |
|------|------------------------------|---------------------|
| Cold standby | Hours | None |
| Warm standby | Minutes | Small |
| Hot standby | Seconds | Minimal |

### Log shipping — how changes get sent to the backup

| Method | Durability | Speed |
|--------|----------------|-----------|
| Synchronous | Guarantees no data loss | Slower |
| Asynchronous | Some data loss is possible | Faster |

### The metrics that actually matter

| Metric | What it measures |
|--------|-----------------------|
| RTO (Recovery Time Objective) | How long it takes before the system is back up and available |
| RPO (Recovery Point Objective) | How much data, at most, you're willing to risk losing |

## Quick recap

| Concept | Key Takeaway |
|---------|------------------|
| Transaction | An all-or-nothing unit of work |
| ACID | Atomicity, Consistency, Isolation, Durability |
| Isolation Levels | READ UNCOMMITTED → READ COMMITTED → REPEATABLE READ → SERIALIZABLE |
| Serializability | Concurrent execution that behaves the same as some serial order |
| 2PL | Two-phase locking — growing phase, then shrinking phase |
| Deadlock | A circular wait between transactions; handled by either prevention or detection-and-recovery |
| MVCC | Keeps multiple data versions around, ideal for read-heavy workloads |
| WAL | Log changes before writing them to disk |
| ARIES | Recovery via Analysis → Redo → Undo |
| RTO/RPO | The objectives that define acceptable downtime and acceptable data loss |
