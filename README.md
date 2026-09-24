# Enclava Agent Skills

Agent-facing skills for the Enclava platform. The [developer docs](https://docs.enclava.dev)
([`enclava-dev-docs`](https://github.com/enclava-labs/enclava-dev-docs)) explain *what* things are, for humans reading end
to end. These
skills encode *how to operate* — procedures, contracts, and failure triage — in the open `SKILL.md` format (a folder with a YAML-frontmattered `SKILL.md`;
understood by Claude Code, Cursor, Codex, pi, and other coding agents) that coding
agents load on demand. If you deploy or manage enclava apps through an agent (Claude Code, Cursor,
Codex, pi, …), install these so the agent gets the platform's real operational
contract instead of guessing.

## Skills

| Skill | Use when |
| --- | --- |
| [`enclava/`](./enclava/) | Deploying, operating, or troubleshooting apps: manual OCI deploys, hosted templates, status/logs/config, unlock/recovery, rollback/destroy. One spine + three journey references. |
| [`enclava-verify/`](./enclava-verify/) | Independently appraising a running origin or saved proof bundle, authoring trust policies. |
| [`enclava-concepts/`](./enclava-concepts/) | Understanding the platform: TEEs, attestation, encrypted storage, who can read what, honest limits. |

## Install

Skills are plain directories with a `SKILL.md`; any agent that supports the format can
load them. For the common agents:

```bash
# Easiest: clone straight into your agent's skills directory — updates are just `git pull`.
# pi (recurses into subdirectories to find SKILL.md):
git clone https://github.com/enclava-labs/enclava-dev-skills ~/.pi/agent/skills/enclava-dev-skills

# Same idea for ~/.agents/skills/ (pi) or ~/.claude/skills/ (Claude Code, if your
# version discovers nested skills):
git clone https://github.com/enclava-labs/enclava-dev-skills ~/.agents/skills/enclava-dev-skills

# Fallback: copy the skill dirs into any skills directory
git clone https://github.com/enclava-labs/enclava-dev-skills /tmp/enclava-skills
cp -r /tmp/enclava-skills/enclava /tmp/enclava-skills/enclava-verify /tmp/enclava-skills/enclava-concepts ~/.claude/skills/
```

For agent managers with per-project skill directories, copy the skill folders into
that project's skill directory the same way.

## Conventions

- **Accuracy over prose.** Every command, flag, and failure signature here must match
  the real `enclava` CLI behavior — not just its `--help` text. Help prose and the
  human docs are useful pointers but can drift from the implementation: for
  executable claims (flags accepted, defaults applied, error-code causes, pinned
  refs), verify against actual behavior or the CLI's argument definitions/source,
  and cite that in the PR. When the CLI changes, sweep the named commands and fix
  drift as part of the release.
- **Compatibility.** Skills target the CLI version noted in each skill's frontmatter.
  Observed CLI behavior is the tiebreaker for any discrepancy.
- **Docs stay canonical for concepts** (`docs/concepts/`, `docs/reference/glossary.md` in the
  [`enclava-dev-docs`](https://github.com/enclava-labs/enclava-dev-docs) repo);
  skills carry the procedural layer and restating of command-level facts, and link out
  for depth.
- **No secrets in skills.** Skills instruct agents to keep passwords and recovery
  mnemonics out of logs, history, and unrequested files — treat that discipline as
  part of the platform's trust model, and preserve it in any edit.
- **Tenant-grade only.** Everything here works through the `enclava` CLI and public
  surfaces; no operator/cluster-internal procedures belong in these files.

## Contributing

Fixes welcome via PR. Claims must be verifiable against the behavior of the current
CLI (preferred — reproduce it), its argument definitions/source, or the developer
docs — cite which in the PR description. `--help` text alone is not sufficient for
executable claims; it has drifted from behavior before.
New failure signatures should come from a reproduced incident, not speculation.
