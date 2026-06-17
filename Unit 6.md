# Indexing and Query Processing

## What indexing actually is

Indexing is a technique for speeding up how quickly data can be retrieved. It works by maintaining a separate structure that holds key values alongside pointers to where the actual records live.

Without an index, the database has no choice but to do a linear search — checking every single record one after another. With an index in place, it can jump straight to the data it's after instead.

Indexing earns its place for a few clear reasons:

- It cuts down search time significantly.
- It reduces how many disk I/O operations are needed.
- It improves the performance of SELECT queries overall.

A simple way to picture it: imagine searching for a student named "Sonam" in a table. Without an index, the database has to check every row one by one. With an index, it looks up "Sonam" directly and jumps straight to the right row.

## Types of index structures

### Single-level index

This is just a straightforward list of keys paired with pointers to records — something like 10 pointing to Block1, 20 pointing to Block2, 30 pointing to Block3. The main downside is that it doesn't scale well once a database gets really large.

### Multi-level index

This builds an index on top of another index. It shrinks the search space dramatically, since you first use the top-level index to narrow down to the right section, and only then use the lower-level index to pin down the exact record.

### Tree-based indexes (B-Tree and B+ Tree)

Modern databases lean heavily on balanced tree structures for indexing.

**B-Tree properties worth knowing:**

- It's a self-balancing tree, with nodes sorted in an inorder fashion.
- A single node can have more than two children.
- Its height works out to roughly log M (where M is the order of the tree and N the number of nodes).
- Every leaf node sits at exactly the same level.
- There are no empty subtrees sitting above the leaf nodes.

**B+ Tree — the variant that matters most for databases:**

- Internal nodes hold only keys, with no data pointers attached.
- Leaf nodes are where the actual data pointers live.
- All leaves sit at the same level, keeping the tree balanced.
- Leaf nodes are linked together like a linked list, which makes sequential access possible.

So why does the B+ Tree end up being the preferred choice for indexing? Because every key value ultimately shows up at the leaf level (since data pointers only exist there), the linked leaves make range queries and ordered access genuinely efficient, and searches become faster and more predictable since every key is guaranteed to be reachable at the leaf level.

### B-Tree vs. B+ Tree, side by side

| Feature | B-Tree | B+ Tree |
|---------|--------|---------|
| Pointers | Both internal and leaf nodes carry data pointers | Only leaf nodes carry data pointers |
| Search speed | Can take longer, since not every key sits at the leaf level | Faster, since every key sits at the leaf level |
| Redundant keys | Doesn't keep duplicate keys around | Keeps duplicate keys, with all of them present at the leaf level |
| Insertion | Takes longer, and the outcome isn't always predictable | Generally easier, with consistent, predictable results |
| Deletion | Removing an internal node gets complicated | Easier, since everything needed is found at the leaf level |
| Leaf nodes | Not linked together structurally | Linked together structurally, like a linked list |
| Sequential access | Not really possible | Possible, much like traversing a linked list |
| Height | Tends to be larger for the same number of nodes | Tends to be smaller than an equivalent B-Tree |
| Typical use | Databases, search engines | Multilevel indexing, database indexing specifically |

It's worth pointing out that most mainstream DBMS platforms — MySQL, Oracle, PostgreSQL — actually rely on B+ Tree indexing under the hood.

## Ordered vs. unordered indices

### Ordered index

Here, records are kept in sorted order based on the key column.

| | |
|---|---|
| Advantages | Works efficiently for range queries (BETWEEN, >, <), and searching is generally faster |
| Disadvantages | Insertions slow down, since the sort order has to be maintained |

### Unordered index (heap file)

Here, records are just stored in whatever order they arrive, typically appended to the end.

| | |
|---|---|
| Advantages | Insertion is fast, since you're simply adding to the end |
| Disadvantages | Searching is slow, since you generally have to scan everything |

### Putting the two side by side

| Feature | Ordered Index | Unordered Index |
|---------|---------------|-----------------|
| Search speed | Fast | Slow |
| Insert speed | Slower | Fast |
| Range queries | Efficient | Inefficient |

## How indexing actually helps with search

The way an index speeds things up is by letting the database locate relevant rows directly, rather than scanning the whole table from start to finish.

Indexes can support a few different kinds of searches:

| Search Type | Example |
|--------------|---------|
| Exact match | Name = 'Sonam' |
| Range query | Age > 20 |
| Pattern matching | Name LIKE 'S%' (depending on the index type used) |

The performance difference is significant: without an index, search time grows linearly — O(n) — but with an index, it drops to logarithmic time — O(log n).

One important caveat though: piling on too many indexes can actually slow down INSERT, UPDATE, and DELETE operations, since every single index needs to be updated whenever the underlying data changes.

## Indexing temporal and spatial data

### Temporal data indexing

Temporal data is anything tied to time — employee salary history, or a patient's records tracked over time, for example. Here, the index gets structured around time values, which makes time-based queries and historical analysis run efficiently.

### Spatial data indexing

Spatial data deals with location or geometry — maps, GPS coordinates, that sort of thing — and it needs its own specialized structures, like R-trees or Quad-trees. The basic distinction to remember: temporal indexing is about giving you efficient access by time, while spatial indexing is about giving you efficient access by location.

## Query processing — what happens after you hit "run"

Query processing is the whole process of turning an SQL query into an efficient execution plan and actually getting the result back. When you type out an SQL query, the DBMS doesn't just run it as-is — it works through several stages first to understand, convert, improve, and finally execute it.

### The steps involved

**1. You write the SQL query**

```sql
SELECT name, age FROM Student WHERE age > 20;
```

**2. Parsing**

The DBMS checks that the query is actually well-formed. It's verifying things like: is the syntax correct, does the Student table actually exist, do the name and age columns exist, and is the condition itself valid? If something's wrong — say, a missing comma between name and age — the DBMS throws an error. In short, parsing is about checking and understanding the structure of what you've written.

**3. Translating into relational algebra**

Once parsed, the query gets converted into relational algebra — the internal mathematical representation the DBMS actually works with. The earlier example becomes something like PROJECT name, age (SELECT age > 20 (Student)) — meaning: take the Student table, keep only the rows where age is greater than 20, and then show just the name and age columns.

**4. Optimization**

There's usually more than one way to execute the same query, so the optimizer steps in to pick the fastest, cheapest route. For a query filtering on age, the DBMS might weigh scanning the whole table against using an index on age instead. If the table's small, a full scan might honestly be fine; if it's large and an index exists, using that index wins out. The optimizer is mainly trying to cut down on disk access, CPU time, memory usage, and overall execution time.

**5. Execution**

Finally, the DBMS runs whichever plan it settled on and hands back the result — actually reading data off disk, applying the conditions, and producing the final output you see.

The whole journey, simplified: SQL query → DBMS checks it → DBMS converts it into internal operations → DBMS picks the best method → DBMS runs it and returns the result. That's really the entire lifecycle of a query, from what you typed to the answer you get back.

## What determines query cost

The cost of running a query mainly comes down to:

- Disk I/O operations — generally the biggest factor by far
- CPU time
- Memory usage

For something simple like SELECT * FROM Student, the cost largely depends on how many disk blocks need to get read. Cutting down disk access is really the key lever for making queries faster.

## How expressions actually get evaluated

A straightforward query like SELECT * FROM Student WHERE Age > 20 follows a simple path: read the table, apply the condition (Age > 20), then output whatever matches.

Something more involved, like a join — SELECT * FROM A JOIN B ON A.id = B.id — takes a few more steps: read table A, read table B, then match up rows according to the join condition.

## Choosing how to actually evaluate a query

A query plan is just a specific strategy for carrying out a query — and different plans can land on the exact same result while performing very differently in practice.

For a join, a few common strategies come up:

| Strategy | How it works |
|----------|------------------|
| Nested Loop Join | Compares every row in table A against every row in table B — simple to understand, but can get slow once the tables grow large. |
| Hash Join | Builds a hash table from one table in memory, then probes it using rows from the other table for faster matching. |
| Sort-Merge Join | Sorts both tables by the join key first, then merges them together. |

The query optimizer automatically picks whichever plan has the lowest estimated cost, aiming for the fastest realistic execution strategy.

## Quick recap

| Topic | Key takeaway |
|-------|----------------|
| Indexing | Speeds up retrieval by giving direct access paths to records, instead of forcing a full scan |
| B+ Tree | The go-to indexing structure in modern databases, since it handles both exact matches and range queries well |
| Ordered indexes | Great for fast search and range queries, at the cost of slower inserts |
| Query processing | Turns an SQL query into an executable plan through parsing, translation, optimization, and execution |
| Query optimization | Picks the cheapest execution plan available, mostly driven by minimizing disk I/O |
