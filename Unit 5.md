# Normalisation and Functional Dependencies

## Why normalisation matters in the first place

When a relation hasn't been normalised properly, it tends to run into a handful of predictable problems:

| Problem | What it looks like |
|---------|----------------------|
| Redundancy | The same fact gets repeated over and over — like storing a department's name and office alongside every single student who belongs to that department, so the office shows up again and again. |
| Update anomaly | Changing one fact means you have to go update it everywhere it's repeated. Move a department's office, and now every student row tied to that department needs editing. |
| Insertion anomaly | You can't record a new fact without dragging in some unrelated fact too. A brand-new department can't be added to the database until at least one student has actually enrolled in it. |
| Deletion anomaly | Deleting one fact accidentally wipes out another fact you still needed. Remove the last student in a department, and you lose all record of the department itself. |

The whole point of good relational design boils down to one simple idea: each fact should live in exactly one place.

## Decomposition — splitting relations apart

Decomposition means breaking a relation down into two or more smaller relations, with the goal of getting rid of redundancy.

### Lossless-join decomposition

If you split a relation and then join the pieces back together, you should land right back where you started — no extra rows showing up, and no rows quietly disappearing. A decomposition earns the label "lossless" when the attributes shared between the two new relations are enough to functionally determine all the attributes of at least one of them.

### Dependency preservation

Ideally, you'd want to be able to check every functional dependency that existed in the original relation just by looking at the decomposed pieces — without having to join everything back together first. When that holds true, the decomposition is said to preserve dependencies.

## Functional dependency theory

A functional dependency (FD) gets written as X → Y, and it means that whenever two rows agree on every attribute in X, they're guaranteed to agree on every attribute in Y too. Put simply: X determines Y.

A few terms worth knowing:

| Term | What it means |
|------|------------------|
| Determinant | The left-hand side of an FD |
| Dependent | The right-hand side of an FD |
| Trivial FD | One where Y is already a subset of X — like AB → A |
| Nontrivial FD | One where Y isn't a subset of X |
| Superkey | Any set of attributes that uniquely identifies a row |
| Candidate key | A superkey that's as small as it can possibly be (minimal) |

### Armstrong's Axioms — the foundational rules

These three rules are the bedrock that everything else about functional dependencies gets built on:

- **Reflexivity:** If Y is a subset of X, then X → Y automatically holds — trivial dependencies are always true by definition.
- **Augmentation:** If X → Y holds, then adding the same attributes Z to both sides still holds: XZ → YZ.
- **Transitivity:** If X → Y and Y → Z, then X → Z — dependencies chain together just like you'd expect.

### A few useful rules derived from those axioms

- **Union:** If X → Y and X → Z, then X → YZ.
- **Decomposition:** If X → YZ, then both X → Y and X → Z hold individually.
- **Pseudotransitivity:** If X → Y and WY → Z, then WX → Z.

### Attribute closure

The closure of an attribute set X — written X⁺ — is everything X can functionally determine, given the set of dependencies you're working with. This turns out to be a genuinely useful tool:

- It tells you whether X qualifies as a superkey.
- It lets you check whether some FD is actually implied by the dependencies you already have.
- It helps you systematically track down all the candidate keys.

### Canonical cover

A canonical cover is the leanest possible version of a set of functional dependencies that's still equivalent to the original. It satisfies three things at once: no extraneous attributes hanging around on either side of any FD, no dependencies that are redundant given the others, and ideally just one FD per left-hand side.

## Normal forms

Normal forms are essentially formal checkpoints for organizing relations so they avoid redundancy and the anomalies that come with it.

### First Normal Form (1NF)

A relation hits 1NF once every attribute holds only atomic values — meaning each cell contains exactly one value, with no repeating groups or multivalued attributes hiding inside it.

A typical violation would be a PhoneNumbers column storing something like "17111111, 17222222" crammed into a single cell. The fix is straightforward: give each phone number its own row so every cell holds just one value.

1NF really is the bedrock the whole relational model rests on — without atomic values, basic relational operations like selection, projection, and joining start to break down.

### Second Normal Form (2NF)

A relation reaches 2NF when two things are true: it's already in 1NF, and every non-prime attribute is fully dependent on the *entire* candidate key, not just part of it.

A couple of definitions help here:

| Term | What it means |
|------|------------------|
| Prime attribute | An attribute that belongs to at least one candidate key |
| Non-prime attribute | An attribute that doesn't belong to any candidate key |
| Partial dependency | A non-prime attribute that only depends on part of a composite candidate key, rather than the whole thing |

A good violation example: ENROLLMENT(StudentID, CourseID, StudentName, CourseName, Grade), with a composite key of (StudentID, CourseID). The problem is that StudentName only really depends on StudentID, and CourseName only depends on CourseID — neither needs the full composite key.

The fix is to decompose it into three relations:

- STUDENT(StudentID, StudentName)
- COURSE(CourseID, CourseName)
- ENROLLMENT(StudentID, CourseID, Grade)

Worth noting: if a relation's key is just a single attribute, partial dependency simply can't happen — so any such relation is automatically in 2NF as long as it's already in 1NF.

### Third Normal Form (3NF)

A relation is in 3NF when it's already in 2NF, and there's no transitive dependency of a non-prime attribute on a candidate key.

More formally: for every nontrivial FD X → A, at least one of the following has to be true — either X is a superkey, or A is a prime attribute.

So what's a transitive dependency, exactly? It happens when a key determines some non-key attribute, and that non-key attribute in turn determines yet another non-key attribute — meaning the second one only depends on the key indirectly, through the first.

Take EMPLOYEE(EmpID, EmpName, DeptID, DeptName), with EmpID as the key, and dependencies EmpID → DeptID and DeptID → DeptName. That chain means EmpID → DeptName transitively, which violates 3NF.

The fix is to split it into:

- EMPLOYEE(EmpID, EmpName, DeptID)
- DEPARTMENT(DeptID, DeptName)

3NF tends to be the workhorse of practical database design — it clears up most of the everyday redundancy problems while still usually managing to preserve all the dependencies. It's generally seen as the sweet spot between theoretical rigor and what's actually practical to implement.

### Boyce-Codd Normal Form (BCNF)

BCNF is a stricter version of 3NF. A relation is in BCNF if, for every nontrivial FD X → A, X has to be a superkey — full stop, no exceptions.

The key difference from 3NF: 3NF gives you a pass if the right-hand side happens to be a prime attribute, even when the left-hand side isn't a superkey. BCNF doesn't offer that pass.

BCNF gives you tighter control over redundancy than 3NF does, but there's a catch — decomposing into BCNF can sometimes fail to preserve all the original functional dependencies. That's exactly why 3NF often gets chosen in practice when preserving dependencies actually matters.

### The BCNF decomposition algorithm

If a relation violates BCNF because of some FD X → Y where X isn't a superkey, you decompose it into:

- R1 = X ∪ Y
- R2 = R − (Y − X)

You keep repeating this process until every resulting relation satisfies BCNF. This approach always guarantees a lossless decomposition, though it might not preserve every dependency along the way.

### The 3NF synthesis algorithm

This is the method for producing a 3NF decomposition that's guaranteed to preserve dependencies:

1. Work out the canonical cover Fc.
2. For each FD X → Y in Fc, create a relation made up of X ∪ Y.
3. If none of the relations you've created contains a full candidate key, add one more relation that does.
4. Drop any relation that's already fully contained within another one.

Following this guarantees you end up with a lossless join, full dependency preservation, and every resulting schema sitting comfortably in 3NF.

### Fourth Normal Form (4NF)

A relation reaches 4NF once it's in BCNF and has no multivalued dependencies (MVDs) left. A multivalued dependency, written X →→ Y, means X determines several independent values of Y at once.

Picture a relation tracking Student, Hobby, and Language. If a student's hobbies and languages are genuinely independent of each other, you'd end up with both Student →→ Hobby and Student →→ Language. Storing all of that together forces you to repeat combinations unnecessarily, even though there's no actual functional dependency being violated.

4NF exists specifically to clean up the redundancy that comes from these kinds of independent multivalued facts.

### Fifth Normal Form (5NF), also known as Project-Join Normal Form

A relation hits 5NF when every nontrivial join dependency it has is already implied by its candidate keys. This addresses a subtler case: situations where a relation can be losslessly broken down into several projections, but the redundancy involved can't be explained just by functional or multivalued dependencies alone.

For most undergraduate-level database work, 1NF through BCNF cover the essentials. 4NF becomes relevant once multivalued attributes show up, while 5NF and DKNF are generally reserved for more advanced study.

### Normal forms at a glance

| Normal Form | Main Condition | What It Removes |
|-------------|------------------|---------------------|
| 1NF | Only atomic values allowed | Repeating groups, multivalued cells |
| 2NF | No partial dependency | Redundancy from a non-key attribute depending on only part of a composite key |
| 3NF | No transitive dependency | Redundancy among non-key attributes |
| BCNF | Every determinant must be a superkey | Redundancy that slips past 3NF |
| 4NF | No multivalued dependencies | Redundancy from independent multivalued facts |
| 5NF | No unexplained join dependencies | Complex, join-based redundancy |

## Multivalued dependencies

A multivalued dependency (MVD), written X →→ Y, shows up when one set of attributes determines several independent values of another set.

This matters whenever a relation holds two or more independent multivalued facts tied to the same key. The textbook example is a relation with Student, Hobby, and Language — if hobbies and languages really are independent of one another, every possible combination ends up represented, which creates serious redundancy even without any functional dependency being broken.

Whenever MVDs like this show up, it's a sign the relation needs to be broken down further to reach 4NF.

## Atomic domains and 1NF, revisited

An "atomic" domain just means the database treats the values in it as indivisible for the purposes of relational operations — and whether something counts as atomic can genuinely depend on the application. A full address might be perfectly atomic in one system, but in another system that needs to query street, city, and postcode separately, it makes more sense to break it apart.

If you spot a table with columns like Phone1, Phone2, Phone3, or a single cell crammed with comma-separated values, that's usually a red flag for poor design, and it's worth decomposing.

1NF earns its keep by enabling:

- A clean, well-defined tuple structure
- Standard relational operations that actually behave as expected
- Easier querying, and an easier path toward further normalisation

## The database design process, in practice

Database design tends to get described as a neat, linear sequence from requirements all the way to implementation — but in reality, it's much more iterative, with designers bouncing back and forth between stages as their understanding deepens.

| Stage | What happens |
|-------|-----------------|
| Requirements analysis | Understand what users actually need — transactions, reports, business rules, constraints, and the data items involved. Talk to stakeholders to figure out exactly what needs to be stored. |
| Conceptual design | Build a high-level data model, typically an ER diagram covering entities, relationships, and constraints — all independent of any specific DBMS. |
| Logical design | Translate that conceptual model into an actual relational schema — tables, attributes, keys, and integrity constraints — producing the logical structure that gets implemented. |
| Schema refinement | Apply functional dependencies, multivalued dependencies, and normal forms to strip out redundancy and anomalies. This is where normalisation does its work. |
| Physical design | Decide on indexes, storage structures, file organisation, and other performance details — this stage is specific to whichever DBMS you're using. |
| Security, authorisation, and application design | Define access control, views, transactions, and the surrounding application logic. |

## Modelling temporal data

Traditional relational design generally only captures the current state of the world — but plenty of real applications actually need to track history and how long facts remained valid.

| Concept | What it captures |
|---------|----------------------|
| Valid time | The period during which a fact was true in the real world — like how long someone actually lived at a particular address. |
| Transaction time | The period during which a fact was recorded in the database itself — useful for auditing and reconstructing the database's own history. |
| Bitemporal data | Data that tracks both valid time and transaction time together. |
| Snapshot | The state of the data as it stood at one specific point in time. |

A common, practical way to handle this is adding start_time and end_time columns to a relation — something like EmployeeSalary(EmployeeID, Salary, StartDate, EndDate). For a record that's still currently valid, the end date might just be set far into the future, or treated as open-ended.

Temporal modelling earns its place when you need to track historical changes, audit compliance with legal or policy requirements, or answer questions like "what was true as of this particular date" rather than just "what's true right now."

That said, temporal data does complicate things — keys, constraints, and querying all get trickier once rows represent facts that hold over an interval of time rather than at a single instant. Designers have to think carefully about whether time belongs on attributes, on entities, on relationships, or in separate history tables altogether.
