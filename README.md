<div align="center">

# jooq-rules

**Use jOOQ as a type-safe SQL DSL, not as an ORM.**

A convention guide for Kotlin/Spring, written as defaults and exceptions rather than
prohibitions — every rule ships with its *why* and its *when not*.

[English](README.md) · [한국어](README.ko.md) · [日本語](README.ja.md)

</div>

> jOOQ is not a tool for hiding SQL. It is a tool for making SQL **verifiable at compile time**.
> That distinction is the root of every rule here.

**The guide itself ([`SKILL.md`](SKILL.md)) is written in Korean.** This README describes what is
in it. It works as a [Claude Code](https://claude.com/claude-code) skill, or as a document you
simply read.

## Where this comes from

These conventions were worked out on personal projects, not proven under production traffic at
scale. Read them as one developer's reasoning, not as field-tested standard practice.

The failure modes cited as justification — connection pool exhaustion, OFFSET scan growth, silent
data inconsistency — are general backend concerns rather than anything specific to jOOQ. What is
specific to jOOQ is where the library lets you avoid them, and that is what the document maps.

---

## What makes it different

jOOQ's own documentation explains the API well. What tends to be missing is **when to choose
what** — and that is where most of the friction actually is.

### Defaults and exceptions, not prohibitions

Most convention documents end at "don't do X". Then you meet the case where X is right, and you
either break the rule or work around it awkwardly.

Here, each rule sets a default and states the conditions under which that default is wrong.

```
selectFrom is not forbidden. Projection is better for most reads,
so projection is the default.

The test, in one line: "does this query read a column it never uses?"
If yes, project. If no, selectFrom is fine —
listing every column is just saying the same thing at greater length.
```

Plain-SQL expressions get the same treatment. Referring to a **schema identifier** by string and
writing an **expression** in plain SQL are different acts. The first is avoided; the second — for
things codegen cannot express, like `ts_rank` or JSON operators — is a legitimate escape hatch,
provided it uses bind variables and lives in one place.

### The cost of a trade-off is stated

Plenty of articles tell you to use cursor pagination. Few tell you what cursor pagination
takes away.

```
The cost of a cursor: sequential movement only.
Giving up "jump to page N" and "how many pages in total" is the trade-off you are making.

Infinite scroll or a load-more button never needed those, so a cursor fits.
An admin table with page numbers in it does need them, so OFFSET fits.
Check the UI requirement first, then choose.
```

`MULTISET` is handled the same way. One query with full type safety is a real advantage, but the
default stays `batch fetch` — switch **after** the round-trip cost has actually been measured.
"A single query looks impressive" is not a reason to adopt one.

---

## What it covers

`SKILL.md` holds the rules and the decision tests. The reasoning, code and procedures live in
`references/`, opened only when a judgement call needs them.

| File | Contents |
|---|---|
| **`SKILL.md`** | Rules, decision tests, checklist — the part a skill loads every time |
| `references/queries.md` | Version baseline and codegen config, projection, nulls, mapping, dynamic queries |
| `references/pagination.md` | Cursor design, tie-breakers, **HMAC signing**, COUNT |
| `references/writes.md` | N+1, **bulk writes**, transaction boundaries and the outbox pattern |
| `references/testing.md` | **What compile-time cannot catch**, and how to test for it |

Two chapters answer questions the rest of the document raises. Since the case for jOOQ rests on
compile-time verification, *what the compiler misses* is exactly the scope of the tests — join
cardinality, cursor boundaries, idempotency, query counts. And a document this careful about N+1 on
the read side owes the same care to writes, so bulk inserts get a chunk size derived from
PostgreSQL's 65535 bind-parameter ceiling rather than a round number.

---

## A few of the rules

**Projection is a contract before it is an optimisation.**
When the columns are written down, you can tell what a query depends on by reading it.

**Don't bolt a nullable aggregate field onto an entity.**
If a caller cannot tell whether that field was populated, **the type is lying**. Give aggregate and
join results their own read-only projection type.

**The adapter carries across what the database said. The domain interprets it.**
Filling a NULL with `?: ""` in the adapter collapses "hasn't arrived yet" and "failed" into one
value. A screen that has to distinguish them no longer can.

**Mappers should be private.**
Make one public and other adapters or use cases start converting records themselves. Then adding a
field means hunting for every place a conversion lives.

**The test for an abstraction: can you picture the SQL from reading the function?**
If not, it hides too much. Don't put an ORM back on top of jOOQ.

**External calls belong outside the transaction.**
When an external API slows down, a database connection is held for exactly that long, and the pool
eventually runs dry. Inside the transaction, record only the intent (outbox); a worker publishes it.

**Sign any cursor you hand out.**
Emit raw sort-key values and a client can tamper with them to walk into another user's range. Seal
the caller identity and the filter conditions inside the cursor, and check them on decode.

---

## Using it

### As a Claude Code skill

```bash
# Globally
mkdir -p ~/.claude/skills/jooq-rules
cp -r SKILL.md references ~/.claude/skills/jooq-rules/

# Or for one project
mkdir -p .claude/skills/jooq-rules
cp -r SKILL.md references .claude/skills/jooq-rules/
```

It attaches itself when you write or review jOOQ queries.

### As a document

Open `SKILL.md` and follow the links. The few lines of YAML at the top are skill metadata;
everything after that is ordinary Markdown.

> **`SKILL.md` and `references/` are the single source of truth.** The three READMEs describe what
> is in them and can fall behind; when they disagree, the skill document wins.

---

## Scope and limits

- Examples are **Kotlin + Spring Boot + PostgreSQL**. For Java or another database, only the syntax
  changes
- §6 assumes a hexagonal (ports and adapters) layout. Without one, that section will not fit
- `MULTISET` requires jOOQ 3.15+
- **If your team has a convention, the team's convention wins.** This is one person's set of rules,
  not a standard
- Drawn from personal projects rather than large-scale production operation — see
  [Where this comes from](#where-this-comes-from)

---

## License

MIT. See [LICENSE](LICENSE).
