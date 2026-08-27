# einfachllm-skills

Agent skills for [einfachllm](https://github.com/einfachllm/einfachllm), an
EU-sovereign, OpenAI-compatible LLM gateway.

A skill is a folder with a `SKILL.md` and supporting docs. The agent reads
`SKILL.md` to decide *whether* a skill applies and *which* action to take, then
loads only the referenced doc — so the detail stays out of context until it is
needed.

## Skills

| Skill | Use it for |
| --- | --- |
| [`einfachllm`](skills/einfachllm/) | Managing a **running** gateway: API keys, routing config, tenancy sync, health, usage |
| [`einfachllm-ops`](skills/einfachllm-ops/) | **Deploying and operating** it: Compose/Helm installs, migrations, upgrades and rollbacks, backup/restore, hardening |

## Layout

```
skills/
  <skill-name>/
    SKILL.md        # frontmatter (name, description) + action table
    docs/*.md       # one doc per action group, loaded on demand
```

Nothing here is generated. Edit the Markdown directly.

## Install

Copy the skill folder into wherever your agent looks for skills:

```bash
git clone https://github.com/einfachllm/einfachllm-skills
cp -r einfachllm-skills/skills/einfachllm ~/.claude/skills/
cp -r einfachllm-skills/skills/einfachllm-ops ~/.claude/skills/
```

For a project-local install, use `.claude/skills/` in the repository instead.

## The gateway checkout

Both skills drive `ectl.py`, the gateway's admin CLI. It ships **with the
gateway, not with this repo** — it lives at
[`tools/ectl.py`](https://github.com/einfachllm/einfachllm/blob/main/tools/ectl.py)
and is baked into the container image at `/app/ectl.py`. Every command in these
skills is written to run from the root of an einfachllm checkout:

```bash
uv run python tools/ectl.py <action>
```

Two actions (`config-validate`, `tenancy-sync`) import the project for schema
validation and therefore need that checkout or the image; the rest are HTTP
against the admin API. See
[`skills/einfachllm/docs/setup.md`](skills/einfachllm/docs/setup.md).

## Keeping these in sync with the gateway

These skills describe a moving target — chart values, admin endpoints, and CLI
actions all change in the gateway repo. When a change lands there, check
whether it invalidates:

- an action table entry or a command in `SKILL.md`,
- a Helm value name or default in `einfachllm-ops/docs/`,
- an admin endpoint or route in `einfachllm/docs/`.

A skill that confidently states something untrue is worse than a missing skill.

## License

Apache-2.0. See [LICENSE](LICENSE).
