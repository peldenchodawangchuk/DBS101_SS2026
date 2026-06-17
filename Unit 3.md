# Structured Query Language (SQL)

## What SQL actually is

SQL — Structured Query Language — is the standard way we talk to relational databases. Since data in these systems lives in tables made up of rows and columns, SQL gives you the tools to create those tables, put data into them, pull data back out, update what's already there, and get rid of data you no longer need.

## The different categories of SQL commands

SQL isn't one single thing — it's split into a handful of sub-languages, each handling a different kind of job:

| Category | What it's for | Commands |
|----------|-----------------|----------|
| DDL (Data Definition Language) | Defining and changing the structure of the database | CREATE, ALTER, DROP |
| DML (Data Manipulation Language) | Changing the data sitting inside tables | INSERT, UPDATE, DELETE |
| DQL (Data Query Language) | Pulling data back out | SELECT |
| DCL (Data Control Language) | Controlling who's allowed to access what | GRANT, REVOKE |
| TCL (Transaction Control Language) | Managing transactions | COMMIT, ROLLBACK |

## Data Definition Language (DDL)

DDL is all about shaping the database itself — building the tables and laying down the rules (constraints) that keep the data trustworthy.

### CREATE TABLE — building a new table

This is how you bring a table into existence, specifying its columns and any rules attached to them.

```sql
CREATE TABLE Student (
    StudentID INT PRIMARY KEY,
    Name VARCHAR(50),
    Age INT
);
```

### ALTER TABLE — changing a table that already exists

Use this when you need to add, tweak, or remove a column from a table you've already created.

```sql
ALTER TABLE Student ADD Email VARCHAR(100);
```

### DROP TABLE — getting rid of a table for good

This permanently removes a table — and everything in it — from the database.

```sql
DROP TABLE Student;
```

### Constraints — the rules that keep data honest

Constraints are how you enforce sensible behavior on the data going into your tables:

| Constraint | What it does |
|------------|----------------|
| PRIMARY KEY | Uniquely identifies every row; can never be NULL |
| FOREIGN KEY | Links one table to another, creating a relationship |
| NOT NULL | Stops a column from ever being left empty |
| UNIQUE | Guarantees every value in a column is different from the rest |

Here's a foreign key in action, linking a Course back to the Student who's taking it:

```sql
CREATE TABLE Course (
    CourseID INT PRIMARY KEY,
    StudentID INT,
    FOREIGN KEY (StudentID) REFERENCES Student(StudentID)
);
```

## Data Manipulation Language (DML)

While DDL shapes the structure, DML is about actually working with the data that lives inside it.

### INSERT — adding new rows

```sql
-- Single row
INSERT INTO Student (StudentID, Name, Age)
VALUES (1, 'Pema', 19);

-- Multiple rows
INSERT INTO Student VALUES (2, 'Rinzin', 22);
INSERT INTO Student VALUES (3, 'Deolkar', 19);
```

### UPDATE — changing data that's already there

The WHERE clause is doing important work here — it tells the database exactly which rows to touch. Leave it out, and every single row in the table gets updated, which is rarely what you actually want.

```sql
UPDATE Student
SET Age = 21
WHERE StudentID = 1;
```

### DELETE — removing rows

The same caution applies here: WHERE narrows down which rows disappear. Skip it, and you'll wipe out every row in the table (though the table itself stays intact, just empty).

```sql
DELETE FROM Student WHERE StudentID = 3;

-- Deletes all rows but keeps the table structure
DELETE FROM Student;
```

## Data Query Language — the SELECT statement

SELECT is how you actually get data back out of the database, and it's by far the most-used command in everyday SQL work.

### The basics

```sql
-- Select specific columns
SELECT Name, Age FROM Student;

-- Select all columns
SELECT * FROM Student;
```

### Filtering with WHERE

WHERE narrows your results down to only the rows that match certain conditions. You've got the usual comparison operators (=, >, <, >=, <=, <>) plus AND, OR, and NOT to combine conditions:

```sql
SELECT Name FROM Student WHERE Age > 18;
SELECT * FROM Student WHERE Age > 18 AND Name = 'Pema';
```

### Pattern matching with LIKE

When you need to match a pattern rather than an exact value, LIKE comes in handy, using % for "any sequence of characters" and _ for "exactly one character":

```sql
SELECT * FROM Student WHERE Name LIKE 'A%';
```

### Checking ranges and sets of values

```sql
SELECT * FROM Student WHERE Age BETWEEN 18 AND 25;
SELECT * FROM Student WHERE Age IN (18, 20, 22);
```

### Sorting with ORDER BY

This controls the order your results come back in — ASC for ascending (the default if you don't specify), DESC for descending.

```sql
SELECT Name, Age FROM Student ORDER BY Age DESC;
```

### Aggregate functions — boiling many rows down to one number

These take a whole bunch of rows and crunch them down into a single summary value:

| Function | What it gives you |
|----------|---------------------|
| COUNT(*) | The number of rows |
| SUM(column) | The total of all the values |
| AVG(column) | The average value |
| MAX(column) | The highest value |
| MIN(column) | The lowest value |

```sql
SELECT COUNT(*) FROM Student;
SELECT AVG(Age) FROM Student;
SELECT MAX(Marks), MIN(Marks) FROM Student;
SELECT SUM(Marks) FROM Student;
```

### GROUP BY — bucketing rows together

This groups rows that share the same value in a particular column, so you can then run aggregate functions on each group separately rather than the whole table at once.

```sql
SELECT Age, COUNT(*)
FROM Student
GROUP BY Age;
```

### HAVING — filtering the groups themselves

It's easy to mix this up with WHERE, so here's the distinction: WHERE filters individual rows *before* grouping happens, while HAVING filters entire groups *after* they've already been formed.

```sql
SELECT Age, COUNT(*)
FROM Student
GROUP BY Age
HAVING COUNT(*) > 1;
```

## Set operations — combining results from multiple queries

Sometimes you want to merge or compare the results of two separate SELECT queries. For this to work, both queries need to return the same number of columns, with matching data types:

| Operation | What it does |
|-----------|----------------|
| UNION | Merges both result sets and strips out duplicates |
| UNION ALL | Merges both result sets and keeps every duplicate |
| INTERSECT | Returns only the rows that show up in both queries |
| EXCEPT | Returns rows from the first query that don't appear in the second |

```sql
SELECT Name FROM Student
UNION
SELECT Name FROM Teacher;

SELECT Name FROM Student
UNION ALL
SELECT Name FROM Teacher;

SELECT Name FROM Student
INTERSECT
SELECT Name FROM Alumni;

SELECT Name FROM Student
EXCEPT
SELECT Name FROM Alumni;
```

One thing worth remembering: UNION ALL tends to run faster than plain UNION, simply because it skips the extra work of checking for and removing duplicates.

## Handling NULL values

NULL represents data that's missing, unknown, or just not available — and it's important to be clear that this is different from zero, a blank space, or an empty string. It's its own special case entirely.

Because of that, you can't check for NULL using the usual = or != operators — you need IS NULL or IS NOT NULL instead:

```sql
-- Find rows with NULL in Age column
SELECT * FROM Student WHERE Age IS NULL;

-- Find rows without NULL
SELECT * FROM Student WHERE Age IS NOT NULL;

-- Set a value to NULL
UPDATE Student SET Age = NULL WHERE StudentID = 2;
```

A couple of details that trip people up: comparing NULL to NULL with an equals sign doesn't actually evaluate to true — NULL just doesn't play by normal comparison rules. And aggregate functions quietly skip over NULL values when doing their calculations, except for COUNT(*), which counts rows regardless.

## Nested subqueries — queries inside queries

A subquery is exactly what it sounds like: a query tucked inside another query, where the inner one produces a result that the outer one then makes use of.

### Single-row subqueries

These return just one value, which makes them perfect for pairing with comparison operators like >, <, or =.

```sql
SELECT Name
FROM Student
WHERE Age > (SELECT AVG(Age) FROM Student);
```

Here, the inner query works out the average age across all students, and the outer query then finds anyone older than that average.

### Multiple-row subqueries

These return a whole list of values rather than just one, so they pair naturally with the IN operator.

```sql
SELECT Name
FROM Student
WHERE StudentID IN (SELECT StudentID FROM Course);
```

The inner query gathers up every StudentID that appears in the Course table, and the outer query then matches students against that list.

### Correlated subqueries

This is the trickiest variety — the inner query actually depends on whichever row the outer query happens to be processing at that moment, so it effectively runs once per row rather than just once overall.

```sql
SELECT StudentID, Subject, Marks
FROM Exam E
WHERE Marks > (
    SELECT AVG(Marks)
    FROM Exam
    WHERE StudentID = E.StudentID
);
```

For every row in the Exam table, the inner query recalculates the average marks for that specific student, and the outer query keeps only the rows where the student scored above their own personal average.

## Quick recap

| Topic | Key takeaway |
|-------|----------------|
| SQL | The standard language for working with relational databases |
| DDL | Shapes structure with CREATE, ALTER, and DROP |
| DML | Changes data with INSERT, UPDATE, and DELETE |
| SELECT | Retrieves data, often paired with WHERE, ORDER BY, GROUP BY, and HAVING |
| Set operations | Combine results from more than one query |
| NULL | Needs special handling via IS NULL / IS NOT NULL, not = or != |
| Aggregate functions | Crunch groups of rows down into single summary values |
| Subqueries | Let you nest one query inside another for more complex logic |
| WHERE with UPDATE/DELETE | Always double-check it — skipping it affects every row in the table |