# Razor

Minimize your branch diff against merge base with structured, explicit tradeoffs.

Razor analyzes your working branch against its merge base and produces a ranked reduction plan: behavior-preserving refactors first, then a ladder of options with increasing functionality loss. You pick which cuts to make.

## What it does

1. Anchors to your merge base (not just HEAD~1)
2. Measures insertions, deletions, netlines, and total churn
3. Ranks files by diff contribution
4. Proposes **behavior-preserving refactors** (section A) — no feature loss
5. Proposes a **functionality-loss ladder** (section B) — low/medium/higher impact options
6. Applies only what you select, re-measures, and loops

## Install

### Claude Code

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/blader/razor.git ~/.claude/skills/razor
```

Then restart Claude Code. The skill will be available as `/razor`.

### Codex

```bash
# Clone into your Codex skills directory
git clone https://github.com/blader/razor.git ~/.codex/skills/razor
```

Then restart Codex. The skill will be available as `$razor`.

### One-liner (auto-detect)

```bash
SKILLS_DIR="${CLAUDE_SKILLS_DIR:-${CODEX_SKILLS_DIR:-$HOME/.claude/skills}}" && \
git clone https://github.com/blader/razor.git "$SKILLS_DIR/razor"
```

## Usage

**Claude Code:**
```
/razor
```

**Codex:**
```
$razor
```

Or just ask naturally:
> Shrink my diff against latest
> Tighten this branch's diff
> Minimize my local changes

## How it works

Razor optimizes in strict priority order:

| Priority | Metric | Formula |
|----------|--------|---------|
| Primary | netlines | `insertions - deletions` |
| Secondary | total churn | `insertions + deletions` |

### Guardrails

- Never recommends reducing tests, docs, or comments
- Never recommends splitting into multiple PRs
- Never uses destructive git commands
- Never cuts the core purpose of the branch
- Tracks declined options across iterations (won't re-suggest them)

## License

MIT
