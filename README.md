# DMSII → MariaDB

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

**Every table gets an `rsn`.** DMSII addresses a record by its record serial
number. A relational row needs a key of its own so that the sets reaching it
have something to point at.

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

AGPL-3.0-or-later — see [LICENSE](LICENSE) and [NOTICE](NOTICE).

A schema generated from your own DASDL is yours. The licence of this
repository places no condition on the generator's output, for the same reason
GNU Bison places none on the parsers it generates. A commercial licence is
available for the cases the AGPL does not fit.
