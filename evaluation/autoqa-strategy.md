# AutoQA Prompt Strategy Guide v2

## Lessons Learned from 55 Submissions

---

## AutoQA Version History

| Era | Format | Notes |
|-----|--------|-------|
| V1 (Jan 22-25) | X/13 | 6 prompting + 3 rationale + 4 diff |
| V2 (Jan 26-Feb 4) | X/Y/Z (pass/fail/borderline) | Fewer visible, borderline added |
| V3 (Feb 5-14) | X/7 | Consolidated |
| **V4 (Feb 15+)** | **X/8+** | **Added `not_substantiating_grading_axes`** |

### V4 New Check: `not_substantiating_grading_axes`

Your rationales must now explicitly connect observations to the 7 grading axes.

**Bad:** "Model A has cleaner code"
**Good:** "Model A's `_bookmarks_menu()` extraction improves organization (separates UI from persistence logic)"

The 7 axes to map against: Logic & Correctness, Naming & Clarity, Organization, Interface Design, Error Handling, Comments & Docs, PR Ready.

---

## Turn 1: Architectural Prompts

### ✅ DO: Specify WHAT, not HOW
```
Add a [system] that [manages/tracks/handles] [domain concept],
with [configurable options] and [callbacks/events] for [responses].
```

### ✅ DO: Leave implementation approach open
Don't name classes, patterns, data structures, or specific algorithms.

### ✅ DO: Include non-functional requirements
- "with configurable options"
- "with callbacks/events for responses"
- "that can be extended for future use cases"

### ❌ AVOID: Enumerating specific factors
**Bad:** "consider type effectiveness, HP percentages, and available move effects"
**Good:** "based on battle context"
**Why:** Listing factors → both models implement the same list → convergence

### ❌ AVOID: Placement/rendering adjectives
**Bad:** "inline sparkline graphs next to current values"
**Good:** "visual trend indicators for key metrics"
**Why:** "inline" constrains rendering approach → convergence

---

## Turn 2-3: PR Reviewer Prompts

### Proven High-Scoring Patterns

| Pattern | Template | AutoQA Score |
|---------|----------|-------------|
| UX polish | "[Component] should [feedback/indicator] when [condition]" | 7/7, 9/9 |
| Distinguish two states | "[State A] should look different from [State B]" | 9/9 |
| Edge case + threshold | "[Feature] can [problem] when [edge]. Add [solution] for [threshold]" | 7/7 |
| Multi-part prescriptive | "Fix [specific issue] + add [related enhancement]" | 7/7 |

### ❌ AVOID in T2-3

| Pattern | Why | Evidence |
|---------|-----|---------|
| Test prompts with exact names | Always converge (one pytest way) | Battle history FAIL |
| Validation prompts | "raises ValueError" → identical | Multiple convergent turns |
| Generic steering | Not prescriptive | AutoQA flags |
| Deferring to model | "what could be improved?" | AutoQA flags |
| Scope creep | New features outside T1 | Difficulty system T2 item healing → FAIL |

---

## All AutoQA Checks (V4)

| # | Check | What It Looks For | How to Pass |
|---|-------|-------------------|-------------|
| 1 | Dictating Implementation | HOW not WHAT | Behavior/outcome only |
| 2 | Scope Creep | T2-3 outside T1 domain | Stay within T1 feature |
| 3 | Anti-Production Steering | "make it hacky" | Don't discourage quality |
| 4 | Generic Follow-ups | Template prompts | Reference specific code behavior |
| 5 | Vague Quality Directives | "production ready" | Concrete requirements |
| 6 | Deferring to Model | "find problems" | YOU identify, model fixes |
| 7 | Weak Comparisons | Generic "A is better" | Cite specific code |
| 8 | Shallow Analysis | No concrete details | Name methods, functions |
| 9 | Circular Reasoning | "better because better" | Independent evidence |
| **10** | **not_substantiating_grading_axes** | **Rationales not mapped to 7 axes** | **Map observations to axes** |

---

## Perfect Score Examples (from our submissions)

### Weather System (9/9)
**T1:** "Add a weather system that affects battles. Weather can be set per map area or triggered by certain attacks, should display a visual indicator during battle, and affect damage calculations based on type matchups."
**T2:** Duration for attack-triggered weather + remaining turns indicator
**T3:** "Distinguish permanent map weather from attack-triggered weather indicators"

### Process Pinning (7/7)
**T1:** "Add a process pinning feature that lets users mark specific processes to always appear at the top of the process list regardless of sort order, with the ability to pin by name or PID and persist pins across refreshes."
**T2:** Visual separator + pin count
**T3:** Persistence behavior refinement

### Process Grouping (7/7)
**T1:** "Add a process grouping feature that aggregates resource usage by application name with expandable/collapsible groups in the terminal display, showing both individual and aggregate statistics."

---

## Divergence Maximizers

1. **Subjective decisions** — AI behavior, UX choices, "good" thresholds
2. **Multiple valid patterns** — OOP vs functional, class vs function
3. **Configurable options** — Forces design decisions about storage/API
4. **Callbacks/events** — Many ways to implement observer pattern
5. **Format flexibility** — JSON vs YAML vs custom
6. **State management** — Where to store, when to update, how to sync

---

*v2*
