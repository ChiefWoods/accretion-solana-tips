# accretion-solana-tips

Agent skill compiling [100 Daily Solana Tips](https://accretion.xyz/blog/100-solana-tips) by Accretion Labs.

## Structure

[SKILL.md](SKILL.md)  
references/  
├── [program-design.md](references/program-design.md)  
├── [accounts.md](references/accounts.md)  
├── [security.md](references/security.md)  
├── [tokens-math.md](references/tokens-math.md)  
├── [runtime.md](references/runtime.md)  
├── [ops-testing.md](references/ops-testing.md)  
└── [checklist.md](references/checklist.md)

## Install

Install it with `npx skills add ChiefWoods/accretion-solana-tips`.

Alternatively, copy or symlink this repo into your agent's skills directory as `accretion-solana-tips`:

| Agent | Project | Personal |
|---|---|---|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| OpenAI Codex | `.agents/skills/` | `~/.agents/skills/` |
| GitHub Copilot | `.github/skills/` or `.agents/skills/` | `~/.copilot/skills/` or `~/.agents/skills/` |
| Gemini CLI | `.gemini/skills/` or `.agents/skills/` | `~/.gemini/skills/` |
| Cursor | `.cursor/skills/` or `.agents/skills/` | `~/.cursor/skills/` or `~/.agents/skills/` |
| Windsurf | `.windsurf/skills/` | `~/.codeium/windsurf/skills/` |
| Cline | `.cline/skills/` or `.claude/skills/` | `~/.cline/skills/` |
| OpenCode | `.opencode/skills/` or `.agents/skills/` | `~/.config/opencode/skills/` |

From this repo:

```bash
mkdir -p ~/.agents/skills/accretion-solana-tips
cp SKILL.md ~/.agents/skills/accretion-solana-tips/
cp -R references ~/.agents/skills/accretion-solana-tips/
# and/or ~/.claude/skills, ~/.cursor/skills, etc.
```

## Credits

Original tweets by [Robert Reith (r0bre)](https://x.com/r0bre), compiled from [100 Daily Solana Tips](https://accretion.xyz/blog/100-solana-tips) on the [Accretion](https://accretion.xyz/) blog.
