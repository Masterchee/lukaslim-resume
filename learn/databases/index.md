# Databases: A Technician's Guide

A database is a program that stores data on disk and lets other programs ask for it quickly and safely. This guide covers what is on the disk, why queries get slow, and what happens between your application and the database server.

---

## Key Terms
Read these first. Come back to them whenever a word is unfamiliar.

| Term | Meaning |
|------|---------|
| **Table / Row / Column** | A set of records / one record / one field of a record |
| **Schema** | The structure of the data: tables, columns, types, constraints |
| **Primary key** | A column (or columns) that uniquely identifies each row |
| **Foreign key** | A column that points to another table's primary key |
| **Join** | A query that combines rows from tables using matching values |
| **Index** | A separate sorted structure that lets the engine find rows without scanning the whole table |
| **Query plan** | The steps the engine chooses to run a query (see it with `EXPLAIN`) |
| **Full table scan** | Reading every row to answer a query |
| **Transaction** | A group of operations that succeed or fail together |
| **ACID** | Atomicity, Consistency, Isolation, Durability: the guarantees of a transaction |
| **WAL** | Write-ahead log. Changes are logged before the data pages, which makes commits durable |
| **Lock** | A mechanism that stops conflicting changes to the same data at once |
| **Normalization** | Organizing tables to avoid duplicated data |
| **Document / Key-value store** | NoSQL models that store flexible records / values looked up by key |
| **Connection pool** | A set of reusable open database connections |
| **Replication** | Keeping copies of data on other servers for availability or read scale |
| **PITR** | Point-in-time recovery: restore to an exact moment using backups plus logs |

---

## 1. The 30-Second Mental Model

A database is a **filing room with a clerk**. The data sits in cabinets (disk). The clerk (the database engine) keeps a few drawers on the desk (memory cache) and an **index** card file that says where things are. Without the card file, the clerk walks through every cabinet to find one record.

```text
Application
    |  SQL query over a network connection
    v
+-----------------------------+
| Database engine             |
|  parser -> planner -> executor
|  memory cache (hot pages)   |
|  write-ahead log (WAL)      |
+--------------+--------------+
               |
          Disk: table files + index files
```

Two ideas matter most:
- **Data model:** how data is shaped (tables vs. documents).
- **Access path:** how the engine finds it (scan every row vs. use an index).

---

## 2. How It Works in Practice

### 2.1 Relational (SQL) vs. NoSQL

| | Relational (SQL) | NoSQL |
|---|---|---|
| Data shape | Tables with rows and columns, fixed schema | Documents, key-value pairs, wide-column or graph |
| Relationships | Joins across tables via keys | Usually stored together inside one item |
| Transactions | Strong ACID guarantees as standard | Varies; often tuned for scale over strict consistency |
| Scaling | Typically scale up first, then replicas | Designed to spread across many nodes |
| Examples | PostgreSQL, MySQL, SQL Server, Amazon RDS/Aurora | MongoDB, DynamoDB, Redis, Cassandra |
| Good fit | Orders, accounts, reporting, anything with relationships | Sessions, caches, catalogs, high-volume simple lookups |

**Choosing:** it is a trade-off about your workload, not a contest. If you need joins, constraints and transactions, start with SQL. If your access pattern is "fetch one item by key, at huge scale", a key-value or document store fits well. Many systems use both.

**ACID** (SQL transactions), in one line each:
- **Atomic:** all steps of a transaction happen, or none.
- **Consistent:** rules (constraints) are never broken.
- **Isolated:** concurrent transactions don't see each other's half-done work.
- **Durable:** once committed, data survives a crash (the write-ahead log makes this possible).

A small relational example:

```sql
CREATE TABLE customers (id INTEGER PRIMARY KEY, name TEXT, email TEXT);
CREATE TABLE orders (
  id INTEGER PRIMARY KEY,
  customer_id INTEGER REFERENCES customers(id),
  total REAL
);

SELECT c.name, SUM(o.total)
FROM customers c JOIN orders o ON o.customer_id = c.id
GROUP BY c.name;
```

The same data as a document:
```json
{ "id": 1, "name": "Ana", "orders": [ { "id": 10, "total": 25.5 } ] }
```

### 2.2 How data hits the disk
Databases read and write fixed-size **pages** (commonly a few KB). A page holds several rows. The engine keeps recently used pages in memory. On a write, it first appends the change to a **write-ahead log (WAL)**, which makes the commit durable, and updates the table pages later. After a crash, the engine replays the log.

### 2.3 Indexes and query speed
An **index** is a separate sorted structure (commonly a **B-tree**) that maps a column's values to row locations. A lookup walks a few levels of the tree instead of reading every row.

| Without index | With index |
|---------------|------------|
| Full table scan, cost grows with row count | Tree lookup, cost grows very slowly |
| Cheap writes | Each write must also update the index |

Indexes use disk and slow writes, so add them for columns you filter, join or sort on, not for everything.

See the plan the engine picked:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- PostgreSQL / MySQL: EXPLAIN ANALYZE or EXPLAIN shows the plan
-- Look for "Seq Scan" / "ALL" (full scan) versus "Index Scan" / "ref"
```

Common reasons a query is slow:
- No usable index on the filtered column.
- `SELECT *` pulling far more data than needed.
- A function on the column in the `WHERE` clause (for example `WHERE LOWER(email) = ...`), which can stop an index from being used.
- Missing or outdated statistics, so the planner picks a poor plan.
- Competing queries holding **locks**.

### 2.4 Connection flows
Your application and the database are separate processes, usually on separate machines. Each query crosses the network.

```text
App  -- TCP connect --> DB:5432 (PostgreSQL) / 3306 (MySQL)
     -- authenticate (TLS recommended)
     -- send query, wait, receive rows
```

- **Latency:** every round trip costs time. 100 small queries in a loop is much slower than one query that fetches all the rows (the "N+1 query" problem).
- **Connection pooling:** opening a connection is expensive and the database limits how many it allows (for example `max_connections`). A **pool** keeps a set of open connections that requests borrow and return. Tools include PgBouncer, RDS Proxy, or the pool built into your application framework.
- **Placement:** keep the application and the database in the same region (and ideally the same availability zone) to keep latency low.
- **Security:** never expose a database port to the whole internet. Allow only your application's security group, and use parameterized queries to stop SQL injection:

```python
# Safe: the value is sent separately from the SQL text
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

### 2.5 Backups
- **Full backup:** a copy of everything.
- **Incremental/differential:** only what changed since the last backup.
- **Point-in-time recovery (PITR):** restore to an exact moment by replaying the log on top of a backup. Managed services such as Amazon RDS offer automated backups with PITR.
- **A backup you have never restored is only a hope.** Test restores on a schedule.

---

## 3. What Happens When It Breaks

| Symptom | Likely cause | What to check |
|---------|--------------|---------------|
| Query is slow | Full scan, missing index | `EXPLAIN` the query, look for a sequential scan |
| App times out connecting | Wrong host/port, firewall or security group, DB down | Network checks from the app host (`nc -vz host 5432`) |
| `Too many connections` | App opening connections without pooling, or leaked connections | Pool size, `max_connections`, connections per client |
| `Authentication failed` | Wrong credentials, or the user isn't allowed from this host | Database user and host rules |
| Queries hang, others wait | Lock contention, a long-running transaction | Active sessions and lock views for your engine |
| Disk full, DB stops writing | Table, log or backup growth | `df -h`, largest tables, log retention |
| Fast in development, slow in production | Small test data hides missing indexes | Test with production-sized data |
| Duplicate or inconsistent data | Missing constraint or transaction | Add unique keys, wrap related writes in a transaction |
| Lost data after a mistake | No usable backup | Restore from backup or PITR |

Debug in this order: **can I connect? → is the server healthy (CPU, memory, disk)? → which query is slow? → how does the engine run it?**

---

## 4. Hands-On Lab

You only need Python, which includes SQLite. Nothing to install or configure.

**Goal:** see an index change a query from scanning every row to a direct lookup.

1. Create `lab.py`:
   ```python
   import sqlite3, time, random

   db = sqlite3.connect(":memory:")
   db.execute("CREATE TABLE orders (id INTEGER PRIMARY KEY, customer_id INTEGER, total REAL)")
   db.executemany(
       "INSERT INTO orders (customer_id, total) VALUES (?, ?)",
       ((random.randint(1, 50000), random.random() * 100) for _ in range(500000)),
   )
   db.commit()

   def timed(label):
       start = time.perf_counter()
       rows = db.execute("SELECT * FROM orders WHERE customer_id = 42").fetchall()
       print(f"{label}: {len(rows)} rows in {(time.perf_counter() - start) * 1000:.2f} ms")
       print("  plan:", db.execute(
           "EXPLAIN QUERY PLAN SELECT * FROM orders WHERE customer_id = 42").fetchall()[0][3])

   timed("Without index")
   db.execute("CREATE INDEX idx_orders_customer ON orders(customer_id)")
   timed("With index   ")
   ```
2. Run it: `python lab.py`
3. **Read the output.** The first plan says `SCAN orders` (every row). The second says `SEARCH orders USING INDEX`. Compare the times.
4. **Break it on purpose.** Add this line at the end and run again, then read the plan:
   ```python
   print(db.execute("EXPLAIN QUERY PLAN SELECT * FROM orders WHERE customer_id + 0 = 42").fetchall()[0][3])
   ```
   Wrapping the column in an expression stops the index from being used.
5. **Transactions:** try inserting a row, then call `db.rollback()` before `db.commit()`, and check whether the row is there.
6. **Check yourself:** why does adding an index make reads faster but writes slightly slower?

---

## Cheat Sheet

| Task | SQL / Command |
|------|---------------|
| Create a table | `CREATE TABLE t (id INTEGER PRIMARY KEY, name TEXT);` |
| Insert / update / delete | `INSERT INTO t ...`, `UPDATE t SET ... WHERE ...`, `DELETE FROM t WHERE ...` |
| Filter and sort | `SELECT * FROM t WHERE col = 1 ORDER BY col;` |
| Count and group | `SELECT col, COUNT(*) FROM t GROUP BY col;` |
| Join two tables | `SELECT ... FROM a JOIN b ON b.a_id = a.id;` |
| Add an index | `CREATE INDEX idx_t_col ON t(col);` |
| See the query plan | `EXPLAIN SELECT ...` (`EXPLAIN ANALYZE` also runs it, in PostgreSQL) |
| Transaction | `BEGIN; ... COMMIT;` or `ROLLBACK;` |
| Safe parameters (Python) | `cursor.execute("... WHERE email = %s", (email,))` |
| Test the DB port | `nc -vz <host> 5432` (PostgreSQL) or `3306` (MySQL) |

| Pick this... | ...when |
|--------------|---------|
| SQL | You need joins, constraints and strict transactions |
| Document / key-value | You fetch items by key at large scale or need a flexible shape |
| Cache (for example Redis) | You need very fast reads of data you can rebuild |

| Slow query? | Check |
|-------------|-------|
| `SCAN` / `Seq Scan` in the plan | Add an index on the filtered column |
| Function around the column in `WHERE` | Rewrite so the bare column is compared |
| Many small queries in a loop | Combine into one query (avoid N+1) |

---

## Further Reading
- [SQL vs NoSQL (MongoDB)](https://www.mongodb.com/resources/basics/databases/nosql-explained/nosql-vs-sql): the two models side by side with use cases.
- [SQL vs NoSQL performance considerations](https://cloudwithsw.hashnode.dev/sql-vs-nosql-performance-optimization-considerations): where each one is faster.
- [SQL vs NoSQL and scaling](https://abstractalgorithms.hashnode.dev/system-design-databases-sql-vs-nosql-and-scaling): replication, sharding and the trade-offs behind the choice.
- [The art of choosing the right database](https://dev.to/dhanush___b/how-to-choose-the-right-db-for-your-application--c7h): how storage engines affect performance.
- [How SQL databases guarantee ACID](https://abstractalgorithms.hashnode.dev/acid-properties-in-sql-databases): what each ACID letter means and how databases implement it.
- [Database Glossary](https://adamdjellouli.com/articles/databases_notes/01_introduction_to_databases/05_glossary): clear definitions for keys, joins, indexes and transactions.

---

## Your Notes
**Have you written your own notes on this lesson?**

Taking notes in your own words is one of the best ways to learn. Before you move on, close this page and try to write down:

- The mental model, explained as if to a friend.
- The three commands or ideas you expect to use most.
- One thing that confused you, and what you will do to figure it out.

Notes you wrote yourself will stick far better than notes you copied.
