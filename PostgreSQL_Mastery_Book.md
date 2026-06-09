# PostgreSQL Mastery
## A Complete Engineer's Guide — From Internals to Production

---

> **How to use this book**
> Every chapter answers three questions in order:
> **WHY** does this exist? **WHAT** exactly is it? **HOW** does it work inside?
> Every concept is followed by runnable SQL you can test yourself.
> Read it linearly. Each chapter builds on the last.

---

# PART I — FOUNDATIONS

> Before you can write a fast query, you need to understand what happens
> the moment you press Enter. This part answers that question completely.

---

# CHAPTER 1 — Architecture Overview

---

## 1.1 WHY You Must Understand the Architecture

Most engineers treat PostgreSQL as a black box. They write SQL, get results, and move on.
That works — until queries become slow, the database locks up, or a crash loses data.

At that point, the engineers who understand what PostgreSQL is doing internally
can diagnose and fix the problem in minutes. The others spend days guessing.

This chapter is the map of that internal world.

---

## 1.2 WHAT PostgreSQL Is — The Big Picture

PostgreSQL is not a single program. It is a **collection of cooperating processes**
sharing memory, reading and writing files on disk.

When you connect from your Python app, here is what actually exists on the server:

```
┌─────────────────────────────────────────────────────────────────┐
│                        SERVER MACHINE                           │
│                                                                 │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐              │
│  │  Backend   │   │  Backend   │   │  Backend   │  ← one per   │
│  │ Process 1  │   │ Process 2  │   │ Process 3  │    client    │
│  │ (your app) │   │  (DBeaver) │   │  (pgAdmin) │              │
│  └─────┬──────┘   └─────┬──────┘   └─────┬──────┘              │
│        │                │                │                      │
│        └────────────────┴────────────────┘                      │
│                         │                                       │
│              ┌──────────▼──────────┐                            │
│              │    SHARED MEMORY    │                            │
│              │  (shared_buffers,   │                            │
│              │   WAL buffers,      │                            │
│              │   lock table)       │                            │
│              └──────────┬──────────┘                            │
│                         │                                       │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │               BACKGROUND PROCESSES                      │    │
│  │  postmaster │ WAL writer │ checkpointer │ autovacuum     │    │
│  │  bgwriter   │ stats collector │ archiver │ wal receiver  │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │                                       │
│              ┌──────────▼──────────┐                            │
│              │    DISK STORAGE     │                            │
│              │  Data files (heap)  │                            │
│              │  Index files        │                            │
│              │  WAL files          │                            │
│              │  Config files       │                            │
│              └─────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

The key insight: **each client connection is a separate OS process**.
Not a thread — a full process. This costs ~5MB RAM per connection,
which is why connection poolers (PgBouncer) are essential in production
when you have hundreds of clients.

---

## 1.3 HOW a Query Travels Through PostgreSQL

Let us trace exactly what happens when your Python application runs:

```python
cursor.execute("SELECT name, price FROM products WHERE sku = 'WID-001'")
```

### Stage 1 — Connection

Your app opens a TCP connection to PostgreSQL (port 5432).
The `postmaster` process (PostgreSQL's master supervisor) forks a new **backend process**
specifically for your connection. This process lives for the duration of your session.

```
Your App  ──TCP:5432──►  postmaster  ──fork──►  backend process #1234
                                                 (lives until you disconnect)
```

### Stage 2 — Parser

The backend receives the raw SQL string `"SELECT name, price FROM products WHERE sku = 'WID-001'"`.

The **lexer** breaks it into tokens:
```
[SELECT] [name] [,] [price] [FROM] [products] [WHERE] [sku] [=] ['WID-001']
```

The **parser** builds an **Abstract Syntax Tree (AST)** — a tree structure
that represents the logical meaning of the query:

```
SelectStmt
├── targetList
│   ├── ResTarget → ColumnRef → "name"
│   └── ResTarget → ColumnRef → "price"
├── fromClause
│   └── RangeVar → "products"
└── whereClause
    └── A_Expr (=)
        ├── ColumnRef → "sku"
        └── A_Const → 'WID-001'
```

**Important:** The parser only checks syntax. It does NOT check if table
`products` or column `sku` actually exists. That comes next.

### Stage 3 — Analyzer (Semantic Analysis)

The analyzer takes the AST and verifies it against the system catalogs
(`pg_class`, `pg_attribute`, `pg_type`):

- Does table `products` exist? → looks up `pg_class`
- Do columns `name`, `price`, `sku` exist on `products`? → looks up `pg_attribute`
- What data type is `sku`? → checks it matches `'WID-001'` (text = text ✅)
- Are there any required permissions? → checks `pg_acl`

Output: a **Query Tree** — same structure as the AST but with all
identifiers resolved to internal OIDs (Object IDs).

### Stage 4 — Rewriter

The rewriter applies **rule-based transformations** to the Query Tree.

The most important use: **VIEW expansion**.

If `products` were actually a view:
```sql
CREATE VIEW products AS
  SELECT * FROM raw_products WHERE deleted_at IS NULL;
```

The rewriter replaces the reference to `products` with the full view definition.
Your query becomes:
```sql
SELECT name, price
FROM raw_products
WHERE deleted_at IS NULL AND sku = 'WID-001'
```

This happens transparently — you never see it.

### Stage 5 — Planner / Optimizer ← The Most Important Stage

This is where PostgreSQL earns its reputation. The planner takes the
rewritten query and **generates every possible way to execute it**,
then estimates the cost of each, and picks the cheapest.

For our simple query, the possibilities include:
- Sequential scan of `products`, filter `sku = 'WID-001'`
- Index scan on `products` using `ix_products_sku`
- Bitmap scan on `products` using `ix_products_sku`

The planner uses **statistics** stored in `pg_statistic` to estimate:
- How many rows does `products` have? (e.g., 50,000)
- How selective is `sku = 'WID-001'`? (1 row, since SKU is unique)
- Is there an index on `sku`? (yes, `ix_products_sku`)

**Cost calculation:**
```
Sequential scan cost:
  = seq_page_cost × number_of_pages + cpu_tuple_cost × number_of_rows
  = 1.0 × 500 + 0.01 × 50,000
  = 500 + 500 = 1000 cost units

Index scan cost (unique index, 1 row):
  = random_page_cost × index_height + random_page_cost × 1
  = 4.0 × 3 + 4.0 × 1
  = 12 + 4 = 16 cost units
```

Winner: Index scan. 62× cheaper.

Output: an **Execution Plan** — a tree of physical operations.

### Stage 6 — Executor

The executor walks the plan tree and physically executes each node,
pulling rows from the bottom up:

```
Limit
  └── Sort (by price)
        └── Hash Join
              ├── Seq Scan on orders
              └── Hash
                    └── Index Scan on customers
```

The executor asks each node for rows one at a time (pipeline model).
The Sort node buffers all rows first, then passes them sorted.
The Index Scan reads from disk (or shared_buffers) on demand.

### Stage 7 — Result Returned

The executor returns rows to the backend process,
which formats them into PostgreSQL's wire protocol
and sends them back over TCP to your application.

```
Full journey time for simple index lookup:
  Parsing:    ~0.05ms
  Analysis:   ~0.05ms
  Rewriting:  ~0.01ms
  Planning:   ~0.1ms
  Execution:  ~0.2ms  (index scan, 1 row in cache)
  Network:    ~0.1ms
  ─────────────────
  Total:      ~0.5ms
```

---

## 1.4 Shared Buffers — The Database Cache

Reading from disk is slow (1–10ms). Reading from RAM is fast (100ns).
PostgreSQL maintains a **shared memory pool** called `shared_buffers`
that caches frequently-accessed disk pages in RAM.

```
Without shared_buffers (every query hits disk):
  SELECT price FROM products WHERE id=1  →  disk read  →  10ms

With shared_buffers (page already cached):
  SELECT price FROM products WHERE id=1  →  RAM read   →  0.01ms
  (1000× faster)
```

### How the Buffer Pool Works

```
shared_buffers (e.g., 4GB = 512,000 pages of 8KB each)

┌──────┬──────┬──────┬──────┬──────┬──────┐
│ pg0  │ pg1  │ pg2  │ FREE │ pg4  │ pg5  │  ← buffer slots
└──────┴──────┴──────┴──────┴──────┴──────┘

When a query needs page 42 from table "orders":
  1. Check buffer pool: is page 42 here? (buffer lookup via hash table)
     → YES (cache hit):  return immediately (fast path)
     → NO  (cache miss): read from disk into a free buffer slot
```

### What Happens When the Buffer Pool is Full

PostgreSQL uses a **clock-sweep algorithm** (approximation of LRU) to evict pages:

```
Each buffer has a "usage count" (0-5)
Clock hand sweeps around the ring:
  - If usage_count > 0: decrement it, skip
  - If usage_count = 0: evict this page (write to disk if dirty)

Frequently accessed pages keep getting their count incremented,
so they survive evictions. Cold pages drain to 0 and get replaced.
```

### Tuning shared_buffers

```sql
-- Check current setting
SHOW shared_buffers;

-- Check cache hit ratio (should be > 99% in production)
SELECT
    sum(heap_blks_hit) AS cache_hits,
    sum(heap_blks_read) AS disk_reads,
    ROUND(100.0 * sum(heap_blks_hit) /
          NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) AS hit_ratio_pct
FROM pg_stathio_user_tables;

-- Rule of thumb: set to 25% of RAM
-- 16GB RAM → shared_buffers = 4GB
-- In postgresql.conf:
-- shared_buffers = 4GB
-- effective_cache_size = 12GB  (hint to planner: total available RAM for caching)
```

---

## 1.5 WAL — Write-Ahead Log

WAL (Write-Ahead Log) is PostgreSQL's mechanism for **durability and crash recovery**.

### The Problem WAL Solves

Imagine PostgreSQL is writing a row to page 42 on disk.
Halfway through writing, the server crashes (power failure, kernel panic).
Page 42 now contains half-old data and half-new data. It is corrupted.

Without WAL, your data is gone or corrupted.

### The WAL Solution

**The rule:** Before modifying any data page, PostgreSQL MUST write
a description of the change to the WAL file first.

```
Without WAL:
  1. Modify data page in memory
  2. Flush data page to disk
  ↑ Crash here = corrupted data

With WAL:
  1. Write change description to WAL buffer (in memory)
  2. Flush WAL buffer to WAL file on disk  ← this MUST complete before COMMIT returns
  3. Modify data page in memory
  4. Eventually flush data page to disk (asynchronously, in background)
  ↑ Crash at step 3 or 4 = no problem:
    On restart, PostgreSQL reads WAL and replays the change
```

The WAL file is a sequential append-only log, making it very fast to write
(no seeking, no random I/O). Data pages are written later in the background.

```sql
-- WAL files live here:
-- $PGDATA/pg_wal/000000010000000000000001
--               000000010000000000000002
--               ... (each file is 16MB by default)

-- Check WAL location
SELECT pg_current_wal_lsn();  -- LSN = Log Sequence Number
-- Result: 0/3A12B410  (unique identifier for position in WAL stream)

-- How far behind is replication?
SELECT client_addr, write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```

### Background Processes That Work With WAL

| Process | Job |
|---------|-----|
| **WAL writer** | Periodically flushes WAL buffers to disk (even without COMMIT) |
| **Checkpointer** | Periodically writes all dirty data pages to disk ("checkpoint"). After checkpoint, WAL files before this point can be recycled |
| **Background writer** | Slowly writes dirty pages to disk in the background, reducing I/O spikes at checkpoint |

---

## 1.6 The Process Model in Practice

```
Connection arrives
      ↓
postmaster forks backend process
      ↓
Backend holds its own:
  - local memory (work_mem for sorts/hashes)
  - connection state
  - transaction state
  - local copy of plan cache
      ↓
Backend shares with all other backends:
  - shared_buffers (page cache)
  - WAL buffers
  - lock table
  - stats counters
      ↓
Backend dies when client disconnects
```

**Why this matters for you:**
- Setting `work_mem = 256MB` with 200 connections = potentially 51GB RAM usage
- Use PgBouncer transaction pooling to keep real connection count low (e.g., 20 backends for 500 app threads)

---

## Chapter 1 Summary

| Concept | One-Line Takeaway |
|---------|------------------|
| Process model | One OS process per connection — expensive, use a pooler |
| Query lifecycle | Parse → Analyze → Rewrite → Plan → Execute → Return |
| Planner | Generates all possible plans, picks cheapest by cost estimate |
| Shared buffers | Page cache in RAM — tune to 25% of RAM, monitor hit ratio |
| WAL | Write-ahead log for crash safety — WAL is written before data pages |
| Background workers | WAL writer, checkpointer, bgwriter — keep database healthy silently |

---

# CHAPTER 2 — ACID Properties

---

## 2.1 WHY ACID Exists

Databases are shared. Multiple people and processes read and write simultaneously.
Without rules, this causes nightmares:

```
Scenario 1 (no atomicity):
  Transfer ₹500 from account A to account B
  Step 1: Deduct ₹500 from A → success
  Step 2: Add ₹500 to B    → server crashes!
  Result: ₹500 has vanished. A is deducted, B never received it.

Scenario 2 (no isolation):
  Ticket booking: only 1 ticket left
  User 1 checks: 1 ticket available → confirms
  User 2 checks: 1 ticket available → confirms (at same millisecond)
  Both get "booking confirmed" → now 2 people hold 1 seat

Scenario 3 (no durability):
  Payment confirmed to customer
  Server crashes before writing to disk
  On restart: payment record is gone, but customer was charged
```

ACID properties are the contract PostgreSQL makes with you:
these scenarios cannot happen.

---

## 2.2 WHAT ACID Means

ACID stands for: **Atomicity, Consistency, Isolation, Durability.**

These four properties together define what it means for a database
to be "reliable" in the presence of concurrent access and failures.

---

## 2.3 HOW Atomicity Works

### What It Means
A transaction is **all-or-nothing**. Either every operation in the
transaction succeeds and is committed, or the entire transaction
is rolled back as if it never happened.

### The Problem Without It

```sql
-- Place an order: 3 operations must all succeed or none should
UPDATE products SET stock_quantity = stock_quantity - 2 WHERE id = 1;  -- step 1
INSERT INTO orders (customer_id, total_amount) VALUES (5, 199.98);      -- step 2
INSERT INTO order_items (...) VALUES (...);                             -- step 3
-- What if step 2 succeeds but step 3 fails?
-- Stock is deducted but no order record exists → inventory is wrong
```

### How PostgreSQL Implements Atomicity

PostgreSQL uses WAL for atomicity. Every change within a transaction
is tagged with the same **Transaction ID (XID)**. If the transaction
is rolled back:

```
BEGIN;  → XID = 5001 assigned

UPDATE products ...    → WAL record: "XID 5001: update page 42 slot 3"
INSERT INTO orders ... → WAL record: "XID 5001: insert page 87 slot 1"
INSERT INTO order_items ... → ERROR: foreign key violation!

ROLLBACK;
→ WAL record: "XID 5001: ABORT"
→ Any changes made by XID 5001 are invisible to all other transactions
→ The updates are not applied to the heap (or if they were, MVCC hides them)
```

No "undo" operation is needed for other transactions because MVCC's
visibility rules simply exclude rows inserted/updated by aborted transactions.

```sql
-- Demonstration
BEGIN;
  UPDATE products SET stock_quantity = 0 WHERE id = 1;
  SELECT stock_quantity FROM products WHERE id = 1;  -- shows 0
ROLLBACK;

SELECT stock_quantity FROM products WHERE id = 1;  -- shows original value
-- As if the UPDATE never happened
```

---

## 2.4 HOW Consistency Works

### What It Means
A transaction moves the database from one **valid state** to another valid state.
"Valid" means all defined constraints are satisfied: primary keys, foreign keys,
CHECK constraints, NOT NULL constraints, UNIQUE constraints.

### How PostgreSQL Implements Consistency

Constraints are checked at specific points during the transaction:

```sql
-- IMMEDIATE constraints: checked after each statement
ALTER TABLE products
  ADD CONSTRAINT ck_price_positive CHECK (price > 0)
  DEFERRABLE INITIALLY IMMEDIATE;  -- default behavior

BEGIN;
  INSERT INTO products (name, sku, price) VALUES ('Test', 'T1', -5);
  -- ERROR immediately: new row violates check constraint "ck_price_positive"
  -- Transaction is aborted right here
ROLLBACK;

-- DEFERRED constraints: checked only at COMMIT time
ALTER TABLE order_items
  ADD CONSTRAINT fk_product FOREIGN KEY (product_id) REFERENCES products(id)
  DEFERRABLE INITIALLY DEFERRED;

BEGIN;
  INSERT INTO order_items (order_id, product_id, ...) VALUES (1, 9999, ...);
  -- No error yet! FK check is deferred
  INSERT INTO products (id, name, ...) VALUES (9999, 'New Product', ...);
  -- Now product 9999 exists
COMMIT;
-- FK check runs here: product 9999 exists → OK ✅
-- Useful when inserting rows with circular references
```

**Important nuance:** Consistency is partially the database's job (constraints)
and partially the application's job (business rules the database cannot enforce).
The database ensures the rules you define are never violated.
You are responsible for defining the right rules.

---

## 2.5 HOW Isolation Works

### What It Means
Concurrent transactions behave as if they were running **serially** (one at a time).
Each transaction sees a consistent snapshot of the database; it cannot see
partial work from other in-progress transactions.

### The Isolation Problem Without It

```
T1 begins
T1: reads account balance = ₹1000
T2 begins
T2: deducts ₹400 → balance = ₹600 → COMMIT
T1: deducts ₹800 (based on its reading of ₹1000) → sets balance = ₹200 → COMMIT
Result: balance = ₹200 but should be ₹1000 - ₹400 - ₹800 = -₹200 (or error)
```

### How PostgreSQL Implements Isolation — MVCC

PostgreSQL uses **Multi-Version Concurrency Control (MVCC)**.
The core idea: instead of locking rows when reading,
PostgreSQL keeps **multiple versions** of each row and shows
each transaction the version it is supposed to see.

```
Initial state: product id=1, price=99, (xmin=100, xmax=0)

T1 (XID=200) begins
T2 (XID=201) begins

T2: UPDATE products SET price=149 WHERE id=1
    → OLD tuple: (xmin=100, xmax=201, price=99)   ← T2 set xmax=201
    → NEW tuple: (xmin=201, xmax=0,   price=149)  ← T2 created new version
T2: COMMIT

T1: SELECT price FROM products WHERE id=1
    → Sees tuple with xmin=100, xmax=201
    → Is xmax=201 committed? YES (T2 committed)
    → Was T2 committed BEFORE T1 started? YES (201 > 200? No... wait)
```

The visibility rule is subtle. Let me explain it precisely:

```
When T1 (XID=200) reads a tuple, it asks:
  "Should I see this version of the row?"

A tuple is VISIBLE to T1 if:
  1. xmin is committed AND xmin < T1's snapshot_xmin
     (the row was created by a transaction that committed before T1 started)
  2. AND xmax is 0, OR xmax is NOT committed, OR xmax > T1's snapshot_xmax
     (the row has NOT been deleted/updated by a committed transaction that T1 knows about)

Each transaction gets a snapshot at START containing:
  - xmin: lowest active XID at start time
  - xmax: highest XID + 1 at start time
  - xip_list: list of active (in-progress) XIDs at start time
```

Let us trace the example again properly:

```
Timeline:
  XID 100: original INSERT of product (committed long ago)
  XID 200: T1 starts (snapshot: xmin=200, xmax=202, xip=[200])
  XID 201: T2 starts and commits UPDATE

T1's snapshot says: "I see all transactions committed before XID 200"

OLD tuple (xmin=100, xmax=201):
  → xmin=100: committed before T1's xmin=200? YES → this version was created
  → xmax=201: is 201 in T1's xip? NO. Is 201 < T1's xmax=202? YES.
               Was 201 committed AFTER T1 started? YES.
               So from T1's perspective, this deletion "hasn't happened yet"
  → T1 SEES price=99 (old version) ✅

NEW tuple (xmin=201, xmax=0):
  → xmin=201: is 201 in T1's xip? NO. But is 201 >= T1's xmax=202? NO (201 < 202).
               Hmm... this gets complex. The actual rule: T1 does not see rows
               inserted by transactions that started after T1's snapshot.
  → T1 DOES NOT SEE price=149

Result: T1 consistently sees price=99 even though T2 already committed price=149.
This is correct "Read Committed" behavior — T1 would see 149 on its NEXT statement.
Under "Repeatable Read" — T1 would see 99 for the ENTIRE transaction.
```

**The beautiful outcome:** Readers never block writers. Writers never block readers.
This is fundamentally different from lock-based systems.

```sql
-- You can observe MVCC yourself:
-- In terminal 1:
BEGIN;
SELECT price FROM products WHERE id=1;  -- see 99

-- In terminal 2 (while terminal 1's transaction is open):
UPDATE products SET price=149 WHERE id=1;
COMMIT;

-- Back in terminal 1:
SELECT price FROM products WHERE id=1;
-- Under READ COMMITTED:  returns 149 (new statement, new snapshot)
-- Under REPEATABLE READ: returns 99  (entire transaction uses same snapshot)

COMMIT;
```

---

## 2.6 HOW Durability Works

### What It Means
Once a transaction is **committed**, the data will not be lost,
even if the server crashes immediately after the COMMIT returns.

### How PostgreSQL Implements Durability

The mechanism is WAL (from Chapter 1), specifically the `fsync` system call.

```
Your app:  COMMIT;

PostgreSQL:
  1. Write COMMIT record to WAL buffer
  2. Call fsync() on WAL file — this forces OS to write WAL to physical disk
                                 (not just OS page cache — actual disk!)
  3. ONLY NOW return success to your application
  4. Later: dirty data pages are written to disk asynchronously

Crash happens between step 3 and 4:
  → WAL is on disk (step 2 ensured this)
  → Data pages may be stale
  → On restart: PostgreSQL replays WAL from last checkpoint
  → Data pages are reconstructed
  → Database returns to consistent committed state
```

**The durability performance tradeoff:**

`fsync=on` (default): every COMMIT waits for disk write → slow but safe
`fsync=off`: COMMIT returns immediately, WAL written asynchronously → fast but data loss on crash

**Never use `fsync=off` in production.** Use it only for bulk loading throwaway data.

```sql
-- Check fsync setting
SHOW fsync;             -- should be "on"
SHOW synchronous_commit; -- "on" = wait for WAL flush before returning
                          -- "off" = return before WAL flush (small window of data loss)
                          -- "local" = wait for local WAL, not replicas

-- synchronous_commit=off is a safe compromise:
-- → Data loss window: wal_writer_delay (default 200ms)
-- → But no corruption (unlike fsync=off)
-- → Good for non-critical high-throughput writes (logging, analytics)
```

---

## 2.7 MVCC and Dead Tuples — The Side Effect of Isolation

Because MVCC never overwrites rows (it creates new versions),
old row versions accumulate on disk. These are called **dead tuples**.

```
After 1000 updates to product id=1:
  The heap file for products contains 1001 versions of this row
  Only 1 is visible (the latest)
  1000 are dead (xmax is set, transaction committed)

These dead tuples:
  → Waste disk space
  → Slow down sequential scans (must be read and skipped)
  → Cause index bloat (dead index entries point to dead tuples)
```

**VACUUM** is the process that cleans up dead tuples. Without it,
tables grow indefinitely. PostgreSQL runs VACUUM automatically
(autovacuum), but you must understand it to configure it correctly.
We cover this in Chapter 25.

---

## Chapter 2 Summary

| Property | Implementation | Key Insight |
|----------|---------------|-------------|
| **Atomicity** | WAL + XID tagging | Aborted transactions are invisible (MVCC) |
| **Consistency** | Constraints (immediate/deferred) | DB ensures rules you define; you must define the right rules |
| **Isolation** | MVCC snapshots | Readers never block writers; multiple row versions exist simultaneously |
| **Durability** | WAL + fsync | COMMIT waits for WAL to hit physical disk before returning |

The cost of ACID: dead tuples from MVCC require VACUUM; WAL fsync
adds ~1–10ms latency to each COMMIT; constraint checking adds CPU overhead.
These are worthwhile costs for correctness. You can tune them per use case.

---

# CHAPTER 3 — Data Types & Storage

---

## 3.1 WHY Storage Internals Matter

You do not need to understand storage to write queries.
But you DO need to understand it to answer these questions:

- Why is my table 50GB when the data should be 10GB?
- Why does this query slow down after millions of updates?
- Why does fetching a column with long text take longer than other columns?
- How does NULL actually get stored?
- Why can I not store more than ~1.6KB in a single row without TOAST kicking in?

This chapter answers all of these.

---

## 3.2 How PostgreSQL Organizes Data on Disk

### The File Hierarchy

Every database object has a physical file (or set of files) on disk:

```
$PGDATA/
├── base/
│   ├── 1/                    ← template1 database (OID=1)
│   ├── 16384/                ← your database (OID=16384)
│   │   ├── 16385             ← products table heap file
│   │   ├── 16385_fsm         ← free space map for products
│   │   ├── 16385_vm          ← visibility map for products
│   │   ├── 16386             ← products_pkey index
│   │   ├── 16387             ← ix_products_sku index
│   │   └── ...
│   └── pg_filenode.map
├── pg_wal/                   ← WAL files
│   ├── 000000010000000000000001
│   └── 000000010000000000000002
├── pg_xact/                  ← transaction commit/abort status
├── postgresql.conf
└── pg_hba.conf

-- Find the file for your table:
SELECT pg_relation_filepath('products');
-- Result: base/16384/16385
```

### The Page — PostgreSQL's Unit of I/O

PostgreSQL does not read individual rows from disk.
It reads **pages** (also called blocks). Every page is **exactly 8192 bytes (8KB)**.

This has a profound implication: if you need one row from a page,
you read the entire 8KB page. If the page has 100 rows and you need 1,
you still read 100 rows' worth of disk I/O.

---

## 3.3 Page Layout — What Is Inside an 8KB Page

```
┌──────────────────────────────────────────────────┐  ← offset 0
│              PAGE HEADER (24 bytes)               │
│  pd_lsn       : 8 bytes (WAL log sequence number) │
│  pd_checksum  : 2 bytes (data integrity check)    │
│  pd_flags     : 2 bytes (page state flags)        │
│  pd_lower     : 2 bytes (offset to start of free) │
│  pd_upper     : 2 bytes (offset to end of free)   │
│  pd_special   : 2 bytes (offset to special space) │
│  pd_pagesize  : 2 bytes (should always be 8192)   │
├──────────────────────────────────────────────────┤
│           ITEM ID ARRAY (line pointers)           │
│  [lp1: offset=8150, len=42] ← points to tuple 1  │
│  [lp2: offset=8100, len=50] ← points to tuple 2  │
│  [lp3: offset=8040, len=60] ← points to tuple 3  │
│  ...                                              │
│       ← pd_lower points here                     │
├──────────────────────────────────────────────────┤
│                                                   │
│              FREE SPACE                           │
│           (pd_upper - pd_lower bytes)             │
│                                                   │
├──────────────────────────────────────────────────┤
│  Tuple 3 (row data, 60 bytes)  ← newest rows     │
│  Tuple 2 (row data, 50 bytes)    are at higher    │
│  Tuple 1 (row data, 42 bytes)    addresses        │
│       ← pd_upper points here                     │
├──────────────────────────────────────────────────┤
│         SPECIAL SPACE (for indexes only)          │
└──────────────────────────────────────────────────┘  ← offset 8191
```

**Key observations:**
- Item IDs grow downward from the top (after the header)
- Tuples (rows) grow upward from the bottom
- Free space is in the middle
- When `pd_lower >= pd_upper`, the page is full — new rows go to a new page
- The `ctid` (physical row identifier) is `(page_number, item_id_offset)`

```sql
-- See the ctid of every row in products:
SELECT ctid, id, name FROM products LIMIT 5;
--  ctid  | id |   name
-- -------+----+---------
-- (0,1)  |  1 | Widget       ← page 0, slot 1
-- (0,2)  |  2 | Gadget       ← page 0, slot 2
-- (0,3)  |  3 | Thing        ← page 0, slot 3
-- (1,1)  |  4 | Doohickey    ← page 1, slot 1  (page 0 was full)
-- (1,2)  |  5 | Whatsit      ← page 1, slot 2
```

---

## 3.4 Tuple Layout — What Is Inside a Row

Each tuple (row) has a **24-byte header** followed by optional **null bitmap**
and then the actual column data:

```
TUPLE HEADER (23 bytes):
┌─────────────┬───────────────────────────────────────────────────┐
│ t_xmin      │ 4 bytes — XID of the transaction that created this │
│             │ tuple (INSERT or UPDATE that made this version)     │
├─────────────┼───────────────────────────────────────────────────┤
│ t_xmax      │ 4 bytes — XID of the transaction that deleted or   │
│             │ updated this tuple (0 if still live)               │
├─────────────┼───────────────────────────────────────────────────┤
│ t_field3    │ 4 bytes — command ID (for statement-level          │
│             │ visibility within a transaction)                    │
├─────────────┼───────────────────────────────────────────────────┤
│ t_ctid      │ 6 bytes — physical location of this tuple (or the  │
│             │ newer version if this one was updated)             │
├─────────────┼───────────────────────────────────────────────────┤
│ t_infomask  │ 2 bytes — flags (has null, has varlen, xmin/xmax   │
│ t_infomask2 │ 2 bytes   committed flags, frozen flag, etc.)      │
├─────────────┼───────────────────────────────────────────────────┤
│ t_hoff      │ 1 byte  — offset to actual data (skips null bitmap)│
└─────────────┴───────────────────────────────────────────────────┘

NULL BITMAP (optional, 1 bit per column):
  Present only if any column is NULL
  1 bit per column: 1 = not null, 0 = null
  Padded to a multiple of 8 bits

ACTUAL COLUMN DATA:
  Fixed-length types (int4, int8, float8, bool) stored directly
  Variable-length types (text, varchar, bytea) stored as varlena
  Alignment padding between columns
```

### The Hidden Cost of Alignment

PostgreSQL aligns column data to natural boundaries (int4 to 4-byte boundary,
int8 to 8-byte boundary). This means **column order affects row size**:

```sql
-- Wasteful layout (alignment padding wastes space):
CREATE TABLE bad_layout (
    a BOOL,      -- 1 byte + 7 bytes padding (aligns next int8)
    b INT8,      -- 8 bytes
    c BOOL,      -- 1 byte + 7 bytes padding
    d INT8,      -- 8 bytes
    e BOOL       -- 1 byte + 3 bytes padding
);
-- Total: 1+7+8+1+7+8+1+3 = 36 bytes per row

-- Efficient layout (largest types first):
CREATE TABLE good_layout (
    b INT8,      -- 8 bytes (no padding needed at start)
    d INT8,      -- 8 bytes
    a BOOL,      -- 1 byte
    c BOOL,      -- 1 byte
    e BOOL       -- 1 byte + 5 bytes padding (to 8-byte row boundary)
);
-- Total: 8+8+1+1+1+5 = 24 bytes per row
-- 33% smaller! More rows per page = fewer page reads = faster queries
```

**Practical rule:** Order columns as: largest fixed types first (int8, float8, timestamps),
then medium (int4, float4), then small (int2, bool, char), then variable-length (text, bytea).

---

## 3.5 NULL Storage

NULL values in PostgreSQL are **not stored in the column data area** at all.
They are recorded in the **null bitmap** in the tuple header:

```
Row: (id=1, name='Widget', price=99.99, description=NULL, category=NULL)

Without NULLs: tuple stores all 5 column values
With 2 NULLs:  tuple stores 3 column values + null bitmap marking positions 4 and 5

The description and category columns take ZERO bytes of storage when NULL.
```

**Implication:** NULL is actually very storage-efficient. An empty string `''` wastes
space; NULL does not (beyond the null bitmap which is tiny).

```sql
-- Comparing NULL storage:
CREATE TABLE t (id INT, val TEXT);
INSERT INTO t VALUES (1, NULL);         -- stores "null" (0 bytes for val)
INSERT INTO t VALUES (2, '');           -- stores "" (4 bytes varlena header + 0 data = 4 bytes for val)
INSERT INTO t VALUES (3, 'hello');      -- stores "hello" (4+5 = 9 bytes)

-- pg_column_size shows storage size
SELECT id, pg_column_size(val) FROM t;
--  id | pg_column_size
-- ----+----------------
--   1 | NULL            ← NULL returns NULL here (not stored)
--   2 | 4               ← empty string: 4-byte varlena header only
--   3 | 9               ← 4-byte header + 5 data bytes
```

---

## 3.6 TOAST — The Oversized Attribute Storage Technique

### The Problem

A page is 8KB. A tuple header is 24 bytes. With reasonable overhead,
a single column value cannot exceed about **2000 bytes** before TOAST kicks in
(the exact threshold is `BLCKSZ / 4` = 2048 bytes).

But text columns, JSONB, and bytea can easily be megabytes or gigabytes.
How does PostgreSQL handle values larger than a page?

### The Solution: TOAST

TOAST is a transparent mechanism that handles oversized values automatically.
When a value is too large to fit inline, PostgreSQL:

1. **Tries to compress it** (using pglz or lz4 algorithm)
2. **If still too large, slices it** into ~2000-byte chunks and stores them
   in a separate **TOAST table** (hidden from you)
3. **Stores a pointer** in the main row that points to the TOAST chunks

```
Main table: products
  Row: id=1, name='Widget', price=99.99, description=<TOAST pointer>
  The description column contains a pointer to TOAST, not the actual text

TOAST table: pg_toast.pg_toast_16385  (hidden, auto-created)
  chunk_id | chunk_seq | chunk_data
  12345    | 0         | <first 2000 bytes of description>
  12345    | 1         | <next 2000 bytes>
  12345    | 2         | <remaining bytes>
```

### TOAST Storage Strategies

You can control how PostgreSQL handles each column:

```sql
-- See current storage strategy for all columns
SELECT attname, attstorage
FROM pg_attribute
WHERE attrelid = 'products'::regclass AND attnum > 0;
-- attstorage: 'p'=PLAIN, 'e'=EXTERNAL, 'x'=EXTENDED, 'm'=MAIN

-- PLAIN: no compression, no out-of-line storage
--   Use for: fixed-length types (int, float, bool) — they can't be compressed
ALTER TABLE products ALTER COLUMN id SET STORAGE PLAIN;

-- EXTENDED (default for text, bytea, jsonb): compress first, then out-of-line
--   Best for: text content that compresses well (logs, descriptions, JSON)
ALTER TABLE products ALTER COLUMN description SET STORAGE EXTENDED;

-- EXTERNAL: out-of-line without compression
--   Use for: data that is already compressed (PNG/JPG images, gzip files)
--   Advantage: substring operations on EXTERNAL are faster (no decompress needed)
ALTER TABLE products ALTER COLUMN image_data SET STORAGE EXTERNAL;

-- MAIN: compress if possible, prefer inline (out-of-line only as last resort)
--   Use for: columns you want to keep in the main row if possible
ALTER TABLE products ALTER COLUMN metadata SET STORAGE MAIN;
```

### The Performance Impact of TOAST

```sql
-- Fetching a TOASTed column requires a SECOND lookup:
-- 1. Read main table page → get TOAST pointer
-- 2. Read TOAST table chunks → reassemble value → decompress

-- This is why SELECT * is expensive when large columns exist:
SELECT * FROM articles;                          -- reads ALL TOAST data
SELECT id, title, author FROM articles;          -- TOAST columns skipped entirely

-- Always select only the columns you need
-- This also reduces network bandwidth
```

### Checking TOAST Tables

```sql
-- Find the TOAST table for products
SELECT relname, reltoastrelid,
       (SELECT relname FROM pg_class WHERE oid = c.reltoastrelid) AS toast_table
FROM pg_class c
WHERE relname = 'products';

-- Check size of TOAST data
SELECT
    relname AS main_table,
    pg_size_pretty(pg_relation_size(oid)) AS main_size,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
    pg_size_pretty(pg_total_relation_size(oid) - pg_relation_size(oid)) AS toast_and_index_size
FROM pg_class
WHERE relname = 'products';
```

---

## 3.7 Tuple Versioning in Detail

We touched on MVCC in Chapter 2. Now let us see exactly what happens
in the heap file when you INSERT, UPDATE, and DELETE rows.

### INSERT

```sql
BEGIN;  -- XID = 500
INSERT INTO products (id, name, price) VALUES (1, 'Widget', 99.99);
COMMIT;

-- Heap now contains:
-- ctid=(0,1), t_xmin=500, t_xmax=0, t_infomask=XMIN_COMMITTED
-- id=1, name='Widget', price=99.99
```

### UPDATE

```sql
BEGIN;  -- XID = 600
UPDATE products SET price = 149.99 WHERE id = 1;
COMMIT;

-- Heap now contains TWO versions:
-- OLD: ctid=(0,1), t_xmin=500, t_xmax=600, t_infomask=XMIN_COMMITTED|XMAX_COMMITTED
--      t_ctid points to (0,2)  ← tells readers where the new version is
-- NEW: ctid=(0,2), t_xmin=600, t_xmax=0,   t_infomask=XMIN_COMMITTED
--      id=1, name='Widget', price=149.99
```

The old tuple at `(0,1)` is still on disk. It is invisible to new transactions
(because its `t_xmax=600` is committed). It is a **dead tuple** waiting for VACUUM.

### DELETE

```sql
BEGIN;  -- XID = 700
DELETE FROM products WHERE id = 1;
COMMIT;

-- Heap still shows:
-- NEW (from before): ctid=(0,2), t_xmin=600, t_xmax=700, XMAX_COMMITTED
-- Row is marked deleted but still physically exists until VACUUM runs
```

### What VACUUM Does

```sql
-- Before VACUUM: page contains old dead tuples
-- (0,1): DEAD (xmax=600, committed)
-- (0,2): DEAD (xmax=700, committed)

VACUUM products;

-- After VACUUM: page contains only free space
-- (0,1): FREE
-- (0,2): FREE
-- Free space is recorded in the FSM (free space map) so new inserts can reuse it
```

### Watching It Live

```sql
-- Check dead tuples accumulating
SELECT relname, n_live_tup, n_dead_tup,
       ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
WHERE relname = 'products';

-- Before lots of updates:
-- relname  | n_live_tup | n_dead_tup | dead_pct
-- products |    10000   |     0      |   0.0

-- After 5000 UPDATE statements on products:
-- products |    10000   |   5000     |  33.3

-- After VACUUM:
-- products |    10000   |     0      |   0.0
```

---

## 3.8 Free Space Map (FSM) and Visibility Map (VM)

### Free Space Map

When PostgreSQL needs to INSERT a row, it needs to find a page with enough free space.
Without a map, it would have to read every page in the table — very slow.

The FSM maintains a **coarse approximation** of free space per page:

```
FSM for products table:
  page 0: ~1000 bytes free
  page 1: ~500 bytes free
  page 2: ~2000 bytes free   ← INSERT will prefer this page
  page 3: full (0 bytes free)
  ...

VACUUM updates the FSM after reclaiming dead tuple space.
Checkpointer and background writer use the FSM when deciding where to write.
```

### Visibility Map

The VM is a tiny bitmap with **2 bits per page**:

```
Bit 1 (all-visible):  ALL tuples on this page are visible to ALL transactions
                      → Index-Only Scan can skip fetching this page!
Bit 2 (all-frozen):   ALL tuples have frozen XIDs
                      → Safe from XID wraparound
```

```sql
-- Check visibility map usage
SELECT relname,
       heap_blks_read,
       heap_blks_hit,
       idx_blks_read,
       heap_blks_hit + heap_blks_read AS total_heap_accesses
FROM pg_stathio_user_tables
WHERE relname = 'products';
```

The all-visible bit is what makes **Index-Only Scans** fast: when the bit is set,
PostgreSQL trusts that any row the index finds is visible without checking the heap.
After heavy updates, many pages lose the all-visible bit → Index-Only Scans
degrade to regular Index Scans. Running VACUUM restores the all-visible bits.

---

## Chapter 3 Summary

| Concept | Key Detail |
|---------|-----------|
| **Page** | 8KB unit of I/O. Reading 1 row = reading its entire 8KB page |
| **Tuple header** | 24 bytes of overhead per row: xmin, xmax, ctid, flags |
| **NULL** | Stored as a bit in the null bitmap — takes zero data storage |
| **Column alignment** | Order columns: largest fixed types first to minimize padding waste |
| **TOAST** | Transparent for values > ~2KB: compress + slice into separate table |
| **Dead tuples** | Every UPDATE and DELETE leaves a dead tuple — VACUUM cleans them |
| **FSM** | Maps free space per page so INSERT finds room without a full scan |
| **VM** | Marks pages all-visible or all-frozen — enables fast Index-Only Scans |

---

# CHAPTER 4 — Normalization

---

## 4.1 WHY Normalization Exists

Normalization is not a bureaucratic exercise.
It solves three concrete bugs that occur when data is stored redundantly.

### Bug 1: Update Anomaly

```sql
-- Un-normalized orders table (customer data embedded):
id | customer_name | customer_email   | product_name | quantity
1  | Rahul Kumar   | rahul@gmail.com  | Widget       | 2
2  | Rahul Kumar   | rahul@gmail.com  | Gadget       | 1
3  | Rahul Kumar   | rahul@gmail.com  | Doohickey    | 3

-- Rahul changes his email:
UPDATE orders SET customer_email = 'rahulk@work.com' WHERE customer_name = 'Rahul Kumar';
-- If you forget row 3, your data is now inconsistent:
-- Two rows show gmail, one shows work — which is correct?
```

### Bug 2: Insertion Anomaly

```sql
-- Same un-normalized table
-- You want to add a new customer who hasn't placed an order yet
INSERT INTO orders (customer_name, customer_email, ...) VALUES ('Priya', 'priya@x.com', ???, ???);
-- You cannot! The table requires a product and quantity.
-- There is no place to store a customer without an order.
```

### Bug 3: Deletion Anomaly

```sql
-- Last order for customer Vikram is deleted
DELETE FROM orders WHERE id = 7;
-- Vikram's email, name — all of his data — is now lost
-- because it only existed as part of this order row
```

**Normalization prevents all three anomalies** by ensuring each fact
is stored exactly once, in exactly the right place.

---

## 4.2 WHAT Normalization Is

Normalization is the process of structuring a relational database schema
to reduce data redundancy and improve data integrity.

It is applied as a series of **Normal Forms** (1NF, 2NF, 3NF, BCNF...),
each building on the previous and eliminating a specific class of redundancy.

To understand normal forms, you must first understand **Functional Dependencies**.

---

## 4.3 Functional Dependencies — The Foundation

A **functional dependency** X → Y means:
"Given any value of X, there is exactly one corresponding value of Y."
Or: "X uniquely determines Y."

```
Examples in a student database:

  student_id → student_name    ← one student_id maps to one name ✅
  student_id → dob             ← one ID maps to one date of birth ✅
  student_id → student_name    ← functional dependency
  student_name → student_id    ← NOT a functional dependency
                                  (two students can have same name)

  zip_code → city              ← one zip code = one city ✅
  city → zip_code              ← NOT (Mumbai has many zip codes)

  {order_id, product_id} → quantity   ← composite key: both together determine quantity
  order_id → quantity                 ← NOT (one order has multiple products/quantities)
  product_id → product_name           ← one product ID = one product name
```

**Why this matters:** A normal form violation is always a hidden functional dependency
that is not properly expressed in the table structure.
Finding anomalies = finding undeclared functional dependencies.

---

## 4.4 First Normal Form (1NF)

### What It Requires

1. Every column stores **atomic** (indivisible) values — no lists, arrays, or sets
2. Every column stores values of a **single type** — no mixing
3. Every row is **uniquely identifiable** (has a primary key)
4. **No repeating column groups** — no phone1, phone2, phone3

### What Violates 1NF and Why It Is a Problem

```sql
-- VIOLATION 1: multi-valued attribute
students:
  id | name  | phone_numbers
  1  | Rahul | "9876543210, 8765432109"  ← two phones in one cell

Problem:
  → How do you query "find student with phone 9876543210"?
  → You must do string matching / LIKE — no index, no FK, no integrity

-- VIOLATION 2: repeating columns
orders:
  id | product_1 | qty_1 | product_2 | qty_2 | product_3 | qty_3
  1  | Widget    | 2     | Gadget    | 1     | NULL       | NULL

Problem:
  → What if an order has 10 products? Add product_10, qty_10?
  → Half the columns are always NULL
  → Querying "all orders containing Gadget" is complex and slow

-- SATISFIES 1NF: atomic, one value per cell
student_phones:
  student_id | phone
  1          | 9876543210
  1          | 8765432109

order_items:
  order_id | product_name | quantity
  1        | Widget       | 2
  1        | Gadget       | 1
```

---

## 4.5 Second Normal Form (2NF)

### What It Requires

1. Must be in 1NF
2. Every non-key attribute must depend on the **entire** primary key — no partial dependencies

**Partial dependency:** A non-key column depends only on part of a composite primary key.

### What Violates 2NF and Why It Is a Problem

```sql
-- Table with composite PK: (order_id, product_id)
order_items:
  order_id | product_id | quantity | product_name | product_price | customer_name
  1        | 101        | 2        | Widget       | 99.99         | Rahul Kumar
  1        | 102        | 1        | Gadget       | 149.99        | Rahul Kumar
  2        | 101        | 3        | Widget       | 99.99         | Priya Sharma

Functional dependencies:
  {order_id, product_id} → quantity          ✅ depends on full PK
  product_id → product_name                  ← depends only on product_id (PARTIAL!)
  product_id → product_price                 ← depends only on product_id (PARTIAL!)
  order_id → customer_name                   ← depends only on order_id (PARTIAL!)

Problems:
  → product_name "Widget" appears twice (rows 1 and 3) — update anomaly
  → Can't change Widget's price without finding all rows with product_id=101
  → Can't add a product without an order

-- FIX: decompose into tables where every non-key column depends on the WHOLE PK

order_items: order_id | product_id | quantity       (PK: {order_id, product_id})
products:    product_id | product_name | product_price   (PK: product_id)
orders:      order_id | customer_name                    (PK: order_id)

Now:
  quantity depends on {order_id, product_id} ✅
  product_name depends on product_id         ✅ (it IS the PK of products)
  customer_name depends on order_id          ✅ (it IS the PK of orders)
```

---

## 4.6 Third Normal Form (3NF)

### What It Requires

1. Must be in 2NF
2. No **transitive dependencies** — non-key columns must not depend on other non-key columns

**Transitive dependency:** A → B → C (A is the PK, B is non-key, C depends on B, not directly on A)

### What Violates 3NF and Why It Is a Problem

```sql
-- employees table (PK: emp_id)
employees:
  emp_id | emp_name | dept_id | dept_name    | dept_location
  1      | Rahul    | D1      | Engineering  | Bengaluru
  2      | Priya    | D1      | Engineering  | Bengaluru
  3      | Vikram   | D2      | Marketing    | Mumbai

Functional dependencies:
  emp_id → emp_name   ✅ direct
  emp_id → dept_id    ✅ direct
  dept_id → dept_name ← transitive: emp_id → dept_id → dept_name
  dept_id → dept_location ← transitive

Problems:
  → "Engineering" appears twice — update anomaly (change dept name = update all employee rows)
  → Can't store a department without employees
  → Delete all engineers → lose the fact that Engineering is in Bengaluru

-- FIX: eliminate transitive dependency by extracting dept into its own table
employees:   emp_id | emp_name | dept_id
departments: dept_id | dept_name | dept_location

Now: emp_id → dept_id (direct) ✅
     dept_id → dept_name (direct, dept_id is PK of departments) ✅
```

---

## 4.7 BCNF — Boyce-Codd Normal Form

### What It Requires

For every functional dependency X → Y in the table,
X must be a **superkey** (a key that uniquely identifies a row).

This is stronger than 3NF. The difference: 3NF allows a non-key column
to determine part of a candidate key, as long as it is not transitive.
BCNF does not allow this exception.

### When 3NF is Not Enough

```sql
-- Problem: university scheduling
-- Rules: each professor teaches exactly one subject
--        each student can study one subject per professor
-- PK candidates: {student, subject} and {student, professor}

enrollment:
  student | professor | subject
  Rahul   | Dr. Gupta | Maths
  Priya   | Dr. Gupta | Maths
  Vikram  | Dr. Singh | Physics

Functional dependencies:
  {student, professor} → subject      ← valid candidate key
  professor → subject                 ← professor determines subject
  {student, subject} → professor      ← valid candidate key

Is this 3NF? Yes — because "professor" is part of a candidate key.
Is this BCNF? NO — because in "professor → subject",
  professor is NOT a superkey (does not uniquely identify a row).

Problem:
  → Dr. Gupta teaches Maths: stored redundantly (rows 1 and 2)
  → If Dr. Gupta switches to Physics, must update multiple rows

FIX (BCNF decomposition):
  professor_subject: professor | subject        (FD: professor → subject) ✅
  student_professor: student   | professor      (PK: {student, professor})

But: we lost the FD {student, subject} → professor
     This is the BCNF trade-off: sometimes BCNF decomposition loses dependency preservation
     In practice: choose 3NF in such cases, accept the minor redundancy
```

---

## 4.8 Practical Normalization — Real Schema Design

Let us normalize a realistic scenario from scratch: an e-commerce system.

### Starting Point — Un-normalized Order Data

```
order_number | order_date | customer_name | customer_email    | customer_phone
             | shipping_addr_street | shipping_addr_city | shipping_addr_pin
             | product_name_1 | product_sku_1 | product_price_1 | qty_1
             | product_name_2 | product_sku_2 | product_price_2 | qty_2
             | product_name_3 | product_sku_3 | product_price_3 | qty_3
             | payment_method | payment_status | payment_date
             | discount_code | discount_pct
```

This violates every normal form. Let us fix it step by step.

### Step 1 — Apply 1NF

Remove repeating product groups. Every fact gets its own row:

```sql
-- Orders (no product info here)
orders:
  order_id | order_date | customer_name | customer_email | customer_phone
  | street | city | pin | payment_method | payment_status | payment_date
  | discount_code | discount_pct

-- Order items (the repeating group, now normalized)
order_items:
  order_id | product_name | product_sku | product_price | quantity
```

### Step 2 — Apply 2NF

Find partial dependencies. In order_items, PK = {order_id, product_sku}:
- `product_name` and `product_price` depend only on `product_sku` → partial dependency

```sql
-- Extract products table
products:
  product_sku | product_name | product_price

-- Clean order_items
order_items:
  order_id | product_sku | quantity
  -- PK: {order_id, product_sku}
  -- All non-key: only quantity, which depends on the full PK ✅
```

In orders, PK = order_id:
- `customer_name`, `customer_email`, `customer_phone` depend only on customer (not the order)

```sql
-- Extract customers
customers:
  customer_id | customer_name | customer_email | customer_phone

-- Clean orders (reference customer by ID)
orders:
  order_id | customer_id | order_date | street | city | pin
  | payment_method | payment_status | payment_date
  | discount_code | discount_pct
```

### Step 3 — Apply 3NF

In orders, find transitive dependencies:
- `discount_pct` depends on `discount_code` (not directly on order_id)
- `payment_status` and `payment_date` are fine (they describe the order)

```sql
-- Extract discount codes
discount_codes:
  code | discount_pct | valid_from | valid_until | max_uses

-- Clean orders
orders:
  order_id | customer_id | order_date | shipping_address_id | payment_method
  | payment_status | payment_date | discount_code_id
```

Note: the shipping address fields could also be extracted if customers can reuse addresses.

### Final Normalized Schema

```sql
CREATE TABLE customers (
    customer_id   BIGSERIAL PRIMARY KEY,
    full_name     VARCHAR(255) NOT NULL,
    email         VARCHAR(255) NOT NULL UNIQUE,
    phone         VARCHAR(20)
);

CREATE TABLE addresses (
    address_id    BIGSERIAL PRIMARY KEY,
    customer_id   BIGINT NOT NULL REFERENCES customers(customer_id),
    street        VARCHAR(255) NOT NULL,
    city          VARCHAR(100) NOT NULL,
    pin_code      VARCHAR(10) NOT NULL,
    is_default    BOOLEAN DEFAULT FALSE
);

CREATE TABLE products (
    product_id    BIGSERIAL PRIMARY KEY,
    sku           VARCHAR(100) NOT NULL UNIQUE,
    name          VARCHAR(255) NOT NULL,
    base_price    NUMERIC(10,2) NOT NULL CHECK (base_price > 0),
    stock_qty     INT NOT NULL DEFAULT 0 CHECK (stock_qty >= 0)
);

CREATE TABLE discount_codes (
    code_id       BIGSERIAL PRIMARY KEY,
    code          VARCHAR(50) NOT NULL UNIQUE,
    discount_pct  NUMERIC(5,2) NOT NULL CHECK (discount_pct BETWEEN 0 AND 100),
    valid_from    DATE NOT NULL,
    valid_until   DATE NOT NULL,
    max_uses      INT,
    current_uses  INT NOT NULL DEFAULT 0
);

CREATE TABLE orders (
    order_id       BIGSERIAL PRIMARY KEY,
    customer_id    BIGINT NOT NULL REFERENCES customers(customer_id),
    address_id     BIGINT NOT NULL REFERENCES addresses(address_id),
    discount_id    BIGINT REFERENCES discount_codes(code_id),
    order_date     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    total_amount   NUMERIC(12,2) NOT NULL,
    payment_method VARCHAR(50),
    payment_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    payment_date   TIMESTAMPTZ
);

CREATE TABLE order_items (
    item_id          BIGSERIAL PRIMARY KEY,
    order_id         BIGINT NOT NULL REFERENCES orders(order_id),
    product_id       BIGINT NOT NULL REFERENCES products(product_id),
    quantity         INT NOT NULL CHECK (quantity > 0),
    price_at_purchase NUMERIC(10,2) NOT NULL,   -- snapshot: can't use product.price
    UNIQUE (order_id, product_id)
);
```

Notice `price_at_purchase` in order_items: this is intentional denormalization.
Product prices change over time; the price at the time of purchase must be preserved.

---

## 4.9 Denormalization — When and Why to Break the Rules

Normalization is correct. Denormalization is a conscious performance trade-off.
You denormalize when a join or aggregation runs too frequently and is a bottleneck,
and the cost of maintaining the redundancy is acceptable.

### When to Denormalize

```
Query runs > 1000x per second AND
  → involves expensive JOINs across large tables
  → result changes infrequently
  → the cost of keeping redundant data synchronized is low
```

### Denormalization Technique 1: Duplicate a Column

```sql
-- Normalized: every order list page requires JOIN to customers
SELECT o.id, c.full_name, o.total_amount, o.order_date
FROM orders o
JOIN customers c ON o.customer_id = c.id
ORDER BY o.order_date DESC
LIMIT 20;

-- Denormalized: store customer_name in orders directly
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(255);
UPDATE orders o SET customer_name = c.full_name
FROM customers c WHERE c.customer_id = o.customer_id;

-- Now:
SELECT id, customer_name, total_amount, order_date
FROM orders
ORDER BY order_date DESC
LIMIT 20;
-- No JOIN needed!

-- Cost: when a customer changes their name, you must update orders too
-- Acceptable if names rarely change and the read gain is significant
```

### Denormalization Technique 2: Pre-computed Aggregate

```sql
-- Normalized: counting orders per customer requires scanning order table
SELECT c.id, c.full_name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.full_name;
-- Full table scan on orders every time!

-- Denormalized: maintain a counter on customers
ALTER TABLE customers ADD COLUMN total_orders INT NOT NULL DEFAULT 0;

-- Keep it in sync with a trigger:
CREATE OR REPLACE FUNCTION sync_order_count() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE customers SET total_orders = total_orders + 1
        WHERE customer_id = NEW.customer_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE customers SET total_orders = total_orders - 1
        WHERE customer_id = OLD.customer_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_order_count
AFTER INSERT OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION sync_order_count();

-- Now:
SELECT id, full_name, total_orders FROM customers;
-- Zero JOINs. Instant.
```

### Denormalization Technique 3: Materialized View

```sql
-- Pre-compute a complex summary that powers a dashboard
CREATE MATERIALIZED VIEW customer_stats AS
SELECT
    c.customer_id,
    c.full_name,
    c.email,
    COUNT(o.order_id) AS total_orders,
    COALESCE(SUM(o.total_amount), 0) AS lifetime_value,
    MAX(o.order_date) AS last_order_date,
    COALESCE(SUM(o.total_amount) / NULLIF(COUNT(o.order_id), 0), 0) AS avg_order_value
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.full_name, c.email;

-- Create index on the materialized view
CREATE INDEX ON customer_stats(lifetime_value DESC);

-- Refresh it (can be scheduled, e.g. every 10 minutes)
REFRESH MATERIALIZED VIEW CONCURRENTLY customer_stats;
-- CONCURRENTLY: allows reads during refresh (requires a unique index)

-- Dashboard query: instant
SELECT * FROM customer_stats ORDER BY lifetime_value DESC LIMIT 20;
```

---

## 4.10 Summary of Normal Forms

```
Start here → Un-normalized (flat, denormalized table)
                │
                ▼
            1NF: Atomic values, no repeating groups, primary key
                │  Fixes: multi-valued columns, repeating column groups
                ▼
            2NF: No partial dependencies (all non-key columns need full PK)
                │  Fixes: data duplicated because of composite keys
                │  Only matters when PK is composite
                ▼
            3NF: No transitive dependencies (non-key → non-key → non-key eliminated)
                │  Fixes: data duplicated because of non-key → non-key dependencies
                ▼
            BCNF: Every determinant is a superkey
                │  Fixes: residual anomalies in complex multi-key tables
                ▼
            4NF: No multi-valued dependencies
                │  Rarely needed in practice
                ▼
            5NF: No join dependencies (lossless decomposition)
                  Theoretical, almost never applied in practice
```

### The Practical Target

In real production systems, design to **3NF** and selectively denormalize
specific paths where performance measurement (not assumption) shows a bottleneck.

**Never pre-emptively denormalize.** Normalize first, measure, then denormalize
only where evidence demands it.

---

## Chapter 4 Summary

| Normal Form | What It Eliminates | Real Benefit |
|-------------|-------------------|-------------|
| **1NF** | Multi-valued cells, repeating columns | Makes data queryable and indexable |
| **2NF** | Partial dependencies on composite PKs | Eliminates product/customer data duplication in junction tables |
| **3NF** | Transitive dependencies | Eliminates department data in employee tables, zip→city in address tables |
| **BCNF** | Non-superkey determinants | Eliminates subtle redundancy in tables with overlapping candidate keys |
| **Denormalization** | Applied after profiling | Eliminates expensive JOIN operations on hot read paths |

---

# END OF PART I — FOUNDATIONS

---

> **What You Now Know**
> - Exactly what happens inside PostgreSQL from the moment SQL arrives to when results return
> - How shared_buffers, WAL, and background processes keep data safe and fast
> - Why ACID properties matter and precisely how PostgreSQL implements each one
> - The physical layout of pages, tuples, NULL, and TOAST on disk
> - How to design a schema that eliminates all three data anomalies
>
> **What Comes Next**
> Part II covers Indexing — how PostgreSQL makes reads fast.
> You will understand B-Trees at the byte level, when each index type is the right tool,
> and the hidden costs of having too many indexes.

---

*Continue with Chapter 5: B-Tree Indexes →*# PART II — INDEXING: MAKING READS FAST

> A table with 10 million rows and no index forces PostgreSQL to read every single row
> to answer a query. With the right index, the same query reads 3–4 pages.
> This part explains exactly how that is possible — and when it is not.

---

# CHAPTER 5 — B-Tree Indexes

---

## 5.1 WHY B-Trees

Sequential scan on 10 million rows:
```
10,000,000 rows × 100 bytes = ~1 GB of data
1 GB ÷ 8KB per page = ~125,000 pages to read
125,000 pages × 0.1ms per page = 12.5 seconds
```

B-Tree index lookup for one row:
```
Tree height for 10M rows ≈ 4 levels
4 page reads × 0.1ms = 0.4ms
```

That is a 30,000× difference. The B-Tree is why databases are usable at scale.

---

## 5.2 WHAT a B-Tree Is

B-Tree stands for **Balanced Tree**. "Balanced" means every path from root to any
leaf node has the same length. This guarantees that no matter which value you
look up, the cost is always the same: O(log n).

A PostgreSQL B-Tree has three types of nodes:

```
                        ┌──────────────────┐
                        │   ROOT NODE      │  ← always 1
                        │  [300 | 700]     │
                        └──┬──────┬────────┘
                           │      │
              ┌────────────┘      └────────────┐
              ▼                                ▼
    ┌──────────────────┐          ┌──────────────────┐
    │  INTERNAL NODE   │          │  INTERNAL NODE   │  ← 0 or more levels
    │ [100|200|300]    │          │ [500|600|700]    │
    └─┬────┬────┬──────┘          └──┬────┬────┬─────┘
      │    │    │                    │    │    │
      ▼    ▼    ▼                    ▼    ▼    ▼
    [L1] [L2] [L3]                [L4] [L5] [L6]
    LEAF NODES  ←─────────────────────────────────→  doubly linked list
```

Each **leaf node** contains:
- The indexed key value (e.g., price = 99.99)
- A pointer to the heap tuple: `(page_number, slot_number)` = ctid

Each **internal node** contains:
- Separator keys
- Pointers to child nodes

The **root** is the entry point. PostgreSQL caches it aggressively in shared_buffers.

---

## 5.3 HOW B-Tree Lookup Works (Point Query)

```sql
CREATE INDEX ix_products_sku ON products(sku);
SELECT * FROM products WHERE sku = 'WID-001';
```

Step-by-step execution:

```
1. Read ROOT node (almost always in shared_buffers = ~0ms)
   Root contains: ['GAD' | 'TH']
   'WID' > 'TH' → go to right child

2. Read INTERNAL node (likely in cache)
   Contains: ['VID' | 'WIG']
   'WID' > 'VID', 'WID' < 'WIG' → go to middle child

3. Read LEAF node (may require disk read = ~0.1ms)
   Contains: ['WGA-001', ctid=(42,3)] ['WID-001', ctid=(43,7)] ['WID-002', ctid=(43,8)]
   Found: 'WID-001' → heap location = page 43, slot 7

4. Read HEAP page 43 (may require disk read = ~0.1ms)
   Fetch slot 7 → full row returned

Total: 4 page reads, ~0.4ms
```

---

## 5.4 HOW B-Tree Range Scan Works

```sql
SELECT * FROM products WHERE price BETWEEN 100 AND 500;
```

This is where B-Trees shine over hash indexes:

```
1. Find the leaf node containing price=100 (same as point lookup)
2. Scan RIGHT through the linked list of leaf nodes
   → Leaf [90,95,100,110,...] → Leaf [150,200,250,...] → Leaf [400,450,500,...]
3. Stop when price > 500
4. For each key found, fetch the corresponding heap page

No need to go back to the root!
The linked list makes range scans sequential — fast.
```

---

## 5.5 HOW Page Splits Work (and Why They Matter)

When a leaf node is full and a new key must be inserted:

```
BEFORE INSERT of price=155:
  Leaf: [100, 120, 140, 160, 180]  ← FULL (simplified, real nodes hold hundreds of keys)

INSERT 155:
  Node is full → SPLIT:
  Left child:  [100, 120, 140]
  Right child: [155, 160, 180]
  Push separator key 155 UP to parent internal node

  If parent is also full → split parent → push UP again
  In worst case, split cascades to root → root splits → tree height increases by 1
```

**Why this matters:**
- Splits cause multiple page writes → WAL records → slower inserts
- After many splits, pages may be half-empty → **index bloat**
- `FILLFACTOR` controls how full pages are packed:

```sql
-- Default: pages packed to 100% → many splits on insert-heavy tables
CREATE INDEX ix_products_price ON products(price);

-- Custom: leave 20% free space per page → fewer future splits
CREATE INDEX ix_products_price ON products(price) WITH (fillfactor = 80);
-- Use when table has frequent INSERTs/UPDATEs on the indexed column
```

---

## 5.6 Index Scan vs Bitmap Scan vs Covering Index

### Index Scan
```
For each key in the index:
  → Immediately fetch the heap page
  → Returns rows one at a time (good for small result sets, ORDER BY)

Problem with large result sets:
  → 1000 matching rows scattered across 1000 different heap pages
  → 1000 random heap reads = 1000 × 0.1ms = 100ms
  → A seq scan of the whole table might be faster!
```

### Bitmap Index Scan (the middle ground)
```
Phase 1 — Build a bitmap:
  Scan the index, collect ALL matching page numbers
  Bitmap: [page 42: YES, page 43: YES, page 100: YES, ...]

Phase 2 — Heap scan:
  Read pages in ORDER (sequential I/O — much faster than random!)
  Check exact tuples within each page

Benefit: converts random I/O to sequential I/O for medium result sets
```

```sql
EXPLAIN SELECT * FROM products WHERE price BETWEEN 50 AND 200;

-- If few rows match:
--   Index Scan using ix_products_price

-- If moderate rows match:
--   Bitmap Heap Scan on products
--     Recheck Cond: (price >= 50 AND price <= 200)
--     -> Bitmap Index Scan on ix_products_price

-- If most rows match:
--   Seq Scan on products
--     Filter: (price >= 50 AND price <= 200)
```

### Covering Index (Index-Only Scan)

When the index contains ALL columns your query needs, heap is never touched:

```sql
-- Normal index: returns pointer, then fetches heap
CREATE INDEX ix_sku ON products(sku);
SELECT name FROM products WHERE sku = 'WID-001';
-- Index: finds ctid for WID-001
-- Heap: fetches page to get "name" column

-- Covering index: contains both the key AND extra columns
CREATE INDEX ix_sku_covering ON products(sku) INCLUDE (name, price);
SELECT name, price FROM products WHERE sku = 'WID-001';
-- Index: finds WID-001, also has name and price right there
-- Heap: NOT FETCHED AT ALL → "Index Only Scan"
```

```sql
-- EXPLAIN confirms:
EXPLAIN SELECT name, price FROM products WHERE sku = 'WID-001';
--  Index Only Scan using ix_sku_covering on products
--    Index Cond: (sku = 'WID-001')
--    Heap Fetches: 0   ← zero heap reads!
```

**Heap Fetches > 0** means some pages are not marked all-visible in the visibility map.
Run `VACUUM products` to restore the all-visible bits, dropping Heap Fetches to 0.

---

## 5.7 Multi-Column (Composite) Indexes

```sql
CREATE INDEX ix_orders_customer_date ON orders(customer_id, created_at DESC);
```

This index is like a phone book sorted first by customer_id, then by created_at within each customer.

```
Index entries:
  (customer_id=1, created_at='2024-03-15') → ctid
  (customer_id=1, created_at='2024-02-10') → ctid
  (customer_id=1, created_at='2024-01-05') → ctid
  (customer_id=2, created_at='2024-03-20') → ctid
  (customer_id=2, created_at='2024-01-01') → ctid
  ...
```

**Queries this index CAN answer:**
```sql
WHERE customer_id = 5                             -- ✅ prefix match
WHERE customer_id = 5 AND created_at > '2024-01-01'  -- ✅ full match
WHERE customer_id = 5 ORDER BY created_at DESC    -- ✅ index already sorted this way
```

**Queries this index CANNOT answer efficiently:**
```sql
WHERE created_at > '2024-01-01'   -- ❌ leading column missing → seq scan
ORDER BY created_at DESC          -- ❌ without customer_id filter → seq scan
```

**The prefix rule:** A composite index on (A, B, C) can be used for:
- Queries filtering on A
- Queries filtering on A and B
- Queries filtering on A, B, and C
- But NOT for queries filtering only on B, or only on C, or B and C without A

---

## Chapter 5 Summary

| Concept | Key Insight |
|---------|-------------|
| B-Tree structure | Balanced tree: O(log n) lookup for any value, regardless of table size |
| Range scans | Leaf nodes are a linked list → range scans are sequential, fast |
| Page splits | INSERT into full page triggers split → causes write amplification; use fillfactor |
| Index scan | Best for small result sets and ORDER BY |
| Bitmap scan | Best for medium result sets — converts random to sequential I/O |
| Index-Only scan | Fastest possible — heap never touched; requires covering index + VACUUM |
| Composite index | Column ORDER matters: must match query's leading columns |

---

# CHAPTER 6 — Index Types

---

## 6.1 WHY Multiple Index Types Exist

A B-Tree is excellent for ordered, comparable data (numbers, text, dates).
But what about:
- "Find all products where `tags` contains 'electronics'"? (array containment)
- "Find documents where the text contains the word 'postgresql'"? (full-text search)
- "Find all orders within the last 30 days"? (range overlap)
- "Find the nearest restaurant to GPS coordinates (12.97, 77.59)"? (geometric proximity)

B-Trees cannot answer these efficiently because they rely on total ordering
(less than / equal to / greater than). Different data structures are needed.

---

## 6.2 Hash Index

### What It Is
A hash index maps each key to a hash bucket. Lookup is O(1) — constant time,
regardless of table size. Faster than B-Tree for exact equality.

### When to Use It
```sql
CREATE INDEX ix_sessions_token ON sessions USING HASH (token);

-- Perfect for:
SELECT * FROM sessions WHERE token = 'abc123xyz';  -- exact equality → O(1)

-- Cannot do:
SELECT * FROM sessions WHERE token > 'abc';  -- range → not possible with hash
SELECT * FROM sessions ORDER BY token;       -- sort → not possible with hash
```

### Reality Check
In practice, B-Tree is usually chosen over Hash because:
- B-Tree handles equality AND range AND ORDER BY
- Hash index is only marginally faster for equality
- Hash indexes were not WAL-logged before PostgreSQL 10 (crash unsafe)

Use Hash when: the column is very long (long strings), equality-only queries,
and you want the slightly smaller index size.

---

## 6.3 GIN — Generalized Inverted Index

### What It Is
GIN is designed for values that contain **multiple sub-values** —
arrays, JSONB documents, and tsvector (full-text search tokens).

An inverted index maps each sub-value → list of rows containing it.
Like a book's index: word → page numbers where it appears.

```
GIN index on tags column:

tags value in row 1: ['electronics', 'wireless', 'keyboard']
tags value in row 2: ['electronics', 'mouse']
tags value in row 3: ['furniture', 'desk']

GIN stores:
  'electronics' → [row1, row2]
  'wireless'    → [row1]
  'keyboard'    → [row1]
  'mouse'       → [row2]
  'furniture'   → [row3]
  'desk'        → [row3]

Query: "find rows where tags @> ARRAY['electronics']"
  → Look up 'electronics' in GIN → [row1, row2] → fetch those rows
  → No need to scan all rows!
```

### GIN for Arrays

```sql
CREATE INDEX ix_products_tags ON products USING GIN (tags);

-- Contains all specified tags:
SELECT * FROM products WHERE tags @> ARRAY['electronics', 'wireless'];

-- Overlaps with any of the specified tags:
SELECT * FROM products WHERE tags && ARRAY['keyboard', 'mouse'];

-- Contains a specific element:
SELECT * FROM products WHERE 'electronics' = ANY(tags);
```

### GIN for JSONB

```sql
CREATE INDEX ix_orders_metadata ON orders USING GIN (metadata);

-- Containment: find orders where metadata contains this sub-document
SELECT * FROM orders WHERE metadata @> '{"status": "shipped", "carrier": "FedEx"}';

-- Key existence:
SELECT * FROM orders WHERE metadata ? 'tracking_number';

-- Any of these keys:
SELECT * FROM orders WHERE metadata ?| ARRAY['tracking_number', 'delivery_date'];
```

### GIN for Full-Text Search

```sql
-- Store pre-computed tsvector for efficiency
ALTER TABLE articles ADD COLUMN search_vector tsvector;
UPDATE articles SET search_vector = to_tsvector('english', title || ' ' || content);
CREATE INDEX ix_articles_fts ON articles USING GIN (search_vector);

-- Or index the expression directly:
CREATE INDEX ix_articles_fts ON articles
  USING GIN (to_tsvector('english', title || ' ' || content));

-- Full-text search:
SELECT title FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql & indexing');

-- LIKE '%keyword%' (requires pg_trgm extension):
CREATE EXTENSION pg_trgm;
CREATE INDEX ix_products_name_trgm ON products USING GIN (name gin_trgm_ops);
SELECT * FROM products WHERE name ILIKE '%widget%';
```

### GIN Trade-offs

| Property | Detail |
|----------|--------|
| Build time | Slow — must index all sub-values |
| Update time | Slow — inserting one row may add many GIN entries (one per tag) |
| Lookup time | Fast — O(1) per sub-value |
| Size | Large — the inverted list can be huge |
| Best for | Read-heavy, infrequently updated, array/JSON/text columns |

To reduce update cost, GIN uses a **pending list** — new entries are collected
in a small list and merged into the main index later. This is controlled by
`gin_pending_list_limit` (default 4MB).

---

## 6.4 GiST — Generalized Search Tree

### What It Is
GiST is a **framework** for building balanced tree indexes for any data type
that has a notion of "containment" or "proximity" — not just ordering.

PostgreSQL ships GiST implementations for:
- Geometric types (point, polygon, circle, box)
- Range types (daterange, tsrange, int4range)
- Full-text search (tsvector, as an alternative to GIN)
- Network addresses (inet, cidr)

### GiST for Range Types

```sql
-- Find hotel reservations that overlap with a date range
CREATE TABLE reservations (
    id          SERIAL PRIMARY KEY,
    room_id     INT NOT NULL,
    guest_name  VARCHAR(100),
    stay        DATERANGE NOT NULL
);

CREATE INDEX ix_reservations_stay ON reservations USING GIST (stay);

-- Find all reservations overlapping with Jan 10-20:
SELECT * FROM reservations
WHERE stay && '[2024-01-10, 2024-01-20]'::daterange;

-- Is a date contained in any reservation?
SELECT * FROM reservations
WHERE stay @> '2024-01-15'::date;

-- Find rooms available for a given period (not overlapping):
SELECT DISTINCT room_id FROM reservations
WHERE NOT stay && '[2024-01-10, 2024-01-20]'::daterange;
```

### GiST for Geometric Types

```sql
-- Find restaurants within 5km of a location
CREATE TABLE restaurants (
    id       SERIAL PRIMARY KEY,
    name     VARCHAR(100),
    location POINT
);

CREATE INDEX ix_restaurants_location ON restaurants USING GIST (location);

-- Nearest restaurants (KNN query):
SELECT name, location <-> point(77.5946, 12.9716) AS distance
FROM restaurants
ORDER BY location <-> point(77.5946, 12.9716)
LIMIT 10;
-- <-> = distance operator; GiST enables this without a full table scan
```

---

## 6.5 BRIN — Block Range Index

### What It Is
BRIN stands for **Block Range Index**. It is a tiny index that stores
the **minimum and maximum values** for each range of heap pages (default: 128 pages per range).

```
Table: events (10 million rows, ~80,000 pages)

BRIN divides into 128-page ranges:
  Range 0 (pages 0-127):   min_created_at='2020-01-01', max='2020-01-15'
  Range 1 (pages 128-255): min_created_at='2020-01-15', max='2020-01-30'
  Range 2 (pages 256-383): min_created_at='2020-01-30', max='2020-02-15'
  ...

Query: WHERE created_at > '2024-01-01'
  → Check each range: does [min,max] overlap with [2024-01-01, ∞]?
  → Ranges 0..N from 2020 → clearly no overlap → SKIP those 80,000 pages
  → Only read the recent ranges
```

### When BRIN Works vs When It Fails

BRIN requires that the indexed column is **physically correlated** with the
heap's physical order. This is only true for:

```
✅ Naturally ordered tables:
  - Event/log tables (rows appended in timestamp order)
  - Auto-increment IDs (always increasing, rows inserted in order)
  - Sensor readings (data arrives chronologically)

❌ Tables with scattered values:
  - Customer email (random order on disk)
  - Product price (rows inserted in any order)
  - Any column that is updated frequently
```

```sql
-- Check correlation (1.0 = perfectly ordered, 0.0 = random)
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'events' AND attname = 'created_at';
-- correlation = 0.99 → BRIN is excellent
-- correlation = 0.02 → BRIN is useless

-- Create BRIN index (tiny — a few KB even for billion-row tables):
CREATE INDEX ix_events_created_brin ON events USING BRIN (created_at);
-- Default pages_per_range = 128; lower = more precise but larger index
CREATE INDEX ix_events_created_brin ON events USING BRIN (created_at)
  WITH (pages_per_range = 32);
```

### BRIN vs B-Tree vs Partition for Time-Series

| Strategy | Index size | Query speed | Insert speed |
|----------|-----------|------------|-------------|
| No index | 0 | Seq scan (slow) | Fast |
| BRIN | Tiny (KB) | Good for ordered data | Fast |
| B-Tree | Large (GB for billions) | Excellent | Moderate |
| Table partitioning | N/A | Excellent | Fast |

For time-series data with billions of rows: use **BRIN + partitioning** together.

---

## Chapter 6 Summary

| Index Type | Best For | Operators Supported | Key Trade-off |
|-----------|---------|-------------------|--------------|
| **B-Tree** | General purpose, ordered data | =, <, >, BETWEEN, LIKE 'x%' | Balanced performance |
| **Hash** | Equality only, long keys | = only | Cannot do range/sort |
| **GIN** | Arrays, JSONB, full-text | @>, &&, @@, ? | Slow updates, large size |
| **GiST** | Geometric, ranges, proximity | &&, @>, <-> | Flexible but complex |
| **BRIN** | Huge ordered tables (logs, events) | =, <, >, BETWEEN | Only works with physical correlation |

---

# CHAPTER 7 — Index Strategy

---

## 7.1 WHY Strategy Matters

Creating indexes without strategy causes:
- Tables with 15 indexes: every INSERT updates 15 data structures → 10× slower writes
- Unused indexes wasting disk space and slowing down every write
- Missing indexes causing sequential scans on million-row tables
- Wrong composite index column order: index exists but is never used

Good index strategy means: **exactly the right indexes, no more, no less**.

---

## 7.2 Partial Indexes — Index Only What You Query

A partial index covers only a subset of rows. It is smaller, faster to scan,
and update costs affect only the indexed subset.

```sql
-- Full index: indexes ALL orders (millions)
CREATE INDEX ix_orders_status ON orders(status);

-- Most queries only care about 'pending' orders (0.1% of rows)
-- Partial index: indexes ONLY pending orders (thousands)
CREATE INDEX ix_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- This query uses the partial index (condition matches the WHERE clause):
SELECT * FROM orders
WHERE status = 'pending'
ORDER BY created_at;

-- Index size comparison:
-- Full index on status: 500MB for 10M orders
-- Partial index (pending only): 5MB (10,000 pending orders)
-- 100× smaller → fits entirely in shared_buffers → much faster

-- Other useful partial indexes:
-- Active users (not deleted):
CREATE INDEX ix_users_active ON users(email)
WHERE deleted_at IS NULL;

-- Unprocessed jobs:
CREATE INDEX ix_jobs_unprocessed ON jobs(priority DESC, created_at)
WHERE processed_at IS NULL;

-- High-value orders:
CREATE INDEX ix_orders_high_value ON orders(customer_id)
WHERE total_amount > 10000;
```

**Critical rule:** The partial index is only used when the query's WHERE clause
logically implies the index's WHERE clause. PostgreSQL must be able to prove
that any row the index points to satisfies the query's conditions.

---

## 7.3 Expression Indexes — Index Computed Values

```sql
-- Problem: case-insensitive search ignores the index
CREATE INDEX ix_customers_email ON customers(email);

SELECT * FROM customers WHERE email = 'Rahul@Gmail.com';  -- uses index ✅
SELECT * FROM customers WHERE LOWER(email) = 'rahul@gmail.com';  -- seq scan! ❌
-- Wrapping email in LOWER() prevents index use

-- Solution: index the expression itself
CREATE INDEX ix_customers_email_lower ON customers(LOWER(email));

SELECT * FROM customers WHERE LOWER(email) = 'rahul@gmail.com';  -- uses index ✅

-- More examples:
-- Index on extracted date (for date-only queries on timestamp column):
CREATE INDEX ix_orders_date ON orders(DATE(created_at));
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-15';  -- uses index ✅

-- Index on JSON field:
CREATE INDEX ix_events_type ON events((payload->>'event_type'));
SELECT * FROM events WHERE payload->>'event_type' = 'purchase';  -- uses index ✅

-- Index on computed hash (for very long text, e.g. URLs):
CREATE INDEX ix_pages_url_hash ON pages(MD5(url));
SELECT * FROM pages WHERE MD5(url) = MD5('https://very-long-url...');
```

---

## 7.4 Composite Index Column Order — The Most Common Mistake

```sql
-- Query 1: filter by customer, then sort by date
SELECT * FROM orders
WHERE customer_id = 5
ORDER BY created_at DESC
LIMIT 10;

-- Query 2: filter by customer AND date range
SELECT * FROM orders
WHERE customer_id = 5
  AND created_at > '2024-01-01';

-- CORRECT index for both queries:
CREATE INDEX ix_orders_customer_date ON orders(customer_id, created_at DESC);
-- The planner can:
--   → Seek to customer_id=5 block
--   → Scan forward in created_at order
--   → Stop after LIMIT 10

-- WRONG (reversed):
CREATE INDEX ix_orders_date_customer ON orders(created_at DESC, customer_id);
-- For Query 1: cannot efficiently use this for customer_id=5 filter
-- → Planner will likely choose seq scan instead

-- The mental model:
-- Composite index (A, B) = sorted first by A, then by B within each A
-- Use this when queries filter on A (alone) or A+B together
-- Useless if query filters only on B (no A)
```

**Choosing column order:**
1. Put equality columns first: `WHERE status = 'active' AND customer_id = 5` → `(status, customer_id)`
2. Put range columns last: `WHERE customer_id = 5 AND created_at > '2024-01-01'` → `(customer_id, created_at)`
3. Put high-cardinality columns first: more selective columns first eliminate more rows faster

---

## 7.5 Index Bloat and Maintenance

Over time, B-Tree indexes accumulate dead entries from UPDATEs and DELETEs.
Dead entries waste space and slow down scans.

```sql
-- Check index bloat
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS times_used,
    idx_tup_read AS tuples_returned
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find indexes that were never used (candidates for removal):
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%_pkey'  -- don't remove primary keys
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild a bloated index without locking the table:
REINDEX INDEX CONCURRENTLY ix_orders_customer_date;

-- Or rebuild ALL indexes on a table:
REINDEX TABLE CONCURRENTLY orders;
```

**When to run REINDEX:**
- After a bulk UPDATE or DELETE of >50% of rows
- When `pg_relation_size(indexrelid)` is much larger than expected
- When EXPLAIN shows index scans are slower than expected

---

## Chapter 7 Summary

| Strategy | When to Apply | Benefit |
|----------|-------------|---------|
| Partial index | Column with low cardinality WHERE clause | Tiny index, fast scan, low write cost |
| Expression index | Queries wrap column in function | Makes previously un-indexable queries use index |
| Composite ordering | Queries filter on multiple columns | Single index answers multiple query patterns |
| INCLUDE columns | Queries need extra columns beyond the key | Enables Index-Only Scan |
| REINDEX CONCURRENTLY | After heavy DELETE/UPDATE | Removes bloat without downtime |

---

# CHAPTER 8 — Index Pitfalls

---

## 8.1 Over-Indexing — The Write Tax

Every index is a liability on write operations.

```sql
-- Table with 8 indexes:
CREATE TABLE orders (id BIGSERIAL, customer_id INT, status TEXT, ...);
CREATE INDEX ix_1 ON orders(customer_id);
CREATE INDEX ix_2 ON orders(status);
CREATE INDEX ix_3 ON orders(created_at);
CREATE INDEX ix_4 ON orders(total_amount);
CREATE INDEX ix_5 ON orders(customer_id, created_at);
CREATE INDEX ix_6 ON orders(status, created_at);
CREATE INDEX ix_7 ON orders(customer_id, status);
CREATE UNIQUE INDEX ix_8 ON orders(reference_number);

-- Every INSERT into orders:
1. Write heap row
2. Update ix_1: B-Tree insert
3. Update ix_2: B-Tree insert
4. Update ix_3: B-Tree insert
5. Update ix_4: B-Tree insert
6. Update ix_5: B-Tree insert
7. Update ix_6: B-Tree insert
8. Update ix_7: B-Tree insert
9. Update ix_8: B-Tree insert + uniqueness check

-- Result: 9× the I/O of a single-page heap write
-- Plus WAL records for all 9 operations
-- Insert throughput: 10,000/sec with 0 indexes → ~2,000/sec with 8 indexes
```

**Rule:** Only create indexes that you can prove — through EXPLAIN ANALYZE
and query frequency analysis — are actually used and make a measurable difference.

---

## 8.2 Implicit Cast — The Silent Index Killer

```sql
-- Table:
CREATE TABLE orders (customer_id INT, ...);
CREATE INDEX ix_orders_customer ON orders(customer_id);

-- Query from application (passing string instead of integer):
SELECT * FROM orders WHERE customer_id = '12345';  -- string, not integer!

-- PostgreSQL must cast '12345'::text to integer to compare
-- Does it use the index? Check EXPLAIN:
EXPLAIN SELECT * FROM orders WHERE customer_id = '12345';
--  Index Scan using ix_orders_customer on orders
--    Index Cond: (customer_id = '12345'::integer)  ← implicit cast applied ✅
-- PostgreSQL IS smart enough to cast the literal here

-- BUT: if you pass the wrong type via a bind parameter:
-- Python: cursor.execute("SELECT * FROM orders WHERE customer_id = %s", ("12345",))
-- This sends '12345' as text parameter
-- PostgreSQL: compares INT column to TEXT parameter → cast needed → may avoid index!

-- ALWAYS pass correctly typed parameters from your application:
-- Python: cursor.execute("SELECT * FROM orders WHERE customer_id = %s", (12345,))
--                                                                          ↑ integer!

-- The GUARANTEED index-killer: casting the column (not the value)
SELECT * FROM orders WHERE CAST(customer_id AS TEXT) = '12345';  -- seq scan!
-- Rule: NEVER wrap the indexed column in a function or cast
```

---

## 8.3 LIKE Patterns — When Indexes Help and When They Don't

```sql
-- Index on name:
CREATE INDEX ix_products_name ON products(name);

-- Prefix search (anchored LEFT): uses index ✅
SELECT * FROM products WHERE name LIKE 'Widget%';
-- B-Tree: seek to 'Widget', scan forward until prefix no longer matches

-- Contains search: cannot use B-Tree index ❌
SELECT * FROM products WHERE name LIKE '%Widget%';
-- B-Tree has no way to know which entries contain 'Widget' in the middle

-- Suffix search: cannot use B-Tree index ❌
SELECT * FROM products WHERE name LIKE '%Widget';

-- SOLUTION for contains/suffix: pg_trgm extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX ix_products_name_trgm ON products USING GIN (name gin_trgm_ops);

-- Now ALL of these use the trigram index:
SELECT * FROM products WHERE name LIKE '%Widget%';   -- ✅
SELECT * FROM products WHERE name LIKE '%Widget';    -- ✅
SELECT * FROM products WHERE name ILIKE '%widget%';  -- ✅ (case-insensitive)
SELECT * FROM products WHERE name ~ 'Wid.*t';        -- ✅ (regex)

-- How trigrams work:
-- 'Widget' is broken into 3-character trigrams: 'Wid', 'idg', 'dge', 'get'
-- GIN index stores: 'Wid'→[row1,row5], 'idg'→[row1], 'dge'→[row1], 'get'→[row1,row8]
-- Query '%Widget%': find rows containing ALL trigrams of 'Widget' → intersect lists
```

---

## 8.4 NULL Handling in Indexes

Unlike some databases (Oracle, SQL Server), PostgreSQL B-Tree indexes
**include NULL values**. This means IS NULL queries CAN use a B-Tree index:

```sql
CREATE INDEX ix_orders_delivered ON orders(delivered_at);

-- These use the index:
SELECT * FROM orders WHERE delivered_at IS NULL;      -- ✅
SELECT * FROM orders WHERE delivered_at IS NOT NULL;  -- ✅
SELECT * FROM orders WHERE delivered_at = '2024-01-15'; -- ✅

-- BUT: if most rows have NULL, the index is not selective
-- "Find all undelivered orders" → 99% of rows → index not helpful
-- PostgreSQL will choose seq scan for low selectivity

-- Better: partial index
CREATE INDEX ix_orders_undelivered ON orders(created_at)
WHERE delivered_at IS NULL;
-- Small index, only indexes undelivered orders
-- Very selective, very fast
```

---

## Chapter 8 Summary

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Over-indexing | Slow writes, high disk usage | Remove unused indexes (check `idx_scan = 0`) |
| Wrong type in query | Index exists but seq scan chosen | Pass correctly typed bind parameters |
| Function on column | `WHERE LOWER(email) = ...` does seq scan | Create expression index on `LOWER(email)` |
| LIKE `'%x%'` | Seq scan despite index on column | Create GIN trigram index |
| Low selectivity | Index exists but planner ignores it | Create partial index for specific subset |
| Index bloat | Index larger than expected, slower scans | `REINDEX INDEX CONCURRENTLY` |

# PART III — JOINS: UNDER THE HOOD

> Joins are the heart of relational databases. Understanding which physical
> algorithm PostgreSQL uses — and why — is the difference between a query
> that runs in 50ms and one that runs in 50 seconds.

---

# CHAPTER 9 — Join Algorithms

---

## 9.1 WHY Three Algorithms Exist

There is no single best way to join two tables. The best algorithm depends on:
- Size of each table
- Whether the join column is indexed
- Whether the inputs are already sorted
- How much memory is available

PostgreSQL implements three physical join algorithms and picks the cheapest
based on cost estimates. You need to understand all three to know when
the planner is making a good choice and when it is wrong.

---

## 9.2 Nested Loop Join

### What It Is

The simplest join algorithm. For every row in the **outer** table,
scan the **inner** table looking for matching rows.

```
Pseudocode:
  for each row R1 in OUTER:
      for each row R2 in INNER:
          if R1.key == R2.key:
              emit (R1, R2)
```

### How PostgreSQL Executes It

```
EXPLAIN SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending';

Nested Loop
  →  Seq Scan on orders (filter: status='pending')  ← outer
  →  Index Scan on customers using pkey             ← inner (one lookup per order)
```

For each pending order, PostgreSQL does ONE index lookup into customers by primary key.
If there are 500 pending orders → 500 index lookups into customers.
Each lookup: 3–4 page reads → 500 × 4 = 2,000 page reads.

### Cost: O(n × log m) with index on inner

Without an index on the inner table:
```
Cost: O(n × m)   — catastrophic for large tables
500 orders × 100,000 customers = 50,000,000 row comparisons
```

With an index on the inner join column:
```
Cost: O(n × log m)  — excellent for small outer results
500 orders × log(100,000) = 500 × 17 = 8,500 comparisons
```

### When PostgreSQL Chooses Nested Loop

- Outer table result is **small** (after filters)
- Inner table has an **index** on the join column
- The join is `=` on a primary key or unique key

```sql
-- Force nested loop to see its plan (for testing):
SET enable_hashjoin = off;
SET enable_mergejoin = off;
EXPLAIN ANALYZE SELECT ...;
RESET ALL;
```

---

## 9.3 Hash Join

### What It Is

Hash Join works in two phases:
1. **Build phase:** Read the smaller table, build a hash table in memory keyed on join column
2. **Probe phase:** Read the larger table, look up each row in the hash table

```
Phase 1 — Build hash table from customers (smaller table):
  hash_table = {}
  for each customer row:
      bucket = hash(customer.id) % num_buckets
      hash_table[bucket].append(customer_row)

  Result:
  { bucket_0: [cust_id=10, cust_id=500, ...],
    bucket_1: [cust_id=7, cust_id=203, ...],
    ... }

Phase 2 — Probe with orders (larger table):
  for each order row:
      bucket = hash(order.customer_id) % num_buckets
      for each row in hash_table[bucket]:
          if row.id == order.customer_id:
              emit (order, row)
```

### Why Hash Join is Fast

- Build phase: **one full scan** of the smaller table
- Probe phase: **one full scan** of the larger table
- No random I/O — everything is sequential!
- Total: O(n + m) — linear in total data size

```sql
EXPLAIN SELECT o.id, c.name, o.total_amount
FROM orders o
JOIN customers c ON c.id = o.customer_id;

Hash Join  (cost=2500..15000 rows=1000000 width=52)
  Hash Cond: (o.customer_id = c.id)
  →  Seq Scan on orders          ← probe (larger table)
  →  Hash                        ← hash table built from customers
       →  Seq Scan on customers  ← build (smaller table)
```

### When Hash Join Spills to Disk

The hash table must fit in `work_mem`. If it does not:

```
Hash (cost=...) (actual rows=500000 loops=1)
  Batches: 4      ← hash table did not fit in memory
  Memory Usage: 4096kB
  Disk Usage: 24576kB  ← spilled to disk!
```

When batches > 1, the join is done in multiple passes, each writing
and reading from disk. Performance degrades significantly.

```sql
-- Fix: increase work_mem for this session
SET work_mem = '256MB';
EXPLAIN ANALYZE SELECT ...;
-- Batches: 1  ← fits in memory now
```

### When PostgreSQL Chooses Hash Join

- Both tables are **large**
- No useful index on the join column (or index wouldn't help for full table join)
- Enough `work_mem` to hold the smaller table

---

## 9.4 Merge Join

### What It Is

Merge Join requires both inputs to be **sorted** on the join column.
Then it scans both sorted inputs simultaneously, like merging two sorted lists.

```
Sorted customers by id: [1, 2, 3, 4, 5, 6, 7, ...]
Sorted orders by customer_id: [1, 1, 2, 3, 3, 3, 5, 6, ...]

Two pointers, both moving forward:
  customer=1, order=1 → match! emit
  customer=1, order=1 → match! emit (second order for customer 1)
  customer=1, advance orders → order=2, no more customer=1 orders
  customer=2, order=2 → match!
  customer=3, order=3 → match! ...
```

### The Sort Cost

Inputs are rarely pre-sorted. Merge Join usually requires sorting first:
```
Cost = sort(outer) + sort(inner) + merge
     = O(n log n) + O(m log m) + O(n + m)
```

However, if an index already provides a sorted scan, sorting is free:
```
Merge Join
  → Index Scan on customers using customers_pkey  ← already sorted by id
  → Index Scan on orders using ix_orders_customer ← already sorted by customer_id
  → No sort needed! Very efficient.
```

### When PostgreSQL Chooses Merge Join

- Both sides have **indexes** providing sorted output on the join key
- Both tables are **large** (too large for Hash Join hash table)
- The join column has **high cardinality** (many unique values)

```sql
EXPLAIN SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
ORDER BY c.id;  -- ORDER BY same as join key → merge join likely

Merge Join
  Merge Cond: (c.id = o.customer_id)
  → Index Scan on customers using pkey  (sorted by id)
  → Index Scan on orders using ix_customer (sorted by customer_id)
```

---

## 9.5 Choosing the Right Join Algorithm

```
┌────────────────────────────────────────────────────────────────┐
│                    PLANNER DECISION TREE                       │
│                                                                │
│  Outer table result is small (< ~100 rows)?                   │
│    YES → Inner table has index on join column?                │
│              YES → NESTED LOOP  (fast random lookups)         │
│              NO  → HASH JOIN or MERGE JOIN                    │
│                                                               │
│  Both tables large? Enough work_mem?                          │
│    YES → HASH JOIN (if hash table fits in memory)            │
│                                                               │
│  Both sides already sorted (via index)?                       │
│    YES → MERGE JOIN (free sort, linear scan)                 │
│                                                               │
│  Hash table doesn't fit, no useful indexes?                   │
│    → MERGE JOIN with explicit sort (slowest)                 │
└────────────────────────────────────────────────────────────────┘
```

---

## Chapter 9 Summary

| Algorithm | Cost | Best When | Danger |
|-----------|------|-----------|--------|
| **Nested Loop** | O(n × log m) | Small outer + indexed inner | Catastrophic without index (O(n×m)) |
| **Hash Join** | O(n + m) | Large unsorted tables | Spill to disk if work_mem too small |
| **Merge Join** | O(n + m) after sort | Pre-sorted inputs (indexes) | Sort cost if inputs not already sorted |

---

# CHAPTER 10 — Join Types

---

## 10.1 INNER JOIN

Returns only rows where a match exists in BOTH tables.

```sql
SELECT o.id, o.total_amount, c.full_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id;

-- Only orders that have a matching customer are returned
-- Orders with customer_id pointing to a deleted customer → excluded
-- Customers with no orders → excluded

-- Venn diagram: only the overlapping center
```

**When to use:** When you only care about rows with complete data on both sides.
Most common join type in OLTP applications.

---

## 10.2 LEFT JOIN (LEFT OUTER JOIN)

Returns ALL rows from the left table, and matched rows from the right.
Right side columns are NULL when no match exists.

```sql
-- Find all customers and their order count (including customers with no orders)
SELECT c.id, c.full_name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.full_name;

-- Customers with no orders: order_count = 0 (COUNT(NULL) = 0)

-- Find customers who have NEVER ordered:
SELECT c.id, c.full_name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;  ← o.id is NULL only when no match was found
-- This is called an ANTI JOIN pattern
```

---

## 10.3 RIGHT JOIN

Mirror of LEFT JOIN. All rows from the right table, matched from left.

```sql
-- Same as LEFT JOIN with tables swapped:
SELECT o.id, c.full_name
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.id;
-- Equivalent to:
SELECT o.id, c.full_name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
```

**In practice:** RIGHT JOIN is rarely used. It is always clearer to rewrite
it as a LEFT JOIN by swapping table order.

---

## 10.4 FULL OUTER JOIN

Returns all rows from both tables. NULL-fills when no match on either side.

```sql
-- Find all customers and all orders, showing mismatches on both sides
SELECT c.full_name, o.id AS order_id, o.total_amount
FROM customers c
FULL OUTER JOIN orders o ON o.customer_id = c.id;

-- Customers with no orders:    full_name='Rahul', order_id=NULL, total_amount=NULL
-- Orders with no customer:     full_name=NULL, order_id=99, total_amount=500
-- Matched:                     full_name='Priya', order_id=5, total_amount=200
```

**When to use:** Data reconciliation — finding rows that exist in one system
but not another, or detecting orphaned records.

---

## 10.5 CROSS JOIN (Cartesian Product)

Returns every combination of rows from both tables. No join condition.

```sql
-- Every product combined with every color option:
SELECT p.name, c.color_name
FROM products p
CROSS JOIN colors c;
-- 100 products × 10 colors = 1,000 rows

-- Use cases: generate test data, build all combinations for reports
-- DANGER: CROSS JOIN two million-row tables = a trillion rows → out of memory

-- Implicit cross join (avoid this old syntax):
SELECT * FROM products, colors;  -- same as CROSS JOIN
```

---

## 10.6 LATERAL JOIN

LATERAL allows a subquery in the FROM clause to reference columns from
tables earlier in the FROM clause. It is like a correlated subquery
that can return multiple rows.

```sql
-- For each customer, get their 3 most recent orders
SELECT c.full_name, recent.id, recent.total_amount, recent.created_at
FROM customers c
CROSS JOIN LATERAL (
    SELECT id, total_amount, created_at
    FROM orders
    WHERE customer_id = c.id      ← reference to outer table
    ORDER BY created_at DESC
    LIMIT 3
) recent;

-- Without LATERAL, this is impossible in a single SQL statement
-- With LATERAL: PostgreSQL runs the subquery once per customer row
-- and returns up to 3 rows per execution

-- Use LEFT JOIN LATERAL to include customers with no orders:
SELECT c.full_name, recent.id, recent.total_amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT id, total_amount
    FROM orders
    WHERE customer_id = c.id
    ORDER BY created_at DESC
    LIMIT 3
) recent ON true;  ← ON true = always join (like a CROSS JOIN)
```

**Performance note:** LATERAL is essentially a nested loop join.
The subquery runs once per outer row. This is efficient when:
- The outer result is small
- The subquery uses an index

---

## 10.7 Self Join

A table joining with itself. The table appears twice with different aliases.

```sql
-- Employee hierarchy: find each employee and their manager
CREATE TABLE employees (
    id         INT PRIMARY KEY,
    name       VARCHAR(100),
    manager_id INT REFERENCES employees(id)  -- self-reference!
);

SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
-- LEFT JOIN so we include the CEO (who has no manager → manager=NULL)

-- Find employees who earn more than their manager:
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

---

## Chapter 10 Summary

| Join Type | Returns | Common Use Case |
|-----------|---------|----------------|
| INNER JOIN | Matched rows only | Standard data retrieval |
| LEFT JOIN | All left + matched right | Include rows with no related data |
| FULL OUTER JOIN | All rows from both | Reconciliation, finding orphans |
| CROSS JOIN | All combinations | Generate test data, combination reports |
| LATERAL | Correlated multi-row subquery | Top-N per group |
| Self Join | Same table joined to itself | Hierarchies, comparisons |

---

# CHAPTER 11 — Join Optimization

---

## 11.1 join_collapse_limit — How Far the Planner Searches

When you write a query with 5 tables, the planner tries different join orders
to find the cheapest plan. The number of possible orders is n! (factorial):

```
2 tables:  2! = 2 orderings
3 tables:  3! = 6 orderings
5 tables:  5! = 120 orderings
8 tables:  8! = 40,320 orderings
10 tables: 10! = 3,628,800 orderings  ← planning itself takes seconds!
```

`join_collapse_limit` (default: 8) limits how many tables trigger exhaustive search.
Beyond this limit, the planner uses a greedy algorithm:

```sql
SHOW join_collapse_limit;  -- 8

-- For a query with 12 tables:
-- Tables 1-8: exhaustive search (optimal)
-- Tables 9-12: greedy (may not be optimal)

-- If you have a complex query and the plan is bad:
SET join_collapse_limit = 1;
-- Forces PostgreSQL to use your written JOIN order exactly
-- Use when you know the best join order and the planner is wrong

-- Or increase for more thorough search (slower planning):
SET join_collapse_limit = 20;
```

---

## 11.2 Row Estimate Accuracy — The Most Common Plan Problem

When EXPLAIN shows a large gap between estimated and actual rows,
the planner is working with bad information → bad plan choice.

```sql
EXPLAIN ANALYZE
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'completed' AND o.created_at > '2024-01-01';

-- BAD output:
-- Hash Join (cost=... rows=50 width=...)    ← estimated 50 rows
--           (actual time=... rows=125000 ..)  ← actually 125,000!

-- The planner thought 50 rows would come from orders after filters
-- Based on that small estimate, it chose nested loop
-- But 125,000 rows with nested loop = disaster
```

### Why Estimates Are Wrong

```sql
-- 1. Statistics are stale (most common):
ANALYZE orders;  -- update statistics, run after bulk loads

-- 2. Column correlation: planner treats columns as independent
-- WHERE status='completed' AND created_at > '2024-01-01'
-- Actual: completed orders are mostly recent (correlated!)
-- Planner: treats them as independent → underestimates

-- Fix: extended statistics
CREATE STATISTICS stat_orders_status_date ON status, created_at FROM orders;
ANALYZE orders;
-- Planner now knows status and created_at are correlated

-- 3. Statistics target too low (not enough histogram buckets):
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;  -- default is 100
ANALYZE orders;

-- 4. Highly skewed data:
-- 99% of orders have status='completed', 1% have 'pending'
-- Default 100 histogram buckets may not capture this precisely
-- Increase statistics target for skewed columns
```

---

## 11.3 Improving Join Performance with Indexes

```sql
-- Missing index on join column: Hash Join or Nested Loop without index
-- Adding the right index can force Index Scan on the inner side:

-- Before:
EXPLAIN SELECT * FROM order_items oi JOIN products p ON p.id = oi.product_id;
-- Hash Join (full scan of both tables)

-- After:
CREATE INDEX ix_order_items_product ON order_items(product_id);
EXPLAIN SELECT * FROM order_items oi JOIN products p ON p.id = oi.product_id;
-- May now use Nested Loop + Index Scan if products is small/filtered

-- The join column on BOTH sides should ideally be indexed:
-- products.id   → covered by PRIMARY KEY ✅
-- order_items.product_id → needs explicit index ← this is commonly missing

-- Check for missing FK indexes:
SELECT
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table,
    ccu.column_name AS foreign_column
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage ccu ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
AND NOT EXISTS (
    SELECT 1 FROM pg_index i
    JOIN pg_attribute a ON a.attrelid = i.indrelid AND a.attnum = i.indkey[0]
    WHERE i.indrelid = tc.table_name::regclass
    AND a.attname = kcu.column_name
);
-- Lists FK columns without indexes → add indexes to all of these
```

---

## Chapter 11 Summary

| Optimization | When to Apply | Expected Result |
|-------------|-------------|----------------|
| `join_collapse_limit = 1` | Planner picks bad join order | Use your manually optimized join order |
| `CREATE STATISTICS` | Correlated column filters underestimate rows | Better cardinality estimates → better plan |
| `ANALYZE` after bulk load | Stale statistics causing wrong plan | Updated stats → accurate estimates |
| Index FK columns | Hash joins on large tables | Enables Nested Loop + Index Scan |
| Increase statistics target | Skewed data in WHERE columns | More histogram buckets → precise estimates |

---

# CHAPTER 12 — Subqueries vs Joins vs CTEs

---

## 12.1 Correlated Subqueries — The Hidden N+1

A correlated subquery references columns from the outer query.
PostgreSQL executes it once per row of the outer query.

```sql
-- "Find each customer and their total spend"

-- CORRELATED SUBQUERY: runs once per customer row = N+1!
SELECT
    c.id,
    c.full_name,
    (SELECT SUM(total_amount) FROM orders WHERE customer_id = c.id) AS total_spend
FROM customers c;

-- If there are 50,000 customers:
-- → 50,000 subquery executions
-- → Each scans orders by customer_id (index scan)
-- → Total: ~50,000 index scans

-- EQUIVALENT JOIN: single pass through both tables
SELECT
    c.id,
    c.full_name,
    COALESCE(SUM(o.total_amount), 0) AS total_spend
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.full_name;

-- Join: one hash join, one scan of orders, one scan of customers
-- Much faster for large datasets
```

**Important:** Modern PostgreSQL sometimes rewrites correlated subqueries
as joins automatically. Use EXPLAIN to verify:

```sql
EXPLAIN SELECT c.id, (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) FROM customers c;
-- If plan shows "Hash Join" instead of "SubPlan": planner rewrote it ✅
-- If plan shows "SubPlan ... loops=50000": it's still N+1 ← fix it manually
```

---

## 12.2 EXISTS vs IN — Which is Faster?

```sql
-- Question: "Find customers who have placed at least one order over ₹1000"

-- IN: builds a full list of matching IDs, then checks membership
SELECT * FROM customers
WHERE id IN (
    SELECT customer_id FROM orders WHERE total_amount > 1000
);

-- EXISTS: stops as soon as ONE match is found (short-circuits)
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id
    AND o.total_amount > 1000
);

-- Performance comparison:
-- IN:     builds list of ALL matching customer_ids first, then checks each customer
-- EXISTS: for each customer, checks if ANY matching order exists → stops at first

-- When there are many duplicate customer_ids in the subquery result:
-- IN → large list with duplicates → slower
-- EXISTS → stops at first match → faster

-- PostgreSQL often rewrites IN as a semi-join internally:
EXPLAIN SELECT * FROM customers WHERE id IN (SELECT customer_id FROM orders);
-- Plan shows: Hash Semi Join  ← PostgreSQL converted IN to semi-join ✅

-- NOT IN vs NOT EXISTS (CRITICAL difference with NULLs):
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NULL);
-- If ANY customer_id in orders is NULL → NOT IN returns ZERO rows!
-- NULL IS NOT EQUAL TO anything, including NOT IN checks

-- SAFE alternative: NOT EXISTS (handles NULLs correctly)
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
-- Correctly returns customers with no orders, regardless of NULLs
```

**Rule:** Prefer `EXISTS`/`NOT EXISTS` over `IN`/`NOT IN` for subqueries,
especially when NULL values might be present.

---

## 12.3 CTEs — When They Are Optimization Fences

A CTE (Common Table Expression) defined with `WITH` is sometimes
**materialized** — computed once and stored in a temporary table.
This prevents the planner from pushing WHERE clauses inside the CTE.

```sql
-- PostgreSQL 11 and earlier: ALL CTEs were materialized (always an optimization fence)
-- PostgreSQL 12+: CTEs are inlined by default (planner can optimize through them)
-- Exception: CTEs with side effects (DML), or explicitly MATERIALIZED

-- EXAMPLE — PostgreSQL 12+ behavior:
WITH recent_orders AS (
    SELECT * FROM orders WHERE created_at > '2024-01-01'
)
SELECT * FROM recent_orders WHERE customer_id = 5;

-- PostgreSQL 12+: planner inlines CTE, produces:
--   Seq Scan on orders
--     Filter: (created_at > '2024-01-01' AND customer_id = 5)
--   → Uses index on both conditions

-- Force materialization (old behavior, useful for optimization hints):
WITH recent_orders AS MATERIALIZED (
    SELECT * FROM orders WHERE created_at > '2024-01-01'
)
SELECT * FROM recent_orders WHERE customer_id = 5;
-- → CTE is computed fully first (all orders since 2024-01-01)
-- → Then filtered for customer_id=5
-- → Useful when you want to prevent the planner from changing the plan

-- When CTEs are ALWAYS materialized (even in PG 12+):
-- 1. CTE contains DML (INSERT/UPDATE/DELETE ... RETURNING)
-- 2. CTE is referenced more than once in the outer query
-- 3. Explicitly marked MATERIALIZED
```

---

## 12.4 Semi-Join and Anti-Join (How the Planner Thinks)

These are not SQL keywords but optimizer concepts the planner uses internally.

**Semi-join:** "Does a matching row exist?" — stops at first match, no duplicates

```sql
-- SQL that becomes a semi-join:
SELECT * FROM customers WHERE id IN (SELECT customer_id FROM orders);
SELECT * FROM customers c WHERE EXISTS (SELECT 1 FROM orders WHERE customer_id = c.id);

-- EXPLAIN shows: Hash Semi Join or Nested Loop Semi Join
-- Key property: returns each customer at most once (no duplicates from JOIN)
```

**Anti-join:** "Does NO matching row exist?"

```sql
-- SQL that becomes an anti-join:
SELECT * FROM customers WHERE id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL);
SELECT * FROM customers c WHERE NOT EXISTS (SELECT 1 FROM orders WHERE customer_id = c.id);

-- EXPLAIN shows: Hash Anti Join or Nested Loop Anti Join
```

---

## Chapter 12 Summary

| Technique | When It's Fast | When It's Slow | Fix |
|-----------|---------------|---------------|-----|
| Correlated subquery | Single row outer | Large outer table | Rewrite as JOIN or LATERAL |
| IN subquery | Small subquery result | Large subquery with NULLs | Use EXISTS; PG rewrites anyway |
| NOT IN | No NULLs | Any NULL in subquery | Always use NOT EXISTS |
| CTE (inlined) | Complex queries, PG 12+ | — | Default behavior in PG 12+ |
| CTE (MATERIALIZED) | Referenced multiple times | Single-use CTEs | Use only when needed |

# PART IV — TRANSACTIONS, ISOLATION, LOCKING, DEADLOCKS

---

# CHAPTER 13 — Transactions

---

## 13.1 WHY Transactions Exist

Every real-world business operation involves multiple steps.
Transferring money: debit one account, credit another — two steps.
Placing an order: create order record, create order items, deduct stock — at least three steps.

If any step fails halfway through, the database must be left in a consistent state —
not half-done. Transactions are the mechanism that enforces this.

---

## 13.2 BEGIN, COMMIT, ROLLBACK

```sql
-- Explicit transaction:
BEGIN;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;   -- debit
UPDATE accounts SET balance = balance + 500 WHERE id = 2;   -- credit

-- Both succeeded? Commit permanently:
COMMIT;

-- OR something went wrong? Undo everything:
ROLLBACK;
```

**Implicit transactions:** Every SQL statement that is NOT inside an explicit
BEGIN/COMMIT block is automatically wrapped in its own single-statement transaction.

```sql
-- This single statement is automatically atomic:
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE id = 5;
-- Equivalent to: BEGIN; UPDATE...; COMMIT;
-- If the UPDATE fails (e.g. CHECK constraint), it automatically rolls back
```

**Transaction states:**
```
BEGIN
  │
  ├── All statements execute successfully
  │     → COMMIT: all changes become permanent and visible
  │
  └── Any statement fails (error)
        → Transaction is "aborted" state
        → Every subsequent statement until ROLLBACK returns:
          "ERROR: current transaction is aborted, commands ignored until ROLLBACK"
        → Must issue ROLLBACK before starting new work
```

---

## 13.3 SAVEPOINT — Partial Rollbacks

SAVEPOINTs let you mark intermediate points within a transaction
and roll back to that point without abandoning the entire transaction.

```sql
BEGIN;

INSERT INTO orders (customer_id, total_amount) VALUES (5, 299.99);
-- Order inserted, order_id = 1001

SAVEPOINT before_items;

INSERT INTO order_items (order_id, product_id, quantity, price_at_purchase)
VALUES (1001, 9999, 2, 149.99);
-- ERROR! product_id 9999 doesn't exist (foreign key violation)

-- Without savepoint: entire transaction is aborted, must ROLLBACK completely
-- WITH savepoint: roll back only to the savepoint

ROLLBACK TO SAVEPOINT before_items;
-- ↑ order INSERT is still alive, only the bad order_item is undone

INSERT INTO order_items (order_id, product_id, quantity, price_at_purchase)
VALUES (1001, 1, 2, 149.99);
-- Now with correct product_id=1

SAVEPOINT after_items;

UPDATE products SET stock_quantity = stock_quantity - 2 WHERE id = 1;

COMMIT;
-- All three operations committed: order + order_item + stock update
```

SAVEPOINTs are especially useful in application code where individual
operations may fail but the overall transaction should continue:

```python
# Python example with psycopg2
with conn:
    with conn.cursor() as cur:
        cur.execute("BEGIN")
        cur.execute("INSERT INTO orders ...")
        
        for item in order_items:
            cur.execute("SAVEPOINT item_sp")
            try:
                cur.execute("INSERT INTO order_items ...", item)
                cur.execute("UPDATE products SET stock = stock - %s WHERE id = %s", ...)
            except Exception as e:
                cur.execute("ROLLBACK TO SAVEPOINT item_sp")
                log_error(item, e)
                # Continue with next item
        
        cur.execute("COMMIT")
```

---

## 13.4 Transaction ID (XID) and its Lifecycle

Every transaction is assigned a **32-bit Transaction ID (XID)**:

```sql
-- See your current transaction's XID:
SELECT txid_current();
-- Returns: 12345678

-- XIDs are assigned sequentially. The sequence is:
-- 0, 1, 2, 3, ... 4,294,967,294, wrap back to 3
-- (XIDs 0, 1, 2 are reserved: invalid, bootstrap, frozen)
```

**The XID lifecycle:**

```
XID assigned at BEGIN (or first statement if autocommit)
  ↓
Statements execute, changes tagged with this XID
  ↓
COMMIT:
  → XID recorded as "committed" in pg_xact (transaction status file)
  → Changes become visible to new transactions
  → WAL record written

ROLLBACK:
  → XID recorded as "aborted" in pg_xact
  → Changes invisible to all (MVCC visibility rules exclude aborted XIDs)
  → WAL record written (for recovery)
```

---

## 13.5 XID Wraparound — The Most Critical PostgreSQL Operational Issue

PostgreSQL uses **modular arithmetic** on 32-bit XIDs.
The XID space is treated as a circle of 4.2 billion positions.

```
XID circle (simplified):
         XID 1
        /      \
  XID 4B        XID 2
        \      /
         XID 3B

"Past" = 2 billion XIDs behind current position
"Future" = 2 billion XIDs ahead of current position

A row is VISIBLE if its xmin is in the "past" 2 billion.
```

**The wraparound problem:**

```
Database has processed 4.2 billion transactions.
New transaction XID = 1 (wrapped around).

Old row with xmin = 3,000,000,000:
  From XID 1's perspective:
  Distance = |1 - 3,000,000,000| mod 4,294,967,296 = 1,294,967,296
  This is within the "future" 2 billion range!
  → Row appears to be "in the future" → INVISIBLE
  → Data appears to vanish!
```

**Prevention — Freezing:**

PostgreSQL periodically "freezes" old tuples. A frozen tuple's xmin
is replaced with a special "FrozenTransactionId" value that is ALWAYS
in the past, for all future transactions.

```sql
-- Check XID age (danger threshold: > 1.5 billion)
SELECT datname,
       age(datfrozenxid) AS xid_age,
       2000000000 - age(datfrozenxid) AS transactions_remaining
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Check table-level XID age:
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 20;

-- Emergency freeze if age is dangerously high:
VACUUM FREEZE ANALYZE products;
-- VACUUM FREEZE: aggressively freezes all tuples below vacuum_freeze_min_age

-- Autovacuum prevents this automatically if tuned correctly:
-- autovacuum_freeze_max_age = 200,000,000 (default)
-- When table age > this, autovacuum runs automatically to freeze

-- Monitor and alert:
-- Alert level: age > 1,000,000,000
-- Emergency level: age > 1,500,000,000
```

---

## Chapter 13 Summary

| Concept | Key Takeaway |
|---------|-------------|
| BEGIN/COMMIT | Explicit transaction boundary; autocommit wraps each statement |
| ROLLBACK | Undoes all changes since BEGIN; aborted transactions also need ROLLBACK |
| SAVEPOINT | Partial rollback without losing entire transaction |
| XID | Sequential 32-bit transaction ID; tags all changes |
| XID Wraparound | Occurs after 4.2B transactions; prevented by VACUUM FREEZE |

---

# CHAPTER 14 — Isolation Levels

---

## 14.1 WHY Isolation Levels Exist

Full isolation (perfect serializability) is expensive — it requires
locking or extensive validation. Many applications can tolerate
weaker guarantees in exchange for better performance.

Isolation levels let you declare: "How much consistency do I need?"

---

## 14.2 The Four Concurrency Anomalies

### Dirty Read
Reading data written by an in-progress (not yet committed) transaction.

```
T1: UPDATE balance = 800 (not committed yet)
T2: SELECT balance → sees 800  ← DIRTY READ
T1: ROLLBACK → balance is still 1000
T2 acted on data that never existed!
```

### Non-Repeatable Read
The same row is read twice in the same transaction and returns different values.

```
T1: SELECT price FROM products WHERE id=1  → 99
T2: UPDATE products SET price=149 WHERE id=1; COMMIT
T1: SELECT price FROM products WHERE id=1  → 149  ← different value!
T1 sees inconsistent data within its own transaction.
```

### Phantom Read
A query returns different rows when repeated within the same transaction.

```
T1: SELECT COUNT(*) FROM orders WHERE customer_id=5  → 3
T2: INSERT INTO orders (customer_id=5, ...); COMMIT
T1: SELECT COUNT(*) FROM orders WHERE customer_id=5  → 4  ← phantom row appeared!
```

### Serialization Anomaly
The combined result of concurrent transactions is impossible to achieve
by running them one at a time in any order.

```
Table: reports (type, value)
  (type='A', value=10)
  (type='B', value=20)

T1: INSERT INTO reports VALUES ('A', (SELECT SUM(value) FROM reports WHERE type='B'))
T2: INSERT INTO reports VALUES ('B', (SELECT SUM(value) FROM reports WHERE type='A'))

If T1 ran first: A gets 20, then B gets 30
If T2 ran first: B gets 10, then A gets 30
Concurrent result: A gets 20, B gets 10  ← impossible in serial execution!
```

---

## 14.3 Isolation Level Comparison Table

```
┌──────────────────────┬──────────────┬──────────────────┬──────────────┬──────────────────────┐
│ Isolation Level      │ Dirty Read   │ Non-Repeatable   │ Phantom Read │ Serialization        │
│                      │              │ Read             │              │ Anomaly              │
├──────────────────────┼──────────────┼──────────────────┼──────────────┼──────────────────────┤
│ Read Uncommitted     │ Possible     │ Possible         │ Possible     │ Possible             │
│ Read Committed (★)   │ Not Possible │ Possible         │ Possible     │ Possible             │
│ Repeatable Read      │ Not Possible │ Not Possible     │ Not Possible*│ Possible             │
│ Serializable         │ Not Possible │ Not Possible     │ Not Possible │ Not Possible         │
└──────────────────────┴──────────────┴──────────────────┴──────────────┴──────────────────────┘
★ = PostgreSQL default
* = PostgreSQL's Repeatable Read also prevents phantom reads (stronger than SQL standard requires)
```

---

## 14.4 READ COMMITTED — The Default

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- OR: this is the default, no need to set it

-- Behavior: each STATEMENT gets a new snapshot of committed data
-- Result: two SELECTs in the same transaction can see different data
```

**Concrete behavior:**

```sql
-- Terminal 1:
BEGIN;
SELECT price FROM products WHERE id=1;  -- returns 99

-- Terminal 2 (runs while Terminal 1 is still open):
UPDATE products SET price=149 WHERE id=1;
COMMIT;

-- Terminal 1 (same transaction, second SELECT):
SELECT price FROM products WHERE id=1;  -- returns 149 ← changed!
-- This is non-repeatable read — allowed under READ COMMITTED
COMMIT;
```

**Use when:**
- General OLTP applications
- You don't need the same query to return consistent results across multiple statements
- Maximum concurrency is needed

---

## 14.5 REPEATABLE READ

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Behavior: the entire transaction uses the snapshot from the transaction START
-- Result: all SELECTs see the same data throughout the transaction
```

**Concrete behavior:**

```sql
-- Terminal 1:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT price FROM products WHERE id=1;  -- returns 99

-- Terminal 2:
UPDATE products SET price=149 WHERE id=1;
COMMIT;

-- Terminal 1 (same transaction):
SELECT price FROM products WHERE id=1;  -- STILL returns 99!
-- The snapshot was taken at transaction start, before T2's change
COMMIT;
```

**The UPDATE conflict:**

```sql
-- Terminal 1 (REPEATABLE READ):
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT price FROM products WHERE id=1;  -- 99

-- Terminal 2:
UPDATE products SET price=149 WHERE id=1;
COMMIT;

-- Terminal 1 tries to update the same row:
UPDATE products SET price=199 WHERE id=1;
-- ERROR: could not serialize access due to concurrent update
-- PostgreSQL refuses the update to prevent inconsistency

-- Application must catch this error and RETRY the transaction
```

**Use when:**
- Reports and analytics that must see consistent data across multiple queries
- Complex calculations that span multiple SELECTs
- Any scenario where non-repeatable reads would cause incorrect results

---

## 14.6 SERIALIZABLE — Full Safety with SSI

PostgreSQL implements Serializable using **SSI (Serializable Snapshot Isolation)**,
not traditional locking. SSI allows high concurrency while detecting actual
serialization anomalies and aborting one of the conflicting transactions.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- PostgreSQL tracks read/write dependencies between transactions
-- If a cycle is detected (T1 read data that T2 wrote, T2 read data T1 wrote),
-- one transaction is aborted with:
-- ERROR: could not serialize access due to read/write dependencies
-- DETAIL: Reason code: Canceled on identification as a pivot, during commit attempt.

-- Application MUST be prepared to retry serialization failures:
for attempt in range(MAX_RETRIES):
    try:
        with conn.transaction():
            conn.execute("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
            # business logic here
        break  # success
    except SerializationError:
        if attempt == MAX_RETRIES - 1:
            raise
        sleep(backoff_time(attempt))
```

**Use when:**
- Financial transactions (transfers, accounting)
- Inventory management (prevent overselling)
- Any scenario where the "write skew" or serialization anomaly would cause real harm

---

## Chapter 14 Summary

| Level | When to Use | Cost |
|-------|------------|------|
| Read Committed | OLTP, web apps, general use | Lowest |
| Repeatable Read | Reports, analytics, multi-statement consistency needed | Low |
| Serializable | Financial, inventory, anything where anomalies are unacceptable | Moderate (retries possible) |

---

# CHAPTER 15 — Locking

---

## 15.1 WHY Locking Exists Alongside MVCC

MVCC handles read-read and read-write conflicts automatically via snapshots.
But write-write conflicts still require locks:

```
Two transactions both try to UPDATE the same row:
  T1: UPDATE products SET stock=stock-1 WHERE id=5  ← which value does T1 use?
  T2: UPDATE products SET stock=stock-1 WHERE id=5  ← and T2?

Without locking:
  Both read stock=10, both write stock=9 → net result: stock=9 (lost one decrement)
  Should be stock=8!

With row locking:
  T1 acquires row lock on id=5, performs update
  T2 waits for T1 to release the lock
  T2 then sees stock=9 (T1's committed value), updates to stock=8
  Correct!
```

---

## 15.2 Table-Level Locks — All 8 Modes

PostgreSQL has 8 table lock modes, ordered from least to most restrictive:

```sql
-- 1. ACCESS SHARE — acquired by SELECT
-- Conflicts with: ACCESS EXCLUSIVE only
-- "I am reading this table"
SELECT * FROM products;  -- acquires ACCESS SHARE

-- 2. ROW SHARE — acquired by SELECT FOR UPDATE/SHARE
-- Conflicts with: EXCLUSIVE, ACCESS EXCLUSIVE
SELECT * FROM products WHERE id=1 FOR SHARE;  -- acquires ROW SHARE

-- 3. ROW EXCLUSIVE — acquired by INSERT, UPDATE, DELETE
-- Conflicts with: SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE
UPDATE products SET price=100 WHERE id=1;  -- acquires ROW EXCLUSIVE

-- 4. SHARE UPDATE EXCLUSIVE — acquired by VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY
-- Conflicts with: itself, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE
VACUUM products;  -- acquires SHARE UPDATE EXCLUSIVE

-- 5. SHARE — acquired by CREATE INDEX (non-concurrent)
-- Conflicts with: ROW EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE
CREATE INDEX ix_price ON products(price);  -- acquires SHARE
-- ⚠️ BLOCKS ALL WRITES during index build!

-- 6. SHARE ROW EXCLUSIVE
-- Conflicts with: ROW EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE
-- Rarely used directly

-- 7. EXCLUSIVE
-- Conflicts with: everything except ACCESS SHARE
-- Rarely used directly

-- 8. ACCESS EXCLUSIVE — acquired by ALTER TABLE, DROP TABLE, VACUUM FULL
-- Conflicts with: EVERYTHING including SELECT
ALTER TABLE products ADD COLUMN category TEXT;  -- acquires ACCESS EXCLUSIVE
-- ⚠️ BLOCKS ALL QUERIES including reads!
```

**Lock compatibility matrix:**
```
              AS  RS  RE  SUE  SH  SRE EX  AX
ACCESS SHARE  ✅  ✅  ✅   ✅  ✅   ✅  ✅  ❌
ROW SHARE     ✅  ✅  ✅   ✅  ✅   ✅  ❌  ❌
ROW EXCL      ✅  ✅  ✅   ✅  ❌   ❌  ❌  ❌
SH UPD EXCL  ✅  ✅  ✅   ❌  ❌   ❌  ❌  ❌
SHARE         ✅  ✅  ❌   ❌  ✅   ❌  ❌  ❌
SH ROW EXCL  ✅  ✅  ❌   ❌  ❌   ❌  ❌  ❌
EXCLUSIVE     ✅  ❌  ❌   ❌  ❌   ❌  ❌  ❌
ACCESS EXCL  ❌  ❌  ❌   ❌  ❌   ❌  ❌  ❌
✅=compatible, ❌=conflict (queued)
```

**Production impact of ACCESS EXCLUSIVE:**

```sql
-- Dangerous pattern: ALTER TABLE on a busy production table
ALTER TABLE orders ADD COLUMN notes TEXT;
-- This queues behind all running queries, then blocks ALL new queries
-- until the ALTER completes and releases the lock

-- Safe pattern: use lock_timeout to fail fast instead of blocking:
SET lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN notes TEXT;
-- If lock not acquired in 3s: ERROR (retry during low-traffic window)

-- Even safer: use pg_repack or compatible migrations:
-- 1. Add column as nullable (fast, brief lock)
ALTER TABLE orders ADD COLUMN notes TEXT;
-- 2. Populate in batches (no lock needed)
UPDATE orders SET notes = '' WHERE id BETWEEN 1 AND 10000;
-- 3. Add NOT NULL constraint with DEFAULT already set (PG 11+ is fast)
```

---

## 15.3 Row-Level Locks

```sql
-- FOR UPDATE: exclusive row lock
-- Blocks: other FOR UPDATE, FOR NO KEY UPDATE, UPDATE, DELETE on same rows
-- Use: when you will definitely update the row
SELECT * FROM products WHERE id=1 FOR UPDATE;

-- FOR NO KEY UPDATE: exclusive but allows key-share on FK references
-- Use: when updating non-key columns, allowing concurrent FK inserts

-- FOR SHARE: shared row lock
-- Blocks: FOR UPDATE, FOR NO KEY UPDATE, UPDATE, DELETE
-- Allows: multiple FOR SHARE simultaneously
-- Use: when you need to read and ensure the row is not deleted/updated

-- FOR KEY SHARE: weakest row lock
-- Blocks: FOR UPDATE, DELETE (key changes)
-- Allows: FOR SHARE, FOR NO KEY UPDATE, most updates
-- Use: FK reference checks in concurrent insert scenarios

-- NOWAIT: fail immediately if lock unavailable (don't queue)
SELECT * FROM products WHERE id=1 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "products"
-- Use: when your app can handle "try again later" gracefully

-- SKIP LOCKED: skip rows that are locked, process available ones
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY priority DESC
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- Perfect for implementing a concurrent job queue:
-- Worker 1 and Worker 2 both run this simultaneously
-- Worker 1 gets job_id=10 (locks it)
-- Worker 2 gets job_id=11 (10 is locked, so it's skipped)
-- No deadlock, no worker sitting idle waiting for a lock
```

---

## 15.4 Advisory Locks

Advisory locks are entirely managed by your application.
PostgreSQL provides the mechanism; you decide what the lock means.

```sql
-- Use case: "Only one process should recalculate statistics at a time"

-- Process 1:
SELECT pg_advisory_lock(42);       -- acquires exclusive lock on key 42
-- ... do the calculation ...
SELECT pg_advisory_unlock(42);     -- release

-- Process 2 (while Process 1 holds lock):
SELECT pg_advisory_lock(42);       -- BLOCKS until Process 1 releases
-- or non-blocking:
SELECT pg_try_advisory_lock(42);   -- returns FALSE immediately if locked
-- → Application can decide: "already running, skip this run"

-- Transaction-scoped advisory locks (released at COMMIT/ROLLBACK automatically):
BEGIN;
SELECT pg_advisory_xact_lock(100);
-- ... do work ...
COMMIT;  -- lock auto-released

-- Shared advisory locks (multiple readers):
SELECT pg_advisory_lock_shared(200);   -- shared — multiple can hold
SELECT pg_advisory_unlock_shared(200);

-- Namespace your lock keys to avoid collisions:
-- Convention: use hash of a meaningful string
SELECT pg_advisory_lock(hashtext('recalculate_daily_stats'));
```

---

## Chapter 15 Summary

| Lock Type | Acquired By | Blocks | Use Case |
|-----------|-------------|--------|---------|
| ACCESS SHARE | SELECT | ACCESS EXCLUSIVE only | Reading — almost never a problem |
| ROW EXCLUSIVE | INSERT/UPDATE/DELETE | SHARE and above | All normal writes |
| SHARE | CREATE INDEX | All writes | Index build blocks writes! Use CONCURRENTLY |
| ACCESS EXCLUSIVE | ALTER TABLE | Everything | DDL changes — use lock_timeout |
| Row FOR UPDATE | SELECT FOR UPDATE | Other writers on same row | Optimistic locking pattern |
| FOR UPDATE SKIP LOCKED | SELECT FOR UPDATE SKIP LOCKED | Nothing (skips) | Job queues |
| Advisory | pg_advisory_lock() | Other advisory locks on same key | Application-level coordination |

---

# CHAPTER 16 — Deadlocks

---

## 16.1 WHY Deadlocks Happen

A deadlock occurs when two (or more) transactions are each waiting
for a lock held by the other. Neither can proceed. Both are stuck forever.

```
Timeline:
  T1: LOCK row A → success, T1 holds lock on A
  T2: LOCK row B → success, T2 holds lock on B
  T1: LOCK row B → WAIT (T2 holds it)
  T2: LOCK row A → WAIT (T1 holds it)
  ← DEADLOCK: T1 waits for T2, T2 waits for T1
```

---

## 16.2 HOW PostgreSQL Detects Deadlocks

PostgreSQL does NOT detect deadlocks immediately. It waits for
`deadlock_timeout` (default: 1 second) before running deadlock detection.

The detection algorithm builds a **wait-for graph**:

```
Wait-for graph:
  T1 → T2  (T1 is waiting for a lock held by T2)
  T2 → T1  (T2 is waiting for a lock held by T1)
  
  A cycle in this graph = deadlock

Detection: depth-first search for cycles
  O(n²) where n = number of waiting transactions
  Runs every deadlock_timeout milliseconds
```

When a cycle is found, PostgreSQL **aborts the youngest transaction**
(lowest cost to retry) and returns this error:

```
ERROR:  deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5678; blocked by process 5678.
        Process 5678 waits for ShareLock on transaction 1234; blocked by process 1234.
HINT:   See server log for query details.
```

The surviving transaction's lock is granted and it continues.
The aborted transaction receives the error — the application must ROLLBACK and retry.

---

## 16.3 Common Deadlock Patterns

### Pattern 1: Opposite Update Order (Most Common)

```sql
-- Application code that processes multiple accounts/rows:

-- Transaction T1: process account 1 then account 2
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- LOCK id=1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- WAIT for T2's lock on id=2

-- Transaction T2 (concurrent): process account 2 then account 1
BEGIN;
UPDATE accounts SET balance = balance - 200 WHERE id = 2;  -- LOCK id=2
UPDATE accounts SET balance = balance + 200 WHERE id = 1;  -- WAIT for T1's lock on id=1
-- DEADLOCK!

-- FIX: Always update rows in consistent order (by ID, ascending)
-- Both T1 and T2:
BEGIN;
-- Lock id=1 first, then id=2 (always this order)
UPDATE accounts SET balance = balance + delta WHERE id IN (1, 2) ORDER BY id;
COMMIT;
-- T1 and T2 both try to lock id=1 first → one waits → no cycle → no deadlock
```

### Pattern 2: SELECT then UPDATE Without Locking

```sql
-- T1: read, decide, write
BEGIN;
SELECT stock FROM products WHERE id=5;  -- reads 10 (NO LOCK acquired)
-- Application decides: 10 > 0, so place order
-- [time passes — T2 runs here and also reads stock=10]
UPDATE products SET stock = stock - 1 WHERE id=5;  -- now acquires lock
COMMIT;

-- T2: same pattern
BEGIN;
SELECT stock FROM products WHERE id=5;  -- reads 10 (no lock)
UPDATE products SET stock = stock - 1 WHERE id=5;  -- waits for T1's lock
COMMIT;
-- Both reduce by 1 based on reading 10: correct result = 8 NOT 9
-- Actually this is NOT a deadlock but a race condition (lost update)

-- FIX: Lock the row at read time
BEGIN;
SELECT stock FROM products WHERE id=5 FOR UPDATE;  -- acquires lock immediately
-- T2 now WAITS here until T1 commits or rolls back
-- T2 then reads the updated stock value
UPDATE products SET stock = stock - 1 WHERE id=5;
COMMIT;
```

### Pattern 3: FK Constraint Induced Deadlock

```sql
-- order_items has FK to products
-- T1: INSERT order_item for product 5, then UPDATE product 5
-- T2: UPDATE product 5, then INSERT order_item for product 5

-- T1's INSERT into order_items: acquires KEY SHARE lock on products row 5
-- T2's UPDATE on products row 5: acquires ROW EXCLUSIVE lock → waits for T1
-- T1's UPDATE on products row 5: tries to upgrade to ROW EXCLUSIVE → DEADLOCK

-- FIX: consistent order of operations
-- Always UPDATE the parent (products) BEFORE inserting child (order_items)
```

---

## 16.4 Preventing Deadlocks

```sql
-- Rule 1: Always acquire locks in a consistent global order
-- Sort IDs, always update lower ID first

-- Rule 2: Use SELECT FOR UPDATE to acquire locks early (at read time, not write time)
BEGIN;
SELECT * FROM products WHERE id IN (1, 5, 10) FOR UPDATE ORDER BY id;
-- Lock all 3 rows at once, in order → no interleaving possible
UPDATE products SET stock = stock - qty WHERE id = 1;
UPDATE products SET stock = stock - qty WHERE id = 5;
COMMIT;

-- Rule 3: Keep transactions short
-- Longer transactions hold locks longer → more contention → more deadlocks

-- Rule 4: Set deadlock_timeout to detect quickly
-- In postgresql.conf:
-- deadlock_timeout = '500ms'  -- default 1000ms; lower detects faster

-- Rule 5: Use lock_timeout to fail fast rather than queue forever
SET lock_timeout = '5000ms';
-- If you can't get the lock in 5 seconds, fail and retry rather than waiting indefinitely

-- Rule 6: Application must retry on deadlock
-- Deadlock error code: '40P01'
try:
    perform_transaction()
except psycopg2.errors.DeadlockDetected:
    conn.rollback()
    time.sleep(random.uniform(0.1, 0.5))  # random backoff
    perform_transaction()  # retry
```

---

## 16.5 Monitoring and Diagnosing Locks

```sql
-- See all current locks:
SELECT
    pg_stat_activity.pid,
    pg_stat_activity.query,
    pg_stat_activity.wait_event_type,
    pg_stat_activity.wait_event,
    pg_locks.relation::regclass AS locked_table,
    pg_locks.mode,
    pg_locks.granted
FROM pg_stat_activity
JOIN pg_locks ON pg_stat_activity.pid = pg_locks.pid
WHERE pg_locks.relation IS NOT NULL
ORDER BY pg_stat_activity.pid;

-- Find blocking queries (who is blocking whom):
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    NOW() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';

-- Kill a specific blocking query (graceful — sends SIGINT):
SELECT pg_cancel_backend(blocking_pid);

-- Kill forcefully (use only if cancel doesn't work):
SELECT pg_terminate_backend(blocking_pid);

-- Log all lock waits in postgresql.conf:
-- log_lock_waits = on
-- deadlock_timeout = '1s'  -- waits longer than this get logged

-- Check for deadlocks in the log:
-- grep "deadlock detected" /var/log/postgresql/postgresql.log
```

---

## Chapter 16 Summary

| Topic | Key Takeaway |
|-------|-------------|
| Detection | PostgreSQL checks for deadlock cycles every `deadlock_timeout` (default 1s) |
| Resolution | Youngest (cheapest) transaction is aborted; app must retry |
| Primary cause | Acquiring locks in different orders in different transactions |
| Prevention | Lock in consistent order; use `FOR UPDATE` early; keep transactions short |
| `lock_timeout` | Fail fast rather than queue forever; prevents lock queue cascades |
| Monitoring | `pg_stat_activity` + `pg_blocking_pids()` shows real-time lock contention |

# PART V — QUERY OPTIMIZATION & EXECUTION PLANS

---

# CHAPTER 17 — The Query Planner

---

## 17.1 WHY the Planner Exists

SQL is declarative — you say WHAT you want, not HOW to get it.
The planner's job is to figure out HOW.

For a query joining 3 tables with 2 WHERE conditions, there are dozens of possible
execution plans. Some complete in 2ms, others in 2 minutes. The planner's job is
to find the fast one without trying all of them.

---

## 17.2 The Cost Model

The planner assigns a **numeric cost** to every possible plan.
Cost units are arbitrary but proportional to actual wall-clock time.

The base parameters (tunable in `postgresql.conf`):

```sql
-- Show current cost parameters:
SHOW seq_page_cost;            -- 1.0  (cost to read one page sequentially)
SHOW random_page_cost;         -- 4.0  (cost to read one page randomly — for HDD)
SHOW cpu_tuple_cost;           -- 0.01 (cost to process one row)
SHOW cpu_index_tuple_cost;     -- 0.005
SHOW cpu_operator_cost;        -- 0.0025 (cost per operation: comparison, etc.)
SHOW parallel_tuple_cost;      -- 0.1  (cost to pass row between parallel workers)

-- CRITICAL: Tune for SSD (random reads are cheap on SSD):
-- In postgresql.conf:
random_page_cost = 1.1  -- SSD: random ≈ sequential (not 4× more expensive)
effective_cache_size = 12GB  -- 75% of RAM; hints planner how much OS can cache
```

**Example cost calculation:**

```sql
-- Table: products (50,000 rows, 500 pages of 8KB)
-- Query: SELECT * FROM products WHERE sku = 'WID-001'

-- Plan A: Sequential Scan + Filter
-- Cost = seq_page_cost × pages + cpu_tuple_cost × rows_examined
-- Cost = 1.0 × 500 + 0.01 × 50,000 = 500 + 500 = 1,000 cost units

-- Plan B: Index Scan (index on sku, ~1 matching row, B-Tree height = 3)
-- Cost = random_page_cost × index_pages + random_page_cost × heap_pages
-- Cost = 4.0 × 3 + 4.0 × 1 = 12 + 4 = 16 cost units

-- Winner: Plan B (index scan), 62× cheaper
-- This is why EXPLAIN shows: Index Scan using ix_products_sku
```

---

## 17.3 Statistics Catalog — pg_statistic

The planner uses statistics to estimate how many rows each filter returns.
Without accurate statistics, cost estimates are wrong → bad plan.

```sql
-- Statistics are collected by ANALYZE (and autovacuum runs ANALYZE automatically)
ANALYZE products;

-- View the statistics:
SELECT
    attname AS column,
    n_distinct,               -- number of distinct values (-1 = all unique)
    correlation,              -- physical correlation (1.0 = sorted, 0.0 = random)
    most_common_vals,         -- top N most common values
    most_common_freqs,        -- frequency of each most_common_val
    histogram_bounds          -- equal-frequency histogram
FROM pg_stats
WHERE tablename = 'products';

-- Example output for 'status' column:
-- n_distinct: 3  (only 3 distinct statuses: pending/active/closed)
-- most_common_vals: {active, pending, closed}
-- most_common_freqs: {0.75, 0.15, 0.10}
-- → Planner knows: "WHERE status='active'" returns ~75% of rows

-- Example output for 'sku' column:
-- n_distinct: -1  (all values are unique: -1 means 100% distinct)
-- histogram_bounds: alphabetical distribution
-- → Planner knows: "WHERE sku='WID-001'" returns ~1 row (1/50,000 = 0.002%)
```

**What goes wrong without good statistics:**

```sql
-- After bulk inserting 1M new rows without ANALYZE:
-- pg_statistic still thinks table has 10,000 rows
-- Planner estimates 100 rows returned by filter (actually 100,000!)
-- Planner chooses: Nested Loop (good for 100 rows)
-- Actual performance: catastrophic for 100,000 rows

-- Fix:
ANALYZE products;  -- update statistics immediately

-- Best practice: always ANALYZE after bulk loads
INSERT INTO products SELECT ... FROM import_data;
ANALYZE products;
```

---

## 17.4 Selectivity Estimation

Selectivity = fraction of rows a predicate returns.

```
Low selectivity:  WHERE status = 'active'  → 75% of rows (low selectivity = many rows)
High selectivity: WHERE sku = 'WID-001'    → 0.002% of rows (high selectivity = few rows)

Planner rule:
  High selectivity (< 5%) → use index (few random reads worth it)
  Low selectivity (> 5-10%) → use seq scan (index overhead not worth it)
```

**The independence assumption:** By default, the planner assumes all predicates
are statistically independent. This is often wrong:

```sql
-- WHERE city = 'Mumbai' AND state = 'Maharashtra'
-- Planner assumes: P(city=Mumbai) × P(state=Maharashtra) = independent
-- Reality: every Mumbai order is ALSO Maharashtra → they are perfectly correlated

-- Result: planner underestimates rows → chooses wrong plan

-- Fix: extended statistics for correlated columns
CREATE STATISTICS stat_city_state (dependencies) ON city, state FROM orders;
ANALYZE orders;
-- Planner now learns: city and state are 100% correlated
-- Estimates become accurate → correct plan chosen
```

---

## 17.5 Planner Hints Workaround

PostgreSQL does NOT have a native query hint system (unlike Oracle's `/*+ INDEX(t ix) */`).

**Legitimate approaches:**

```sql
-- Approach 1: Disable/enable specific algorithms (session-level, testing only)
SET enable_seqscan = off;   -- forces index use when planner would choose seq scan
SET enable_hashjoin = off;  -- forces nested loop or merge join
SET enable_nestloop = off;  -- forces hash or merge join
EXPLAIN ANALYZE SELECT ...;
RESET ALL;  -- always reset after testing!

-- Approach 2: Run ANALYZE to update stale statistics (usually the real fix)
ANALYZE tablename;

-- Approach 3: Raise statistics target for important skewed columns
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
-- Default is 100 histogram buckets; 500 gives more precision
ANALYZE orders;

-- Approach 4: Rewrite query to guide the planner
-- Force a specific join order by disabling join_collapse:
SET join_collapse_limit = 1;  -- planner uses your written join order exactly
SELECT ... FROM small_table JOIN large_table ON ... JOIN another ON ...;
RESET join_collapse_limit;

-- Approach 5: pg_hint_plan extension (third-party, production-safe)
-- Install: CREATE EXTENSION pg_hint_plan;
/*+ SeqScan(products) HashJoin(orders customers) */
SELECT o.id, c.name FROM orders o JOIN customers c ON o.customer_id = c.id;
```

---

## Chapter 17 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Cost model | seq_page_cost=1, random_page_cost=4 (HDD); set to 1.1 for SSD |
| Statistics | Collected by ANALYZE; stale stats = wrong plans; always ANALYZE after bulk loads |
| Selectivity | High selectivity (<5%) → index; low selectivity (>10%) → seq scan |
| Extended statistics | Fixes independence assumption for correlated columns |
| Planner hints | Use enable_* flags for testing only; fix root cause (stats/indexes) in production |

---

# CHAPTER 18 — EXPLAIN ANALYZE

---

## 18.1 WHY EXPLAIN ANALYZE is Your Most Important Tool

EXPLAIN ANALYZE is the only way to see what PostgreSQL is actually doing.
Without it, query optimization is guesswork. With it, you can:
- See exactly which indexes are used (or not)
- See how many rows the planner estimated vs how many actually came back
- See exactly where time is spent
- See when sorts and hash joins spill to disk
- Find the specific node that is slow in a complex plan

Every performance investigation must start here.

---

## 18.2 Reading a Plan — Complete Reference

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, c.full_name, o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.total_amount > 1000
ORDER BY o.created_at DESC
LIMIT 10;
```

Annotated output:
```
Limit  (cost=2345.67..2345.70 rows=10 width=48)
       (actual time=45.123..45.234 rows=10 loops=1)
  Buffers: shared hit=1200 read=300
  ├─ Sort  (cost=2345.67..2370.67 rows=10000 width=48)
  │        (actual time=44.890..45.100 rows=10 loops=1)
  │    Sort Key: o.created_at DESC
  │    Sort Method: top-N heapsort  Memory: 27kB
  │    Buffers: shared hit=1200 read=300
  │    └─ Hash Join  (cost=500.00..1900.00 rows=10000 width=48)
  │                  (actual time=15.234..40.567 rows=8432 loops=1)
  │         Hash Cond: (o.customer_id = c.id)
  │         Buffers: shared hit=1200 read=300
  │         ├─ Seq Scan on orders o  (cost=0.00..1200.00 rows=50000 width=24)
  │         │                        (actual time=0.123..25.456 rows=50000 loops=1)
  │         │    Filter: (total_amount > 1000)
  │         │    Rows Removed by Filter: 41568
  │         │    Buffers: shared hit=800 read=200
  │         └─ Hash  (cost=300.00..300.00 rows=20000 width=24)
  │                  (actual time=12.345..12.345 rows=20000 loops=1)
  │              Buckets: 32768  Batches: 1  Memory Usage: 1248kB
  │              Buffers: shared hit=400 read=100
  │              └─ Seq Scan on customers c  (cost=0.00..300.00 rows=20000 width=24)
  │                                         (actual time=0.045..8.123 rows=20000 loops=1)
  │                  Buffers: shared hit=400 read=100
Planning Time: 1.234 ms
Execution Time: 45.678 ms
```

**Decoding every field:**

```
cost=2345.67..2345.70
      ↑           ↑
  startup cost  total cost
  (time to first row)  (time for ALL rows)

rows=10   ← estimated rows this node will output
width=48  ← estimated bytes per output row

actual time=45.123..45.234
             ↑          ↑
          time to      time for
          first row    all rows
          (milliseconds)

loops=1   ← how many times this node was executed
           (multiply actual time × loops for total node time)

Buffers: shared hit=1200 read=300
                      ↑         ↑
                 from cache  from DISK
                 (fast)      (SLOW — each is a random I/O)

Filter: (total_amount > 1000)
Rows Removed by Filter: 41568
← 41,568 rows read and discarded → add index on total_amount!

Sort Method: top-N heapsort  Memory: 27kB
← "top-N heapsort" = efficient sort for LIMIT queries (only keeps top 10)
← if "external merge  Disk: 4096kB" → sort spilled to disk → increase work_mem

Batches: 1  Memory Usage: 1248kB
← Batches > 1 means hash join spilled to disk → increase work_mem
```

---

## 18.3 The loops Multiplier — The Most Common Mistake

```sql
-- Plan shows:
Nested Loop  (actual time=0.01..0.05 rows=1 loops=10000)
  → Index Scan (actual time=0.01..0.03 rows=1 loops=10000)

-- "That looks fast — 0.05ms per loop!"
-- WRONG: total time = 0.05ms × 10,000 loops = 500ms

-- Always multiply: actual_time × loops = real node time
-- The EXPLAIN output shows PER-LOOP timing, not total timing!
```

---

## 18.4 Warning Signs Checklist

```
⚠️  Seq Scan on large table (rows > 10,000)
    → Add index on the filter column

⚠️  Rows Removed by Filter: very high number
    → The filter is removing most rows → index would avoid reading them

⚠️  Sort Method: external merge  Disk: Xkb
    → Sort spilled to disk → increase work_mem or add index on ORDER BY column

⚠️  Hash  Batches: 4  (or any number > 1)
    → Hash join spilled to disk → increase work_mem

⚠️  Estimated rows=10, actual rows=100000
    → Stale or inaccurate statistics → run ANALYZE, add extended statistics

⚠️  Nested Loop with loops=N where N is large
    → N × inner_table_cost is the real cost → likely missing index on inner side

⚠️  Buffers: read=very_large_number
    → Most data coming from disk → increase shared_buffers or optimize indexes
    → run the query twice: second run should show mostly "hit" (cached)

⚠️  Planning Time: 500ms
    → Planning itself is slow → too many tables, join_collapse_limit issue
    → Cache query plans (use prepared statements)
```

---

## 18.5 Using explain.dalibo.com for Visual Analysis

1. Run: `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) your_query;`
2. Copy the entire JSON output (including the square brackets)
3. Go to [explain.dalibo.com](https://explain.dalibo.com)
4. Paste and click "Submit"

The visual output shows:
- Width of each node box = relative cost (wider = more expensive)
- Red/orange nodes = hotspots (most time spent here)
- Arrows showing row counts
- Estimated vs actual rows side by side

---

## Chapter 18 Summary

| EXPLAIN Field | What to Look For |
|--------------|----------------|
| `actual time` | Per-loop time; multiply by `loops` for real time |
| `Rows Removed by Filter` | High value → missing index on filter column |
| `Sort Method: external merge` | Spilled to disk → increase `work_mem` |
| `Batches > 1` | Hash join spilled to disk → increase `work_mem` |
| `Buffers: read=N` | Disk reads → cache miss; optimize indexes or increase `shared_buffers` |
| `estimated rows ≠ actual rows` | Stale stats → run `ANALYZE`; or add extended statistics |

---

# CHAPTER 19 — Scan Nodes

---

## 19.1 Sequential Scan

Reads every page in the table from first to last. No index involved.

```sql
EXPLAIN SELECT * FROM products WHERE stock_quantity < 10;

-- Sequential Scan chosen when:
-- 1. No index exists on stock_quantity
-- 2. stock_quantity < 10 matches > ~10% of rows (low selectivity → seq scan is cheaper)
-- 3. Table is small (< ~1000 rows — index overhead not worth it)

-- Output:
Seq Scan on products  (cost=0.00..250.00 rows=5000 width=64)
                      (actual time=0.045..12.345 rows=4821 loops=1)
  Filter: (stock_quantity < 10)
  Rows Removed by Filter: 195179

-- 195,179 rows read and discarded → 4,821 returned
-- This means 97.6% of all rows were read and thrown away!
-- → GREAT candidate for an index: CREATE INDEX ON products(stock_quantity)
```

---

## 19.2 Index Scan

Uses B-Tree to find heap row locations, then fetches each from the heap.

```sql
EXPLAIN SELECT * FROM products WHERE sku = 'WID-001';

Index Scan using ix_products_sku on products
  Index Cond: (sku = 'WID-001')
  -- No "Rows Removed by Filter" — index was perfectly selective

-- Two I/O steps per matching row:
-- 1. Read index page → get ctid (page 43, slot 7)
-- 2. Read heap page 43 → get full row

-- When index scan is WORSE than seq scan:
-- Query returns 50,000 scattered rows from a 100,000 row table
-- 50,000 random heap reads = 50,000 × 0.1ms = 5 seconds
-- Sequential scan: 100,000 rows in ~400 pages = 400 × 0.08ms = 32ms
-- → Planner correctly chooses seq scan when selectivity is low
```

---

## 19.3 Bitmap Heap Scan — The Middle Ground

For queries that return too many rows for Index Scan but too few for Seq Scan.

```sql
EXPLAIN SELECT * FROM products WHERE price BETWEEN 50 AND 500;

-- Returns 30% of products — too many for Index Scan, too few to bother with Seq Scan
Bitmap Heap Scan on products
  Recheck Cond: (price >= 50 AND price <= 500)
  →  Bitmap Index Scan on ix_products_price
       Index Cond: (price >= 50 AND price <= 500)

-- What happens:
-- Phase 1 (Bitmap Index Scan):
--   Reads index, collects ALL matching page numbers into a bitmap
--   Bitmap: [page 0=YES, page 1=NO, page 2=YES, ...]
-- Phase 2 (Bitmap Heap Scan):
--   Reads marked pages IN ORDER (sequential I/O, fast!)
--   "Recheck Cond" = verify exact rows within each page (in case of lossy bitmap)

-- Multiple indexes combined with BitmapAnd/BitmapOr:
EXPLAIN SELECT * FROM products WHERE price > 100 AND stock_quantity < 10;

Bitmap Heap Scan on products
  Recheck Cond: (price > 100 AND stock_quantity < 10)
  →  BitmapAnd
       →  Bitmap Index Scan on ix_products_price
       →  Bitmap Index Scan on ix_products_stock
-- Uses BOTH indexes simultaneously!
```

---

## 19.4 Index Only Scan

When the index contains all columns the query needs, heap is never accessed.

```sql
CREATE INDEX ix_products_sku_name_price ON products(sku) INCLUDE (name, price);

EXPLAIN SELECT name, price FROM products WHERE sku = 'WID-001';

Index Only Scan using ix_products_sku_name_price on products
  Index Cond: (sku = 'WID-001')
  Heap Fetches: 0   ← zero! Never touched the heap table

-- Heap Fetches > 0 means some pages are not "all-visible" in the visibility map
-- Fix: VACUUM products;
-- After vacuum: Heap Fetches drops to 0
```

---

## 19.5 TID Scan

Fetches a specific row by its physical address (ctid). Rarely used directly.

```sql
-- Direct physical row fetch:
SELECT * FROM products WHERE ctid = '(0,1)';
-- TID Scan on products
--   TID Cond: (ctid = '(0,1)'::tid)

-- Use cases: internal PostgreSQL operations, debugging, rarely in applications
-- ctid changes after VACUUM FULL or CLUSTER → never store ctid in application
```

---

## Chapter 19 Summary

| Scan Type | When Used | I/O Pattern | Sweet Spot |
|-----------|----------|------------|-----------|
| Seq Scan | No index, low selectivity, small table | Sequential (fast) | > 10% of rows |
| Index Scan | High selectivity, small result | Random (1 index + 1 heap per row) | < 1% of rows |
| Bitmap Heap Scan | Medium selectivity | Index random → heap sequential | 1–10% of rows |
| Index Only Scan | Covering index, all-visible pages | Index only, no heap | Any selectivity if covering |

---

# CHAPTER 20 — Query Rewriting

---

## 20.1 SARGable Predicates

SARG = **Search ARGument** capable. A predicate is SARGable if PostgreSQL
can use an index to satisfy it.

**The golden rule:** The indexed column must appear **bare** (unwrapped) in the predicate.

```sql
-- ❌ NOT SARGable (function on column → index ignored):
WHERE UPPER(name) = 'WIDGET'           → seq scan
WHERE DATE(created_at) = '2024-01-15'  → seq scan
WHERE price * 1.1 > 110               → seq scan
WHERE CAST(id AS TEXT) = '42'          → seq scan
WHERE LEFT(sku, 3) = 'WID'             → seq scan

-- ✅ SARGable rewrites:
WHERE name = 'Widget'                  → index on name
WHERE created_at >= '2024-01-15'
  AND created_at < '2024-01-16'        → index on created_at
WHERE price > 100                      → index on price
WHERE id = 42                          → index on id (integer, not text)
WHERE sku LIKE 'WID%'                  → index on sku (prefix only)

-- The pattern: move the function to the VALUE side, not the column side:
-- BAD:  WHERE YEAR(created_at) = 2024
-- GOOD: WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- Or create an expression index to make non-SARGable predicates SARGable:
CREATE INDEX ON customers(UPPER(name));
WHERE UPPER(name) = 'RAHUL'  -- now uses the expression index ✅
```

---

## 20.2 Parallel Query — Using Multiple CPU Cores

PostgreSQL can split a single query across multiple parallel worker processes:

```sql
-- See parallel settings:
SHOW max_parallel_workers_per_gather;  -- default: 2
SHOW min_parallel_table_scan_size;     -- table must be > 8MB for parallelism

-- Parallel query in action:
EXPLAIN SELECT AVG(total_amount) FROM orders;

Finalize Aggregate  (cost=5000..5001 rows=1 width=32)
  →  Gather  (cost=4800..4801 rows=2 width=32)
       Workers Planned: 2        ← 2 parallel workers
       →  Partial Aggregate
            →  Parallel Seq Scan on orders
                 Workers Planned: 2

-- Each worker scans 1/3 of the table (leader + 2 workers = 3 parts)
-- Each computes a partial AVG
-- Leader combines partial results → final AVG
-- Speedup: ~2.5× for I/O-bound queries, ~3× for CPU-bound

-- Force more workers (per session):
SET max_parallel_workers_per_gather = 4;

-- Disable parallelism (for testing or when it hurts):
SET max_parallel_workers_per_gather = 0;
```

**When parallelism HURTS:**
- Small tables: overhead of spawning workers exceeds benefit
- OLTP queries that return few rows: already fast, parallelism adds overhead
- When work_mem × workers × connections exceeds RAM

---

## 20.3 work_mem Tuning — Sorts and Hash Joins

`work_mem` is the memory limit per **sort operation or hash table**,
not per query or connection.

```sql
SHOW work_mem;  -- default: 4MB (very low for production)

-- A single complex query might use multiple work_mem allocations:
-- SELECT ... JOIN ... JOIN ... ORDER BY ... GROUP BY ...
-- → Hash Join 1: uses 1 work_mem
-- → Hash Join 2: uses 1 work_mem
-- → Sort:        uses 1 work_mem
-- Total: 3 × work_mem = 3 × 4MB = 12MB for ONE query

-- Danger: 100 connections × 3 sorts × 64MB = 19GB RAM!
-- Set work_mem conservatively for global setting:
-- work_mem = 16MB  (in postgresql.conf)

-- Set higher for specific analytical queries:
SET work_mem = '256MB';
EXPLAIN ANALYZE SELECT ... complex analytics ...;
RESET work_mem;

-- Diagnosing work_mem issues:
-- Sort spill: Sort Method: external merge  Disk: 4096kB → increase work_mem
-- Hash spill: Hash  Batches: 4  Disk: 8192kB → increase work_mem
```

---

# PART VI — PAGINATION STRATEGIES

---

# CHAPTER 21 — OFFSET/LIMIT

---

## 21.1 HOW OFFSET Actually Works

The SQL clause `OFFSET 1000` does NOT skip 1000 rows efficiently.
It READS 1000 rows and discards them:

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 1000;

-- What PostgreSQL does:
-- 1. Sort ALL orders by created_at DESC (if no index → full table sort)
-- 2. Read rows 1 through 1020
-- 3. Discard rows 1 through 1000
-- 4. Return rows 1001 through 1020

-- EXPLAIN:
Limit  (actual rows=20 loops=1)
  →  Sort  (actual rows=1020 loops=1)  ← sorts 1020 rows to get 20
       Sort Key: created_at DESC
       →  Seq Scan on orders  (actual rows=1000000 loops=1)  ← scans ALL rows!
```

**Performance degradation curve:**
```
OFFSET 0:       reads 20 rows    → fast    (~2ms)
OFFSET 1000:    reads 1020 rows  → OK      (~5ms)
OFFSET 10000:   reads 10020 rows → slow    (~50ms)
OFFSET 100000:  reads 100020 rows → very slow (~500ms)
OFFSET 1000000: reads 1000020 rows → terrible (~5000ms)
```

Each page is proportionally slower. Page 50,001 requires reading
1,000,000+ rows to throw away 999,980 of them.

---

## 21.2 When OFFSET is Acceptable

```sql
-- OFFSET is fine when:
-- 1. Table is small (< 10,000 rows): even OFFSET 9000 is instant
-- 2. Admin/internal tools: low traffic, occasional large offsets OK
-- 3. Fixed small pagination: reports that never go past page 10
-- 4. When combined with a good index (ORDER BY clause uses index → no sort needed)

-- With index on created_at: index scan + OFFSET = faster but still degrades
CREATE INDEX ix_orders_created ON orders(created_at DESC);

EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
Limit
  →  Index Scan Backward using ix_orders_created on orders
-- Now: scans 10,020 index entries + 10,020 heap fetches
-- Better than before, but still reads 10,020 rows to return 20
```

---

# CHAPTER 22 — Keyset (Cursor) Pagination

---

## 22.1 HOW Keyset Pagination Works

Instead of OFFSET, use the last seen row's value as a filter:

```sql
-- Page 1: no cursor yet
SELECT id, total_amount, created_at
FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Last row returned: created_at='2024-03-15 10:30:00', id=5432

-- Page 2: use last row as cursor
SELECT id, total_amount, created_at
FROM orders
WHERE (created_at, id) < ('2024-03-15 10:30:00', 5432)
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- This query uses the index DIRECTLY — no rows are thrown away
-- Returns exactly 20 rows, reads exactly those 20 rows from the index

-- Page 3: use page 2's last row as cursor
-- ... and so on
```

**Performance:**
```
Page 1:    reads 20 rows   → 2ms
Page 2:    reads 20 rows   → 2ms (same!)
Page 1000: reads 20 rows   → 2ms (same!)
Page 50000: reads 20 rows  → 2ms (same!)
```

Every page is O(log n) → constant performance regardless of page depth.

---

## 22.2 Index Alignment for Keyset Pagination

The keyset WHERE clause must match the ORDER BY clause, and both must
align with an index:

```sql
-- ORDER BY: created_at DESC, id DESC
-- WHERE:    (created_at, id) < (cursor_date, cursor_id)
-- Required index: (created_at DESC, id DESC)

CREATE INDEX ix_orders_keyset ON orders(created_at DESC, id DESC);

-- EXPLAIN confirms:
EXPLAIN SELECT * FROM orders
WHERE (created_at, id) < ('2024-03-15', 5432)
ORDER BY created_at DESC, id DESC
LIMIT 20;

Limit  (rows=20)
  →  Index Scan Backward using ix_orders_keyset on orders
       Index Cond: ROW(created_at, id) < ROW('2024-03-15'::timestamptz, 5432)
-- No Sort node! Index already provides the order.
-- Reads exactly 20 pages: efficient!
```

---

## 22.3 Stable Sort Requirements

If rows can have the same `created_at` value (ties), you MUST include
a unique tie-breaker in both ORDER BY and the cursor:

```sql
-- BAD: created_at alone is not unique — ties cause missing or duplicated rows
WHERE created_at < '2024-03-15'
ORDER BY created_at DESC
LIMIT 20;

-- BAD but less obvious: row order within same created_at is undefined
-- Some rows might appear on page 1 AND page 2 (duplicate) or neither (missing)

-- GOOD: always include id as tie-breaker (id is always unique)
WHERE (created_at, id) < ('2024-03-15', 5432)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- The (created_at, id) tuple is now globally unique → stable sort → no duplicates
```

---

## 22.4 Handling Deletions Mid-Pagination

When rows are deleted while a user paginates, keyset pagination handles it gracefully:

```sql
-- User is on page 5, cursor = (created_at='2024-01-10', id=3000)
-- Meanwhile: rows with id 2900-2950 are deleted

-- OFFSET pagination: page 6 would be shifted (different rows than expected)
-- KEYSET pagination: continues from cursor position
WHERE (created_at, id) < ('2024-01-10', 3000)
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Simply returns the 20 rows after the cursor that STILL EXIST
-- No duplicates, no missing rows (except the deleted ones, which is correct)
```

---

# CHAPTER 23 — Advanced Pagination Patterns

---

## 23.1 Window Functions for Ranked Pagination

```sql
-- Top 3 products by revenue for each category:
SELECT category, name, revenue, rank_in_category
FROM (
    SELECT
        p.category,
        p.name,
        SUM(oi.quantity * oi.price_at_purchase) AS revenue,
        RANK() OVER (
            PARTITION BY p.category
            ORDER BY SUM(oi.quantity * oi.price_at_purchase) DESC
        ) AS rank_in_category
    FROM products p
    JOIN order_items oi ON oi.product_id = p.id
    GROUP BY p.category, p.id, p.name
) ranked
WHERE rank_in_category <= 3
ORDER BY category, rank_in_category;

-- Window function reference:
-- ROW_NUMBER(): 1,2,3,4,5 — always unique, no ties
-- RANK():       1,2,2,4,5 — ties get same rank, gaps after ties
-- DENSE_RANK(): 1,2,2,3,4 — ties get same rank, no gaps
-- NTILE(n):     divides rows into n equal groups
-- LAG(col, n):  value from n rows before current row
-- LEAD(col, n): value from n rows after current row
```

---

## 23.2 COUNT(*) Estimation

For pagination UIs that show "~1,234,567 results":

```sql
-- SLOW: exact count requires seq scan of entire table
SELECT COUNT(*) FROM orders;  -- scans every row, takes seconds on large tables

-- FAST: use statistics estimate (instant, from cached metadata)
SELECT reltuples::BIGINT AS estimated_count
FROM pg_class
WHERE relname = 'orders';
-- Accurate to within 1-5% after regular ANALYZE; instant response

-- Better estimate (updated after each ANALYZE):
SELECT n_live_tup AS estimated_live_rows
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- For filtered counts (harder to estimate):
-- Use EXPLAIN's row estimate:
EXPLAIN SELECT COUNT(*) FROM orders WHERE status = 'pending';
-- Aggregate  (rows=150)  ← planner's estimate
-- If good statistics: this is usually within 20% of true count
```

---

## 23.3 Deferred Join for Large Offsets

When OFFSET pagination is unavoidable (admin UIs, exports), this technique
dramatically improves performance by minimizing columns in the inner scan:

```sql
-- SLOW: fetches all columns for every row up to OFFSET
SELECT id, customer_id, total_amount, created_at, status, notes, shipping_address
FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 50000;
-- → Sorts 50,020 full rows (wide) → discards 50,000

-- FAST: deferred join — sort/offset on tiny ID-only subquery
SELECT o.*
FROM (
    SELECT id
    FROM orders
    ORDER BY created_at DESC
    LIMIT 20 OFFSET 50000
) ids
JOIN orders o ON o.id = ids.id
ORDER BY o.created_at DESC;

-- Inner query: sorts only id column (8 bytes) → much smaller, faster sort
-- Outer join: fetches exactly 20 full rows by PK → 20 index lookups
-- Total heap reads for full data: 20 (vs 50,020)

-- With proper index:
CREATE INDEX ix_orders_created_id ON orders(created_at DESC) INCLUDE (id);
-- Inner query becomes Index Only Scan → no heap reads at all in inner query
```

---

# CHAPTER 24 — Real-World Pagination

---

## 24.1 API Design with Cursor Tokens

Expose keyset pagination through a clean API:

```python
# FastAPI example
from base64 import b64encode, b64decode
import json

def encode_cursor(created_at, id):
    data = {"created_at": created_at.isoformat(), "id": id}
    return b64encode(json.dumps(data).encode()).decode()

def decode_cursor(token):
    data = json.loads(b64decode(token.encode()).decode())
    return data["created_at"], data["id"]

@app.get("/api/orders")
def list_orders(cursor: str = None, limit: int = 20):
    if cursor:
        created_at, last_id = decode_cursor(cursor)
        rows = db.execute("""
            SELECT id, total_amount, created_at FROM orders
            WHERE (created_at, id) < (%s, %s)
            ORDER BY created_at DESC, id DESC
            LIMIT %s
        """, (created_at, last_id, limit + 1))
    else:
        rows = db.execute("""
            SELECT id, total_amount, created_at FROM orders
            ORDER BY created_at DESC, id DESC
            LIMIT %s
        """, (limit + 1,))
    
    rows = list(rows)
    has_next = len(rows) > limit
    if has_next:
        rows = rows[:limit]
    
    next_cursor = None
    if has_next and rows:
        last = rows[-1]
        next_cursor = encode_cursor(last["created_at"], last["id"])
    
    return {
        "data": rows,
        "next_cursor": next_cursor,
        "has_next": has_next
    }
```

**API response:**
```json
{
    "data": [
        {"id": 5432, "total_amount": 299.99, "created_at": "2024-03-15T10:30:00Z"},
        ...
    ],
    "next_cursor": "eyJjcmVhdGVkX2F0IjogIjIwMjQtMDMtMTRUMDg6MDA6MDBaIiwgImlkIjogNTQxMn0=",
    "has_next": true
}
```

---

## 24.2 Cursor-Based Streaming for Exports

For exporting millions of rows without loading all in memory:

```python
# psycopg2 server-side cursor (streams from PostgreSQL)
with conn.cursor(name='export_cursor') as cur:  # named = server-side
    cur.itersize = 1000  # fetch 1000 rows per network round-trip
    cur.execute("""
        SELECT id, customer_id, total_amount, created_at
        FROM orders
        WHERE created_at >= %s
        ORDER BY created_at, id
    """, (start_date,))
    
    writer = csv.writer(output_file)
    writer.writerow(['id', 'customer_id', 'total_amount', 'created_at'])
    
    for row in cur:  # fetches 1000 rows at a time, processes one at a time
        writer.writerow(row)
    
# Memory used: ~1000 rows at a time, regardless of total export size
# Works for 1 billion rows
```

---

# PART VII — MAINTENANCE & OBSERVABILITY

---

# CHAPTER 25 — VACUUM & Autovacuum

---

## 25.1 WHY VACUUM is Not Optional

Every UPDATE and DELETE in PostgreSQL leaves dead tuples.
Without VACUUM:
1. Tables grow indefinitely (dead tuples are never reclaimed)
2. Sequential scans get slower (must read dead tuples to skip them)
3. Indexes bloat (dead index entries remain)
4. XID wraparound → data loss (most critical: covered in Chapter 13)

VACUUM is not maintenance. It is a fundamental requirement of MVCC.

---

## 25.2 HOW VACUUM Works Internally

```
Before VACUUM: page contents
┌────────────────────────────────────────────────────┐
│ [ALIVE: xmin=100, xmax=0] [DEAD: xmin=90, xmax=95]│
│ [DEAD: xmin=88, xmax=91]  [ALIVE: xmin=102, xmax=0]│
│ [FREE SPACE]                                        │
└────────────────────────────────────────────────────┘

VACUUM process:
1. Scan each page
2. For each tuple: is it dead? (xmax is committed and visible to all transactions)
3. Mark dead tuples' space as reusable
4. Update the Free Space Map (FSM)
5. If ALL tuples on a page are dead: mark in Visibility Map
6. Remove dead index entries for dead tuples
7. Freeze old tuples (if xmin is old enough)

After VACUUM: same page
┌────────────────────────────────────────────────────┐
│ [ALIVE: xmin=100, xmax=0] [FREE]                   │
│ [FREE]                    [ALIVE: xmin=102, xmax=0]│
│ [FREE SPACE — MORE NOW]                            │
└────────────────────────────────────────────────────┘
```

**Important:** Regular VACUUM does NOT return space to the OS.
The space is marked reusable WITHIN the table file.

To return space to the OS: `VACUUM FULL` (rewrites the entire table — very slow, locks table).

---

## 25.3 Autovacuum — The Automatic VACUUM Daemon

PostgreSQL runs autovacuum automatically in the background.
It triggers VACUUM (and ANALYZE) when a table accumulates enough dead tuples.

```sql
-- Autovacuum trigger thresholds (from postgresql.conf):
autovacuum_vacuum_threshold = 50          -- minimum dead tuples before trigger
autovacuum_vacuum_scale_factor = 0.2      -- + 20% of table size
-- Total trigger: 50 + 0.20 × n_live_tuples

-- For 1,000,000 row table:
-- Triggers when: dead_tuples > 50 + 0.20 × 1,000,000 = 200,050 dead tuples
-- This means 200,000 updates can happen before vacuum runs — a lot!

-- Tune for high-traffic tables (more frequent vacuum):
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- trigger at 1% dead tuples
    autovacuum_vacuum_threshold = 100,
    autovacuum_analyze_scale_factor = 0.005
);

-- Check autovacuum activity:
SELECT relname, last_autovacuum, last_autoanalyze,
       n_dead_tup, n_live_tup,
       ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Check if autovacuum is running right now:
SELECT pid, datname, relation::regclass, phase, heap_blks_scanned, heap_blks_vacuumed
FROM pg_stat_progress_vacuum;
```

---

## 25.4 VACUUM FULL — When and Why to Avoid It

```sql
-- Regular VACUUM: marks space reusable, no table lock, fast
VACUUM orders;

-- VACUUM FULL: rewrites entire table to disk, returns space to OS
VACUUM FULL orders;
-- ⚠️ Takes ACCESS EXCLUSIVE lock → blocks ALL queries!
-- ⚠️ Requires 2× disk space during rewrite
-- ⚠️ Takes minutes to hours for large tables
-- ⚠️ ALL indexes are rebuilt

-- Use VACUUM FULL only when:
-- 1. Table has grown massively and you need to recover disk space
-- 2. During a maintenance window with zero traffic

-- BETTER alternative: pg_repack (no exclusive lock, online operation)
-- pg_repack builds new table copy while allowing reads/writes,
-- then swaps atomically in < 1 second lock window
```

---

## Chapter 25 Summary

| Operation | Lock | Returns Space to OS | Use When |
|-----------|------|-------------------|---------|
| `VACUUM` | SHARE UPDATE EXCLUSIVE (allows DML) | No | Regular maintenance; autovacuum handles this |
| `VACUUM ANALYZE` | Same | No | After bulk loads |
| `VACUUM FREEZE` | Same | No | XID age approaching danger threshold |
| `VACUUM FULL` | ACCESS EXCLUSIVE (blocks everything) | Yes | Maintenance window; massive table bloat |
| `REINDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | N/A | Index bloat after heavy updates |

---

# CHAPTER 26 — Statistics & ANALYZE

---

## 26.1 HOW ANALYZE Works

ANALYZE reads a random sample of rows from each table and builds statistics.

```sql
ANALYZE products;
-- Samples ~300 × statistics_target rows (default: 30,000 rows)
-- Computes for each column:
--   - most_common_vals and their frequencies
--   - histogram bounds (equal-frequency buckets)
--   - n_distinct (count of unique values)
--   - correlation (physical ordering)
-- Stores in pg_statistic

-- Cost: reads the sample (usually < 1 second for most tables)
-- Does NOT read the entire table
-- Safe to run on production at any time
```

---

## 26.2 Statistics Target

The statistics target controls sample size and histogram granularity:

```sql
-- Default target: 100 (100 most_common_vals, 100 histogram buckets)
SHOW default_statistics_target;  -- 100

-- Increase for a specific column (useful for skewed or high-cardinality columns):
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
-- Now: 500 buckets for status column
-- Planner estimates "WHERE status = 'pending'" more accurately

-- Set globally in postgresql.conf (affects all new tables):
default_statistics_target = 200

-- When to increase statistics target:
-- 1. Column has many distinct values (high cardinality)
-- 2. Planner estimates are consistently wrong for that column
-- 3. EXPLAIN shows large actual/estimated row discrepancy

-- Cost of higher target: more memory in pg_statistic, slower ANALYZE
-- Usually worth it for key columns in important queries
```

---

## 26.3 Extended Statistics

When two columns are correlated, the planner's independence assumption
causes underestimates:

```sql
-- Example: product table with category and subcategory
-- All 'Electronics' products have subcategory in ['Phone','Laptop','TV']
-- No 'Electronics' products have subcategory 'Chair'

-- Query: WHERE category='Electronics' AND subcategory='Phone'
-- Planner (without extended stats):
--   P(category='Electronics') × P(subcategory='Phone') = 0.3 × 0.1 = 0.03 (3%)
-- Reality: subcategory='Phone' only appears within Electronics → much higher

-- Create extended statistics:
CREATE STATISTICS stat_category_subcategory (dependencies)
  ON category, subcategory
  FROM products;
ANALYZE products;
-- Planner now knows: P(subcategory='Phone' | category='Electronics') ≈ 0.5

-- Types of extended statistics:
-- dependencies: functional dependency detection
-- ndistinct: multi-column distinct value counts
-- mcv: most common value combinations
CREATE STATISTICS stat_multi (dependencies, ndistinct, mcv)
  ON category, subcategory, price_tier
  FROM products;
```

---

# CHAPTER 27 — Monitoring Queries

---

## 27.1 pg_stat_statements — Find Slow Queries

```sql
-- Enable (requires adding to postgresql.conf and restart):
-- shared_preload_libraries = 'pg_stat_statements'

-- Install the extension:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 slowest queries by average execution time:
SELECT
    LEFT(query, 100) AS query,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS avg_ms,
    ROUND(total_exec_time::numeric / 1000, 2) AS total_seconds,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows / calls AS avg_rows,
    ROUND(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS cache_hit_pct
FROM pg_stat_statements
WHERE calls > 10  -- ignore one-off queries
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Top queries by total time (highest impact to fix):
SELECT query, total_exec_time, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries with poor cache hit ratio (reading too much from disk):
SELECT query, calls,
    shared_blks_hit, shared_blks_read,
    ROUND(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS cache_hit_pct
FROM pg_stat_statements
WHERE shared_blks_hit + shared_blks_read > 1000
ORDER BY cache_hit_pct ASC
LIMIT 10;

-- Reset statistics (after fixing queries, start fresh measurement):
SELECT pg_stat_statements_reset();
```

---

## 27.2 pg_stat_activity — Real-Time Query Monitoring

```sql
-- All non-idle connections with query duration:
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    NOW() - query_start AS duration,
    LEFT(query, 120) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND pid != pg_backend_pid()  -- exclude yourself
ORDER BY duration DESC;

-- Long-running queries (over 30 seconds):
SELECT pid, NOW() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
  AND NOW() - query_start > INTERVAL '30 seconds'
ORDER BY duration DESC;

-- Queries waiting for locks:
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';

-- Cancel a specific query (sends SIGINT — graceful):
SELECT pg_cancel_backend(12345);  -- pid from above

-- Terminate a connection (sends SIGTERM — forceful):
SELECT pg_terminate_backend(12345);
```

---

## 27.3 Slow Query Log

```
# postgresql.conf settings for slow query logging:
log_min_duration_statement = 1000    # log queries > 1000ms (1 second)
log_duration = off                    # don't log ALL query durations (too verbose)
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_destination = 'csvlog'
logging_collector = on
log_directory = 'pg_log'
```

---

## 27.4 auto_explain — Automatic EXPLAIN for Slow Queries

Instead of manually running EXPLAIN on suspected queries,
auto_explain logs the plan automatically for any query exceeding a threshold:

```sql
-- Add to postgresql.conf (requires restart):
shared_preload_libraries = 'pg_stat_statements,auto_explain'
auto_explain.log_min_duration = 1000   -- explain queries > 1 second
auto_explain.log_analyze = on          -- include actual rows and timing
auto_explain.log_buffers = on          -- include buffer statistics
auto_explain.log_format = text         -- or json

-- Session-level (no restart needed, for investigation):
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 100;  -- explain queries > 100ms
SET auto_explain.log_analyze = on;

-- Run your queries normally; check server log for plans
-- /var/log/postgresql/postgresql-2024-01-15_0000.log
```

---

# CHAPTER 28 — Capstone Practice

---

## 28.1 Diagnose and Fix a Slow Query — Complete Walkthrough

**Scenario:** Your `/api/products` endpoint takes 3+ seconds. Here is the exact process.

**Step 1: Find the query**
```sql
SELECT LEFT(query, 100), mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 5;
-- Found: SELECT ... FROM products WHERE category_id=$1 ORDER BY price LIMIT 20 OFFSET $2
-- avg: 3200ms, calls: 10000
```

**Step 2: Run EXPLAIN ANALYZE**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT id, name, price, stock_quantity
FROM products
WHERE category_id = 5
ORDER BY price
LIMIT 20 OFFSET 200;

-- Output:
Limit  (actual rows=20 loops=1)
  Buffers: shared hit=50 read=9800  ← reading 9800 pages from disk!
  →  Sort  (actual rows=220 loops=1)
       Sort Key: price
       Sort Method: external merge  Disk: 4096kB  ← SPILLING TO DISK
       →  Seq Scan on products  (actual rows=80000 loops=1)  ← full scan!
            Filter: (category_id = 5)
            Rows Removed by Filter: 920000  ← throwing away 920,000 rows!
```

**Step 3: Identify problems**
1. `Rows Removed by Filter: 920000` → missing index on `category_id`
2. `Sort Method: external merge Disk` → work_mem too low for this sort
3. `OFFSET 200` → reading 220 rows to return 20 (keyset would be better)

**Step 4: Fix — Add index**
```sql
CREATE INDEX CONCURRENTLY ix_products_category_price ON products(category_id, price);
-- Index on (category_id, price) covers both the filter AND the ORDER BY!
```

**Step 5: Verify fix**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, price, stock_quantity
FROM products
WHERE category_id = 5
ORDER BY price
LIMIT 20 OFFSET 200;

-- New output:
Limit  (actual rows=20 loops=1)
  Buffers: shared hit=8 read=0  ← only 8 pages! (was 9850)
  →  Index Scan using ix_products_category_price on products
       Index Cond: (category_id = 5)
       -- No Sort node! Index already provides price order.
       -- OFFSET 200: scans 220 index entries total — still some overhead
```

**Step 6: Further optimize with keyset (optional)**
```sql
-- For API: switch to keyset pagination
-- Page 1:
SELECT id, name, price, stock_quantity FROM products
WHERE category_id = 5
ORDER BY price ASC, id ASC
LIMIT 20;
-- Last row: price=45.99, id=1234

-- Page 2 (keyset):
SELECT id, name, price, stock_quantity FROM products
WHERE category_id = 5
  AND (price, id) > (45.99, 1234)
ORDER BY price ASC, id ASC
LIMIT 20;
-- Index Scan: reads exactly 20 rows. 0.5ms!
```

**Result:** 3200ms → 2ms. 1600× improvement.

---

## 28.2 Design a Schema Under Load

**Requirements:** 10M customers, 500M orders, 50M products, 10,000 writes/second.

```sql
-- Use BIGSERIAL (8-byte) not SERIAL (4-byte): prevents overflow at 2B rows
CREATE TABLE customers (
    id            BIGSERIAL PRIMARY KEY,
    email         VARCHAR(255) NOT NULL UNIQUE,
    full_name     VARCHAR(255) NOT NULL,
    phone         VARCHAR(20),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ  -- soft delete
);

-- Critical indexes for customers:
CREATE UNIQUE INDEX ix_customers_email ON customers(email)
  WHERE deleted_at IS NULL;  -- partial: only active customers
CREATE INDEX ix_customers_created ON customers(created_at DESC);

-- Orders: partition by time for manageability
CREATE TABLE orders (
    id            BIGSERIAL,
    customer_id   BIGINT NOT NULL,
    total_amount  NUMERIC(12,2) NOT NULL,
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Monthly partitions: queries for recent orders only hit one or two partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE orders_2024_02 PARTITION OF orders
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... create new partition each month

-- Indexes on each partition:
CREATE INDEX ON orders_2024_01(customer_id, created_at DESC);
CREATE INDEX ON orders_2024_01(status, created_at DESC)
  WHERE status IN ('pending', 'processing');  -- partial for hot statuses
```

---

## 28.3 Reproduce, Observe, and Fix a Deadlock

**Step 1: Create the scenario**
```sql
-- Setup:
CREATE TABLE accounts (id INT PRIMARY KEY, balance NUMERIC(12,2));
INSERT INTO accounts VALUES (1, 1000), (2, 1000);
```

**Step 2: Reproduce in two terminals**
```sql
-- Terminal 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SELECT pg_sleep(2);  -- pause to let Terminal 2 run
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- WILL DEADLOCK
COMMIT;

-- Terminal 2 (run immediately after Terminal 1's first UPDATE):
BEGIN;
UPDATE accounts SET balance = balance - 200 WHERE id = 2;
UPDATE accounts SET balance = balance + 200 WHERE id = 1;  -- WILL DEADLOCK
COMMIT;
```

**Step 3: Observe**
```sql
-- While deadlock is happening:
SELECT pid, wait_event_type, wait_event, LEFT(query,60) AS query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';

-- After ~1 second (deadlock_timeout):
-- ERROR:  deadlock detected
-- DETAIL: Process 12345 waits for ShareLock on transaction 67890
```

**Step 4: Fix**
```sql
-- Always lock in consistent order (lowest ID first):
-- Both Terminal 1 and Terminal 2 use this pattern:
BEGIN;
-- Lock all rows at once, in order:
SELECT balance FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Terminal 1: id=1 → id=2  (always this order, no circular wait possible)
-- Terminal 2: id=1 → id=2  (same order → T2 simply waits for T1 to finish id=1)
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

---

## 28.4 Benchmark OFFSET vs Keyset Pagination

```sql
-- Setup: 1 million orders
INSERT INTO orders (customer_id, total_amount, status, created_at)
SELECT
    (random() * 100000 + 1)::BIGINT,
    ROUND((random() * 1000)::NUMERIC, 2),
    CASE (random() * 3)::INT WHEN 0 THEN 'pending' WHEN 1 THEN 'completed' ELSE 'cancelled' END,
    NOW() - (random() * 365 * 3)::INT * INTERVAL '1 day'
FROM generate_series(1, 1000000);

ANALYZE orders;

CREATE INDEX ix_orders_bench ON orders(created_at DESC, id DESC);

-- Benchmark OFFSET (page 1, 100, 1000, 5000):
\timing on

-- Page 1:
SELECT id, total_amount FROM orders ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 0;
-- Page 100 (offset 2000):
SELECT id, total_amount FROM orders ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 2000;
-- Page 1000 (offset 20000):
SELECT id, total_amount FROM orders ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 20000;
-- Page 5000 (offset 100000):
SELECT id, total_amount FROM orders ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 100000;

-- Expected results (approximate):
-- Page 1:    ~1ms
-- Page 100:  ~5ms
-- Page 1000: ~50ms
-- Page 5000: ~300ms

-- Benchmark Keyset (equivalent pages):
-- First: get cursor for page 100 position:
SELECT created_at, id FROM orders ORDER BY created_at DESC, id DESC
LIMIT 1 OFFSET 1999;
-- Returns: created_at='2023-08-15 10:30:00', id=345678

-- Keyset page 100 (equivalent):
SELECT id, total_amount FROM orders
WHERE (created_at, id) < ('2023-08-15 10:30:00', 345678)
ORDER BY created_at DESC, id DESC LIMIT 20;
-- Expected: ~1ms (same as page 1!)
```

**Expected benchmark results:**
```
OFFSET pages:
  Page 1:    1ms
  Page 100:  5ms
  Page 1000: 50ms
  Page 5000: 500ms  ← 500× slower than page 1

Keyset pages:
  Page 1:    1ms
  Page 100:  1ms  (same!)
  Page 1000: 1ms  (same!)
  Page 5000: 1ms  (same!)
```

---

## 28.5 Production Readiness Checklist

```sql
-- ══════════════════════════════════════════════
-- CONFIGURATION (postgresql.conf)
-- ══════════════════════════════════════════════
shared_buffers = 25% of RAM
effective_cache_size = 75% of RAM
work_mem = 16-64MB (monitor for spills)
random_page_cost = 1.1  -- for SSD
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_connections = 100  -- use PgBouncer for more
log_min_duration_statement = 1000
log_lock_waits = on
shared_preload_libraries = 'pg_stat_statements,auto_explain'

-- ══════════════════════════════════════════════
-- SCHEMA
-- ══════════════════════════════════════════════
-- 1. Use BIGSERIAL for all PKs (future-proof)
-- 2. Add indexes on all FK columns
-- 3. Add partial indexes for soft-delete patterns
-- 4. Normalize to 3NF, then denormalize only with evidence
-- 5. Use timestamptz (not timestamp) for all datetime columns

-- ══════════════════════════════════════════════
-- INDEXES
-- ══════════════════════════════════════════════
-- Check unused indexes weekly:
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexname NOT LIKE '%pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Check index hit ratio:
SELECT SUM(idx_blks_hit) / NULLIF(SUM(idx_blks_hit + idx_blks_read), 0) AS idx_hit_ratio
FROM pg_stathio_user_indexes;
-- Should be > 0.99

-- ══════════════════════════════════════════════
-- VACUUM / BLOAT
-- ══════════════════════════════════════════════
-- Check dead tuple accumulation:
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- Check XID age (alert at 1B, emergency at 1.5B):
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age DESC;

-- ══════════════════════════════════════════════
-- MONITORING
-- ══════════════════════════════════════════════
-- Slow queries:
SELECT LEFT(query,80), mean_exec_time, calls
FROM pg_stat_statements
WHERE mean_exec_time > 100  -- queries averaging over 100ms
ORDER BY total_exec_time DESC LIMIT 20;

-- Active long-running queries:
SELECT pid, NOW() - query_start AS dur, LEFT(query,80)
FROM pg_stat_activity
WHERE state = 'active' AND NOW() - query_start > INTERVAL '5 seconds'
ORDER BY dur DESC;

-- Lock contention:
SELECT COUNT(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock';
-- Alert if > 10 connections waiting for locks
```

---

## 28.6 Complete Reference Summary

```sql
-- INDEXING
CREATE INDEX CONCURRENTLY ix ON tbl(col);                  -- no lock during build
CREATE INDEX ON tbl(col1, col2) WHERE active = true;       -- composite + partial
CREATE INDEX ON tbl(LOWER(email));                         -- expression index
CREATE INDEX ON tbl USING GIN(jsonb_col);                  -- JSONB
CREATE INDEX ON tbl USING BRIN(created_at);                -- time-series

-- EXPLAIN
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
SET enable_seqscan = off; -- force index (test only)

-- TRANSACTIONS
BEGIN; SAVEPOINT sp1; ROLLBACK TO sp1; RELEASE sp1; COMMIT;
SET lock_timeout = '5s'; SET deadlock_timeout = '500ms';

-- LOCKING
SELECT ... FOR UPDATE NOWAIT;
SELECT ... FOR UPDATE SKIP LOCKED;
SELECT pg_advisory_xact_lock(key);

-- PAGINATION
-- Keyset: WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20
-- Deferred: SELECT t.* FROM (SELECT id FROM t ORDER BY x LIMIT 20 OFFSET N) ids JOIN t USING(id)
-- Estimate: SELECT reltuples::BIGINT FROM pg_class WHERE relname = 'orders';

-- MONITORING
SELECT query, mean_exec_time, calls FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
SELECT pid, NOW()-query_start, LEFT(query,80) FROM pg_stat_activity WHERE state='active' ORDER BY 2 DESC;
SELECT pid, pg_blocking_pids(pid) FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
SELECT pg_cancel_backend(pid); SELECT pg_terminate_backend(pid);

-- MAINTENANCE
VACUUM ANALYZE tbl;
VACUUM FREEZE tbl;              -- prevent XID wraparound
REINDEX INDEX CONCURRENTLY ix;
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age DESC;
SELECT relname, n_dead_tup FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;
```

---

*End of PostgreSQL Mastery Book — Chapters 1 through 28*

---

# PART VIII — DATA MODELS & QUERY LANGUAGES
> *From "Designing Data-Intensive Applications" — Chapter 2*

> Data models are the most important part of developing software,
> because they have such a profound effect on how we think about
> the problem we are solving. — Martin Kleppmann

---

# CHAPTER 29 — Relational Versus Document Databases Today

---

## 29.1 WHY This Comparison Matters

The NoSQL movement of the 2010s promised to replace relational databases.
A decade later, neither won. Both coexist, each dominating different use cases.

Understanding WHY they differ — not just WHAT they are — lets you make the
right architectural decision instead of following trends.

---

## 29.2 The Case FOR Document Databases

Document databases (MongoDB, CouchDB, Firestore) store data as self-contained
JSON-like documents. They excel when your data has a natural tree structure.

```
Example: a resume / LinkedIn profile

Document model — one document, all data:
{
  "user_id": 251,
  "full_name": "Rahul Kumar",
  "summary": "Senior Software Engineer with 8 years experience",
  "positions": [
    {
      "title": "Senior Engineer",
      "company": "Infosys",
      "start": "2020-01",
      "end": null
    },
    {
      "title": "Engineer",
      "company": "TCS",
      "start": "2017-01",
      "end": "2020-01"
    }
  ],
  "education": [
    {"institution": "IIT Bombay", "degree": "B.Tech", "year": 2017}
  ],
  "skills": ["Python", "PostgreSQL", "Kubernetes"]
}

Relational model — same data, multiple tables:
  users:      id, full_name, summary
  positions:  id, user_id, title, company, start_date, end_date
  education:  id, user_id, institution, degree, year
  user_skills: user_id, skill_name

To reconstruct one profile: users JOIN positions JOIN education JOIN user_skills
→ 4 table reads, 3 JOINs, application assembles the tree
```

**Document model advantages for this use case:**
1. **Locality** — one read, complete profile. No JOINs.
2. **Schema flexibility** — different users can have different position fields
3. **One-to-many fit** — positions/education are owned by one user (tree structure)

---

## 29.3 The Case FOR Relational Databases

Relational databases (PostgreSQL, MySQL) shine when data is interconnected —
when many entities relate to many other entities in complex ways.

```
Example: product orders in e-commerce

Order → Customer (many orders → one customer)
Order → Products (many orders → many products, via order_items)
Product → Categories (many products → many categories)
Customer → Addresses (one customer → many addresses)

If you store this as documents:
{
  "order_id": 1001,
  "customer": {"name": "Rahul", "email": "rahul@x.com", "phone": "9876543210"},
  "items": [
    {"product": "Widget", "sku": "W001", "category": "Electronics", "price": 99},
    {"product": "Cable",  "sku": "C001", "category": "Electronics", "price": 15}
  ]
}

Problem 1 — Duplication:
  "Electronics" category stored in EVERY product in EVERY order
  Customer email stored in EVERY order
  If Rahul changes email → update every order document

Problem 2 — Many-to-Many is painful:
  "Find all orders containing Electronics products" requires:
  → Scan every order document
  → Parse every item's category
  → No index possible without application-level workaround
```

**Relational advantages:**
1. **No duplication** — categories, customers stored once, referenced by ID
2. **Many-to-many** — natural via join tables
3. **Joins are fast** — purpose-built, indexed, optimized
4. **Consistency** — foreign keys prevent orphaned data

---

## 29.4 Relational vs Document: Head-to-Head Today

| Dimension | Relational (PostgreSQL) | Document (MongoDB) |
|-----------|------------------------|-------------------|
| Schema | Fixed at definition (schema-on-write) | Flexible (schema-on-read) |
| Joins | Native, optimized, indexed | Manual in app code or $lookup (slow) |
| Many-to-Many | Natural — join tables | Painful — duplication or references |
| One-to-Many trees | Requires joins | Natural — nested arrays |
| Locality | Multiple page reads for related data | One document = one read |
| ACID transactions | Full, battle-tested | Added later, limited |
| Query language | SQL (declarative, powerful) | Pipeline API (expressive but verbose) |
| Full-text search | GIN + tsvector (good) | Text indexes (good) |
| Geographic data | PostGIS (excellent) | 2dsphere indexes (good) |

**The modern answer:** PostgreSQL supports JSONB — you get relational AND document
in one system. Use normalized tables for interconnected data, JSONB for
flexible/hierarchical fields within those rows.

```sql
-- Best of both worlds in PostgreSQL:
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL,    -- normalized (queried, indexed)
    full_name   VARCHAR(255) NOT NULL,            -- normalized (searched)
    profile     JSONB                             -- flexible document (varies per user)
);

-- profile column stores variable schema data:
INSERT INTO users (email, full_name, profile) VALUES (
    'rahul@x.com',
    'Rahul Kumar',
    '{
        "positions": [{"title": "Senior Engineer", "company": "Infosys"}],
        "skills": ["Python", "PostgreSQL"],
        "certifications": ["AWS Solutions Architect"]
    }'
);

-- Query normalized fields fast (index):
SELECT id FROM users WHERE email = 'rahul@x.com';  -- uses unique index ✅

-- Query inside JSONB (GIN index):
CREATE INDEX ON users USING GIN(profile);
SELECT full_name FROM users WHERE profile @> '{"skills": ["PostgreSQL"]}';  -- ✅

-- Mix both:
SELECT u.full_name, u.profile->'positions'->0->>'company' AS current_company
FROM users u
WHERE u.email = 'rahul@x.com';
```

---

## 29.5 Schema-on-Write vs Schema-on-Read

**Schema-on-write (relational):** Schema is enforced when data is written.
Every row must conform. Migrations required to change structure.

```sql
-- Adding a field in relational: requires ALTER TABLE
ALTER TABLE users ADD COLUMN linkedin_url TEXT;
-- Runs on ALL existing rows → potentially slow on millions of rows
-- But: every future row is guaranteed to have the field
```

**Schema-on-read (document):** No schema enforcement on write.
Application interprets the structure when reading.

```javascript
// Adding a field in document: just start writing it
db.users.insertOne({ name: "Rahul", linkedin_url: "..." });
// Old documents don't have linkedin_url
// Application must handle: if (user.linkedin_url) { ... }
// No migration needed, but: inconsistent data across documents
```

**Which is better?**
Schema-on-write: when all records follow the same structure and you want guarantees.
Schema-on-read: when records naturally vary in structure (e.g., sensor data from heterogeneous devices).

---

# CHAPTER 30 — Query Languages for Data

---

## 30.1 WHY Declarative Queries Won

In the 1970s, before SQL, databases were queried imperatively.
You wrote code that specified exactly HOW to retrieve data:

```
IMS (hierarchical database, 1960s):
  Navigate pointer from parent record
  For each child record:
    If matches condition:
      Add to result list
  → YOU write the traversal. YOU manage the navigation. YOU optimize.
```

SQL introduced declarative queries:
```sql
SELECT name FROM products WHERE stock > 10;
-- You say WHAT you want. The database figures out HOW.
```

**Why declarative won:**
1. The database can optimize freely (index, reorder, parallelize) without changing your query
2. Your query stays valid even as the database engine improves
3. Far less code to write and maintain

---

## 30.2 Declarative Queries on the Web — CSS Analogy

CSS is a perfect analogy for why declarative beats imperative:

```css
/* Declarative CSS: "make all selected list items blue" */
li.selected { color: blue; }
/* Browser figures out which elements match and renders them */

/* Equivalent JavaScript imperative: */
var items = document.getElementsByTagName("li");
for (var i = 0; i < items.length; i++) {
    if (items[i].className === "selected") {
        items[i].style.color = "blue";
    }
}
```

The CSS version:
- Works even if the browser adds new optimization techniques
- Works even if you add/remove list items dynamically
- Is shorter and easier to read

SQL has the same property. `SELECT ... WHERE ... ORDER BY ...` works
regardless of whether PostgreSQL uses a seq scan, index scan, or
parallel workers under the hood.

---

## 30.3 MapReduce Querying

MapReduce is a programming model invented at Google for processing
huge datasets across thousands of machines. It was popularized by Hadoop.

### The Model

Every MapReduce job has exactly two user-defined functions:

```
map(key, value) → list of (intermediate_key, intermediate_value)
reduce(key, list of values) → list of output values
```

The framework handles: distributing data, running map on each chunk,
sorting intermediate results by key, running reduce on each group.

### A Real Example — Count Orders Per City

```python
# Dataset: millions of order records
# {customer_id, city, total_amount, product_ids, ...}

# MAP FUNCTION: called once per order record
def map(order_id, order_record):
    city = order_record['city']
    emit(city, 1)  # output: (city_name, count_1)

# Intermediate output (after map, before reduce):
# ("Mumbai",  1)
# ("Delhi",   1)
# ("Mumbai",  1)  ← different order, same city
# ("Chennai", 1)
# ("Mumbai",  1)
# ...

# Framework sorts all intermediate by key:
# ("Chennai", [1])
# ("Delhi",   [1])
# ("Mumbai",  [1, 1, 1])  ← all Mumbai values grouped together

# REDUCE FUNCTION: called once per unique key
def reduce(city, count_list):
    emit(city, sum(count_list))  # output: (city, total_count)

# Final output:
# ("Chennai", 1)
# ("Delhi",   1)
# ("Mumbai",  3)
```

SQL equivalent (much simpler):
```sql
SELECT city, COUNT(*) as order_count
FROM orders
GROUP BY city
ORDER BY order_count DESC;
```

### MapReduce in MongoDB

MongoDB had a MapReduce API (now deprecated in favor of the Aggregation Pipeline):

```javascript
// MongoDB MapReduce (legacy):
db.orders.mapReduce(
    function() { emit(this.city, 1); },           // map
    function(key, values) { return sum(values); }, // reduce
    { out: "order_counts" }
);

// Modern replacement: Aggregation Pipeline (declarative):
db.orders.aggregate([
    { $group: { _id: "$city", count: { $sum: 1 } } },
    { $sort: { count: -1 } }
]);
```

### Why MapReduce Lost (for most use cases)

```
MapReduce problems:
  1. Two-step only: complex analytics need many chained map-reduce jobs
  2. Not interactive: batch processing only (minutes to hours per job)
  3. Disk-heavy: each step writes intermediate results to disk
  4. Hard to debug: distributed execution is opaque

Modern replacements:
  → Apache Spark: keeps intermediate data in memory, 100× faster
  → BigQuery/Redshift: SQL on distributed storage
  → PostgreSQL with parallelism: for moderate scale
```

---

## 30.4 Graph-Like Data Models

When EVERYTHING is interconnected, graph databases are natural.

### When to Use a Graph

```
Relational excels when: many entities, each with many attributes, moderate interconnection
Graph excels when:      entities are simple, but connections are the data

Real-world graph use cases:
  Social networks:        Person --FOLLOWS--> Person
  Knowledge graphs:       City --CAPITAL_OF--> Country --LOCATED_IN--> Continent
  Recommendation:         User --BOUGHT--> Product --SIMILAR_TO--> Product
  Fraud detection:        Account --TRANSFERRED_TO--> Account (detect circular patterns)
  Network topology:       Router --CONNECTS_TO--> Router
  Org charts:             Employee --REPORTS_TO--> Manager
  Railway/road maps:      Station --ROUTE--> Station --DISTANCE--> 42km
```

---

## 30.5 The Property Graph Model

The most common graph model. Used by Neo4j, Amazon Neptune, TigerGraph.

**Two types of objects:**

**Nodes (vertices):** represent entities
```
Node example:
  Labels: [:Person]
  Properties: {name: "Rahul Kumar", born: 1990, city: "Mumbai"}

Node example:
  Labels: [:Company]
  Properties: {name: "Google", founded: 1998, country: "USA"}
```

**Edges (relationships):** represent connections between nodes
```
Edge example:
  Type: WORKS_AT
  From: Rahul Kumar (Person node)
  To:   Google (Company node)
  Properties: {since: 2020, role: "Senior Engineer"}

Edge example:
  Type: KNOWS
  From: Rahul Kumar
  To:   Priya Sharma
  Properties: {since: 2018, context: "college"}
```

**Key properties of Property Graphs:**
1. Every node can have any label(s) — flexible typing
2. Every edge has a direction (from → to) and a type
3. Both nodes and edges can have arbitrary key-value properties
4. Any node can be connected to any other node by any edge type
5. Edges can be traversed in either direction in queries

---

## 30.6 The Cypher Query Language

Cypher is Neo4j's declarative graph query language.
It uses ASCII art patterns to describe the graph structure you want to find.

```
Node syntax:  (variable:Label {property: value})
Edge syntax:  -[variable:TYPE {property: value}]->
Path syntax:  (a)-[:EDGE_TYPE]->(b)-[:EDGE_TYPE]->(c)
```

### Basic Cypher Examples

```cypher
-- Create nodes and relationships:
CREATE (rahul:Person {name: "Rahul Kumar", city: "Mumbai"})
CREATE (priya:Person {name: "Priya Sharma", city: "Delhi"})
CREATE (google:Company {name: "Google"})
CREATE (rahul)-[:WORKS_AT {since: 2020}]->(google)
CREATE (rahul)-[:KNOWS {since: 2018}]->(priya)

-- Find: who does Rahul know?
MATCH (rahul:Person {name: "Rahul Kumar"})-[:KNOWS]->(friend:Person)
RETURN friend.name, friend.city

-- Find: mutual friends between Rahul and Priya
MATCH (rahul:Person {name: "Rahul Kumar"})-[:KNOWS]->(mutual:Person)<-[:KNOWS]-(priya:Person {name: "Priya Sharma"})
RETURN mutual.name

-- Find: people who work at the same company as Rahul
MATCH (rahul:Person {name: "Rahul Kumar"})-[:WORKS_AT]->(company)<-[:WORKS_AT]-(colleague:Person)
WHERE colleague <> rahul
RETURN colleague.name, company.name

-- Find: path of up to 5 hops from Rahul to anyone named "Vikram"
MATCH path = (rahul:Person {name: "Rahul Kumar"})-[:KNOWS*1..5]->(vikram:Person {name: "Vikram"})
RETURN path, length(path) AS hops
ORDER BY hops

-- Aggregation: count connections per person
MATCH (p:Person)-[:KNOWS]->(friend)
RETURN p.name, COUNT(friend) AS friend_count
ORDER BY friend_count DESC
```

---

## 30.7 Graph Queries in SQL — Recursive CTEs

PostgreSQL can answer graph queries using **recursive CTEs** (Common Table Expressions).
This is powerful but verbose compared to Cypher. Good for moderate graph queries
without deploying a dedicated graph database.

```sql
-- Table structure for a graph in PostgreSQL:
CREATE TABLE persons (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    city VARCHAR(100)
);

CREATE TABLE knows (
    from_person_id BIGINT REFERENCES persons(id),
    to_person_id   BIGINT REFERENCES persons(id),
    since          DATE,
    PRIMARY KEY (from_person_id, to_person_id)
);

-- Query 1: Direct friends of Rahul
SELECT p.name
FROM persons rahul
JOIN knows k ON k.from_person_id = rahul.id
JOIN persons p ON p.id = k.to_person_id
WHERE rahul.name = 'Rahul Kumar';

-- Query 2: Friends of friends (2 hops)
WITH friends AS (
    SELECT k.to_person_id AS friend_id
    FROM persons rahul
    JOIN knows k ON k.from_person_id = rahul.id
    WHERE rahul.name = 'Rahul Kumar'
)
SELECT DISTINCT p.name
FROM friends f
JOIN knows k ON k.from_person_id = f.friend_id
JOIN persons p ON p.id = k.to_person_id;

-- Query 3: ALL reachable people from Rahul (any number of hops)
-- This is where recursive CTEs shine:
WITH RECURSIVE reachable AS (
    -- Base case: start with Rahul himself
    SELECT id, name, 0 AS hops
    FROM persons
    WHERE name = 'Rahul Kumar'

    UNION ALL

    -- Recursive case: extend the frontier by one hop
    SELECT p.id, p.name, r.hops + 1
    FROM reachable r
    JOIN knows k ON k.from_person_id = r.id
    JOIN persons p ON p.id = k.to_person_id
    WHERE r.hops < 6           -- limit depth (prevent infinite loops)
      AND p.id NOT IN (SELECT id FROM reachable)  -- avoid cycles
)
SELECT name, MIN(hops) AS min_hops
FROM reachable
WHERE name != 'Rahul Kumar'
GROUP BY name
ORDER BY min_hops, name;

-- Query 4: Shortest path between Rahul and a specific person
WITH RECURSIVE paths AS (
    SELECT
        ARRAY[rahul.id] AS path,
        rahul.id AS current_id,
        0 AS depth
    FROM persons rahul
    WHERE rahul.name = 'Rahul Kumar'

    UNION ALL

    SELECT
        p.path || k.to_person_id,
        k.to_person_id,
        p.depth + 1
    FROM paths p
    JOIN knows k ON k.from_person_id = p.current_id
    WHERE p.depth < 10
      AND NOT k.to_person_id = ANY(p.path)  -- no cycles
)
SELECT array_to_string(
    ARRAY(SELECT name FROM persons WHERE id = ANY(path) ORDER BY array_position(path, id)),
    ' → '
) AS path_string,
depth AS hops
FROM paths
WHERE current_id = (SELECT id FROM persons WHERE name = 'Vikram')
ORDER BY hops
LIMIT 1;
```

**Performance note:** For deep graph traversals (6+ hops) on large graphs
(millions of nodes), a dedicated graph database (Neo4j) is significantly
faster because it stores adjacency lists physically on disk,
enabling constant-time neighbor lookups regardless of graph size.

---

## 30.8 Triple-Stores and SPARQL

### What Triple-Stores Are

Triple-stores store data as **(subject, predicate, object)** triples.
This is the data model of the **Semantic Web** and **Resource Description Framework (RDF)**.

```
Every fact is a triple:
  (Rahul Kumar, worksAt, Google)
  (Rahul Kumar, livesIn, Mumbai)
  (Google, locatedIn, USA)
  (Mumbai, partOf, Maharashtra)
  (Maharashtra, partOf, India)
  (India, continent, Asia)

Same data, triple notation:
  subject         predicate     object
  ──────────────  ────────────  ──────────────
  rahul:Person    works_at      google:Company
  rahul:Person    lives_in      mumbai:City
  google:Company  located_in    usa:Country
  mumbai:City     part_of       maharashtra:State
```

Notice: objects can themselves be subjects in other triples.
This creates a graph where every fact is a directed edge.

### SPARQL — Query Language for Triple-Stores

```sparql
PREFIX ex: <http://example.org/>

# Find all people who work at Google:
SELECT ?person ?city
WHERE {
    ?person ex:works_at ex:Google .
    ?person ex:lives_in ?city .
}

# Find the continent of the city where Rahul lives:
SELECT ?continent
WHERE {
    ex:rahul ex:lives_in ?city .
    ?city ex:part_of ?state .
    ?state ex:part_of ?country .
    ?country ex:continent ?continent .
}
# Traverses 3 hops in one query

# SPARQL can do optional matches (like LEFT JOIN):
SELECT ?person ?phone
WHERE {
    ?person ex:works_at ex:Google .
    OPTIONAL { ?person ex:phone ?phone }
}
```

### Triple-Stores vs Property Graphs

| Feature | Triple-Store (RDF) | Property Graph (Neo4j) |
|---------|-------------------|----------------------|
| Edge properties | Awkward (need reification) | First-class |
| Schema | Very flexible (open world) | Flexible |
| Standards | W3C standards (RDF, SPARQL, OWL) | Vendor-specific |
| Reasoning | Yes (OWL, RDFS inference) | Limited |
| Use case | Linked data, knowledge graphs | Social networks, recommendations |

---

## 30.9 Datalog — The Foundation

Datalog is a query language from the 1980s that influenced both Prolog
and modern database query systems. It is used in data lineage systems,
program analysis, and Datomic database.

Datalog defines data as **facts** and **rules**:

```prolog
% Facts: raw data
works_at(rahul, google).
works_at(priya, microsoft).
works_at(vikram, google).
located_in(google, usa).
located_in(microsoft, usa).
country_of(usa, north_america).

% Rules: derived facts
% "Two people are colleagues if they work at the same company"
colleagues(X, Y) :- works_at(X, C), works_at(Y, C), X \= Y.

% "A person lives on a continent if they work at a company located in a country on that continent"
lives_on(Person, Continent) :-
    works_at(Person, Company),
    located_in(Company, Country),
    country_of(Country, Continent).

% Query: who are Rahul's colleagues?
?- colleagues(rahul, Who).
% Output: Who = vikram  (both work at google)

% Query: what continent does Rahul live on?
?- lives_on(rahul, Continent).
% Output: Continent = north_america
```

**Why Datalog matters today:**
- Datomic (Clojure database) uses Datalog for queries
- Cascalog (Hadoop processing framework) is based on Datalog
- Program analysis tools (Soufflé, LogiQL) use Datalog
- It proves that rules + facts can express recursive graph queries that SQL CAN express (via recursive CTEs) but SQL was not originally designed for

---

# PART IX — STORAGE AND RETRIEVAL (DEEP DIVE)
> *From "Designing Data-Intensive Applications" — Chapter 3*

---

# CHAPTER 31 — Data Structures That Power Databases

---

## 31.1 WHY Storage Structures Matter

The fastest query is one that reads the fewest bytes from the slowest medium (disk).
The storage structure determines:
- How much data must be read for a lookup
- How much data must be written for an insert
- Whether reads or writes are fast (the fundamental trade-off)

Every database makes a choice about this trade-off.
Understanding the choices lets you pick the right database for your workload.

---

## 31.2 The Simplest Possible Database

```bash
#!/bin/bash
# World's simplest database: two shell functions

db_set() {
    echo "$1,$2" >> database.txt    # append key-value pair to file
}

db_get() {
    grep "^$1," database.txt | tail -n 1 | cut -d',' -f2  # find last entry for key
}

# Usage:
db_set rahul "Mumbai"
db_set priya "Delhi"
db_set rahul "Bengaluru"  # update: just append again

db_get rahul  # returns "Bengaluru" (tail -n 1 gets the latest)
db_get priya  # returns "Delhi"
```

**Performance analysis:**
- `db_set`: O(1) — always just appends. Very fast write.
- `db_get`: O(n) — must scan entire file for the key. Terrible for large files.
- Delete: not supported (would need to append a "tombstone" marker)

**The key insight:** Append-only writes are extremely fast (sequential I/O).
The challenge is making reads fast too. This is what indexes solve.

---

## 31.3 Hash Indexes — Internals

### What a Hash Index Really Is

A hash index is exactly like a programming language's hash map (dict in Python,
Map in JavaScript) — but persisted to disk and kept consistent with the data file.

```
In-memory hash table:
  key         → byte_offset_in_file
  ──────────────────────────────────
  "rahul"     → 0
  "priya"     → 20
  "vikram"    → 45
  "rahul"     → 78   ← updated: rahul moved to offset 78

db_get("rahul"):
  1. Look up "rahul" in hash table → byte offset 78
  2. Seek to position 78 in file → read value
  Total: 1 hash lookup + 1 disk seek
```

### Log-Structured Storage with Hash Index (Bitcask Model)

Bitcask (Riak's storage engine) is a real database that uses this model:

```
Data file (append-only):
  [key=rahul, val=Mumbai]     ← offset 0
  [key=priya, val=Delhi]      ← offset 20
  [key=rahul, val=Bengaluru]  ← offset 45  (update appended)

In-memory hash index:
  rahul → 45  (points to LATEST value, offset 45)
  priya → 20

Lookup: rahul
  → hash_index["rahul"] = 45
  → seek(file, 45), read → "Bengaluru"
  Total: O(1) — no scanning!
```

### Segmentation and Compaction

Appending forever makes the file grow without bound.
**Compaction** merges segments, keeping only the latest value per key:

```
Segment 1 (old):          Segment 2 (old):
  rahul → Mumbai            vikram → Chennai
  priya → Delhi             rahul  → Bengaluru  ← latest rahul
  vikram → Mumbai

After compaction (Merged Segment):
  rahul  → Bengaluru   ← latest only
  priya  → Delhi
  vikram → Chennai

Old segments are deleted after merge completes.
Compaction runs in background thread → reads continue on old segments during compaction.
```

### Hash Index Limitations

```
1. All keys must fit in RAM
   → 100M unique keys × 100 bytes each = 10GB RAM for hash table alone
   → Not practical for large key spaces

2. Range queries are impossible
   → WHERE name >= 'P' AND name <= 'R' requires scanning all keys
   → Hash function destroys ordering

3. Hash collisions require handling (chaining or open addressing)

4. Hash index survives crashes only if explicitly saved to disk
   → Bitcask keeps a "hint file" per segment for fast startup

5. Only one file → single-threaded writes (one writer at a time)
```

---

## 31.4 SSTables and LSM-Trees

### SSTables — Sorted String Tables

An SSTable is like the compacted segment above, but with one crucial difference:
**keys are sorted alphabetically**.

```
SSTable (sorted by key):
  ┌─────────────────────────────────────────────────────┐
  │  Key       │ Value                                   │
  │ ────────── │ ─────────────────────────────────────── │
  │  "apple"   │ {"count": 5, "category": "fruit"}       │
  │  "banana"  │ {"count": 12, "category": "fruit"}      │
  │  "cherry"  │ {"count": 3, "category": "fruit"}       │
  │  "date"    │ {"count": 7, "category": "fruit"}       │
  │  "elderb." │ {"count": 1, "category": "fruit"}       │
  └─────────────────────────────────────────────────────┘

Benefits of sorted order:
  1. Merging SSTables is efficient (like merge sort — merge two sorted lists)
  2. Sparse in-memory index works: store every 100th key
     → To find "banana": index says "apple"=offset_0, "cherry"=offset_500
     → Scan from offset_0 to find "banana" — at most 500 bytes
  3. Range queries are sequential reads: WHERE key >= 'b' AND key <= 'c'
     → Seek to 'b' position, scan forward until 'c' — all sequential!
```

### LSM-Trees — Log-Structured Merge Trees

LSM-Trees (used by Cassandra, LevelDB, RocksDB, and PostgreSQL's BRIN-like structures)
combine the fast writes of an append-only log with the efficient reads of SSTables.

```
LSM-Tree Architecture:

WRITES:
  Application → Write to Memtable (in-memory sorted tree, e.g. Red-Black tree)
                                                    ↓
                When Memtable reaches threshold (~4MB):
                  → Flush to disk as new SSTable (Level 0)
                  → New Memtable starts
                                                    ↓
                Background compaction:
                  Level 0 SSTables → merge → Level 1 (larger SSTables)
                  Level 1 SSTables → merge → Level 2 (even larger)
                  ...continues in levels

READS:
  1. Check Memtable (in-memory) — most recent writes
  2. Check Level 0 SSTables (newest to oldest) — with Bloom filter
  3. Check Level 1 SSTables
  4. ... deeper levels ...

Bloom filter: probabilistic data structure
  → "Is key X definitely NOT in this SSTable?" → yes/no (no false negatives)
  → Avoids reading SSTables that definitely don't contain the key
  → Reduces unnecessary disk reads by ~99%
```

**Visual LSM-Tree flow:**

```
                ┌────────────────────────┐
  WRITE ──────► │ Memtable (in-memory)   │  ← fast, sorted, ~4MB
                │ [apple, banana, cherry] │
                └──────────┬─────────────┘
                           │ flush (when full)
                           ▼
                ┌────────────────────────┐
  Level 0:      │ SSTable 1 (oldest→new) │  ← may overlap in key range
                │ SSTable 2              │
                │ SSTable 3              │
                └──────────┬─────────────┘
                           │ compaction (merge + sort)
                           ▼
                ┌────────────────────────┐
  Level 1:      │ SSTable A (10MB each)  │  ← no key overlap within level
                │ SSTable B              │
                └──────────┬─────────────┘
                           │ compaction
                           ▼
                ┌────────────────────────┐
  Level 2:      │ SSTable X (100MB each) │
                └────────────────────────┘
```

---

## 31.5 Comparing B-Trees and LSM-Trees

This is one of the most important architectural decisions in database storage:

### Write Amplification

**Write amplification:** how many times data is written to disk per one logical write.

```
B-Tree write amplification:
  User writes 1 row:
  → WAL write (1 write)
  → Data page write (1 write, possibly triggers page split = 2-3 writes)
  → Index updates (1 write per index)
  Write amplification: ~3-10×

LSM-Tree write amplification:
  User writes 1 row:
  → WAL write (1 write)
  → Memtable (in memory, free)
  → SSTable flush (1 write per compaction level)
  → Compaction rewrites data at each level: L0→L1→L2→...
  Write amplification: ~10-30× (higher!)
  But: each write is sequential (fast), so throughput is still higher than B-Tree
```

### Read Amplification

```
B-Tree read:
  → At most tree_height page reads (typically 3-4)
  → Very predictable, always O(log n)

LSM-Tree read (worst case):
  → Check Memtable (fast, in-memory)
  → Check each Level 0 SSTable (multiple files, possible)
  → Check Level 1, 2, 3... (each level has one large SSTable)
  → Read amplification can be 10-50× for cold data
  → Bloom filters reduce this significantly in practice
```

### Space Amplification

```
B-Tree: dead rows accumulate until VACUUM runs
  → Space amplification: ~1-2× after vacuum, up to 3-4× without

LSM-Tree: old versions exist until compaction runs
  → Multiple SSTables may contain the same key
  → Space amplification: 1.1-1.5× in steady state (compaction handles this)
```

### Summary Comparison

| Property | B-Tree | LSM-Tree |
|----------|--------|----------|
| Write throughput | Moderate (random I/O) | High (sequential I/O) |
| Read throughput | High (predictable O(log n)) | Moderate (check multiple levels) |
| Write amplification | 3-10× | 10-30× |
| Read amplification | 1-4× | 1-50× (Bloom filters help) |
| Space efficiency | Good (after VACUUM) | Good (after compaction) |
| Compaction | VACUUM (B-Tree only prunes dead entries) | Full compaction (rewrites data) |
| Best for | Read-heavy OLTP | Write-heavy, time-series |
| Examples | PostgreSQL, MySQL (InnoDB) | Cassandra, RocksDB, LevelDB, HBase |

**PostgreSQL uses B-Trees** (with MVCC on top). This gives excellent read performance
for OLTP workloads. The VACUUM process handles dead tuple cleanup
(the equivalent of LSM compaction).

---

## 31.6 Other Indexing Structures

### Multi-Dimensional Indexes

Standard B-Trees are one-dimensional. What about queries on two dimensions?

```sql
-- Find all restaurants within a bounding box:
SELECT name FROM restaurants
WHERE latitude BETWEEN 12.9 AND 13.1
  AND longitude BETWEEN 77.5 AND 77.7;

-- B-Tree on (latitude, longitude):
-- Can efficiently filter on latitude (first column)
-- But must then scan all rows within that latitude range to filter longitude
-- Not ideal for 2D spatial queries

-- Solutions:
-- 1. Space-filling curve (e.g., Z-order / Morton code): maps 2D → 1D preserving locality
-- 2. R-Tree: tree where each node is a bounding rectangle (used by PostGIS)
-- 3. GiST with geometry types: what PostgreSQL's PostGIS uses

-- PostGIS example:
CREATE EXTENSION postgis;
ALTER TABLE restaurants ADD COLUMN location GEOMETRY(POINT, 4326);
CREATE INDEX ON restaurants USING GIST(location);

-- Now spatial queries use R-Tree internally:
SELECT name FROM restaurants
WHERE ST_DWithin(
    location,
    ST_MakePoint(77.5946, 12.9716)::geography,
    5000  -- within 5000 meters
);
```

### Full-Text Search Indexes

Standard indexes cannot answer "find documents containing the word 'database'":

```sql
-- PostgreSQL full-text search:
-- tsvector: preprocessed document (words, stemmed, stop words removed)
-- tsquery: search query with boolean operators

CREATE INDEX ON articles USING GIN(to_tsvector('english', content));

SELECT title FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database & indexing');

-- How tsvector works:
SELECT to_tsvector('english', 'PostgreSQL databases use B-Tree indexes for fast retrieval');
-- Output: 'b-tree':5 'databas':2 'fast':8 'index':6 'postgresql':1 'retriev':9 'use':3
-- Stop words ('for') removed, words stemmed ('databases'→'databas', 'indexes'→'index')
-- Position numbers kept for phrase matching
```

---

# CHAPTER 32 — Transaction Processing vs Analytics (OLTP vs OLAP)

---

## 32.1 WHY Two Paradigms Exist

Databases are used for two fundamentally different workloads that have
opposite performance requirements:

```
OLTP (Online Transaction Processing):
  Who uses it: End users, applications, APIs
  Pattern: Look up specific records by key, insert/update small sets of rows
  Example queries:
    SELECT * FROM orders WHERE id = 12345
    UPDATE products SET stock = stock - 1 WHERE id = 5
    INSERT INTO events (user_id, type) VALUES (100, 'click')
  Row count: single digits to hundreds per query
  Frequency: thousands to millions per second
  Latency required: < 10ms per query
  Indexes: crucial for fast key lookups

OLAP (Online Analytical Processing):
  Who uses it: Data analysts, data scientists, business intelligence tools
  Pattern: Scan millions/billions of rows, aggregate, group
  Example queries:
    SELECT city, SUM(revenue) FROM orders WHERE year=2024 GROUP BY city
    SELECT product, COUNT(*), AVG(price) FROM order_items GROUP BY product ORDER BY count DESC
    SELECT cohort_month, SUM(ltv) / COUNT(*) AS avg_ltv FROM user_cohorts GROUP BY 1
  Row count: millions to billions per query
  Frequency: dozens to hundreds per day
  Latency required: seconds to minutes (interactive), hours (batch)
  Indexes: less useful; column scans dominate
```

---

## 32.2 Data Warehousing

A **data warehouse** is a separate database optimized for OLAP queries.
It contains a read-only copy of data from multiple OLTP systems, transformed
and loaded for analytics.

```
OLTP Systems:              ETL Pipeline:           Data Warehouse:
  Orders DB    ──────────► Extract from         ──► Warehouse DB
  Inventory DB ──────────► each source,         ──► (BigQuery, Redshift,
  CRM DB       ──────────► Transform data,      ──► Snowflake, Redshift)
  Website logs ──────────► Load into warehouse  ──►
                           (nightly batch or
                            continuous streaming)
```

**Why not just query the OLTP database for analytics?**

```
Problem 1: Performance
  Long analytical queries scan millions of rows
  → blocks OLTP operations (tables are locked or vacuumed)
  → OLTP latency spikes → users see slow pages

Problem 2: Data structure
  OLTP: normalized (3NF) — optimized for writes
  Analytics: denormalized star schema — optimized for reads
  Running analytics on normalized OLTP data requires many JOINs → slow

Problem 3: Historical data
  OLTP: keeps current state (overwrite old values)
  Analytics: needs history (what was the price last year?)
  → Warehouse keeps full history with timestamps

Solution: separate warehouse, updated via ETL, optimized for analytics
```

---

## 32.3 Stars and Snowflakes — Schemas for Analytics

### Star Schema

The most common data warehouse schema.

```
                    ┌───────────────────────┐
                    │      fact_orders       │  ← CENTER: measurements
                    │  order_id (PK)         │
                    │  customer_id (FK) ──────────────────────────┐
                    │  product_id (FK) ──────────────────┐        │
                    │  date_id (FK) ──────────┐           │        │
                    │  store_id (FK) ─────┐   │           │        │
                    │  quantity          │   │           │        │
                    │  revenue           │   │           │        │
                    │  discount_amount   │   │           │        │
                    └───────────────────┬┘   │           │        │
                                        │    │           │        │
                    ┌───────────────────┘    │           │        │
                    ▼                        ▼           ▼        ▼
             ┌──────────────┐    ┌──────────────┐  ┌──────────┐  ┌──────────────┐
             │  dim_store   │    │   dim_date   │  │dim_product│  │ dim_customer │
             │ store_id(PK) │    │  date_id(PK) │  │product_id│  │ customer_id  │
             │ store_name   │    │  date        │  │ name     │  │ full_name    │
             │ city         │    │  month       │  │ category │  │ city         │
             │ region       │    │  quarter     │  │ brand    │  │ age_group    │
             │ store_type   │    │  year        │  │ supplier │  │ segment      │
             └──────────────┘    │  is_weekend  │  └──────────┘  └──────────────┘
                                 │  is_holiday  │
                                 └──────────────┘
```

**Fact table:** contains measurements, metrics, and foreign keys to dimensions.
Usually very large (billions of rows). Append-only.

**Dimension tables:** descriptive attributes. Usually small (thousands to millions of rows).
Slowly changing (customer city might update; date never changes).

**Why "star"?** The fact table is the center; dimension tables radiate out like points of a star.

### Snowflake Schema

A snowflake schema normalizes dimension tables further:

```
dim_product (in star schema):
  product_id | name | category_name | brand_name | supplier_name

dim_product (in snowflake schema):
  product_id | name | category_id (FK) | brand_id (FK) | supplier_id (FK)

dim_category:  category_id | category_name | department_id (FK)
dim_brand:     brand_id | brand_name | brand_country
dim_supplier:  supplier_id | supplier_name | country
dim_department: department_id | department_name
```

**Star vs Snowflake:**
- Star: denormalized → fewer JOINs in analytics queries → faster queries, simpler SQL
- Snowflake: more normalized → less storage, easier maintenance, more complex queries
- **Recommended: star schema** for most data warehouses (query speed matters more than storage)

### Analytics Queries on Star Schema

```sql
-- Revenue by city for Q1 2024:
SELECT
    c.city,
    SUM(f.revenue) AS total_revenue,
    COUNT(DISTINCT f.order_id) AS order_count,
    SUM(f.revenue) / COUNT(DISTINCT f.order_id) AS avg_order_value
FROM fact_orders f
JOIN dim_customer c ON c.customer_id = f.customer_id
JOIN dim_date d ON d.date_id = f.date_id
WHERE d.year = 2024 AND d.quarter = 1
GROUP BY c.city
ORDER BY total_revenue DESC;

-- Product category performance by month:
SELECT
    d.year,
    d.month,
    p.category,
    SUM(f.revenue) AS revenue,
    SUM(f.quantity) AS units_sold
FROM fact_orders f
JOIN dim_product p ON p.product_id = f.product_id
JOIN dim_date d ON d.date_id = f.date_id
WHERE d.year = 2024
GROUP BY d.year, d.month, p.category
ORDER BY d.year, d.month, revenue DESC;
```

---

# CHAPTER 33 — Column-Oriented Storage

---

## 33.1 WHY Column Storage Exists

Row storage (what PostgreSQL uses by default) stores all columns of a row together:

```
Row storage layout on disk:
  Page 1:
  [order_id=1, customer_id=101, product_id=5, date_id=20240115, quantity=2, revenue=199.98]
  [order_id=2, customer_id=102, product_id=8, date_id=20240115, quantity=1, revenue=99.99]
  [order_id=3, customer_id=101, product_id=3, date_id=20240116, quantity=5, revenue=49.95]
  ...
```

For an OLAP query like `SELECT SUM(revenue) FROM fact_orders WHERE year=2024`:

```
Row storage:
  Must read ALL columns of ALL rows to get just the revenue column
  100 bytes per row × 1 billion rows = 100GB read from disk
  But revenue column is only 8 bytes per row = 8GB of actual data needed
  → 92GB wasted I/O

Column storage:
  Revenue column stored separately:
  [199.98, 99.99, 49.95, 299.99, ...]  ← only revenue values, all together
  8GB of actual data, and we READ only 8GB
  → 12× less I/O for this query
```

---

## 33.2 Column Storage Layout

```
Fact table: fact_orders (5 columns, 1 billion rows)

ROW STORAGE (PostgreSQL heap):
  file: fact_orders
    row1: [1, 101, 5, 20240115, 2, 199.98]
    row2: [2, 102, 8, 20240115, 1, 99.99]
    row3: [3, 101, 3, 20240116, 5, 49.95]
    ...

COLUMN STORAGE (BigQuery, Redshift, Parquet):
  file: fact_orders.order_id
    [1, 2, 3, 4, 5, ...]  ← all 1B order IDs

  file: fact_orders.customer_id
    [101, 102, 101, 103, ...]  ← all 1B customer IDs

  file: fact_orders.revenue
    [199.98, 99.99, 49.95, 299.99, ...]  ← all 1B revenues

  -- Row 3 is: order_id[3], customer_id[3], revenue[3] = 3, 101, 49.95
  -- Row number preserves alignment across column files
```

---

## 33.3 Column Compression

Column storage compresses extremely well because all values in a column
have the same data type and are often similar.

### Run-Length Encoding

```
Raw customer_id column:
  [101, 101, 101, 102, 102, 103, 101, 101, 101, 101]

After sorting by customer_id (common in column stores):
  [101, 101, 101, 101, 101, 101, 102, 102, 103]

Run-Length Encoded (RLE):
  [(101, 6), (102, 2), (103, 1)]
  ← value, run_length pairs

Storage: 10 values × 4 bytes = 40 bytes → 3 pairs × 8 bytes = 24 bytes
Compression ratio: 40/24 = 1.67×

In practice, with millions of rows per customer:
  [(101, 50000), (102, 23000), ...]
Compression ratio: 50,000 values × 4 bytes = 200KB → 2 pairs × 8 bytes = 16 bytes
Compression ratio: 12,500× for this column!
```

### Bitmap Encoding

```
category column: ['Electronics', 'Electronics', 'Furniture', 'Electronics', 'Books']

Bitmap index per category:
  Electronics: [1, 1, 0, 1, 0]
  Furniture:   [0, 0, 1, 0, 0]
  Books:       [0, 0, 0, 0, 1]

Query: WHERE category = 'Electronics'
  → Bitmap: [1, 1, 0, 1, 0]
  → Fetch rows at positions 1, 2, 4 (where bit=1)

Combined query: WHERE category = 'Electronics' AND is_weekend = true
  → Electronics bitmap: [1, 1, 0, 1, 0]
  → is_weekend bitmap:  [0, 1, 1, 0, 1]
  → AND:                [0, 1, 0, 0, 0]
  → Only row 2 matches (bitwise AND, extremely fast with CPU SIMD instructions)
```

### Dictionary Encoding

```
product_name column (many repeating values):
  ["Widget Pro", "Gadget Basic", "Widget Pro", "Widget Pro", "Gadget Basic"]

Dictionary:
  0 → "Widget Pro"
  1 → "Gadget Basic"

Encoded column:
  [0, 1, 0, 0, 1]  ← 1 byte each instead of 10-15 bytes per string

Query: WHERE product_name = 'Widget Pro'
  → Look up 'Widget Pro' in dictionary → code 0
  → Scan encoded column for code 0 → [0, 1, 0, 0, 1] → rows 1,3,4
  → No string comparison needed! Integer comparison is 10× faster
```

---

## 33.4 Sort Order in Column Storage

Column stores typically sort data by one or more "sort keys":

```sql
-- Sort by date_id then customer_id:
-- All rows with date_id=20240101 come first, within that sorted by customer_id

Advantage 1: Range queries on sort key are fast
  WHERE date_id BETWEEN 20240101 AND 20240131
  → Contiguous block on disk → sequential read → fast

Advantage 2: Compression improves dramatically
  date_id column (sorted): [20240101, 20240101, 20240101, 20240102, ...]
  Runs are very long → RLE compresses to almost nothing

Advantage 3: Multiple sort orders via replicas
  -- Primary copy sorted by (date_id, customer_id) → good for date range queries
  -- Replica sorted by (customer_id, date_id) → good for per-customer queries
  -- Both are up-to-date; query optimizer picks which copy to read
  (Vertica, Redshift, Snowflake do this)
```

---

## 33.5 Writing to Column-Oriented Storage

Column storage optimizes reads at the cost of writes.
Writing a single row requires updating many column files:

```
INSERT one row: (order_id=1001, customer_id=50, revenue=299.99, ...)
→ Must append to order_id file
→ Must append to customer_id file
→ Must append to revenue file
→ ... (one write per column)
→ Must maintain sort order (if sorted → expensive re-sort for every insert)
```

**Solution: LSM-Tree approach for column stores**

```
Most column stores use a two-tier approach:

Tier 1: In-memory row-oriented buffer (small, fast writes)
  → Accepts all new writes in row format
  → Periodically sorts and flushes to column files

Tier 2: On-disk column files (large, optimized for reads)
  → Compressed, sorted column data
  → Background compaction merges in-memory flushes

Query: combines results from both tiers
  → Recent data: from in-memory buffer (fast)
  → Historical data: from column files (compressed, fast scans)
```

This is why BigQuery, Redshift, and Snowflake can accept streaming inserts
while also supporting fast analytical scans.

---

## 33.6 Aggregation: Data Cubes and Materialized Views

### The Problem with Raw Aggregations

```sql
-- Dashboard: total revenue by country, product category, month
-- 1 billion fact rows, query runs 100× per day = 100 billion row-reads per day

SELECT
    c.country,
    p.category,
    d.month,
    SUM(f.revenue)
FROM fact_orders f
JOIN dim_customer c ON ...
JOIN dim_product p ON ...
JOIN dim_date d ON ...
GROUP BY c.country, p.category, d.month;
-- Even with column storage: scanning 1B rows × 100 = extremely slow
```

### Materialized Views — Pre-Computed Results

```sql
-- Pre-compute and store the aggregate:
CREATE MATERIALIZED VIEW mv_revenue_summary AS
SELECT
    c.country,
    p.category,
    d.year,
    d.month,
    SUM(f.revenue) AS total_revenue,
    COUNT(f.order_id) AS order_count
FROM fact_orders f
JOIN dim_customer c ON c.customer_id = f.customer_id
JOIN dim_product p ON p.product_id = f.product_id
JOIN dim_date d ON d.date_id = f.date_id
GROUP BY c.country, p.category, d.year, d.month;

CREATE INDEX ON mv_revenue_summary(country, year, month);

-- Dashboard now queries the materialized view (thousands of rows, not billions):
SELECT country, SUM(total_revenue)
FROM mv_revenue_summary
WHERE year = 2024 AND month = 3
GROUP BY country;
-- Reads thousands of rows instead of billions. Instant!

-- Refresh (rebuild) when new data arrives:
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_revenue_summary;
-- CONCURRENTLY: allows reads during refresh
-- Requires unique index: CREATE UNIQUE INDEX ON mv_revenue_summary(country, category, year, month);
```

### Data Cubes (OLAP Cubes)

A data cube is a special case of materialized view — it pre-computes
aggregates for ALL combinations of dimensions:

```
Dimensions: Country, Category, Month (3 dimensions)
Measures: Revenue, Order Count

Data cube pre-computes:
  (India, Electronics, Jan) → $50,000, 500 orders
  (India, Electronics, Feb) → $45,000, 450 orders
  (India, Furniture, Jan)   → $20,000, 100 orders
  (USA, Electronics, Jan)   → $200,000, 2000 orders
  ... every combination ...

  Plus ALL subtotals:
  (India, Electronics, ALL) → $95,000, 950 orders  ← any month
  (India, ALL, Jan)         → $70,000, 600 orders  ← any category
  (ALL, Electronics, Jan)   → $250,000, 2500 orders ← any country
  (ALL, ALL, ALL)           → $315,000, 3050 orders ← grand total

SQL CUBE operator:
SELECT country, category, month, SUM(revenue)
FROM fact_orders_summary
GROUP BY CUBE(country, category, month);
-- Generates all 2^3 = 8 combinations of groupings automatically
```

**Trade-off:** Data cubes are extremely fast for pre-defined queries
but inflexible — adding a new dimension requires rebuilding the entire cube.

---

## Chapter 32–33 Summary

| Concept | Key Insight |
|---------|------------|
| OLTP | Few rows, high frequency, low latency — row storage wins |
| OLAP | Many rows, low frequency, high throughput — column storage wins |
| Data warehouse | Separate OLAP system; copy of OLTP data, transformed for analytics |
| Star schema | Fact table (measurements) + dimension tables (context) — fewest JOINs |
| Column storage | Read only the columns you need → 10-100× less I/O for analytics |
| Compression | Columns compress 10-1000× because same-type, similar values cluster together |
| Sort order | Sorting column data improves both compression and range query speed |
| Materialized views | Pre-computed aggregates → dashboard queries go from billions of rows to thousands |
| Data cubes | Pre-compute ALL dimension combinations → instant drill-down at cost of flexibility |

---

# PART IX — COMPLETE REFERENCE: DATA MODELS DECISION GUIDE

---

## When to Use Each Data Model

```
Your data fits ONE of these patterns?

  ┌─────────────────────────────────────────────────────────────────┐
  │                    START HERE                                   │
  │         What is the dominant structure of your data?            │
  └─────────────────┬──────────────────────┬────────────────────────┘
                    │                      │
                    ▼                      ▼
         ┌──────────────────┐   ┌──────────────────────┐
         │  Self-contained  │   │  Interconnected data  │
         │  documents/trees │   │  with relationships   │
         │  (profiles,      │   │  (orders, products,   │
         │   configs, logs) │   │   customers, etc.)    │
         └────────┬─────────┘   └──────────┬───────────┘
                  │                        │
                  ▼                        ▼
       ┌─────────────────────┐   ┌─────────────────────┐
       │  Document database  │   │  Relational database │
       │  (MongoDB, Firestore│   │  (PostgreSQL, MySQL) │
       │  or PostgreSQL JSONB│   │                      │
       └─────────────────────┘   └─────────────────────┘

OR:

  Everything relates to everything
  (social networks, knowledge graphs)?
         ↓
  ┌──────────────────────┐
  │  Graph database      │
  │  (Neo4j, Neptune)    │
  │  or recursive CTEs   │
  │  in PostgreSQL       │
  └──────────────────────┘

OR:

  Analytical workload, billions of rows, aggregate queries?
         ↓
  ┌──────────────────────┐
  │  Data warehouse with │
  │  column storage      │
  │  (BigQuery, Redshift,│
  │   Snowflake,         │
  │   or TimescaleDB)    │
  └──────────────────────┘
```

## Write Throughput vs Read Throughput Trade-off

```
Storage Engine Choice:

HIGH WRITE THROUGHPUT needed?          HIGH READ THROUGHPUT needed?
  → LSM-Tree (Cassandra, RocksDB)        → B-Tree (PostgreSQL, MySQL)
  → Sequential writes, fast inserts      → Predictable O(log n) reads
  → Read amplification (check levels)    → Write amplification (WAL + splits)
  → Good for: IoT, event streaming       → Good for: OLTP, web apps

ANALYTICAL READS on huge datasets?
  → Column storage (BigQuery, Redshift)
  → Read only needed columns
  → Excellent compression
  → Good for: data warehousing, BI
```

---

# PART X — MISSING POSTGRESQL FEATURES

---

# CHAPTER 34 — Window Functions

---

## 34.1 WHY Window Functions Exist

Before window functions, any "ranking within a group" or "running total"
required either a self-join or a correlated subquery — both slow and complex.

```sql
-- Problem: Find the top 3 products by revenue per category
-- Without window functions:
SELECT p.category, p.name, SUM(oi.revenue) AS revenue
FROM products p
JOIN order_items oi ON oi.product_id = p.id
GROUP BY p.category, p.id, p.name
HAVING SUM(oi.revenue) >= (
    SELECT SUM(oi2.revenue)
    FROM products p2
    JOIN order_items oi2 ON oi2.product_id = p2.id
    WHERE p2.category = p.category
    GROUP BY p2.id
    ORDER BY SUM(oi2.revenue) DESC
    OFFSET 2 LIMIT 1
);
-- Complex, slow, nearly unreadable

-- With window functions:
SELECT category, name, revenue
FROM (
    SELECT
        p.category,
        p.name,
        SUM(oi.revenue) AS revenue,
        RANK() OVER (PARTITION BY p.category ORDER BY SUM(oi.revenue) DESC) AS rk
    FROM products p
    JOIN order_items oi ON oi.product_id = p.id
    GROUP BY p.category, p.id, p.name
) ranked
WHERE rk <= 3;
-- Clean, fast, readable
```

---

## 34.2 WHAT a Window Function Is

A window function performs a calculation across a **set of related rows**
(the "window") without collapsing them into a single output row like GROUP BY does.

```sql
-- GROUP BY collapses rows:
SELECT department, AVG(salary) FROM employees GROUP BY department;
-- Returns ONE row per department. Individual employees are gone.

-- Window function KEEPS all rows AND adds the aggregate:
SELECT
    name,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg,
    salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;
-- Returns EVERY employee row PLUS their department average
-- name=Rahul, dept=Engineering, salary=90000, dept_avg=85000, diff=+5000
-- name=Priya, dept=Engineering, salary=80000, dept_avg=85000, diff=-5000
-- name=Vikram, dept=Marketing, salary=60000, dept_avg=65000, diff=-5000
```

---

## 34.3 Window Function Syntax

```sql
function_name(args) OVER (
    [PARTITION BY column(s)]   -- divide rows into groups (like GROUP BY, but doesn't collapse)
    [ORDER BY column(s)]       -- order within each partition
    [frame_clause]             -- which rows in the partition to include
)
```

---

## 34.4 Ranking Functions

```sql
-- Setup:
-- employees: id, name, department, salary, hire_date

-- ROW_NUMBER(): unique sequential number, no ties
SELECT name, department, salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num
FROM employees;
-- Even if two people have same salary, they get different row numbers (arbitrary order)

-- RANK(): ties get same rank, gaps after ties
SELECT name, department, salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
FROM employees;
-- salary: 90000 → rank 1
-- salary: 85000 → rank 2
-- salary: 85000 → rank 2 (tie)
-- salary: 80000 → rank 4 (skips 3 because two people were rank 2)

-- DENSE_RANK(): ties get same rank, NO gaps
SELECT name, department, salary,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk
FROM employees;
-- salary: 90000 → rank 1
-- salary: 85000 → rank 2
-- salary: 85000 → rank 2 (tie)
-- salary: 80000 → rank 3 (no gap!)

-- NTILE(n): divide rows into n equal groups
SELECT name, salary,
    NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;
-- quartile 1 = bottom 25%, quartile 4 = top 25%

-- PERCENT_RANK(): relative rank as percentage (0 to 1)
SELECT name, salary,
    ROUND(PERCENT_RANK() OVER (ORDER BY salary) * 100, 1) AS percentile
FROM employees;
-- Rahul at 90th percentile → percentile = 90.0

-- CUME_DIST(): cumulative distribution (what fraction of rows have value ≤ this)
SELECT name, salary,
    ROUND(CUME_DIST() OVER (ORDER BY salary) * 100, 1) AS cum_dist_pct
FROM employees;
```

---

## 34.5 Navigation Functions — LAG, LEAD, FIRST_VALUE, LAST_VALUE

```sql
-- LAG(column, n, default): value from n rows BEFORE current row
-- LEAD(column, n, default): value from n rows AFTER current row
SELECT
    order_date,
    revenue,
    LAG(revenue, 1, 0) OVER (ORDER BY order_date) AS prev_day_revenue,
    revenue - LAG(revenue, 1, 0) OVER (ORDER BY order_date) AS day_over_day_change,
    ROUND(100.0 * (revenue - LAG(revenue, 1) OVER (ORDER BY order_date))
          / NULLIF(LAG(revenue, 1) OVER (ORDER BY order_date), 0), 2) AS pct_change
FROM daily_revenue
ORDER BY order_date;

-- FIRST_VALUE / LAST_VALUE: value of first/last row in the window
SELECT
    name,
    department,
    salary,
    FIRST_VALUE(name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) AS highest_paid_in_dept,
    LAST_VALUE(name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- important!
    ) AS lowest_paid_in_dept
FROM employees;
```

---

## 34.6 Aggregate Window Functions

Any aggregate function (SUM, AVG, COUNT, MIN, MAX) can be used as a window function:

```sql
-- Running total (cumulative sum)
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (ORDER BY order_date) AS running_total
FROM daily_revenue;
-- Each row shows all revenue up to and including that day

-- Running total PER category (reset for each category)
SELECT
    category,
    order_date,
    revenue,
    SUM(revenue) OVER (
        PARTITION BY category
        ORDER BY order_date
    ) AS category_running_total
FROM category_daily_revenue;

-- Moving average (last 7 days)
SELECT
    order_date,
    revenue,
    ROUND(AVG(revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW  -- 7-day window
    ), 2) AS moving_avg_7d
FROM daily_revenue;

-- Percentage of total
SELECT
    category,
    revenue,
    ROUND(100.0 * revenue / SUM(revenue) OVER (), 2) AS pct_of_total,
    ROUND(100.0 * revenue / SUM(revenue) OVER (PARTITION BY category), 2) AS pct_of_category
FROM category_revenue;
```

---

## 34.7 Frame Clause — Controlling Which Rows Are in the Window

```sql
-- Frame clause options:
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- from first row to current
ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING           -- 2 before, 2 after
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING   -- from current to last
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- all rows in partition

-- RANGE vs ROWS:
-- ROWS: physical row offset (count rows)
-- RANGE: logical range based on ORDER BY value (include ties)

-- Example: salary within ±10000 of current salary
SELECT name, salary,
    AVG(salary) OVER (
        ORDER BY salary
        RANGE BETWEEN 10000 PRECEDING AND 10000 FOLLOWING
    ) AS nearby_avg
FROM employees;
```

---

# CHAPTER 35 — PL/pgSQL: Stored Procedures, Functions & Triggers

---

## 35.1 WHY PL/pgSQL Exists

Some business logic MUST run in the database:
- Complex validation that requires reading multiple tables
- Automatic actions when data changes (audit logs, stock updates)
- Batch operations that would cause N+1 if done from the application

PL/pgSQL is PostgreSQL's procedural language — SQL with variables, loops, conditionals, and exception handling.

---

## 35.2 Creating Functions

```sql
-- Basic function: calculate order total with discount
CREATE OR REPLACE FUNCTION calculate_order_total(
    p_order_id  BIGINT,
    p_discount  NUMERIC DEFAULT 0
)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    v_subtotal  NUMERIC;
    v_total     NUMERIC;
BEGIN
    -- Calculate subtotal from order items
    SELECT SUM(quantity * price_at_purchase)
    INTO v_subtotal
    FROM order_items
    WHERE order_id = p_order_id;

    IF v_subtotal IS NULL THEN
        RAISE EXCEPTION 'Order % not found or has no items', p_order_id;
    END IF;

    v_total := v_subtotal * (1 - p_discount / 100);

    RETURN ROUND(v_total, 2);
END;
$$;

-- Usage:
SELECT calculate_order_total(1001);            -- no discount
SELECT calculate_order_total(1001, 10);         -- 10% discount
SELECT calculate_order_total(1001, 0.0);        -- explicit no discount
```

---

## 35.3 Functions That Return Tables

```sql
-- Return a table of low-stock products with reorder suggestions
CREATE OR REPLACE FUNCTION get_low_stock_report(p_threshold INT DEFAULT 10)
RETURNS TABLE (
    product_id    BIGINT,
    product_name  VARCHAR,
    sku           VARCHAR,
    current_stock INT,
    reorder_qty   INT
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT
        p.id,
        p.name,
        p.sku,
        p.stock_quantity,
        GREATEST(50 - p.stock_quantity, 10) AS reorder_qty
    FROM products p
    WHERE p.stock_quantity <= p_threshold
    ORDER BY p.stock_quantity ASC;
END;
$$;

-- Usage (works like a table):
SELECT * FROM get_low_stock_report(5);
SELECT * FROM get_low_stock_report() WHERE reorder_qty > 20;
```

---

## 35.4 Stored Procedures — Transactions Inside the Database

Unlike functions, procedures can COMMIT and ROLLBACK:

```sql
-- Procedure: process a batch of pending orders
CREATE OR REPLACE PROCEDURE process_pending_orders(p_batch_size INT DEFAULT 100)
LANGUAGE plpgsql
AS $$
DECLARE
    v_order     RECORD;
    v_processed INT := 0;
    v_failed    INT := 0;
BEGIN
    FOR v_order IN
        SELECT id, customer_id, total_amount
        FROM orders
        WHERE status = 'pending'
        ORDER BY created_at
        LIMIT p_batch_size
        FOR UPDATE SKIP LOCKED  -- safe for concurrent workers
    LOOP
        BEGIN
            -- Process the order
            UPDATE orders SET status = 'processing', updated_at = NOW()
            WHERE id = v_order.id;

            -- Simulate work (in real code: call payment API, etc.)
            PERFORM pg_sleep(0.001);

            UPDATE orders SET status = 'completed', updated_at = NOW()
            WHERE id = v_order.id;

            v_processed := v_processed + 1;

        EXCEPTION WHEN OTHERS THEN
            -- Individual order failure: rollback just this one
            UPDATE orders SET status = 'failed', error_message = SQLERRM
            WHERE id = v_order.id;
            v_failed := v_failed + 1;
        END;

        -- Commit every 10 orders to release locks faster
        IF (v_processed + v_failed) % 10 = 0 THEN
            COMMIT;
        END IF;
    END LOOP;

    COMMIT;
    RAISE NOTICE 'Processed: %, Failed: %', v_processed, v_failed;
END;
$$;

-- Run it:
CALL process_pending_orders(50);
```

---

## 35.5 Triggers — Automatic Actions on Data Changes

Triggers fire automatically when INSERT, UPDATE, or DELETE happens on a table.

```sql
-- Trigger 1: Automatically update updated_at timestamp
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;  -- RETURN NEW for row-level BEFORE triggers
END;
$$;

CREATE TRIGGER trg_products_updated_at
BEFORE UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

-- Now every UPDATE on products auto-sets updated_at:
UPDATE products SET price = 149 WHERE id = 1;
-- updated_at is automatically NOW() — no application code needed

-- Trigger 2: Audit log — record all price changes
CREATE TABLE product_price_audit (
    id          BIGSERIAL PRIMARY KEY,
    product_id  BIGINT NOT NULL,
    old_price   NUMERIC(10,2),
    new_price   NUMERIC(10,2),
    changed_by  TEXT DEFAULT current_user,
    changed_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION audit_price_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF OLD.price IS DISTINCT FROM NEW.price THEN
        INSERT INTO product_price_audit (product_id, old_price, new_price)
        VALUES (NEW.id, OLD.price, NEW.price);
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_audit_price
AFTER UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION audit_price_change();

-- Trigger 3: Prevent negative stock
CREATE OR REPLACE FUNCTION prevent_negative_stock()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.stock_quantity < 0 THEN
        RAISE EXCEPTION 'Stock for product % cannot go below 0. Attempted: %',
            NEW.id, NEW.stock_quantity;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_prevent_negative_stock
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION prevent_negative_stock();

-- Trigger 4: Statement-level trigger (fires once per statement, not per row)
CREATE OR REPLACE FUNCTION log_bulk_update()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO audit_log (action, table_name, performed_at, user_name)
    VALUES (TG_OP, TG_TABLE_NAME, NOW(), current_user);
    RETURN NULL;  -- statement-level triggers return NULL
END;
$$;

CREATE TRIGGER trg_log_updates
AFTER UPDATE ON products
FOR EACH STATEMENT
EXECUTE FUNCTION log_bulk_update();
```

---

## 35.6 Control Flow in PL/pgSQL

```sql
CREATE OR REPLACE FUNCTION demo_control_flow(p_value INT)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
    v_result TEXT;
    v_count  INT;
    v_record RECORD;
BEGIN
    -- IF/ELSIF/ELSE
    IF p_value > 100 THEN
        v_result := 'large';
    ELSIF p_value > 10 THEN
        v_result := 'medium';
    ELSE
        v_result := 'small';
    END IF;

    -- CASE
    v_result := CASE
        WHEN p_value > 100 THEN 'large'
        WHEN p_value > 10  THEN 'medium'
        ELSE 'small'
    END;

    -- Simple FOR loop
    FOR i IN 1..5 LOOP
        RAISE NOTICE 'Iteration: %', i;
    END LOOP;

    -- FOR loop over query results
    FOR v_record IN
        SELECT id, name FROM products WHERE stock_quantity < 5
    LOOP
        RAISE NOTICE 'Low stock: % (id=%)', v_record.name, v_record.id;
    END LOOP;

    -- WHILE loop
    v_count := 0;
    WHILE v_count < p_value LOOP
        v_count := v_count + 1;
    END LOOP;

    -- EXIT a loop early
    FOR i IN 1..1000 LOOP
        EXIT WHEN i > 10;  -- break out of loop
        RAISE NOTICE '%', i;
    END LOOP;

    RETURN v_result;

EXCEPTION
    WHEN division_by_zero THEN
        RETURN 'division error';
    WHEN OTHERS THEN
        RAISE NOTICE 'Unexpected error: %', SQLERRM;
        RETURN 'error: ' || SQLERRM;
END;
$$;
```

---

# CHAPTER 36 — Views

---

## 36.1 Regular Views

A view is a named query stored in the database. It behaves exactly like a table
from the caller's perspective, but reads are always computed from the underlying query.

```sql
-- Create a view for active customers with their order stats
CREATE VIEW active_customer_summary AS
SELECT
    c.id,
    c.full_name,
    c.email,
    COUNT(o.id) AS total_orders,
    COALESCE(SUM(o.total_amount), 0) AS lifetime_value,
    MAX(o.created_at) AS last_order_date
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE c.deleted_at IS NULL
GROUP BY c.id, c.full_name, c.email;

-- Use exactly like a table:
SELECT * FROM active_customer_summary WHERE lifetime_value > 10000;
SELECT full_name FROM active_customer_summary ORDER BY lifetime_value DESC LIMIT 10;

-- Views are transparent to the planner:
-- If underlying tables have indexes, the planner uses them
-- The view definition is inlined (not materialized) at query time
```

---

## 36.2 Updatable Views

Simple views (single table, no aggregates, no DISTINCT, no UNION) are automatically updatable:

```sql
CREATE VIEW active_products AS
SELECT id, name, sku, price, stock_quantity
FROM products
WHERE deleted_at IS NULL;

-- Can INSERT, UPDATE, DELETE through the view:
UPDATE active_products SET price = 149 WHERE sku = 'WID-001';
-- Equivalent to: UPDATE products SET price = 149 WHERE sku = 'WID-001' AND deleted_at IS NULL

-- WITH CHECK OPTION: prevent INSERTs/UPDATEs that violate the view's WHERE:
CREATE VIEW active_products AS
SELECT id, name, sku, price, stock_quantity, deleted_at
FROM products
WHERE deleted_at IS NULL
WITH CHECK OPTION;

-- Now this would fail:
UPDATE active_products SET deleted_at = NOW() WHERE id = 1;
-- ERROR: new row violates check option for view "active_products"
-- (setting deleted_at would make the row invisible through the view)
```

---

## 36.3 Materialized Views — When and How

```sql
-- Expensive query: customer lifetime value with product preferences
CREATE MATERIALIZED VIEW customer_analytics AS
SELECT
    c.id AS customer_id,
    c.full_name,
    c.email,
    COUNT(DISTINCT o.id) AS total_orders,
    SUM(o.total_amount) AS lifetime_value,
    MODE() WITHIN GROUP (ORDER BY p.category) AS favorite_category,
    MAX(o.created_at) AS last_order_at,
    MIN(o.created_at) AS first_order_at
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE c.deleted_at IS NULL
GROUP BY c.id, c.full_name, c.email;

-- Create unique index (required for CONCURRENTLY refresh):
CREATE UNIQUE INDEX ON customer_analytics(customer_id);
-- Add indexes for common query patterns:
CREATE INDEX ON customer_analytics(lifetime_value DESC);
CREATE INDEX ON customer_analytics(favorite_category);

-- Query is now instant (reads from pre-computed table):
SELECT * FROM customer_analytics
WHERE favorite_category = 'Electronics'
ORDER BY lifetime_value DESC
LIMIT 20;

-- Refresh options:
REFRESH MATERIALIZED VIEW customer_analytics;             -- blocks reads!
REFRESH MATERIALIZED VIEW CONCURRENTLY customer_analytics; -- allows reads during refresh

-- Automate refresh with pg_cron extension:
SELECT cron.schedule('refresh-analytics', '0 * * * *', -- every hour
    'REFRESH MATERIALIZED VIEW CONCURRENTLY customer_analytics');

-- Check when it was last refreshed:
SELECT schemaname, matviewname, ispopulated
FROM pg_matviews
WHERE matviewname = 'customer_analytics';
```

---

# CHAPTER 37 — JSONB Deep Dive

---

## 37.1 JSON vs JSONB

```sql
-- JSON: stores exact text representation, no indexing on contents
-- JSONB: binary parsed storage, indexable, operators available
-- Always prefer JSONB unless you need to preserve key order or duplicate keys

CREATE TABLE events (
    id         BIGSERIAL PRIMARY KEY,
    type       VARCHAR(50) NOT NULL,
    payload    JSONB NOT NULL,         -- binary, indexed
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 37.2 JSONB Operators

```sql
-- Sample data:
INSERT INTO events (type, payload) VALUES
('purchase', '{"user_id": 5, "items": [{"sku": "W001", "qty": 2}, {"sku": "G001", "qty": 1}], "total": 299.97, "city": "Mumbai"}'),
('login',    '{"user_id": 5, "ip": "192.168.1.1", "device": "mobile"}'),
('purchase', '{"user_id": 7, "items": [{"sku": "W001", "qty": 1}], "total": 99.99, "city": "Delhi"}');

-- -> (arrow): get field as JSON
SELECT payload->'user_id' FROM events;          -- returns: 5, 5, 7 (as JSON)

-- ->> (double arrow): get field as TEXT
SELECT payload->>'city' FROM events WHERE type = 'purchase';  -- "Mumbai", "Delhi"

-- #> (path): navigate nested path
SELECT payload #> '{items, 0, sku}' FROM events WHERE type = 'purchase';  -- first item SKU

-- #>> (path as text):
SELECT payload #>> '{items, 0, sku}' FROM events WHERE type = 'purchase';  -- "W001"

-- @> (contains): does JSONB contain this sub-document?
SELECT * FROM events WHERE payload @> '{"user_id": 5}';        -- events for user 5
SELECT * FROM events WHERE payload @> '{"city": "Mumbai"}';    -- Mumbai events
SELECT * FROM events WHERE payload @> '{"items": [{"sku": "W001"}]}'; -- events with W001

-- <@ (contained by)
SELECT * FROM events WHERE '{"user_id": 5}' <@ payload;        -- same as @>

-- ? (key exists)
SELECT * FROM events WHERE payload ? 'city';                    -- events with 'city' key
SELECT * FROM events WHERE payload ? 'ip';                      -- login events

-- ?| (any of these keys exist)
SELECT * FROM events WHERE payload ?| ARRAY['city', 'ip'];

-- ?& (all of these keys exist)
SELECT * FROM events WHERE payload ?& ARRAY['user_id', 'total'];

-- - (delete key): returns JSONB with key removed
SELECT payload - 'user_id' FROM events LIMIT 1;

-- || (concatenate/merge):
UPDATE events SET payload = payload || '{"processed": true}' WHERE id = 1;
```

---

## 37.3 Indexing JSONB

```sql
-- GIN index on entire JSONB column (supports @>, ?, ?|, ?&):
CREATE INDEX ix_events_payload ON events USING GIN(payload);

-- Faster for specific operators, smaller index:
CREATE INDEX ix_events_payload_ops ON events USING GIN(payload jsonb_path_ops);
-- jsonb_path_ops: supports @> only (not ? operators), but faster and smaller

-- Index on a specific JSON field (expression index):
CREATE INDEX ix_events_user_id ON events((payload->>'user_id'));
-- Now this is fast:
SELECT * FROM events WHERE payload->>'user_id' = '5';

-- Index on nested field:
CREATE INDEX ix_events_city ON events((payload->>'city'));

-- Functional index on integer cast:
CREATE INDEX ix_events_user_int ON events(((payload->>'user_id')::INT));
SELECT * FROM events WHERE (payload->>'user_id')::INT = 5;  -- uses index
```

---

## 37.4 JSONB Path Queries

```sql
-- jsonb_path_query: find all items matching a path expression
SELECT jsonb_path_query(payload, '$.items[*].sku') AS sku
FROM events
WHERE type = 'purchase';
-- Returns each SKU as a separate row

-- jsonb_path_exists: does a path match exist?
SELECT * FROM events
WHERE jsonb_path_exists(payload, '$.items[*] ? (@.qty > 1)');
-- Events where any item has qty > 1

-- jsonb_path_query_array: return all matches as an array
SELECT jsonb_path_query_array(payload, '$.items[*].sku') AS all_skus
FROM events WHERE type = 'purchase';
-- [["W001", "G001"], ["W001"]]

-- Aggregating across JSONB arrays:
SELECT
    e.id,
    SUM((item->>'qty')::INT) AS total_items
FROM events e,
LATERAL jsonb_array_elements(payload->'items') AS item
WHERE e.type = 'purchase'
GROUP BY e.id;
```

---

# CHAPTER 38 — Table Partitioning

---

## 38.1 WHY Partition Tables

For tables with hundreds of millions of rows:
- Queries that filter on the partition key only read relevant partitions
- VACUUM, ANALYZE, REINDEX work on one partition at a time
- Old data can be dropped instantly (`DROP TABLE partition` is instant)
- Different partitions can have different storage (SSD for recent, HDD for archive)

---

## 38.2 Range Partitioning

```sql
-- Create partitioned table:
CREATE TABLE orders (
    id            BIGSERIAL,
    customer_id   BIGINT NOT NULL,
    total_amount  NUMERIC(12,2) NOT NULL,
    status        VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions:
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03 PARTITION OF orders
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Default partition (catches anything not covered):
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- Indexes: create on parent, automatically applied to all partitions
CREATE INDEX ON orders(customer_id, created_at DESC);
CREATE INDEX ON orders(status) WHERE status IN ('pending', 'processing');

-- INSERTs automatically route to correct partition:
INSERT INTO orders (customer_id, total_amount, created_at)
VALUES (5, 299.99, '2024-02-15');
-- → automatically goes to orders_2024_02

-- Partition pruning: query only reads relevant partitions
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-02-01' AND created_at < '2024-03-01';
-- Partitions: orders_2024_02  ← only reads this partition, not all of them!

-- Drop old data instantly (vs DELETE which is slow):
DROP TABLE orders_2023_01;  -- instant! No row-by-row delete needed

-- Automate partition creation (using pg_cron):
SELECT cron.schedule('create-monthly-partition', '0 0 25 * *',  -- 25th of each month
$$
DO $$
DECLARE
    next_month DATE := date_trunc('month', NOW() + INTERVAL '1 month');
    partition_name TEXT := 'orders_' || to_char(next_month, 'YYYY_MM');
BEGIN
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF orders FOR VALUES FROM (%L) TO (%L)',
        partition_name,
        next_month,
        next_month + INTERVAL '1 month'
    );
END;
$$
$$);
```

---

## 38.3 List Partitioning

```sql
-- Partition by a discrete set of values (e.g., country/region)
CREATE TABLE customers (
    id       BIGSERIAL,
    name     VARCHAR(255),
    email    VARCHAR(255),
    region   VARCHAR(20) NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE customers_india PARTITION OF customers
    FOR VALUES IN ('IN', 'India');

CREATE TABLE customers_usa PARTITION OF customers
    FOR VALUES IN ('US', 'USA', 'United States');

CREATE TABLE customers_eu PARTITION OF customers
    FOR VALUES IN ('UK', 'DE', 'FR', 'IT', 'ES');

CREATE TABLE customers_other PARTITION OF customers DEFAULT;

-- Query for India only reads the India partition:
SELECT * FROM customers WHERE region = 'IN';
```

---

## 38.4 Hash Partitioning

```sql
-- Distribute rows evenly across partitions (good for load balancing)
CREATE TABLE sessions (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    BIGINT NOT NULL,
    data       JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY HASH (user_id);

-- 4 partitions, evenly distributed:
CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- user_id=5: hash(5) % 4 = 1 → goes to sessions_1
-- user_id=8: hash(8) % 4 = 0 → goes to sessions_0
```

---

# CHAPTER 39 — Security: Roles, RLS, and Permissions

---

## 39.1 Roles and Permissions

```sql
-- Create roles (PostgreSQL: roles = users + groups)
CREATE ROLE app_readonly;    -- read-only application role
CREATE ROLE app_readwrite;   -- read-write application role
CREATE ROLE app_admin;       -- admin role

-- Grant table permissions to roles:
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT ALL ON ALL TABLES IN SCHEMA public TO app_admin;

-- Grant sequence usage (for INSERT with SERIAL/BIGSERIAL):
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;

-- Future tables automatically get the same permissions:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;

-- Create actual login users and assign roles:
CREATE USER api_service WITH PASSWORD 'strong_password_here' LOGIN;
GRANT app_readwrite TO api_service;

CREATE USER analytics_user WITH PASSWORD 'another_password' LOGIN;
GRANT app_readonly TO analytics_user;

-- Revoke permissions:
REVOKE DELETE ON orders FROM app_readwrite;  -- no deletions via API

-- View current permissions:
\dp products   -- in psql: show privileges on products table
SELECT grantee, privilege_type FROM information_schema.role_table_grants
WHERE table_name = 'products';
```

---

## 39.2 Row-Level Security (RLS)

RLS restricts which rows each user can see or modify.
Essential for multi-tenant applications.

```sql
-- Enable RLS on table:
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: customers can only see their own orders
CREATE POLICY customer_orders_policy ON orders
    FOR ALL
    TO app_readwrite
    USING (customer_id = current_setting('app.current_user_id')::BIGINT);

-- Application sets the context variable before queries:
-- SET app.current_user_id = '5';
-- SELECT * FROM orders; ← automatically filtered to customer_id=5

-- Admin bypass (superuser ignores RLS by default):
-- Or explicitly:
ALTER TABLE orders FORCE ROW LEVEL SECURITY;  -- even table owner must obey RLS
CREATE POLICY admin_all ON orders TO app_admin USING (true);  -- admin sees all

-- Separate read and write policies:
-- SELECT policy:
CREATE POLICY select_own_orders ON orders
    FOR SELECT
    USING (customer_id = current_setting('app.current_user_id')::BIGINT);

-- INSERT policy (customer can only create orders for themselves):
CREATE POLICY insert_own_orders ON orders
    FOR INSERT
    WITH CHECK (customer_id = current_setting('app.current_user_id')::BIGINT);

-- In application code (Python example):
def get_orders_for_user(conn, user_id):
    with conn.cursor() as cur:
        cur.execute("SET LOCAL app.current_user_id = %s", (str(user_id),))
        cur.execute("SELECT * FROM orders ORDER BY created_at DESC")
        return cur.fetchall()
    # SET LOCAL is scoped to the transaction — safe with connection pooling
```

---

# CHAPTER 40 — Replication

---

## 40.1 WHY Replication

```
Single PostgreSQL server problems:
  → If it crashes: zero availability until restart (minutes of downtime)
  → If disk fails: data loss possible
  → All read AND write load goes to one server
  → Can't serve reads while doing maintenance

Replication solutions:
  → Standby server: takes over if primary fails (High Availability)
  → Read replicas: serve SELECT queries, reducing primary load
  → Disaster recovery: geographical copy of data
```

---

## 40.2 Streaming Replication (Physical)

Physical replication copies the exact WAL byte stream to standbys.
The standby replays WAL, keeping an identical copy of all data.

```
Primary:
  Accepts all reads + writes
  Every write → WAL record
  WAL shipped → Standby(s)

Standby (hot standby):
  Replays WAL continuously
  Accepts read-only SELECT queries
  Cannot accept writes (it's a replica)
  Can be promoted to primary on failover
```

**Setup (postgresql.conf on PRIMARY):**
```ini
wal_level = replica              -- or logical; replica enables streaming replication
max_wal_senders = 5              -- max concurrent replication connections
wal_keep_size = 1GB              -- keep this much WAL for slow standbys
synchronous_standby_names = ''   -- '' = async; 'standby1' = synchronous
```

**pg_hba.conf on PRIMARY:**
```
host  replication  replicator  standby_ip/32  scram-sha-256
```

**Setup on STANDBY:**
```bash
# Create base backup from primary:
pg_basebackup -h primary_ip -U replicator -D /var/lib/postgresql/data -P -R
# -R: automatically creates standby.signal and recovery config

# Start PostgreSQL on standby — it will start replaying WAL automatically
```

**Monitor replication lag:**
```sql
-- On PRIMARY:
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag    -- how far behind the standby is
FROM pg_stat_replication;

-- On STANDBY:
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

---

## 40.3 Synchronous vs Asynchronous Replication

```sql
-- ASYNCHRONOUS (default):
-- Primary commits → returns to application
-- Standby receives WAL later (usually milliseconds, but could be more)
-- Risk: primary crashes after commit but before standby receives WAL
--       → data loss of in-flight WAL

-- SYNCHRONOUS:
-- Primary commits → waits for standby to confirm WAL written → returns to application
-- No data loss: standby has confirmed before application gets success
-- Cost: every COMMIT adds round-trip latency to the standby

-- Configure in postgresql.conf:
synchronous_standby_names = 'standby1'          -- wait for 1 standby
synchronous_standby_names = 'ANY 1 (s1,s2,s3)' -- wait for ANY 1 of 3

-- Per-transaction override:
SET synchronous_commit = off;   -- async for this transaction (small latency reduction)
SET synchronous_commit = local; -- wait for local WAL flush only (not standby)
SET synchronous_commit = on;    -- wait for standby (default if synchronous configured)
```

---

## 40.4 Logical Replication

Physical replication copies everything byte-for-byte.
Logical replication replicates at the SQL level — specific tables, filtered rows.

```sql
-- USE CASES:
-- 1. Replicate only some tables (not the entire database)
-- 2. Replicate to a different PostgreSQL version
-- 3. Replicate to a different database system (via logical decoding)
-- 4. Zero-downtime major version upgrades

-- On PUBLISHER (source):
-- postgresql.conf:
-- wal_level = logical   (required for logical replication)

CREATE PUBLICATION my_pub FOR TABLE products, customers, orders;
-- OR: all tables:
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- On SUBSCRIBER (destination):
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=primary_ip dbname=mydb user=replicator password=...'
PUBLICATION my_pub;

-- Monitor:
SELECT subname, pid, relid::regclass, received_lsn, latest_end_lsn
FROM pg_stat_subscription;
```

---

## 40.5 Backup and Point-in-Time Recovery (PITR)

```bash
# Physical backup using pg_basebackup:
pg_basebackup \
    -h localhost \
    -U postgres \
    -D /backup/basebackup_$(date +%Y%m%d) \
    --format=tar \
    --compress=9 \
    --checkpoint=fast \
    --progress

# Logical backup using pg_dump (single database, portable):
pg_dump -h localhost -U postgres mydb > /backup/mydb_$(date +%Y%m%d).sql
pg_dump -h localhost -U postgres -Fc mydb > /backup/mydb_$(date +%Y%m%d).dump  # custom format

# Logical backup of all databases:
pg_dumpall -h localhost -U postgres > /backup/all_$(date +%Y%m%d).sql
```

**PITR Setup (WAL archiving):**
```ini
# postgresql.conf:
archive_mode = on
archive_command = 'cp %p /wal_archive/%f'  -- or use WAL-G, pgBackRest
```

**Recovery to a specific point in time:**
```bash
# 1. Restore base backup
cp -r /backup/basebackup_20240115 /var/lib/postgresql/data

# 2. Create recovery configuration
cat > /var/lib/postgresql/data/recovery.conf << EOF
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'
EOF

# 3. Create recovery signal file
touch /var/lib/postgresql/data/recovery.signal

# 4. Start PostgreSQL — it will replay WAL until the target time
pg_ctl start -D /var/lib/postgresql/data
```

---

# CHAPTER 41 — Connection Pooling with PgBouncer

---

## 41.1 WHY PgBouncer is Essential

PostgreSQL spawns a separate OS process per connection (~5MB RAM each).

```
Problem:
  100 application servers × 10 connections each = 1000 PostgreSQL processes
  1000 × 5MB = 5GB RAM just for connection overhead
  + All 1000 processes may not be active simultaneously → waste

  Each process also:
  → Has its own query cache
  → Competes for shared_buffers
  → Adds OS scheduler overhead

Solution: PgBouncer sits between app and PostgreSQL
  App connects to PgBouncer (thousands of connections supported)
  PgBouncer maintains a small pool of actual PostgreSQL connections (e.g., 20)
  App requests are queued and multiplexed over the 20 real connections
```

---

## 41.2 PgBouncer Pooling Modes

```
Session mode (default):
  Connection assigned to client for the entire session
  Client thinks it has a dedicated connection
  Best compatibility (works with prepared statements, SET, advisory locks)
  Least pooling efficiency (1:1 mapping while session is active)

Transaction mode (recommended for most web apps):
  Connection assigned only during a transaction
  Connection returned to pool after COMMIT/ROLLBACK
  Most efficient: 100 app servers can share 20 PostgreSQL connections
  Limitation: SET, advisory locks, prepared statements don't work across transactions

Statement mode:
  Connection returned after each SQL statement
  Most aggressive pooling, lowest latency
  Very restrictive: multi-statement transactions not possible
```

**pgbouncer.ini:**
```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 10000      -- max app connections to PgBouncer
default_pool_size = 20       -- max real PostgreSQL connections per database+user pair
min_pool_size = 5            -- keep this many connections warm
reserve_pool_size = 5        -- extra connections for emergency
server_idle_timeout = 600    -- close idle server connections after 10 min
```

**Monitoring:**
```sql
-- Connect to PgBouncer admin console:
psql -h localhost -p 6432 -U pgbouncer pgbouncer

SHOW STATS;   -- request rates, average wait time
SHOW POOLS;   -- cl_active, cl_waiting, sv_active, sv_idle
SHOW CLIENTS; -- all client connections
SHOW SERVERS; -- all server connections
```

---

# CHAPTER 42 — Full-Text Search

---

## 42.1 How PostgreSQL Full-Text Search Works

```sql
-- tsvector: preprocessed document for full-text search
SELECT to_tsvector('english', 'PostgreSQL databases are powerful and fast systems');
-- 'databas':2 'fast':6 'postgresql':1 'power':4 'system':7
-- Stop words ('are', 'and') removed
-- Words stemmed: 'databases'→'databas', 'powerful'→'power'
-- Position numbers kept: 'postgresql' is at position 1

-- tsquery: search term with boolean operators
SELECT to_tsquery('english', 'postgresql & (fast | powerful)');
-- 'postgresql' & ( 'fast' | 'power' )

-- Match check:
SELECT to_tsvector('english', 'PostgreSQL is fast') @@ to_tsquery('english', 'postgresql & fast');
-- true

-- Phrase search (words must be adjacent):
SELECT to_tsvector('english', 'full text search') @@ phraseto_tsquery('english', 'full text');
-- true
```

---

## 42.2 Setting Up Full-Text Search

```sql
-- Option 1: index the computed tsvector (recalculated on every query):
CREATE INDEX ix_articles_fts ON articles
    USING GIN (to_tsvector('english', title || ' ' || COALESCE(content, '')));

SELECT title FROM articles
WHERE to_tsvector('english', title || ' ' || content) @@ to_tsquery('english', 'database & indexing');

-- Option 2: store tsvector as a column (faster queries, requires maintenance):
ALTER TABLE articles ADD COLUMN search_vector tsvector;
UPDATE articles SET search_vector =
    to_tsvector('english', title || ' ' || COALESCE(content, ''));
CREATE INDEX ix_articles_fts ON articles USING GIN(search_vector);

-- Keep it updated with a trigger:
CREATE OR REPLACE FUNCTION update_search_vector() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.content, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_search_vector
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION update_search_vector();
-- setweight: 'A' = highest weight, 'D' = lowest
-- Matches in title rank higher than matches in content
```

---

## 42.3 Ranking Results

```sql
-- ts_rank: rank by term frequency
-- ts_rank_cd: rank with cover density (considers proximity of terms)
SELECT
    title,
    ts_rank(search_vector, query) AS rank,
    ts_headline('english', content, query,
        'MaxWords=50, MinWords=20, StartSel=<b>, StopSel=</b>') AS excerpt
FROM articles,
     to_tsquery('english', 'postgresql & indexing') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- ts_headline: generate highlighted excerpt with matched terms
-- The excerpt shows surrounding context with matched terms bolded
```

---

# CHAPTER 43 — Sequences and Auto-Increment

---

## 43.1 SERIAL vs GENERATED vs Sequences

```sql
-- SERIAL (legacy, still works):
CREATE TABLE products (
    id SERIAL PRIMARY KEY  -- creates sequence, sets default, NOT NULL
);
-- Equivalent to:
CREATE SEQUENCE products_id_seq;
CREATE TABLE products (id INT DEFAULT nextval('products_id_seq') NOT NULL);

-- BIGSERIAL (recommended for large tables):
CREATE TABLE orders (id BIGSERIAL PRIMARY KEY);

-- GENERATED ALWAYS (SQL standard, PG 10+):
CREATE TABLE customers (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
-- Cannot insert explicit values (ALWAYS):
INSERT INTO customers (name) VALUES ('Rahul');  -- id auto-assigned
INSERT INTO customers (id, name) VALUES (999, 'Priya');  -- ERROR!

-- GENERATED BY DEFAULT (allows explicit override):
CREATE TABLE customers (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY
);
INSERT INTO customers (id, name) VALUES (999, 'Priya');  -- allowed

-- Current value, next value:
SELECT currval('products_id_seq');  -- current value (within session)
SELECT nextval('products_id_seq');  -- consume next value
SELECT lastval();                   -- last value used in this session
```

---

## 43.2 Sequence Behavior and Gaps

```sql
-- Sequences have gaps — this is NORMAL and EXPECTED:
-- Gap causes:
-- 1. Rolled back transactions: sequence advanced but INSERT rolled back → gap
-- 2. Caching: sequences cache nextval calls (default cache=1 per session)
-- 3. Restarts: server restart may lose cached values

-- Never assume sequential IDs. Never use ID gaps to detect deleted rows.

-- Adjust sequence behavior:
ALTER SEQUENCE orders_id_seq CACHE 10;    -- cache 10 values per session (faster)
ALTER SEQUENCE orders_id_seq CYCLE;       -- wrap around at max value (dangerous)
ALTER SEQUENCE orders_id_seq INCREMENT BY 10;  -- for multi-master setups
ALTER SEQUENCE orders_id_seq RESTART WITH 1000; -- reset to start from 1000

-- For globally unique IDs without sequence gaps → use UUID:
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    ...
);
-- Or PostgreSQL 13+:
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ...
);
```

---

# CHAPTER 44 — Concurrent DDL (Zero-Downtime Schema Changes)

---

## 44.1 The Problem with Schema Changes in Production

```sql
-- DANGEROUS: ALTER TABLE takes ACCESS EXCLUSIVE lock
ALTER TABLE orders ADD COLUMN notes TEXT NOT NULL DEFAULT '';
-- On a table with 100M rows:
-- 1. Acquires ACCESS EXCLUSIVE lock (blocks ALL reads AND writes)
-- 2. Rewrites every row to add the new column
-- 3. Total downtime: minutes to hours!

-- PostgreSQL 11+ is smarter about NOT NULL + DEFAULT:
-- Adding a column with a constant DEFAULT is now instant (no row rewrite)
-- BUT: adding NOT NULL still requires a CHECK
```

---

## 44.2 Safe Schema Change Patterns

```sql
-- Pattern 1: Add nullable column (always instant, no lock needed beyond brief moment)
ALTER TABLE orders ADD COLUMN notes TEXT;  -- instant even for 100M rows (PG 11+)

-- Pattern 2: Add NOT NULL column safely (3-step process)
-- Step 1: Add as nullable (instant)
ALTER TABLE orders ADD COLUMN priority INT;

-- Step 2: Backfill in batches (no long lock)
DO $$
DECLARE v_max_id BIGINT;
BEGIN
    SELECT MAX(id) INTO v_max_id FROM orders;
    FOR i IN 0..v_max_id BY 10000 LOOP
        UPDATE orders SET priority = 5
        WHERE id > i AND id <= i + 10000 AND priority IS NULL;
        PERFORM pg_sleep(0.01);  -- brief pause to not overwhelm
    END LOOP;
END $$;

-- Step 3: Add NOT NULL constraint (uses existing CHECK, no row scan in PG 12+)
ALTER TABLE orders ADD CONSTRAINT orders_priority_not_null CHECK (priority IS NOT NULL) NOT VALID;
VALIDATE CONSTRAINT orders_priority_not_null;  -- validates without full lock
ALTER TABLE orders ALTER COLUMN priority SET NOT NULL;  -- instant since CHECK exists
ALTER TABLE orders DROP CONSTRAINT orders_priority_not_null;

-- Pattern 3: Safe index creation
-- WRONG (blocks writes):
CREATE INDEX ix_orders_status ON orders(status);

-- CORRECT (allows writes during build):
CREATE INDEX CONCURRENTLY ix_orders_status ON orders(status);
-- Takes longer to build but never blocks queries

-- Pattern 4: Rename a column safely (requires application coordination)
-- Step 1: Add new column
ALTER TABLE orders ADD COLUMN customer_reference VARCHAR(100);
-- Step 2: Sync old to new with trigger
-- Step 3: Update application to write to both
-- Step 4: Backfill old → new
-- Step 5: Switch reads to new column
-- Step 6: Remove old column

-- Pattern 5: Dropping a column (instant, just marks as dropped)
ALTER TABLE orders DROP COLUMN notes;  -- fast, doesn't reclaim space immediately
-- Space reclaimed next time table is VACUUMED or rewritten
```

---

# CHAPTER 45 — Array Types and Operations

---

## 45.1 PostgreSQL Arrays

```sql
-- Array columns:
CREATE TABLE products (
    id      BIGSERIAL PRIMARY KEY,
    name    VARCHAR(255),
    tags    TEXT[],           -- array of text
    scores  INT[],            -- array of integers
    matrix  INT[][]           -- 2D array
);

-- Insert with arrays:
INSERT INTO products (name, tags, scores) VALUES
    ('Widget', ARRAY['electronics', 'wireless', 'sale'], ARRAY[5, 4, 5, 3]),
    ('Chair',  ARRAY['furniture', 'office'], ARRAY[4, 4, 3]),
    ('Cable',  ARRAY['electronics', 'accessories'], ARRAY[5, 5]);

-- OR use literal syntax:
INSERT INTO products (name, tags) VALUES ('Desk', '{"furniture","office","ergonomic"}');

-- Access elements (1-indexed!):
SELECT tags[1] FROM products WHERE name = 'Widget';  -- 'electronics'
SELECT tags[1:2] FROM products WHERE name = 'Widget'; -- {'electronics','wireless'}

-- Array functions:
SELECT array_length(tags, 1) FROM products;           -- length of dimension 1
SELECT array_append(tags, 'new_tag') FROM products;   -- add element
SELECT array_remove(tags, 'sale') FROM products;      -- remove element
SELECT array_cat(tags, ARRAY['extra']) FROM products; -- concatenate arrays
SELECT unnest(tags) FROM products WHERE name='Widget'; -- expand to rows
-- Returns: electronics, wireless, sale (one row each)

-- Aggregate into array:
SELECT ARRAY_AGG(name ORDER BY name) FROM products;   -- ['Cable', 'Chair', 'Widget']
SELECT STRING_AGG(name, ', ' ORDER BY name) FROM products;  -- 'Cable, Chair, Widget'

-- Array operators:
SELECT * FROM products WHERE tags @> ARRAY['electronics'];       -- contains 'electronics'
SELECT * FROM products WHERE tags && ARRAY['sale', 'furniture'];  -- overlaps with
SELECT * FROM products WHERE 'wireless' = ANY(tags);             -- element exists

-- GIN index for array containment:
CREATE INDEX ON products USING GIN(tags);
SELECT * FROM products WHERE tags @> ARRAY['electronics'];  -- uses GIN index ✅

-- Update array element:
UPDATE products SET tags = array_append(tags, 'featured') WHERE id = 1;
UPDATE products SET tags = array_remove(tags, 'sale') WHERE id = 1;
```

---

# CHAPTER 46 — JIT Compilation and Parallel Query Depth

---

## 46.1 JIT (Just-In-Time Compilation)

```sql
-- PostgreSQL 11+ includes LLVM-based JIT compilation
-- JIT compiles frequently-executed expression evaluation code to native machine code

-- Check JIT status:
SHOW jit;  -- on or off
SHOW jit_above_cost;          -- default: 100000 (enable JIT for queries > this cost)
SHOW jit_inline_above_cost;   -- default: 500000 (inline functions at this cost)
SHOW jit_optimize_above_cost; -- default: 500000 (run optimizations at this cost)

-- JIT helps: CPU-intensive queries (complex expressions, type casts, many WHERE conditions)
-- JIT hurts: short queries (JIT compilation overhead > execution savings)

-- Check if JIT was used in a query:
EXPLAIN (ANALYZE, FORMAT TEXT) SELECT ...;
-- Look for: "JIT: ..." section at bottom

-- Disable JIT for OLTP (where it hurts more than helps):
SET jit = off;  -- in session
-- In postgresql.conf: jit = off

-- Enable only for analytical queries:
SET jit = on;
SET jit_above_cost = 0;  -- enable for all queries (testing)
```

---

## 46.2 Parallel Query — Full Configuration

```sql
-- How many parallel workers are available:
SHOW max_worker_processes;           -- default: 8  (total worker process pool)
SHOW max_parallel_workers;           -- default: 8  (max for parallel queries)
SHOW max_parallel_workers_per_gather; -- default: 2  (per query)

-- Per-table override:
ALTER TABLE orders SET (parallel_workers = 4);  -- allow 4 workers for this table

-- Minimum size to trigger parallelism:
SHOW min_parallel_table_scan_size;  -- default: 8MB
SHOW min_parallel_index_scan_size;  -- default: 512kB

-- Force parallelism for testing:
SET max_parallel_workers_per_gather = 4;
SET parallel_tuple_cost = 0;          -- pretend parallel is free
SET parallel_setup_cost = 0;          -- pretend setup is free
SET min_parallel_table_scan_size = 0; -- parallelize even tiny tables

-- What can be parallelized:
-- ✅ Parallel Seq Scan
-- ✅ Parallel Index Scan
-- ✅ Parallel Hash Join
-- ✅ Parallel Aggregate (SUM, COUNT, AVG)
-- ✅ Parallel Sort (PG 10+)
-- ❌ Writes (INSERT, UPDATE, DELETE)
-- ❌ Functions marked PARALLEL UNSAFE
-- ❌ Subplans that cannot be parallelized

-- Mark your functions as parallel-safe:
CREATE FUNCTION my_function(...) RETURNS ...
LANGUAGE plpgsql
PARALLEL SAFE  -- or PARALLEL RESTRICTED or PARALLEL UNSAFE (default)
AS $$ ... $$;
```

---

## Chapter 34–46 Summary

| Feature | Critical Knowledge |
|---------|-------------------|
| **Window Functions** | Don't collapse rows; use PARTITION BY for groups; frame clause for moving windows |
| **PL/pgSQL** | Functions for reuse; procedures for transactions; triggers for automation |
| **Views** | Regular=transparent rewrite; Materialized=cached, needs REFRESH CONCURRENTLY |
| **JSONB** | @> for containment; ->> for text extraction; GIN index for queries |
| **Partitioning** | Range for time; List for categories; Hash for distribution; DROP partition for instant purge |
| **RLS** | Enable on table + create policy; SET LOCAL for transaction-scoped context |
| **Replication** | Physical=byte-for-byte WAL; Logical=table-level, different PG versions |
| **PgBouncer** | Transaction mode for web apps; session mode for full compatibility |
| **FTS** | tsvector + tsquery + GIN; setweight for field importance; ts_headline for excerpts |
| **Sequences** | Gaps are normal; GENERATED ALWAYS is modern standard; gen_random_uuid() for UUIDs |
| **Concurrent DDL** | CREATE INDEX CONCURRENTLY; ADD COLUMN with DEFAULT is instant in PG11+ |
| **Arrays** | @> containment; ANY() membership; GIN index; unnest() to rows |

---

# PART XI — DATABASE SYSTEM DESIGN

> System design is not about knowing which tool to pick.
> It is about understanding WHY each trade-off exists,
> so you can reason from first principles when you face a new problem.

---

# CHAPTER 47 — How to Approach Database System Design

---

## 47.1 The Framework

Every database system design problem follows the same structure:

```
Step 1: CLARIFY REQUIREMENTS (5 minutes)
  Functional: What does the system DO?
    → What gets stored? What gets read? What gets written?
    → What queries does the system answer?

  Non-functional: What are the constraints?
    → Scale: how many users? reads/sec? writes/sec?
    → Latency: p99 < 100ms? < 10ms? < 1 second acceptable?
    → Availability: 99.9% (8.7 hours/year downtime)? 99.99%?
    → Consistency: must reads always see latest write?
    → Durability: can we lose any data?

Step 2: ESTIMATE SCALE (5 minutes)
  → Daily active users × requests per user = requests/day
  → Requests/day ÷ 86,400 = average requests/second
  → Peak = average × 5-10× (rule of thumb)
  → Storage: entities × size × retention period

Step 3: HIGH-LEVEL DESIGN (10 minutes)
  → Core entities and relationships (sketch schema)
  → Data flow: write path and read path
  → Choose database type(s)

Step 4: DEEP DIVE (20 minutes)
  → Scaling strategy (partitioning, replication)
  → Indexing strategy
  → Caching strategy
  → Handling hot spots, failures, edge cases

Step 5: TRADE-OFFS (5 minutes)
  → What did you sacrifice? Why was it worth it?
```

---

## 47.2 Scale Estimation — The Numbers That Drive Decisions

```
Key thresholds to know:

Single PostgreSQL server handles:
  → Reads:  10,000–50,000 queries/second (simple indexed lookups)
  → Writes: 5,000–20,000 writes/second
  → Storage: up to ~10TB (depends on disk)
  → Connections: 100–500 (with PgBouncer: thousands of app connections)

When you need to scale out:
  → > 50,000 reads/second → read replicas or caching
  → > 20,000 writes/second → partitioning or sharding
  → > 10TB → partitioning by time/tenant
  → > 10M rows in a table that grows fast → partitioning

Memory: 1 billion rows × 100 bytes = 100GB
Network: 1Gbps = 125MB/s ≈ 1.25M rows/second transferred
Disk: SSD random read = 0.1ms, HDD random read = 5ms
  SSD sequential read = 500MB/s, HDD sequential = 100MB/s
```

---

# CHAPTER 48 — CAP Theorem and Consistency Models

---

## 48.1 CAP Theorem

CAP stands for **Consistency, Availability, Partition Tolerance**.

```
Consistency (C):
  Every read sees the most recent write (or an error, not stale data)
  "All nodes see the same data at the same time"

Availability (A):
  Every request receives a response (not an error), though may not be latest
  "System is always up and responds"

Partition Tolerance (P):
  System continues despite network partitions (some messages between nodes are lost)
  "System survives network splits between nodes"

The CAP theorem (Eric Brewer, 2000):
  A distributed system can guarantee AT MOST TWO of the three simultaneously.
  In practice: network partitions happen and cannot be avoided.
  So the real choice is:
    → CP: Consistency + Partition Tolerance (sacrifice availability during partition)
    → AP: Availability + Partition Tolerance (sacrifice consistency during partition)
```

**Real systems:**
```
CP systems (consistency preferred):
  → PostgreSQL (synchronous replication + failover)
  → HBase, Zookeeper, etcd
  → During partition: system returns errors rather than stale data

AP systems (availability preferred):
  → Cassandra, CouchDB, DynamoDB
  → During partition: system returns stale data rather than errors
  → Eventually consistent: all replicas converge after partition heals

CA systems (not really a thing in distributed context):
  → Single-node PostgreSQL: no partitions since one machine
  → Once you go distributed, you always have P
```

---

## 48.2 Consistency Models (Spectrum)

From strongest to weakest:

```
LINEARIZABILITY (Strict Consistency):
  Every operation appears instantaneous at some point between call and return.
  Reads always see the latest write.
  Cost: high latency (must coordinate all nodes before returning)
  Examples: single-node systems, etcd, ZooKeeper

SEQUENTIAL CONSISTENCY:
  All operations appear to happen in some sequential order.
  Order is consistent across all nodes but may not match real time.

CAUSAL CONSISTENCY:
  Causally related operations are seen in order.
  "If A causes B, everyone sees A before B"
  Concurrent (unrelated) operations may be seen in different orders.

EVENTUAL CONSISTENCY:
  If no new writes happen, all replicas will eventually converge.
  No guarantee on WHEN.
  Reads may see stale data.
  Examples: DNS, Cassandra, DynamoDB (without strong consistency mode)

MONOTONIC READ CONSISTENCY:
  Once you read a value, you never see an older value.
  (You might see the same or newer, never older)

READ-YOUR-WRITES CONSISTENCY:
  After you write a value, you always see it when you read.
  (Others might still see old value)
  Common implementation: route a user's reads to the same replica they wrote to,
  or route writes and subsequent reads to the primary.
```

---

## 48.3 ACID vs BASE

```
ACID (traditional RDBMS):
  Atomicity, Consistency, Isolation, Durability
  → Strong guarantees
  → Single-node or tightly coupled cluster
  → PostgreSQL, MySQL, Oracle

BASE (distributed/NoSQL systems):
  Basically Available, Soft state, Eventually consistent
  → Weak guarantees in exchange for scale and availability
  → Cassandra, DynamoDB, CouchDB

The real world uses both:
  Financial transactions: ACID (correctness critical)
  User session data: BASE (stale session data acceptable)
  Product catalog reads: BASE (1-second stale is fine)
  Inventory stock: ACID (overselling is a real business problem)
```

---

# CHAPTER 49 — Database Selection Guide

---

## 49.1 Choosing the Right Database

```
QUESTION 1: What is your data's primary structure?
  → Tables with relationships → Relational (PostgreSQL)
  → Self-contained documents  → Document (MongoDB) or PostgreSQL JSONB
  → Key → Value pairs         → Redis (in-memory) or DynamoDB
  → Graph relationships       → Neo4j or recursive CTEs in PostgreSQL
  → Time-series measurements  → TimescaleDB (PostgreSQL extension) or InfluxDB
  → Wide-column, sparse       → Cassandra, HBase

QUESTION 2: What is your read pattern?
  → Mostly point lookups by key → Any DB with good indexing
  → Complex joins across entities → Relational (PostgreSQL)
  → Full-text search → PostgreSQL (FTS) or Elasticsearch
  → Aggregations on billions of rows → BigQuery, Redshift, Snowflake
  → Nearest neighbors / geographic → PostGIS or Elasticsearch geo

QUESTION 3: What is your write pattern?
  → Low volume, complex transactions → PostgreSQL
  → High volume, simple writes → Cassandra, DynamoDB
  → Append-only, time-ordered → TimescaleDB, InfluxDB
  → Immutable event log → Kafka + S3/Parquet

QUESTION 4: What are your consistency requirements?
  → Strong consistency required → PostgreSQL, MySQL
  → Eventual consistency acceptable → Cassandra, DynamoDB
  → Mixed (strong for some, eventual for others) → Multiple databases

QUESTION 5: What is your scale?
  → < 100GB, < 10k req/s → Single PostgreSQL + replicas
  → 100GB–10TB, < 100k req/s → PostgreSQL + read replicas + partitioning
  → > 10TB, > 100k req/s → Sharding, or purpose-built distributed DB
```

---

## 49.2 The Polyglot Persistence Pattern

Large systems use multiple databases, each chosen for specific use cases:

```
E-commerce system example:

  Product Catalog:     PostgreSQL (complex queries, consistency needed)
  User Sessions:       Redis (fast, ephemeral, TTL support)
  Shopping Cart:       Redis (temporary, per-user, fast)
  Order History:       PostgreSQL (ACID, relational, complex reports)
  Product Reviews:     MongoDB (variable schema, nested structure)
  Search:              Elasticsearch (full-text, faceted search)
  Analytics:           BigQuery (aggregate queries on billions of rows)
  Real-time Events:    Kafka (event streaming, CDC)
  Recommendations:     Redis + ML model
  Media Files:         S3/GCS (object storage, not a database)

The application layer:
  → Writes to PostgreSQL (source of truth)
  → Syncs to other systems via CDC (Change Data Capture) or event streaming
  → Each system optimized for its specific access pattern
```

---

# CHAPTER 50 — Scaling Reads: Read Replicas and Caching

---

## 50.1 Read Replicas

```
When to add read replicas:
  → Primary CPU usage > 70% due to SELECT queries
  → Analytics queries slowing down OLTP performance
  → Need to scale read throughput beyond single node capacity

Architecture:
  Application → Load Balancer
                    ├── Writes → PRIMARY PostgreSQL
                    └── Reads  → REPLICA 1
                                 REPLICA 2
                                 REPLICA 3

Consistency trade-off:
  Async replication: replicas lag behind primary (typical: < 100ms, can be seconds)
  → User writes order, immediately reads order list → may not see new order yet!

Solutions for read-your-writes:
  1. Route read to PRIMARY for 1 second after write
  2. Include write timestamp, route reads to primary until replica catches up
  3. "Sticky" routing: route user's reads to same replica until lag is acceptable
  4. Application tracks replication LSN and waits: SELECT pg_wal_replay_wait(lsn, 5000)
```

---

## 50.2 Caching Strategies

```
CACHE ASIDE (Lazy Loading) — most common:
  Read:
    1. Check cache (Redis)
    2. Cache hit → return (fast, ~0.1ms)
    3. Cache miss → read from DB → store in cache → return
  Write:
    → Invalidate cache key (or update it)

  Pro: Only caches what's actually requested
  Con: Cache miss causes DB read (latency spike for first request)

WRITE-THROUGH:
  Write:
    1. Write to cache
    2. Write to DB (both updated synchronously)
  Read:
    → Always in cache

  Pro: Cache always fresh
  Con: Write latency = cache write + DB write; cache fills with rarely-read data

WRITE-BEHIND (Write-Back):
  Write:
    1. Write to cache immediately (return to user)
    2. Asynchronously batch-write to DB
  Read:
    → Always in cache

  Pro: Very fast writes
  Con: Risk of data loss if cache fails before DB write

READ-THROUGH:
  Read:
    1. Always go through cache layer
    2. Cache layer reads from DB on miss (abstracted from application)

TTL (Time to Live):
  Set expiry on cached values
  Trade-off: short TTL = fresher data, more DB hits; long TTL = staler data, fewer hits
```

---

## 50.3 Cache Invalidation Strategies

```
Key-based invalidation:
  Invalidate specific cache key when data changes
  cache.delete(f"product:{product_id}")  → simple, precise

Tag-based invalidation:
  Tag cache entries with related identifiers
  cache entries for product 5 tagged with "products" and "product:5"
  Invalidate all "products" when any product changes

Time-based (TTL):
  Let cache expire naturally
  Acceptable when slight staleness is OK

Event-driven (CDC):
  Database change → event → cache invalidation
  Most consistent but complex infrastructure

Cache Stampede / Thundering Herd:
  Problem: popular cache key expires → thousands of requests miss simultaneously
           → all hit DB at same time → DB overwhelmed

  Solutions:
  1. Probabilistic early expiry: before TTL expires, randomly start refreshing
  2. Lock: first miss acquires lock, refreshes; others wait
  3. Stale-while-revalidate: serve stale while refreshing in background
```

---

# CHAPTER 51 — Sharding and Horizontal Scaling

---

## 51.1 WHY Sharding

```
When single-server limits are hit:
  → > 10TB data → disk full or too slow
  → > 20k writes/second → single node bottleneck
  → Specific tenant/user consumes too much resource → "noisy neighbor"

Sharding = split data across multiple independent database servers
Each server (shard) holds a subset of the total data
```

---

## 51.2 Sharding Strategies

### Range-Based Sharding

```
Shard by a range of values:
  Shard 1: customer_id 1–1,000,000
  Shard 2: customer_id 1,000,001–2,000,000
  Shard 3: customer_id 2,000,001–3,000,000

Pros:
  → Range queries stay on one shard (all orders for customer 5000)
  → Easy to add new shard at the high end

Cons:
  → HOT SPOTS: if most traffic is recent users → shard 3 is overloaded
  → Rebalancing when shard grows too large
```

### Hash-Based Sharding

```
shard_id = hash(customer_id) % number_of_shards

  customer_id=5:   hash(5) % 4 = 1 → Shard 1
  customer_id=100: hash(100) % 4 = 0 → Shard 0
  customer_id=101: hash(101) % 4 = 1 → Shard 1

Pros:
  → Even distribution (no hot spots)
  → No range-based imbalance

Cons:
  → Range queries span ALL shards (must query all shards and merge)
  → Adding/removing shards requires rehashing all data
  → Solution: Consistent Hashing (see below)
```

### Consistent Hashing

```
Problem with modulo hashing:
  With 4 shards: hash(key) % 4
  Add a 5th shard: hash(key) % 5  → ~80% of keys remapped → massive migration

Consistent hashing solution:
  Place shards on a "ring" of 0–2^32 positions
  Each shard covers a range of the ring
  Each key hashes to a position on the ring → goes to next shard clockwise

  Adding shard 5: only keys between shard 4 and shard 5 move
  Removing shard 2: only its keys move to shard 3
  Typical: only 1/n keys remapped (1/5 = 20% when adding 5th to 4)

Virtual nodes:
  Each physical shard gets multiple positions on the ring (e.g., 150 positions)
  Ensures even distribution even with uneven key distribution
  Used by: Cassandra, Amazon DynamoDB, Riak
```

---

## 51.3 What to Shard By

```
The shard key is the most critical decision. Rules:

1. High cardinality: many distinct values so data distributes evenly
   Good: user_id (millions of users)
   Bad:  country (only ~200 countries → uneven distribution)

2. Even distribution: no hot spots
   Good: user_id (if users are similarly active)
   Bad:  celebrity user_id (millions of followers → this shard overloaded)

3. Queries stay on one shard when possible
   Good: customer_id (all orders for a customer are on one shard)
   Bad:  product_id for order_items (orders join across products → cross-shard join)

4. Immutable once assigned: changing shard key requires data migration
   Good: user_id (users don't change ID)
   Bad:  user_email (users change email addresses)

Common shard keys:
  Social platform:      user_id
  E-commerce:           customer_id (for orders), product_id (for catalog)
  Multi-tenant SaaS:    tenant_id (all tenant data on same shard)
  IoT:                  device_id
  Time-series:          time (range sharding by time period)
```

---

## 51.4 Cross-Shard Queries — The Hard Problem

```
Query: "Find total revenue across all customers last month"
  Sharded by customer_id → data spread across all shards

Option 1: Scatter-Gather
  → Send query to ALL shards in parallel
  → Collect results
  → Merge and sort at coordinator
  Cost: O(number_of_shards) queries per cross-shard query
  Works for: aggregations (SUM, COUNT), results that can be merged

Option 2: Denormalization
  → Keep pre-computed totals per shard
  → Shard coordinator aggregates shard totals
  Works for: frequent aggregation queries

Option 3: Dedicated Analytics Database
  → Keep OLTP data sharded (for writes)
  → ETL to a central analytics DB (for complex cross-shard queries)
  → Best for read-heavy analytics

Option 4: Avoid cross-shard joins by co-locating related data
  → If orders always access customer data → shard both by customer_id
  → All data for one customer on same shard → no cross-shard join needed
```

---

# CHAPTER 52 — Distributed Transactions

---

## 52.1 The Problem

In a sharded or microservices system, a single business operation
may touch multiple databases. How do you ensure ACID across them?

```
E-commerce order placement:
  → Deduct stock from Products DB (shard 2)
  → Create order in Orders DB (shard 1)
  → Charge customer in Payments DB (external service)

Problem: What if step 2 succeeds but step 3 fails?
  → Order exists without payment charged → business logic violated
  → How do you roll back step 1 and 2 after step 3 fails?

This is the distributed transaction problem.
```

---

## 52.2 Two-Phase Commit (2PC)

```
Phase 1 — PREPARE:
  Coordinator → Shard 1: "Prepare to commit your part"
  Coordinator → Shard 2: "Prepare to commit your part"
  Coordinator → Payment: "Prepare to commit your part"

  Each participant:
    → Writes changes to WAL
    → Acquires all locks needed
    → Responds: PREPARED (ready) or ABORT (cannot proceed)

Phase 2 — COMMIT or ABORT:
  If ALL responded PREPARED:
    Coordinator → ALL: "COMMIT"
    → Each participant commits and releases locks

  If ANY responded ABORT:
    Coordinator → ALL: "ABORT"
    → Each participant rolls back and releases locks

Problems with 2PC:
  1. BLOCKING: if coordinator crashes after PREPARE but before COMMIT
     → Participants are locked, waiting indefinitely
  2. SLOW: 2 network round trips for every transaction
  3. SINGLE POINT OF FAILURE: coordinator must be highly available

PostgreSQL supports 2PC with PREPARE TRANSACTION / COMMIT PREPARED:
  BEGIN;
  ... do work ...
  PREPARE TRANSACTION 'txn-unique-id';
  -- At this point: transaction is prepared but not committed
  -- Another process can COMMIT PREPARED 'txn-unique-id' or ROLLBACK PREPARED 'txn-unique-id'
```

---

## 52.3 SAGA Pattern — Avoiding Distributed Transactions

The SAGA pattern breaks a distributed transaction into a sequence of local transactions,
each with a **compensating transaction** that undoes its effect if something fails later.

```
Order placement SAGA:

Step 1: Reserve stock (Products DB)
  → Success: emit "StockReserved" event
  → Failure: N/A (nothing to compensate)

Step 2: Create order (Orders DB)
  → Success: emit "OrderCreated" event
  → Failure → compensate: release stock reservation

Step 3: Charge payment (Payments Service)
  → Success: emit "PaymentCharged" event
  → Failure → compensate:
    Cancel order (Orders DB)
    Release stock reservation (Products DB)

Step 4: Confirm order (Orders DB)
  → Success: emit "OrderConfirmed"
  → Failure → compensate:
    Refund payment
    Cancel order
    Release stock

If step 3 fails:
  SAGA runs compensation:
    → Rollback step 2: cancel order
    → Rollback step 1: release stock reservation
  Final state: clean, consistent (no order, no charge, stock restored)
```

### SAGA Implementation Patterns

```
Choreography (event-driven):
  Each service listens for events and reacts
  No central coordinator
  Simple but hard to track overall status

  Products DB: listens for "OrderPlaced" → reserves stock → emits "StockReserved"
  Orders DB: listens for "StockReserved" → creates order → emits "OrderCreated"
  Payments: listens for "OrderCreated" → charges → emits "PaymentCharged"

Orchestration (centralized):
  A SAGA orchestrator tells each service what to do
  Easier to monitor and debug
  Single orchestrator is a coordination point

  OrderSaga orchestrator:
    1. Call ProductsService.reserveStock()
    2. Call OrdersService.createOrder()
    3. Call PaymentsService.charge()
    4. On failure: call compensating transactions in reverse order
```

---

## 52.4 The Outbox Pattern — Reliable Event Publishing

```
Problem: "Save to DB AND publish event" is NOT atomic:
  BEGIN;
  INSERT INTO orders ...;  ← database write
  COMMIT;
  publish("OrderCreated", order);  ← what if this fails? Order saved but event lost!

Solution: The Outbox Pattern
  Use a transactional outbox table in the SAME database:

  BEGIN;
  INSERT INTO orders (id, customer_id, total) VALUES (...);
  INSERT INTO outbox (event_type, payload, created_at)
      VALUES ('OrderCreated', '{"order_id": ...}', NOW());
  COMMIT;
  -- Both inserts are atomic! Event cannot be lost.

  Separate publisher process:
    → Polls outbox table for unpublished events
    → Publishes to message queue (Kafka, RabbitMQ)
    → Marks events as published

  OR use Debezium (CDC tool):
    → Watches PostgreSQL WAL for changes to outbox table
    → Automatically publishes to Kafka
    → No polling needed
```

---

# CHAPTER 53 — Change Data Capture (CDC)

---

## 53.1 What CDC Is

Change Data Capture streams every INSERT, UPDATE, DELETE from the database
as a real-time event stream, without modifying the application.

```
PostgreSQL WAL → Debezium (CDC connector) → Kafka
                                                ↓
                                     Search Index (Elasticsearch)
                                     Cache Invalidation (Redis)
                                     Data Warehouse (BigQuery)
                                     Other microservices
```

---

## 53.2 Logical Decoding for CDC

```sql
-- PostgreSQL exposes WAL as a logical change stream:
-- postgresql.conf: wal_level = logical

-- Create a replication slot:
SELECT pg_create_logical_replication_slot('my_cdc_slot', 'wal2json');

-- Read changes (for testing):
SELECT * FROM pg_logical_slot_get_changes('my_cdc_slot', NULL, NULL);
-- Returns JSON describing each INSERT/UPDATE/DELETE

-- In production: Debezium connects as a logical replication client
-- Configuration in Debezium connector:
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "database.hostname": "localhost",
  "database.port": "5432",
  "database.user": "replicator",
  "database.dbname": "mydb",
  "database.server.name": "myserver",
  "table.include.list": "public.orders,public.products",
  "plugin.name": "wal2json"
}

-- Result: every change produces a Kafka message:
{
  "op": "u",                    -- u=update, c=create, d=delete, r=read
  "before": {"id": 1, "price": 99.99},
  "after":  {"id": 1, "price": 149.99},
  "source": {"table": "products", "lsn": "0/3A12B410"}
}
```

---

## 53.3 Event Sourcing

Instead of storing the CURRENT STATE, store every EVENT that led to that state.

```
Traditional (current state):
  orders table: id=1, status='completed', total=299.99

Event sourcing (event log):
  order_events:
    {order_id:1, event:'OrderPlaced',   data:{customer:5, items:[...]}}
    {order_id:1, event:'PaymentMade',   data:{amount:299.99}}
    {order_id:1, event:'OrderShipped',  data:{tracking:'FDX123'}}
    {order_id:1, event:'OrderDelivered',data:{at:'2024-01-20'}}

Current state = replay of all events for that entity
```

**Benefits:**
- Complete audit history built-in (every state is recorded)
- Can replay to rebuild derived views (CQRS)
- Time travel: "what did order 1 look like on Jan 15?"
- Easy to debug: see every step

**Costs:**
- Reading current state requires replaying all events (use snapshots for optimization)
- More complex queries
- Event schema evolution is hard

---

## 53.4 CQRS — Command Query Responsibility Segregation

```
Separate read model (QUERY) from write model (COMMAND):

WRITE SIDE:
  Application → Command (PlaceOrder)
              → Domain logic validates
              → Write to event store (PostgreSQL: order_events table)
              → Publish event to message bus

READ SIDE:
  Event consumer → reads OrderPlaced event
                 → updates read model (denormalized view)
                 → e.g., Redis cache, Elasticsearch, separate PostgreSQL read DB

Query:
  Application → Read from read model (fast, denormalized, perfect for UI)

Benefits:
  Write side: normalized, consistent, easy to reason about
  Read side: optimized per query (each API endpoint has perfect schema)
  Scale independently: many read replicas, fewer write nodes
  Read model can be rebuilt from events anytime

Example read models from the same write event stream:
  → "Order list" view: PostgreSQL table (order_id, customer, total, status, date)
  → "Search" view: Elasticsearch (full-text on customer name, order notes)
  → "Analytics" view: BigQuery (denormalized, partitioned by date)
  → "Dashboard cache" view: Redis (pre-computed stats)
```

---

# CHAPTER 54 — Hot Spots and Skew

---

## 54.1 What is a Hot Spot

A hot spot is a shard, partition, or node that receives disproportionately
more traffic than others, becoming the bottleneck.

```
Causes:
  1. CELEBRITY PROBLEM: One user has 10M followers → every follower read hits same shard
  2. TIME SKEW: Sharding by time + all writes are "now" → newest shard gets all writes
  3. HASH COLLISION: Hash function clusters many popular keys to same shard
  4. HOT KEY IN CACHE: One Redis key (e.g., homepage) gets millions of reads

Detection:
  → Monitor CPU/QPS per shard: one shard at 90%, others at 10% → hot spot
  → pg_stat_user_tables: one table has 100× more sequential scans than others
```

---

## 54.2 Solving Hot Spots

```
1. Add salt to the shard key (spread one logical key across multiple shards):
   Instead of: shard = hash(user_id) % 4
   Use:        shard = hash(user_id + random_suffix(1..10)) % 4
   → One popular user's data spread across 10 shards (10 machines share the load)
   → Reads must query all 10 shards and merge (scatter-gather)

2. Cache the hot key with multiple copies:
   Instead of: GET "homepage_data" from Redis (one key, one node)
   Use:        key = f"homepage_data_{random.randint(1,100)}"
   → 100 copies of the same data → 100 Redis nodes share the load

3. Read replicas for hot rows:
   Route ALL reads for celebrity user_id=1 to a dedicated read replica
   Other users route to the main cluster

4. Denormalize to avoid the hot join:
   Instead of: JOIN to famous user's profile on every read
   Store: embedded user snapshot in each dependent record
   Trade-off: slightly stale data, but no hot join

5. Asynchronous write aggregation:
   Don't write every "view" immediately to DB
   Buffer in Redis counter → periodically flush to DB in batch
   Trade-off: view count is approximate
```

---

# CHAPTER 55 — Connection Management at Scale

---

## 55.1 The C10K Problem for Databases

```
Problem: 10,000 concurrent application instances each want 10 DB connections
         = 100,000 PostgreSQL backend processes
         = 100,000 × 5MB RAM = 500GB RAM just for connections
         + Process scheduling overhead
         + Shared buffer contention

Real numbers for a PostgreSQL server:
  Comfortable: 100–300 connections
  Possible:    500–1000 connections (with enough RAM)
  Pain zone:   > 1000 connections (performance degrades)
```

---

## 55.2 Connection Pooling Architecture

```
Application Tier (1000 instances, 10 threads each = 10,000 app threads)
          ↓
PgBouncer (transaction mode, 1 process per server)
  max_client_conn = 50,000  (accepts 50k app connections)
  default_pool_size = 50    (maintains 50 real PostgreSQL connections)
          ↓
PostgreSQL (50 backend processes = 50 × 5MB = 250MB overhead)

How it works:
  10,000 app threads each hold a PgBouncer "connection" (just a file descriptor)
  Only 50 of them are actively executing SQL at any time
  PgBouncer queues others until a server connection is free
  Average wait: microseconds to milliseconds
  Result: PostgreSQL processes 50k req/s with only 50 connections
```

---

# CHAPTER 56 — Database System Design: Common Interview Problems

---

## 56.1 Design: URL Shortener (e.g., bit.ly)

```
Requirements:
  Write: create short URL → long URL mapping
  Read:  short URL → redirect to long URL
  Scale: 100M URLs, 10B redirects/month

Scale estimate:
  Writes: 100M URLs / (5 years × 365 × 86400) = ~0.6 writes/second (tiny)
  Reads: 10B / (30 × 86400) = ~3,900 reads/second
  Read:Write ratio = 6,500:1 → heavily read-dominant

Database schema:
  CREATE TABLE urls (
      short_code  CHAR(7) PRIMARY KEY,   -- 7-char base62 code
      long_url    TEXT NOT NULL,
      created_at  TIMESTAMPTZ DEFAULT NOW(),
      expires_at  TIMESTAMPTZ,
      click_count BIGINT DEFAULT 0
  );
  CREATE INDEX ON urls(long_url);  -- for "has this URL been shortened before?"

Short code generation:
  Option 1: Hash (MD5/SHA256 of long_url), take first 7 chars
    → Collisions possible (birthday problem)
    → Check DB before returning
  Option 2: Auto-increment ID converted to base62
    id=1 → '0000001', id=12345 → '0003lp'
    → No collisions, monotonically increasing
    → But: sequential codes are guessable

Scaling reads (3,900 req/s — single PostgreSQL handles this easily):
  → Add Redis cache: short_code → long_url, TTL=24h
  → Cache hit rate ~99%: <100ms redirect
  → Cache miss: read from PostgreSQL, store in cache

Click counting (without database bottleneck):
  → Don't UPDATE click_count on every redirect (write hotspot)
  → Use Redis INCR: INCR "clicks:{short_code}"
  → Periodically batch-write to PostgreSQL
```

---

## 56.2 Design: Instagram / Photo Feed

```
Key tables:
  users:    id, username, bio, follower_count, following_count
  posts:    id, user_id, image_url, caption, created_at
  follows:  follower_id, followee_id, created_at (PK: both cols)
  likes:    user_id, post_id, created_at (PK: both cols)
  comments: id, post_id, user_id, body, created_at

Home feed challenge ("get posts from people I follow, ordered by time"):
  SELECT p.*
  FROM posts p
  JOIN follows f ON f.followee_id = p.user_id
  WHERE f.follower_id = 5
  ORDER BY p.created_at DESC
  LIMIT 20;

  This JOIN can be expensive if user follows 1000 people
  and each has hundreds of posts → scanning 100,000+ rows

Approaches to feed generation:

  PULL model (fan-out on read):
    → Query at read time (above SQL)
    → Simple, consistent (always fresh)
    → Slow for users who follow many people (celebrity followers)

  PUSH model (fan-out on write, pre-computed feed):
    → When user A posts → write to feed of ALL A's followers
    → Feed table: user_id | post_id | created_at
    → Read feed: SELECT FROM feed WHERE user_id=5 ORDER BY created_at DESC
    → Fast read: O(1) indexed lookup
    → Slow write: user with 10M followers → 10M inserts per post!
    → Celebrity problem: can't use push for celebrities

  HYBRID (used by Instagram, Twitter):
    → Regular users: push model (pre-compute feed)
    → Celebrities (> N followers): pull model (compute at read time)
    → Merge results at API layer
```

---

## 56.3 Design: Ride-Sharing (Uber/Ola) — Location Tracking

```
Challenge: Track 1M drivers sending GPS location every 4 seconds
  Write rate: 1,000,000 / 4 = 250,000 location updates/second
  This is WAY beyond single PostgreSQL (max ~20k writes/second)

Schema for driver locations:
  PostgreSQL is NOT the right tool for the write path

  Better approach: Redis for current location
    GEOADD "drivers" longitude latitude driver_id
    → GEORADIUS "drivers" lng lat 5km ASC COUNT 10  -- nearest drivers

  PostgreSQL for ride history (lower write rate):
    CREATE TABLE ride_locations (
        ride_id    BIGINT,
        driver_id  BIGINT,
        longitude  DECIMAL(10,7),
        latitude   DECIMAL(10,7),
        recorded_at TIMESTAMPTZ
    ) PARTITION BY RANGE (recorded_at);
    -- Time-partitioned: each day is a separate partition

  Architecture:
    Driver app → Kafka (250k writes/second)
              → Consumer 1: update Redis GEOADD (current location)
              → Consumer 2: batch write to PostgreSQL (historical)

Matching algorithm:
  1. Rider requests → API server
  2. GEORADIUS from rider location → nearest 10 available drivers (Redis, ~1ms)
  3. Send ride request to each driver in order
  4. First to accept → matched
  5. Write ride record to PostgreSQL (PostgreSQL can handle this rate: ~100 rides/sec)
```

---

## 56.4 Design: Notification System

```
Requirements:
  → Send push/email/SMS notifications to users
  → 100M users, ~10 notifications/user/day = 1B notifications/day
  → Must be reliable: no duplicate, no lost notifications
  → Some notifications are time-sensitive (<1 second), some are best-effort

Schema:
  CREATE TABLE notifications (
      id           BIGSERIAL PRIMARY KEY,
      user_id      BIGINT NOT NULL,
      type         VARCHAR(50) NOT NULL,  -- push, email, sms
      title        VARCHAR(255),
      body         TEXT,
      status       VARCHAR(20) DEFAULT 'pending', -- pending, sent, failed
      scheduled_at TIMESTAMPTZ,
      sent_at      TIMESTAMPTZ,
      created_at   TIMESTAMPTZ DEFAULT NOW()
  ) PARTITION BY RANGE (created_at);

  CREATE INDEX ON notifications(status, scheduled_at)
    WHERE status = 'pending';  -- partial index on pending notifications only

Worker pattern (competing consumers with SKIP LOCKED):
  Multiple workers each run:
    BEGIN;
    SELECT id, user_id, type, body
    FROM notifications
    WHERE status = 'pending'
      AND scheduled_at <= NOW()
    ORDER BY scheduled_at
    LIMIT 10
    FOR UPDATE SKIP LOCKED;  -- each worker gets different rows, no overlap

    -- Process each notification
    -- Call push/email/SMS API

    UPDATE notifications SET status = 'sent', sent_at = NOW()
    WHERE id IN (...);
    COMMIT;

Rate limiting: don't send too many to one user:
  Redis: INCR "notif_count:{user_id}:{hour}" + EXPIRE 3600
  If count > threshold: schedule for next window
```

---

## 56.5 Design: Leaderboard System (Gaming)

```
Requirements:
  → 10M players, scores updated in real-time
  → Instant leaderboard: top 100, rank of specific player
  → Weekly and all-time leaderboards

Why not PostgreSQL for real-time rank?
  SELECT RANK() OVER (ORDER BY score DESC) FROM scores WHERE player_id = 5;
  → Must scan/sort ALL 10M rows for every rank query → slow

Redis Sorted Set:
  ZADD leaderboard:weekly score player_id     -- O(log n) update
  ZREVRANK leaderboard:weekly player_id       -- O(log n) rank lookup
  ZREVRANGE leaderboard:weekly 0 99 WITHSCORES -- O(log n + 100) top 100

  10M players in Redis sorted set:
  → Rank query: ~0.1ms (O(log 10M) = 23 operations)
  → Score update: ~0.1ms
  → Top 100: ~0.1ms

PostgreSQL for persistence and complex analytics:
  → Nightly sync: Redis → PostgreSQL for permanent storage
  → Historical analysis: "was player X ever in top 100?"
  → PostgreSQL leaderboard query with proper indexes:
    CREATE INDEX ON scores(score DESC);
    -- For top-N: fast (just read first N from index)
    -- For specific player rank: slow (must count all higher scores)

Hybrid:
  Write path: → Redis ZADD (fast, in-memory)
  Read path:  → Redis ZREVRANK (fast rank), ZREVRANGE (fast top-N)
  Persistence: → Async write to PostgreSQL
  Analytics:   → PostgreSQL queries
```

---

# CHAPTER 57 — Distributed System Design Patterns Summary

---

## 57.1 Data Patterns Quick Reference

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PROBLEM                    │  PATTERN               │  TOOLS               │
├──────────────────────────────────────────────────────────────────────────────┤
│  Too many DB reads          │  Cache aside            │  Redis + PostgreSQL  │
│  Stale cache after write    │  Write-through or CDC   │  Redis + Debezium    │
│  Read replicas lag          │  Read-your-writes       │  Sticky routing      │
│  Write bottleneck           │  Sharding               │  Citus, Vitess       │
│  Hot shard                  │  Consistent hashing     │  Application layer   │
│  Distributed transaction    │  SAGA pattern           │  Application + Kafka │
│  Event publishing + DB      │  Outbox pattern         │  PostgreSQL + Kafka  │
│  Rebuild derived views      │  Event sourcing + CQRS  │  Kafka + consumers   │
│  Real-time change sync      │  CDC                    │  Debezium + Kafka    │
│  Competing job workers      │  SKIP LOCKED queue      │  PostgreSQL          │
│  Celebrity hot spot         │  Hybrid push/pull       │  Application logic   │
│  Realtime ranking           │  Redis sorted set       │  Redis               │
│  Geographic search          │  PostGIS / Redis GEO    │  PostGIS, Redis      │
│  Full-text search           │  Inverted index         │  PostgreSQL FTS,     │
│                             │                         │  Elasticsearch       │
│  Time-series at scale       │  Partitioning + BRIN    │  TimescaleDB         │
│  Massive write volume       │  LSM-Tree engine        │  Cassandra, RocksDB  │
│  Analytics on big data      │  Column storage         │  BigQuery, Redshift  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 57.2 Database Selection Quick Reference

```
PostgreSQL → Default choice for relational data, JSONB, full-text search,
             geospatial, moderate scale. The Swiss Army knife.

Redis →      Caching, sessions, counters, rate limiting, queues,
             pub/sub, sorted sets (leaderboards), geo queries.
             In-memory: fast but RAM-limited.

Cassandra → Write-heavy, time-series, wide-column, multi-datacenter,
             no single point of failure. Eventual consistency.
             Poor for: complex queries, transactions.

MongoDB →    Flexible document schema, nested data.
             Decent choice when PostgreSQL JSONB is not enough.

Elasticsearch → Full-text search, log analytics, faceted search.
                Not for primary storage — use as search index alongside PostgreSQL.

BigQuery/Redshift/Snowflake → Analytical workloads, petabyte scale,
                               SQL on columnar storage. Not for OLTP.

Kafka →      Event streaming, CDC, between-service communication,
             durable ordered log. Not a database but pairs with all of them.

DynamoDB →   Managed NoSQL on AWS, auto-scaling, key-value and simple document.
             Good for: serverless, unpredictable scale, AWS ecosystem.

TimescaleDB → PostgreSQL extension for time-series. Best of both worlds:
              full SQL + hypertables optimized for time-ordered data.

Neo4j →      Graph database. Use when relationships ARE the data and you need
             multi-hop traversals at millisecond latency on billion-edge graphs.
```

---

## 57.3 System Design Checklist

Before finalizing any database design, verify:

```
SCHEMA
☐ All entities identified and modeled
☐ Relationships correct (1:1, 1:N, M:N)
☐ Normalized to 3NF (then selectively denormalized with justification)
☐ Primary keys are BIGINT/UUID (not INT — avoids overflow at 2B rows)
☐ Timestamps use TIMESTAMPTZ (not TIMESTAMP — timezone aware)
☐ Soft deletes instead of hard deletes where history matters

INDEXES
☐ All foreign key columns are indexed
☐ Columns in frequent WHERE clauses are indexed
☐ Composite index column order matches query patterns
☐ Partial indexes for common filtered subsets
☐ Covering indexes for hot read paths (Index-Only Scan)

SCALE
☐ Write rate estimate fits within chosen database's capacity
☐ Read rate estimate — can single node + replicas handle it? If not: cache/shard
☐ Storage size estimate — will it exceed single disk? If so: partition or shard
☐ Peak traffic handled (peak = 5-10× average)

AVAILABILITY
☐ Single point of failure identified and eliminated
☐ Primary/replica setup for automatic failover
☐ Backup strategy defined (pg_dump + WAL archiving + PITR)
☐ RPO (max data loss): 0 for financial, minutes for others
☐ RTO (max downtime): seconds for financial, minutes for others

CONSISTENCY
☐ Right isolation level chosen per transaction type
☐ Distributed transaction handled (SAGA or 2PC with justification)
☐ Cache invalidation strategy defined
☐ Replication lag handled (read-your-writes consistency)

SECURITY
☐ Roles and permissions defined (least privilege)
☐ RLS policies for multi-tenant isolation
☐ Sensitive columns encrypted or masked (pg_crypto, column-level encryption)
☐ Connection over SSL/TLS
☐ No plaintext passwords in connection strings (use secrets management)

OPERATIONS
☐ Monitoring defined (slow query log, replication lag, vacuum status)
☐ Alerting thresholds set (XID age, dead tuples, connection count, disk usage)
☐ PgBouncer configured (transaction mode for web apps)
☐ Autovacuum tuned for table write rate
☐ Partitioning strategy for tables expected to exceed 100M rows
☐ Schema migration strategy (backward compatible, zero-downtime)
```

---

*End of Database System Design section — Chapters 47 through 57*

---

# PART XII — SCALING: REGIONAL AND GLOBAL

> Scaling is not just about adding more machines.
> It is about understanding WHERE your data lives, WHO needs it,
> HOW FAST they need it, and WHAT happens when things break.
> This part covers the full journey: from a single server to a globally
> distributed database serving millions of users across continents.

---

# CHAPTER 58 — The Scaling Journey

---

## 58.1 The Four Stages of Database Scaling

Every system starts small and grows. Understanding the progression
prevents over-engineering early and under-engineering late.

```
STAGE 1: Single Server
  ┌─────────────────────────────┐
  │   App + PostgreSQL          │
  │   on ONE machine            │
  │   (your laptop, t3.medium)  │
  └─────────────────────────────┘
  Handles: 0–10k req/day
  Breaks when: CPU > 80% sustained, or RAM exhausted

STAGE 2: Separate App and DB
  ┌─────────────┐       ┌─────────────────┐
  │  App Server │──────►│  PostgreSQL DB  │
  │  (EC2)      │       │  (RDS/dedicated)│
  └─────────────┘       └─────────────────┘
  Handles: 10k–500k req/day
  Breaks when: DB becomes the bottleneck (CPU, connections, disk)

STAGE 3: Read Replicas + Cache
  ┌─────────────┐    ┌───────────────────────────────────────┐
  │  App Server │───►│  Primary DB (writes)                  │
  │  (multiple) │    │  Replica 1 (reads)                    │
  │             │    │  Replica 2 (reads)                    │
  │             │───►│  Redis Cache (hot reads)              │
  └─────────────┘    └───────────────────────────────────────┘
  Handles: 500k–50M req/day
  Breaks when: write volume exceeds single primary capacity

STAGE 4: Sharding / Distributed DB
  ┌───────────────────────────────────────────────────────────┐
  │  Shard 1: customer_id 0-25%      (Primary + Replicas)    │
  │  Shard 2: customer_id 25-50%     (Primary + Replicas)    │
  │  Shard 3: customer_id 50-75%     (Primary + Replicas)    │
  │  Shard 4: customer_id 75-100%    (Primary + Replicas)    │
  └───────────────────────────────────────────────────────────┘
  Handles: 50M+ req/day, petabytes of data
```

---

## 58.2 Vertical Scaling — Scale Up First

Before distributing, always ask: can I just get a bigger machine?

```
Vertical scaling (scale up): increase CPU/RAM/disk on single server
Horizontal scaling (scale out): add more servers

When vertical scaling wins:
  → Simpler: no distributed system complexity
  → No application changes required
  → PostgreSQL gets more RAM → more data fits in shared_buffers → fewer disk reads
  → More CPU → more parallel query workers
  → NVMe SSD → lower random_page_cost → planner uses more index scans

Modern server limits (AWS/GCP):
  CPU:    1 → 128 vCPUs
  RAM:    1GB → 24TB (AWS x2iedn.metal: 4TB RAM, 128 vCPU)
  Disk:   1GB → 64TB SSD
  IOPS:   100 → 256,000 IOPS

For PostgreSQL specifically:
  Most workloads fit on: 16–32 vCPU, 64–256GB RAM, 4TB SSD
  Upgrade path: t3.micro → t3.large → r6g.4xlarge → r6g.16xlarge
  Cost difference: $10/month → $2,000/month → ~$20,000/month
  Often: vertical scaling to $2k/month server is cheaper than horizontal sharding

Vertical scaling limits:
  → Eventually hits physical limits (no bigger single machine)
  → Single point of failure (no HA without replication)
  → Downtime required for most upgrades
  → At some point: cost grows faster than performance
```

---

# CHAPTER 59 — Regional Scaling

---

## 59.1 What "Regional" Means

A region is a geographic area containing one or more data centers.
AWS regions: ap-south-1 (Mumbai), us-east-1 (Virginia), eu-west-1 (Ireland), etc.

Regional scaling means: all your servers are in the same region,
but distributed across multiple machines within that region.

**Why stay in one region first:**
- Network latency within a region: 1–5ms
- Network latency across regions: 50–300ms
- Cross-region transactions require waiting for remote confirmations
- Keep it simple until you truly need global scale

---

## 59.2 High Availability Within a Region (Patroni)

Single primary database is a single point of failure.
HA within a region uses automatic failover.

```
NORMAL OPERATION:
  ┌─────────────────────────────────────────────────────────────┐
  │                    AWS ap-south-1 (Mumbai)                  │
  │                                                             │
  │  ┌─────────────────┐    WAL streaming   ┌────────────────┐  │
  │  │    PRIMARY DB    │───────────────────►│   STANDBY 1    │  │
  │  │  (az: south-1a) │                    │  (az: south-1b)│  │
  │  └─────────────────┘                    └────────────────┘  │
  │           │               WAL streaming  ┌────────────────┐  │
  │           └──────────────────────────────►   STANDBY 2    │  │
  │                                          │  (az: south-1c)│  │
  │                                          └────────────────┘  │
  │  ┌──────────────────────────────────────────────────────┐    │
  │  │  PATRONI (distributed coordination via etcd/Consul)  │    │
  │  │  Monitors primary health every 2 seconds             │    │
  └──┴──────────────────────────────────────────────────────┴────┘

PRIMARY FAILS:
  1. Patroni detects primary unhealthy (missed heartbeats)
  2. Patroni runs leader election (via etcd/Consul DCS)
  3. Most advanced standby (highest LSN) is promoted to primary
  4. Patroni updates DNS/HAProxy to point to new primary
  5. Old primary (if recovered) becomes new standby

Total failover time: 10–30 seconds typically
Data loss: up to replication_lag at time of failure
            (0 with synchronous replication; milliseconds with async)
```

**Deploying Patroni:**
```yaml
# patroni.yml on each node
scope: postgres-ha-cluster
namespace: /service/
name: postgresql-node-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1-ip:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB lag max before excluding from election
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1-ip:5432
  data_dir: /var/lib/postgresql/data
  pg_hba:
    - host replication replicator 0.0.0.0/0 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256
  parameters:
    wal_level: replica
    hot_standby: on
    max_wal_senders: 5
    synchronous_commit: on
    synchronous_standby_names: 'ANY 1 (postgresql-node-2, postgresql-node-3)'
```

---

## 59.3 Regional Read Scaling

When reads exceed primary capacity, add read replicas.

```
ARCHITECTURE: Primary + Read Replicas within region
              ┌────────────────────────────────────────────────────┐
              │                  Load Balancer                     │
              │          (PgBouncer or HAProxy or AWS RDS Proxy)   │
              └──────┬──────────────────────────┬──────────────────┘
                     │ writes                   │ reads
                     ▼                          ▼
              ┌──────────────┐     ┌────────────────────────────┐
              │   PRIMARY    │────►│  REPLICA 1  │  REPLICA 2   │
              │ (all writes) │     │  (reads)    │  (reads)     │
              └──────────────┘     └────────────────────────────┘

Application routing logic:
  def get_db_connection(is_write):
      if is_write:
          return primary_pool.connection()
      else:
          return read_replica_pool.connection()  # round-robin across replicas

Read replica lag monitoring:
  SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
  -- Alert if > 5 seconds

PgBouncer for read replicas:
  [databases]
  mydb_primary = host=primary port=5432 dbname=mydb
  mydb_replica = host=replica1,replica2 port=5432 dbname=mydb

  [pgbouncer]
  pool_mode = transaction
  max_client_conn = 10000
  default_pool_size = 25
```

---

## 59.4 Regional Write Scaling — Partitioning First

Before sharding, partition within a single server:

```sql
-- Orders table partitioned by month within ap-south-1:
CREATE TABLE orders (
    id          BIGSERIAL,
    customer_id BIGINT NOT NULL,
    total       NUMERIC(12,2),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Each partition can be on different tablespaces (different disks):
CREATE TABLESPACE fast_ssd LOCATION '/mnt/nvme/pg_data';
CREATE TABLESPACE slow_hdd LOCATION '/mnt/hdd/pg_data';

-- Recent data on fast SSD:
CREATE TABLE orders_2024_06 PARTITION OF orders
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01')
    TABLESPACE fast_ssd;

-- Old data on slower (cheaper) HDD:
CREATE TABLE orders_2022_01 PARTITION OF orders
    FOR VALUES FROM ('2022-01-01') TO ('2022-02-01')
    TABLESPACE slow_hdd;

-- Benefits:
-- 1. Query pruning: WHERE created_at > '2024-01-01' → only scans 2024 partitions
-- 2. Tiered storage: hot data on NVMe, cold data on HDD
-- 3. Drop old data: DROP TABLE orders_2021_01 → instant, no row-by-row delete
-- 4. Parallel across partitions: Parallel Append node scans partitions simultaneously
```

---

## 59.5 Caching Layer Within a Region

```
FULL REGIONAL CACHING ARCHITECTURE:

┌──────────────────────────────────────────────────────────────────┐
│                      APPLICATION SERVERS                         │
│                    (auto-scaling group)                          │
└─────────────────────┬────────────────────────────────────────────┘
                      │
              ┌───────▼───────┐
              │ In-Process    │  ← Local cache (e.g., caffeine/guava)
              │ Cache (L1)    │    ~1ms, tiny (MB), per app instance
              │ ~10,000 items │    best for: config, static lookups
              └───────┬───────┘
                      │ cache miss
              ┌───────▼───────┐
              │ Redis Cluster │  ← Distributed cache (L2)
              │ (L2 Cache)    │    ~0.5ms, large (GBs), shared across all apps
              │ 3 masters +   │    best for: session, per-user data, hot queries
              │ 3 replicas    │
              └───────┬───────┘
                      │ cache miss
              ┌───────▼───────┐
              │ PgBouncer     │  ← Connection pooler
              └──┬────────────┘
                 │
        ┌────────▼─────────┐
        │    PRIMARY DB    │ ← Only hit on cache miss (rare)
        │  + 2 REPLICAS    │
        └──────────────────┘

Cache hit rates in production:
  L1 (in-process): ~60% (config, frequently accessed items)
  L2 (Redis):      ~35% (most other reads)
  DB:              ~5%  (cache misses, writes, admin queries)
```

---

## 59.6 Regional Sharding — When Partitioning Is Not Enough

When write volume exceeds a single primary's capacity (~20k writes/sec):

```
REGIONAL SHARDED ARCHITECTURE (4 shards within ap-south-1):

                    ┌──────────────────────────┐
                    │    SHARD ROUTER           │
                    │  (Citus, Vitess, or       │
                    │   application-level)      │
                    └──┬──────┬──────┬──────┬──┘
                       │      │      │      │
                hash(customer_id) % 4
                       │      │      │      │
            ┌──────────┘  ┌───┘  ┌───┘  └──────────┐
            ▼             ▼      ▼                   ▼
        ┌────────┐   ┌────────┐  ┌────────┐  ┌────────┐
        │SHARD 0 │   │SHARD 1 │  │SHARD 2 │  │SHARD 3 │
        │Primary │   │Primary │  │Primary │  │Primary │
        │+Replica│   │+Replica│  │+Replica│  │+Replica│
        └────────┘   └────────┘  └────────┘  └────────┘
        0–25%        25–50%      50–75%      75–100%
        customer_ids customer_ids customer_ids customer_ids

Citus Extension (PostgreSQL-native sharding):
  -- On coordinator node:
  CREATE EXTENSION citus;

  -- Distribute a table across shards:
  SELECT create_distributed_table('orders', 'customer_id');
  SELECT create_distributed_table('order_items', 'customer_id');  -- co-locate with orders

  -- Co-location ensures: orders and order_items for same customer are on same shard
  -- → join between orders and order_items stays local (no cross-shard)

  -- After distribution, queries work exactly the same:
  SELECT * FROM orders WHERE customer_id = 5;
  -- Citus routes to correct shard automatically

  -- Cross-shard aggregate:
  SELECT SUM(total) FROM orders WHERE created_at > '2024-01-01';
  -- Citus: parallel query to all shards, merge results at coordinator

  -- Reference tables (replicated to ALL shards):
  SELECT create_reference_table('countries');  -- small, frequently joined
  -- countries is available on every shard → joins don't need cross-shard communication
```

---

# CHAPTER 60 — Global Scaling

---

## 60.1 WHY Global Distribution

```
Reasons to distribute globally:

1. LATENCY: Users in Mumbai experience 300ms to US-East servers
   Physics: speed of light limits to ~100ms Mumbai→Virginia (round trip)
   With network overhead: 200–300ms typical
   Acceptable for many apps: NO (users feel it)
   Solution: serve Mumbai users from Mumbai data center

2. AVAILABILITY: Regional disasters (datacenter fire, region outage, cable cut)
   AWS ap-south-1 outage in 2021: all services in Mumbai went down
   Globally distributed: Mumbai fails → automatically route to Singapore

3. DATA SOVEREIGNTY / COMPLIANCE:
   GDPR: EU user data must stay in EU
   India PDPB: Indian citizen data must stay in India
   HIPAA: US health data stays in US
   Multi-region architecture with data residency ensures compliance

4. SCALE: Single region cannot handle global traffic
   Netflix: 200+ countries, streaming HD video
   Every country needs edge servers for acceptable performance

Latency reality (approximate round-trip times):
  Same datacenter:     < 1ms
  Same city:           1–5ms
  Same country:        5–50ms
  Mumbai → Singapore:  30–50ms
  Mumbai → London:     100–150ms
  Mumbai → Virginia:   200–250ms
  Mumbai → Sydney:     150–200ms
```

---

## 60.2 Active-Passive Multi-Region

The simplest multi-region setup: one primary region serves all traffic,
second region is a hot standby for disaster recovery.

```
ACTIVE-PASSIVE ARCHITECTURE:

PRIMARY REGION (ap-south-1 Mumbai) ← All read + write traffic
┌─────────────────────────────────────────────────────────────┐
│  Primary DB  ──WAL──►  Replica 1  ──WAL──►  Replica 2      │
│      │                                                      │
│      └── async WAL streaming (WAN) ──────────────────────┐  │
└─────────────────────────────────────────────────────────────┘
                                                          │
              Cross-region WAL replication (async)       │
              Typical lag: 50–200ms                      │
                                                          ▼
DR REGION (ap-southeast-1 Singapore) ← No traffic normally
┌─────────────────────────────────────────────────────────────┐
│  Standby DB (hot standby — replaying WAL from Mumbai)       │
│  Can be promoted to primary within 30–120 seconds           │
└─────────────────────────────────────────────────────────────┘

FAILOVER SCENARIO:
  1. Mumbai region becomes unavailable (earthquake, power failure, AWS outage)
  2. Monitoring detects Mumbai unreachable (health checks fail)
  3. Patroni/operator promotes Singapore standby to primary
  4. DNS update: db.myapp.com → Singapore IP
  5. Application reconnects, resumes operations from Singapore
  6. RPO (data loss): amount of WAL not yet replicated to Singapore
     (with async: potentially seconds to minutes of data loss)
  7. RTO (recovery time): 2–10 minutes typically

Making RPO = 0 (no data loss):
  Use synchronous replication across regions:
  synchronous_standby_names = 'ANY 1 (singapore_standby)'
  -- Every COMMIT waits for Singapore to acknowledge → 50-200ms added to every write!
  -- Trade-off: zero data loss but slower writes
  -- Acceptable for: financial transactions, medical records
  -- Not acceptable for: high-throughput gaming, social feeds
```

---

## 60.3 Active-Active Multi-Region — The Hard Problem

Active-Active means users in EVERY region read AND write to a LOCAL database.
No cross-region latency for any user.

```
ACTIVE-ACTIVE ARCHITECTURE (2 regions):

Mumbai Users                            Singapore Users
     │                                       │
     ▼                                       ▼
┌─────────────────┐    Conflict-free    ┌─────────────────┐
│  Mumbai Primary │◄──────────────────►│Singapore Primary│
│  (local reads   │  bidirectional      │  (local reads   │
│   and writes)   │  async replication  │   and writes)   │
└─────────────────┘                    └─────────────────┘

PROBLEM — Conflict:
  Mumbai:    UPDATE accounts SET balance = balance - 500 WHERE id = 1;
             (reads balance=1000, writes 500)
  Singapore: UPDATE accounts SET balance = balance - 300 WHERE id = 1;
             (reads balance=1000, writes 700, at same millisecond)
  After replication: CONFLICT! Which value wins? 500 or 700? (correct: 200!)

Active-Active works for:
  ✅ Geographically partitioned data (Mumbai users only write Mumbai data,
     Singapore users only write Singapore data → no overlap → no conflicts)
  ✅ Eventually consistent data (comments, likes — last-write-wins is OK)
  ✅ Append-only data (events, logs — each write is unique, no conflicts)
  ✅ CRDTs (counters, sets with specific merge semantics)

Active-Active is DANGEROUS for:
  ❌ Account balances (conflict is money appearing/disappearing)
  ❌ Inventory counts (conflict leads to overselling)
  ❌ Sequential IDs (two regions generating same ID)
  ❌ Any data where last-write-wins is incorrect
```

---

## 60.4 Conflict Resolution Strategies

When two regions write the same row simultaneously:

```
STRATEGY 1: Last Write Wins (LWW)
  The write with the latest timestamp wins.
  Implementation: include a last_modified TIMESTAMPTZ column
  → Mumbai writes balance=500 at 10:00:00.100
  → Singapore writes balance=700 at 10:00:00.150
  → Singapore's write (later timestamp) wins → balance=700
  → Mumbai's -500 operation is LOST silently!

  Problem: clocks between regions are never perfectly synchronized
           (NTP drift can be 10ms–100ms)
           → "later" timestamp may not be the actually later write
  Acceptable for: profile pictures, display names, non-financial updates

STRATEGY 2: Application-Level Conflict Detection
  Use vector clocks or version vectors:
  Each row has a version: {mumbai: 5, singapore: 3}
  On conflict: application decides which version to keep (or merge)
  Complex but correct.
  Used by: Amazon DynamoDB, CouchDB

STRATEGY 3: Causal Ordering (CRDTs)
  Design data structures that can always be merged without conflicts:
  → Counter: each region has its own counter, total = sum of all regions
    Mumbai counter: 5 decrements
    Singapore counter: 3 decrements
    Total: 8 decrements applied to original value
    No conflict — they simply add up
  → Set: adding elements never conflicts (use tombstones for deletes)
  Used by: Redis CRDT (Enterprise), Riak

STRATEGY 4: Avoid Conflicts by Design (Partitioning by Region)
  Users in India ALWAYS write to Mumbai database
  Users in Singapore ALWAYS write to Singapore database
  Bidirectional replication for reads only (not writes to same rows)
  → No conflicts possible if partitioning is strict

  Implementation:
  users table: id, name, home_region ('IN', 'SG')
  Mumbai DB: PRIMARY for all users where home_region='IN'
  Singapore DB: PRIMARY for all users where home_region='SG'
  Cross-region reads: eventual consistency (200ms lag acceptable)
  Cross-region writes: not allowed (rejected with redirect)
```

---

## 60.5 Globally Distributed Databases

For teams that need global ACID without building their own conflict resolution:

### Google Spanner

```
What it is: Google's globally distributed relational database
  → True global transactions with external consistency (stronger than linearizability)
  → SQL interface (PostgreSQL-compatible dialect)
  → Automatic sharding and replication across regions
  → 99.999% availability SLA (< 5 minutes downtime per year)

How it achieves global consistency (TrueTime):
  → Google uses atomic clocks + GPS in every datacenter
  → TrueTime API provides: "current time is within [earliest, latest] uncertainty"
  → Uncertainty: ±4ms typically (atomic clock precision)
  → Spanner uses this: if transaction commits at time T,
    it waits until the uncertainty window passes before returning
  → Guarantees: any later transaction anywhere in the world sees this commit

Performance:
  Write latency: ~10ms for same region, ~100ms for cross-continent
  Read latency: ~1ms (stale reads acceptable), ~10ms (strong reads)
  IOPS: auto-scaling

Cost:
  Processing: $0.90/node/hour (a 3-node instance: ~$600/month)
  Storage: $0.30/GB/month
  Expensive but no operational burden

Use when: global ACID is non-negotiable, budget allows, avoiding ops burden

GCP Cloud Spanner example:
  CREATE TABLE orders (
    order_id  INT64 NOT NULL,
    user_id   INT64 NOT NULL,
    total     NUMERIC,
    created_at TIMESTAMP
  ) PRIMARY KEY (order_id);

  -- Interleaved tables (physical co-location, like partitioning):
  CREATE TABLE order_items (
    order_id   INT64 NOT NULL,
    item_id    INT64 NOT NULL,
    product_id INT64,
    quantity   INT64
  ) PRIMARY KEY (order_id, item_id),
  INTERLEAVE IN PARENT orders ON DELETE CASCADE;
```

### CockroachDB

```
What it is: Open-source distributed SQL database, PostgreSQL-compatible wire protocol
  → Runs on commodity hardware (not Google's atomic clocks)
  → Uses HLC (Hybrid Logical Clocks) for consistency
  → Automatic sharding (ranges), replication, and rebalancing
  → Can run on-premises or in any cloud

Architecture:
  → Data split into "ranges" (~64MB each)
  → Each range: 3 or 5 replicas across different nodes/regions
  → Consensus via Raft (majority must agree before commit)
  → SQL layer on top (largely PostgreSQL compatible)

Consistency:
  → Serializable isolation by default (stronger than most databases)
  → "Follower reads": slightly stale reads from local replicas
    (use for reads where 1–2 second staleness is OK → low latency)
  → Strong reads: always goes to range leaseholder (slower but fresh)

Write latency (cross-region commits):
  → 3 nodes in Mumbai: ~2ms (same city)
  → 3 nodes: Mumbai + Singapore + Tokyo: ~100ms (consensus takes 1 cross-region RTT)

PostgreSQL compatibility:
  MOST PostgreSQL queries work unchanged
  Differences: no sequences (use UUID instead), some functions differ

Use when: want Spanner-like guarantees on-premises or multi-cloud, team manages infrastructure
```

### YugabyteDB

```
What it is: PostgreSQL-compatible distributed SQL, similar to CockroachDB
  → Uses RocksDB as storage (LSM-Tree → good write throughput)
  → Uses Raft for consensus
  → Full PostgreSQL compatibility (supports triggers, stored procedures)
  → Two APIs: YSQL (PostgreSQL-compatible), YCQL (Cassandra-compatible)

Differentiation from CockroachDB:
  → Better PostgreSQL compatibility (triggers, stored procedures work)
  → YCQL API for existing Cassandra users
  → More configuration options for consistency vs latency trade-offs

Use when: need PostgreSQL feature compatibility (triggers, etc.) + distribution
```

---

## 60.6 Global Read Scaling with CDN and Edge

Not every read needs the database. Static and semi-static data can be cached globally:

```
GLOBAL CACHING ARCHITECTURE:

User in Tokyo
    │
    ▼
CDN Edge (Tokyo POP)
  ├── Cache HIT: return in ~5ms
  └── Cache MISS:
          ↓
      Origin Load Balancer
          ↓
      Nearest Region (Singapore)
          ├── Redis Cache HIT: ~50ms (Tokyo→Singapore round trip)
          └── Redis Cache MISS:
                  ↓
              PostgreSQL Primary (Mumbai): ~150ms (Tokyo→Mumbai)

Content types and TTLs:
  Product catalog, prices:    CDN TTL=5min, Redis TTL=1min
  Homepage banners/hero:      CDN TTL=1hour
  User profile:               CDN TTL=0 (private), Redis TTL=15min
  Order status:               CDN TTL=0 (must be fresh)
  Static assets (images):     CDN TTL=30 days (immutable with hash in URL)

Cache-Control headers for CDN:
  Cache-Control: public, max-age=300, s-maxage=300     ← 5 min CDN cache
  Cache-Control: private, max-age=60                   ← user-specific, no CDN
  Cache-Control: no-store                              ← never cache (financial)
  Cache-Control: public, max-age=31536000, immutable   ← 1 year (hashed asset)
```

---

## 60.7 GeoDNS and Global Load Balancing

Route users to the nearest region automatically:

```
GeoDNS:
  User in Mumbai:   db.myapp.com → 13.232.x.x (ap-south-1)
  User in Berlin:   db.myapp.com → 18.185.x.x (eu-west-1)
  User in New York: db.myapp.com → 54.x.x.x   (us-east-1)

  Implementation options:
    → AWS Route 53 with Latency-Based Routing
    → Cloudflare Load Balancing (+ health checks)
    → GCP Cloud Load Balancing (global anycast)

Anycast (used by Cloudflare, CDNs):
  Multiple servers worldwide share the SAME IP address
  BGP routing: network routes your packet to the topologically nearest server
  No DNS lookup required → fastest possible routing
  Used for: CDN edge nodes, DDoS protection, DNS resolvers

Health checks and automatic failover:
  Every 10 seconds: health check to each region
  If a region fails health check 3× in a row:
    → Remove from DNS responses automatically
    → Traffic shifts to remaining healthy regions
    → Failover time: 30–60 seconds (DNS TTL dependent)

Low DNS TTL for faster failover:
  db.myapp.com TTL=60 seconds
  → After region failure: max 60 seconds before DNS update propagates
  → Clients with cached DNS: still try failed region for up to 60s
  → Solution: short TTL (60s) + client-side retry with DNS re-resolution
```

---

## 60.8 Data Residency and Compliance in Global Systems

```
GDPR (General Data Protection Regulation — EU):
  EU citizen data must remain within EU borders (or adequate countries)
  Data subjects have: right to erasure, right to access, right to portability
  Applies to: any company processing EU citizen data, regardless of company location

  Implementation:
    Shard/partition by user's country_code
    EU users → eu-west-1 (Ireland) or eu-central-1 (Frankfurt)
    Non-EU users → other regions
    Cross-region queries for EU data: forbidden (must stay in EU)

India PDPB (Personal Data Protection Bill):
  "Sensitive personal data" must be stored in India
  Critical personal data: ONLY in India (no cross-border transfer)
  Non-sensitive: can be mirrored globally

HIPAA (US Health Insurance Portability and Accountability Act):
  Protected Health Information (PHI) must be encrypted at rest and in transit
  Access logs required (audit trails)
  Business Associate Agreements (BAA) with cloud providers

Implementation pattern for multi-jurisdiction compliance:

users table:
  id          BIGINT PRIMARY KEY
  data_region VARCHAR(10) NOT NULL  -- 'EU', 'IN', 'US', 'APAC'

Strategy 1: Separate databases per region (strict isolation):
  EU_DB (Frankfurt):    stores EU user data
  IN_DB (Mumbai):       stores India user data
  US_DB (Virginia):     stores US user data
  APAC_DB (Singapore):  stores rest

  Application: routes each query to correct DB based on data_region
  Cross-region queries: not allowed (return error if attempted)

Strategy 2: Row-level data residency with enforcement:
  All data in one logical schema, physically stored in correct region partition
  Application-level enforcement: before any read/write, verify user's data_region
    matches current request's region
  Cross-region: proxy through VPN with audit log (for GDPR: only with legal basis)

Deletion (Right to Erasure — GDPR):
  Hard delete: actually DELETE rows (simple but irreversible, event sourcing is complex)
  Crypto-shredding: encrypt user data with user-specific key
    → "Delete" = destroy the encryption key
    → Data physically exists but is permanently unreadable
    → Works with event sourcing (events become undecipherable, not missing)

  CREATE TABLE user_encryption_keys (
      user_id     BIGINT PRIMARY KEY,
      dek         BYTEA NOT NULL,  -- data encryption key (encrypted with master key)
      created_at  TIMESTAMPTZ DEFAULT NOW(),
      deleted_at  TIMESTAMPTZ     -- NULL = active, NOT NULL = key destroyed = data deleted
  );
```

---

## 60.9 Multi-Region Write Architecture — The Complete Picture

```
FULL GLOBAL ARCHITECTURE (3 regions):

REGION 1: ap-south-1 (Mumbai) — Primary for India users
┌─────────────────────────────────────────────────────────────────┐
│  Primary DB      Replica 1    Replica 2    Redis Cluster        │
│  (writes)        (reads)      (reads)      (cache)              │
│       │               └──────────┘                              │
│       │ async WAL                                               │
└───────┼─────────────────────────────────────────────────────────┘
        │ cross-region WAL replication
        │ (async: ~50ms lag)
        ▼
REGION 2: ap-southeast-1 (Singapore) — Primary for SEA users
┌─────────────────────────────────────────────────────────────────┐
│  Primary DB      Replica 1    Redis Cluster                     │
│  (SEA writes)    (reads)      (cache)                           │
│       │                                                         │
└───────┼─────────────────────────────────────────────────────────┘
        │ cross-region WAL replication
        ▼
REGION 3: eu-west-1 (Ireland) — Primary for EU users
┌─────────────────────────────────────────────────────────────────┐
│  Primary DB      Replica 1    Redis Cluster                     │
│  (EU writes)     (reads)      (cache)                           │
└─────────────────────────────────────────────────────────────────┘

DATA ROUTING RULES:
  India user reads:    → Mumbai primary or replica
  India user writes:   → Mumbai primary only
  EU user reads:       → Ireland replica (GDPR: EU data stays in EU)
  EU user writes:      → Ireland primary only
  SEA user reads:      → Singapore replica
  SEA user writes:     → Singapore primary

  "Global" data (product catalog, etc.):
    Primary: Mumbai
    Replicated to: Singapore, Ireland
    Writes always go to Mumbai primary (single source of truth)
    Reads served locally from each region's replica

MONITORING ACROSS REGIONS:
  ┌────────────────────────────────────────────────────┐
  │  Datadog / Grafana / CloudWatch cross-region view  │
  │                                                    │
  │  Mumbai  → replication_lag=45ms  ✅ healthy        │
  │  Singapore→ replication_lag=52ms ✅ healthy        │
  │  Ireland → replication_lag=185ms ⚠️ elevated       │
  │                                                    │
  │  Alert if replication_lag > 10 seconds             │
  └────────────────────────────────────────────────────┘
```

---

## 60.10 Write Amplification in Global Systems

Every write to a global primary must be replicated everywhere:

```
Problem: single write → replicated to N regions → N× write amplification

Example: Product catalog update
  UPDATE products SET price=149 WHERE id=1

  Mumbai Primary:     1 write
  Singapore Replica:  1 WAL apply (async, 50ms later)
  Ireland Replica:    1 WAL apply (async, 150ms later)
  CDN invalidation:   API call to invalidate cached pages
  Redis invalidation: DEL "product:1" in each region's Redis
  Elasticsearch sync: reindex product document

Total cost of 1 write: ~5–7 operations across the globe
Write throughput limit: 1 / (replication bandwidth per write × N replicas)

Solution: prioritize what gets replicated:
  Critical data (accounts, orders): strong consistency, sync replication to all regions
  Product catalog: async replication, CDN TTL, eventual consistency acceptable
  Analytics data: batch ETL to data warehouse, not real-time replication
```

---

# CHAPTER 61 — Scaling Anti-Patterns to Avoid

---

## 61.1 The N+1 Query — Most Common Scaling Killer

```python
# BAD: N+1 queries
orders = db.query("SELECT * FROM orders LIMIT 100")
for order in orders:
    # Executes 1 query PER ORDER — 100 extra queries!
    customer = db.query("SELECT * FROM customers WHERE id = %s", order.customer_id)
    order.customer_name = customer.name

# GOOD: single JOIN query
orders = db.query("""
    SELECT o.*, c.full_name AS customer_name
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    LIMIT 100
""")

# ALSO GOOD: batch lookup
order_ids = [o.id for o in orders]
customers = db.query("SELECT * FROM customers WHERE id = ANY(%s)", [order_ids])
customer_map = {c.id: c for c in customers}
for order in orders:
    order.customer = customer_map[order.customer_id]
```

---

## 61.2 SELECT * — The Bandwidth Killer

```sql
-- BAD: fetches all columns (including large JSONB, TEXT, BYTEA columns)
SELECT * FROM products;
-- If products has a description column (500 bytes avg) × 100k rows = 50MB per query

-- GOOD: fetch only needed columns
SELECT id, name, price, stock_quantity FROM products;
-- 40 bytes avg × 100k rows = 4MB (12× less data)

-- Impact on global systems:
-- Mumbai→Singapore: 50MB at 100Mbps = 4 seconds transfer
-- Mumbai→Singapore: 4MB  at 100Mbps = 0.32 seconds transfer
```

---

## 61.3 Missing Indexes on Foreign Keys

```sql
-- This is the single most commonly missed optimization:
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id)
    -- ← NO INDEX on customer_id!
);

-- Query: find all orders for customer 5
SELECT * FROM orders WHERE customer_id = 5;
-- FULL TABLE SCAN on orders (seq scan, reads entire table)
-- If orders has 100M rows: 10+ seconds

-- Fix: always index FK columns
CREATE INDEX ix_orders_customer_id ON orders(customer_id);
-- Now: index scan, < 10ms

-- Rule: EVERY foreign key column needs an index
-- Script to find missing FK indexes:
SELECT
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS referenced_table
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
    ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND NOT EXISTS (
      SELECT 1 FROM pg_index pi
      JOIN pg_attribute pa ON pa.attrelid = pi.indrelid
          AND pa.attnum = pi.indkey[0]
      WHERE pi.indrelid = tc.table_name::regclass
        AND pa.attname = kcu.column_name
  );
```

---

## 61.4 Unbounded Queries (Missing LIMIT)

```sql
-- BAD: fetches unlimited rows
SELECT * FROM events WHERE user_id = 5 ORDER BY created_at DESC;
-- User with 10 years of events: 100M rows returned to application → OOM crash

-- GOOD: always paginate
SELECT * FROM events WHERE user_id = 5 ORDER BY created_at DESC LIMIT 100;

-- BAD: no timeout on slow query
SELECT * FROM complex_report_query;  -- might run for hours

-- GOOD: statement timeout
SET statement_timeout = '30s';
SELECT * FROM complex_report_query;  -- fails after 30 seconds, not hours
RESET statement_timeout;
```

---

## 61.5 Over-Indexing

```sql
-- BAD: indexing every column "just in case"
CREATE INDEX ON orders(id);                         -- PK already has this
CREATE INDEX ON orders(customer_id);                -- useful ✅
CREATE INDEX ON orders(customer_id, created_at);   -- also useful ✅
CREATE INDEX ON orders(total_amount);               -- rarely queried → waste
CREATE INDEX ON orders(status);                     -- low cardinality → waste
CREATE INDEX ON orders(created_at);                 -- probably covered by composite ✅
CREATE INDEX ON orders(payment_method);             -- never queried → waste
CREATE INDEX ON orders(notes);                      -- large text → huge GIN index

-- EVERY index:
-- → Costs ~5MB per 1M rows (for simple B-Tree)
-- → Adds ~10–30% overhead to every INSERT/UPDATE/DELETE
-- → Requires maintenance (VACUUM, REINDEX)
-- → Fills shared_buffers (evicts hot data pages)

-- DISCIPLINE: only add index if you can show:
-- 1. Query that would benefit exists
-- 2. Query runs frequently (pg_stat_statements shows > 10 calls/hour)
-- 3. EXPLAIN shows seq scan on large table (current performance is bad)
```

---

# CHAPTER 62 — Complete Scaling Reference

---

## 62.1 Scaling Decision Tree

```
START: System is slow or expected to be slow

STEP 1: Profile first (don't guess)
  → pg_stat_statements: find the slowest queries
  → EXPLAIN ANALYZE: understand why each is slow
  → pg_stat_activity: find lock contention
  → pg_stat_user_tables: find tables with high dead tuples

STEP 2: Fix the query (most common fix, zero infrastructure cost)
  → Add missing index
  → Remove function wrapping (SARGable)
  → Replace N+1 with JOIN
  → Add LIMIT

STEP 3: Optimize the server configuration
  → shared_buffers = 25% RAM
  → work_mem = 16–64MB
  → random_page_cost = 1.1 (SSD)
  → PgBouncer with transaction mode

STEP 4: Scale vertically
  → Upgrade to larger instance (more CPU, RAM, faster SSD)
  → Check: can this solve the problem? (often YES for < $2k/month servers)

STEP 5: Add read replicas
  → Reads consuming > 70% of primary capacity
  → Analytics queries slowing OLTP
  → Add 1–3 read replicas, route reads to replicas

STEP 6: Add caching layer
  → Hot data (same data read 1000x/sec) → Redis cache
  → Reduce DB load by 80–95% for cacheable data

STEP 7: Partition large tables
  → Tables > 100M rows → partition by time or tenant
  → No application changes needed (PostgreSQL handles routing)

STEP 8: Shard (only if truly necessary)
  → Writes exceed single primary capacity (> 20k/sec)
  → Data size exceeds single server (> 10TB)
  → Use Citus (PostgreSQL-native) or Vitess (MySQL)

STEP 9: Global distribution
  → Users in multiple continents experiencing > 200ms latency
  → Regulatory requirements force regional data storage
  → Deploy primary+replicas in each region
  → Use GeoDNS to route users to nearest region
  → Handle global data (catalog, config) with async replication
```

---

## 62.2 Scaling Numbers to Memorize

```
SINGLE PostgreSQL SERVER:
  Reads:   10,000–50,000 simple queries/second
  Writes:  5,000–20,000 writes/second
  Connections (with PgBouncer): 50–200 actual DB connections
  Storage: up to ~10TB practical (with partitioning: much more)

AFTER READ REPLICAS (×3 replicas):
  Reads:   40,000–150,000 queries/second (spread across 3 replicas + primary)
  Writes:  unchanged (still one primary)

AFTER REDIS CACHE:
  Reads:   500,000–1,000,000+ cached reads/second (Redis handles 95% of reads)
  DB reads: 5% of original (only cache misses)

AFTER SHARDING (×4 shards):
  Writes:  20,000–80,000 writes/second
  Reads:   40,000–200,000 queries/second (per shard × 4)

REDIS SINGLE NODE:
  Operations: 100,000–1,000,000/second
  Latency: 0.1–1ms

LATENCY TARGETS (p99):
  Cache hit:         < 5ms
  DB indexed read:   < 10ms
  DB write:          < 20ms
  Cross-region read: < 100ms
  Cross-region write: 50–200ms

NETWORK:
  1Gbps = 125MB/s = ~1.25M small rows/second
  Speed of light Mumbai→Singapore: ~15ms one-way
  Speed of light Mumbai→London:    ~60ms one-way
  Speed of light Mumbai→NY:        ~95ms one-way
  Real RTT (with network overhead): 2–3× the speed-of-light time
```

---

## 62.3 Global Scaling Architecture Checklist

```
REGIONAL (single region, multiple AZs):
  ☐ Primary + 2 standbys across different Availability Zones
  ☐ Patroni for automatic failover (< 30 seconds RTO)
  ☐ Synchronous replication to at least 1 standby (RPO = 0)
  ☐ PgBouncer in transaction mode
  ☐ Redis cluster (3 masters + 3 replicas) for caching
  ☐ Table partitioning for tables > 50M rows
  ☐ All FK columns indexed
  ☐ pg_stat_statements enabled, slow query log set to 1 second
  ☐ Autovacuum tuned (scale_factor = 0.01 for large tables)
  ☐ WAL archiving to S3 + PITR configured
  ☐ Monitoring: replication lag, XID age, dead tuples, connection count

MULTI-REGION:
  ☐ GeoDNS routing users to nearest region
  ☐ CDN caching static and semi-static responses (TTL set appropriately)
  ☐ Async streaming replication across regions (monitor lag)
  ☐ Data residency rules enforced at application layer
  ☐ Cross-region failover tested and documented (runbook)
  ☐ Acceptable RPO/RTO defined and agreed with stakeholders
  ☐ Data sovereignty compliance verified (GDPR, PDPB, HIPAA)
  ☐ Global write conflicts addressed (partition by region or CRDT)

OBSERVABILITY:
  ☐ Distributed tracing across services (Jaeger, Datadog APM)
  ☐ Cross-region replication lag dashboard
  ☐ Per-region error rate and latency monitoring
  ☐ Alerting: p99 > 200ms → page on-call
  ☐ Runbooks for: primary failover, replica lag, shard rebalancing
  ☐ Chaos engineering: test failover regularly (not just on real failures)
```

---

*End of Scaling section — Chapters 58 through 62*
*The complete PostgreSQL Mastery Book now covers Chapters 1–62*

---

# PART XIII — DDIA DEEP TOPICS

---

# CHAPTER 63 — Encoding and Evolution

---

## 63.1 WHY Encoding Matters

Every piece of data stored in a database or sent over a network
must be encoded into bytes. The choice of encoding determines:
- How much space data takes (impacts storage and network costs)
- How fast serialization/deserialization is (impacts CPU)
- Whether old code can read data written by new code (schema evolution)
- Whether new code can read data written by old code (backward compatibility)

In microservices: services upgrade independently.
Service A (v2) sends data to Service B (v1).
If the encoding does not support schema evolution → deployment breaks.

---

## 63.2 Language-Specific Formats — Why to Avoid

```python
# Python pickle: serializes any Python object
import pickle
data = {"user_id": 5, "name": "Rahul", "scores": [90, 85, 92]}
encoded = pickle.dumps(data)

# Problems:
# 1. Language-specific: only Python can decode pickle
# 2. Security: pickle can execute arbitrary code during deserialization (RCE vulnerability)
# 3. Performance: slow for large objects
# 4. No schema: no guarantee about data shape

# Java ObjectOutputStream, Ruby Marshal — same problems

# Rule: NEVER use language-specific serialization for:
# - Storing data in databases (you'll need to read it from another language someday)
# - Sending data between services
# - Any long-term storage
```

---

## 63.3 JSON and XML — Human Readable but Costly

```json
// JSON is the most common encoding for APIs
{
  "order_id": 1001,
  "customer": {"id": 5, "name": "Rahul Kumar"},
  "items": [
    {"product_id": 10, "quantity": 2, "price": 99.99}
  ],
  "total": 199.98,
  "created_at": "2024-01-15T10:30:00Z"
}
```

**JSON problems:**
```
1. Ambiguous numbers:
   JSON has no integer/float distinction
   Some parsers: 2^53 is the max safe integer (JavaScript)
   Large IDs (Twitter uses 64-bit): "id": 1234567890123456789
   JavaScript reads as 1234567890123456800 (precision loss!)
   Fix: send large IDs as strings: "id": "1234567890123456789"

2. No binary support:
   Binary data (images, files) must be Base64-encoded
   Base64 increases size by 33%

3. Schema: no built-in schema validation (need JSON Schema separately)

4. Size: verbose (field names repeated in every object)
   {"user_id": 5, "name": "Rahul"} = 28 bytes
   Binary equivalent: 2 + 4 + 5 = ~11 bytes (60% smaller)

5. Speed: string parsing is slow compared to binary formats
```

---

## 63.4 Protocol Buffers (Protobuf) — Google's Binary Format

Protobuf encodes data as compact binary using a schema defined in a `.proto` file.

### Schema Definition

```protobuf
// order.proto
syntax = "proto3";

message OrderItem {
  int32 product_id = 1;
  int32 quantity = 2;
  double price = 3;
}

message Customer {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

message Order {
  int32 order_id = 1;
  Customer customer = 2;
  repeated OrderItem items = 3;
  double total = 4;
  string created_at = 5;
}
```

### How Protobuf Encodes Data

Each field is encoded as `(field_number, wire_type, value)`:

```
Field 1 (order_id=1001):
  Tag: (1 << 3) | 0 = 8   [field_number=1, wire_type=0 (varint)]
  Value: 1001 encoded as varint = 0xE9 0x07

Field 2 (customer):
  Tag: (2 << 3) | 2 = 18  [field_number=2, wire_type=2 (length-delimited)]
  Length: byte count of nested Customer message
  Value: encoded Customer bytes

Wire types:
  0 = Varint (int32, int64, bool, enum)
  1 = 64-bit (double, fixed64)
  2 = Length-delimited (string, bytes, embedded messages, repeated)
  5 = 32-bit (float, fixed32)

Key insight: field NAMES are not stored in the binary!
Only field NUMBERS are stored.
Schema (the .proto file) is needed to decode.
```

### Varint Encoding — Why Protobuf is Compact

```
Small numbers (< 128) take 1 byte.
Larger numbers take 2-4 bytes.
This matters: most integers in real data are small!

order_id=1:    encoded in 1 byte  (not 4 bytes like int32)
order_id=1000: encoded in 2 bytes
order_id=1M:   encoded in 3 bytes

JSON: "order_id": 1000 = 15 bytes
Protobuf: field_tag + varint(1000) = 3 bytes
5× smaller!
```

### Code Example (Python)

```python
# Generate Python code from .proto:
# protoc --python_out=. order.proto

import order_pb2

# Encode:
order = order_pb2.Order()
order.order_id = 1001
order.customer.id = 5
order.customer.name = "Rahul Kumar"
order.total = 199.98

encoded = order.SerializeToString()
print(len(encoded))  # much smaller than JSON equivalent

# Decode:
decoded = order_pb2.Order()
decoded.ParseFromString(encoded)
print(decoded.customer.name)  # "Rahul Kumar"
```

---

## 63.5 Schema Evolution — The Most Important Feature

Real systems evolve. You add fields, remove fields, rename fields.
Protobuf handles this safely with field numbers:

```protobuf
// Version 1 of Order message:
message Order {
  int32 order_id = 1;
  int32 customer_id = 2;
  double total = 3;
}

// Version 2: add shipping_address, remove customer_id (replaced by customer object)
message Order {
  int32 order_id = 1;
  // customer_id = 2 is REMOVED — field number 2 is now "reserved"
  double total = 3;
  string shipping_address = 4;  // NEW field
  Customer customer = 5;        // NEW field

  reserved 2;                   // Never reuse field number 2!
  reserved "customer_id";       // Never reuse field name "customer_id"!
}
```

**Backward compatibility** (new code reads old data):
```
Old data (v1): has fields 1, 2, 3
New code (v2): expects fields 1, 3, 4, 5

New code reads old data:
  Field 1 (order_id): present → reads normally
  Field 2 (customer_id): present but reserved → skipped/ignored
  Field 3 (total): present → reads normally
  Field 4 (shipping_address): MISSING → uses default value ("")
  Field 5 (customer): MISSING → uses default (empty Customer)

→ Works! Old data is readable by new code.
```

**Forward compatibility** (old code reads new data):
```
New data (v2): has fields 1, 3, 4, 5
Old code (v1): expects fields 1, 2, 3

Old code reads new data:
  Field 1 (order_id): present → reads normally
  Field 3 (total): present → reads normally
  Field 4 (shipping_address): UNKNOWN field → SKIPPED (not rejected!)
  Field 5 (customer): UNKNOWN field → SKIPPED

→ Works! New fields are silently ignored by old code.
```

---

## 63.6 Apache Avro — Schema in Registry

Avro takes a different approach: the schema is NOT embedded in each record.
Instead, a schema registry stores schemas, and each record carries only a schema ID.

```json
// Avro schema (order.avsc):
{
  "type": "record",
  "name": "Order",
  "fields": [
    {"name": "order_id", "type": "int"},
    {"name": "customer_name", "type": "string"},
    {"name": "total", "type": "double"},
    {"name": "notes", "type": ["null", "string"], "default": null}
  ]
}
```

**Writer schema vs Reader schema:**
```
Writer (producer) encodes data using Writer Schema
Reader (consumer) decodes using Reader Schema (may be different version)

Avro resolves differences:
  Field in writer but not reader: ignored
  Field in reader but not writer: use default value
  Field renamed in reader: use "aliases" in schema

Example:
  Writer schema v1: {order_id, customer_name, total}
  Reader schema v2: {order_id, customer_name, total, discount=null}

  Reading v1 data with v2 schema:
  → discount field missing → use default null → works!
```

**Confluent Schema Registry:**
```python
from confluent_kafka.avro import AvroProducer, AvroConsumer

# Schema Registry stores all schema versions
# Producers register schemas on first use
# Each Kafka message: [magic_byte][schema_id_4_bytes][avro_data]

producer = AvroProducer({
    'bootstrap.servers': 'kafka:9092',
    'schema.registry.url': 'http://schema-registry:8081'
}, default_value_schema=order_schema)

producer.produce('orders', value={"order_id": 1001, "customer_name": "Rahul", "total": 199.98})
```

---

## 63.7 Comparison: JSON vs Protobuf vs Avro

| Property | JSON | Protobuf | Avro |
|----------|------|----------|------|
| Human readable | ✅ Yes | ❌ Binary | ❌ Binary |
| Schema required to decode | ❌ No | ✅ Yes (.proto) | ✅ Yes (Avro schema) |
| Schema evolution | ⚠️ Manual | ✅ Field numbers | ✅ Reader/Writer schema |
| Size vs JSON | 1× | 3–10× smaller | 3–7× smaller |
| Speed vs JSON | 1× | 5–10× faster | 3–5× faster |
| Language support | Universal | Google SDKs | Apache project |
| Best for | APIs, debug | gRPC, internal services | Kafka, Hadoop |

---

# CHAPTER 64 — Replication Deep Dive

---

## 64.1 Leader-Based Replication (Covered) vs Multi-Leader

We covered single-leader replication (one primary, multiple standbys).
Now: what if you need writes in multiple regions simultaneously?

---

## 64.2 Multi-Leader Replication

Also called **active-active** or **master-master** replication.
Multiple nodes accept writes. Each node also acts as a follower to others.

```
Use cases where multi-leader makes sense:
1. Multi-datacenter (each DC has its own leader — writes are local!)
2. Clients with offline operation (CouchDB, Google Docs — sync when reconnected)
3. Collaborative editing (Google Docs — each user is their own "leader")

Architecture — 2 leaders across regions:
  Mumbai Leader  ◄──────replication──────►  Singapore Leader
  Accepts writes                              Accepts writes
  Local reads (fast)                         Local reads (fast)
  Replicates to Singapore (async, ~50ms lag) Replicates to Mumbai (async, ~50ms lag)
```

### The Write Conflict Problem

```
Mumbai:    INSERT INTO posts (title) VALUES ('Hello World');  → post_id=1
Singapore: INSERT INTO posts (title) VALUES ('Hola Mundo');  → post_id=1 ← CONFLICT!
Both generate id=1, both replicate to each other — which one survives?

Strategies:
1. Last Write Wins (LWW): attach timestamp to each write, higher timestamp wins
   Risk: clock skew between servers means "later" may not be actually later
   Data loss: the losing write is silently discarded

2. On-write conflict detection: detect conflict during replication, reject one
   Require application to retry

3. On-read conflict detection (CouchDB style): store BOTH conflicting values
   Return BOTH to the application on read
   Application decides which to keep (or merge them)

4. Conflict-free by design: ensure same data can only be written in one region
   User with account in Mumbai → all their writes go to Mumbai ONLY
   No conflict possible (different regions never write same row)
   This is the recommended approach for most systems
```

---

## 64.3 Leaderless Replication — Dynamo Style

Popularized by Amazon Dynamo (2007). Used by Cassandra, Riak, DynamoDB.

**No single leader.** Any node accepts writes and reads.
**Quorum-based consistency**: require agreement from W write nodes and R read nodes.

```
Setup: 5 nodes (N=5)
Write quorum: W=3 (write succeeds if 3 nodes confirm)
Read quorum:  R=3 (read returns after 3 nodes respond)

Quorum rule: W + R > N  →  overlapping read/write sets → see latest write
  3 + 3 > 5 → 1 node always in common → reads always see latest write

Write operation (W=3):
  Client writes to ALL 5 nodes in parallel
  Waits for response from W=3 nodes
  Returns success to client
  Remaining 2 nodes may be slow/offline → they'll catch up later

Read operation (R=3):
  Client reads from ALL 5 nodes in parallel
  Waits for R=3 responses
  Takes the response with the highest version number (last-write-wins by version)

Failure scenarios:
  1 node down: W=3 from 4 healthy → still works (can lose 2 nodes and still function)
  2 nodes down: W=3 from 3 healthy → still works
  3 nodes down: W=3 from 2 healthy → FAILS (cannot reach quorum)
  Formula: can tolerate floor((N-1)/2) failures = 2 failures for N=5
```

### Read Repair and Anti-Entropy

```
How do lagging nodes catch up?

Read repair (during reads):
  Client reads from 3 nodes, gets different values:
    Node 1: version=5, value="Rahul"
    Node 2: version=5, value="Rahul"
    Node 3: version=3, value="Rahul K" ← stale!
  Client: detects node 3 is stale → writes latest value to node 3
  → Stale data corrected during normal reads

Anti-entropy process (background):
  Nodes continuously compare data using Merkle trees
  Find differences and sync missing updates
  Eventually all nodes converge to same state

Sloppy quorum (Dynamo):
  If W nodes in preferred set are unavailable:
  Write to W nodes from ANY available nodes (including non-preferred)
  When preferred nodes recover: hand off the writes (hinted handoff)
  → Higher availability but weaker consistency guarantees
```

---

## 64.4 Replication Lag Problems

Even with single-leader replication, async replication creates
temporary inconsistency that can cause real bugs:

### Read-Your-Own-Writes Problem

```
1. User (Rahul) submits a form → POST /api/profile → writes to PRIMARY
2. Browser redirects → GET /api/profile → reads from REPLICA
3. Replica hasn't received the write yet (50ms lag)
4. Rahul sees OLD profile → thinks his update was lost!

Solutions:
1. Always read from primary for 1 second after any write
2. Track last_write_timestamp in session; route to primary until replica catches up
3. Use same replica for same user (sticky routing)
4. For own-profile reads: always use primary
   For other users' profiles: can use replica

PostgreSQL implementation:
  Read from primary until replica_lsn >= write_lsn:
  SELECT pg_wal_replay_wait($write_lsn, timeout_ms := 5000);
  -- Waits up to 5 seconds for replica to catch up to write LSN
```

### Monotonic Reads Problem

```
1. Rahul reads post comments → gets 3 comments (from replica A, lag=100ms)
2. Priya refreshes → gets 1 comment (from replica B, lag=5s — further behind!)
3. Priya sees 2 comments DISAPPEAR compared to what Rahul saw

Solution: route same user to same replica (sticky routing by user_id)
  hash(user_id) % num_replicas → always same replica for same user
  Replica crash: reroute to next replica (user sees stale data briefly)
```

---

# CHAPTER 65 — Partitioning Deep Dive

---

## 65.1 Secondary Indexes on Partitioned Data

Secondary indexes on sharded data are the hardest part of sharding.

### Local Secondary Index (Index Per Shard)

```
Shard 1 (users 0-25%):        Shard 2 (users 25-50%):
  users: [user records]          users: [user records]
  local_index on city:           local_index on city:
    Mumbai → [user1, user5]        Mumbai → [user8, user12]
    Delhi  → [user3]               Delhi  → [user9]

Query: "Find all users in Mumbai"
  → Must query ALL shards (scatter)
  → Each shard returns its local Mumbai users
  → Coordinator merges results
  Cost: O(shards) queries per search, then merge
  Used by: MongoDB, Cassandra (local indexes)
```

### Global Secondary Index (Index Across Shards)

```
Users sharded by user_id:        Global city index (sharded by city):
  Shard 1: user_id 0-25%           City shard 1: cities A-M
  Shard 2: user_id 25-50%            Mumbai → [user1, user5, user8, user12]
  Shard 3: user_id 50-75%            Kolkata → [...]
  Shard 4: user_id 75-100%         City shard 2: cities N-Z
                                      New Delhi → [user3, user9]

Query: "Find all users in Mumbai"
  → Query city index (1 shard, hash('Mumbai') → city shard 1)
  → Get list of user_ids: [1, 5, 8, 12]
  → Query user shards for those IDs (scatter only to specific shards)
  Cost: 1 index query + N user shard queries (N = users in that city)

Problem with global secondary index:
  Write: when user_id=5 changes city Mumbai→Delhi:
    → Update user data in shard 1 (by user_id)
    → Update city index: remove from Mumbai list, add to Delhi list (different shard!)
  → Cross-shard write, cannot do atomically without distributed transaction
  Solution: accept eventual consistency for index updates
  Used by: DynamoDB Global Secondary Indexes, Elasticsearch
```

---

## 65.2 Rebalancing Partitions

When you add a shard, data must be redistributed. This is rebalancing.

### Fixed Number of Partitions

```
Pre-create many partitions, assign multiple to each node:
  10 nodes, 1000 partitions total
  Each node: ~100 partitions

Adding node 11:
  Move some partitions from existing nodes to node 11
  No rehashing needed (partition boundaries don't change)
  Only data that was in moved partitions needs to transfer

  Before: Node 1 has partitions [1, 45, 67, 200, ...]
  After:  Node 1 has partitions [1, 45, 200, ...]
          Node 11 gets partitions [67, ...] (moved from node 1)

Used by: Riak, Elasticsearch, Couchbase, Voldemort
```

### Dynamic Partitioning

```
Partitions split when they grow too large, merge when too small:
  HBase, MongoDB default behavior

  One partition initially (all data in one shard)
  When partition > 10GB: split into two ~5GB partitions
  Each half-partition assigned to a different node

Advantage: partitions adapt to data size
Disadvantage: new databases start with one partition → cold start problem
Solution: pre-split into N initial partitions for known key distribution
```

### Consistent Hashing with Rebalancing

```
Virtual nodes (vnodes) enable smooth rebalancing:

  Physical Node 1: has virtual nodes [v1, v5, v8, v12, ...]
  Physical Node 2: has virtual nodes [v2, v6, v9, v13, ...]
  Physical Node 3: has virtual nodes [v3, v7, v10, v14, ...]

Adding Physical Node 4:
  → Takes a few vnodes from each existing node
  → Only data in those vnodes transfers (not all data)
  → Each node gives up ~1/N of its load (fair)
  → Minimal data movement

Cassandra uses 256 virtual tokens per node by default
Rebalancing: gossip protocol coordinates vnode reassignment
```

---

## 65.3 Request Routing — How Clients Find the Right Shard

When a client sends a request, which shard handles it?

```
APPROACH 1: Client-side routing
  Client knows the partitioning scheme
  Client calculates: shard = hash(key) % num_shards
  Client connects directly to correct shard
  Advantage: no extra hop
  Disadvantage: client must be updated when partitioning changes

APPROACH 2: Routing tier (shard proxy)
  All requests go to a router
  Router knows where each partition lives
  Router forwards to correct shard

  Client → Router → Shard A
                  → Shard B
                  → Shard C

  Advantage: clients are simple (just talk to router)
  Disadvantage: extra network hop, router is potential bottleneck
  Used by: MongoDB mongos, Redis Cluster

APPROACH 3: Gossip protocol (any node can route)
  Client can connect to ANY node
  Each node knows the partition layout for all nodes
  If the connected node isn't responsible: forward to correct node
  Advantage: no single routing tier
  Used by: Cassandra (coordinator node handles routing)

Partition layout changes:
  ZooKeeper / etcd used to track which partition is on which node
  When partitions move: update ZooKeeper → routers/nodes learn new layout
  Example: Kafka uses ZooKeeper for broker/partition leader tracking
```

---

# CHAPTER 66 — The Trouble with Distributed Systems

---

## 66.1 WHY This Chapter is Critical

This is the most important chapter for any senior engineer working with distributed systems.
Every distributed system problem — mysterious timeouts, split-brain scenarios,
data loss on failover — traces back to the fundamental problems described here.

You cannot build reliable distributed systems without understanding
WHY they are fundamentally harder than single-machine systems.

---

## 66.2 Unreliable Networks

In a distributed system, nodes communicate over networks.
Networks are **asynchronous** and **unreliable**.

```
What can go wrong when you send a request over the network:

1. Request lost (network packet dropped)
   → Sender: waits forever or times out
   → Receiver: never knew request was sent

2. Request waiting in queue (network congestion)
   → Sender: times out, thinks request failed
   → Receiver: processes request 30 seconds later (as "late arrival")

3. Remote node crashed
   → Network: connection refused or timeout
   → Sender: cannot distinguish from network failure

4. Remote node paused (GC, page fault)
   → Request arrives, node is unresponsive for 10 seconds
   → Sender: times out, retries → now two copies of request processed!

5. Response lost (request succeeded, response dropped)
   → Sender: thinks request failed, retries
   → Receiver: processes same request TWICE

The fundamental problem:
  You CANNOT tell the difference between:
  a) The remote node crashed
  b) The network is slow
  c) The remote node is slow (GC pause, heavy load)

  All three look identical to the sender: timeout or no response.

Implication:
  Timeouts don't prove failure — they prove: "no response within N seconds"
  The request may have been processed; the response may be in transit
  Retrying may duplicate the operation
```

### Timeouts — No Correct Answer

```
Too short timeout:
  → Declare healthy nodes as failed (due to slow network)
  → Trigger unnecessary failovers
  → Risk cascading failure (each failover adds load to remaining nodes)

Too long timeout:
  → Wait minutes for a failed node
  → Users experience long delays
  → Resources held while waiting (locks, connections)

Adaptive timeouts (phi accrual failure detector):
  Track historical response times to calculate probability of failure
  Phi (φ) = -log10(probability of not receiving response by now)
  φ > 8: very likely failed (Cassandra uses φ=8)
  → Timeout adapts to observed network conditions
  → More accurate than fixed timeout

TCP timeout limitations:
  TCP keepalive: detects broken TCP connection
  But: it cannot detect: application crash, thread deadlock, infinite loop
  → Always use application-level timeouts AND heartbeats
```

---

## 66.3 Unreliable Clocks

In a distributed system, each node has its own clock.
Clocks are **not synchronized** and **can jump**.

```
Clock types:
  Time-of-day clock (wall clock):
    Returns current date/time
    Synchronized by NTP (Network Time Protocol)
    Can jump BACKWARD if NTP corrects a slow clock!
    Accuracy: ±1–100ms vs true UTC
    USE FOR: displaying time to users, calculating durations (with caution)
    DO NOT USE FOR: ordering events across machines

  Monotonic clock:
    Only moves forward (never jumps back)
    Not synchronized between machines (just counts "time since boot")
    USE FOR: measuring elapsed time (timeouts, latencies)
    DO NOT USE FOR: comparing timestamps across different machines

Why clocks are wrong:
  1. NTP sync: clocks corrected periodically, may jump by minutes if far off
  2. Leap seconds: time jumps by 1 second occasionally
  3. Virtual machines: clock may freeze during VM migration/pause
  4. Temperature: quartz clocks drift with temperature changes
  5. Leap smearing: some NTP servers spread leap second over 24 hours
```

### Clock Skew Causes Real Bugs

```
Scenario: Two nodes generate events with timestamps

Node A (clock: 10:00:00.100): writes row version 5
Node B (clock: 10:00:00.050): writes row version 6 (0.05 seconds "earlier" by clock)

If using Last Write Wins: Node A's write wins (later clock)
But Node B's write actually happened AFTER Node A's write!
→ Node B's update is silently discarded → data loss

Real PostgreSQL implication:
  Logical replication uses LSN (monotonic, per server) not timestamps
  Cross-cluster comparison: always use LSNs or explicit version numbers
  NEVER use timestamps to determine write order across machines

Lamport timestamps (logical clocks):
  Instead of real time, use a counter:
  Each event: increment counter, attach to event
  Receive message with counter C: max(my_counter, C) + 1
  → Captures causal ordering without real time
  → If event A caused event B: A's counter < B's counter (always)

Vector clocks:
  [node1_counter, node2_counter, node3_counter]
  Each node increments its own counter on each event
  Can detect causality AND concurrency:
    A=[1,0,0] → B=[2,0,0]: B happened after A (causal)
    A=[1,0,0] and C=[0,1,0]: concurrent (neither caused the other)
```

---

## 66.4 Process Pauses

A running process can be paused unpredictably:

```
Causes of process pauses:
  1. Garbage collection (Java/Go GC): 100ms–10 seconds for "stop the world" GC
  2. Virtual machine pause: hypervisor suspends VM for migration or maintenance
  3. OS scheduling: process swapped out when CPU busy with other processes
  4. Disk I/O: synchronous disk write blocks the thread
  5. Swap: paging memory to disk causes 100ms+ pauses

Why this breaks distributed logic:

Example — Leader election:
  Node A declares itself leader (based on ZooKeeper lease)
  Node A experiences GC pause for 15 seconds
  ZooKeeper lease expires after 10 seconds → Node B becomes leader
  Node A resumes, STILL thinks it's leader!
  → Two leaders simultaneously (split-brain)
  → Both write to the database

The "too much time has passed" problem:
  Any system that assumes "I'm still the leader because I was 100ms ago" is wrong
  In those 100ms: GC, network, OS scheduler — anything could have invalidated that assumption

Solution — Fencing tokens:
  Leader lease includes a monotonically increasing token
  Every write to the database includes the fencing token
  Database rejects writes with old tokens:

  Node A (old leader, token=33, paused): writes with token=33
  Node B (new leader, token=34): writes with token=34
  Database: receives write with token=33 AFTER receiving token=34 → rejects!
  → Old leader's writes are rejected even if it doesn't know it's dethroned

  This is how Zookeeper and etcd implement safe leader transitions
```

---

## 66.5 Byzantine Faults

So far: nodes crash or are slow. They don't lie.

Byzantine fault: a node sends **deliberately incorrect or malicious** messages.

```
Examples:
  1. Bug: hardware memory corruption sends wrong data (appears correct to sender)
  2. Malicious node: in a multi-organization system, one org's node cheats
  3. Tampered firmware: supply chain attack

Byzantine Fault Tolerance (BFT):
  System can tolerate f Byzantine (lying) nodes if total nodes >= 3f+1
  3 nodes: can tolerate 0 Byzantine faults
  4 nodes: can tolerate 1 Byzantine fault
  10 nodes: can tolerate 3 Byzantine faults

Cost: BFT algorithms are 10-100× more expensive than crash-fault-tolerant ones

Where BFT is needed:
  Blockchain: nodes don't trust each other (permissionless)
  Multi-organization systems: some orgs might cheat

Where BFT is NOT needed (but crash-fault-tolerance is):
  Internal microservices: we own all nodes, they don't lie (just crash)
  PostgreSQL, Kafka, etc.: designed for crash-fault-tolerance only

PostgreSQL assumption:
  All connected clients are trusted.
  If client sends bad SQL: it's rejected by type checking, not "Byzantine fault"
  If disk corrupts data: checksums detect it (not Byzantine fault handling)
```

---

# CHAPTER 67 — Consistency and Consensus

---

## 67.1 Linearizability — The Strongest Consistency

Linearizability (also called atomic consistency or strong consistency) means:
**the system behaves as if there is a single copy of the data.**

```
Non-linearizable (typical async replication):
  Client A writes x=1 to primary
  Client B reads x from replica: gets x=0 (replica hasn't synced yet)
  Client A reads x from replica: gets x=0 too!
  → Client A sees its own write vanish temporarily

Linearizable:
  Client A writes x=1 to primary
  Write returns → every subsequent read by ANY client gets x=1
  The write appears to have happened at a single instant in time
  No one ever sees x=0 after the write returns

Formal definition:
  Operations appear to happen instantaneously at some point
  between when they are invoked and when they return.
  The order must be consistent with real time.
```

### Where Linearizability is Required

```
1. Leader election (ZooKeeper):
   Exactly one node must believe it is the leader.
   Non-linearizable → two nodes think they are leader → split-brain

2. Distributed locks:
   Exactly one client holds the lock at a time.
   Non-linearizable → two clients both believe they hold the lock

3. Unique constraint enforcement (across shards):
   Username must be unique across all shards.
   Non-linearizable → two users register same username simultaneously

4. Bank account balance:
   Must never go below zero.
   Non-linearizable → balance check and deduction are not atomic across replicas
```

### The Cost of Linearizability

```
CAP theorem implication:
  During a network partition:
    Linearizable system: refuse requests that can't guarantee linearizability
    → Unavailable during partition (CP system)

    Non-linearizable system: return potentially stale data
    → Available but inconsistent (AP system)

Multi-leader and leaderless: NOT linearizable by design
  (they allow concurrent writes to multiple nodes → cannot guarantee global order)

Single-leader with sync replication: linearizable
  BUT: if leader and replica are partitioned from each other
  → Must stop accepting writes until partition heals
  → Unavailable during partition

Performance cost:
  Every operation must be coordinated across all replicas
  Each write: wait for all nodes to acknowledge
  Each read: must verify no concurrent writes in progress
  Latency: multiple network round trips
  → This is why Spanner charges a premium: atomic clocks reduce the coordination cost
```

---

## 67.2 The Raft Consensus Algorithm

Raft is the consensus algorithm used by etcd, CockroachDB, TiDB, Consul.
It was designed to be understandable (unlike Paxos).

### The Three Roles

```
LEADER: handles all client requests
FOLLOWER: passive, responds to leader's commands
CANDIDATE: trying to become leader (during election)

At any time: exactly 1 leader, rest are followers
             (or briefly: 1 candidate + followers during election)
```

### Leader Election

```
All nodes start as followers with election_timeout (~150-300ms random).

When election timeout expires:
  1. Follower increments term counter, becomes Candidate
  2. Votes for itself
  3. Sends RequestVote RPC to all other nodes

Each node votes YES if:
  → It hasn't voted this term yet
  → Candidate's log is at least as up-to-date as its own log

If candidate gets votes from majority (floor(N/2) + 1):
  → Becomes leader, sends heartbeats to prevent new elections

If no majority:
  → Election timeout again (random to prevent split-votes)

Randomized timeouts prevent split votes:
  Node A: timeout=150ms → wins election before others start
```

### Log Replication

```
CLIENT REQUEST: "SET x = 5"
  ↓
LEADER receives request:
  1. Appends log entry to its own log: [term=3, index=42, cmd="SET x=5"]
  2. Sends AppendEntries RPC to ALL followers
  3. Waits for majority to respond "appended"
  4. Commits: applies to state machine, updates x=5
  5. Returns success to client
  6. Next AppendEntries to followers: tells them entry 42 is committed

FOLLOWER receives AppendEntries:
  1. Verifies previous log entry matches (consistency check)
  2. Appends new entry to own log
  3. Responds OK

LEADER CRASH:
  1. Followers stop receiving heartbeats
  2. Election starts (first to time out becomes candidate)
  3. New leader must have ALL committed entries (by election rule)
  4. New leader's term is higher → old leader (if recovered) rejects its own AppendEntries
```

### Safety Guarantee

```
Raft's key safety property:
  "If a log entry is committed, it will be present in all future leaders' logs."

How: a candidate must have log at least as up-to-date as a MAJORITY of nodes.
     A committed entry is on a majority of nodes.
     → At least one node in any majority has the committed entry
     → Candidate can only win if it has that entry (or a more recent one)
     → Committed entries are never lost, even across leader changes

Practical result:
  You can safely COMMIT when: majority (3 of 5 nodes) acknowledge
  If leader crashes after commit: new leader has that entry guaranteed
  If leader crashes before majority: entry is not committed, may be lost
```

---

## 67.3 ZooKeeper — Distributed Coordination Service

ZooKeeper provides a tree-structured key-value store with
**linearizable reads and writes**, used as a coordination service.

```
ZooKeeper namespace (like a filesystem):
/
├── /services/
│   ├── /services/my-app/
│   │   ├── /services/my-app/node-1  ← ephemeral node (leader=node-1)
│   │   └── /services/my-app/node-2  ← ephemeral node (follower)
├── /locks/
│   └── /locks/resource-A            ← distributed lock
└── /config/
    └── /config/db-url               ← centralized config

Node types:
  Persistent: stays until explicitly deleted
  Ephemeral: deleted when creating client session ends (client crashes → node gone!)
  Sequential: gets auto-incremented suffix (/locks/lock-000000001, etc.)

Watches:
  Client sets watch on a node
  ZooKeeper notifies client when node changes/deleted
  One-time: watch fires once, client must re-register

Leader election using ZooKeeper:
  1. All nodes create ephemeral sequential nodes under /election/:
     node A: /election/node-000000001
     node B: /election/node-000000002
     node C: /election/node-000000003

  2. Node with LOWEST number is leader

  3. Each non-leader watches the node JUST BEFORE it:
     node B watches node A
     node C watches node B

  4. If leader (node A) crashes: ephemeral node deleted
     node B is notified, checks: is it now the lowest? YES → becomes leader

  5. Result: exactly one leader at all times, no split-brain
```

---

# CHAPTER 68 — Stream Processing

---

## 68.1 WHY Stream Processing

Batch processing (MapReduce, Spark) processes bounded datasets.
You wait until you have all the data, then process it.

```
Batch processing:
  Collect all orders for January → run report → publish → users see January data
  Latency: hours to days
  Use case: end-of-day reports, data warehouse loads, backfill operations

Stream processing:
  Process each order the moment it arrives → update report continuously
  Latency: milliseconds to seconds
  Use case: real-time dashboards, fraud detection, live notifications
```

---

## 68.2 Apache Kafka — The Durable Message Log

Kafka is not just a message queue. It is a **distributed, durable, ordered log**.

### Core Concepts

```
TOPIC: a named, ordered, immutable log of messages
  Orders topic: [msg1, msg2, msg3, msg4, msg5, ...]

PARTITION: topic is split into partitions (sharded)
  Orders topic, partition 0: [msg1, msg3, msg5, ...]  (key hash=even)
  Orders topic, partition 1: [msg2, msg4, msg6, ...]  (key hash=odd)

  Partitions enable parallelism: each partition consumed by ONE consumer in a group

OFFSET: position in a partition (monotonically increasing integer)
  msg at offset 42 in partition 0 = orders-0-42
  Consumers track their offset → can resume from where they stopped

BROKER: Kafka server node
  Cluster: typically 3–6 brokers
  Each partition: one broker is leader, others are ISR (in-sync replicas)

PRODUCER: writes messages to a topic
CONSUMER: reads messages from a topic
CONSUMER GROUP: set of consumers dividing partitions among themselves
```

### Message Flow

```
PRODUCER → KAFKA:

  1. Producer decides partition: hash(message_key) % num_partitions
     OR: round-robin (if no key)
     OR: custom partitioner

  2. Producer sends batch to partition leader broker:
     {topic: "orders", partition: 2, messages: [...]}

  3. Broker appends to partition log (sequential write to disk — fast!)

  4. ISR replicas fetch and replicate

  5. When min.insync.replicas have acknowledged: marked as committed

  6. Broker responds to producer: success + offset

CONSUMER → KAFKA:

  1. Consumer group subscribes to topic
  2. Kafka assigns partitions to consumers (one partition per consumer in group)
     3 partitions, 3 consumers → 1 partition each
     3 partitions, 2 consumers → one consumer gets 2 partitions
     3 partitions, 4 consumers → one consumer idle

  3. Consumer fetches from its assigned partitions, tracks offset
  4. On success: commits offset to Kafka (__consumer_offsets topic)
  5. On restart: resumes from last committed offset
```

### Why Kafka is Durable and Replayable

```
Traditional message queues: delete message after consumer acknowledges
Kafka: messages STAY on disk (configurable retention: days, weeks, forever)

Benefits of persistence:
  1. Multiple consumer groups read the same topic independently
     Orders topic → Group A (inventory service) → Group B (analytics) → Group C (email)
     Each group has its own offset → doesn't affect others

  2. Replay: re-process all orders from 30 days ago
     Reset consumer offset → consume from 30 days ago offset
     Impossible with traditional queues

  3. Fault recovery: consumer crashes → resume from last committed offset
     Only re-process messages since last commit (at-least-once)

  4. Debugging: go back in time to see exactly what messages caused a bug

Retention configuration:
  retention.ms = 604800000    # 7 days
  retention.bytes = 1073741824  # 1GB per partition
  log.cleanup.policy = delete   # OR compact (keep only latest per key)
```

---

## 68.3 Exactly-Once Semantics

Delivering a message exactly once is the hardest problem in distributed systems.

```
At-most-once:  message delivered 0 or 1 times. May be lost. Fast.
At-least-once: message delivered 1 or more times. May be duplicated. Common.
Exactly-once:  message delivered exactly 1 time. Hard. Requires coordination.

At-least-once problems:
  Consumer processes message, crashes BEFORE committing offset
  → Consumer restarts, reprocesses same message
  → Duplicate processing: same order processed twice = double charge!

Idempotent consumer (at-least-once + deduplication = effectively exactly-once):
  Design operations to be safe to repeat:
  INSERT INTO orders ... ON CONFLICT (order_id) DO NOTHING
  → Duplicate order_id → silently ignored → safe to retry

Kafka transactions (true exactly-once):
  Producer transactional API:
    producer.initTransactions()
    producer.beginTransaction()
    producer.send(record1)
    producer.send(record2)
    consumer.commitOffset(offsets, transaction)  ← commit offset atomically with sends!
    producer.commitTransaction()

  Result: all messages sent AND offset committed atomically
  Either all succeed or all rolled back → no duplicates, no loss

Kafka Streams: handles exactly-once automatically if configured:
  processing.guarantee = exactly_once_v2
```

---

## 68.4 Stream Processing Concepts

### Time in Stream Processing

```
Event time: when the event actually happened (embedded in the message)
Processing time: when the event is processed by the stream processor

Why they differ:
  Mobile app sends order while offline → order arrives 30 minutes later
  Event time: 10:00:00 (when user tapped "Buy")
  Processing time: 10:30:00 (when Kafka received the message)

For correct analytics: always use EVENT TIME
  "Revenue per hour" should group by the hour the purchase was made
  Not the hour Kafka received it

Late arrivals:
  How long to wait for late events?
  At 10:05: process all 10:00-10:01 events (1 minute window is "complete"?)
  At 10:35: order arrives with event_time=10:00:00 (30 minutes late!)

  Watermark: "I've seen all events up to time T minus allowed_lateness"
  Allowed lateness = 30 minutes
  At 10:35: watermark is 10:05
  Late order (10:00) arrives before watermark → included
  Very late order (10:00) arrives at 11:30 → after watermark → dropped or sent to side output
```

### Windows in Stream Processing

```
TUMBLING WINDOW: fixed-size, non-overlapping
  [10:00-10:01) [10:01-10:02) [10:02-10:03) ...
  Revenue per minute: sum all orders in each 1-minute window
  No overlap between windows

HOPPING WINDOW: fixed-size, overlapping
  Window size=5min, hop=1min:
  [10:00-10:05) [10:01-10:06) [10:02-10:07) ...
  5-minute rolling average updated every minute

SLIDING WINDOW: window moves with each event
  "Orders in the last 5 minutes from THIS order"
  Each event has its own window based on its event time

SESSION WINDOW: activity-based, gaps define boundaries
  User activity: click at 10:00, 10:01, 10:02 | gap 30min | click at 10:35, 10:36
  Session 1: [10:00-10:02] | Session 2: [10:35-10:36]
  Gap inactivity threshold: 10 minutes (configurable)
  Use case: user session analytics, funnel analysis
```

### Stream-Table Joins

```
Stream: incoming orders (high volume, real-time)
Table: product catalog (relatively static, changes infrequently)

Join: enrich each order with product details

Option 1: Join at processing time (lookup from DB)
  For each order in stream:
    product = redis.get(f"product:{order.product_id}")  # cache
    enriched_order = {**order, "product_name": product.name}
  → Requires cache/DB call per message → latency, DB load

Option 2: Stream-table join (Kafka Streams, Flink)
  Load product table into local state store (RocksDB instance)
  For each order:
    product = local_store.get(order.product_id)  # in-process!
    enriched_order = {**order, "product_name": product.name}
  → No network call, microsecond lookup
  → State store updated when product catalog changes (via CDC or Kafka compacted topic)
```

---

## 68.5 Kafka in Production Architecture

```
COMPLETE EVENT-DRIVEN ARCHITECTURE:

     [PostgreSQL Primary]
           │
           │ CDC via Debezium (reads WAL)
           ▼
     [Kafka: orders topic]  [Kafka: products topic]  [Kafka: payments topic]
           │                        │                        │
    ┌──────┴────────────────────────┴────────────────────────┘
    │
    ├──► [Inventory Service Consumer] → updates stock in PostgreSQL
    │      at-least-once, idempotent: ON CONFLICT DO NOTHING
    │
    ├──► [Email Service Consumer] → sends order confirmation email
    │      exactly-once: deduplication table (message_id → processed)
    │
    ├──► [Analytics Consumer] → writes to BigQuery via batch sink
    │      at-least-once OK (analytics tolerates duplicates)
    │
    ├──► [Real-time Dashboard Consumer] → updates Redis counters
    │      at-least-once OK (counters are approximate)
    │
    └──► [Data Warehouse Sink] → nightly load to Snowflake
           micro-batching: every 5 minutes to reduce Snowflake write costs
```

---

# End of Ch 63–68

---

# CHAPTER 69 — Probabilistic Data Structures

---

## 69.1 WHY Probabilistic Structures Exist

Exact data structures (hash sets, B-Trees) give 100% correct answers
but require memory proportional to the number of elements.

```
Exact: "Has user 5 seen article 12345?"
  Store: SET of (user_id, article_id) pairs
  1 billion users × 100 articles each = 100 billion pairs
  At 16 bytes each = 1.6 TB RAM — impossible!

Probabilistic: trade perfect accuracy for massive memory reduction
  Bloom filter: "Probably yes" (0.1% false positive rate) or "Definitely no"
  Memory: 1.2 GB for 100 billion entries with 0.1% false positive rate
  1300× smaller than exact structure
```

---

## 69.2 Bloom Filter

A Bloom filter answers: "**Is this element possibly in the set?**"
- If it says NO → the element is **definitely NOT** in the set
- If it says YES → the element is **probably** in the set (small chance of false positive)

### How It Works

```
Data structure: bit array of M bits, initialized to all 0s
Operations: insert(x) and query(x)
Uses K hash functions

INSERT("apple"):
  hash1("apple") = 3    → set bit[3] = 1
  hash2("apple") = 11   → set bit[11] = 1
  hash3("apple") = 17   → set bit[17] = 1

INSERT("banana"):
  hash1("banana") = 7   → set bit[7] = 1
  hash2("banana") = 11  → set bit[11] = 1  (already 1)
  hash3("banana") = 22  → set bit[22] = 1

QUERY("apple"):
  hash1("apple") = 3    → bit[3] = 1 ✓
  hash2("apple") = 11   → bit[11] = 1 ✓
  hash3("apple") = 17   → bit[17] = 1 ✓
  All bits set → "POSSIBLY in set" ✓ (correct!)

QUERY("cherry"):
  hash1("cherry") = 7   → bit[7] = 1 ✓ (set by "banana")
  hash2("cherry") = 3   → bit[3] = 1 ✓ (set by "apple")
  hash3("cherry") = 22  → bit[22] = 1 ✓ (set by "banana")
  All bits set → "POSSIBLY in set" ✗ (FALSE POSITIVE! cherry was never inserted)

QUERY("grape"):
  hash1("grape") = 5    → bit[5] = 0
  At first 0 bit: STOP → "DEFINITELY NOT in set" ✓ (guaranteed correct!)
```

### False Positive Rate

```
False positive probability ≈ (1 - e^(-kn/m))^k
Where:
  k = number of hash functions
  n = number of inserted elements
  m = bit array size

Example: m=1000 bits, k=3 hash functions, n=100 elements:
  False positive rate ≈ 1.2% (1 in 83 queries is a false positive)

Optimal k for given m and n: k = (m/n) × ln(2) ≈ 0.693 × (m/n)
Trade space for accuracy: double m → false positive rate drops exponentially

Rule of thumb: 10 bits per element → ~1% false positive rate
               20 bits per element → ~0.0001% false positive rate
```

### Real-World Use Cases

```python
# Use case 1: Cassandra LSM-Tree (avoid reading disk)
# Problem: checking if a key exists requires reading SSTables (disk I/O)
# Solution: each SSTable has a Bloom filter
#   Query key → if Bloom filter says NO → skip this SSTable entirely
#   Only read from SSTable if Bloom filter says POSSIBLY YES
# Result: 99% of disk reads eliminated for missing keys

# Use case 2: Web crawler deduplication
# Problem: have I crawled this URL before? (billions of URLs)
# Solution: Bloom filter of crawled URLs
#   If Bloom says NO → crawl it
#   If Bloom says YES → probably already crawled → skip
# False positives: occasionally skip uncrawled URLs (acceptable)
# False negatives: NEVER happen → never re-crawl a crawled URL

# Use case 3: Chrome Safe Browsing
# Problem: is this URL in the blocklist? (checking Google server has latency/privacy)
# Solution: Chrome downloads a Bloom filter of all malicious URLs
#   Check locally in microseconds
#   False positive → verify with Google server (rare)
#   False negative → NEVER → no malicious URL is missed

# Python implementation:
import hashlib

class BloomFilter:
    def __init__(self, size: int, num_hashes: int):
        self.size = size
        self.num_hashes = num_hashes
        self.bit_array = [0] * size

    def _hashes(self, item: str):
        for seed in range(self.num_hashes):
            h = hashlib.md5(f"{seed}:{item}".encode()).hexdigest()
            yield int(h, 16) % self.size

    def add(self, item: str):
        for idx in self._hashes(item):
            self.bit_array[idx] = 1

    def might_contain(self, item: str) -> bool:
        return all(self.bit_array[idx] for idx in self._hashes(item))

bf = BloomFilter(size=10000, num_hashes=3)
bf.add("user:5:article:123")
bf.add("user:5:article:456")
print(bf.might_contain("user:5:article:123"))  # True (definitely yes or false positive)
print(bf.might_contain("user:5:article:789"))  # False (definitely NOT seen)
```

---

## 69.3 HyperLogLog (HLL) — Counting Uniques

HyperLogLog answers: "**How many distinct elements are there?**"
with only logarithmic memory.

### The Problem

```
Exact unique count: store all seen values in a HashSet
  1 billion unique user IDs × 8 bytes = 8 GB memory

HyperLogLog: ~12 KB for any number of distinct elements
  Accuracy: ±0.81% standard error
  1 billion unique users → HLL reports: 999,185,000 – 1,000,815,000 (within 0.1%)
```

### Intuition Behind HLL

```
Insight: if you hash elements uniformly to binary strings,
the probability of seeing a string starting with k zeros is 1/2^k.

If the MAXIMUM run of leading zeros you've seen is k,
then you've probably seen about 2^k distinct elements.

Example:
  Elements hashed to: 00101, 11001, 01101, 00011, 10001
  Maximum leading zeros: 2 (in "00011" and "00101")
  Estimate: 2^2 = 4 ≈ 5 actual elements (rough estimate)

HyperLogLog improvements:
  → Uses multiple hash functions (divides into m = 2^b buckets)
  → Uses harmonic mean of all bucket estimates (reduces variance)
  → Result: ±0.81% / sqrt(m) standard error

Standard error of 0.81% with m=16384 registers:
  0.81% / sqrt(16384) = 0.81% / 128 = 0.0063% error
```

### PostgreSQL HyperLogLog

```sql
-- Redis: built-in PFADD and PFCOUNT
-- PFADD: add element to HLL
PFADD page_views:2024-01-15 user:1001
PFADD page_views:2024-01-15 user:1002
PFADD page_views:2024-01-15 user:1001  -- duplicate, ignored

-- PFCOUNT: approximate distinct count
PFCOUNT page_views:2024-01-15  -- returns ~2 (approximate)

-- Merge multiple HLL counters:
PFMERGE page_views:week PFCOUNT page_views:2024-01-13 page_views:2024-01-14 page_views:2024-01-15
PFCOUNT page_views:week  -- unique users across 3 days (de-duplicated!)

-- PostgreSQL with hll extension:
CREATE EXTENSION hll;

CREATE TABLE daily_visitors (
    day DATE PRIMARY KEY,
    visitors hll
);

INSERT INTO daily_visitors (day, visitors)
VALUES ('2024-01-15', hll_empty());

-- Add user visits:
UPDATE daily_visitors
SET visitors = hll_add(visitors, hll_hash_bigint(user_id))
WHERE day = '2024-01-15';

-- Count distinct visitors:
SELECT day, hll_cardinality(visitors)::BIGINT AS unique_visitors
FROM daily_visitors;

-- Weekly unique visitors (merge HLLs from 7 days):
SELECT hll_cardinality(hll_union_agg(visitors))::BIGINT AS weekly_uniques
FROM daily_visitors
WHERE day >= '2024-01-09' AND day <= '2024-01-15';
-- This correctly de-duplicates! User who visited Mon AND Wed is counted ONCE.
```

---

## 69.4 Count-Min Sketch — Frequency Estimation

Count-Min Sketch answers: "**How many times have I seen element X?**"
with sub-linear memory.

### How It Works

```
Data structure: 2D array of counters: d rows × w columns
                d = number of hash functions
                w = width (higher = more accurate)

INSERT("product:123", count=3):
  row 0: col = hash0("product:123") % w → increment cell [0][col] by 3
  row 1: col = hash1("product:123") % w → increment cell [1][col] by 3
  row 2: col = hash2("product:123") % w → increment cell [2][col] by 3

QUERY("product:123"):
  row 0: read cell [0][hash0("product:123") % w]
  row 1: read cell [1][hash1("product:123") % w]
  row 2: read cell [2][hash2("product:123") % w]
  Return MINIMUM of all three values

Why minimum?
  Cells may be incremented by OTHER elements that hash to same column (collision)
  Minimum eliminates most collision overcount
  Result: always >= true count (overestimate, never underestimate)
  Error: true_count ≤ estimate ≤ true_count + ε × N
    where ε = 2/w and N = total elements inserted
```

### Real-World Use Cases

```python
# Use case 1: Top-K queries (heavy hitters)
# Problem: which 10 products are ordered most in the last hour?
# Solution: Count-Min Sketch tracks frequency of each product
#   Each order: increment sketch for product_id
#   Find top-K: maintain a min-heap of size K
#   Every insert: if frequency > heap minimum → replace

# Use case 2: Rate limiting per IP
# Problem: count requests per IP address in a sliding window
# Use case: attack with many IPs → Count-Min tracks them all in fixed memory

# use case 3: Network monitoring
# "Which IP pairs exchange the most traffic?" (elephant flows)
# Exact tracking: millions of IP pairs × 8 bytes = too much memory
# Count-Min: fixed memory, approximate counts, detect top talkers

# Python implementation:
import hashlib

class CountMinSketch:
    def __init__(self, width: int = 1000, depth: int = 5):
        self.width = width
        self.depth = depth
        self.table = [[0] * width for _ in range(depth)]

    def _hash(self, item: str, row: int) -> int:
        h = hashlib.sha256(f"{row}:{item}".encode()).hexdigest()
        return int(h, 16) % self.width

    def add(self, item: str, count: int = 1):
        for row in range(self.depth):
            self.table[row][self._hash(item, row)] += count

    def estimate(self, item: str) -> int:
        return min(self.table[row][self._hash(item, row)] for row in range(self.depth))

cms = CountMinSketch(width=10000, depth=5)
# Track page views:
cms.add("product:123", 1)
cms.add("product:456", 1)
cms.add("product:123", 1)  # product 123 viewed twice
print(cms.estimate("product:123"))  # returns 2 (approximately)
print(cms.estimate("product:999"))  # returns 0 or small number
```

---

## 69.5 Consistent Hashing — Deep Dive

We covered consistent hashing briefly. Here is the complete algorithm:

```
THE RING:

  Hash space: 0 to 2^32 - 1 (arranged in a circle)

  Node placement: hash(node_name) → position on ring
    Node A: hash("nodeA") = 1000000
    Node B: hash("nodeB") = 2000000
    Node C: hash("nodeC") = 3000000

  Key assignment: hash(key) → position on ring → go clockwise to first node
    hash("user:5") = 1500000 → clockwise → hits Node B → assigned to B
    hash("user:8") = 2800000 → clockwise → hits Node C → assigned to C
    hash("user:2") = 500000  → clockwise → hits Node A → assigned to A

ADDING NODE D:
  Node D placed at position 2500000 (between B and C)
  Keys in range (2000000, 2500000] that were assigned to C → now assigned to D
  Only 1/N fraction of keys moves: very little data migration!

REMOVING NODE B:
  Keys that were on Node B → now assigned to Node C (next clockwise)
  Only B's keys move: 1/N fraction

VIRTUAL NODES (vnodes):
  Problem: with 3 nodes, key distribution may be uneven
           if Node A's hash happens to cover 50% of the ring

  Solution: each physical node gets V virtual positions:
    Node A: hash("nodeA-0")=100, hash("nodeA-1")=1050, hash("nodeA-2")=2030, ...
    Node B: hash("nodeB-0")=200, hash("nodeB-1")=1100, ...
    Node C: hash("nodeC-0")=500, hash("nodeC-1")=1200, ...

    With V=150 virtual nodes per physical node: distribution is statistically uniform
    Adding/removing a node: 150 positions added/removed, data from/to multiple nodes
    Each physical node contributes/absorbs ~1/N of total data (balanced)

BOUNDED LOAD (Google):
  Standard consistent hashing: if node A is fast, keys don't rebalance
  Bounded load: if a node's load exceeds (1+ε) × average_load:
    Overflow keys are assigned to next node on the ring
    Naturally load-balances without manual rebalancing
  Used by: Google services, Nginx consistent hashing
```

---

# CHAPTER 70 — Rate Limiting

---

## 70.1 WHY Rate Limiting

```
Without rate limiting:
  Malicious user: sends 100,000 requests/second to /api/login
  → Tries passwords until one works (credential stuffing)
  → Database overwhelmed → legitimate users can't log in (DoS)

  Bug in client code: infinite retry loop → millions of requests
  → Cascades through the system

Rate limiting: each client can send at most N requests per time window
  → Protects backend from abuse AND bugs
  → Ensures fair resource sharing between clients
```

---

## 70.2 Token Bucket Algorithm

The most commonly used rate limiting algorithm.

```
Concept:
  Each client has a "bucket" that holds up to CAPACITY tokens
  Tokens refill at RATE per second
  Each request consumes 1 token
  If bucket empty: reject request (or wait)

Example: capacity=10, rate=2 tokens/second

  t=0s:  bucket=10 (full)
  → 10 requests arrive instantly: all accepted (consume all 10 tokens)
  → bucket=0

  t=0s:  11th request: REJECTED (bucket empty)

  t=1s:  bucket=2 (refilled 2 tokens)
  → 2 requests: accepted
  → bucket=0

  t=5s:  bucket=10 (capped at capacity, can't exceed 10)

Properties:
  ✅ Allows short bursts (up to CAPACITY requests instantly)
  ✅ Smooth long-term rate (RATE per second average)
  ✅ Simple to implement
```

### Redis Implementation

```python
import redis
import time

r = redis.Redis()

def is_allowed(user_id: str, capacity: int = 10, rate: float = 2.0) -> bool:
    key = f"rate_limit:{user_id}"
    now = time.time()

    pipe = r.pipeline()
    pipe.hgetall(key)
    pipe.execute()  # We use a Lua script for atomicity

    # Atomic Lua script (runs as single Redis command):
    lua_script = """
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])
    local rate = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    local bucket = redis.call('HGETALL', key)
    local tokens, last_refill

    if #bucket == 0 then
        tokens = capacity
        last_refill = now
    else
        tokens = tonumber(bucket[2])
        last_refill = tonumber(bucket[4])
    end

    -- Refill tokens based on elapsed time
    local elapsed = now - last_refill
    tokens = math.min(capacity, tokens + elapsed * rate)

    if tokens >= 1 then
        tokens = tokens - 1
        redis.call('HSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)
        return 1  -- allowed
    else
        return 0  -- rejected
    end
    """

    result = r.eval(lua_script, 1, key, capacity, rate, now)
    return result == 1
```

---

## 70.3 Sliding Window Log Algorithm

More accurate than token bucket but higher memory usage.

```
Data structure: sorted set of request timestamps per client

REQUEST arrives from user:
  1. Remove timestamps older than (now - window_size)
  2. Count remaining timestamps = requests_in_window
  3. If count < limit: allow + add current timestamp
  4. Else: reject

Example: limit=5 requests per minute

  t=0:10: request → log=[0:10] → count=1 → allow
  t=0:20: request → log=[0:10, 0:20] → count=2 → allow
  t=0:30: request → log=[0:10, 0:20, 0:30] → count=3 → allow
  t=0:40: request → log=[0:10, 0:20, 0:30, 0:40] → count=4 → allow
  t=0:50: request → log=[0:10, ..., 0:50] → count=5 → allow
  t=0:55: request → log=[0:10, ..., 0:50] → count=5 → REJECT

  t=1:15: request → remove timestamps < 0:15 → log=[0:20, 0:30, 0:40, 0:50] → count=4 → allow

Redis implementation (sorted set):
  ZADD rate_limit:{user_id} {now_ms} {now_ms}
  ZREMRANGEBYSCORE rate_limit:{user_id} -inf {now_ms - window_ms}
  count = ZCARD rate_limit:{user_id}
  EXPIRE rate_limit:{user_id} {window_seconds}
  if count <= limit: allow else reject
```

---

## 70.4 Distributed Rate Limiting

Single-server rate limiting is simple. Multiple app servers sharing rate limits is hard.

```
Problem:
  Limit: 100 requests/second per user
  3 app servers: each has local rate limiter
  User sends 300 requests → 100 to each server → each allows 100 = 300 total!

Solutions:

Option 1: Centralized Redis (most common)
  All rate limit state in Redis
  Each app server checks Redis before processing request
  Cost: one Redis call per request (~0.5ms overhead)
  Single Redis: 500k ops/second capacity → sufficient for most

Option 2: Redis Cluster with sticky routing
  Route user_id → always same Redis node via consistent hashing
  Avoids hot spots on single Redis node

Option 3: Approximate local counting with sync
  Each server keeps local counter, syncs to Redis every 1 second
  Local check: fast (no network)
  May allow slightly more than limit between syncs: acceptable approximation
  Good for: high-traffic scenarios where ±5% accuracy is acceptable

Option 4: Token sharing (Envoy/Nginx approach)
  Centralized token generator (Redis)
  Each app server pre-fetches batch of tokens (e.g., 10 at a time)
  Use local tokens until depleted, then fetch more
  Reduces Redis calls by 10×
  Trade-off: brief over-usage when multiple servers pre-fetch simultaneously
```

---

# CHAPTER 71 — Distributed Locking

---

## 71.1 WHY Distributed Locks

```
Single-server lock: mutex/semaphore works within one process
Distributed lock: coordinate multiple servers accessing shared resource

Use cases:
  Job scheduler: ensure only ONE server runs a cron job at a time
  Leader election: ensure one server is the "leader" for a task
  Payment processing: ensure a payment is not processed twice
  Inventory: "get and decrement" stock atomically across services
```

---

## 71.2 Redis SETNX — Simple Distributed Lock

```python
import redis
import uuid
import time

r = redis.Redis()

def acquire_lock(lock_name: str, ttl_seconds: int = 30) -> str | None:
    """Returns lock token if acquired, None if not."""
    token = str(uuid.uuid4())  # unique token per lock holder
    key = f"lock:{lock_name}"

    # SET key value NX EX ttl
    # NX = only set if key doesn't exist
    # EX = expire after ttl seconds (prevents deadlock if holder crashes)
    acquired = r.set(key, token, nx=True, ex=ttl_seconds)

    if acquired:
        return token  # we got the lock
    return None  # lock held by someone else

def release_lock(lock_name: str, token: str) -> bool:
    """Release lock ONLY if we own it (compare-and-delete)."""
    key = f"lock:{lock_name}"

    # Must be atomic: check token then delete
    # Use Lua script to prevent race condition:
    lua_script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    result = r.eval(lua_script, 1, key, token)
    return result == 1

def extend_lock(lock_name: str, token: str, ttl_seconds: int = 30) -> bool:
    """Extend lock TTL if we still own it (heartbeat pattern)."""
    key = f"lock:{lock_name}"
    lua_script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("expire", KEYS[1], ARGV[2])
    else
        return 0
    end
    """
    result = r.eval(lua_script, 1, key, token, ttl_seconds)
    return result == 1

# Usage pattern:
def run_scheduled_job():
    token = acquire_lock("daily_report_job", ttl_seconds=300)  # 5 min max
    if token is None:
        print("Another server is running this job, skipping")
        return

    try:
        # Start heartbeat thread to extend lock while working
        generate_daily_report()
    finally:
        release_lock("daily_report_job", token)
```

---

## 71.3 Redlock Algorithm — Multi-Node Distributed Lock

Single Redis node: if it crashes → lock state lost → no one can acquire lock.

Redlock uses N Redis nodes (typically 5) for fault tolerance:

```python
# Redlock: acquire lock on majority of N Redis nodes

import redis
import time
import uuid

redis_nodes = [
    redis.Redis(host='redis1', port=6379),
    redis.Redis(host='redis2', port=6379),
    redis.Redis(host='redis3', port=6379),
    redis.Redis(host='redis4', port=6379),
    redis.Redis(host='redis5', port=6379),
]

def redlock_acquire(lock_name: str, ttl_ms: int = 30000) -> tuple[str, int] | None:
    token = str(uuid.uuid4())
    key = f"lock:{lock_name}"
    start = time.time_ns() // 1_000_000  # ms

    acquired_count = 0
    for node in redis_nodes:
        try:
            if node.set(key, token, nx=True, px=ttl_ms):
                acquired_count += 1
        except Exception:
            pass  # node unavailable

    elapsed = (time.time_ns() // 1_000_000) - start
    validity = ttl_ms - elapsed - 10  # small safety margin

    # Need majority AND enough time left
    if acquired_count >= 3 and validity > 0:
        return token, validity  # acquired successfully
    else:
        # Failed: release whatever we acquired
        redlock_release(lock_name, token)
        return None

def redlock_release(lock_name: str, token: str):
    key = f"lock:{lock_name}"
    lua = """
    if redis.call("get",KEYS[1]) == ARGV[1] then
        return redis.call("del",KEYS[1])
    else return 0 end
    """
    for node in redis_nodes:
        try:
            node.eval(lua, 1, key, token)
        except Exception:
            pass

# Redlock safety: even if 2 of 5 Redis nodes crash:
# - Can still acquire lock (need 3 of 5)
# - Lock state preserved on 3+ nodes
# - When crashed nodes recover: lock has expired (TTL) → safe
```

### Redlock Controversy

Martin Kleppmann (DDIA author) argues Redlock is NOT safe under:
- GC pauses (process paused between acquiring lock and using it)
- Clock drift between Redis nodes (TTL expires at different times)

For truly safe distributed locking: use ZooKeeper or etcd with fencing tokens.
For "good enough" locking (most practical cases): Redis SETNX is fine.

---

# CHAPTER 72 — Circuit Breaker Pattern

---

## 72.1 WHY Circuit Breaker

```
Without circuit breaker:
  Database is overloaded → queries take 30 seconds → timeout
  Every app server: keeps sending queries → each times out after 30s
  → Thread pool exhausted waiting for timeouts → other features break
  → Cascading failure: one slow DB brings down the whole application

With circuit breaker:
  Database is slow → first 10 calls time out → circuit OPENS
  While open: immediately reject DB calls (fast fail, no waiting)
  After 30 seconds: try one request ("half-open")
  If it succeeds: close circuit (resume normal operation)
  If it fails: stay open for another 30 seconds

Result: DB problems are contained, other features still work
```

---

## 72.2 Circuit Breaker States

```
                    ┌──────────────────────────────────────┐
                    │                                      │
    ┌──────────────►│          CLOSED                      │◄─────────────────┐
    │               │  (normal operation)                  │                  │
    │               │  Track: failures, successes          │                  │
    │               │  Open if: failure_rate > threshold   │  success         │
    │               └──────────────────────────────────────┘                  │
    │                              │                                           │
    │                       failure threshold                                  │
    │                        exceeded                                          │
    │                              │                                           │
    │                              ▼                                           │
    │               ┌──────────────────────────────────────┐                  │
    │               │                                      │                  │
    │               │          OPEN                        │                  │
    │               │  (fast-fail all requests)            │                  │
    │               │  → return error immediately          │                  │
    │               │  After: timeout period expires       │                  │
    │               └──────────────────────────────────────┘                  │
    │                              │                                           │
    │                      timeout expires                                     │
    │                              │                                           │
    │                              ▼                                           │
    │               ┌──────────────────────────────────────┐                  │
    │               │                                      │                  │
    │               │          HALF-OPEN                   │──────────────────┘
    │               │  (let one trial request through)     │    success
    │               │  Success → CLOSE                     │
    └───────────────│  Failure → OPEN                      │
        failure     └──────────────────────────────────────┘
```

---

## 72.3 Implementation

```python
import time
from enum import Enum
from threading import Lock

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,      # open after 5 consecutive failures
        success_threshold: int = 2,       # close after 2 consecutive successes in half-open
        timeout: float = 30.0,            # seconds to stay open before trying half-open
        expected_exception: type = Exception
    ):
        self.failure_threshold = failure_threshold
        self.success_threshold = success_threshold
        self.timeout = timeout
        self.expected_exception = expected_exception

        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self._lock = Lock()

    def call(self, func, *args, **kwargs):
        with self._lock:
            if self.state == CircuitState.OPEN:
                if time.time() - self.last_failure_time >= self.timeout:
                    self.state = CircuitState.HALF_OPEN
                    self.success_count = 0
                else:
                    raise CircuitOpenError("Circuit is OPEN — fast failing")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except self.expected_exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        with self._lock:
            if self.state == CircuitState.HALF_OPEN:
                self.success_count += 1
                if self.success_count >= self.success_threshold:
                    self.state = CircuitState.CLOSED
                    self.failure_count = 0
            elif self.state == CircuitState.CLOSED:
                self.failure_count = 0  # reset on success

    def _on_failure(self):
        with self._lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.OPEN
            elif self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN

class CircuitOpenError(Exception):
    pass

# Usage:
db_circuit = CircuitBreaker(failure_threshold=5, timeout=30)

def query_database(sql: str):
    return db_circuit.call(actual_db_query, sql)

# If DB is down for > 5 calls:
# CircuitOpenError raised immediately (no 30-second timeout wait)
# After 30 seconds: tries one real query
# If it succeeds: circuit closes, normal operation resumes
```

---

# CHAPTER 73 — Leader Election

---

## 73.1 WHY Leader Election

```
Distributed system with multiple identical nodes:
  Problem: all nodes do the same thing → duplicate work, conflicts
  Solution: elect one leader, others are followers

Use cases:
  Job scheduler: only ONE node should trigger the hourly report
  Primary selection: which DB node accepts writes
  Shard management: which node coordinates shard assignment
  Rate limit enforcement: which node is the "master counter"
```

---

## 73.2 Leader Election with Redis

```python
import redis
import uuid
import time
import threading

r = redis.Redis()
LEADER_KEY = "app:leader"
MY_ID = str(uuid.uuid4())

def try_become_leader(ttl: int = 15) -> bool:
    """Try to acquire leadership. Returns True if successful."""
    return bool(r.set(LEADER_KEY, MY_ID, nx=True, ex=ttl))

def is_leader() -> bool:
    """Check if this node is still the leader."""
    return r.get(LEADER_KEY) == MY_ID.encode()

def renew_leadership(ttl: int = 15) -> bool:
    """Extend leadership TTL (heartbeat)."""
    lua = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("expire", KEYS[1], ARGV[2])
    else
        return 0
    end
    """
    return bool(r.eval(lua, 1, LEADER_KEY, MY_ID, ttl))

def leader_heartbeat(interval: int = 5):
    """Background thread: renew leadership every interval seconds."""
    while True:
        time.sleep(interval)
        if not renew_leadership():
            print(f"Node {MY_ID}: lost leadership!")
            break

def run_leader_tasks():
    """Tasks that only the leader should perform."""
    while is_leader():
        print(f"Node {MY_ID}: running leader tasks...")
        generate_report()
        time.sleep(60)

# Main election loop:
def main():
    while True:
        if try_become_leader():
            print(f"Node {MY_ID}: I am the leader!")
            heartbeat_thread = threading.Thread(target=leader_heartbeat, daemon=True)
            heartbeat_thread.start()
            run_leader_tasks()
        else:
            leader = r.get(LEADER_KEY)
            print(f"Node {MY_ID}: following leader {leader}")
            time.sleep(10)  # wait and check again
```

---

## 73.3 Leader Election with ZooKeeper/etcd

```python
# etcd-based leader election (production-grade):
import etcd3
import time

etcd = etcd3.client()

def etcd_election(election_name: str, candidate_id: str):
    """
    etcd uses lease-based election:
    1. Create a lease (TTL-based key ownership)
    2. Try to create election key with your lease
    3. If key exists: watch it (fire when current leader's lease expires)
    4. When key deleted: try again
    """
    lease = etcd.lease(ttl=15)  # 15-second TTL

    election_key = f"/election/{election_name}"

    # Try to put key with our lease (only creates if not exists):
    result = etcd.transaction(
        compare=[etcd.transactions.version(election_key) == 0],  # key doesn't exist
        success=[etcd.transactions.put(election_key, candidate_id, lease)],
        failure=[]
    )

    if result.succeeded:
        print(f"{candidate_id}: I am the leader!")
        # Keep lease alive:
        lease.refresh()
        # Do leader work...
    else:
        print(f"{candidate_id}: Following current leader")
        # Watch for leader change:
        events, cancel = etcd.watch(election_key)
        for event in events:
            if isinstance(event, etcd3.events.DeleteEvent):
                cancel()
                etcd_election(election_name, candidate_id)  # try again
                break
```

---

# CHAPTER 74 — pgvector and Vector Databases

---

## 74.1 WHY Vector Search

Modern AI applications need to find **semantically similar** content:
- "Find products similar to this one" (recommendation)
- "Find documents that answer this question" (RAG for LLMs)
- "Find faces similar to this image" (facial recognition)
- "Find code similar to this snippet" (code search)

Traditional databases find exact matches or range queries.
They cannot answer "which of these 1M documents is semantically closest to my query?"

The solution: **embeddings** (dense vector representations) + **vector similarity search**.

---

## 74.2 What Embeddings Are

```python
from openai import OpenAI

client = OpenAI()

# Convert text to embedding (384 or 1536-dimensional vector):
text = "How do I optimize PostgreSQL queries?"

response = client.embeddings.create(
    model="text-embedding-3-small",
    input=text
)

embedding = response.data[0].embedding
# embedding = [0.023, -0.156, 0.891, ..., 0.034]  # 1536 numbers

# Semantically similar texts have embeddings that are CLOSE to each other
# (measured by cosine similarity or L2 distance)

# "How to make PostgreSQL faster?" → embedding1
# "PostgreSQL query optimization tips" → embedding2
# "What's the weather in Mumbai?" → embedding3

# cosine_similarity(embedding1, embedding2) ≈ 0.95  (very similar!)
# cosine_similarity(embedding1, embedding3) ≈ 0.12  (very different!)
```

---

## 74.3 pgvector — Vector Search in PostgreSQL

```sql
-- Install extension:
CREATE EXTENSION vector;

-- Create table with vector column:
CREATE TABLE articles (
    id          BIGSERIAL PRIMARY KEY,
    title       VARCHAR(500) NOT NULL,
    content     TEXT,
    embedding   vector(1536),  -- OpenAI text-embedding-3-small dimension
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Store embeddings:
INSERT INTO articles (title, content, embedding)
VALUES (
    'PostgreSQL Indexing Guide',
    'B-Tree indexes are the most common...',
    '[0.023, -0.156, 0.891, ...]'::vector  -- 1536 numbers
);

-- Search by similarity (cosine similarity):
SELECT id, title,
    1 - (embedding <=> '[query_embedding]'::vector) AS cosine_similarity
FROM articles
ORDER BY embedding <=> '[query_embedding]'::vector  -- <=> = cosine distance
LIMIT 10;

-- Distance operators:
-- <=>  Cosine distance (use for text, normalized vectors)
-- <->  Euclidean distance (L2, use for unnormalized vectors)
-- <#>  Negative inner product (use for maximum inner product search)

-- Index for fast approximate search:
-- IVFFlat: divide vectors into lists, search nearest lists
CREATE INDEX ON articles USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);  -- 100 partition lists (sqrt of row count is good start)

-- HNSW (better for most cases — PG 16, pgvector 0.5+):
CREATE INDEX ON articles USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
-- m: number of connections per node (higher = better recall, more memory)
-- ef_construction: candidate list size during index build

-- Increase search quality (at cost of speed):
SET hnsw.ef_search = 200;  -- default 40; increase for better recall
```

---

## 74.4 RAG — Retrieval-Augmented Generation

RAG combines vector search with LLMs to answer questions using your own data:

```python
from openai import OpenAI
import psycopg2
import json

client = OpenAI()

def embed(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

def retrieve_relevant_docs(question: str, top_k: int = 5) -> list[dict]:
    """Find documents semantically similar to the question."""
    query_embedding = embed(question)

    conn = psycopg2.connect("...")
    cur = conn.cursor()

    cur.execute("""
        SELECT id, title, content,
               1 - (embedding <=> %s::vector) AS similarity
        FROM articles
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (query_embedding, query_embedding, top_k))

    results = [
        {"id": row[0], "title": row[1], "content": row[2], "similarity": row[3]}
        for row in cur.fetchall()
    ]
    conn.close()
    return results

def rag_answer(question: str) -> str:
    """Answer question using retrieved documents as context."""
    # Step 1: Retrieve relevant documents
    docs = retrieve_relevant_docs(question, top_k=5)

    # Step 2: Build context from retrieved docs
    context = "\n\n".join([
        f"Title: {doc['title']}\nContent: {doc['content'][:500]}"
        for doc in docs
    ])

    # Step 3: Ask LLM to answer using the context
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "Answer questions using only the provided context. If the answer is not in the context, say so."},
            {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {question}"}
        ]
    )

    return response.choices[0].message.content

# Usage:
answer = rag_answer("How do I fix slow PostgreSQL queries?")
print(answer)
# "Based on the provided context: First, run EXPLAIN ANALYZE to identify..."
```

---

## 74.5 Hybrid Search — Vector + Keyword

```sql
-- Combine vector similarity with full-text search for best results:
WITH vector_results AS (
    SELECT id,
           RANK() OVER (ORDER BY embedding <=> '[query_embedding]'::vector) AS vector_rank
    FROM articles
    ORDER BY embedding <=> '[query_embedding]'::vector
    LIMIT 20
),
fts_results AS (
    SELECT id,
           RANK() OVER (ORDER BY ts_rank(search_vector, to_tsquery('english', 'postgresql & index')) DESC) AS fts_rank
    FROM articles
    WHERE search_vector @@ to_tsquery('english', 'postgresql & index')
    LIMIT 20
),
combined AS (
    SELECT
        COALESCE(v.id, f.id) AS id,
        COALESCE(1.0 / (60 + v.vector_rank), 0) +
        COALESCE(1.0 / (60 + f.fts_rank), 0) AS rrf_score  -- Reciprocal Rank Fusion
    FROM vector_results v
    FULL OUTER JOIN fts_results f ON v.id = f.id
)
SELECT a.id, a.title, c.rrf_score
FROM combined c
JOIN articles a ON a.id = c.id
ORDER BY rrf_score DESC
LIMIT 10;

-- Reciprocal Rank Fusion (RRF): combines rankings from multiple retrieval methods
-- 1/(60+rank): smooth combination that doesn't overweight top results
-- k=60 is a standard constant that works well in practice
```

---

# CHAPTER 75 — PostGIS: Geospatial Data

---

## 75.1 WHY PostGIS

Location-based features are in every modern app:
"Show restaurants within 5km", "Plot delivery route", "Find nearest driver",
"Which orders shipped from this warehouse?"

PostGIS adds full GIS (Geographic Information System) capabilities to PostgreSQL.

---

## 75.2 PostGIS Basics

```sql
CREATE EXTENSION postgis;

-- Geometry types:
-- POINT: single location (restaurant, user)
-- LINESTRING: road, route, river
-- POLYGON: area, building footprint, delivery zone
-- MULTIPOLYGON: country (multiple islands), state

CREATE TABLE restaurants (
    id       BIGSERIAL PRIMARY KEY,
    name     VARCHAR(200),
    location GEOMETRY(POINT, 4326),   -- 4326 = WGS84 (GPS coordinates)
    city     VARCHAR(100),
    rating   NUMERIC(3,1)
);

-- Insert with coordinates (longitude, latitude):
INSERT INTO restaurants (name, location, city, rating) VALUES
    ('Café Coffee Day', ST_MakePoint(77.5946, 12.9716), 'Bengaluru', 4.2),
    ('MTR Restaurant',  ST_MakePoint(77.5719, 12.9612), 'Bengaluru', 4.7),
    ('Udupi Palace',    ST_MakePoint(72.8777, 19.0760), 'Mumbai',    4.1);

-- Create spatial index (essential for geographic queries):
CREATE INDEX ON restaurants USING GIST(location);
```

---

## 75.3 Spatial Queries

```sql
-- Distance queries (geography type = uses meters, not degrees):
-- Convert to geography for accurate distance in meters:
CREATE TABLE restaurants (
    id       BIGSERIAL PRIMARY KEY,
    name     VARCHAR(200),
    location GEOGRAPHY(POINT, 4326)  -- geography = accurate spherical distances
);

-- Find restaurants within 5km of Bengaluru city center:
SELECT name, city,
    ST_Distance(location, ST_MakePoint(77.5946, 12.9716)::geography) / 1000 AS dist_km
FROM restaurants
WHERE ST_DWithin(
    location,
    ST_MakePoint(77.5946, 12.9716)::geography,  -- center point
    5000  -- 5000 meters = 5km
)
ORDER BY location <-> ST_MakePoint(77.5946, 12.9716)::geography;  -- KNN order

-- Find nearest 10 restaurants (KNN — K Nearest Neighbors):
SELECT name,
    ST_Distance(location, 'POINT(77.5946 12.9716)'::geography) AS dist_meters
FROM restaurants
ORDER BY location <-> 'POINT(77.5946 12.9716)'::geography  -- <-> uses spatial index!
LIMIT 10;

-- Delivery zones (polygon queries):
CREATE TABLE delivery_zones (
    id      BIGSERIAL PRIMARY KEY,
    name    VARCHAR(100),
    zone    GEOGRAPHY(POLYGON, 4326)
);

INSERT INTO delivery_zones (name, zone) VALUES (
    'Koramangala Zone',
    ST_MakePolygon(ST_GeomFromText(
        'LINESTRING(77.61 12.93, 77.64 12.93, 77.64 12.96, 77.61 12.96, 77.61 12.93)'
    ))::geography
);

-- Which zone does a delivery address fall in?
SELECT z.name AS delivery_zone
FROM delivery_zones z
WHERE ST_Within(
    ST_MakePoint(77.625, 12.945)::geometry,
    z.zone::geometry
);

-- Route length (linestring):
SELECT ST_Length(
    ST_MakeLine(
        ARRAY[
            ST_MakePoint(77.5946, 12.9716),
            ST_MakePoint(77.5994, 12.9698),
            ST_MakePoint(77.6100, 12.9650)
        ]
    )::geography
) / 1000 AS route_km;

-- Area of a polygon (in square kilometers):
SELECT ST_Area(zone::geography) / 1000000 AS area_sq_km
FROM delivery_zones WHERE name = 'Koramangala Zone';
```

---

# CHAPTER 76 — TimescaleDB

---

## 76.1 WHY TimescaleDB

Regular PostgreSQL tables struggle with time-series data:
- IoT sensors sending 100k readings/second → table grows 1M rows/day
- Queries always filter by time range → need efficient time-based access
- Old data can be downsampled/deleted → needs automatic management

TimescaleDB is a PostgreSQL extension that adds:
- Hypertables (automatically partitioned by time)
- Continuous aggregates (auto-computed summaries)
- Automatic retention policies (delete old data)
- Compression (reduce storage 90%+)

---

## 76.2 Hypertables

```sql
CREATE EXTENSION timescaledb;

-- Create a regular table first:
CREATE TABLE sensor_readings (
    time        TIMESTAMPTZ NOT NULL,
    sensor_id   INT NOT NULL,
    temperature NUMERIC(6,2),
    humidity    NUMERIC(5,2),
    pressure    NUMERIC(8,2)
);

-- Convert to hypertable (partitioned by time automatically):
SELECT create_hypertable('sensor_readings', 'time',
    chunk_time_interval => INTERVAL '1 day');  -- each chunk = 1 day of data

-- TimescaleDB now automatically:
-- Creates a new "chunk" table for each day
-- Routes inserts to correct chunk
-- Queries only scan chunks within the time range

-- Insert works exactly like normal PostgreSQL:
INSERT INTO sensor_readings (time, sensor_id, temperature)
VALUES (NOW(), 1, 23.5);

-- Query is automatically optimized to only scan relevant chunks:
SELECT AVG(temperature)
FROM sensor_readings
WHERE time > NOW() - INTERVAL '7 days'
  AND sensor_id = 1;
-- Without hypertable: scan 365 days of data
-- With hypertable: scan 7 chunks (1 per day) — 52× less data
```

---

## 76.3 Continuous Aggregates

```sql
-- Pre-compute hourly averages automatically:
CREATE MATERIALIZED VIEW hourly_sensor_avg
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    sensor_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp,
    COUNT(*) AS reading_count
FROM sensor_readings
GROUP BY hour, sensor_id;

-- Set refresh policy (auto-update as new data arrives):
SELECT add_continuous_aggregate_policy('hourly_sensor_avg',
    start_offset => INTERVAL '3 hours',  -- refresh data from 3 hours ago
    end_offset   => INTERVAL '1 hour',   -- up to 1 hour ago
    schedule_interval => INTERVAL '1 hour');  -- run every hour

-- Query the pre-computed aggregate (fast!):
SELECT hour, avg_temp
FROM hourly_sensor_avg
WHERE sensor_id = 1
  AND hour > NOW() - INTERVAL '30 days'
ORDER BY hour;
-- Reads 30 pre-aggregated rows instead of millions of raw readings!
```

---

## 76.4 Compression and Retention

```sql
-- Compress chunks older than 7 days (90% space savings typical):
ALTER TABLE sensor_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id',  -- group by sensor for best compression
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('sensor_readings', INTERVAL '7 days');
-- Chunks older than 7 days: automatically compressed
-- Recent data: uncompressed (fast reads/writes)

-- Automatic retention (delete data older than 1 year):
SELECT add_retention_policy('sensor_readings', INTERVAL '1 year');

-- Check compression stats:
SELECT hypertable_name, compression_enabled,
    pg_size_pretty(before_compression_total_bytes) AS before,
    pg_size_pretty(after_compression_total_bytes) AS after,
    ROUND(100 - 100.0 * after_compression_total_bytes / before_compression_total_bytes, 1) AS pct_reduction
FROM chunk_compression_stats('sensor_readings');
-- Typical output: before=10GB, after=800MB, pct_reduction=92%
```

---

# End of Ch 69–76

---

# CHAPTER 77 — Range Types, Generated Columns, Custom Domains

---

## 77.1 Range Types

Range types represent a range of values with inclusive/exclusive bounds.

```sql
-- Built-in range types:
-- int4range, int8range, numrange, tsrange, tstzrange, daterange

-- Hotel booking system using daterange:
CREATE TABLE reservations (
    id          BIGSERIAL PRIMARY KEY,
    room_id     INT NOT NULL,
    guest_name  VARCHAR(200),
    stay        DATERANGE NOT NULL,  -- [check_in, check_out)
    EXCLUDE USING GIST (room_id WITH =, stay WITH &&)  -- prevent overlapping bookings!
);

-- Insert bookings:
INSERT INTO reservations (room_id, guest_name, stay) VALUES
    (101, 'Rahul Kumar', '[2024-01-10, 2024-01-15)'),  -- Jan 10-14
    (101, 'Priya Sharma', '[2024-01-15, 2024-01-20)'); -- Jan 15-19 (no overlap)

-- Try to double-book:
INSERT INTO reservations (room_id, guest_name, stay)
VALUES (101, 'Vikram', '[2024-01-12, 2024-01-18)');
-- ERROR: conflicting key value violates exclusion constraint
-- PostgreSQL prevented the double-booking automatically!

-- Range operators:
-- @> : contains (does range contain value or other range?)
-- <@ : is contained by
-- && : overlaps
-- << : strictly left of
-- >> : strictly right of

-- Find all rooms booked during a specific period:
SELECT DISTINCT room_id
FROM reservations
WHERE stay && '[2024-01-14, 2024-01-16)'::daterange;

-- Find available rooms for a date range:
SELECT r.id AS room_id
FROM rooms r
WHERE NOT EXISTS (
    SELECT 1 FROM reservations res
    WHERE res.room_id = r.id
      AND res.stay && '[2024-01-20, 2024-01-25)'::daterange
);

-- Range functions:
SELECT
    lower('[2024-01-10, 2024-01-15)'::daterange) AS check_in,   -- 2024-01-10
    upper('[2024-01-10, 2024-01-15)'::daterange) AS check_out,  -- 2024-01-15
    upper('[2024-01-10, 2024-01-15)'::daterange)
    - lower('[2024-01-10, 2024-01-15)'::daterange) AS nights;   -- 5
```

---

## 77.2 Generated Columns

Computed columns whose value is always derived from other columns:

```sql
-- STORED: computed and stored on disk (takes space, but fast to query)
-- VIRTUAL: computed on the fly (not stored) — PostgreSQL only supports STORED

CREATE TABLE orders (
    id           BIGSERIAL PRIMARY KEY,
    quantity     INT NOT NULL,
    unit_price   NUMERIC(10,2) NOT NULL,
    total_price  NUMERIC(12,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
    tax_amount   NUMERIC(10,2) GENERATED ALWAYS AS (quantity * unit_price * 0.18) STORED,
    grand_total  NUMERIC(12,2) GENERATED ALWAYS AS (quantity * unit_price * 1.18) STORED
);

INSERT INTO orders (quantity, unit_price) VALUES (5, 99.99);
-- total_price, tax_amount, grand_total computed automatically!

SELECT * FROM orders;
-- id=1, quantity=5, unit_price=99.99, total_price=499.95, tax_amount=89.99, grand_total=589.94

-- UPDATE quantity → generated columns automatically recalculated:
UPDATE orders SET quantity = 10 WHERE id = 1;

-- Limitations:
-- Cannot reference other tables
-- Cannot use non-deterministic functions (NOW(), random())
-- Cannot be directly set (GENERATED ALWAYS → error if you try to insert)

-- Use case: full_name from first + last:
CREATE TABLE persons (
    first_name  VARCHAR(100) NOT NULL,
    last_name   VARCHAR(100) NOT NULL,
    full_name   VARCHAR(201) GENERATED ALWAYS AS (first_name || ' ' || last_name) STORED
);

CREATE INDEX ON persons(full_name);  -- index generated column for fast search
SELECT * FROM persons WHERE full_name = 'Rahul Kumar';
```

---

## 77.3 Custom Domains — Type Safety

Domains create named types with constraints, reusable across tables:

```sql
-- Define domain types with constraints:
CREATE DOMAIN email_address AS VARCHAR(255)
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
    NOT NULL;

CREATE DOMAIN phone_number AS VARCHAR(20)
    CHECK (VALUE ~ '^\+?[1-9]\d{9,14}$');  -- international format

CREATE DOMAIN positive_money AS NUMERIC(12,2)
    CHECK (VALUE > 0);

CREATE DOMAIN percentage AS NUMERIC(5,2)
    CHECK (VALUE >= 0 AND VALUE <= 100);

-- Use in table definitions:
CREATE TABLE customers (
    id      BIGSERIAL PRIMARY KEY,
    email   email_address,        -- auto-validated on every INSERT/UPDATE
    phone   phone_number,         -- auto-validated
    discount percentage DEFAULT 0
);

-- Insert with invalid email → error:
INSERT INTO customers (email) VALUES ('not-an-email');
-- ERROR: value for domain email_address violates check constraint

-- Valid insert:
INSERT INTO customers (email, phone, discount) VALUES
    ('rahul@example.com', '+919876543210', 10);  -- all pass validation

-- Reuse across multiple tables — consistent validation everywhere:
CREATE TABLE suppliers (
    id      BIGSERIAL PRIMARY KEY,
    email   email_address,   -- same domain, same validation
    contact phone_number
);
```

---

# CHAPTER 78 — Foreign Data Wrappers (FDW)

---

## 78.1 WHY FDW

Query external data sources from PostgreSQL as if they are local tables.

```
Without FDW:
  1. Application queries PostgreSQL
  2. Application queries MySQL separately
  3. Application joins data in memory
  → Complex application code, manual joins, no query optimization

With FDW:
  1. Mount MySQL as a foreign table in PostgreSQL
  2. Write a JOIN between local and remote table
  3. PostgreSQL query planner optimizes automatically
  → Transparent, optimized, single query
```

---

## 78.2 postgres_fdw — Connect to Another PostgreSQL

```sql
-- On the LOCAL PostgreSQL server:

-- 1. Install the FDW extension:
CREATE EXTENSION postgres_fdw;

-- 2. Create a foreign server (connection to remote PostgreSQL):
CREATE SERVER remote_inventory_db
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'inventory-db.example.com', port '5432', dbname 'inventory');

-- 3. Create user mapping (local user → remote user credentials):
CREATE USER MAPPING FOR local_app_user
    SERVER remote_inventory_db
    OPTIONS (user 'remote_user', password 'remote_password');

-- 4. Import foreign tables (or create manually):
IMPORT FOREIGN SCHEMA public
    FROM SERVER remote_inventory_db
    INTO local_shadow_schema;

-- 5. Query foreign tables like local tables:
SELECT
    o.id AS order_id,
    o.customer_id,
    p.name AS product_name,      -- from LOCAL products table
    i.quantity AS remote_stock   -- from REMOTE inventory system!
FROM orders o
JOIN products p ON p.id = o.product_id
JOIN local_shadow_schema.inventory i ON i.product_id = p.id  -- cross-server join!
WHERE o.status = 'pending';

-- PostgreSQL pushes WHERE clauses to the remote server automatically:
-- Remote server handles: SELECT quantity FROM inventory WHERE product_id IN (...)
-- Local server: joins with local data
```

---

## 78.3 Other FDW Types

```sql
-- mysql_fdw: connect to MySQL
CREATE EXTENSION mysql_fdw;

-- file_fdw: query CSV files as tables
CREATE EXTENSION file_fdw;
CREATE SERVER csv_files FOREIGN DATA WRAPPER file_fdw;
CREATE FOREIGN TABLE import_data (
    id INT, name VARCHAR(100), amount NUMERIC
) SERVER csv_files
OPTIONS (filename '/tmp/import.csv', format 'csv', header 'true');
SELECT * FROM import_data WHERE amount > 1000;

-- http_fdw (third-party): query REST APIs as tables
-- redis_fdw (third-party): query Redis from PostgreSQL
-- elasticsearch_fdw: query Elasticsearch from PostgreSQL
```

---

# CHAPTER 79 — Database Migrations and Testing

---

## 79.1 Database Migration Strategy

```
Migration principles:
1. Every schema change is a migration file (versioned, ordered)
2. Migrations are run in order, exactly once
3. Each migration has an UP (apply) and DOWN (rollback)
4. Migrations are committed to version control alongside code
5. CI/CD runs migrations automatically before deploying new code

Tools:
  Python + SQLAlchemy: Alembic
  Java: Flyway, Liquibase
  Node.js: Knex, Prisma Migrate
  Ruby: Rails Migrations
  Go: golang-migrate
```

### Alembic (Python) Patterns

```python
# alembic/versions/20240115_add_priority_to_orders.py

from alembic import op
import sqlalchemy as sa

revision = '20240115'
down_revision = '20240110'

def upgrade():
    # Step 1: Add nullable column (instant, no lock)
    op.add_column('orders', sa.Column('priority', sa.Integer(), nullable=True))

    # Step 2: Backfill in batches (no long lock, breaks work into small transactions)
    op.execute("""
        DO $$
        DECLARE
            batch_size INT := 10000;
            offset_val INT := 0;
            rows_updated INT;
        BEGIN
            LOOP
                UPDATE orders SET priority = 5
                WHERE id IN (
                    SELECT id FROM orders
                    WHERE priority IS NULL
                    ORDER BY id
                    LIMIT batch_size
                    OFFSET offset_val
                );
                GET DIAGNOSTICS rows_updated = ROW_COUNT;
                EXIT WHEN rows_updated = 0;
                offset_val := offset_val + batch_size;
                PERFORM pg_sleep(0.01);  -- brief pause to reduce load
            END LOOP;
        END $$;
    """)

    # Step 3: Add NOT NULL constraint (PostgreSQL 12+ validates without full lock)
    op.execute("ALTER TABLE orders ADD CONSTRAINT orders_priority_not_null CHECK (priority IS NOT NULL) NOT VALID")
    op.execute("ALTER TABLE orders VALIDATE CONSTRAINT orders_priority_not_null")
    op.alter_column('orders', 'priority', nullable=False)
    op.execute("ALTER TABLE orders DROP CONSTRAINT orders_priority_not_null")

def downgrade():
    op.drop_column('orders', 'priority')
```

---

## 79.2 Database Testing

```python
# conftest.py - pytest fixtures for database testing

import pytest
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_AUTOCOMMIT

TEST_DB_NAME = "test_db"
MAIN_DB_URL = "postgresql://postgres:password@localhost/postgres"

@pytest.fixture(scope="session")
def test_db():
    """Create a fresh test database for the test session."""
    conn = psycopg2.connect(MAIN_DB_URL)
    conn.set_isolation_level(ISOLATION_LEVEL_AUTOCOMMIT)
    cur = conn.cursor()

    cur.execute(f"DROP DATABASE IF EXISTS {TEST_DB_NAME}")
    cur.execute(f"CREATE DATABASE {TEST_DB_NAME}")
    conn.close()

    # Run migrations on test DB:
    os.system(f"DATABASE_URL=postgresql://postgres:password@localhost/{TEST_DB_NAME} alembic upgrade head")

    yield f"postgresql://postgres:password@localhost/{TEST_DB_NAME}"

    # Cleanup:
    conn = psycopg2.connect(MAIN_DB_URL)
    conn.set_isolation_level(ISOLATION_LEVEL_AUTOCOMMIT)
    conn.cursor().execute(f"DROP DATABASE IF EXISTS {TEST_DB_NAME}")
    conn.close()

@pytest.fixture
def db_conn(test_db):
    """Give each test a transaction that rolls back after the test."""
    conn = psycopg2.connect(test_db)
    conn.autocommit = False

    yield conn

    conn.rollback()  # undo all changes after each test
    conn.close()

# Test example:
def test_create_order(db_conn):
    cur = db_conn.cursor()

    # Arrange: insert test customer
    cur.execute("INSERT INTO customers (email, full_name) VALUES (%s, %s) RETURNING id",
                ('test@example.com', 'Test User'))
    customer_id = cur.fetchone()[0]

    # Act: create order
    cur.execute("INSERT INTO orders (customer_id, total_amount) VALUES (%s, %s) RETURNING id",
                (customer_id, 299.99))
    order_id = cur.fetchone()[0]

    # Assert:
    cur.execute("SELECT total_amount FROM orders WHERE id = %s", (order_id,))
    result = cur.fetchone()
    assert float(result[0]) == 299.99

# db_conn rolls back after each test → no test data pollution
```

---

## 79.3 pgbench — Load Testing

```bash
# Initialize pgbench test data (default TPC-B schema):
pgbench -i -s 10 mydb  # scale factor 10 = 10×100k accounts = 1M rows

# Run benchmark: 10 clients, 2 threads, 60 seconds:
pgbench -c 10 -j 2 -T 60 mydb

# Output:
# transaction type: <builtin: TPC-B (sort of)>
# scaling factor: 10
# number of clients: 10
# number of threads: 2
# duration: 60 s
# number of transactions actually processed: 45230
# tps = 753.8 (including connections establishing)
# latency average = 13.27 ms
# latency stddev = 8.43 ms

# Custom SQL script:
cat > my_test.sql << 'EOF'
\set customer_id random(1, 100000)
SELECT * FROM orders WHERE customer_id = :customer_id ORDER BY created_at DESC LIMIT 10;
EOF

pgbench -c 50 -j 4 -T 120 -f my_test.sql mydb

# Progressive load test: find breaking point
for clients in 10 25 50 100 200; do
    echo "Testing $clients clients:"
    pgbench -c $clients -j 4 -T 30 mydb 2>&1 | grep tps
done
```

---

# CHAPTER 80 — Batch Processing

---

## 80.1 MapReduce Deep Dive

We covered MapReduce briefly. Here is the full picture:

```
MapReduce is a programming model for processing large datasets
across a cluster of commodity machines.

Input: files on distributed filesystem (HDFS, GCS, S3)
Output: files on distributed filesystem

Three phases:
  1. MAP:    split input into (key, value) pairs
  2. SHUFFLE: group all values by key (framework handles this)
  3. REDUCE: aggregate values for each key
```

### Word Count Example (Classic)

```python
# Input: 100GB of text files across 1000 machines
# Goal: count occurrences of each word

# MAP function (runs on each machine, for each input record):
def map(filename, text_content):
    for word in text_content.split():
        yield (word.lower(), 1)

# After map, each machine produces: [("the",1), ("quick",1), ("the",1), ...]
# SHUFFLE: framework collects all (word, [list of 1s]) → sends to reducer

# REDUCE function (runs once per unique key):
def reduce(word, count_list):
    yield (word, sum(count_list))

# Output: [("a", 5234), ("and", 8921), ("the", 12345), ...]
```

### Why MapReduce is Still Relevant

```
Apache Spark replaced Hadoop MapReduce for most use cases (100× faster)
But MapReduce ideas live on:

1. Spark: same map/reduce concepts, but keeps data in memory between stages
2. BigQuery: distributed SQL built on MapReduce-like execution
3. Flink, Kafka Streams: stream processing with similar concepts
4. PostgreSQL parallel query: same idea within one machine

Key MapReduce insights that survive:
  → Fault tolerance: if one machine fails, re-run just that mapper/reducer
  → Data locality: send computation to data (not data to computation)
  → Sorting is free (shuffle sorts all output by key — useful for joins)
  → Scale-out: add machines to process more data linearly
```

---

## 80.2 Apache Spark — Modern Batch Processing

```python
# Spark: keeps intermediate data in RAM (not disk like MapReduce)
# Result: 10-100× faster for multi-stage computations

from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder.appName("OrderAnalysis").getOrCreate()

# Load data from S3/BigQuery/PostgreSQL:
orders_df = spark.read.format("jdbc").options(
    url="jdbc:postgresql://host:5432/mydb",
    dbtable="orders",
    user="user", password="password"
).load()

products_df = spark.read.format("parquet").load("s3://bucket/products/")

# Transformations (lazy — not executed until action):
result = (
    orders_df
    .filter(F.col("created_at") >= "2024-01-01")
    .join(products_df, orders_df.product_id == products_df.id)
    .groupBy("products.category", F.month("created_at"))
    .agg(
        F.sum("total_amount").alias("revenue"),
        F.count("*").alias("order_count"),
        F.avg("total_amount").alias("avg_order")
    )
    .orderBy("category", "month")
)

# Action (triggers actual computation):
result.write.format("parquet").mode("overwrite").save("s3://bucket/revenue_by_category/")
# OR:
result.show(20)

# Fault tolerance: Spark tracks lineage (how data was computed)
# If a partition is lost: recompute from source using lineage graph
# No need to checkpoint intermediate results to disk (unlike MapReduce)
```

---

# CHAPTER 81 — Future of Data Systems

---

## 81.1 Lambda Architecture

Lambda architecture handles both batch and stream processing:

```
SOURCE (raw events, e.g., Kafka)
    │
    ├──► BATCH LAYER (Spark, MapReduce)
    │      → Processes ALL historical data
    │      → Computes accurate results
    │      → Runs once per hour/day
    │      → Writes to: Batch Views (BigQuery, Parquet)
    │
    └──► SPEED LAYER (Kafka Streams, Flink)
           → Processes only RECENT events
           → Computes approximate/real-time results
           → Updates continuously
           → Writes to: Realtime Views (Redis, HBase)

SERVING LAYER merges batch + realtime views:
  Query: "Revenue today by category"
  → Batch view: accurate revenue up to midnight (yesterday)
  → Realtime view: approximate revenue since midnight
  → Merge: yesterday_batch + today_realtime = current answer

Problem with Lambda:
  Same logic written TWICE (once for batch, once for stream)
  Complex to maintain consistency between both implementations
```

---

## 81.2 Kappa Architecture

Kappa simplifies Lambda by using only streaming:

```
SOURCE (Kafka — durable, replayable log)
    │
    └──► STREAM PROCESSING ONLY (Flink, Kafka Streams)
           → Processes all events (historical replay + real-time)
           → Single code path (no batch vs stream split)
           → Results written to: Serving Database (Redis, PostgreSQL)

Reprocessing (when you need to fix logic or backfill):
  → Change the stream processing code
  → Reset consumer offset to beginning of Kafka topic
  → Reprocess entire history through new code
  → New results replace old results in serving DB

Requirements for Kappa:
  → Kafka must retain data long enough for full replay (months/years)
  → Stream processor must handle high throughput (historical replay)
  → Reprocessing is slower than batch (stream overhead)

When to use:
  → Kappa: simpler, fewer moving parts, sufficient for most cases
  → Lambda: when batch accuracy is critical AND you can maintain dual code
```

---

## 81.3 Data Mesh — Organizational Approach

```
Problem with centralized data team:
  All data engineering owned by one team → bottleneck
  Domain teams wait weeks for data pipelines
  Data quality is the "data team's problem" (not the owning team)

Data Mesh principles:
  1. Domain ownership: orders team owns orders data pipeline
     Marketing team owns marketing data
     Each team treats data as a product they ship

  2. Data as a product: each domain exposes data with:
     → Clear schema and documentation
     → SLA on freshness and quality
     → Self-serve access for consumers

  3. Self-serve infrastructure: platform team provides:
     → Tools for data teams to build pipelines without central help
     → Data catalog (Datahub, Apache Atlas)
     → Query engines (Trino, Spark)

  4. Federated governance: global policies (GDPR, security) enforced
     via automated tooling, not manual approval

Technology stack for data mesh:
  → Each domain: PostgreSQL + dbt (SQL transformations) + Kafka (events)
  → Platform: Airflow (orchestration), Trino (federated query), Datahub (catalog)
  → Cloud: S3/GCS for storage (Parquet/Delta Lake format)
```

---

# CHAPTER 82 — Complete Reference: Everything In One Place

---

## 82.1 PostgreSQL Performance Quick Reference

```sql
-- ═══════════ DIAGNOSE SLOW QUERIES ═══════════
-- Top slowest by average time:
SELECT LEFT(query,80), calls, ROUND(mean_exec_time::numeric,2) AS avg_ms
FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;

-- Run EXPLAIN:
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) <your_query>;

-- Warning signs in EXPLAIN:
-- Seq Scan + large table → add index
-- Rows Removed by Filter → HIGH → add index on filter column
-- Sort Method: external merge → increase work_mem
-- Hash Batches > 1 → increase work_mem
-- Estimated rows ≠ actual rows → run ANALYZE

-- ═══════════ INDEXING ═══════════
CREATE INDEX CONCURRENTLY ix ON tbl(col);                  -- no lock
CREATE INDEX ON tbl(col1, col2) WHERE deleted_at IS NULL;  -- composite + partial
CREATE INDEX ON tbl(LOWER(email));                         -- expression
CREATE INDEX ON tbl USING GIN(jsonb_col);                  -- JSONB / arrays
CREATE INDEX ON tbl USING BRIN(created_at);                -- time-series
CREATE INDEX ON tbl(sku) INCLUDE (name, price);            -- covering index

-- ═══════════ JSONB ═══════════
payload->>'field'                  -- get as text
payload->'nested'->'field'         -- nested navigation
payload @> '{"key": "value"}'      -- containment
payload ? 'key'                    -- key exists
jsonb_path_query(payload, '$.items[*].sku')  -- path query
CREATE INDEX ON tbl USING GIN(payload);      -- index for @>, ?

-- ═══════════ WINDOW FUNCTIONS ═══════════
ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC)
RANK() OVER (ORDER BY score DESC)
SUM(revenue) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)  -- 7-day rolling
LAG(price, 1) OVER (ORDER BY date)  -- previous row value
LEAD(price, 1) OVER (ORDER BY date) -- next row value

-- ═══════════ TRANSACTIONS ═══════════
BEGIN;
SAVEPOINT sp1;
ROLLBACK TO SAVEPOINT sp1;
COMMIT;
SET lock_timeout = '5s';
SET statement_timeout = '30s';

-- ═══════════ LOCKING ═══════════
SELECT * FROM tbl WHERE id=1 FOR UPDATE SKIP LOCKED;  -- job queue
SELECT pg_advisory_xact_lock(key);                     -- advisory lock

-- ═══════════ MONITORING ═══════════
SELECT pid, NOW()-query_start AS dur, LEFT(query,80) FROM pg_stat_activity
WHERE state='active' ORDER BY 2 DESC;

SELECT pid, pg_blocking_pids(pid) AS blocked_by, query
FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;

SELECT pg_cancel_backend(pid);    -- graceful kill
SELECT pg_terminate_backend(pid); -- force kill

-- ═══════════ MAINTENANCE ═══════════
VACUUM ANALYZE tbl;
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age DESC;
SELECT relname, n_dead_tup FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;
REINDEX INDEX CONCURRENTLY ix_name;
```

---

## 82.2 System Design Cheat Sheet

```
PICK THE RIGHT DATABASE:
  Relational + ACID         → PostgreSQL (default choice)
  Cache / Session           → Redis
  Write-heavy / Scale-out   → Cassandra
  Full-text search          → Elasticsearch (or PostgreSQL FTS)
  Analytics / Warehouse     → BigQuery / Redshift / Snowflake
  Time-series               → TimescaleDB
  Graph                     → Neo4j (or recursive CTEs)
  Vector similarity         → pgvector (or Pinecone/Weaviate)
  Event streaming           → Kafka

SCALING DECISION:
  Read bottleneck           → Read replicas + Redis cache
  Write bottleneck          → Partition → Shard → Cassandra/DynamoDB
  Latency (far users)       → Multi-region + GeoDNS + CDN
  Complex analytics         → Separate data warehouse + ETL

CONSISTENCY:
  Financial / inventory     → PostgreSQL with SERIALIZABLE isolation
  Social feeds / likes      → Eventual consistency OK
  Distributed transactions  → SAGA pattern (not 2PC)
  Publish event atomically  → Outbox pattern

RATE LIMITING:
  Per-user limit            → Token bucket in Redis (SETNX + EXPIRE)
  Distributed               → Centralized Redis with Lua atomic script

DISTRIBUTED LOCKING:
  Simple                    → Redis SETNX
  Fault-tolerant            → Redlock (5 Redis nodes)
  Critical / financial      → etcd or ZooKeeper with fencing tokens

PROBABILISTIC:
  "Have I seen this?"       → Bloom filter (0% false negative)
  "How many distinct?"      → HyperLogLog (±0.81% error)
  "How often seen this?"    → Count-Min Sketch (over-estimates)

STREAM PROCESSING:
  Real-time events          → Kafka (durable) + Flink/Kafka Streams
  Exactly-once              → Kafka transactions + idempotent consumer
  Event enrichment          → Stream-table join (local state store)

NUMBERS:
  Single PostgreSQL         → 50k reads/s, 20k writes/s
  Redis                     → 1M ops/s
  Speed of light Mumbai→SG  → ~30ms RTT
  Speed of light Mumbai→US  → ~200ms RTT
  Cache hit                 → <1ms
  DB indexed read           → <10ms
  DB write (with WAL fsync) → 5-20ms
```

---

## 82.3 Complete Topic Map

```
FOUNDATIONS
  ├── DBMS internals (pages, tuples, WAL, MVCC)
  ├── ACID (atomicity via WAL, consistency via constraints,
  │         isolation via MVCC snapshots, durability via fsync)
  ├── Data types and storage (TOAST, alignment, null bitmap)
  └── Normalization (1NF→BCNF, functional dependencies)

INDEXING
  ├── B-Tree (structure, splits, range scans, covering indexes)
  ├── GIN (arrays, JSONB, full-text search)
  ├── GiST (geometric, ranges, PostGIS)
  ├── BRIN (time-series, physically ordered data)
  ├── Hash (equality only)
  ├── Strategy (composite order, partial, expression, fillfactor)
  └── Pitfalls (implicit cast, LIKE, NULL, over-indexing)

QUERY PROCESSING
  ├── Planner (cost model, statistics, selectivity, hints)
  ├── EXPLAIN ANALYZE (reading plans, warning signs)
  ├── Scan nodes (Seq, Index, Bitmap, Index-Only, TID)
  ├── Join algorithms (Nested Loop, Hash Join, Merge Join)
  ├── Join types (INNER, LEFT, CROSS, LATERAL, self)
  └── Optimization (SARGable, parallel, work_mem, CTEs)

TRANSACTIONS
  ├── BEGIN/COMMIT/ROLLBACK/SAVEPOINT
  ├── XID and wraparound
  ├── Isolation levels (RC, RR, Serializable/SSI)
  ├── MVCC visibility rules
  ├── Locking (table, row, advisory, SKIP LOCKED)
  └── Deadlocks (detection, patterns, prevention)

POSTGRESQL FEATURES
  ├── Window functions (RANK, LAG, LEAD, running totals)
  ├── PL/pgSQL (functions, procedures, triggers)
  ├── Views (regular, updatable, materialized)
  ├── JSONB (operators, indexing, path queries)
  ├── Arrays (operators, GIN, unnest)
  ├── Range types (daterange, EXCLUDE constraint)
  ├── Generated columns
  ├── Custom domains
  ├── Table partitioning (range, list, hash)
  ├── Full-text search (tsvector, GIN, ranking)
  ├── Security (roles, RLS, row policies)
  └── Sequences (GENERATED ALWAYS, UUID)

DATA MODELS (DDIA)
  ├── Relational vs Document vs Graph
  ├── Query languages (SQL, Cypher, SPARQL, Datalog, MapReduce)
  ├── Storage engines (B-Tree vs LSM-Tree, SSTables)
  ├── Encoding (JSON, Protobuf, Avro, schema evolution)
  ├── OLTP vs OLAP
  ├── Column storage and compression
  └── Data warehousing (star schema, materialized views)

DISTRIBUTED SYSTEMS (DDIA)
  ├── Unreliable networks (timeouts, retries, idempotency)
  ├── Unreliable clocks (NTP, lamport clocks, vector clocks)
  ├── Process pauses (GC, VM, fencing tokens)
  ├── Replication (single-leader, multi-leader, leaderless, quorums)
  ├── Partitioning (range, hash, consistent hashing, secondary indexes)
  ├── CAP theorem and consistency models
  ├── Linearizability (definition, cost, when needed)
  ├── Raft algorithm (leader election, log replication)
  └── ZooKeeper / etcd (coordination service)

SCALING
  ├── Regional: Patroni HA, read replicas, Citus sharding
  ├── Global: active-passive, active-active, conflict resolution
  ├── Spanner (TrueTime), CockroachDB (HLC+Raft)
  ├── GeoDNS, CDN, anycast
  └── Data residency (GDPR, PDPB, HIPAA)

STREAM PROCESSING
  ├── Kafka (topics, partitions, offsets, consumer groups)
  ├── Exactly-once semantics
  ├── Event time vs processing time, watermarks
  ├── Windows (tumbling, hopping, sliding, session)
  ├── Stream-table joins
  ├── Lambda vs Kappa architecture
  └── Data Mesh

SYSTEM DESIGN PATTERNS
  ├── Caching (cache-aside, write-through, write-behind, TTL)
  ├── Rate limiting (token bucket, sliding window, distributed)
  ├── Distributed locking (Redis SETNX, Redlock, etcd)
  ├── Circuit breaker (closed, open, half-open states)
  ├── Leader election (Redis, ZooKeeper)
  ├── SAGA pattern (choreography, orchestration)
  ├── Outbox pattern (reliable event publishing)
  ├── CDC (Change Data Capture, Debezium)
  ├── Event sourcing + CQRS
  ├── Sharding (range, hash, consistent hashing)
  └── Hot spots and mitigation

PROBABILISTIC STRUCTURES
  ├── Bloom filter (membership, 0% false negative)
  ├── HyperLogLog (distinct count, ±0.81%)
  └── Count-Min Sketch (frequency estimation)

MODERN EXTENSIONS
  ├── pgvector (embeddings, KNN, HNSW index)
  ├── RAG architecture (embed → store → retrieve → LLM)
  ├── PostGIS (spatial queries, ST_DWithin, KNN)
  └── TimescaleDB (hypertables, continuous aggregates, compression)

OPERATIONS
  ├── VACUUM (dead tuples, visibility map, autovacuum tuning)
  ├── Statistics (pg_statistic, ANALYZE, extended statistics)
  ├── Monitoring (pg_stat_statements, pg_stat_activity, auto_explain)
  ├── Replication (streaming, logical, WAL archiving, PITR)
  ├── PgBouncer (transaction mode, pool sizing)
  ├── Concurrent DDL (safe migrations, ADD COLUMN patterns)
  ├── Database testing (transaction rollback per test)
  └── Load testing (pgbench, k6)
```

---

*PostgreSQL Mastery Book — Complete*
*82 Chapters | Parts I through XIV*
*From DBMS Fundamentals to AI-Ready Data Systems*
