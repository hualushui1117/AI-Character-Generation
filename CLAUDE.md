# CLAUDE.md

## First Things To Read

- `SKILL.md` — complete workflow reference (image gen → video gen → transparent bg)
- `docs/workflow-image-generation.md` — image generation design details
- `docs/workflow-video-generation.md` — video generation design details

## Required Setup

```bash
npm install -g pixverse   # Node.js >= 20
pixverse auth login       # OAuth device flow
ffmpeg                    # required only for transparent background post-processing
```

## Entry Point

User triggers this skill by running `/角色生成` or describing a character they want to generate.
Read `SKILL.md` in full before starting.

## Slash Command

The skill is also available as a Claude Code slash command:
`.claude/commands/角色生成.md`

## Tool Notes

- Always use `--json` flag for every `pixverse` call
- Use `Bash` tool to run `pixverse` and `ffmpeg` commands
- Use `WebFetch` to fetch PixVerse upstream skill docs in Phase 0
