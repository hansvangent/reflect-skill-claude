# /reflect

A Claude Code skill that runs a 3-phase retrospective at the end of any session and keeps your CLAUDE.md sharp over time.

Most sessions end with a close button and you lose the corrections, the preferences, the workflows that finally clicked. `/reflect` extracts what's worth keeping, audits your existing config against what actually happened, and (when the file gets bloated) compresses CLAUDE.md against codebase truth without losing any live rule.

## Install

The skill is a single file. Drop it into your global skills directory.

```bash
mkdir -p ~/.claude/skills/reflect
curl -fsSL https://raw.githubusercontent.com/<your-handle>/reflect/main/SKILL.md \
  -o ~/.claude/skills/reflect/SKILL.md
```

Or clone the repo and symlink it:

```bash
git clone https://github.com/<your-handle>/reflect.git
ln -s "$(pwd)/reflect/SKILL.md" ~/.claude/skills/reflect/SKILL.md
```

Restart Claude Code (or open a fresh session). Type `/reflect` to test.

The skill has `disable-model-invocation: true`, so it only fires when you explicitly invoke it. Nothing runs in the background.

## Usage

Args are space-separated tokens after `/reflect`. Order doesn't matter.

| Command | What it does |
|---|---|
| `/reflect` | Phase 1 → Phase 2 → Phase 3 (if size triggers fire) |
| `/reflect project` | Same, but extends scope to the project's `CLAUDE.md` and `.claude.local.md` in cwd |
| `/reflect compress` | Skip Phases 1 + 2, run only Phase 3 (compression pass) |
| `/reflect compress project` | Compression pass over both global and project files |

## How each phase works

### Phase 1: Extract

Walks the conversation top-to-bottom and sorts everything into four buckets:

1. **Durable facts** about you (role, tools, environment, preferences) → appended to `~/.claude/CLAUDE.md`
2. **Standing corrections** (you said "stop doing X" or "always do Y") → appended with the WHY and WHEN
3. **Repeatable workflows** that pass a conservative "would you actually rerun this?" bar → saved as a new skill
4. **Ephemeral / project-specific** → dropped, listed in the report so you can pull anything back

Phase 1 saves immediately. No asking, no review step. You correct from the final report.

In `/reflect project` mode, Bucket-1 and Bucket-2 facts that are tied to the current project (not universal) get routed to the project's `CLAUDE.md` instead of being dropped.

### Phase 2: Optimize

Audits existing config against what THIS SESSION revealed. Every proposal traces to a concrete moment.

Looks for:
- Contradictions between CLAUDE.md and observed behavior
- Redundant entries that can merge into one canonical line
- Stale references (verified with grep/Read before flagging)
- Permissions you approved repeatedly that should be promoted to the allowlist
- Existing commands that need a rename, a missing arg, or a different response shape
- Multi-step patterns done manually 3+ times in the session, candidate for a new command
- Skill descriptions that have drifted from what the skill actually does
- Tool-call inefficiencies (re-reading unchanged files, sequential calls that should batch, etc.)
- Self-caught mistakes you'd flag even if the user didn't comment

Phase 2 proposes a numbered list and waits for per-item approval. Reply with `apply 1,3,5`, `apply all`, `skip`, or `apply N — but <tweak>`.

### Phase 3: Compress

Phase 3 runs only when at least one of these is true:
- Global `CLAUDE.md` exceeds 250 lines
- Project `CLAUDE.md` exceeds 300 lines (in `/reflect project` mode)
- You explicitly typed `/reflect compress`
- Phase 2 surfaced ≥3 contradiction/redundancy/stale items (signal of drift)

It reads files against actual codebase truth and proposes line-level compressions in six buckets:

1. **Verbose**: multi-sentence rules that flatten to one line
2. **Duplicate within file**: two lines saying the same thing under different headers
3. **Stale by reference**: file paths, commands, flags, or tool names that no longer exist
4. **Historical noise**: "previously X / now Y" pairs where only Y is the live rule
5. **Cross-file duplicate**: same rule in global AND project
6. **Hierarchy mismatch**: global rule with explicit project-specific text (demote), or project rule with no project signal that fits global terrain (lift)

Same approval gate as Phase 2.

## Safety

- **Phase 1 never asks** because every change is purely additive. Worst case: the report shows you what was saved and you delete the bad entry.
- **Phases 2 + 3 never edit anything without per-item approval.** "apply all" is your explicit choice.
- **Phase 3 never auto-deletes a live rule.** It rephrases, merges, or relocates. If the whole rule is dead, it flags it back to you with "should this be removed?" rather than guessing.
- **Phase 3 backs up every file in scope to `<path>.bak-YYYYMMDD` before the first edit.** After backup, it prunes old backups for that path to the most recent 10.
- **Sensitive content stays out**: API keys, OAuth tokens, account numbers, third-party emails, anything else that doesn't belong in a global config.
- **Sycophancy-inducing instructions stay out**: "agree more" or "stop questioning my plans" are not preferences worth keeping.

## What the output looks like

After Phase 1:

```
**/reflect — Phase 1 (saved)**

Durable preferences → ~/.claude/CLAUDE.md:
- ...

Standing instructions → ~/.claude/CLAUDE.md:
- ...

New skill:
- ...

Dropped:
- ...
```

After Phase 2 analysis (waits for approval):

```
**/reflect — Phase 2 (proposals)**

1. <title>
   File: <path>
   Issue: ...
   Change: ...
   Why: ...
2. ...

Reply with: "apply 1,3,5" / "apply all" / "skip" / "apply N — but <tweak>"
```

After Phase 3 applies:

```
**/reflect — compressed:**
  ~/.claude/CLAUDE.md: 287 → 198 lines (-31%)
- #1: ...
- #2: ...
Backup: ~/.claude/CLAUDE.md.bak-20260610
```

## When to run it

End of any session that did real work. Five seconds of typing `/reflect`, ten seconds of reading the proposals, done.

Tip: pair it with a habit. I run it after every session that included at least one correction or a new pattern. If you've never run it before, run `/reflect compress` first to clean baseline drift.

## License

MIT.

## Contributing

PRs welcome. Open an issue first for anything bigger than a typo or a one-line clarification. The skill's design is deliberately tight (additive Phase 1, session-grounded Phase 2, codebase-grounded Phase 3); proposals that blur those boundaries will probably get pushback.

## Credits

Built by [Hans Van Gent](https://hansvangent.substack.com) for his own daily Claude Code workflow. If it saves you a session, drop a star.
