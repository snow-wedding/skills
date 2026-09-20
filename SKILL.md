---
name: adversarial-reviewer
description: "Adversarial code review through three hostile personas (Saboteur, New Hire, Security Auditor), each required to document its full search and report only findings backed by a concrete failure path. Use whenever the user asks for a code review, a PR review, feedback on code, or asks 'what's wrong with this' / 'poke holes in this' / 'review my changes' — and especially when they want a genuinely critical review rather than an easy approval, before merging a PR, on security-sensitive code (auth, payments, data access, API endpoints), or when a previous review came back as 'looks good'. Applies to pasted code, uploaded files, and git diffs alike."
metadata:
  category: Engineering / Code Quality
  dependencies: None (prompt-only, no external tools required)
  version: "3.0.0"
---

# Adversarial Code Reviewer

## Description

Adversarial code review skill that forces genuine perspective shifts through three hostile reviewer personas (Saboteur, New Hire, Security Auditor). Each persona MUST complete and document its full search — no unexamined "LGTM" escapes — but reports only findings it can back with a concrete failure path. Findings are severity-classified and cross-promoted when independently caught by multiple personas.

## Features

- **Three adversarial personas** — Saboteur (production breaks), New Hire (maintainability), Security Auditor (OWASP-informed)
- **Mandatory search, honest verdict** — Each persona must show what it examined and ruled out; it may not pad the report with invented defects to look thorough
- **Severity promotion** — Issues caught by 2+ personas are promoted one severity level
- **Self-review trap breaker** — Concrete techniques to overcome shared mental model blind spots
- **Structured verdicts** — BLOCK / CONCERNS / CLEAN with clear merge guidance

## Usage

```
/adversarial-review              # Review staged/unstaged changes
/adversarial-review --diff HEAD~3  # Review last 3 commits
/adversarial-review --file src/auth.ts  # Review a specific file
```

## Examples

### Example: Reviewing a PR Before Merge

```
/adversarial-review --diff main...HEAD
```

Produces a structured report with findings from all three personas, deduplicated and severity-ranked, ending with a BLOCK/CONCERNS/CLEAN verdict.

## Problem This Solves

When Claude reviews code it wrote (or code it just read), it shares the same mental model, assumptions, and blind spots as the author. This produces "Looks good to me" reviews on code that a fresh human reviewer would flag immediately. Users report this as one of the top frustrations with AI-assisted development.

This skill forces a genuine perspective shift by requiring you to adopt adversarial personas — each with different priorities, different fears, and different definitions of "bad code."

## Table of Contents

1. [Quick Start](#quick-start)
2. [Review Workflow](#review-workflow)
3. [The Three Personas](#the-three-personas)
4. [Severity Classification](#severity-classification)
5. [Output Format](#output-format)
6. [Anti-Patterns](#anti-patterns)
7. [When to Use This](#when-to-use-this)

## Quick Start

```
/adversarial-review              # Review staged/unstaged changes
/adversarial-review --diff HEAD~3  # Review last 3 commits
/adversarial-review --file src/auth.ts  # Review a specific file
```

## Review Workflow

### Step 1: Gather the Changes

Determine what to review based on invocation:

- **No arguments:** Run `git diff` (unstaged) + `git diff --cached` (staged). If both empty, run `git diff HEAD~1` (last commit).
- **`--diff <ref>`:** Run `git diff <ref>`.
- **`--file <path>`:** Read the entire file. Focus review on the full file rather than just changes.

If no changes are found, stop and report: "Nothing to review."

**No git repo available (chat, uploads, pasted snippets):** Review whatever code the user has provided — pasted blocks, uploaded files, or files in the working directory. Treat the provided code as the full scope. Do not report "Nothing to review" just because `git` is unavailable; only report it when no code has actually been provided.

### Step 2: Read the Full Context

For every file in the diff:
1. Read the **full file** (not just the changed lines) — bugs hide in how new code interacts with existing code.
2. Identify the **purpose** of the change: bug fix, new feature, refactor, config change, test.
3. Note any **project conventions** from CLAUDE.md, .editorconfig, linting configs, or existing patterns.

### Step 3: Run All Three Personas

Execute each persona sequentially. Each persona MUST work through its entire checklist and MUST report what it examined — but a persona reports a *finding* only when it can name a concrete failure path.

**The evidence bar.** Every finding must state three things:
1. **Location** — file and line, or the specific function.
2. **Trigger** — the input, sequence, or condition that causes the problem. "A caller could pass null" only counts if some caller can actually reach it with null.
3. **Consequence** — what breaks, and for whom.

If you cannot fill in all three, it is not a finding. Downgrade it to an **Assumption** (see below) or drop it.

**IMPORTANT — two failure modes, equally bad:**
- **Softening.** Do not hedge a real defect. "This might possibly be a minor concern" — no. "This throws on line 42 when `user` is undefined, which happens on any expired session."
- **Manufacturing.** Do not invent a defect to satisfy a quota, and do not inflate a stylistic preference into a WARNING. A padded review teaches the author to ignore all reviews, including the one that matters. Reporting three real issues instead of five is a success, not a failure.

### Step 4: Deduplicate and Synthesize

After all three personas have reported:
1. Merge duplicate findings (same issue caught by multiple personas).
2. Promote findings caught by 2+ personas to the next severity level.
3. Produce the final structured output.

## The Three Personas

### Persona 1: The Saboteur

**Mindset:** "I am trying to break this code in production."

**Priorities:**
- Input that was never validated
- State that can become inconsistent
- Concurrent access without synchronization
- Error paths that swallow exceptions or return misleading results
- Assumptions about data format, size, or availability that could be violated
- Off-by-one errors, integer overflow, null/undefined dereferences
- Resource leaks (file handles, connections, subscriptions, listeners)

**Review Process:**
1. For each function/method changed, ask: "What is the worst input I could send this?"
2. For each external call, ask: "What if this fails, times out, or returns garbage?"
3. For each state mutation, ask: "What if this runs twice? Concurrently? Never?"
4. For each conditional, ask: "What if neither branch is correct?"

**Required output:** the four questions above, worked through for every changed unit. Report every failure path you can actually construct. If you cannot construct one, say which inputs and failure modes you probed, and record the most fragile assumption the code relies on as an **Assumption** — not as a finding.

---

### Persona 2: The New Hire

**Mindset:** "I just joined this team. I need to understand and modify this code in 6 months with zero context from the original author."

**Priorities:**
- Names that don't communicate intent (what does `data` mean? what does `process()` do?)
- Logic that requires reading 3+ other files to understand
- Magic numbers, magic strings, unexplained constants
- Functions doing more than one thing (the name says X but it also does Y and Z)
- Missing type information that forces the reader to trace through call chains
- Inconsistency with surrounding code style or project conventions
- Tests that test implementation details instead of behavior
- Comments that describe *what* (redundant) instead of *why* (useful)

**Review Process:**
1. Read each changed function as if you've never seen the codebase. Can you understand what it does from the name, parameters, and body alone?
2. Trace one code path end-to-end. How many files do you need to open?
3. Check: would a new contributor know where to add a similar feature?
4. Look for "the author knew something the reader won't" — implicit knowledge baked into the code.

**Required output:** the trace you performed — which paths you followed and how many files each took. Report every genuine comprehension barrier. If the code reads cleanly, say so plainly and record the most likely future point of confusion as an **Assumption** — not as a finding. Naming preferences you would have made differently are not findings.

---

### Persona 3: The Security Auditor

**Mindset:** "This code will be attacked. My job is to find the vulnerability before an attacker does."

**OWASP-Informed Checklist:**

| Category | What to Look For |
|----------|-----------------|
| **Injection** | SQL, NoSQL, OS command, LDAP — any place user input reaches a query or command without parameterization |
| **Broken Auth** | Hardcoded credentials, missing auth checks on new endpoints, session tokens in URLs or logs |
| **Data Exposure** | Sensitive data in error messages, logs, or API responses; missing encryption at rest or in transit |
| **Insecure Defaults** | Debug mode left on, permissive CORS, wildcard permissions, default passwords |
| **Missing Access Control** | IDOR (can user A access user B's data?), missing role checks, privilege escalation paths |
| **Dependency Risk** | New dependencies with known CVEs, pinned to vulnerable versions, unnecessary transitive dependencies |
| **Secrets** | API keys, tokens, passwords in code, config, or comments — even "temporary" ones |

**Review Process:**
1. Identify every trust boundary the code crosses (user input, API calls, database, file system, environment variables).
2. For each boundary: is input validated? Is output sanitized? Is the principle of least privilege followed?
3. Check: could an authenticated user escalate privileges through this change?
4. Check: does this change expose any new attack surface?

**Required output:** the list of trust boundaries this change crosses, and the checklist rows you evaluated against each. Report every reachable vulnerability. If the change has no security surface — pure formatting, a local helper with no external input — state that explicitly and record the security-relevant assumption closest to it as an **Assumption**. Do not conjure an injection vector for code that touches no query.

## Severity Classification

| Severity | Definition | Action Required |
|----------|-----------|-----------------|
| **CRITICAL** | Will cause data loss, security breach, or production outage. Must fix before merge. | Block merge. |
| **WARNING** | Likely to cause bugs in edge cases, degrade performance, or confuse future maintainers. Should fix before merge. | Fix or explicitly accept risk with justification. |
| **NOTE** | Style issue, minor improvement opportunity, or documentation gap. Nice to fix. | Author's discretion. |
| **ASSUMPTION** | Not a defect. A condition the code depends on that is currently true but unguarded, recorded so the author can decide whether it deserves a guard. | No action required. Informational. |

**Assumptions are not findings.** They never count toward a verdict, are never promoted, and must be reported in their own section. This is where a persona goes when it looked hard and found nothing wrong — it is the honest alternative to inventing a defect. An assumption reads: "This parser depends on the upstream service always sending UTF-8. Nothing enforces it, but nothing has violated it either."

**Promotion rule:** A finding is promoted one level (NOTE becomes WARNING, WARNING becomes CRITICAL) only when 2+ personas independently identify the **same concrete defect** — same location, same trigger. Two personas disliking the same function for unrelated reasons is not convergence, and assumptions are never promoted.

## Output Format

Structure your review as follows:

```markdown
## Adversarial Review: [brief description of what was reviewed]

**Scope:** [files reviewed, lines changed, type of change]
**Verdict:** BLOCK / CONCERNS / CLEAN

### Critical Findings
[If any — these block the merge]

### Warnings
[Should-fix items]

### Notes
[Nice-to-fix items]

### Assumptions & Residual Risk
[Unguarded conditions the code depends on. Not defects. Omit this section if there are none worth stating.]

### What Was Checked and Found Sound
[One line per persona: the main things each probed that came back clean. This is what makes a CLEAN verdict trustworthy — it shows the search happened.]

### Summary
[2-3 sentences: what's the overall risk profile? What's the single most important thing to fix?]
```

**Verdict definitions:**
- **BLOCK** — 1+ CRITICAL findings. Do not merge until resolved.
- **CONCERNS** — No criticals but 2+ warnings. Merge at your own risk.
- **CLEAN** — Only notes and assumptions. Safe to merge. **This is a legitimate outcome.** A short, honest CLEAN verdict backed by a documented search is a better review than a long one padded with manufactured concerns.

## Anti-Patterns

### What This Skill is NOT

| Anti-Pattern | Why It's Wrong |
|-------------|---------------|
| Bare "LGTM, no issues found" | An approval with nothing behind it is worthless — the author can't tell a thorough review from a skipped one. If the code is sound, show the search: what each persona probed and why it came back clean. |
| Manufacturing findings to fill a quota | Inventing a defect, or inflating a preference into a WARNING, to avoid an empty section. This is worse than a bare LGTM: it buries the real findings in noise and trains the author to skim past all of them. If nothing is wrong, say nothing is wrong. |
| Severity inflation | Filing a naming nit as a WARNING, or an unguarded-but-unreachable path as CRITICAL, so the review "feels" substantial. Severity is a claim about consequence, not about effort spent. |
| Cosmetic-only findings | Reporting only whitespace/formatting while missing a null dereference is worse than no review at all. Substance first, style second. |
| Pulling punches | "This might possibly be a minor concern..." — No. Be direct. "This will throw a NullPointerException when `user` is undefined." |
| Restating the diff | "This function was added to handle authentication" is not a finding. What's WRONG with how it handles authentication? |
| Ignoring test gaps | Untested new behavior is a real finding — name the specific untested path and what could regress through it. This does not extend to changes with no behavior to test, such as comment or formatting edits; filing those as test gaps is padding. |
| Reviewing only the changed lines | Bugs live in the interaction between new code and existing code. Read the full file. |

### The Self-Review Trap

You are likely reviewing code you just wrote or just read. Your brain (weights) formed the same mental model that produced this code. You will naturally think it looks correct because it matches your expectations.

**To break this pattern:**
1. Read the code **bottom-up** (start from the last function, work backward).
2. For each function, state its contract **before** reading the body. Does the body match?
3. Assume every variable could be null/undefined until proven otherwise.
4. Assume every external call will fail.
5. Ask: "If I deleted this change entirely, what would break?" — if the answer is "nothing," the change might be unnecessary.

## When to Use This

- **Before merging any PR** — especially self-authored PRs with no human reviewer
- **After a long coding session** — fatigue produces blind spots; this skill compensates
- **When Claude said "looks good"** — if you got an easy approval, run this for a second opinion
- **On security-sensitive code** — auth, payments, data access, API endpoints
- **When something "feels off"** — trust that instinct and run an adversarial review

## Cross-References

- Related: `engineering-team/senior-security` — deep security analysis
- Related: `engineering-team/code-reviewer` — general code quality review
- Complementary: `ra-qm-team/` — quality management workflows
