# khaos-skills

Agent skills for the [Khaos Storage](https://khaosstorage.com) ecosystem. Tiny markdown bundles that teach AI coding agents (Claude Code, Codex, Cursor, OpenCode, et al.) when and how to drive Khaos tools.

## Install

```bash
npx skills add khaos-studio/khaos-skills
```

Or pick a single skill:

```bash
npx skills add khaos-studio/khaos-skills --skill khaos-storage
```

`skills` is the [open agent skills CLI](https://skills.sh). It clones this repo, finds every `skills/*/SKILL.md`, and symlinks the matching folder into your agent's skill directory (`~/.claude/skills/`, `~/.codex/skills/`, etc).

## What's here

| Skill | Description |
| --- | --- |
| [`khaos-storage`](skills/khaos-storage/SKILL.md) | Drive the `khs` CLI for upload, list, share, spaces, and key management against khaosstorage.com. |

More to come as the Khaos ecosystem grows (XIFty metadata extraction, SD-card ingest, etc).

## Authoring

Every skill is a folder under `skills/` with one required file: `SKILL.md`. The frontmatter tells the agent when to invoke; the body tells it how. See [vercel-labs/skills](https://github.com/vercel-labs/skills) for the spec.

## License

MIT. See [LICENSE](LICENSE).
