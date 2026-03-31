---
name: plan-review
description: "Automated plan review loop using Codex. Sends the current plan to Codex for independent review, incorporates feedback, and iterates until Codex approves the plan. Use when the user says /plan-review, asks to review a plan with Codex, wants a second opinion, or says 'refine this plan'."
---

# Plan Review Loop

Automated Claude <-> Codex review loop. Sends a plan to Codex for independent structured review, incorporates feedback, revises the plan, and **repeats until Codex approves** (or a safety cap is reached).

## Prerequisites

Before using this skill, you need:

1. **OpenAI Codex CLI** (v0.75.0+):
   ```bash
   npm i -g @openai/codex
   codex login --api-key "your-openai-api-key"
   ```

2. **Register Codex as an MCP server** in Claude Code:
   ```bash
   claude mcp add codex-cli -s user -- codex mcp-server
   ```

3. **(Optional) Auto-approve Codex MCP calls** -- add `"mcp__codex-cli__*"` to your `permissions.allow` in `~/.claude/settings.local.json` to avoid permission prompts during the review loop.

## When to Use

Activate when the user:
- Says `/plan-review`
- Asks to "review this plan with Codex" or "get Codex feedback"
- Asks for a "second opinion" on a plan
- Says "refine this plan" or "run the review loop"

## Parameters

Parse from the user's message. Use defaults if not specified:

| Parameter | Default | Description |
|-----------|---------|-------------|
| max_rounds | 5 | Safety cap -- stop even if not approved after this many rounds |
| plan_path | auto-detect | Path to the plan file |
| focus | general | One of: general, architecture, edge-cases, security, performance |

The loop is **convergence-based by default**: it runs until Codex signals approval, NOT for a fixed number of iterations. The `max_rounds` parameter is only a safety limit.

## Execution Steps

### Step 0: Detect Plan File

1. If the user specified a path, use it
2. If currently in planning mode with an active plan file, use that path
3. Otherwise, find the most recent plan:
   ```bash
   ls -t ~/.claude/plans/*.md | head -1
   ```
4. Read the plan file. If empty or missing, inform the user and stop

### Step 1: Detect Transport Method

Try the Codex MCP tools first (the `codex-cli` MCP server). If MCP tools are not available or fail, fall back to calling `/opt/homebrew/bin/codex exec` via Bash.

### Step 2: Review Loop (repeats until APPROVED or max_rounds reached)

#### 2a. Build the Review Prompt

For **round 1**, construct this prompt (filling in `{focus}` and `{plan_content}`):

```
You are an expert plan reviewer. Your job is to critically review a software implementation plan created by another AI assistant and either APPROVE it or request REVISIONS.

Focus area: {focus}

Review Criteria:
1. Completeness: Are there missing steps, edge cases, or dependencies?
2. Correctness: Are the technical approaches sound? Any incorrect assumptions?
3. Risk: What could go wrong? What are the failure modes?
4. Ordering: Are the steps in the right sequence? Any dependency issues?
5. Clarity: Is the plan clear enough to implement without ambiguity?

The Plan to Review:

{plan_content}

RESPONSE FORMAT -- you MUST follow this exactly:

First, provide your verdict on a single line:
VERDICT: APPROVED
or
VERDICT: NEEDS REVISION

If APPROVED: Briefly explain why the plan is solid (2-3 sentences). You may still include minor suggestions prefixed with "Optional suggestion:" but these do not block approval.

If NEEDS REVISION: List each issue as:
- [CRITICAL] / [MAJOR] / [MINOR] Category: Issue description -> Recommendation
Group by severity. Be specific about what needs to change and where.

Only use NEEDS REVISION if there are genuinely important issues (critical or major). Minor-only issues should still result in APPROVED with optional suggestions.
Do NOT rewrite the plan. Only provide feedback.
```

For **rounds 2+**, use this follow-up prompt:

```
You previously reviewed a plan and requested revisions. The plan author has revised the plan based on your feedback. Please re-review.

Your previous feedback:
{previous_codex_feedback}

The Revised Plan:
{revised_plan_content}

Check:
1. Were your previous critical/major issues adequately addressed?
2. Did the revisions introduce any NEW critical or major issues?
3. Is the plan now ready for implementation?

RESPONSE FORMAT -- you MUST follow this exactly:

First line must be:
VERDICT: APPROVED
or
VERDICT: NEEDS REVISION

Same format as before. If all critical/major issues are resolved and no new ones were introduced, APPROVE the plan. Do not keep requesting revisions for diminishing returns -- approve when the plan is solid enough to implement well.
Do NOT rewrite the plan. Only provide feedback.
```

#### 2b. Send to Codex

**MCP method (preferred):** Call the `codex-cli` MCP server's tool with the review prompt. If the server exposes a session/conversation mechanism, reuse the same session across rounds for context continuity.

**Bash fallback:** Write the prompt to a temp file and pipe it:
```bash
cat /tmp/plan-review-prompt.txt | /opt/homebrew/bin/codex exec - \
  --skip-git-repo-check \
  -o /tmp/codex-review-output.txt \
  --ephemeral \
  2>/dev/null
```
Then read `/tmp/codex-review-output.txt` for the response.

For rounds 2+ with Bash, if you captured a session/thread ID from round 1's output, use `codex exec resume <id>` for context continuity. If resume fails, include the previous feedback in the prompt (already built into the round 2+ template).

IMPORTANT: Set the Bash timeout to 300000 (5 minutes) for Codex calls, as reviews of large plans can take time.

#### 2c. Parse the Verdict

Look for `VERDICT: APPROVED` or `VERDICT: NEEDS REVISION` in Codex's response.

- If **APPROVED**: proceed to Step 3 (Final Summary). The loop is done.
- If **NEEDS REVISION**: continue to 2d.
- If **no clear verdict found**: treat the response as NEEDS REVISION if it contains issue listings, or ask Codex to clarify its verdict in the next round.

#### 2d. Present Feedback to User

Show a concise summary:
```
Codex Review (Round {N}): NEEDS REVISION -- {X} issues ({Y} critical, {Z} major, {W} minor)
```

Then list the issues grouped by severity. Do NOT hide any critical or major issues.

#### 2e. Revise the Plan

Address the feedback:
- **Critical issues**: Must fix
- **Major issues**: Should fix
- **Minor issues**: Fix if straightforward
- **Suggestions**: Incorporate the good ones

Write the revised plan back to the same plan file. Show the user a brief summary of what changed:
```
Revised: added X, reordered Y and Z, fixed assumption about W
```

Then go back to **Step 2a** with the next round number.

#### 2f. Safety Cap Reached

If `max_rounds` is reached without approval:
- Inform the user: "Reached {max_rounds} review rounds without full approval from Codex."
- Show the remaining open issues from the last round
- Ask the user if they want to continue for more rounds, or accept the plan as-is

### Step 3: Final Summary

Present:

1. **Verdict**: "Plan APPROVED by Codex after {N} round(s)." or "Stopped after {N} rounds (safety cap)."
2. **Round-by-round summary**: "Round 1: 5 issues (2 critical, 2 major, 1 minor) -> revised. Round 2: 1 issue (1 minor) -> APPROVED."
3. Confirm the final plan location
4. If there were optional suggestions in the approval, list them for the user's consideration
5. Note any issues that were intentionally deferred or disagreed with, and why

## Error Handling

- **MCP tool fails**: Fall back to Bash method for that iteration
- **Codex timeout**: Retry once. If it fails again, present what was accomplished so far
- **Plan file too large** (>15K words): Warn the user; consider reviewing section-by-section
- **Session resume fails**: Start a fresh session with previous feedback included in prompt context
- **User interrupts**: The plan is saved after each revision, so progress is never lost
- **Codex keeps requesting revisions on trivial issues**: If round 3+ and only minor issues remain, Claude may judge the plan sufficient and present the remaining minors to the user as optional improvements

## Notes

- The convergence approach is smarter than fixed iterations: simple plans may be approved in 1 round, complex plans may need 4-5
- For security-sensitive plans, use `focus=security` to prioritize threat modeling
- The review prompt deliberately asks Codex NOT to rewrite the plan -- only provide feedback. This keeps Claude in control of the actual revisions and prevents style/context loss
- The `VERDICT:` line makes parsing deterministic -- no ambiguity about whether Codex approved or not
