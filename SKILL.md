---
name: razor
description: "Minimize local diff size against merge base for uncommitted branch work. Use when asked to tighten or shrink a local diff while keeping core behavior, optimizing first for merge-base netlines (insertions - deletions), then for total churn (insertions + deletions), and producing a ranked reduction plan in this order: (1) behavior-preserving refactors, then (2) options with increasing functionality loss."
---

# Razor

Reduce local diff size with merge-base-anchored analysis and explicit tradeoffs.

## Non-negotiables
- Always anchor analysis to merge base: `MB=$(git merge-base <base-ref> HEAD)`.
- **Unified diff only**: Always measure the working tree (committed + staged + unstaged) as ONE diff against merge base using `git diff "$MB"`. Never separately analyze "committed vs MB" and "uncommitted vs HEAD" — the PR diff is the full working-tree state, period. Include untracked new files in the totals (count their lines as insertions).
- Preserve behavior by default.
- Keep idea groups strictly separate:
  1. Group A contains only behavior-preserving ideas (no behavior change).
  2. Group B contains only behavior-changing options (explicit functionality loss or constraint).
  3. Never mix behavior-changing ideas into Group A or behavior-preserving ideas into Group B.
- Optimize in strict priority order against merge base:
  1. Primary: minimize netlines (`insertions - deletions`).
  2. Secondary: minimize total churn (`insertions + deletions`).
- Do not recommend reducing tests, documentation, or comments.
- Prioritize app/runtime code over test code when proposing reductions:
  - Treat app/runtime files as the primary optimization surface.
  - Only propose test-file reductions after credible app/runtime opportunities are exhausted or blocked.
  - Any test-file suggestions must preserve coverage and assertions quality.
- Do not recommend refactors that reduce code clarity (e.g., collapsing multi-line logic into dense ternaries, removing meaningful variable names, compressing well-formatted code into fewer lines). Only suggest refactors that maintain or increase clarity.
- Do not recommend splitting work into multiple PRs.
- Never use destructive git commands (`reset --hard`, `checkout --`, etc.).
- After each cut-down-diff execution pass, start a fresh cut-down-diff planning run only once execution is complete (never in the middle of execution).
- Do not recommend functionality-loss options that remove or undermine the branch’s core feature objective.
- If the branch is centered on a capability (for example formula-preserving exports), loss options must keep that capability intact.
- Do not recommend options that remove, disable, or materially reduce non-main-layer harness coverage.
- Track option-selection history across iterations:
  - Do not re-suggest the same previously declined/skipped B options unless the user explicitly asks to reconsider them.
  - Still propose new, distinct B options each rerun when available (do not treat prior skips as a blanket ban on all future B ideas).
  - When prior B ideas were declined, keep producing a full B ladder with newly available options (low/medium/higher), subject to core-feature and coverage guardrails.

## Workflow

### 1) Establish merge-base anchor
1. Resolve base ref (`origin/latest` unless user specifies another base).
2. Compute merge base: `MB=$(git merge-base <base-ref> HEAD)`.
3. Show divergence: `git rev-list --left-right --count <base-ref>...HEAD`.

### 2) Measure unified working-tree diff against merge base
The diff is the full working tree (committed + uncommitted + untracked) vs merge base. This is exactly what the PR will contain. Never split this into "committed" and "uncommitted" sub-analyses.

1. Run:
   - `git status --short` (overview of tracked modifications and untracked files)
   - `git diff --numstat "$MB"` (unified working-tree diff for tracked files)
   - `git diff --name-status "$MB"` (change types)
   - `git ls-files --others --exclude-standard` (untracked files)
   - `wc -l` on each untracked code file (these lines count as insertions)
2. Report **single** set of totals combining tracked diff + untracked file lines:
   - `insertions` (tracked insertions + untracked file lines)
   - `deletions` (tracked deletions only)
   - `netlines = insertions - deletions` (primary metric)
   - `total churn = insertions + deletions` (secondary metric)
3. Rank top files by touched lines and highlight biggest netline contributors. Include untracked new files in the ranking.
4. Split hotspot reporting into two buckets and prioritize in this order:
   - App/runtime code hotspots (primary)
   - Test code hotspots (secondary)
   Use app/runtime hotspots to drive idea generation unless blocked.

### 3) Generate reduction ideas in two strictly separate groups

#### Group A) Direct refactor ideas (behavior-preserving, no feature loss)
Generate up to 10 ideas for section A (fewer only if there are not enough credible, file-specific opportunities).
Produce concrete, file-specific edits that reduce churn without changing behavior, such as:
1. Replace broad rewrites with targeted edits in existing functions.
2. Reuse existing helpers instead of introducing duplicated logic.
3. Collapse equivalent add/delete churn into in-place edits.
4. Remove incidental mechanical churn not required for behavior.
5. Minimize API-surface changes that force cascading edits.

Prioritization for Group A:
1. Propose app/runtime-code ideas first.
2. Add test-code ideas only after app/runtime opportunities are covered or insufficient.

For each idea include:
1. Files/hunks affected.
2. Estimated netline reduction (`insertions - deletions`).
3. Estimated total-churn reduction (`insertions + deletions`).
4. Why behavior remains unchanged.

#### Group B) Functionality-loss ladder (increasing loss)
Only after section A, provide up to 10 options with explicit rising loss (fewer only if there are not enough credible options):
1. Low loss.
2. Medium loss.
3. Higher loss.
Every Group B option must explicitly change or constrain behavior.

Guardrail:
- Exclude any option that cuts the core purpose of the branch; if such an option would otherwise appear, replace it with a lower-impact constraint that preserves the core behavior.
- Exclude options that remove non-main-layer harnesses or their coverage; propose alternatives that keep that coverage intact.
- Exclude only B ideas previously declined or skipped by the user in earlier cut-down-diff iterations; continue offering new B ideas that were not previously declined.
- Prefer app/runtime behavior constraints over test-scope constraints when both are available.

For each option include:
1. Exact behavior removed or constrained.
2. Estimated netline reduction (`insertions - deletions`).
3. Estimated total-churn reduction (`insertions + deletions`).
4. Risk/impact statement.

### 4) Apply selected options
1. Implement only user-selected options.
2. Keep edits minimal and local.
3. Remove dead references created by those edits.
4. Run targeted validation for remaining behavior.

### 5) Re-measure and report
1. Re-run merge-base diff stats.
2. Show before/after for both metrics:
   - netlines (`insertions - deletions`)
   - total churn (`insertions + deletions`)
3. Show percent reduction for both metrics.
4. List exactly what changed and why.

### 6) Post-execution planning rerun
1. After finishing execution and reporting results, immediately start a new cut-down-diff planning run.
2. Use the updated working tree and the same merge-base anchor unless the user specifies a new base.
3. Generate the next ranked plan from the new residual hotspots.

## Output contract
Always return:
1. Merge base SHA and base ref used.
2. Before stats vs merge base (insertions, deletions, netlines, total churn).
3. Ordered reduction plan:
   - Group A first: direct refactor ideas (no behavior loss), up to 10 ideas.
   - Group B second: functionality-loss ladder (low to high), up to 10 options.
   - Keep Group A and Group B strictly separated (no cross-over ideas).
4. For every proposed and applied option, include estimated/actual impact on:
   - Primary metric: netlines.
   - Secondary metric: total churn.
5. After applying changes: after stats, primary reduction first, secondary reduction second, and residual hotspots.

## Command snippets

```bash
# Setup
BASE_REF=origin/latest
MB=$(git merge-base "$BASE_REF" HEAD)

# Overview
git status --short
git diff --numstat "$MB"         # unified working-tree diff (committed+uncommitted) vs merge base
git diff --name-status "$MB"
git ls-files --others --exclude-standard  # untracked new files
```

```bash
# Totals for tracked files (unified working-tree diff — do NOT use "$MB" HEAD)
git diff --numstat "$MB" \
| awk '{adds+=$1; dels+=$2} END {printf "insertions=%d\ndeletions=%d\nnetlines=%d\ntotal_churn=%d\n", adds, dels, adds-dels, adds+dels}'
```

```bash
# Add untracked code file lines to get full PR totals
# (untracked lines are pure insertions — add to tracked insertions above)
git ls-files --others --exclude-standard | grep -E '\.(go|ts|tsx|js|jsx|py|rs)$' | xargs wc -l | tail -1
```

```bash
# Per-file ranking (tracked files — append untracked files manually)
git diff --numstat "$MB" \
| awk '{t=$1+$2; print t "\t" $1 "\t" $2 "\t" $3}' \
| sort -nr
```
