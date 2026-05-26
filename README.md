# DBMS_05 – From Schema to Data: DDL and DML in Practice

**Module:** Databases · THGA Bochum  
**Lecturer:** Stephan Bökelmann · <sboekelmann@ep1.rub.de>  
**Repository:** <https://github.com/MaxClerkwell/DBMS_05>  
**Prerequisites:** DBMS_01, DBMS_02, DBMS_03, DBMS_04, Lecture 05 (SQL I – DDL & DML)  
**Duration:** 90 minutes

---

## Learning Objectives

After completing this exercise you will be able to:

- Choose appropriate **SQL data types** for a given domain and justify the choice
- Write **`CREATE TABLE`** statements with column and table constraints
  (`PRIMARY KEY`, `FOREIGN KEY`, `NOT NULL`, `UNIQUE`, `CHECK`, `DEFAULT`)
- Declare **referential actions** (`ON DELETE`, `ON UPDATE`) and argue for the
  correct choice per relationship
- Use **`ALTER TABLE`** to add, drop, and modify columns and constraints
- Write **`INSERT`**, **`UPDATE`**, and **`DELETE`** statements — including
  multi-row inserts and updates with subqueries
- Protect destructive DML with **`BEGIN` / `ROLLBACK` / `COMMIT`** and explain
  why Autocommit is dangerous in practice

**After completing this exercise you should be able to answer the following questions independently:**

- Why is `NUMERIC(p,s)` mandatory for monetary values, while `REAL` is not?
- What is the difference between a column constraint and a table constraint —
  and when must a table constraint be used?
- Why does a `CHECK` constraint never reject a `NULL` value?
- What is the effect of a missing `WHERE` clause in `UPDATE` and `DELETE`?

---

## Check Prerequisites

```bash
sqlite3 --version
git --version
```

> You should see two version strings — SQLite 3.x and Git 2.x.
> If SQLite is missing:
>
> ```bash
> sudo apt-get install -y sqlite3   # Debian / Ubuntu
> brew install sqlite3              # macOS
> ```

> **Screenshot 1:** Take a screenshot of your terminal showing both
> successful version checks and insert it here.
>
> <img width="819" height="77" alt="Screenshot 1" src="https://github.com/user-attachments/assets/b520d2b8-86ea-4fd3-8307-e2a492d5b7af" />


---

## 0 – Fork and Clone the Repository

**Step 1 – Fork on GitHub:**  
Navigate to <https://github.com/MaxClerkwell/DBMS_05> and click **Fork**.
Keep the default settings and confirm.

**Step 2 – Clone your fork:**

```bash
git clone git@github.com:<your-username>/DBMS_05.git
cd DBMS_05
ls
```

> You should see only the `README.md`. You will create all further files
> yourself during this exercise.

---

## 1 – The Domain: A Municipal Library

A small municipal library manages its collection, members, and lending
transactions in a relational database. The library needs to track:

- **Books** — each identified by its ISBN, with a title, publication year,
  publisher, and a recommended lending price per day in euro.
- **Copies** — a book can exist in multiple physical copies, each stored at a
  specific shelf location.
- **Members** — registered with name, date of birth, e-mail address, and the
  date they joined the library.
- **Loans** — a member borrows a specific copy on a given date. The return date
  is recorded when the copy is handed back; until then it remains unknown.

The entity-relationship structure is deliberately given to you so that this
exercise can focus entirely on DDL and DML. Your task is to implement this
schema correctly in SQL.

### The Relations

| Relation    | Attributes (informal)                                                               | Primary Key           |
|-------------|-------------------------------------------------------------------------------------|-----------------------|
| `buch`      | isbn, titel, erscheinungsjahr, verlag, tagesgebuehr (in €)                         | isbn                  |
| `exemplar`  | exemplar_id, isbn (FK), standort                                                    | exemplar_id           |
| `mitglied`  | mitglied_id, nachname, vorname, geburtsdatum, email, beitritt_datum                 | mitglied_id           |
| `ausleihe`  | ausleihe_id, exemplar_id (FK), mitglied_id (FK), ausleihe_datum, rueckgabe_datum   | ausleihe_id           |

### Task 1 – Identify the Correct Data Types

For each attribute in the table above, choose the most appropriate SQL standard
data type and write it into the table below. Justify each choice in one sentence.
Use `NUMERIC(p,s)` for monetary values; choose the most precise date/time type
for each temporal attribute.

## Task 1 – Identify the Correct Data Types

For each attribute in the table above, choose the most appropriate SQL standard data type and justify each choice in one sentence.

| Attribute | Your Type | Justification |
|---|---|---|
| isbn | `TEXT` | An ISBN is an identifier. It can contain hyphens, so I would not store it as a number. |
| titel | `TEXT` | A book title is normal text. |
| erscheinungsjahr | `INTEGER` | The publication year is a whole number. |
| verlag | `TEXT` | The publisher name is text. |
| tagesgebuehr | `NUMERIC(6,2)` | This is a money value, so it should be stored with two decimal places. |
| exemplar_id | `INTEGER` | This is an ID for a physical copy of a book. |
| standort | `TEXT` | A location like `A-01-3` contains letters, numbers, and symbols. |
| mitglied_id | `INTEGER` | This is an ID for a library member. |
| nachname | `TEXT` | A last name is text. |
| vorname | `TEXT` | A first name is text. |
| geburtsdatum | `DATE` | A birth date is a date without time. |
| email | `TEXT` | An email address is text. |
| beitritt_datum | `DATE` | The joining date is a date. |
| ausleihe_id | `INTEGER` | This is an ID for a loan record. |
| ausleihe_datum | `DATE` | The loan date is a date. |
| rueckgabe_datum | `DATE` | The return date is a date, but it can be unknown at first. |

### Questions for Task 1

**Question 1.1:** `tagesgebuehr` could be stored as `REAL`. Give a concrete
example — using arithmetic — of why `REAL` would produce an incorrect result
for a lending fee calculation. Which type must be used instead?

> *Your answer:* I would not store `tagesgebuehr` as `REAL`, because `REAL` uses floating-point numbers. Floating-point numbers can have small rounding errors. For example, values like `0.10` are not always stored exactly inside the computer.

For money, this is a problem because even small differences are not good. A lending fee should be exact. That is why `NUMERIC(6,2)` is better here.



**Question 1.2:** `rueckgabe_datum` must be nullable. Explain what `NULL` means
in this specific context. Is `NULL` the same as "zero days"? Justify with
reference to the three-valued logic of SQL.

> *Your answer:*
>  In this database, `NULL` in `rueckgabe_datum` means that the book has not been returned yet. It does not mean “zero days”. If a book is borrowed and returned on the same day, then the return   date would be the same as the loan date. But if the value is `NULL`, then there is currently no   return date.
Because of SQL’s `NULL` logic, we should write:
In this database, `NULL` in `rueckgabe_datum` means that the book has not been returned yet.
It does not mean “zero days”. If a book is borrowed and returned on the same day, then the return date would be the same as the loan date. But if the value is `NULL`, then there is currently no return date.
Because of SQL’s `NULL` logic, we should write:
sql
rueckgabe_datum IS NULL
and not:
rueckgabe_datum = NULL
The second version is wrong because comparisons with NULL do not work like normal comparisons.

**Question 1.3:** `beitritt_datum` should default to today's date when no value
is provided. Write the `DEFAULT` expression you would use and explain why this
is preferable to always supplying the date explicitly in the application. 

> *Your answer:*For beitritt_datum, I would use:
DEFAULT CURRENT_DATE
This is useful because the database automatically inserts the current date when a new member is created. Then the application does not always have to send the date manually. It also avoids mistakes if different programs insert data into the database.


---

## 2 – DDL: Create the Schema

### Task 2a – Write schema.sql

```bash
vim schema.sql
```

Write `CREATE TABLE` statements for all four relations. Requirements:

- Every column must have an explicit type.
- Apply `NOT NULL` everywhere a value must always be present.
- `email` in `mitglied` must be unique across all members.
- `tagesgebuehr` must be greater than zero.
- `rueckgabe_datum`, if not `NULL`, must be greater than or equal to
  `ausleihe_datum` — express this as a table-level `CHECK` constraint.
- `beitritt_datum` must default to the current date.
- All foreign keys must declare `ON DELETE` and `ON UPDATE` actions:
  - Deleting a `buch` record must be refused as long as copies exist.
  - Deleting an `exemplar` record must be refused as long as active loans exist.
  - Deleting a `mitglied` record must be refused as long as loans exist.
  - Updating a primary key value must cascade to all dependent tables.
- Use SQLite types only (`INTEGER`, `TEXT`, `REAL`, `NUMERIC`, `DATE`).

> **Hint:** In SQLite, foreign key enforcement is off by default.
> Always run `PRAGMA foreign_keys = ON;` before your DDL and DML statements
> within the same session.

<details>
<summary>Solution skeleton — try it yourself first</summary>

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE buch (
    isbn              TEXT           PRIMARY KEY,
    titel             TEXT           NOT NULL,
    erscheinungsjahr  INTEGER        NOT NULL,
    verlag            TEXT           NOT NULL,
    tagesgebuehr      NUMERIC(6,2)   NOT NULL CHECK (tagesgebuehr > 0)
);

CREATE TABLE exemplar (
    exemplar_id  INTEGER  PRIMARY KEY,
    isbn         TEXT     NOT NULL,
    standort     TEXT     NOT NULL,
    FOREIGN KEY (isbn) REFERENCES buch(isbn)
        ON DELETE RESTRICT ON UPDATE CASCADE
);

CREATE TABLE mitglied (
    mitglied_id     INTEGER  GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nachname        TEXT     NOT NULL,
    vorname         TEXT     NOT NULL,
    geburtsdatum    DATE     NOT NULL,
    email           TEXT     NOT NULL UNIQUE,
    beitritt_datum  DATE     NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE ausleihe (
    ausleihe_id      INTEGER  PRIMARY KEY,
    exemplar_id      INTEGER  NOT NULL,
    mitglied_id      INTEGER  NOT NULL,
    ausleihe_datum   DATE     NOT NULL,
    rueckgabe_datum  DATE,
    FOREIGN KEY (exemplar_id) REFERENCES exemplar(exemplar_id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    FOREIGN KEY (mitglied_id) REFERENCES mitglied(mitglied_id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    CHECK (rueckgabe_datum IS NULL OR rueckgabe_datum >= ausleihe_datum)
);
```

> **Note:** SQLite does not implement `GENERATED ALWAYS AS IDENTITY`. Use
> `INTEGER PRIMARY KEY` instead — SQLite automatically assigns the next
> available integer when `NULL` is inserted into such a column. This is
> SQLite-specific behaviour; the standard syntax is shown in the lecture.

</details>

### Task 2b – Load the Schema and Verify

```bash
sqlite3 bibliothek.db < schema.sql
sqlite3 bibliothek.db ".tables"
sqlite3 bibliothek.db ".schema"
```

> Expected tables: `ausleihe  buch  exemplar  mitglied`

> **Screenshot 2:** Take a screenshot showing the `.tables` and `.schema`
> output in your terminal.
>
> <img width="561" height="762" alt="Screenshot 2" src="https://github.com/user-attachments/assets/f795abc1-f656-4b15-9fdd-2e466e3f557f" />


### Task 2c – Test Constraints

Without modifying `schema.sql`, run the following statements directly in
`sqlite3` (enable foreign keys first with `PRAGMA foreign_keys = ON;`) and
record what happens:

```sql
-- Test A: insert a book with a negative daily fee
INSERT INTO buch VALUES ('000-0-0000-0000-0', 'Fehlertest', 2024, 'Verlag X', -1.50);

-- Test B: insert a member without an e-mail
INSERT INTO mitglied (nachname, vorname, geburtsdatum)
VALUES ('Mustermann', 'Max', '2000-01-01');

-- Test C: insert a loan with a return date earlier than the loan date
INSERT INTO buch   VALUES ('978-3-16-148410-0', 'Testbuch', 2023, 'Verlag Y', 2.00);
INSERT INTO exemplar VALUES (1, '978-3-16-148410-0', 'Regal A1');
INSERT INTO mitglied (nachname, vorname, geburtsdatum, email)
VALUES ('Muster', 'Anna', '1990-05-20', 'anna@example.com');
INSERT INTO ausleihe VALUES (1, 1, 1, '2026-05-10', '2026-05-01');
```

> *Describe the error or result for each test:*
>
> - Test A: CHECK constraint failed: tagesgebuehr > 0 (19

> - Test B:NOT NULL constraint failed: mitglied.email (19)
> - Test C:CHECK constraint failed: rueckgabe_datum IS NULL 
        OR rueckgabe_datum >= ausleihe_datum (19)




### Questions for Task 2

**Question 2.1:** The `CHECK` on `rueckgabe_datum` was written as a table
constraint rather than a column constraint. Why is a column constraint
insufficient here?

> *Your answer:* The CHECK constraint for the return date has to compare two columns:
rueckgabe_datum >= ausleihe_datum
This rule depends on both rueckgabe_datum and ausleihe_datum. Because of that, it is better to write it as a table-level constraint, not only as a column constraint.


**Question 2.2:** You chose `ON DELETE RESTRICT` for all foreign keys.
Describe a realistic alternative: for which relationship would `ON DELETE
CASCADE` be appropriate instead, and why?

> *Your answer:* ON DELETE CASCADE could be useful if deleting a book should also automatically delete all its copies.
For example, if a library completely removes a book from its system, then all related exemplars could also be deleted automatically.
In this exercise, I think RESTRICT is safer. It prevents deleting a book if there are still exemplars or loans connected to it. This helps avoid losing important data by mistake.

**Question 2.3:** `email` is declared `UNIQUE`. According to the SQL standard,
how many `NULL` values may a `UNIQUE` column contain? Explain using the
three-valued logic of SQL.

> *Your answer:* A UNIQUE column can normally have more than one NULL value, because NULL means unknown. Two unknown values are not treated as equal.
However, in our table, email is also NOT NULL. That means every member must have an email address, and the email must also be unique.


---

## 3 – DML: Populate and Modify Data

### Task 3a – Write data.sql

```bash
vim data.sql
```

Populate the database with the following data. Insert them in the correct
dependency order (no foreign key violation).

**Books (`buch`):**

| isbn              | titel                          | erscheinungsjahr | verlag             | tagesgebuehr |
|-------------------|--------------------------------|------------------|--------------------|--------------|
| 978-3-423-08733-2 | Steppenwolf                    | 1927             | dtv                | 0.50         |
| 978-3-518-36893-4 | Homo Faber                     | 1957             | Suhrkamp           | 0.50         |
| 978-3-257-20456-6 | Der Vorleser                   | 1995             | Diogenes           | 0.75         |
| 978-3-596-18296-4 | Das Parfum                     | 1985             | Fischer            | 0.75         |
| 978-3-423-13571-9 | Die Verwandlung                | 1915             | dtv                | 0.30         |

**Copies (`exemplar`):**

| exemplar_id | isbn              | standort |
|-------------|-------------------|----------|
| 1           | 978-3-423-08733-2 | A-01-3   |
| 2           | 978-3-423-08733-2 | A-01-4   |
| 3           | 978-3-518-36893-4 | A-02-1   |
| 4           | 978-3-257-20456-6 | B-01-7   |
| 5           | 978-3-596-18296-4 | B-02-2   |
| 6           | 978-3-423-13571-9 | A-03-1   |

**Members (`mitglied`):**  
*(Omit `beitritt_datum` to test the DEFAULT; supply it explicitly for Klara
Sommer to simulate a historic membership date.)*

| nachname | vorname | geburtsdatum | email                      | beitritt_datum |
|----------|---------|--------------|----------------------------|----------------|
| Berger   | Jonas   | 2001-04-12   | jonas.berger@mail.de       | *(default)*    |
| Sommer   | Klara   | 1985-11-30   | klara.sommer@web.de        | 2019-03-15     |
| Hartmann | Lea     | 1998-07-08   | lea.hartmann@example.com   | *(default)*    |

**Loans (`ausleihe`):**

| ausleihe_id | exemplar_id | mitglied_id | ausleihe_datum | rueckgabe_datum |
|-------------|-------------|-------------|----------------|-----------------|
| 1           | 1           | 1           | 2026-05-01     | 2026-05-10      |
| 2           | 3           | 2           | 2026-05-05     | *(NULL)*        |
| 3           | 4           | 1           | 2026-05-12     | *(NULL)*        |
| 4           | 6           | 3           | 2026-04-20     | 2026-04-28      |

```bash
sqlite3 bibliothek.db < data.sql
```

Verify row counts:

```sql
SELECT 'buch',     COUNT(*) FROM buch
UNION ALL SELECT 'exemplar',  COUNT(*) FROM exemplar
UNION ALL SELECT 'mitglied',  COUNT(*) FROM mitglied
UNION ALL SELECT 'ausleihe',  COUNT(*) FROM ausleihe;
```

> Expected: 5, 6, 3, 4.

Commit:

```bash
git add schema.sql data.sql
git commit -m "feat: DDL and initial data for library database"
```

### Task 3b – UPDATE Statements

Write and execute the following updates. Save them in `updates.sql`.

1. The publisher `dtv` has changed its official name to `Deutscher Taschenbuch
   Verlag`. Update all affected rows with a single `UPDATE` statement.
2. Exemplar 3 (*Homo Faber*, currently on loan) has been returned today.
   Record today's date (`CURRENT_DATE`) as the return date for loan 2.
3. Raise the daily fee for all books published before 1960 by 10 cents.

For each update, first write it inside a `BEGIN` / `ROLLBACK` block and verify
the result with a `SELECT`. Then replace `ROLLBACK` with `COMMIT`.

```sql
BEGIN;
-- your UPDATE here
SELECT * FROM buch;  -- verify
ROLLBACK;            -- change to COMMIT after verification
```

### Task 3c – DELETE Statements

Write and execute the following deletions. Save them in `deletes.sql`.

1. Remove all loans where the return date is more than 30 days before today.
   Use `julianday(CURRENT_DATE) - julianday(rueckgabe_datum) > 30` as the
   condition.
2. Attempt to delete exemplar 3. Describe the error you expect and the
   referential integrity rule that causes it.
3. After successfully deleting all associated loans (they are all historic and
   have been returned), delete exemplar 3.

For each deletion, wrap it in `BEGIN` / `ROLLBACK`, verify, then `COMMIT`.

### Questions for Task 3

**Question 3.1:** The multi-table UPDATE in Task 3b.1 (renaming the publisher)
works because all affected rows are in the same table. Why can a standard SQL
`UPDATE` not update rows in two different tables simultaneously, and what would
you use instead in a production system?

> *Your answer:* A normal SQL UPDATE statement updates one table at a time. It cannot directly update two different tables in one simple UPDATE.
In a real system, I would use a transaction with multiple UPDATE statements. This way, all changes belong together. If one update fails, the whole transaction can be rolled back.


**Question 3.2:** Task 3b.3 raises the fee for books published before 1960
by 10 cents. Write the equivalent statement using `NUMERIC` arithmetic:
`tagesgebuehr = tagesgebuehr + 0.10`. Would the same statement work correctly
with `REAL`? Explain the risk. 

> *Your answer:* The equivalent update statement is:
UPDATE buch
SET tagesgebuehr = tagesgebuehr + 0.10
WHERE erscheinungsjahr < 1960;
This increases the daily fee by 10 cents for all books published before 1960.
With NUMERIC, this is okay because the fee is stored as a decimal money value. If we used REAL, there could be small rounding errors, which is not ideal for prices or fees

**Question 3.3:** Task 3c.1 deletes loans where the return date is more than
30 days ago. A `DELETE` without a `WHERE` clause would delete all loans.
Describe the operational consequence and explain how `BEGIN` / `ROLLBACK`
protects against this mistake.

> *Your answer:* A DELETE statement without a WHERE clause is dangerous because it deletes all rows from the table.
     For example:
        DELETE FROM ausleihe;
        This would delete the complete loan history.
        Using a transaction is safer:
        BEGIN;
        DELETE FROM ausleihe;
        ROLLBACK;
    With ROLLBACK, we can undo the deletion if we notice that it was a mistake. Only after checking the result should         we    use COMMIT.


---

## 4 – ALTER TABLE: Evolving the Schema

Over time, the library's requirements change. Perform the following schema
migrations. Save them in `migration.sql`.

### Task 4a – Add a Column

The library wants to record a phone number for each member (optional —
not every member provides one).

```sql
ALTER TABLE mitglied
    ADD COLUMN telefon TEXT;
```

Verify with `.schema mitglied` in `sqlite3`.

### Task 4b – Add a Named Constraint

The library decides that a book's publication year must be between 1450
(invention of the printing press) and the current year.

```sql
ALTER TABLE buch
    ADD CONSTRAINT buch_jahr_plausibel
    CHECK (erscheinungsjahr BETWEEN 1450 AND 2100);
```

> **Note:** SQLite does not support `ADD CONSTRAINT` for `CHECK` constraints
> via `ALTER TABLE`. In SQLite, to add a new constraint to an existing table
> you must: (1) create a new table with the constraint, (2) copy the data,
> (3) drop the old table, (4) rename. Document this limitation in a comment
> in `migration.sql` and write the four-step migration instead.
>
> In standard SQL (and in PostgreSQL, for example), `ADD CONSTRAINT` works
> directly.

### Task 4c – Change a Column Type

The library wants to increase the maximum length of `standort` (currently
unbounded `TEXT`) to enforce a maximum of 10 characters. In standard SQL:

```sql
ALTER TABLE exemplar
    ALTER COLUMN standort SET DATA TYPE VARCHAR(10);
```

> **Note:** This operation is also unsupported in SQLite. Document the
> limitation and describe the four-step workaround in a comment.
> Write the standard SQL statement as a comment above it.

### Questions for Task 4

**Question 4.1:** `ALTER TABLE mitglied ADD COLUMN telefon TEXT` adds a
nullable column. Why is this simpler than adding a `NOT NULL` column to an
already-populated table? What steps would be needed for a `NOT NULL` column?

> *Your answer:* Adding a nullable column is easy because old rows can simply have NULL in that new column.
Adding a NOT NULL column is more difficult because old rows would immediately need a value. Otherwise, they would violate the NOT NULL rule.
One possible solution is to add a default value:
ALTER TABLE mitglied
ADD COLUMN telefon TEXT NOT NULL DEFAULT 'unknown';
Another solution is to first add the column as nullable, fill in the values, and then recreate the table with the NOT NULL constraint.


**Question 4.2:** SQLite's limited `ALTER TABLE` support is a deliberate
design decision. What does this tell you about the trade-off between a
lightweight embedded database and a full-featured server database system?
Name one scenario where SQLite is the right choice and one where it is not.

> *Your answer:* SQLite is simple and lightweight. It is very good for small projects, local applications, teaching, and prototypes.
However, SQLite does not support all advanced database features. For example, some schema changes are harder than in systems like PostgreSQL.
For this library exercise, SQLite is a good choice because the database is small and easy to test locally. For a large real library system with many users at the same time, a server database like PostgreSQL would be better.


Commit:

```bash
git add migration.sql
git commit -m "feat: schema migration – telefon column and constraint notes"
```

---

## 5 – Transactions: Borrowing as an Atomic Operation

Lending a book copy to a member is not a single SQL statement — it is a
two-step operation:

1. Check that the copy is not currently on loan (no `ausleihe` row with
   `rueckgabe_datum IS NULL` for this `exemplar_id`).
2. Insert a new `ausleihe` row.

If step 2 fails (e.g. due to a constraint violation) after step 1 has
been checked, the database must remain consistent.

### Task 5a – Simulate a Safe Lending Transaction

Write a `BEGIN` / `COMMIT` block in `lend.sql` that lends exemplar 5
(*Das Parfum*) to member 3 (Lea Hartmann) starting today.

```sql
PRAGMA foreign_keys = ON;

BEGIN;

-- Step 1: verify the copy is available (no open loan)
SELECT COUNT(*) AS open_loans
FROM   ausleihe
WHERE  exemplar_id = 5
  AND  rueckgabe_datum IS NULL;

-- Step 2: insert the loan (only proceed if the count above is 0)
INSERT INTO ausleihe (ausleihe_id, exemplar_id, mitglied_id, ausleihe_datum)
VALUES (5, 5, 3, CURRENT_DATE);

COMMIT;
```

Verify:

```sql
SELECT * FROM ausleihe WHERE ausleihe_id = 5;
```

> **Screenshot 3:** Take a screenshot showing the inserted row.
>
> <img width="535" height="105" alt="Screenshot 3" src="https://github.com/user-attachments/assets/99e9fa34-2b38-491a-8dd3-5873e2cec772" />


### Task 5b – Simulate a Rollback

Now attempt to lend exemplar 3 (*Homo Faber*) to member 1, even though it
was returned in Task 3b (step 2). First re-open the loan artificially:

```sql
BEGIN;
UPDATE ausleihe SET rueckgabe_datum = NULL WHERE ausleihe_id = 2;
-- Now exemplar 3 appears to be on loan again.
-- The following INSERT would succeed (SQLite does not auto-prevent it):
INSERT INTO ausleihe (ausleihe_id, exemplar_id, mitglied_id, ausleihe_datum)
VALUES (6, 3, 1, CURRENT_DATE);
-- Having seen both statements, we decide to abort:
ROLLBACK;
```

Verify that neither change persisted:

```sql
SELECT rueckgabe_datum FROM ausleihe WHERE ausleihe_id = 2;
SELECT COUNT(*) FROM ausleihe WHERE ausleihe_id = 6;
```

> *Describe what you see and explain why `ROLLBACK` reversed both changes:*
> In Task 5b, I first started a transaction with `BEGIN`. Inside the transaction, I temporarily set the return date of  loan 2 back to `NULL`, so exemplar 3 looked like it was on loan again. Then I inserted a new loan with `ausleihe_id = 6`.

After that, I used `ROLLBACK` instead of `COMMIT`. Because both changes were inside the same transaction, SQLite reversed both of them. The return date of loan 2 stayed unchanged, and the new loan 6 was not saved.

The verification showed that loan 2 still has its return date and that the count for loan 6 is `0`. This proves that `ROLLBACK` undid all changes made in the transaction.

### Questions for Task 5

**Question 5.1:** In the lending scenario, why is it important that the
availability check and the insert happen inside the same transaction?
What could go wrong if they ran as separate Autocommit statements?

> *Your answer:* The availability check and the insert should be in the same transaction because lending a book is one complete operation.
First, we check if the exemplar is available. Then, we insert the new loan. If these two steps are separate, another user could borrow the same exemplar in between.
That could lead to two open loans for the same book copy. A transaction helps prevent this kind of problem.


**Question 5.2:** The lecture states: "Ein fehlendes `WHERE` aktualisiert
alle Zeilen." Write the single most dangerous `UPDATE` statement possible
on this database and explain the damage it would cause. Then explain how
`BEGIN` / `ROLLBACK` would allow you to recover.

> *Your answer:* A dangerous statement would be:
UPDATE ausleihe
SET rueckgabe_datum = CURRENT_DATE;
without a WHERE condition.
This would mark all loans as returned, even the ones that are still open. That would destroy important information about which books are currently borrowed.
Inside a transaction, the mistake can be undone:
BEGIN;


**Question 5.3:** Autocommit is convenient for read-only queries (`SELECT`).
Is it also safe for DML in an interactive session? Give a concrete example
from this exercise where Autocommit would have caused irreversible data loss.

> *Your answer:* Autocommit is useful for simple statements, but it can be dangerous for changes like UPDATE, DELETE, and INSERT.
For example:
DELETE FROM ausleihe;
If Autocommit is active, this deletion is saved immediately. If it was a mistake, the data is gone.
If we use a transaction, we can first check the result and then decide between COMMIT and ROLLBACK.


Commit:

```bash
git add lend.sql
git commit -m "feat: transaction examples for safe lending operations"
```

---

## 6 – Reflection

**Question A – Type discipline:**  
The lecture warns against using `TEXT` for everything. Looking at the
`buch` table: which column would be most tempting to store as `TEXT` when
it should be a more specific type, and what concrete query would break or
produce wrong results if the wrong type were used?

> *Your answer:* In the buch table, erscheinungsjahr should be stored as INTEGER, not as TEXT.
If the year was stored as text, comparisons could behave incorrectly. For example:
SELECT *
FROM buch
WHERE erscheinungsjahr < 1960;
This query should compare years as numbers. With INTEGER, the database knows that it is a numeric comparison.


**Question B – DDL as documentation:**  
A colleague reads your `schema.sql` and says: "Constraints slow down inserts
— I'd rather check these rules in the application." Give two concrete
reasons why enforcing constraints in the database is preferable to
enforcing them only in application code.

> *Your answer:* Database constraints are useful because they protect the data directly inside the database.
For example, the database can make sure that:
an email is not missing,
an email is unique,
a daily fee is positive,
a return date is not before the loan date.
This is better than only checking these rules in the application, because another program or user could also insert data into the database. The constraints are always active.
Also, the schema becomes easier to understand because the rules are written directly in the DDL.


**Question C – NULL semantics in lending:**  
In `ausleihe`, `rueckgabe_datum IS NULL` means "currently on loan". Could
this semantic be expressed without using `NULL` — e.g. by using a status
column instead? What are the trade-offs?

> *Your answer:* Yes, we could also use a status column, for example:
status TEXT CHECK (status IN ('open', 'returned'))
This would make the status very clear.
But it also creates possible redundancy. For example, the status could say returned, but rueckgabe_datum could still be NULL. Then the data would be inconsistent.
Using rueckgabe_datum IS NULL is simple because if there is no return date, the book is still borrowed.


**Question D – `TRUNCATE` vs. `DELETE`:**  
If you wanted to reset the entire database and reload the sample data from
scratch, you would need to empty all four tables. Can you use `TRUNCATE`
in SQLite? What alternative would you use, and in what order must the tables
be emptied to respect foreign key constraints?

> *Your answer:* SQLite does not support TRUNCATE.
Instead, we can use:
DELETE FROM table_name;
To avoid foreign key problems, we should delete the child tables first and then the parent tables:
DELETE FROM ausleihe;
DELETE FROM exemplar;
DELETE FROM mitglied;
DELETE FROM buch;
After that, we can reload the sample data with:
sqlite3 bibliothek.db < data.sql


> **Screenshot 4:** Take a screenshot showing the output of the row-count
> verification from Task 3a after completing all DML tasks, with
> `.headers on` and `.mode column` active.
>
> <img width="434" height="164" alt="Screenshot 4" src="https://github.com/user-attachments/assets/0138ccf3-1d82-4eff-b330-9228f069dd93" />


---

## Bonus Tasks

1. **`INSERT INTO … SELECT`:** The library acquires a second copy of every
   book that has been borrowed more than once. Write a single
   `INSERT INTO exemplar … SELECT` statement that inserts one additional
   row per qualifying book, with `standort = 'Neu-' || standort` of the
   existing copy.

2. **Overdue calculation:** Write a `SELECT` that lists all currently open
   loans (no return date), the member's full name, the book title, and the
   number of days the book has been borrowed (using `julianday(CURRENT_DATE)
   - julianday(ausleihe_datum)`). Sort by days descending.

3. **Lending fee invoice:** Write a `SELECT` that computes the total lending
   fee for each completed loan (return date not null):
   `(julianday(rueckgabe_datum) - julianday(ausleihe_datum)) * tagesgebuehr`.
   Join all necessary tables and show the member's name, book title, and
   amount due.

4. **GitHub Actions:** Add `.github/workflows/ci.yml` that installs SQLite,
   runs `schema.sql` and `data.sql` against a fresh database, and verifies
   the row counts with a shell assertion. Trigger the workflow by pushing
   any commit to `main`.

---

## Further Reading

- ISO/IEC 9075 (SQL Standard) — official reference; most universities have access
- [SQLite – Core Functions](https://www.sqlite.org/lang_corefunc.html)
- [SQLite – Date and Time Functions](https://www.sqlite.org/lang_datefunc.html)
- [SQLite – Foreign Key Support](https://www.sqlite.org/foreignkeys.html)
- [SQLite – ALTER TABLE Limitations](https://www.sqlite.org/lang_altertable.html)
- Lecture 05 handout – *SQL I: DDL & DML*
