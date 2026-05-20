# my-claude-skills

A collection of [Claude Skills](https://www.anthropic.com/news/skills) I use
day-to-day. Each skill is a self-contained folder with a `SKILL.md` (the
entry point Claude loads) and optional `references/` for progressive
disclosure of detailed patterns.

## Skills in this repo

| Skill | What it does |
|---|---|
| [`fastapi/`](./fastapi/) | Async-first FastAPI with SQLAlchemy 2.0, Pydantic V2, PyJWT auth, ARQ background tasks, pytest-asyncio testing. |

## How to use

### Claude Code

Drop the skill folder into your project's `.claude/skills/` directory (or
your user-level skills directory at `~/.claude/skills/`):

```bash
git clone https://github.com/<your-username>/my-claude-skills.git
mkdir -p ~/.claude/skills
cp -r my-claude-skills/fastapi ~/.claude/skills/
```

Claude Code picks up skills automatically — no restart needed.

### Claude.ai / Claude Desktop

Package the skill folder into a `.skill` file (a zip with a specific layout)
and upload it via Settings → Capabilities → Skills:

```bash
cd my-claude-skills
zip -r fastapi.skill fastapi/
```

Then upload `fastapi.skill` through the UI.

### Claude API

Pass the skill contents in your system prompt or via the
[Skills feature](https://docs.claude.com/en/docs/agents-and-tools/skills)
when calling the API. See Anthropic's docs for the exact wiring.

## Skill anatomy

Every skill in this repo follows the same layout:

```
skill-name/
├── SKILL.md              # Entry point — Claude always loads this when the skill triggers
└── references/           # Optional — loaded on demand based on pointers in SKILL.md
    ├── topic-a.md
    └── topic-b.md
```

`SKILL.md` starts with YAML frontmatter (`name`, `description`, optionally
`risk`, `source`, `date_added`) followed by markdown. The `description` is
the primary triggering signal — Claude reads it to decide whether the skill
applies to the current task.

For progressive disclosure, `SKILL.md` lists reference files with a short
"read this when…" pointer. Claude loads them only when the task matches,
keeping context lean for unrelated questions.

## Contributing / forking

Fork freely. If you build something useful on top of one of these skills, a
PR is welcome but not required.

## License

[MIT](./LICENSE) — do whatever you want with these, no warranty.
