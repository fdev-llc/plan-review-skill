# plan-review

Automated Claude Code <-> Codex plan review loop. Sends your implementation plan to OpenAI Codex for independent structured review, incorporates feedback, revises the plan, and repeats until Codex approves.

## Install

```bash
npx skills add fdev-llc/plan-review-skill -g
```

## Prerequisites

1. **OpenAI Codex CLI** (v0.75.0+):
   ```bash
   npm i -g @openai/codex
   codex login --api-key "your-openai-api-key"
   ```

2. **Register Codex as an MCP server** in Claude Code:
   ```bash
   claude mcp add codex-cli -s user -- codex mcp-server
   ```

3. **(Optional)** Auto-approve Codex MCP calls by adding `"mcp__codex-cli__*"` to `permissions.allow` in `~/.claude/settings.local.json`.

## Usage

In Claude Code:

```
/plan-review
```

Or with options:

```
/plan-review focus=security max_rounds=3
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| max_rounds | 5 | Safety cap for review rounds |
| plan_path | auto-detect | Path to the plan file |
| focus | general | One of: general, architecture, edge-cases, security, performance |

## How It Works

1. Detects your current/latest plan file from `~/.claude/plans/`
2. Sends it to Codex via MCP for structured review (completeness, correctness, risk, ordering, clarity)
3. Codex responds with `VERDICT: APPROVED` or `VERDICT: NEEDS REVISION` with specific issues
4. If revisions needed, Claude addresses the feedback and resubmits
5. Loop continues until Codex approves or safety cap is reached

## License

MIT
