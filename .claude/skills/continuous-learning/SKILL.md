---
name: continuous-learning
description: Mine coding sessions for reusable patterns — capture corrections, score confidence, and feed the lessons back into memory
---

# Learning as you go

## Pulling patterns out of a session

When a meaningful session wraps, sort what you learned into three buckets:

1. **Corrections** — mistakes caught in review or by the user
2. **What worked** — approaches worth repeating
3. **What didn't** — approaches that caused trouble and should be avoided

## The shape of a learning entry

```yaml
pattern:
  id: "LEARN-2025-0042"
  category: "error-handling"
  type: "correction"         # correction | success | anti-pattern
  confidence: 0.85           # 0.0 to 1.0
  language: "typescript"
  context: "API error responses"
  observation: "Returning raw error messages from database exceptions exposes internals"
  lesson: "Always map database errors to application-level error codes before returning"
  example:
    before: "catch (e) { res.status(500).json({ error: e.message }) }"
    after: "catch (e) { logger.error(e); res.status(500).json({ error: 'INTERNAL_ERROR' }) }"
  frequency: 3               # times this pattern has been observed
  last_seen: "2025-06-15"
```

## Scoring confidence

| Score | Meaning | Action |
|-------|---------|--------|
| 0.95+ | Verified across multiple projects | Apply automatically |
| 0.80-0.94 | Confirmed in this codebase | Apply and mention |
| 0.60-0.79 | Observed but not fully validated | Suggest with caveat |
| 0.40-0.59 | Hypothesis based on limited data | Ask before applying |
| <0.40 | Speculative, needs validation | Document but do not apply |

Update confidence based on:
- +0.10 when pattern is confirmed correct by user
- +0.05 when pattern is observed again in a different context
- -0.15 when pattern leads to a correction
- -0.20 when pattern is explicitly rejected by user

## Wrapping up a session

At the close of a session, or just before context gets compacted:

1. **Review changes made** - Scan diffs for patterns
2. **Identify corrections** - What was changed after initial implementation?
3. **Note successful first-attempts** - What worked without revision?
4. **Record environment details** - Framework versions, config specifics
5. **Update confidence scores** - Adjust based on session outcomes
6. **Write to knowledge base** - Append new entries to CLAUDE.md or LEARNED.md

```markdown
## Session Learnings (2025-06-15)

### Corrections Applied
- [0.85] TypeScript: Use `satisfies` instead of `as` for type narrowing with object literals
- [0.90] Next.js: Server Actions must be async functions, even for synchronous operations

### Successful Patterns
- [0.80] PostgreSQL: Partial indexes on status columns reduced query time by 60%
- [0.75] React: Extracting data fetching into Server Components eliminated 3 useEffect hooks

### Anti-Patterns Identified
- [0.70] Avoid: Nesting more than 2 levels of Suspense boundaries (causes waterfall)
- [0.65] Avoid: Using `any` to suppress TypeScript errors in catch blocks (use `unknown`)
```

## Organizing the knowledge base

Split the knowledge base by domain:

```
knowledge/
  error-handling.md      # Error patterns across languages
  testing.md             # Test patterns and anti-patterns
  performance.md         # Optimization learnings
  api-design.md          # API design decisions
  deployment.md          # Infrastructure learnings
  project-specific.md    # Current project conventions
```

Every file uses the same entry format. When two entries share an `observation`, fold them together — bump `frequency` and update `confidence` rather than keeping duplicates.

## Tracking corrections

Whenever the user corrects the code or the approach:

1. Record what was originally produced
2. Record what the correction was
3. Identify the root cause (wrong assumption, missing context, outdated pattern)
4. Create or update a learning entry
5. Search for similar patterns that might need the same correction

```markdown
### Correction Log
- **Original**: Used `useEffect` to fetch data on mount
- **Correction**: Moved data fetching to Server Component
- **Root cause**: Applied client-side SPA pattern in Server Component context
- **Generalization**: In Next.js App Router, prefer server-side data fetching for initial page data
- **Confidence**: 0.90 (confirmed across 4 components)
```

## Reinforcing patterns

Keep a tally of how often a pattern gets applied and whether it holds up:

```
Pattern: "Use zod for API input validation"
  Applied: 12 times
  Confirmed: 11 times
  Corrected: 1 time (edge case with file uploads)
  Confidence: 0.92
  Status: ESTABLISHED
```

Statuses:
- **EMERGING** (frequency < 3) - New pattern, needs validation
- **GROWING** (frequency 3-7) - Building evidence, apply with mention
- **ESTABLISHED** (frequency 8+, confidence > 0.85) - Apply automatically
- **DEPRECATED** - Once valid, now superseded by a better approach

## Feeding it back into memory

Persist learnings in the project's memory file (CLAUDE.md or equivalent):

- High-confidence learnings (>0.85) go in the main instructions section
- Medium-confidence (0.60-0.84) go in a dedicated "Learnings" section
- Low-confidence (<0.60) stay in session notes until validated
- Deprecated patterns move to an archive section with reason for deprecation

Prune the knowledge base about once a month: drop anything untouched for 90 days that's still sitting below 0.70 confidence.
