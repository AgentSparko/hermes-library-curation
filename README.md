# Hermes Library Curation

A [Hermes Agent](https://hermes-agent.nousresearch.com) skill: a verified end-to-end
playbook for consolidating a growing skill library (via the Curator) and optimizing
bounded long-term memory — the exact loop of pre-flight, consolidation passes,
post-run audit, and memory eviction.

**Contents**
- `SKILL.md` — the skill itself (Hermes skill format, agentskills.io compatible)
- `scripts/sync.sh` — refresh this mirror from your live skill directory
- `LICENSE` — MIT

**Install** (copy into your Hermes skills dir):
```bash
mkdir -p ~/.hermes/skills/devops/hermes-library-curation
cp SKILL.md ~/.hermes/skills/devops/hermes-library-curation/SKILL.md
```
Or install an entire folder as an external skill directory via Hermes config.

**Prerequisites:** Hermes Agent with the built-in `curator` + `memory` tools
(default on any current install). No external services; all commands are local
`hermes` CLI + shell. The LLM consolidation pass routes through your configured
`auxiliary.curator` model — see SKILL.md Phase 0 for health-checking it.

**Usage:** load with `skill_view(hermes-library-curation)`, or invoke as
`/hermes-library-curation` once installed. Trigger it when skills pile up or
memory writes start hitting the char-limit error.

**External prerequisite for --consolidate:** an LLM endpoint for the aux slot
(any provider you already configured — works fully local). The prune-only
default and all Phase 2/3 steps need no model at all.
