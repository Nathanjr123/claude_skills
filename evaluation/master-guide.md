# Alignerr Code Human Preferences — Master Guide v6

## Executive Summary
You're creating RLHF training data by comparing two SOTA model responses on coding tasks. **Sample selection > evaluation rigor** — 20-40% of preference data is noise. Your job is to maximize information content per comparison while writing rationales that pass review.

**Core insight:** You + Claude collaboration = calibrated evaluation that generic LLMs can't match. Claude has project context, acceptance criteria, research on biases, and memory of what patterns work.



---

## Phase 1: Pre-Task Setup (3 min)

### Copy-Paste Block
```bash
# 1. Kill old sessions (prevents UUID contamination)
tmux kill-server

# 2. Reset repo to clean state (git reset HEAD catches staged files!)
cd /mnt/c/Projects/<REPO>
git reset HEAD 2>/dev/null
git checkout -- . && git clean -fdx
git rev-parse HEAD  # ← SAVE THIS HASH

# 3. Contamination check (SKIP for first task on repo)
grep -r "<PREVIOUS_TASK_KEYWORDS>" <REPO_SRC>/ tests/ --include="*.py" | head -5
# Should return nothing — if it does, clean harder

# 4. Create TAR BEFORE any model interaction
rm -rf /tmp/<REPO> && cp -r . /tmp/<REPO>
cd /tmp/<REPO> && rm -rf .git __pycache__ .pytest_cache venv node_modules .mypy_cache
find . -name "__pycache__" -exec rm -rf {} + 2>/dev/null
cd /tmp && tar --exclude='latest' -cf <REPO>.tar ./<REPO>
cp /tmp/<REPO>.tar ~/Downloads/

# 5. Start fresh HFI session
cd /mnt/c/Projects/<REPO>
claude-hfi --tmux  # ← SAVE UUID immediately
```

### Pre-Task Checklist
- [ ] `tmux kill-server` ran
- [ ] `git reset HEAD` + `checkout` + `clean -fdx`
- [ ] Contamination check (grep for previous task keywords)
- [ ] Commit hash saved
- [ ] TAR created with `--exclude='latest'` and in Downloads
- [ ] `claude-hfi --tmux` started
- [ ] UUID saved

---

## Phase 2: Turn 1 — Architectural Prompt (HIGH DIVERGENCE)

### The Goldilocks Formula (150-300 tokens)
```
[Clear goal, ambiguous HOW] + [Non-functional requirement] + [Configurable options]
```

### Template
```
Add [system/feature] that [manages/tracks/handles] [domain concept],
with [configurable options] and [callbacks/events] for [responses].
[Optional: constraint that must be preserved]
```

### What Makes Turn 1 Good
- Specifies **WHAT**, not **HOW**
- No "make it PR ready" instruction
- Multiple valid implementation approaches exist
- Leaves open: library choice, paradigm (OOP/functional), data structures
- Challenging enough that model doesn't solve in 1 turn
- **Does NOT enumerate specific factors/criteria** (models implement the same list → convergence)
- **Does NOT use placement adjectives** like "inline" that constrain rendering approach

### Turn 1 Execution
1. Submit prompt
2. Monitor BOTH trajectories (`Ctrl+b` then `1` or `2`)
3. If either model produces 0 code → **kill and restart** (3 min vs gambling $300)
4. Approve permissions as needed
5. **Check diff sizes immediately:**
```bash
git -C ~/.cache/claude-hfi/-mnt-c-Projects-<REPO>/A diff <HASH> | wc -l
git -C ~/.cache/claude-hfi/-mnt-c-Projects-<REPO>/B diff <HASH> | wc -l
```
6. **KILL RULE: If either model >800 lines, prompt was too ambitious** — consider restarting
7. Generate full diffs for Claude analysis:
```bash
git -C ~/.cache/claude-hfi/-mnt-c-Projects-<REPO>/A diff <HASH> > ~/model_a.txt
git -C ~/.cache/claude-hfi/-mnt-c-Projects-<REPO>/B diff <HASH> > ~/model_b.txt
cp ~/model_a.txt ~/model_b.txt ~/Downloads/
```
8. Upload to Claude, get analysis
9. Rate with effects-focused pros/cons
10. Verify winner synced: `git status`

### Diff Size Thresholds (from 55 submissions)
| Lines | Risk | Action |
|-------|------|--------|
| < 100 | TOO SMALL | Task too easy, models converge |
| 200-600 | **SWEET SPOT** | Most accepted tasks fall here |
| 600-800 | ACCEPTABLE | Watch for scope |
| 800-1000 | HIGH RISK | Consider restarting |
| > 1000 | **VERY HIGH RISK** | 1061 lines → FAIL, 1163 lines → FAIL |

---

## Phase 3: Turn 2-3 — PR Reviewer Steering (PRESCRIPTIVE)

### Mindset Shift
- Turn 1: You are the **user** (WHAT to build)
- Turn 2-3: You are the **PR reviewer** (HOW to improve)

### What CREATES Divergence (USE THESE)
| Pattern | Example | Why |
|---------|---------|-----|
| UX polish | "Add visual separator with count" | Multiple rendering approaches |
| Edge case | "Handle all-pinned case, spam cooldown" | Multiple guard strategies |
| Multi-part | "Fix X + add Y + update Z" | Different priorities |
| Behavioral refinement | "Pin by name not PID for persistence" | Architecture choice |
| "Distinguish two visual states" | "Permanent vs triggered weather indicators" | Proven 9/9 pattern |

### What CAUSES Convergence (AVOID)
| Pattern | Example | Why |
|---------|---------|-----|
| Test prompts with exact names | "test_save_creates_file" | One pytest way |
| Simple display change | "Display X in yellow" | One obvious solution |
| Validation prompts | "raises ValueError when..." | Identical checks |
| Generic steering | "improve code quality" | Not prescriptive |
| Deferring to model | "what could be improved?" | You identify, model fixes |

### Scope Rules
- ✅ Stay within Turn 1 scope
- ✅ Fix issues YOU identified (not asking model to find them)
- ✅ Point to specific lines/files/behavior
- ❌ NO scope creep (new features outside T1 domain)
- ❌ NO generic steering
- ❌ If T2 converged (all 4s) → T3 MUST target different code layer

### Junk File Check After Each Turn
```bash
git -C ~/.cache/claude-hfi/-mnt-c-Projects-<REPO>/[winner] diff --name-only <HASH>
```
**Flag and steer away:** PLAN_*.md, SUMMARY_*.md, DESIGN_*.md, CHANGELOG additions, .bak files, custom test scripts not using pytest, AI chain-of-thought comments.

---

## Phase 4: Rating & Rationale Writing

### The 7 Axes — Quick Signals

| Axis | Key Questions | Red Flags |
|------|---------------|-----------|
| **Logic & Correctness** | Edge cases? Off-by-one? Race conditions? | Tests don't pass, boundary errors |
| **Naming & Clarity** | Names express purpose? Units in names? | Abbreviations, inconsistent terms |
| **Organization** | SRP? DRY? Functions <50 lines? | God classes, >3 nesting levels |
| **Interface Design** | Params <5? Easy to use correctly? | Train wreck expressions, hidden side effects |
| **Error Handling** | Specific exceptions? Context in messages? | Bare `except:`, swallowed errors |
| **Comments & Docs** | WHY not WHAT? No AI chain-of-thought? | Obvious comments, stale docs |
| **PR Ready** | Tests pass? No debug code? Linted? | TODO comments, print statements |

### Rationale Writing Rules

#### Pros/Cons Format
- **Independent per model** (no "better than Model B")
- **Effects over files** ("Yielding is properly decoupled" not "Modified file.py")
- **Map observations to grading axes** (new V4 check: `not_substantiating_grading_axes`)
- **Ground in code changes** (cite specific methods, classes, behavior)
- Bullet points OK

#### Overall Preference Format
- **5-7 sentences**, self-contained
- **Mention BOTH models** by name (Model A / Model B)
- **Compare** directly (this is where comparisons go)
- **Substantiate** claims with specific code references
- **Human-written only** — typos OK, AI-polished language is NOT

#### AI Detection Avoidance
**Phrases that trigger detection (NEVER USE):**
- "ensures code style compliance"
- "leveraging the existing framework"
- "comprehensive error handling"
- "robust implementation"
- "well-structured approach"

**What passes cleanly:**
- Typos and informal phrasing (paradoxically protective)
- First-person observations: "I noticed the edge case..."
- Colloquial shorthand: "this is a bit janky because..."
- Domain-specific references: "the `_highlight_text` returns styled spans"

### Standard Analysis Template (Claude collaboration)

```
## 📊 Axis Ratings
| Axis | Rating | Notes |
|------|--------|-------|
| Logic & Correctness | [A/B/Tie] ([1-7]) | [Observation] |
| Naming & Clarity | ... | ... |
| Organization | ... | ... |
| Interface Design | ... | ... |
| Error Handling | ... | ... |
| Comments & Docs | ... | ... |
| PR Ready | ... | ... |
| **Overall** | **[A/B/Tie]** | |

## Model A
**Pros:**
- [Effect-focused observation mapped to axis]

**Cons:**
- [Issue with specific code reference]

## Model B
**Pros:**
- [Effect-focused observation mapped to axis]

**Cons:**
- [Issue]

## Overall Preference (5-7 sentences)
[Self-contained justification mentioning both models with code evidence]
```

---

## Phase 5: Handling Convergence

1. **Use middle ratings (4)** for all axes — NOT N/A
2. **Don't force artificial preferences**
3. **Note convergence in rationale**
4. **If 2+ turns converge → task is likely a FAIL. Salvage with a radically different T3**

### Convergence Prevention
- Leave implementation approach open
- Don't prescribe data structures or enumerate specific factors
- Include subjective decisions (UX choices, "good" thresholds)
- Use architectural prompts over bug fixes
- Include configurable options (forces design decisions)

---

## Phase 6: Pre-Submission Verification (CRITICAL)

### Verify Session Archive Has All Turns
```bash
UUID=<YOUR_UUID>

PROMPTS=$(ls /tmp/claude-hfi/$UUID/prompt-*.json 2>/dev/null | wc -l)
FEEDBACKS=$(ls /tmp/claude-hfi/$UUID/feedback-step-*.json 2>/dev/null | wc -l)

echo "Prompts: $PROMPTS | Turns rated: $FEEDBACKS"

if [ "$FEEDBACKS" -lt 2 ]; then
  echo "❌ STOP: Only $FEEDBACKS turns rated. Minimum is 2. DO NOT SUBMIT."
else
  echo "✅ PASS: $FEEDBACKS turns rated. Safe to submit."
fi
```

### Generate and Verify Diff
```bash
cd /mnt/c/Projects/<REPO>
git add -A
git --no-pager diff --cached <HASH> > ~/<UUID>_final.diff
wc -l ~/<UUID>_final.diff  # Should NOT be 0, should be 200-600 ideally

# Verify diff applies cleanly
rm -rf /tmp/<REPO>-verify && cp -r . /tmp/<REPO>-verify
cd /tmp/<REPO>-verify
git reset --hard <HASH> && git clean -fd
git apply ~/<UUID>_final.diff && echo "✅ DIFF APPLIES" || echo "❌ DIFF FAILED"

# Create session archive
cp -r /tmp/claude-hfi/$UUID ~/
cd ~ && tar -cf ${UUID}.tar ./$UUID

# Copy to Downloads
cp ~/${UUID}_final.diff ~/Downloads/
cp ~/${UUID}.tar ~/Downloads/
```

---

## Bias Checklist (Check Before EVERY Submission)

| Bias | Question | Mitigation |
|------|----------|------------|
| **Verbosity** | Am I preferring longer code? | "If same length, which is better?" |
| **Formatting** | Am I swayed by prettier markdown? | Focus on logic, not presentation |
| **Familiarity** | Am I penalizing unfamiliar patterns? | Evaluate against rubric first |
| **Recency** | Am I favoring Model B because I read it last? | Re-read A before deciding |

**47.5% of preference flips are due to length alone.** Actively check for verbosity bias.

---

## Kill Rules (Abandon Task If Triggered)

| Trigger | Action |
|---------|--------|
| 2+ turns with all-4 ratings | Abandon — no recovery path |
| Total diff > 1000 lines | Prompt too ambitious — restart |
| Same concept submitted before on this repo | Don't submit |
| Both models identical on T1 | Restart with better prompt |
| Winner pattern A→B→A across turns | Risky — justify switches explicitly |
| Model produces 0 code | Kill session, restart (3 min) |
| Junk .md files in diff after steering | Possible automated detection |

---

## Anti-Patterns Quick Reference

### Prompt Anti-Patterns
| ❌ Don't | ✅ Do Instead |
|----------|---------------|
| "Make it PR ready" | Let model learn this implicitly |
| "Use a dict here" | Specify WHAT, not HOW |
| "Consider type effectiveness, HP%, move effects" | "Based on battle context" (let model decide factors) |
| "Add validation that raises ValueError" | "Add tests verifying [behavior]" |
| "Improve code quality" | "The function at line 52 has cyclomatic complexity >10" |
| "Add inline sparkline graphs next to" | "Add visual trend indicators" (no placement words) |
| Too easy (1-turn solve) | Add complexity/ambiguity |

### Model Output Anti-Patterns to Steer Away
| Anti-Pattern | Steering Prompt |
|-------------|-----------------|
| Creates PLAN_*.md, SUMMARY_*.md | "Delete any planning or summary files" |
| Custom test scripts instead of pytest | "Use existing test suite with pytest" |
| Leaves backup files | "Remove backup files" |
| Doesn't run tests | "Run `pytest tests/` -v" |
| AI chain-of-thought comments | "Remove implementation comments that explain reasoning" |
| README/CHANGELOG modifications | "Revert changes to README and CHANGELOG" |

### Rationale Anti-Patterns
| ❌ Bad | ✅ Good |
|--------|---------|
| "Model A's error handling is better than Model B's" (in pros/cons) | "Model A returns errors in res object without throwing" |
| "The application crashes" | "The app crashes because div inside p tag in page.tsx" |
| "Includes 73 new test cases" | "Includes 73 tests but most are trivial input validation" |
| "The code was good overall" | "The `_bookmarks_menu()` extraction improves organization (Axis: Organization)" |
| "Model A leverages robust architecture" | "Model A extracts the health check loop into a separate `HealthMonitor` class" |

---

## Hidden Metrics Awareness

These aren't visible in AutoQA but strongly correlate with acceptance/rejection:

| Metric | What It Checks | Threshold |
|--------|---------------|-----------|
| **Code Divergence** | A/B similarity | <70% overlap across turns |
| **Diff Size** | Total lines changed | 200-600 sweet spot |
| **Rating Variance** | Are ratings all 4s? | ≥2 axes outside 3-5 per non-convergent turn |
| **Junk Files** | PLAN_*.md, SUMMARY_*.md, .bak | Zero tolerance |
| **AI Detection** | Rationale text classifier | Write rough/human, not polished/AI |
| **Concept Reuse** | Same feature on same repo | Never repeat |
| **Winner Coherence** | A→B→A switching pattern | Switches must be justified |
| **Turn Productivity** | New code per turn | 50+ meaningful lines per turn |

See `evaluation_metrics_complete.md` for full analysis of all 10 hypothesized hidden metrics.

---

## Speed Optimizations

| Optimization | Time Saved | When |
|-------------|-----------|------|
| Kill & restart > debug corrupted session | 15-20 min | Model produces 0 code or session errors |
| `wc -l` diff check after T1 | Catches scope blowup before investing in analysis | After every turn |
| Write rationales in note app, paste into CLI | Better writing, less claustrophobic | Every turn |
| Pre-built contamination keywords per repo | No thinking time on grep patterns | Pre-task |
| Claude provides axis table, you rewrite in own words | Structured analysis without AI-sounding output | Every turn |

---

## Target Metrics

| Metric | Target | Current |
|--------|--------|---------|
| Time per task | 30-45 min | ~35 min |
| Turns per task | 3 (min 2) | 3 |
| Acceptance rate | 60%+ | **76.5%** |
| Tasks per day | 3+ | varies |

---

*Version 6.0*

*Changes from v5: Added kill rules, diff size thresholds, junk file detection, pre-submission verification, AI detection avoidance, hidden metrics summary, speed optimizations, failure autopsy integration, V4 AutoQA check, contamination workflow*
