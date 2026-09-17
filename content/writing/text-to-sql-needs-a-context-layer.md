---
title: "Text-to-SQL needs a context layer, not a bigger model"
date: 2026-09-17
draft: true
summary: "Natural-language-to-SQL fails on real databases because column names do not say what the data is for. The fix is curated meaning, not a larger model."
tags: [sql, llm, rag, python]
---

Natural-language-to-SQL demos work on toy schemas and fall over on real ones.
A model can read a column called `status_flag` and has no idea it means a
supplier has been suspended for non-payment. Column names carry technical
meaning. The business meaning lives in people's heads and in a scattering of
queries someone wrote three years ago.

Feeding the model a bigger schema dump does not fix this. It needs to know what
the objects are *for*, and what good queries against them look like.

## The design

A curated context store, kept separate from the database it describes.

**Schema objects** (tables, views, procedures) carry a human-written display
name, business description, purpose and keywords, alongside auto-extracted
column detail. **Code snippets** carry a description, use cases, parameters and
the SQL itself. **A link table** records which snippets touch which objects.

Retrieval then pulls both halves: the descriptions of the tables the question
seems to be about, and proven queries against those tables. Generation happens
inside that context. A single-admin web interface curates it, a command-line
tool automates it, and the database, model and vector store are all pluggable.

## Decisions

**Business context is first-class, human-editable data.** Not inferred from
column names, not regenerated on every run. The person who knows what a table
means writes it down once, and every later query benefits.

**Snippets link to objects.** So retrieval hands the model a working query as
well as a description. The pairing matters more than either half alone: a
description tells the model what a table is, a snippet shows it how the table
is actually joined and filtered in practice.

**Pluggable from the start.** The goal was a tool a team could point at their
own SQL Server with their own model, on their own infrastructure. Tying it to
one vendor would have defeated the point.

## Where it stands

Phase one, the metadata store and repository layer, is complete with tests and
a demo. The command-line tool was next.

It paused there, and the honest reason is worth stating. The initial framing
had been "write a better prompt". It became clear the prompt was never the
bottleneck; the missing information was, and no prompt could supply
information that did not exist anywhere the model could reach. Answering that
design question was the interesting part. The rest is build.

The code is at [github.com/conormc93/nlsql-assistant](https://github.com/conormc93/nlsql-assistant).
