# Alignerr Workflow v4

## Pre-Task (3 min)
```bash
# 1. Kill old sessions (prevents UUID contamination)
tmux kill-server

# 2. Reset repo
cd /mnt/c/Projects/<REPO>
git checkout -- . && git clean -fd
git rev-parse HEAD  # SAVE HASH

# 3. TAR FIRST (clean state, before any model interaction)
rm -rf /tmp/<REPO> && cp -r . /tmp/<REPO>
cd /tmp/<REPO> && rm -rf .git __pycache__ .pytest_cache venv node_modules
cd /tmp && tar -cf <REPO>.tar ./<REPO>
cp /tmp/<REPO>.tar ~/Downloads/

# 4. Start fresh session
cd /mnt/c/Projects/<REPO>
claude-hfi --tmux  # SAVE UUID immediately
```

## Turn 1 - Architectural Prompt
- Submit high-divergence prompt (feature system, multiple components)
- Monitor BOTH trajectories (Ctrl+b then 1 or 2)
- Approve permissions as needed
- **YOU review diffs** in VS Code/editor (not AI)
- Rate with effects-focused pros/cons
- Verify winner synced: `git status`

## Turn 2-3 - Behavior Tests
- Prescriptive PR-reviewer steering
- "Add tests verifying [behavior] when [condition]"
- **NO validation prompts** (causes convergence)
- Rate each turn, effects-focused rationales

## Final Submission (3 min)
```bash
cd /mnt/c/Projects/<REPO>
git add -A
git --no-pager diff --cached <HASH> > ~/<UUID>_final.diff

# Verify diff applies
rm -rf /tmp/<REPO>-verify && cp -r . /tmp/<REPO>-verify
cd /tmp/<REPO>-verify
git reset --hard <HASH>
git apply ~/<UUID>_final.diff && echo "SUCCESS"

cp ~/<UUID>_final.diff ~/Downloads/
```

## Form Fields
- UUID: from claude-hfi startup
- Commit Hash: from git rev-parse HEAD
- Turns: 3
- Tar: `<REPO>.tar`
- Diff: `<UUID>_final.diff`

---

## Prompts That Work

**Turn 1 (Architectural - HIGH DIVERGENCE):**
- Feature systems with callbacks/events
- Data collection + visualization
- Modular subsystems (AI behavior, state management)
- Multiple interacting components

**Turn 2-3 (Behavior Tests - NOT validation):**
- "Add tests verifying [component] [behavior] when [condition]"
- "Test that [system] correctly [action] when [trigger]"
- Point to specific lines/files like a PR reviewer

**AVOID:**
- "Add validation that raises ValueError..."
- Generic steering ("improve code quality")
- Scope creep

---

## Rationale Rules (Discord-Validated)

1. **Implementing isn't a pro** - focus on HOW WELL
2. **Effects over files** - "Yielding is properly decoupled" not "Modified file.py"
3. **Atomic per model** - no comparisons in pros/cons
4. **5-7 sentences overall** - self-contained, both models mentioned
---

## Session Checklist

Before each task:
- [ ] `tmux kill-server`
- [ ] `git checkout -- . && git clean -fd`
- [ ] Hash saved
- [ ] Tar created and in Downloads
- [ ] Fresh `claude-hfi --tmux`
- [ ] UUID saved

After task:
- [ ] 3 turns completed
- [ ] Final diff created
- [ ] Diff verified (applies cleanly)
- [ ] Form submitted
