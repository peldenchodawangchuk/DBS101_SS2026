# Database Design — The ER Model & Relational Model

## How database design actually unfolds

Designing a database isn't a one-shot thing — it happens in stages, each building on the last:

| Stage | What happens |
|-------|---------------|
| Requirements Collection & Analysis | Talk to the actual users, dig out the business rules, and figure out what data the system genuinely needs to track. |
| Conceptual Database Design | Sketch out an ER diagram — no DBMS involved yet, just the ideas. |
| Logical Database Design | Turn that ER diagram into real tables, with keys and foreign keys, then normalize everything. |
| Physical Database Design | Get into indexes, storage layout, and performance tuning. |

## The building blocks of the ER model

### Entities and entity sets

An **entity** is just a real-world thing you care about — a Student, a Course, a Department. An **entity set** is the collection of all entities of the same type, like every Student in the system.

### The different flavors of attributes

Not every piece of data behaves the same way. Some attributes can be broken down further, some can't, and some are calculated rather than stored directly:

| Attribute Type | What it means | Example |
|----------------|----------------|---------|
| Simple | Can't be broken down any further | Age, Price, StudentID |
| Composite | Made up of smaller parts | Address (Village, Gewog, Dzongkhag), Name (First, Last) |
| Multivalued | Can hold more than one value at once | Phone numbers {123, 456}, Skills |
| Derived | Worked out from other data rather than stored | Age calculated from Date of Birth, Experience from Joining Date |

Worth noting: composite and multivalued attributes can actually nest inside each other. Something like PreviousDegrees(College, Year, Degree, Field) could be a multivalued attribute where each value is itself composite.

## Keys — how we tell rows apart

| Key Type | What it means | Example |
|----------|----------------|---------|
| Primary Key | Uniquely identifies each row; can't be NULL, and should be as minimal as possible | StudentID |
| Candidate Key | Any attribute that *could* have been chosen as the primary key | StudentID, Email |
| Composite Key | A primary key made up of two or more columns together | (StudentID, CourseID) in an Enrollment table |

A single entity type can actually have more than one candidate key sitting around. Take a CAR: the VehicleEngineNumber could work as a key, and so could the combination of (Number, State) on the license plate — either one uniquely identifies the car.

## Relationships and how they connect

### Cardinality — how many on each side

| Type | What it means | Example |
|------|----------------|---------|
| 1:1 | One entity connects to exactly one other | A Person and their Passport, an Employee managing a Department |
| 1:N | One entity connects to many on the other side | One Department has many Employees |
| N:1 | Many entities connect to a single one | Many Students belong to one Department |
| M:N | Many connect to many | Students and Courses — this needs its own junction table to work |

### Participation constraints — is it optional or required?

This is about whether an entity *has* to take part in a relationship or not:

- **Total participation** (drawn with a double line): every single entity must be involved — its existence depends on it. For example, every Employee must work for some Department.
- **Partial participation** (drawn with a single line): participation is optional. Not every Employee manages a Department, after all.

### The (min, max) shorthand

Instead of just "total" or "partial," you can be precise and write (min, max) next to each side of a relationship. So if an Employee in a WORKS_FOR relationship is marked (1,1), that means each employee belongs to exactly one department — no more, no less. If nothing's specified, the default assumption is min=0, max=n (basically: optional, and no upper limit).

## Weak entities — the ones that can't stand alone

A **weak entity** doesn't have a key attribute of its own strong enough to identify it uniquely. It depends entirely on another entity — its **owner** (or identifying entity type) — to be identified. The way you actually pin it down is by combining its own **partial key** with the owner's primary key. In an ER diagram, you'll spot these because they're drawn with a double rectangle.

A classic example from the COMPANY database: DEPENDENT is a weak entity, tied to EMPLOYEE through the DEPENDENTS_OF identifying relationship — a dependent only makes sense in the context of the employee they belong to.

## Recursive relationships — when an entity relates to itself

Sometimes an entity set needs to relate to *itself*, just with different roles on each side. To avoid confusion, you give each side a **role name**. The go-to example is SUPERVISION: an Employee acting as the boss supervises another Employee acting as the worker — same entity type, two different roles.

## Relationships can have their own attributes too

It's not just entities that carry attributes — relationships can have them as well. Take WORKS_ON: it might carry an attribute like `HoursPerWeek`, capturing how many hours a particular employee puts into a particular project. That value doesn't belong to the Employee or the Project alone — it only makes sense as a property of the relationship between them.

## Extended ER (EER) — the upgraded toolkit

The original ER model had some gaps, so Extended ER (EER) added a few more concepts to handle real-world complexity:

| Feature | What it means | Example |
|---------|----------------|---------|
| Specialization | Going top-down: starting from a general superclass and breaking it into more specific subclasses | Employee splits into Manager, Engineer |
| Generalization | Going bottom-up: starting from several specific entities and pulling out a common superclass | Car and Truck generalize into Vehicle |
| Inheritance | A subclass automatically gets all the attributes of its superclass | Manager inherits ID and Name from Employee |
| Categories (Union) | An entity can belong to one of several different entity sets | An Owner could be either a Person or a Company |

## Turning an ER diagram into actual tables

Once you've got your ER diagram sorted, there are some pretty mechanical rules for converting it into relational tables:

| Situation | How you convert it |
|-----------|----------------------|
| Regular entity | Becomes one table; every simple and composite attribute becomes a column. |
| Multivalued attribute | Gets pulled out into its own separate table, linked back with a foreign key to the owner. |
| M:N relationship | Needs a brand-new junction table, with the primary keys of both entities combined as a composite primary key. |
| 1:N relationship | The foreign key goes on the "many" side. |
| 1:1 relationship | The foreign key can go in either table — it's your call which one. |
| Weak entity | Its table includes the owner's primary key as a foreign key, and that foreign key becomes part of the weak entity's own primary key. |

A concrete example from the COMPANY database: the M:N relationship WORKS_ON turns into a junction table — WORKS_ON(EmployeeSSN, ProjectNumber, Hours).

## The relational model — basic vocabulary

| Term | What it actually means |
|------|--------------------------|
| Relation | A table |
| Tuple | A row |
| Attribute | A column |
| Domain | The set of allowed values for an attribute (e.g., Age has to fall between 0 and 120) |

### What makes something a proper relation

A few rules apply to keep a relation well-behaved:

- The order of the rows doesn't matter at all.
- Every row has to be unique — no exact duplicates.
- Attribute values need to be atomic — no cramming repeating groups into a single cell.
- Every attribute is tied to a domain that defines what values it can hold.

## Getting rid of redundant attributes

Here's a design mistake that's easy to fall into: imagine a Student table where the DeptLocation column is repeated for every single student who happens to belong to the same department. That's pure redundancy — the same fact stored over and over.

The fix is to split it into two separate tables: a Department table holding (DeptID, DeptLocation), and a Student table holding (StudentID, Name, DeptID), with DeptID linking the two.

Why does this redundancy actually matter? It causes a few real problems:

- **Data inconsistency** — the same fact stored in multiple places can drift out of sync.
- **Update anomalies** — changing one fact means having to update it everywhere it's duplicated.
- **Storage inefficiency** — you're wasting space storing the same thing repeatedly.

## Tricky design decisions — entity, attribute, or relationship?

A few judgment calls come up again and again when you're designing a schema:

| The dilemma | How to decide |
|-------------|-----------------|
| Should this be an entity or just an attribute? | If the thing has its own properties worth tracking, make it an entity. For instance, treat Phone Number as its own entity if you need to track multiple phones along with details like their type. |
| Should this be an entity or a relationship? | Enrollment is generally better modeled as a relationship between Student and Course rather than its own entity — unless it carries its own attributes, like a Grade, in which case giving it more structure starts to make sense. |
| Should this be a binary or ternary relationship? | Reach for a ternary relationship when three entities genuinely need to interact together at once — like a Supplier supplying a Part to a Project, where all three pieces matter simultaneously. |