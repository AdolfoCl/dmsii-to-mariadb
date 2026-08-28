# DMSII → SQL

A DMSII database describes itself in DASDL. This shows what comes out the other
side when you compile that description into a relational schema.

```
19,011 lines of a production DASDL description
263 data sets · 568 sets and subsets · 4,181 items
                        ↓  0.29 s
      780 tables · 69 indexes · 1,034 triggers
```

That run is on a real database taken off an MCP machine. It is not in this
repository — it belongs to its owner — but everything in `samples/` is, and
those come straight out of the Unisys manual, so you can read the input and the
output side by side and judge the translation yourself.

## What is here

| | Input | Model | Schema |
|---|---|---|---|
| | `samples/*.dasdl` | `model/*.model.json` | `schema/*.sql` |

Eight database descriptions from *Enterprise Database Server DASDL Programming
Reference Manual*, 8600 0213-424, and the MariaDB schema generated from each:

| Sample | What it exercises | Tables | Indexes | Views | Triggers |
|---|---|---:|---:|---:|---:|
| `1-personnel` | one data set, four items | 1 | – | – | – |
| `2-sets` | sets over a data set | 1 | 2 | – | – |
| `3-options` | database options | 2 | – | – | – |
| `4-physical` | physical specifications | 2 | 2 | – | – |
| `5-items` | every item type | 1 | – | – | – |
| `6-remaps` | remaps | 3 | – | 3 | – |
| `7-logical-database` | logical databases | 7 | 7 | 5 | – |
| `8-subsets` | subsets and automatic subsets | 11 | 2 | – | 10 |
| | | **28** | **13** | **8** | **10** |

## The target is a choice, not the design

The compiler builds a model of the database first, and emits from the model.
MariaDB is the emitter whose output is published here; the model itself carries
nothing MariaDB-specific, and the same structures reach SQL Server, PostgreSQL
or a non-relational target by writing another emitter, not by starting over.
The original of this work ran against **SQL Server**.

## The smallest one, end to end

`samples/1-personnel.dasdl` — Section 3, Example 1 of the manual:

```
PERSONNEL DATA SET
 (
   NAME        ALPHA(30);
   EMPLOYEE-NO NUMBER(6);
   DEPARTMENT  ALPHA(20);
   PHONE       NUMBER(10);
 );
```

`schema/1-personnel.sql`:

```sql
CREATE TABLE IF NOT EXISTS `personnel` (
  `rsn` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` CHAR(30),
  `employee_no` DECIMAL(6,0) UNSIGNED,
  `department` CHAR(20),
  `phone` DECIMAL(10,0) UNSIGNED,
  PRIMARY KEY (`rsn`)
) ENGINE=InnoDB;
```

## The decisions that matter

A schema like this is easy to get superficially right and quietly wrong. The
generated SQL carries its reasoning in comments; these are the choices behind it.

Most DMSII databases in service were not written by hand. They come out of EAE
(Enterprise Application Environment, formerly LINC) or ABSuite, and those emit
flat structures — tables, without the remaps and subsets DASDL allows. If yours
came from there, the translation is direct and most of what follows never
arises. The hand-written ones are where it does.

**Every table gets an `rsn`.** DMSII addresses a record by its physical
address — the `AAWORD`, which encodes the block and the offset within it. A
relational row needs a key that does not move when the record does, so the sets
reaching it have something to point at.

**A set becomes an index. A subset becomes a table.** A set spans its whole data
set, so an index expresses it exactly. A subset holds entries for only some of
the records, and an index cannot leave rows out — so it gets a table, and reading
whole records through it is a join on `rsn`. Reading only its keys and `DATA`
items — a `FIND KEY OF` — never leaves that table.

**Remaps and logical databases become views.** They are alternative readings of
the same records: renamed entries, hidden ones, regroupings, virtual items. A
view is what that is.

**Automatic subsets get triggers.** DMSII keeps them current as the data set
changes; the schema does the same, on the server, so nothing outside has to
remember. The generator can also leave that to the writer instead.

## The schema is the easy half

A schema is worth having only if something keeps it current, and dumping a
production DMSII database on a schedule does not scale — not at this size, and
not inside any window an operation will give you.

The answer is not to read the database. DMSII already writes down every create,
modify and delete, in the audit trail. Reading the changes as each audit block
is written keeps the SQL side current to the minute, with no downtime window, no
locks, and nothing touching the production database.

Nor is the schema the only thing generated. The chain that moves the data is
generated too — the COBOL programs on the MCP side, the WFL jobs that run them,
the Python that lands the rows in SQL. A new table in the DASDL means
regenerating, not writing code.

And the question that decides whether any of this is worth building is not
whether it can be built. It is who fixes it at three in the morning. The event
that breaks a hand-built extraction is a **reorganisation**: a structure changes
in the DASDL and the extractor is suddenly reading a layout that no longer
exists. The generated WFL carries exception handling for exactly that — a failed
program is detected and the chain rebuilt, and where the cause was a
reorganisation the table is reloaded in full and picks the audit trail back up
from there. No downtime window, nobody called out.

Detecting the reorganisation needs nothing to be instrumented and nobody to
raise a flag: watch the description file's `CREATIONDATE`, and a change means
somebody recompiled the DASDL. That alone decides nothing — a physical
specification may have moved and nothing that matters changed — so the new
description is compiled and the model compared against the previous one:
structure count and names, and within each structure every item, its type and
its `PICTURE`. The chain is regenerated only where something actually differs.

Which is where the compiler stops being a component of this and becomes the
thing the rest rests on. Without one, "detect a reorganisation" means a person
reads the new description and decides. With one, it is a comparison of two
models, and it runs at three in the morning without anybody.

That half is not in this repository either. It is the part worth talking about.

## What is not here

The DASDL compiler itself — the grammar, the model and the SQL generator — is
not in this repository. This is its output, published so the translation can be
read and argued with.

If you have a DMSII database you need in SQL and want to talk about it, that is
the point of this repo. Contact below.

## Author

Adolfo Díaz — Unisys ClearPath MCP: DMSII, DASDL, WFL, COBOL-74, ALGOL, and the
compilers for them.

## License

MIT — see [LICENSE](LICENSE) and [NOTICE](NOTICE).

What is here is output and documentation, and the licence is permissive so
that reading, quoting and copying it is unencumbered. The compiler that
produced these schemas is not in this repository and is licensed separately.

A schema generated from your own DASDL is yours. The compiler claims nothing
over its output, the same position GNU Bison takes on the parsers it generates.
