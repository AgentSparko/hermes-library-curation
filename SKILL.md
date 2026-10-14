---
name: hermes-library-curation
description: "Audit and consolidate Hermes skills + memory end to end."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, skills, memory, curator, maintenance]
    related_skills: [hermes-official-docs]
---

# Hermes library curation — skills + memory consolidation

Use when the skill library has grown unwieldy (near-duplicate, stale, oversized skills) or when MEMORY.md/USER.md approach their character caps. Full loop: pre-flight → curator consolidation passes → post-run audit → memory eviction. Verified end-to-end on a 160+ skill library, 2026-10.

## When to Use
- User asks to "consolidate/clean up skills and memory"
- `hermes curator status` shows a large stale count or `hermes journey list` looks crowded
- Memory writes start failing with the "at X/Y chars" overflow error

## Phase 0 — Pre-flight (never skip)
1. `hermes curator status` — note managed vs unmanaged counts. **Only agent-created skills are curated**; foreground-created (`created_by: learn`), bundled, and hub skills are invisible to it. Unmanaged skills you want curated: `hermes curator adopt <name>` (explicit, refuses bundled/hub).
2. Snapshot: `hermes curator backup --reason "before-consolidate-<date>"` (the run takes its own snapshot too; this is a second net).
3. Pin anything never to be touched: `hermes curator pin <skill>` (agent-created only; pin blocks curator archive + `skill_manage delete`, edits still allowed).
4. **Verify the aux model slot resolves to a LIVE engine.** The LLM pass uses `auxiliary.curator` (default `provider: auto` = the main chat model). A dead/unreachable chat lane makes the pass die with `"custom didn't respond in time on any of 3 attempts"` after ~5 min of silent retries — check lanes first (`curl -m3 http://<engine>/health`, `hermes config get auxiliary.curator`). Pin a live one: `hermes config set auxiliary.curator.provider <name>` + `...auxiliary.curator.model <served-model-id>`, then confirm with `GET <base_url>/models`.
5. Dry-run: `hermes curator run --dry-run --consolidate` — takes ~5–10 min (22+ skill_view calls). Read `~/.hermes/logs/curator/<timestamp>/REPORT.md`. Expect: cluster plans with `from → into` pairs, a keep-list, and a reviewer note flagging the riskiest steps.

## Phase 1 — Consolidation passes
- Real run: `hermes curator run --consolidate` (long: 20–30 min, 40–100 API calls). Run it **backgrounded with notification**, never as a foreground guess-and-timeout.
- A single pass may stop at its iteration ceiling having done 2 of 4 planned clusters — **re-run until the report shows no new archives**. Later passes re-cluster with fresh judgment and may absorb skills the first pass merely kept (that's fine).
- `--consolidate` overrides the default-off `curator.consolidate: false` for one run only; leave the config default off unless you want it every 7 days.

## Phase 2 — Post-run audit (mandatory, catches the real bugs)
The LLM pass can leave dangling cross-skill pointers. Sweep live skills for references into archived names:
```bash
pat=$(ls ~/.hermes/skills/.archive/ | paste -sd'|')
grep -rn --include='*.md' -E "$pat" ~/.hermes/skills \
  --exclude-dir=.archive --exclude-dir=.curator_backups -l
```
Review each hit: historical narration ("absorbed from X") = benign; `related_skills: [archived]` or "see X skill" pointers = **fix by patch** (repoint to the umbrella or make the reference internal). Also spot-read one absorbed umbrella: confirm its SKILL.md still holds the durable rules (standing user rules are the thing that must survive; prose is lossy by design).
- Final snapshot: `hermes curator backup --reason "post-consolidate-final"`.
- Pin the umbrellas the pass created so nothing re-merges them away.
- Reports live in `~/.hermes/logs/curator/<run>/REPORT.md`; full record in `run.json`; every mutation is in the ledger (`hermes curator ledger`). Undo one run: `hermes curator rollback -y`; undo one edit: `hermes curator rollback <entry-id>`; un-archive: `hermes curator restore <skill>`.

## Phase 3 — Memory optimization
Docs rule of thumb: **memory = facts ("what"), skills = procedures ("how"), AGENTS.md = standing instructions, session_search = history.** Memory's ~2,200 + ~1,375 chars ride in EVERY prompt — eviction, not expansion, is the win.
1. Audit each entry against those 4 homes. Typical evictions: per-project port maps → that project's skill/AGENTS.md; "never mention X during benchmarks"-style task etiquette → the matching task skill, as a `references/` file in it; tool-quirk lessons → their own small skill; **transient state ("lane is dead", container status) → just DELETE** — it goes stale within days and a live probe beats a stale note (one such entry was wrong when audited).
2. Duplicate of AGENTS.md content → delete from memory (see Pitfall 6 before hardening into AGENTS.md).
3. Merge the surviving engine/routing facts into ONE entry. Target: **<60% full**.
4. Batching: when full, `memory` refuses adds — issue ONE `operations` batch (remove + replace + add) against the final-size budget. `replace` swaps the WHOLE matched entry: pass the complete new entry text, not a patch.
5. Optional headroom: `hermes config set memory.memory_char_limit 2600` — only if genuinely full after eviction; on long-context models the marginal token cost is trivial but eviction is free.
6. Verify by reading the stores: `cat ~/.hermes/memories/MEMORY.md USER.md` — claimed saves must be ON DISK (models sometimes confirm writes they never made).

## Pitfalls
1. Running a mutating curator pass without backup + pin + dry-run. Never `run --consolidate` cold.
2. Expecting the curator to touch your hand-made/foreground skills — it won't (by design); use `adopt` to hand over, then it participates.
3. The 24k-char SKILL.md threshold: consolidators patch umbrellas toward it; a 58k-char SKILL.md costs ~14k tokens on every load — distill topic depth to `references/`.
4. Dry-run report showing `Model: (not resolved)` — means zero curator-managed candidates (nothing to review), not a broken model. Conversely, "custom didn't respond in time" = the resolved engine was unreachable; fix the aux slot (Phase 0.4), don't retry blindly.
5. Running the LLM pass through a dead aux slot burns 5 min ×3 attempts per invocation. Always health-check the model behind `auxiliary.curator` first.
6. **AGENTS.md is a protected agent-instruction file** — writes require an interactive approval prompt; a timeout is a refusal, not a pending state, and must not be retried via another path. For messaging sessions the user may not be watching: warn them to keep the chat open during approval, and keep the safety rule in memory meanwhile (cwd-scoped AGENTS.md also doesn't follow sessions started in other directories — safety rules deserve double coverage).
7. Skill-author budget at `create` time: description ≤60 chars, YAML frontmatter required — a failed op aborts the whole batch (rollback), so validate before batching.
8. After any archive, the live skills index changes mid-session ("skills changed" notice) — old references in your own context are stale; re-`skill_view` before citing them.

## Verification Checklist
- [ ] `hermes curator status` — last run summary shows 0 new archives (converged)
- [ ] Dangling-ref sweep returns only benign historical mentions
- [ ] One absorbed umbrella spot-read: standing user rules intact
- [ ] Umbrellas pinned
- [ ] `cat ~/.hermes/memories/MEMORY.md` shows ≤~60% usage, facts survive
- [ ] Every "done" claim read back from disk
