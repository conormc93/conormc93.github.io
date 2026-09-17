---
title: "I wrote a rule for my AI agent, then broke it twice in an afternoon"
date: 2026-09-17
draft: true
summary: "How a notes system for an AI coding agent turned into a set of scripts that enforce their own rules, after every written rule got broken and every check I wrote passed things it should have blocked."
tags: [ai-tooling, developer-experience, git, bash]
showtoc: true
---

The rule was simple and I wrote it myself: nothing that names my employer goes
into my personal GitHub repository. Hostnames, ticket prefixes, internal system
names, none of it. The repository holds the skills and instructions I use with
an AI coding agent, and I wanted it to survive a change of job, which means it
cannot carry the current one around inside it.

I broke that rule twice in one afternoon. The first time was two example
ticket keys in a skill description. The second was an entire document that
named the employer, the team and the security software on my laptop. Both
times I caught it only because I happened to run a search afterwards. Neither
time did anything stop me.

That afternoon is why the system I use now looks the way it does. It started as
a notes system. It ended as a set of scripts that check whether I am following
my own rules, because I had proven I would not.

## Starting point: an agent that forgets everything

An AI coding agent begins every session blank. Anything it learned about how I
work, what bit me last week, or which repository a service lives in is gone by
the next morning. The usual answer is a large instructions file that loads at
start-up. Mine reached sixty lines within a day and was heading for two hundred,
which is the point at which the agent starts ignoring parts of it.

So I split it. The part that is about *how to work with me* (write plainly,
disagree when you disagree, do not pipe a command whose exit code you are about
to judge) went into a file in git, versioned, portable across employers. The
part that is about *where I work* (team, forges, ticket conventions, what I am
measured on) went into a local file that never leaves the machine. The
instructions file the agent actually loads became two lines:

```markdown
@~/github/claude-skills/CLAUDE.portable.md

@~/.claude/CLAUDE.work.md
```

One copy of each thing. I mention this because my first attempt was a
*template* of the portable file for disaster recovery, and it drifted from the
live file within hours. I had added a rule about which shell to use and
forgotten the copy. The fix was not a sync script. It was deleting the copy and
importing the original. That pattern, one source imported rather than
duplicated, came up again and again.

## The vault, and the filter that makes it usable

For memory that has to outlive a session but not load into every one, I use an
Obsidian vault: a folder of Markdown files, searched on demand. The temptation
with a vault is to write everything down. I have watched that fail before. A
vault that captures everything becomes a diary, and nobody reads their diary.

So every note has to pass a filter before it is written:

```text
Write it down only if at least ONE is true:
- It cost more than ~30 minutes to work out
- It is a *why* not recoverable from the code or commit message
- It changed my mental model of the system
- A teammate would hit the same wall
- It is a decision where a real alternative was rejected
```

And every note is capped at fifteen lines. Anything longer belongs in an
article, which has its own shape. Most sessions produce nothing for the vault,
and the agent is told that writing nothing is usually the correct outcome.

The other rule that shaped the vault came from a search that silently found
nothing. I had written a note containing an error string and then searched for
it:

```bash
$ rg "/usr/bin/bash: line 12: no such file"
$ echo $?
1
```

No match. I concluded the note did not exist. It did. In Git Bash, a search
pattern that begins with `/` is rewritten into a Windows path before the native
`rg.exe` ever sees it, so the pattern that reached the binary was something like
`C:/Program Files/Git/usr/bin/bash...`. The search ran cleanly and matched
nothing, which is indistinguishable from the note never having been written.

```bash
$ grep -r "/usr/bin/bash: line 12" "$VAULT"
Obsidian Vault/20-Notes/2026-09-16-git-bash-is-the-shell.md
```

The `grep` that ships with Git for Windows is an MSYS program, and MSYS only
rewrites arguments when it launches a native Windows executable such as
`rg.exe`, so `grep` receives the pattern untouched. Error
messages are full of slashes, and error messages are exactly what I search the
vault for, so the retrieval commands in the vault's own schema now use `grep`,
with this reason written beside them.

## The guard that passed everything

Back to the rule I broke twice. The obvious fix is a pre-commit hook: scan what
is staged, refuse the commit if it contains a forbidden word. I wrote one. The
word list itself is employer-specific, so it lives in a gitignored file, and the
hook is fail-closed: no list, no commit.

The scanning line looked like this:

```bash
hits=$(git diff --cached --unified=0 | grep -E '^\+' | grep -v '^\+\+\+' \
        | grep -inE "$patterns" || true)
```

Take the added lines, drop the `+++ b/file` header, look for forbidden words.
I installed it, staged a file containing the employer's name, and committed.

The commit went through.

I stared at that for a while. The hook was executable. It had LF line endings.
It ran when I called it by hand and exited zero. Then I ran the pipeline one
stage at a time:

```bash
$ git diff --cached --unified=0 | grep -E '^\+'
+++ b/leak-test.md
+This mentions Acme and PROJ-1234.

$ git diff --cached --unified=0 | grep -E '^\+' | grep -v '^\+\+\+'
$
```

The second `grep` removed *both* lines. In basic regular expressions, which is
what `grep -v` uses without `-E`, `\+` is not a literal plus sign. It is the GNU
quantifier meaning *one or more*. So `'^\+\+\+'` did not mean "three pluses at
the start of the line". It meant "at least one plus at the start of the line",
which matched every added line in the diff. The hook scanned an empty string
and passed everything, while looking installed and working.

```bash
# what I meant
hits=$(git diff --cached --unified=0 \
        | awk '/^\+/ && !/^\+\+\+/' \
        | grep -inE "$patterns" || true)
```

The lesson I took from this is not about regular expressions. It is that I had
tested that the guard *ran*, and never tested that it *blocked*. A check that
silently passes everything is worse than no check, because you stop looking by
hand. Every guard I have written since gets a test where it must refuse
something:

```bash
$ echo "Internal note about Acme and ticket PROJ-9999." > leak-test.md
$ git add leak-test.md
$ git commit -m "this must be refused"
pre-commit: BLOCKED -- employer-specific detail in staged changes.
  In added lines:
    1:+Internal note about Acme and ticket PROJ-9999.
$ echo $?
1
```

## The reset that undid more than the commit

To test the guard I had made a throwaway commit, then undone it:

```bash
$ git reset --hard HEAD~1
```

That also reverted an uncommitted edit to `.gitignore`, the one that ignored the
pattern file. On the next `git add -A`, the pattern file, which contains every
word I was guarding against, got staged. The hook did not block it, because the
hook was reading its patterns *from* that file and the file was on the list of
things being committed. It was committed. It was not pushed, so it was
recoverable, but for about four minutes the repository's history contained the
complete list of things that must never appear in it.

Two things came out of that. `.gitattributes` now pins line endings for hooks
and shell scripts, because a hook whose shebang ends in `\r` fails silently and
that is the other way a guard ends up passing everything:

```gitattributes
hooks/** text eol=lf
*.sh text eol=lf
```

And the installer that wires the hooks into place refuses to run if any of them
has a carriage return in its first two hundred bytes.

There was a third mistake in the same hour, smaller but of the same family. I
had piped the commit command into `head` to trim its output:

```bash
$ git commit -m "..." 2>&1 | head -6
```

`head` closed the pipe after six lines, the hook was killed mid-write, and I got
a commit whose own hook output said it had been blocked. Never judge a command's
success through a pipe. The pipe returns the last command's exit code, not the
one you care about.

## Two risks that are not the same risk

Once the guard actually blocked things, it blocked too much. I wanted a
portable professional profile in the same repository, the kind of thing a CV
site could read. A CV has to name employers. The guard refused it.

The mistake was treating two different risks as one. An internal hostname must
never leave the building. The name of the company on my CV is public
information that I am about to publish myself. So the guard became two tiers:

```text
.leak-internal   hostnames, ticket keys, internal orgs and systems,
                 security tooling. BLOCKED EVERYWHERE. No exceptions.

.leak-employer   employer and team names.
                 Blocked everywhere EXCEPT paths listed in .leak-exempt.

.leak-exempt     profile/
```

Writing the profile immediately found an imprecision in my original word list.
The company's hostname contains the company's name in lower case, and I had
listed the bare word. That blocked the legal name of the company as if it were
the hostname. The internal list now matches the hostname *forms* specifically,
and the tiered guard was tested the same way as before: internal detail inside
the exempt folder must still be refused, and it is.

## Facts are generated; knowledge is earned

The last piece came from cloning fifty-five work repositories in one go. The
agent's rules said to offer a note for any repository without one. It offered
none. It had forgotten, fifty-five times, and I had not noticed.

The tempting fix was fifty-five stub notes. That would have buried the two
notes that contained real information under fifty-three that contained nothing.
Instead I separated two things that had been tangled together.

A **factual index** is generated. A script walks the checkouts and rewrites one
file: which host each repository is on, how many of my commits it has, when it
was last touched, what it is mostly written in. It runs at session start,
throttled to once a day because a full scan takes eighty seconds, and it is
never hand-edited.

```text
| Repo                        | Forge  | Mine | Last commit | Main types |
|-----------------------------|--------|------|-------------|------------|
| core/monolith               | GitLab |   73 | 2026-09-15  | java xml   |
| core/release-scripts        | GitLab |   47 | 2026-09-15  | sql txt    |
| web/storefront-frontend     | GitLab |   46 | 2026-07-06  | js jsx     |
```

A **knowledge note** is earned. It exists only when something non-obvious was
learned: a README that lies, a naming convention that breaks scripted lookups, a
risk worth stating. The index answers *where is this and do I work in it*. The
notes answer *what will bite me*.

And because the agent had proven it would forget to offer a note, a hook now
runs after every turn. If I have been working in a repository with no note, or
one whose note is older than my recent commits there, it injects a line telling
the agent so. Once per repository per day, so it prompts rather than nags.

## What it looks like now

Every session starts with three scripts. One pulls the latest versions of my
skills and the team's shared plugin, throttled to every twelve hours. One
regenerates the repository index, once a day. One validates the vault against
its own schema, every note's type and kind, the fifteen-line cap, the required
sections of an article, and prints only when something is wrong:

```text
vault-check: 3 problem(s) against SCHEMA.md:
  - bad-1.md: unknown kind 'rambling' (allowed: debug, alert, idea)
  - bad-1.md: body is 20 lines, cap is 15
  - bad-article.md: missing 'Decision log' section
```

Every commit to the skills repository runs the two-tier guard. Every commit to
the public site runs the internal tier of the same guard, and on the very first
commit it refused the CV page for naming an internal product. That is the
system working.

If I were starting again I would not begin with the notes. I would begin with
the check that tells me when I have broken my own rule. Everything else
followed from having to make each rule checkable, and I now believe that a rule
I cannot check is a rule I am going to break.
