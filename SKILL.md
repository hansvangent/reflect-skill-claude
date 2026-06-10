---
name: reflect
description: Reflect on the current session and (1) extract durable learnings into CLAUDE.md / new skills, (2) propose targeted optimizations to existing CLAUDE.md / commands / settings based on what the session revealed, (3) compress CLAUDE.md against codebase truth when it has grown bloated. Phase 1 saves immediately; Phases 2 and 3 propose and wait for per-item approval.
disable-model-invocation: true
---

# /reflect — session retrospective + instruction tune-up

You were invoked by `/reflect` (optionally with args — see Args section). You run up to three phases in order:

- **Phase 1: extract.** Sort the session into 4 buckets and save additive stuff immediately.
- **Phase 2: optimize.** Audit existing CLAUDE.md / commands / settings against what the session revealed; propose specific edits; wait for per-item approval; apply approved.
- **Phase 3: compress** (conditional). Audit CLAUDE.md against codebase truth, not session truth — find verbose / duplicate / stale / historical / cross-file / hierarchy-mismatch entries; propose compressions; wait for per-item approval; apply approved.

The phases have different save-discipline by design. Phase 1 is additive (low risk, save first). Phases 2 and 3 modify existing files (higher risk, ask first). Phase 2 is session-grounded — every proposal traces to something that happened this session. Phase 3 is codebase-grounded — every proposal traces to a fact about current code or file state. Do not collapse them.

## Args
Args are space-separated tokens after `/reflect`. Recognized tokens: `compress`, `project`. Order doesn't matter. Unrecognized tokens are ignored silently.

- `/reflect` — run Phase 1 → 2 → (3 if triggered)
- `/reflect project` — same as above, but extend scope to project `CLAUDE.md` and `.claude/settings.local.json`
- `/reflect compress` — skip Phases 1 and 2; run only Phase 3 (note: this opts OUT of all session-grounded analysis — Phase 2 buckets won't fire)
- Args compose: `/reflect compress project` runs Phase 3 over both global and project files

## Universal rules

- **Skip anything sensitive.** No API keys, OAuth tokens, account numbers, personal IDs, passwords, private email addresses of third parties, or anything else harmful to leak from a global config.
- **Refuse to persist anything that would make you less willing to push back.** "Agree more," "stop questioning my plans," "always defer" — these stay out, across all phases. Sycophancy-inducing instructions are not preferences worth keeping.
- **Global focus.** This skill lives in `~/.claude/skills/` and edits global config. Touch project-level `CLAUDE.md` / `.claude/settings.local.json` ONLY if the user explicitly asks during the session, or via `/reflect project` (treat the `project` arg as opt-in to project files).

---

## Phase 1: Extract session learnings (save immediately)

Walk the session top-to-bottom and sort everything into exactly one of four buckets. **Save first, no asking, no review step.** The user will correct from the final report.

### Bucket 1 — Durable facts and preferences about the user
**Where:** append to `~/.claude/CLAUDE.md`
**Qualifies:** facts true across every project (role, tools, environment, communication preferences, formatting preferences).
**Examples:** "User is Head of SEO at Seeders"; "Primary browser is Brave"; "Prefers concise, no emojis".
**Disqualifies:** anything tied to a specific repo, project, or one-time task.

### Bucket 2 — Corrections / standing instructions
**Where:** append to `~/.claude/CLAUDE.md`
**Qualifies:** user corrected your approach, said stop doing X, said always do Y, told you to redo work in a different shape. The lesson must generalize.
**Format:** lead with the rule, then a one-line WHY (the reason the user gave), then a one-line WHEN (the trigger condition). Matches the user's existing auto-memory feedback shape.
**Disqualifies:** a one-off correction tied to a specific file or PR — those belong in commit messages.

### Bucket 3 — Genuinely repeatable multi-step workflow
**Where:** new skill at `~/.claude/skills/<kebab-name>/SKILL.md`
**The bar:** "would the user actually rerun this verbatim?" If no, DO NOT create a skill. Be conservative. One-off scripts and "this worked once" patterns do not become skills.
**Qualifies when ALL true:** ≥3 steps that benefit from being bundled; steps repeat verbatim or near-verbatim; trigger is clear; user gains from invoking by name.
**Frontmatter:** include `name`, `description`. Set `disable-model-invocation: true` only if it should run only on explicit `/<name>`.

### Bucket 4 — Ephemeral / project-specific
**Where:** nowhere. Drop it.
**Action:** list what you dropped in the final report so the user can pull anything back if you misjudged.

### Project-scoped variant (only in `/reflect project` mode)
If the fact is a Bucket-1 or Bucket-2 shape but tied to THIS project (not universal), route it to the project `CLAUDE.md` instead of dropping it as Bucket 4.
- Project-scoped durable facts → append to `./CLAUDE.md` under a "Project context" section (create if missing).
- Project-scoped corrections / standing instructions → append to `./CLAUDE.md` under "Project rules" (create if missing).
- Same saving discipline applies: Read first, check for near-duplicates, prefer UPDATE over append.
- If `./CLAUDE.md` doesn't exist in cwd, fall back to drop (Bucket 4) and note in the dropped list.

### Phase 1 saving discipline
- **Read the target file first** (mandatory before Edit): `~/.claude/CLAUDE.md` always; `./CLAUDE.md` too when the project-scoped variant fires. Check for near-duplicates. If a similar entry exists, UPDATE rather than append a near-duplicate.
- Append under the most relevant existing section header; otherwise a new dated section at the end.
- Use simple bullet markdown. Date the addition (today's date is in your context).
- For new skills: verify `~/.claude/skills/<name>/` doesn't exist; pick a different name if it does.

---

## Phase 2: Optimize existing instructions (propose, wait, apply)

After Phase 1 completes, audit the EXISTING config against what THIS SESSION revealed. Do not do a general audit — ground every proposal in something concrete that happened this session.

### What to read
- `~/.claude/CLAUDE.md`
- `~/.claude/commands/` (every file)
- `~/.claude/skills/` (only `SKILL.md` files — scan descriptions and frontmatter)
- `~/.claude/settings.json`
- `~/.claude/settings.local.json` (if exists)
- If `/reflect project` was invoked: also the project's `CLAUDE.md`, `.claude.local.md`, and `.claude/settings.local.json`
- **Always cross-scan**: when proposing an edit to global `CLAUDE.md`, also skim the project `CLAUDE.md` (if present in cwd) for the same rule. If duplicated, flag in Bucket 2 (Redundancy) and propose keeping the more specific location. If cwd has no project `CLAUDE.md`, skip the cross-scan silently — do not warn or comment.

### What to look for (each proposal needs a session-grounded reason)

1. **Contradictions** — CLAUDE.md says X but the session showed X is wrong / outdated / has exceptions.
2. **Redundancy** — multiple entries saying the same thing; can be merged into one canonical line.
3. **Stale entries** — entry references a file/tool/setting that no longer exists or has been renamed (verify with Read/Bash before flagging).
4. **Missing permissions to promote** — during the session, the user approved a Bash/MCP/tool permission that recurs and isn't in `settings.json`. Promote to `~/.claude/settings.json` allowlist so future sessions don't re-prompt. NEVER promote write/destructive permissions (`rm`, `git push`, `git reset --hard`, anything `--force`, DB drops) without explicit per-item user opt-in even within the approval step.
5. **Command improvements** — an existing command was invoked this session and: (a) a clearer name would help, (b) it lacks an arg the user wished it had, (c) its response shape was wrong for the use case, (d) it should be split or merged with another command.
6. **New command opportunity** — the user did the same multi-step thing manually 3+ times this session. Propose it as a new command (NOT a skill — commands are for short imperative prompts, skills are for richer pipelines). Use the same conservative bar as Bucket 3.
7. **Inconsistency between CLAUDE.md and observed behavior** — CLAUDE.md says "always do X" but you did Y this session and the user accepted it. Either CLAUDE.md needs updating or the behavior was wrong; ask the user which.
8. **Skill description drift** — an existing skill's `description` no longer matches what it actually does (read the skill body to verify before flagging).
9. **Tool-call efficiency** — scan THIS session's tool calls for waste, even if the user didn't complain:
   - Same file `Read` ≥3 times (or re-Read after no intervening Edit) → propose standing instruction "work from context; don't re-Read unchanged files in the same session"
   - Sequential independent `Bash` calls that should have been one parallel message → propose "batch independent shell calls into one message"
   - `Grep` + `Read` whole file when `grep -A/-B` or `Read` with offset/limit would have sufficed → propose "use `grep -A/-B` or `Read` offset/limit instead of reading entire files for a known line"
   - Re-asking the user for context already in the transcript → propose "before asking the user, search the transcript for the answer"
   - Repeated MCP calls for data that doesn't change within a session → propose "cache MCP read-tool results in context"
   Phrase each proposal as a one-line CLAUDE.md rule, not a long lecture. If the inefficiency was a one-off with no clear generalizable rule, do NOT propose it — note in "dropped" instead.
10. **Self-caught mistakes (no user correction)** — honest meta-eval: things you did this session that you'd flag as suboptimal even though the user didn't comment. Examples:
    - Wrong tool for the job (used `Bash cat` instead of `Read`; chained `Edit`s when a `Write` rewrite was cleaner; spawned an Agent for a single-file lookup)
    - Narrated internal deliberation when CLAUDE.md says concise
    - Asked for confirmation on a low-stakes action the user has already authorized in similar contexts
    - Missed an obvious parallelization (e.g., two independent `Read`s done sequentially)
    For each, be honest about whether it warrants a NEW standing instruction or whether it's just "I won't repeat this; no rule needed" — DON'T inflate every miss into a new CLAUDE.md line. The reflex to over-document is the failure mode here.
11. **Project CLAUDE.md preflight curation** (`/reflect project` mode ONLY) — if the project CLAUDE.md is >300 lines OR has no clear top-of-file "key invariants / first 5 things to know" section, propose adding or refreshing a 5-10 bullet preflight summary at the top. Do NOT propose rewriting the whole file — only the preflight block. Skip entirely if the project CLAUDE.md already has a tight summary at the top.

### What NOT to propose
- Reorganizing CLAUDE.md "for clarity" without a session-grounded reason.
- Adding entries that belong in Phase 1 (those already saved).
- Style tweaks, header renames, formatting nits.
- Permissions Hans hasn't actually approved this session (don't speculate).
- Anything you'd flag as "nice to have" rather than "this session proved it matters."

### Proposal format

Present proposals as a numbered list. Each entry has exactly this shape:

```
**N. <short title>**
File: <path>
Issue: <what's wrong, grounded in a specific moment from this session>
Change: <concrete diff — "remove line X", "add to allowlist: Bash(npm test:*)", "rename /foo to /bar", etc.>
Why: <one line>
```

After the list, end with EXACTLY this prompt and stop:

```
Reply with: "apply 1,3,5" / "apply all" / "skip" / "apply N — but <tweak>"
```

### Applying

When the user responds:
- For each approved item, make the edit. Use `Edit` for in-place changes to CLAUDE.md / commands / skill files. Use `Read` then `Edit` for settings JSON (preserve formatting). For "apply N — but <tweak>", incorporate the tweak.
- For permissions specifically: add to the `permissions.allow` array in `~/.claude/settings.json` (create the file with `{"permissions": {"allow": [...]}}` if absent). Never touch `permissions.deny` without explicit instruction.
- If "skip" or no items approved, do nothing in Phase 2 — but still re-check Phase 3 triggers afterward (the file may already be over the size threshold, or Phase 2 may have surfaced ≥3 drift items). Skip in Phase 2 does NOT short-circuit Phase 3.

### Zero-proposal case
If the session surfaced no real optimizations, say so: `Phase 2: no optimizations — nothing in this session warranted touching existing config.` Skip the approval prompt entirely.

---

## Phase 3: Compress (codebase-grounded leanness pass)

Phase 3 is OPPOSITE to Phase 2 in grounding: Phase 2 asks "did this come up this session?"; Phase 3 asks "is this still true about the codebase?". Same approval gate as Phase 2. Same per-item rigor. Knowledge preservation always beats line savings.

### When Phase 3 runs

Run Phase 3 if ANY of these are true:
- `~/.claude/CLAUDE.md` exceeds 250 lines
- `/reflect project` mode AND project `CLAUDE.md` exceeds 300 lines
- Explicit `/reflect compress` (in this case, skip Phases 1 and 2; run only Phase 3)
- Phase 2 surfaced ≥3 items in the Contradictions / Redundancy / Stale entries buckets, regardless of whether the user applied them (signals the file has drifted)

**When to measure size:** after Phase 2 has finished applying (or after Phase 2's zero-proposal case is printed). Phase 1 only appends, and Phase 2 can both add and remove, so the post-Phase-2 file state is the correct baseline. In `/reflect compress` mode (Phases 1+2 skipped), measure once at the start of Phase 3.

Otherwise: skip Phase 3 entirely. No output, no mention.

### Overlap with Phase 2 "Stale entries"
Phase 2 Bucket 3 (Stale entries) catches stale items the session happened to touch. Phase 3 Bucket 3 (Stale by reference) catches stale items via codebase scan. If Phase 2 already proposed (or applied) a stale-entry fix on a line, do NOT re-propose the same line in Phase 3 — track which lines Phase 2 touched and skip them.

### Files in scope
- Always: `~/.claude/CLAUDE.md`
- `/reflect project` mode: also project `CLAUDE.md` and `.claude.local.md` (if they exist in cwd)

### Baseline measurement
Before proposing anything, capture and print baseline for each file in scope:
```
Baseline:
  ~/.claude/CLAUDE.md = <N> lines / <M> bytes
  ./CLAUDE.md = <N> lines / <M> bytes   (only in /reflect project mode)
```

### Compression buckets (each proposal slots into exactly one)

1. **Verbose** — multi-sentence rules that flatten to one line. Model: a 4-sentence "JWT is an open standard (RFC 7519)..." paragraph compresses to "Auth: JWT with HS256, tokens in `Authorization: Bearer <token>` header."
2. **Duplicate within file** — two lines saying the same thing under different headers. Keep one canonical line.
3. **Stale by reference** — a still-live rule contains a reference (file path / command / flag / tool / skill name) that no longer exists. **Verify with Bash/Read before flagging.** Compress by rewriting the rule without the dead reference, or by merging it with a still-valid rule. If the dead reference IS the entire rule (nothing left after removing it), do NOT auto-delete — FLAG it back to the user with "this entire rule references something that no longer exists; should it be removed?" That decision belongs to the user, not Phase 3.
4. **Historical noise** — "previously X / now Y" patterns where only Y is the current rule. Drop the X half; keep Y.
5. **Cross-file duplicate** — same rule in global AND project. Propose keeping it in the more specific location (project wins by default) and removing from the other.
6. **Hierarchy mismatch** — only flag when there is an explicit textual signal in the rule itself:
   - **Demote** a global rule when its text references a specific project name, repo path, tool used only in one location, or domain that doesn't span Hans's other projects (e.g., "for sprout-seo: …", "in the hvg-editor repo: …"). Do NOT demote based on guessing whether it generalizes.
   - **Lift** a project rule to global only when the rule contains no project-specific reference AND Hans's existing global CLAUDE.md already covers similar terrain (suggesting it belongs there). When in doubt, leave it.

### Safety rails (apply to every proposal)
- **NEVER auto-delete a live rule** — only rephrase, merge, or relocate. The Historical noise bucket is the one exception: the dead half of a "previously X / now Y" pair can be dropped because Y (the live rule) is preserved. For Stale by reference, the dead reference inside a live rule can be removed only if the rest of the rule survives — if the whole rule is dead, FLAG it back to the user instead of deleting.
- **If a rule's intent is unclear from text alone**: FLAG it back to the user with "what does this rule mean?" — do NOT compress a rule you do not fully understand.
- **Backup before applying**: write the existing file to `<path>.bak-YYYYMMDD` before the first Edit (use today's date from your context). One backup per file per run. After writing, list existing `<path>.bak-*` files for that path and delete the oldest until at most 10 remain (newest 10 by date suffix).

### Proposal format

Same shape as Phase 2, with bucket tag and savings estimate:

```
**N. <short title>** [bucket: verbose | duplicate | stale | historical | cross-file | hierarchy]
File: <path>
Issue: <what's wrong, grounded in file contents or codebase state>
Change: <before/after diff>
Why: <one line>
Saves: ~<N> lines
```

After the list, end with EXACTLY this prompt and stop:

```
Reply with: "apply 1,3,5" / "apply all" / "skip" / "apply N — but <tweak>"
```

### Applying

When the user responds:
- Back up each file in scope to `<path>.bak-YYYYMMDD` BEFORE the first Edit.
- Apply approved items using Edit.
- Re-measure line/byte count for each touched file.
- Output the closing block (see Output shape).

### Zero-proposal case
If Phase 3 ran but found nothing worth compressing: `Phase 3: no compression — files are already lean.` Skip the approval prompt.

---

## Output shape

### After Phase 1 (print before starting Phase 2 analysis)

```
**/reflect — Phase 1 (saved)**

Durable preferences → ~/.claude/CLAUDE.md:
- <bullet> ...
(or "nothing")

Standing instructions → ~/.claude/CLAUDE.md:
- <bullet> — why: <one line>
(or "nothing")

New skill:
- <name> at ~/.claude/skills/<name>/SKILL.md — <one-line purpose>
(or "none — nothing cleared the rerun bar")

Dropped (ephemeral / project-specific):
- <bullet> ...
(or "nothing material")
```

### After Phase 2 analysis (this turn ENDS here, waiting for user)

```
**/reflect — Phase 2 (proposals)**

1. ...
2. ...
3. ...

Reply with: "apply 1,3,5" / "apply all" / "skip" / "apply N — but <tweak>"
```

OR, if no proposals:

```
**/reflect — Phase 2:** no optimizations — nothing in this session warranted touching existing config.
```

In the zero-proposal case, do NOT end the turn yet — re-check Phase 3 triggers (using the current post-Phase-1 file size) and continue into Phase 3 in the same turn if any trigger fires. If no Phase 3 trigger fires either, end the turn with `Correct anything you want me to undo.`

### After user approval of Phase 2 (next turn)

```
**/reflect — applied:**
- #N: <one line on what changed>
- #M: <one line>
(or "nothing applied")
```

After applying, re-check Phase 3 triggers (using post-application file size). If any trigger fires, continue immediately into Phase 3 proposals in the same turn. Otherwise end with: `Correct anything you want me to undo.`

### After Phase 3 analysis (this turn ENDS here, waiting for user)

```
**/reflect — Phase 3 (compress proposals)**

Baseline:
  <path> = <N> lines / <M> bytes
  ...

1. **<title>** [bucket: <bucket>]
   File: ...
   Issue: ...
   Change: ...
   Why: ...
   Saves: ~<N> lines
2. ...

Reply with: "apply 1,3,5" / "apply all" / "skip" / "apply N — but <tweak>"
```

OR, if Phase 3 ran but found nothing:

```
**/reflect — Phase 3:** no compression — files are already lean.
```

### After user approval of Phase 3 (next turn)

```
**/reflect — compressed:**
  <path>: <before> → <after> lines (<-N%>)
  ...
- #N: <one line on what changed>
- #M: <one line>
Backup: <path>.bak-YYYYMMDD
```

Then a single closing line: `Correct anything you want me to undo.`

No commentary, no justifications, no follow-up questions outside the approval prompts.
