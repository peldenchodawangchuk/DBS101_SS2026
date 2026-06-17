# Introduction to Database Systems

## What is a database?

Data, on its own, is just raw stuff, a name, a number, a grade, nothing organized about it. A database is what happens when you take that raw stuff and structure it so it actually means something, then store it electronically so it can be used. The DBMS (database management system) is the software that sits on top of all that, letting you create, manage, and access the data without having to deal with the messy details underneath.

## Why bother with database systems at all?

Before databases became standard, people stored data in plain file systems back in the 1950s, and it was a mess, the same information got duplicated all over the place, updates didn't always sync up, and there was no good way for multiple people to use the data at once. Database systems exist to fix exactly those problems: they cut down on redundant copies of data, keep things consistent so an update in one place shows up everywhere it should, let multiple users work with the data simultaneously, enforce sensible rules (like not letting someone's age be negative), control who can see or touch what, and make sure everything can be recovered if something crashes.

## Three ways to look at the same data

It helps to think about data on three different levels. At the **physical level**, you're dealing with how the data actually sits on disk sectors, indexes, that kind of thing. At the **logical level**, you're thinking about what data exists and how it relates to other data, like a Student table with an ID, Name, and Department. At the **view level**, different users only get to see the slice of data that's relevant to them — a lecturer might see a student's grades but not their home address.

## How we got here

The evolution of database systems is basically a story of solving each generation's problems:

- **1950s–60s:** File systems — riddled with redundancy and inconsistency.
- **1960s:** Hierarchical model — data organized like a tree, each record with one parent (IBM's IMS is the classic example).
- **1970s:** Network model (CODASYL) — records could have multiple parents instead of just one.
- **1970s–present:** Relational model — pioneered by Edgar Codd, organized everything into tables and gave us SQL. This is the one that stuck, mostly because it was simpler to work with.
- **Modern day:** NoSQL, object-oriented, and distributed cloud systems handling needs the relational model wasn't built for.

## The languages databases speak

SQL isn't just one thing, it's really a bundle of sub-languages, each handling a different job:

- **DDL** — defines or changes the structure of the database (CREATE, ALTER, DROP)
- **DML** — manipulates the actual data (INSERT, UPDATE, DELETE)
- **DQL** — retrieves data (SELECT)
- **DCL** — handles permissions (GRANT, REVOKE)
- **TCL** — manages transactions (COMMIT, ROLLBACK)

## How the pieces talk to each other

Database architecture is about how your application and your database actually communicate:

- **1-Tier:** Everything happens on one machine, like running MS Access locally.
- **2-Tier:** A client app talks directly to a database server.
- **3-Tier:** The model you'll see most often for web applications — split into a UI layer, an application logic layer, and a database layer. Think of your browser talking to a web server, which then talks to the database server.

## What's actually running the show

At the heart of any DBMS is the **database engine**, which handles the heavy lifting: storing data on disk, processing queries efficiently, managing transactions, and making sure concurrent users don't step on each other's toes. InnoDB (used by MySQL), the PostgreSQL engine, and Oracle's engine are all real-world examples.

## ACID — the rules transactions live by

When people talk about reliable transactions, they're talking about ACID:

- **Atomicity** — a transaction is all-or-nothing; if any part fails, the whole thing rolls back.
- **Consistency** — the database moves from one valid state to another, with rules like "balance can't go negative" always respected.
- **Isolation** — transactions behave as though they're running alone, even when others are happening at the same time.
- **Durability** — once something's committed, it survives even a crash or power outage.

A good way to picture this is booking the last seat on a flight: the booking either goes through completely or not at all (atomic), the price charged matches the fare (consistent), only one person actually gets that seat even if others tried at the same time (isolated), and once it's confirmed, it stays confirmed (durable).

## Building a database, step by step

Designing a database typically happens in three stages:

1. **Conceptual** — sketch out entities and relationships in an ER model.
2. **Logical** — translate that into actual relational tables.
3. **Physical** — figure out indexes, storage, and how data will actually be accessed.

The goal throughout is to minimize redundancy, keep data integrity intact, and make sure queries run fast.

## Who's actually using all this

- **DBA (Database Administrator)** — handles security, backups, performance tuning, and user accounts.
- **Application programmers** — write the software that talks to the database.
- **Sophisticated users** — write their own complex queries directly.
- **End users** — like bank customers or students, just use the apps built on top of it all, often without realizing there's a database underneath.
- **Specialized users** — people building CAD or AI systems, with their own particular needs from the database.

## Where you'll actually run into databases

They're everywhere once you start looking: banks rely on them for accounts and transactions, airlines for reservations and schedules, universities for grades and registration, e-commerce sites for products and orders, hospitals for patient records, and telecom companies for billing and usage tracking. Social media platforms, government record-keeping, and online shopping all run on the same basic idea.