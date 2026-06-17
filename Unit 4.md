# Intermediate & Advanced SQL

## 4.1 Join Expressions

Joins are how you pull rows together from two or more tables, based on columns that relate them to each other.

| Join Type | What it gives you |
|-----------|----------------------|
| INNER JOIN | Only the rows that have a match in both tables |
| LEFT OUTER JOIN | Every row from the left table, plus matches from the right (NULL where there's no match) |
| RIGHT OUTER JOIN | Every row from the right table, plus matches from the left |
| FULL OUTER JOIN | Every row from both tables, matched where possible |
| NATURAL JOIN | Joins automatically on columns that share the same name — handy, but use carefully since it can silently match on the wrong column |
| CROSS JOIN | A full cartesian product, pairing every row from one table with every row from the other |

A few examples to see these in action:

```sql
-- Inner join
SELECT * FROM student INNER JOIN enrollment ON student.id = enrollment.student_id;

-- Left outer join
SELECT * FROM student LEFT OUTER JOIN enrollment ON student.id = enrollment.student_id;

-- Natural join (same column names)
SELECT * FROM student NATURAL JOIN department;
```

## 4.2 Views

A view is essentially a virtual table — it's defined by a query, but it doesn't physically store any data of its own.

### Creating a view

```sql
CREATE VIEW cs_students AS
SELECT id, name
FROM student
WHERE dept = 'CS';
```

### When can you actually update through a view?

Not every view supports updates. A view stays updatable only if it steers clear of a few things:

- No DISTINCT, GROUP BY, or HAVING
- No subqueries inside the SELECT
- Generally limited to a single underlying table, if you want simple updates to work cleanly

### Materialized views — when you want the data actually stored

Unlike a regular view, a **materialized view** physically stores its results and gets refreshed on a schedule rather than recalculated every time. This trades a bit of freshness for a real performance boost.

```sql
CREATE MATERIALIZED VIEW dept_stats AS
SELECT dept, COUNT(*) AS cnt
FROM student
GROUP BY dept;
```

### Dropping a view

```sql
DROP VIEW cs_students CASCADE;  -- Drops dependent views too
```

## 4.3 Transactions

A transaction is a unit of work that's meant to behave as one atomic, consistent, isolated, and durable operation — the same ACID guarantees that come up throughout database theory.

| Property | What it guarantees |
|----------|------------------------|
| Atomicity | Either everything in the transaction happens, or none of it does |
| Consistency | The database's constraints stay intact before and after |
| Isolation | Transactions running at the same time don't interfere with each other |
| Durability | Once you commit, the change sticks around even through a crash |

### Controlling a transaction

```sql
BEGIN;           -- Start transaction (or START TRANSACTION)
-- SQL statements
COMMIT;          -- Save changes permanently
ROLLBACK;        -- Undo all changes since BEGIN
```

### Savepoints — rolling back partway

Savepoints let you undo just part of a transaction rather than the whole thing:

```sql
BEGIN;
INSERT INTO student VALUES (1, 'Alice');
SAVEPOINT sp1;
INSERT INTO student VALUES (2, 'Bob');
ROLLBACK TO sp1;   -- Bob undone, Alice kept
COMMIT;
```

## 4.4 Integrity Constraints

Constraints are the rules that keep your data honest and internally consistent.

| Constraint | What it's for |
|------------|------------------|
| PRIMARY KEY | Forces uniqueness and disallows NULL |
| FOREIGN KEY | Points to a primary key in another table |
| UNIQUE | Blocks duplicate values |
| NOT NULL | Won't allow an empty value |
| CHECK | Enforces whatever custom condition you define |
| DEFAULT | Fills in a default value when none is supplied |

```sql
CREATE TABLE student (
    id INT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    age INT CHECK (age >= 18),
    status VARCHAR(20) DEFAULT 'active'
);

-- Foreign key with cascade
CREATE TABLE enrollment (
    student_id INT,
    course_id INT,
    FOREIGN KEY (student_id) REFERENCES student(id) 
        ON DELETE CASCADE ON UPDATE CASCADE
);
```

### What happens when a constraint would be violated

| Action | What it does |
|--------|----------------|
| ON DELETE CASCADE | Deletes the related child rows too |
| ON DELETE SET NULL | Sets the foreign key to NULL instead of deleting |
| ON DELETE RESTRICT | Blocks the deletion outright |
| ON UPDATE CASCADE | Propagates the update down to child rows |

## 4.5 SQL Data Types and Schemas

### The built-in data types

| Category | Types |
|----------|-------|
| Numeric | INT, SMALLINT, DECIMAL(p,s), FLOAT, REAL |
| Character | CHAR(n), VARCHAR(n), TEXT |
| Date/Time | DATE, TIME, TIMESTAMP, INTERVAL |
| Boolean | BOOLEAN |
| Large Object | BLOB, CLOB |
| JSON | JSON, JSONB (PostgreSQL) |

### Defining your own types

The SQL standard also lets you build custom types when the built-ins don't quite fit:

```sql
CREATE TYPE address_type AS (
    street VARCHAR(100),
    city VARCHAR(50),
    zip CHAR(5)
);

CREATE TABLE person (
    id INT,
    address address_type
);
```

### Domains — a type plus built-in constraints

A domain bundles a base type together with a constraint, so you don't have to repeat that constraint everywhere the type is used:

```sql
CREATE DOMAIN positive_int AS INT CHECK (VALUE > 0);
CREATE TABLE product (
    id INT,
    price positive_int
);
```

### Schemas — organizing your tables

A schema is just a logical container that groups together tables, views, and other database objects.

```sql
CREATE SCHEMA university;
CREATE TABLE university.student (...);

SET search_path TO university, public;  -- Set default schema
DROP SCHEMA university CASCADE;
```

## 4.6 Indexes

Indexes exist purely to speed up queries — especially ones involving WHERE, JOIN, or ORDER BY.

### Creating indexes

```sql
-- Simple index
CREATE INDEX idx_student_name ON student(name);

-- Unique index
CREATE UNIQUE INDEX idx_unique_email ON student(email);

-- Composite index
CREATE INDEX idx_enrollment_ids ON enrollment(student_id, course_id);

-- Bitmap index (for low-cardinality columns)
CREATE BITMAP INDEX idx_status ON student(status);
```

### Dropping an index

```sql
DROP INDEX idx_student_name;
```

### When indexing actually helps — and when it doesn't

| Good candidates for an index | Better left unindexed |
|-------------------------------|--------------------------|
| Columns frequently used in WHERE or JOIN | Columns that get updated constantly (the index adds overhead every time) |
| Foreign key columns | Small tables, where scanning is already cheap |
| Columns with high selectivity (lots of unique values) | Low-selectivity columns, like a boolean or gender field |
| Columns used in ORDER BY or GROUP BY | — |

## 4.7 Authorization

This is about who's allowed to do what with your data.

### Privileges you can grant

| Privilege | What it allows |
|-----------|-------------------|
| SELECT | Reading data |
| INSERT | Adding new rows |
| UPDATE | Modifying existing rows |
| DELETE | Removing rows |
| REFERENCES | Creating foreign keys pointing at this table |
| ALL PRIVILEGES | Everything above, bundled together |

### Granting privileges

```sql
GRANT SELECT, INSERT ON student TO 'alice'@'localhost';

GRANT ALL ON university.* TO 'admin'@'%' WITH GRANT OPTION;
```

### Revoking privileges

```sql
REVOKE INSERT ON student FROM 'alice'@'localhost';

REVOKE GRANT OPTION FOR SELECT ON student FROM 'admin'@'%';
```

### Roles — grouping privileges together

Rather than granting permissions one user at a time, you can bundle them into a role and hand that role out:

```sql
CREATE ROLE instructor;
GRANT SELECT, UPDATE ON student TO instructor;
GRANT instructor TO alice, bob;
```

## 4.8 Accessing SQL from a Programming Language

### JDBC (Java)

Connecting to a database from Java generally follows the same handful of steps:

```java
// Step 1: Load driver
Class.forName("org.postgresql.Driver");

// Step 2: Connect
Connection conn = DriverManager.getConnection(
    "jdbc:postgresql://localhost/db", "user", "pass");

// Step 3: Create statement
Statement stmt = conn.createStatement();

// Step 4: Execute query
ResultSet rs = stmt.executeQuery("SELECT * FROM student");
while (rs.next()) {
    System.out.println(rs.getString("name"));
}

// Step 5: PreparedStatement (prevents SQL injection)
PreparedStatement pstmt = conn.prepareStatement(
    "INSERT INTO student VALUES (?, ?)");
pstmt.setInt(1, 101);
pstmt.setString(2, "Alice");
pstmt.executeUpdate();

// Step 6: Close
rs.close(); stmt.close(); conn.close();
```

### Guarding against SQL injection

Building queries by directly stitching user input into a string is a classic security mistake — it opens the door for someone to inject their own SQL. Parameterized queries close that gap by keeping the user's input separate from the query structure:

```python
# BAD (vulnerable)
cursor.execute(f"SELECT * FROM user WHERE name = '{user_input}'")

# GOOD (parameterized)
cursor.execute("SELECT * FROM user WHERE name = %s", (user_input,))
```

## 4.9 Functions and Stored Procedures

### Functions — return a value

```sql
CREATE FUNCTION get_student_count(dept_name VARCHAR)
RETURNS INT
LANGUAGE SQL
AS $$
    SELECT COUNT(*) FROM student WHERE dept = dept_name;
$$;

-- Call function
SELECT get_student_count('CS');
```

### Stored procedures — no return value, but can manage transactions

```sql
CREATE PROCEDURE transfer_money(
    from_acc INT,
    to_acc INT,
    amount DECIMAL
)
LANGUAGE SQL
AS $$
    UPDATE account SET balance = balance - amount WHERE id = from_acc;
    UPDATE account SET balance = balance + amount WHERE id = to_acc;
$$;

-- Call procedure
CALL transfer_money(101, 102, 500);
```

### Going further with procedural logic (PL/pgSQL, PostgreSQL)

When plain SQL isn't expressive enough — say, you need conditional branching — PL/pgSQL lets you write something closer to a real program:

```sql
CREATE FUNCTION calc_bonus(salary DECIMAL, rating INT)
RETURNS DECIMAL
LANGUAGE plpgsql
AS $$
DECLARE
    bonus DECIMAL;
BEGIN
    IF rating >= 4 THEN
        bonus := salary * 0.20;
    ELSIF rating >= 2 THEN
        bonus := salary * 0.10;
    ELSE
        bonus := 0;
    END IF;
    RETURN bonus;
END;
$$;
```

## 4.10 Triggers

Triggers fire automatically whenever an INSERT, UPDATE, or DELETE happens — no manual call required.

| Trigger Type | When it fires |
|---------------|------------------|
| BEFORE | Before the write happens, often used to modify the data first |
| AFTER | After the write, good for logging or cascading other actions |
| INSTEAD OF | Used on views, to handle more complex update logic |
| FOR EACH ROW | Runs once per affected row |
| FOR EACH STATEMENT | Runs once for the entire statement, regardless of how many rows it touches |

### Dropping a trigger

```sql
DROP TRIGGER salary_log_trigger ON employee;
```

## 4.11 Recursive Queries

Some queries need to reference themselves — usually to walk through some kind of hierarchy. SQL handles this through a Common Table Expression (CTE) marked as RECURSIVE.

### Example: walking an org chart

```sql
WITH RECURSIVE org_chart AS (
    -- Base: CEO (no manager)
    SELECT id, name, manager_id, 1 AS level
    FROM employee
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: employees reporting up
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employee e
    INNER JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level;
```

This starts from the top of the org chart (whoever has no manager) and keeps joining back in, one level deeper each time, until it's covered everyone.

### Example: generating a Fibonacci sequence

```sql
WITH RECURSIVE fib(a, b) AS (
    SELECT 0::INT, 1::INT
    UNION ALL
    SELECT b, a + b FROM fib WHERE b < 1000
)
SELECT a FROM fib;
```

## 4.12 Advanced Aggregation Features

### ROLLUP — subtotals plus a grand total

```sql
SELECT dept, year, SUM(salary)
FROM employee
GROUP BY ROLLUP(dept, year);
-- Produces: (dept, year), (dept, NULL), (NULL, NULL)
```

### CUBE — every possible combination of groupings

```sql
SELECT dept, year, SUM(salary)
FROM employee
GROUP BY CUBE(dept, year);
-- Produces: (dept,year), (dept,NULL), (NULL,year), (NULL,NULL)
```

### GROUPING SETS — picking exactly the combinations you want

```sql
SELECT dept, year, SUM(salary)
FROM employee
GROUP BY GROUPING SETS ((dept, year), (dept), ());
```

### The GROUPING() function — telling apart two kinds of NULL

Because ROLLUP and CUBE produce their own NULLs to represent "all values," it gets confusing if your actual data also contains NULLs. GROUPING() tells you which is which:

```sql
SELECT 
    dept,
    year,
    SUM(salary),
    GROUPING(dept) AS is_dept_null,
    GROUPING(year) AS is_year_null
FROM employee
GROUP BY ROLLUP(dept, year);
```

### Window functions — calculating across rows without losing them

Unlike GROUP BY, which collapses rows down into groups, window functions let you perform calculations across a set of rows while still keeping each row individually visible.

```sql
-- ROW_NUMBER: sequential numbering
SELECT 
    name, 
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS rank;

-- RANK (with ties)
SELECT 
    name, 
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank;

-- Running total
SELECT 
    date, 
    amount,
    SUM(amount) OVER (ORDER BY date) AS running_total;

-- Partition by department
SELECT 
    name, 
    dept, 
    salary,
    AVG(salary) OVER (PARTITION BY dept) AS dept_avg
FROM employee;
```

### The common window functions, side by side

| Function | What it does |
|----------|-----------------|
| ROW_NUMBER() | Gives each row a unique sequential number |
| RANK() | Ranks rows, leaving gaps when there's a tie |
| DENSE_RANK() | Ranks rows without leaving gaps |
| LEAD(column, n) | Looks ahead n rows for a value |
| LAG(column, n) | Looks back n rows for a value |
| FIRST_VALUE() | Grabs the first value within the window |
| LAST_VALUE() | Grabs the last value within the window |

### Example: comparing year-over-year performance

```sql
SELECT 
    year,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY year) AS prev_year_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY year) AS growth
FROM sales;
```

LAG() here pulls in the previous year's revenue right alongside the current row, making it easy to calculate how much things grew (or shrank) year over year without needing a separate self-join.
