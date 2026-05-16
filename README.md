# plan-review

Automated Claude Code <-> Codex plan review loop, grounded in your real codebase. Sends your implementation plan to OpenAI Codex for an independent structured review -- Codex inspects the actual project files (read-only) to verify the plan's assumptions -- then incorporates the feedback, revises the plan, and repeats until Codex approves.

A plan-text-only review can only judge a plan's internal logic. By giving Codex read access to the project, this loop also catches hallucinated file paths, wrong assumptions about existing code, missed reuse opportunities, and approaches that clash with the current architecture.

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
| max_rounds | 10 | Safety cap — stops the loop even if Codex has not approved yet (not a fixed round count) |
| plan_path | auto-detect | Path to the plan file |
| project_root | auto-detect | Absolute path of the project Codex inspects (working directory, or Git repo root) |
| focus | general | One of: general, architecture, edge-cases, security, performance |
| review_codebase | true | `true`: Codex inspects the real project files. `false`: plan-text-only review (greenfield plans, or a pure logic check) |
| model | gpt-5.5 | Codex model used for the review |
| reasoning_effort | xhigh | Codex reasoning effort: minimal, low, medium, high, or xhigh (deepest) |

The loop is **convergence-based**: it runs until Codex approves the plan, not for a fixed number of rounds. `max_rounds` is only a safety limit — simple plans may be approved in one round.

## How It Works

1. Detects your current/latest plan file from `~/.claude/plans/` and the project root
2. Sends the plan to Codex via MCP, with read-only access to the project so Codex can check it against the actual code
3. Codex inspects the relevant files, then returns `VERDICT: APPROVED` or `VERDICT: NEEDS REVISION` with specific issues (codebase grounding, completeness, correctness, risk, ordering, clarity)
4. If revisions are needed, Claude addresses the feedback and resubmits
5. The review session persists across rounds, so Codex reuses its codebase exploration instead of repeating it
6. The loop continues until Codex approves or the safety cap is reached

## License

MIT
