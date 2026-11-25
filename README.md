# agent-skills

Skills for coding agents (Claude Code, Codex, Cursor and others), taken from real cases in my own setup. Each skill is one `SKILL.md` file: plain instructions an agent reads and follows.

## Skills

| Skill | What it does |
|---|---|
| [docker-disk-hygiene](skills/docker-disk-hygiene/SKILL.md) | Finds what takes Docker's disk space, cleans it up with your approval (never touching volumes), and sets limits so it doesn't pile up again |

## How to use

**Any agent:** give it the link to the skill file and ask it to follow it, for example:

> Read https://github.com/Marid5/agent-skills/blob/main/skills/docker-disk-hygiene/SKILL.md and check my Docker disk usage.

**Claude Code:** copy the skill folder into `~/.claude/skills/` (all projects) or `.claude/skills/` (one project). Claude picks it up when the task matches the skill's description.

```bash
git clone https://github.com/Marid5/agent-skills.git
cp -r agent-skills/skills/docker-disk-hygiene ~/.claude/skills/
```

## License

MIT
