# CheaperGen skills

Agent Skills and a Claude Code plugin for [CheaperGen](https://cheapergen.com), one API for official video and image models from the major labs (Seedance, Kling, Veo, Wan, MiniMax, FLUX, GPT Image, Nano Banana and more) at prices below the official list.

The repo has two parts:

- **The `cheapergen` skill** ([`skills/cheapergen/SKILL.md`](skills/cheapergen/SKILL.md)) teaches an agent the CheaperGen REST API. It covers choosing a model, reading its input schema, quoting, submitting, polling or webhooks, `@Image1`-style reference mentions, uploads, pricing and errors. It works in any agent that supports [Agent Skills](https://agentskills.io).
- **The Claude Code plugin `cheapergen`** bundles that skill with the hosted CheaperGen MCP server, `https://cheapergen.com/api/mcp`. The server's tools are `list_models`, `get_model`, `quote` (it also says whether your balance covers the job), `generate`, `get_generation`, `upload_file` (turns a local photo, video or audio file into an https URL to generate from; free), `compare_prices` (price against the official price and other API platforms; free), `get_account` (balance, and the account's API limits) and `add_credits` (a payment link the user opens when the balance is low). Finished images come back with a small inline preview in the chat. You sign in with your CheaperGen account in the browser, so there's no API key to paste.
- **The Claude Code plugin `cheapergen-dev`** ([`cheapergen-dev/`](cheapergen-dev)) is for developers integrating the API: the `cheapergen-api` skill ([`cheapergen-dev/skills/cheapergen-api/SKILL.md`](cheapergen-dev/skills/cheapergen-api/SKILL.md): API keys, endpoints, quotes, polling and webhooks, errors) plus the read-only CheaperGen Developer MCP server, `https://cheapergen.com/api/mcp/dev`. Its tools are `list_models`, `get_model`, `quote` and `compare_prices` (CheaperGen's live price against the official price and other API platforms, with the date each was checked and its source), with no sign-in; connected with an API key it also has `get_account` and quotes your own prices. It never generates media, spends credits or creates payments.

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

### Claude Code plugin for developers (API skill + read-only MCP server)

```
/plugin marketplace add CheaperGen/skills
/plugin install cheapergen-dev@cheapergen
```

It works right away, without sign-in: it looks up models, prices, quotes and price comparisons. Your code generates through the REST API with an API key. To also see your own prices and balance in the connection, add it with that key instead:

```
claude mcp add --transport http cheapergen-dev https://cheapergen.com/api/mcp/dev --header "Authorization: Bearer $CHEAPERGEN_API_KEY"
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

In the Claude apps (claude.ai and Claude Desktop), zip the `cheapergen` folder and upload it as a custom skill. To connect the MCP server there, [open this link](https://claude.ai/customize/connectors?modal=add-custom-connector&connectorName=CheaperGen&connectorUrl=https%3A%2F%2Fcheapergen.com%2Fapi%2Fmcp): it opens the custom connector dialog with CheaperGen filled in: choose **Continue**, then **Add**, then **Connect** and sign in. On Team and Enterprise, an owner uses [the organization link](https://claude.ai/admin-settings/connectors?modal=add-custom-connector&connectorName=CheaperGen&connectorUrl=https%3A%2F%2Fcheapergen.com%2Fapi%2Fmcp) and members then choose Connect (or switch to a personal account). Or add `https://cheapergen.com/api/mcp` as a custom connector under **Customize → Connectors** yourself. Every other client is on [cheapergen.com/agents](https://cheapergen.com/agents).

## Authentication

- **MCP server:** sign in with OAuth when your client asks. Approved apps are listed under [Dashboard → API keys → Connected apps](https://cheapergen.com/dashboard/keys), where you can disconnect them. For headless use or CI, you can instead send an API key as `Authorization: Bearer cg_live_…`.
- **REST API (what the skill teaches):** create an API key at [cheapergen.com/dashboard/keys](https://cheapergen.com/dashboard/keys) and export it as `CHEAPERGEN_API_KEY`.

Generations are charged to your prepaid balance, and failed ones are refunded automatically.

## Layout

```
.claude-plugin/marketplace.json                 Claude Code marketplace "cheapergen" (plugins cheapergen, source ./, and cheapergen-dev)
.claude-plugin/plugin.json                      plugin manifest + the remote MCP server
skills/cheapergen/SKILL.md                      the Agent Skill
cheapergen-dev/.claude-plugin/plugin.json       developer plugin manifest + the read-only MCP server (/api/mcp/dev)
cheapergen-dev/skills/cheapergen-api/SKILL.md   the API integration skill
```

Installed plugins update only when the `version` in their `plugin.json` changes, so every change to these files ships with a version bump (both plugins carry the same version). To get the latest version, run `claude plugin update cheapergen@cheapergen`, or turn on **Enable auto-update** for the `cheapergen` marketplace under `/plugin` → **Marketplaces** (marketplaces outside Anthropic's don't auto-update by default).

## Links

- Docs: https://cheapergen.com/docs
- MCP setup: https://cheapergen.com/docs/mcp
- Models and prices: https://cheapergen.com/models
- `llms.txt`: https://cheapergen.com/llms.txt
- Support: support@cheapergen.com

## License

MIT
