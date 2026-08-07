---
name: comment-slop
description: "Find and remove AI-slop comments — ones that restate the code, describe another layer's behavior, or narrate planning that never shipped — while protecting the comments that carry real reasoning. Use when a reviewer calls a comment useless, unclear, or AI-written; when auditing or cleaning up comments and docstrings on a branch, diff, or PR; when comments should be reduced, simplified, or restyled to ASD-STE100 / simplified technical English; or before sending a change for review."
license: MIT
---

# Comment Slop

A comment earns its place only if it states a fact that is:

1. **True at this layer** — about the code it sits on, not a caller's or a callee's behavior.
2. **Not already visible** in the code in front of the reader.
3. **Not stated better nearby** — in the signature, in the module docstring, in a test name, or on the very next line.

Every slop comment fails one of those three clauses. That is the whole diagnosis. The rest of this skill is how to apply it, and — just as important — how to recognize the comments that pass, so a cleanup pass does not delete the reasoning that was worth keeping.

Slop is not a length problem. It is a content-selection problem, which is why shortening comments makes it worse rather than better (see *Trimming is not the fix*).

## When to Use

- A reviewer says a comment is useless, confusing, or "seems like an AI comment"
- Cleaning up comments and docstrings on a branch or PR before sending it for review
- A comment uses a term the reader has to go look up, or describes machinery that is not in the file
- Auditing agent-written or generated code for filler prose

## Failure modes

Each mode below is one clause of the test failing. The examples are real; the reviewer reactions are quotes from an actual review.

### 1. Wrong layer

The comment describes another module's semantics, written from the author's whole-call-graph vantage rather than the reader's.

```python
def retention_config_to_proto(user_parameters: list[UserParameter]) -> RetentionConfig:
    """Project a contract's retention user parameter into the RetentionConfig proto.

    No retention parameter means the contract has no override and uses the
    package defaults entirely.
    """
```

> *"the comment mentions a package and I don't see a package here"*

User parameters go in, a proto comes out. No package crosses this boundary. "Uses the package defaults" is the *resolver's* behavior, one layer up — true of the system, not of this function.

**Fix**: say what an empty result means *at this layer* ("the contract stores no override") and stop. Whoever consumes the empty proto decides what to do about it.

**Heuristic**: name every noun in the comment. If a noun never appears in the parameters, the return type, the body, or the things the body calls, the comment is describing somewhere else.

### 2. Redundant with the line next to it

```python
# The endpoint proto replaces the dataclass request. It also adds `@service_method`.
# TODO: add @service_method and switch to the endpoint proto.
```

> *"Seems like a useless AI comment?"*

The TODO already carries the fact, in the imperative, in the form someone can act on. The sentence above it is the same fact restated as description.

**Fix**: delete the prose, keep the TODO. When two adjacent comments say one thing, keep the one that is actionable.

### 3. Undefined term at the point of use

```python
# The caller resolves the current (unsealed) contract
```

> *"What does 'unsealed' mean?"*

"Sealed" was defined in the module docstring 200 lines above, and this comment uses the *negation* of that term, so the reader has to find the definition and then invert it.

The instinct is to define the term inline. Check first whether the term is needed at all. Here it was not: the "current contract" query already selects on the absence of a usage invoice, which *is* the definition of unsealed. The comment restated a filter the query itself expresses.

**Fix**: deleted, not defined.

**Rule**: before defining a confusing term, look at whether the code already encodes the concept. A term that needs a definition to earn its place usually did not earn its place.

### 4. Restates the signature

```python
def validate_retention_overrides(overrides: list[RetentionOverride]) -> None:
    """Validate sparse numeric contract retention overrides."""
```

The name says validate. The parameter says retention overrides. `-> None` says it either raises or does nothing. "Sparse" and "numeric" are properties of the type. Zero information.

**Fix**: a docstring on a function this well-named earns its place only by saying what the validation *rejects*, what it raises, or that it is the sole enforcement point for an invariant. If none of that is true, delete it.

### 5. Forward references and planning artifacts

Comments about work that does not exist yet ("this arrives when a sibling service needs to write retention"), phase and step labels (P1, M1, "Step 1b"), ticket IDs, links to internal planning documents.

The reader cannot verify any of it from the repository, and it goes stale the moment plans change. Commit messages and the ticket are the durable homes for intent; in code it reads as the author narrating their own roadmap.

**Fix**: delete. If the code genuinely depends on future work, that dependency belongs in a TODO that names the concrete thing to do here.

### 6. Prose that contradicts the code

```python
# A write here never reaches the live successor and it rewrites the sealed record.
raise ContractSealedError(...)
```

Stated as present-tense fact, but the code raises precisely to prevent it. The reader has to work out that the sentence describes the world where the guard is absent.

**Fix**: use the conditional. "A write here *would not* reach the live successor and *would* rewrite the sealed record." One word turns a contradiction into the reason the guard exists.

### 7. Narrating the code

```python
# Increment the retry count by 1
retries += 1
```

The line already says it. Reserve the comment for the fact the line cannot carry — why three retries, which upstream API returns spurious 500s.

## What to keep

A comment is load-bearing if deleting it leaves a reader with a question the code cannot answer. Keep these, and keep them even when they are wordy:

| Keep when the comment... | Test |
|---|---|
| explains **why** this approach and not the obvious one | Would a competent reader "simplify" the code away if the comment were gone? |
| names a race, a lock's purpose, or an ordering constraint | Does it say what breaks without the ordering? |
| states an invariant and what enforces it | Is the invariant impossible to see from one function? |
| carries context from an external system, spec, or requirement | Is that fact unavailable anywhere else in the repo? |
| records a deliberate trade-off | Does it name the cost that was accepted? |
| justifies a guard that looks removable | Would deleting the guard still pass the tests? |

Two that pass:

```python
# isdigit() alone also accepts digits int() cannot parse, such as superscripts,
# and this guard must return False rather than raise
```

```python
# Read the raw column: loading the invoice itself would cost a query on every
# sealed contract
```

Both state a fact that is invisible in the code and true at this layer. Neither is short.

**The asymmetry**: deleting a load-bearing comment is worse than leaving a bland one. A bland comment costs a reader two seconds; a deleted rationale costs the next person the bug the rationale was preventing. When genuinely torn, keep it and move on.

## Trimming is not the fix

Mechanically shortening comments makes slop *worse*.

A style pass — ASD-STE100 simplified English, "short sentences, one idea each, active voice", a general instruction to be concise — constrains sentence *structure*. It cannot do content *selection*. Applied to a comment that carried real reasoning, it strips the clause holding the *why* and leaves a bland declarative that reads exactly like AI filler.

A real case:

```python
# Before (never drew a review comment):
# A sibling service may only reach this package through its `__init__`, so this
# is the write's supported entry point.

# After a trimming pass (immediately flagged as AI slop):
# Callers outside the services tree use this method.
```

The specificity (which callers, through what) and the reason (the package boundary) both vanished. The trimmed version is shorter, grammatically simpler, and worthless.

The judgment a mechanical pass cannot make: **a comment that survives trimming with only obvious statements left should be deleted, not shortened.** Decide keep-or-delete first. Only then edit wording — to fix the layer, the mood, an undefined term, or to apply the style rules below.

## Style for what survives (ASD-STE100)

Style is the second pass, never the first. Content selection (keep / rewrite / delete) decides *which facts* a comment states; only then does style decide *how the sentences say them*. Run the two in the other order and you get the trimming failure above.

For the comments you rewrite — and for any comment text you write yourself — apply the ASD-STE100 habits that survive contact with code:

- **Short sentences, one idea each.** A comment holding two facts is two sentences.
- **Active voice, named actor.** "The resolver applies the defaults", not "defaults are applied" — the actor is often the load-bearing information (which layer does it: mode 1's question).
- **Present tense for what the code does; conditional for what a guard prevents** (mode 6).
- **One term, one meaning.** Reuse the exact identifier, or the term the module docstring defines; a synonym forces the reader to check whether it names the same thing (mode 3).
- **No filler openers or intensifiers.** "Note that", "simply", "it is important to" — delete the phrase, keep the sentence.

Kept comments stay untouched by default — restyling every keep is churn. Restyle a keep only when its wording breaks these rules badly enough to slow the reader (a buried actor, a 40-word sentence), and then without touching content.

The override rule: **a style edit may not delete a fact.** If a rule and a fact collide — the sentence runs long because the why-clause is long — the fact wins and the rule loses. ASD-STE100 constrains wording; the three-clause test alone decides content.

## Workflow

### 1. Scope to the diff

Slop arrives with new code, so audit the change rather than the repository. A repo-wide sweep produces mass edits nobody can review and touches comments whose context was never loaded.

```bash
base=main                                      # the branch the PR targets
git diff "$base"...HEAD --stat                 # pick the files in scope
git diff "$base"...HEAD -U15 -- path/to/file   # wide context for judgment
```

For an inventory of what arrived, list the added comment lines — then open each site rather than acting on this list:

```bash
git diff "$base"...HEAD -U0 | grep -E '^\+\s*(#|//|/\*|\*|"""|<!--)'
```

This pattern only matches lines that *start* with a comment marker, so it under-counts: trailing comments (`x += 1  # why`) and docstring body lines do not appear in it. Treat it as a starting list — the full-diff read in step 2 is what catches the rest.

### 2. Read the code the comment sits on

Every verdict needs the function's parameters, return type, and body on screen. A comment that looks redundant in a grep line often names a constraint that is real three lines down; a comment that reads fine in isolation is often describing another module entirely — mode 1 is invisible without the signature. Never judge a comment from a search hit alone.

### 3. Verify what the comment claims

A comment asserting a fact may simply be stale: the code moved and the sentence did not. Check the claim against the code before preserving it. "It explains why" does not make it true, and a wrong comment is worse than an empty one. A false claim is a rewrite or a delete, never a keep.

### 4. Give one verdict per comment

| Verdict | When |
|---|---|
| **keep** | Passes all three clauses. Leave it alone — rewording a good comment is churn that hides the real edits. |
| **rewrite** | The fact is real but stated at the wrong layer, in the wrong mood, with an undefined term, or wrapped in redundant restatement. Restate it at this layer, or *reduce* it — delete the clauses that fail the test, keep the ones that pass. Then apply the [style rules](#style-for-what-survives-asd-ste100). Reduction is selection by the test, not compression. |
| **delete** | Once the wrong-layer content and the already-visible content are removed, nothing is left. |

Expect the pass to be a net deletion. Adding a fresh explanatory paragraph where a bad one was removed is how the next reviewer arrives at the same complaint.

### 5. Watch the published-docs case

Docstrings that feed generated documentation (Sphinx, godoc, rustdoc, JSDoc) have a second audience that never sees the source. One that restates the signature still fails the test for a reader of the code, but deleting it can blank an entry in the published API docs. Rewrite these into what the function returns, rejects, or raises rather than removing them.

### 6. Commit the cleanup on its own

A docs-only commit diffs to nothing but comment lines and reviews in a minute. Mixed into behavior changes, the reviewer has to check every hunk for a hidden logic edit. Before committing, confirm the diff touches comment lines only — if a line of code moved, split it out.

Finally, re-read each touched hunk as someone who has not seen the rest of the branch. That is the check that catches the wrong-layer failures the author cannot see from inside their own context.
