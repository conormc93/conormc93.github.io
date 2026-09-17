---
title: "Making an AI coding agent remember, and keep its own rules"
date: 2026-09-17
draft: true
summary: "An instruction layer, a memory bank with a hard filter, and scripts that enforce the rules. Built because a written rule got broken twice in one afternoon."
tags: [ai-tooling, developer-experience, git, bash]
---

An AI coding agent starts every session blank. Anything it learned about how I
work, what bit me last week, or which repository a service lives in is gone.
The usual fix is a large instructions file, which grows until the agent stops
following it, and a notes folder, which becomes a diary nobody rereads.

Underneath that sits a harder problem. **Rules that are only written down get
broken.** Over three days I wrote a rule that no employer detail may enter my
personal repository, then broke it twice, and only caught it because I happened
to search afterwards.

This is what I built instead, and what failed on the way.

## Three layers with different lifetimes

**Instructions**, loaded every session, kept short. Split into a portable file
(how to work with me, judgment rules, how to verify your own work) that lives
in git, and a local file (employer, level, local conventions) that never does.
The live instructions file is two import lines. One copy of each thing, so
nothing can drift.

**A vault**, searched on demand and never loaded wholesale. Four note types
with a schema saying what each is for. Notes are capped at fifteen lines and
must pass an inclusion filter: did this cost more than thirty minutes, is it a
*why* the code cannot tell you, would a colleague hit the same wall. Most
sessions produce nothing, and that is correct. A vault that captures everything
is a diary.

**Checks**, which is where it stopped being a notes system. A script validates
the vault against its own schema at every session start. A pre-commit hook
blocks employer detail in two tiers: hostnames and ticket keys are blocked
everywhere, employer *names* are blocked everywhere except a profile folder,
because a CV has to name employers. A post-turn hook notices when I have been
working in a repository that has no note and says so.

## Decisions worth stealing

**Import rather than copy.** I built a template of the instructions file for
disaster recovery. It drifted from the live file within hours. The fix was not
a sync script. It was deleting the copy and importing the original.

**Every rule that can be a check becomes one, in the same sitting.** The leak
guard exists because the written rule failed twice in one afternoon. If a
script could verify a rule and I did not write the script, I was choosing to
rely on memory, and memory had already lost.

**Facts are generated; knowledge is earned.** After cloning fifty-five
repositories the temptation was fifty-five stub notes. Instead a script
regenerates a factual index (which host, how many of my commits, last
activity), and a knowledge note is written only when something non-obvious was
learned. The index answers "where is this and do I work in it". The notes
answer "what will bite me".

**Redact by omission.** Publishing a private article publicly means rewriting
it from a blank page and carrying across only what I consciously choose. A
denylist only removes what someone thought to list, and a private document is
full of things nobody thought to list.

## What failed, which is the useful part

**The guard passed everything while looking installed.** Its first version
used `grep -v '^\+\+\+'` to drop the diff header. In basic regular expressions
`\+` means *one or more*, so it stripped every added line and every commit
sailed through. I found it only because I tested that the guard *blocked*, not
merely that it ran. A check that silently passes is worse than no check,
because you stop looking by hand.

**`git reset --hard` undid more than the commit.** Undoing a test commit also
reverted an uncommitted `.gitignore` edit, so the pattern list containing every
word I was guarding against got staged and committed. It was unpushed, so
recoverable. Now `.gitattributes` pins line endings and the installer refuses
to wire a hook with CRLF endings, because a hook whose shebang ends in `\r`
fails silently.

**Piping through `head` masked the result.** Running `git commit | head -6`
closed the pipe early, killed the hook mid-write, and produced a commit that
the hook's own output said was blocked. Never judge a command's success through
a pipe.

**`rg` finds nothing for a slash-prefixed pattern in Git Bash.** The shell
rewrites `/usr/bin` into a Windows path before the native binary sees it, so
the search matches nothing and exits clean. Error messages are full of slashes.
The vault's retrieval commands now use `grep -r`, and the reason is written
next to them.

## What I would tell someone starting this

Do not begin with the notes. Begin with the check that tells you when you have
broken your own rule. Everything else follows from having to make rules
checkable, and a rule you cannot check is a rule you will break.
