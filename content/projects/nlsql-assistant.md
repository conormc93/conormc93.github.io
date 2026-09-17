---
title: "nlsql-assistant"
date: 2025-09-22
summary: "A natural-language-to-SQL assistant with a human-curated context layer, so the model knows what tables mean, not just what they are called."
tags: [python, sql, rag, llm]
weight: 1
---

**Repository:** [github.com/conormc93/nlsql-assistant](https://github.com/conormc93/nlsql-assistant)
· Python · SQL Server via ODBC · SQLite metadata store · Status: phase one complete, paused

## The problem it answers

Text-to-SQL demos work on toy schemas and fail on real ones. A model can read a
column called `order_accept_bit` and has no idea it means a supplier has been
suspended. Column names carry technical meaning; the business meaning lives in
people's heads and in queries someone wrote years ago.

A bigger schema dump does not fix that. The model needs to know what the
objects are *for*, and what good queries against them look like.

## What it does

A curated context store, kept separate from the database it describes:

- **Schema objects** carry a human-written name, description, purpose and
  keywords alongside auto-extracted column detail.
- **Code snippets** carry a description, use cases, parameters and the SQL.
- **A link table** records which snippets touch which objects.

Retrieval pulls both halves, the description of the relevant tables and proven
queries against them, and generation happens inside that context. A web UI
curates it, a CLI automates it, and the database, model and vector store are
all pluggable.

## Decisions worth noting

**Business context is first-class, human-editable data.** Not inferred, not
regenerated. The person who knows what a table means writes it down once.

**Snippets link to objects**, so the model gets a working query as well as a
description. That pairing matters more than either half alone.

**Pluggable from the start.** The goal was a tool a team could point at their
own database with their own model, not a hosted product.

## Where it stopped, and why

Phase one, the metadata store and repository layer, is complete with 16 passing
tests and a demo. The CLI was next on the plan. It paused there: the design
question it set out to answer, that the bottleneck is missing context rather
than model size, was the interesting part, and the day job's own SQL tooling
took the time.
