---
title: "The model could read my schema. It had no idea what any of it meant."
date: 2026-09-17
draft: true
summary: "Natural-language-to-SQL worked on the demo database and failed on a real one, and the reason was not the model. It was that nobody had written down what the tables were for."
tags: [sql, llm, rag, python]
showtoc: true
---

The demo was convincing. Point a language model at a small database, ask it
"which customers ordered more than five times last month", and get back a
correct query in a second. I had the retrieval pipeline working, the model
answering, and a growing sense that this would be useful at work, where
analysts spend a lot of time asking engineers to write SQL for them.

Then I pointed it at a database that looked like a real one, and it fell over
in a way that was more interesting than a simple failure.

## The question it could not answer

The schema had a relationships table with a column called `status_flag`. In
the domain this table came from, that column meant something specific: a
supplier had been suspended, usually for non-payment, and their products should
no longer be sold. Anyone who had worked with the system for a week knew that.

I asked: "how many suppliers are currently suspended?"

```sql
-- what the model produced
SELECT COUNT(*)
FROM suppliers
WHERE status = 'suspended';
```

There is no `status` column on `suppliers`. There is no `suspended` value
anywhere. The model had done exactly what a reasonable person with no context
would do: guessed that suspension lives on the supplier and is spelled out as a
word. The real answer was on a different table, in a bit column, with a name
that says nothing about suspension:

```sql
-- what was actually needed
SELECT COUNT(DISTINCT supplier_id)
FROM supplier_relationships
WHERE status_flag = 0;
```

I tried the obvious things. A better prompt. The full schema dump, all
forty-odd tables with every column. Sample rows. Each one made the model more
confident and no more correct, because the information it needed was not in any
of them. Nowhere in the schema does it say that `status_flag = 0` means
suspended. That fact lived in people's heads and in a handful of queries
someone had written years earlier and saved in a wiki page nobody could find.

That was the moment the project changed shape. The bottleneck was not the model
and it was not the prompt. It was that the meaning of the data had never been
written down anywhere a machine could reach.

## Designing a place for meaning to live

If the problem is missing context, the fix is a place to put context. Not a
bigger prompt, which is assembled fresh each time and forgotten. A store,
curated once by the people who know, and consulted on every query.

I settled on three tables of my own, kept entirely separate from the database
they describe:

```sql
CREATE TABLE schema_objects (
    id                TEXT PRIMARY KEY,   -- "so_001"
    schema_name       TEXT,               -- "dbo"
    object_name       TEXT,               -- "supplier_relationships"
    object_type       TEXT,               -- table | view | stored_procedure
    display_name      TEXT,               -- "Supplier to retailer relationships"
    description       TEXT,               -- what it is, in business terms
    business_purpose  TEXT,               -- what problem it solves
    business_keywords TEXT,               -- "suspension, non-payment, hold"
    columns_info      TEXT,               -- auto-extracted technical detail
    created_at        TIMESTAMP,
    updated_at        TIMESTAMP
);

CREATE TABLE code_snippets (
    id           TEXT PRIMARY KEY,        -- "cs_001"
    name         TEXT,                    -- "Currently suspended suppliers"
    snippet_type TEXT,                    -- query | snippet | procedure
    description  TEXT,
    use_cases    TEXT,                    -- when you would reach for this
    sql_code     TEXT,                    -- the actual, known-good SQL
    parameters   TEXT,
    created_at   TIMESTAMP,
    updated_at   TIMESTAMP
);

CREATE TABLE code_schema_links (
    snippet_id       TEXT REFERENCES code_snippets(id),
    schema_object_id TEXT REFERENCES schema_objects(id),
    PRIMARY KEY (snippet_id, schema_object_id)
);
```

The first table is where a human writes down what a table *means*. The
`columns_info` field is filled automatically from the database, because the
technical shape is not the hard part. The description, purpose and keywords are
written by a person, because that is the part no machine can derive.

For the relationships table, the record looks like this:

```yaml
object_name:       supplier_relationships
display_name:      Supplier to retailer relationships
description: >
  One row per supplier-retailer pair. Carries the commercial state of the
  relationship, including whether the supplier is currently allowed to
  receive orders.
business_purpose: >
  Source of truth for whether a supplier's products should be live on a
  retailer's site.
business_keywords: suspension, suspended, on hold, non-payment, order accept
columns_info: >
  status_flag BIT -- 1 = accepting orders, 0 = suspended. This is the column
  people mean when they say "suspended". Nothing on the suppliers table
  records this.
```

That last line of `columns_info` is the whole project in one sentence. It is
the thing the model could never have guessed and the thing every new engineer
has to be told.

## Why snippets, and why the link table

The second table came from watching what actually helped. A description tells
the model what a table is. It does not tell the model how the table is joined,
filtered or aggregated in practice. For that, nothing beats a query that someone
has already run and trusts:

```yaml
name:        Currently suspended suppliers
description: Suppliers whose relationship with any retailer is on hold.
use_cases:   Weekly suspension report; checking before a reinstatement.
sql_code: |
  SELECT DISTINCT s.supplier_id, s.supplier_name
  FROM supplier_relationships r
  JOIN suppliers s ON s.supplier_id = r.supplier_id
  WHERE r.status_flag = 0;
```

The link table connects that snippet to `supplier_relationships` and
`suppliers`. When a question comes in about suspension, retrieval finds the
relationships table through its keywords, follows the links, and pulls the
snippet as well. The model then sees a description of the table *and* a working
query against it. Either alone was noticeably weaker. The pairing is what moved
the answers from plausible to right.

Ask the same question again with the context store in front of the model:

```sql
SELECT COUNT(DISTINCT r.supplier_id)
FROM supplier_relationships r
WHERE r.status_flag = 0;
```

Same question. Different answer. Nothing about the model changed.

## The decisions I would defend

**Context is first-class, human-editable data.** I considered having the model
infer descriptions from column names and sample rows, and regenerate them
periodically. It would have been faster to build. It would also have produced
confident descriptions of `status_flag` that were wrong in exactly the way the
original query was wrong. The person who knows what a table means writes it
down once, and every later question benefits.

**Snippets link to objects, not the other way round.** A query touches several
tables; a table is touched by many queries. The many-to-many link means
retrieval can start from either end. Ask about a table, get its proven
queries. Ask a question that matches a snippet, get the tables it depends on.

**Everything pluggable, from the first commit.** The database driver, the model,
the vector store. The goal was a tool a team could point at their own SQL Server
with their own model on their own infrastructure. Building it around one vendor
would have made it a demo again.

**SQLite for the context store.** The store is small, single-writer and needs
no infrastructure of its own. A metadata database that needs its own database
server would have been a barrier to anyone trying it.

## Where it stands

Phase one is the metadata store and the repository layer over it:

```text
src/core/models.py         SQLAlchemy models for the three tables
src/core/repositories.py   CRUD and keyword search
src/core/database.py       connection and migrations
scripts/demo_database.py   populates a sample context store
tests/                     16 passing
```

The command-line tool for curating context was next on a six-week plan. It
paused there, and the honest reason belongs in the write-up. The design
question I had set out to answer, whether the bottleneck was the model or the
missing meaning, had been answered by the failing query at the top of this
piece. The rest was build, and the day job's own SQL tooling took the time.

What I took from it applies well beyond this project. When a model gives a
confident wrong answer about your data, the first question is not "which model
would get this right". It is "where, anywhere, is the fact it needed written
down". Usually the answer is nowhere, and that is the thing to fix.

The code is at
[github.com/conormc93/nlsql-assistant](https://github.com/conormc93/nlsql-assistant).
