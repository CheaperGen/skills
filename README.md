# Cheaper Gen. skills

Agent Skills and a Claude Code plugin for [Cheaper Gen.](https://cheapergen.com), one API for official ByteDance Seedance and Seedream, Alibaba Wan and MiniMax video and image models at prices below the official list.

The repo has two parts:

- **The `cheapergen` skill** ([`skills/cheapergen/SKILL.md`](skills/cheapergen/SKILL.md)) teaches an agent the Cheaper Gen. REST API. It covers choosing a model, reading its input schema, quoting, submitting, polling or webhooks, `@Image1`-style reference mentions, uploads, pricing and errors. It works in any agent that supports [Agent Skills](https://agentskills.io).
- **The Claude Code plugin `cheapergen`** bundles that skill with the hosted Cheaper Gen. MCP server, `https://cheapergen.com/api/mcp`. The server's tools are `list_models`, `get_model`, `quote`, `generate`, `get_generation` and `get_account`. You sign in with your Cheaper Gen. account in the browser, so there's no API key to paste.

## Install

### Claude Code plugin (skill + MCP server)

```
/plugin marketplace add CheaperGen/skills
/plugin install cheapergen@cheapergen
```

Then run `/mcp`, select **plugin:cheapergen:cheapergen** and choose **Authenticate**. Sign in at cheapergen.com and approve the connection. From a terminal, the same steps are:

```bash
claude plugin marketplace add CheaperGen/skills
claude plugin install cheapergen@cheapergen
```

### Skills CLI (Claude Code, Cursor, Codex, Gemini CLI and others)

```bash
npx skills add CheaperGen/skills
```

Add `-g` to install for all projects, `-a claude-code` (or `-a cursor`, `-a codex`…) to choose the agent, and `--skill cheapergen` to skip the prompt. This installs only the skill. To add the MCP server too, see [MCP setup](https://cheapergen.com/docs/mcp).

### Manual

Copy the skill folder into your agent's skills directory:

```bash
git clone https://github.com/CheaperGen/skills cheapergen-skills
cp -r cheapergen-skills/skills/cheapergen ~/.claude/skills/        # all projects
cp -r cheapergen-skills/skills/cheapergen .claude/skills/          # this project only
```

Or download just the file:

```bash
mkdir -p ~/.claude/skills/cheapergen
curl -fsSL https://cheapergen.com/skills/cheapergen/SKILL.md -o ~/.claude/skills/cheapergen/SKILL.md
```

In the Claude apps (claude.ai and Claude Desktop), zip the `cheapergen` folder and upload it as a custom skill. To connect the MCP server there, add `https://cheapergen.com/api/mcp` as a custom connector under **Customize → Connectors**.

## Authentication

- **MCP server:** sign in with OAuth when your client asks. Approved apps are listed under [Dashboard → API keys → Connected apps](https://cheapergen.com/dashboard/keys), where you can disconnect them. For headless use or CI, you can instead send an API key as `Authorization: Bearer cg_live_…`.
- **REST API (what the skill teaches):** create an API key at [cheapergen.com/dashboard/keys](https://cheapergen.com/dashboard/keys) and export it as `CHEAPERGEN_API_KEY`.

Generations are charged to your prepaid balance, and failed ones are refunded automatically.

## Layout

```
.claude-plugin/marketplace.json   Claude Code marketplace "cheapergen" (one plugin, source ./)
.claude-plugin/plugin.json        plugin manifest + the remote MCP server
skills/cheapergen/SKILL.md        the Agent Skill
```

## Links

- Docs: https://cheapergen.com/docs
- MCP setup: https://cheapergen.com/docs/mcp
- Models and prices: https://cheapergen.com/models
- `llms.txt`: https://cheapergen.com/llms.txt
- Support: support@cheapergen.com

## License

MIT
