# Alignerr Quick Reference v5

## Stats (Feb 28, 2026)
- **Acceptance rate:** 76%+
- **Revenue:** 

## Pre-Task
```bash
tmux kill-server
cd /mnt/c/Projects/<REPO>
git reset HEAD 2>/dev/null
git checkout -- . && git clean -fdx
git rev-parse HEAD  # SAVE HASH

# Contamination check
grep -r "<PREVIOUS_KEYWORDS>" <SRC>/ tests/ --include="*.py" | head -5

# TAR
rm -rf /tmp/<REPO> && cp -r . /tmp/<REPO>
cd /tmp/<REPO> && rm -rf .git __pycache__ .pytest_cache venv .mypy_cache
cd /tmp && tar --exclude='latest' -cf <REPO>.tar ./<REPO>
cp /tmp/<REPO>.tar ~/Downloads/

cd /mnt/c/Projects/<REPO>
claude-hfi --tmux  # SAVE UUID
```

## Per-Turn Diff + Size Check
```bash
HASH=<HASH>
git -C ~/.cache/claude-hfi/-mnt-c-Projects-<REPO>/A diff $HASH > ~/model_a.txt
git -C ~/.cache/claude-hfi/-mnt-c-Projects-<REPO>/B diff $HASH > ~/model_b.txt
wc -l ~/model_a.txt ~/model_b.txt  # CHECK: 200-600 = good, >800 = danger
cp ~/model_a.txt ~/model_b.txt ~/Downloads/
```

## Pre-Submission Verification
```bash
UUID=<UUID>
FEEDBACKS=$(ls /tmp/claude-hfi/$UUID/feedback-step-*.json 2>/dev/null | wc -l)
echo "Turns rated: $FEEDBACKS"
[ "$FEEDBACKS" -lt 2 ] && echo "❌ STOP" || echo "✅ GO"
```

## Final Submission
```bash
cd /mnt/c/Projects/<REPO>
git add -A
git --no-pager diff --cached $HASH > ~/${UUID}_final.diff
wc -l ~/${UUID}_final.diff  # NOT 0

rm -rf /tmp/<REPO>-verify && cp -r . /tmp/<REPO>-verify
cd /tmp/<REPO>-verify && git reset --hard $HASH && git clean -fd
git apply ~/${UUID}_final.diff && echo "✅ SUCCESS"

cp -r /tmp/claude-hfi/$UUID ~/
cd ~ && tar -cf ${UUID}.tar ./$UUID
cp ~/${UUID}_final.diff ~/${UUID}.tar ~/Downloads/
```

## Kill Rules
- 2+ turns all-4 ratings → abandon
- Diff > 1000 lines → restart with tighter scope
- Same concept on same repo → don't submit
- Models identical on T1 → restart
- A→B→A winner switching → risky
- Junk .md files after steering → automated detection risk

## Turn 1 Template
```
Add [system] that [manages/tracks/handles] [domain concept],
with [configurable options] and [callbacks/events] for [responses].
```
- WHAT not HOW — no classes, patterns, data structures
- Don't enumerate specific factors (models build same list)
- No placement adjectives ("inline", "sidebar")

## Turn 2-3 Templates
```
# UX polish (proven 9/9)
[Component] should [provide feedback/display indicator] when [condition].

# Edge case
[Feature] can [problem] when [edge condition]. Add [solution]
so [desired behavior] for at least [threshold].

# Distinguish two states (proven pattern)
[State A] should [look/behave differently] from [State B].
Ensure the [indicator] distinguishes between these two cases.
```

## Rationale Checklist
- [ ] Pros/cons: independent per model, NO comparisons
- [ ] Effects over files (cite methods/classes, not filenames)
- [ ] Map observations to 7 grading axes (V4 AutoQA check)
- [ ] Overall: 5-7 sentences, mentions BOTH models, substantiated
- [ ] Human-written — no "leverages", "ensures", "comprehensive", "robust"
- [ ] Verbosity bias check: "if same length, which is better?"
- [ ] Recency bias check: re-read Model A before deciding

## Diff Size Sweet Spot
| Lines | Verdict |
|-------|---------|
| < 100 | Too easy |
| 200-600 | ✅ Target |
| 600-800 | Acceptable |
| > 800 | ⚠️ Danger |
| > 1000 | ❌ Kill |

---
*v5 — Feb 2026*
