---
name: plan-review
description: "Automated plan review loop using Codex, grounded in the real codebase. Sends a plan to Codex for an independent structured review -- Codex inspects the actual project files (read-only) to verify the plan's assumptions -- then incorporates feedback and iterates until Codex approves the plan. Use when the user says /plan-review, asks to review a plan with Codex, wants a second opinion, or says 'refine this plan'."
---

# Plan Review Loop

Automated Claude <-> Codex review loop. Sends a plan to Codex for an independent, structured review **grounded in the actual codebase** -- Codex reads the real project files (read-only) to check whether the plan's assumptions, file paths, and approach match reality -- incorporates the feedback, revises the plan, and **repeats until Codex approves** (or a safety cap is reached).

A plan-text-only review can only judge a plan's internal logic. By giving Codex read access to the project, this loop also catches hallucinated file paths, wrong assumptions about existing code, missed reuse opportunities, and approaches that clash with the current architecture -- the failures that actually break implementation.

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
| max_rounds | 10 | Safety cap -- stop even if not approved after this many rounds |
| plan_path | auto-detect | Path to the plan file |
| project_root | auto-detect | Absolute path of the project Codex inspects (Claude's working directory, or the Git repo root) |
| focus | general | One of: general, architecture, edge-cases, security, performance |
| review_codebase | true | `true`: Codex inspects the real project files. `false`: plan-text-only review (use for greenfield plans with no code yet, or a pure logic check) |
| model | gpt-5.5 | Codex model used for the review |
| reasoning_effort | xhigh | Codex reasoning effort -- one of: minimal, low, medium, high, xhigh (xhigh = "extra high", the deepest) |

The loop is **convergence-based by default**: it runs until Codex signals approval, NOT for a fixed number of iterations. The `max_rounds` parameter is only a safety limit.

## Execution Steps

### Step 0: Detect Plan File and Project Root

1. **Plan file:**
   - If the user specified a path, use it
   - If currently in planning mode with an active plan file, use that path
   - Otherwise, find the most recent plan:
     ```bash
     ls -t ~/.claude/plans/*.md | head -1
     ```
   - Read the plan file. If empty or missing, inform the user and stop

2. **Project root:** Determine the absolute path of the project Codex should inspect. This is normally Claude Code's current working directory -- run `pwd`. If the working directory is a subdirectory of a Git repository, prefer the repo root (`git rev-parse --show-toplevel`). Store the absolute path as `{project_root}`. Always detect this, even when `review_codebase` is false.

3. **Git check:** Record whether `{project_root}` is a Git repository:
   ```bash
   git -C "{project_root}" rev-parse --is-inside-work-tree 2>/dev/null
   ```
   This informs the transport choice in Step 1.

### Step 1: Detect Transport Method

There are two ways to reach Codex:

- **MCP (preferred)** -- the `codex-cli` server: `mcp__codex-cli__codex` starts a session, `mcp__codex-cli__codex-reply` continues it. Preferred because the session persists across rounds, so Codex's codebase exploration from round 1 carries into later rounds.
- **Bash fallback** -- `/opt/homebrew/bin/codex exec`.

Try the MCP tools first. If they are unavailable or a call fails, fall back to Bash for that round.

Note: if `{project_root}` is **not** a Git repository, the MCP `codex` tool may reject it (it has no `--skip-git-repo-check` option). This is expected -- fall back to the Bash method, which passes `--skip-git-repo-check`. It is not a real error.

### Step 2: Review Loop (repeats until APPROVED or max_rounds reached)

#### 2a. Build the Review Prompt

For **round 1**, construct this prompt (fill in `{project_root}`, `{focus}`, `{plan_content}`):

```
You are an expert plan reviewer with read access to the project's codebase. Your job is to critically review a software implementation plan created by another AI assistant and either APPROVE it or request REVISIONS.

Project root: {project_root}
Focus area: {focus}

GROUND YOUR REVIEW IN THE ACTUAL CODEBASE.
Before judging the plan, inspect the project at the path above. Read the files, modules, and configuration the plan touches or depends on -- you do NOT need to read the whole repo, just what the plan references. Use this to verify:
- Do the files, functions, types, and paths the plan mentions actually exist as described?
- Are the plan's assumptions about current behavior and APIs correct?
- Does the proposed approach fit the existing architecture, conventions, and patterns?
- Does the plan miss existing code it should reuse, or conflict with code already there?

Review Criteria:
1. Codebase grounding: Does the plan match the real state of the project? Flag incorrect assumptions and missing or renamed files.
2. Completeness: Are there missing steps, edge cases, or dependencies?
3. Correctness: Are the technical approaches sound?
4. Risk: What could go wrong? What are the failure modes?
5. Ordering: Are the steps in the right sequence? Any dependency issues?
6. Clarity: Is the plan clear enough to implement without ambiguity?

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
Group by severity. Be specific -- cite concrete file paths and line numbers from the codebase where relevant.

Only use NEEDS REVISION if there are genuinely important issues (critical or major). Minor-only issues should still result in APPROVED with optional suggestions.
Do NOT rewrite the plan and do NOT modify any files. Only provide feedback.
```

For **rounds 2+**, use this follow-up prompt:

```
You previously reviewed a plan for the project at {project_root} and requested revisions. The plan author has revised the plan based on your feedback. Please re-review.

Your previous feedback:
{previous_codex_feedback}

The Revised Plan:
{revised_plan_content}

If you reviewed an earlier version of this plan in this session, the codebase context you gathered then still holds -- the project's code has not changed between rounds, only the plan. Do not re-explore from scratch; re-inspect only the files the revisions newly touch. Then check:
1. Were your previous critical/major issues adequately addressed?
2. Did the revisions introduce any NEW critical or major issues, or any new mismatch with the actual codebase?
3. Is the plan now ready for implementation?

RESPONSE FORMAT -- you MUST follow this exactly:

First line must be:
VERDICT: APPROVED
or
VERDICT: NEEDS REVISION

Same format as before. If all critical/major issues are resolved and no new ones were introduced, APPROVE the plan. Do not keep requesting revisions for diminishing returns -- approve when the plan is solid enough to implement well.
Do NOT rewrite the plan and do NOT modify any files. Only provide feedback.
```

If `review_codebase` is **false**, drop the `Project root` line, the "GROUND YOUR REVIEW" block, and review criterion 1 (`Codebase grounding`) -- this gives a pure plan-text review.

#### 2b. Send to Codex

The Codex settings below are the same whether or not `review_codebase` is true -- only the prompt content from 2a differs.

**MCP method (preferred):**

- **Round 1** -- call `mcp__codex-cli__codex` with:
  - `prompt`: the round-1 review prompt
  - `cwd`: `{project_root}` (absolute path) -- this is what lets Codex see the project
  - `sandbox`: `"read-only"` -- Codex inspects but never modifies the repo or the plan
  - `approval-policy`: `"never"` -- keeps the loop non-interactive; safe because a read-only sandbox cannot take destructive actions
  - `model`: `{model}` -- the review model (default `gpt-5.5`)
  - `config`: `{ "model_reasoning_effort": "{reasoning_effort}" }` -- sets reasoning depth (default `xhigh`); the `codex` tool has no dedicated effort parameter, so effort goes through `config`

  Then capture the session/thread id from the response (look for `threadId`, `conversationId`, or `thread_id` in the structured result).

- **Rounds 2+** -- call `mcp__codex-cli__codex-reply` with:
  - `threadId`: the id captured in round 1
  - `prompt`: the round-2+ review prompt

  Reusing the thread means Codex keeps its `cwd`, sandbox, model, reasoning effort, and codebase exploration from round 1. If no thread id was captured, call `mcp__codex-cli__codex` fresh instead (with the same `cwd`/`sandbox`/`approval-policy`/`model`/`config` as round 1) -- the round-2+ prompt already embeds the previous feedback and full revised plan.

**Bash fallback:**

Write the round's prompt to `/tmp/plan-review-prompt.txt` first, then:

- **Round 1:**
  ```bash
  cat /tmp/plan-review-prompt.txt | /opt/homebrew/bin/codex exec - \
    -C "{project_root}" \
    -s read-only \
    -m {model} \
    -c model_reasoning_effort="{reasoning_effort}" \
    --skip-git-repo-check \
    -o /tmp/codex-review-output.txt \
    2>/dev/null
  ```

- **Rounds 2+** (resume round 1 so codebase context carries over):
  ```bash
  cat /tmp/plan-review-prompt.txt | /opt/homebrew/bin/codex exec resume --last - \
    -m {model} \
    -c model_reasoning_effort="{reasoning_effort}" \
    --skip-git-repo-check \
    -o /tmp/codex-review-output.txt \
    2>/dev/null
  ```
  `resume` inherits `cwd` and `sandbox` from round 1. If resume fails or picks the wrong session it is non-fatal -- the round-2+ prompt already contains the previous feedback and full revised plan, so a plain `codex exec -` (the round-1 form) also works.

- Read `/tmp/codex-review-output.txt` for Codex's response.
- Do **not** use `--ephemeral`: it stops the session from being saved, which breaks `resume`.

IMPORTANT: Set the Bash timeout to 600000 (10 minutes) for Codex calls -- inspecting the codebase plus reviewing a large plan can take several minutes. MCP calls can likewise take a few minutes; that is expected.

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

Codex's feedback is now grounded in the real code, so it may cite concrete file paths and code facts -- use them to make the revision accurate (correct a wrong path, reuse an existing utility Codex pointed to, align with a pattern it flagged).

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

- **MCP tool fails**: Fall back to the Bash method for that round
- **MCP `codex` rejects a non-Git project root**: Expected for non-Git projects -- use the Bash method (`--skip-git-repo-check`); not a real error
- **Codex timeout**: Retry once. If it fails again, present what was accomplished so far
- **Plan file too large** (>15K words): Warn the user; consider reviewing section-by-section
- **Session resume fails**: Start a fresh `codex exec` (round-1 form) -- the previous feedback is already embedded in the round-2+ prompt
- **Codex reports the plan references files that do not exist**: This is a valid CRITICAL finding -- surface it; it usually means the plan author assumed paths that are not in the codebase
- **User interrupts**: The plan is saved after each revision, so progress is never lost
- **Codex keeps requesting revisions on trivial issues**: If round 3+ and only minor issues remain, Claude may judge the plan sufficient and present the remaining minors to the user as optional improvements

## Notes

- **Codebase-grounded review is the core value.** Codex reviews with read-only access to the actual project (`cwd` + `sandbox: read-only`), so it catches hallucinated paths, wrong assumptions about existing code, and approaches that clash with the architecture -- failures a plan-text-only review cannot see.
- `sandbox: read-only` lets Codex read every file but modify nothing -- not the repo and not the plan. Claude stays the sole editor of the plan; Codex only critiques. This keeps Claude in control of revisions and prevents style/context loss.
- Codex's codebase exploration is reused, not repeated. The review session persists across rounds (MCP `threadId`, or Bash `resume`), so rounds 2+ build on round 1's exploration and only re-inspect what the revised plan newly touches -- the project's code does not change between rounds, only the plan. A full re-exploration happens only on the degraded path where `resume` fails and a fresh `codex exec` is used.
- Set `review_codebase=false` for greenfield plans (no code written yet) or when you want a pure logic review of the plan text alone.
- The review runs Codex at `model` / `reasoning_effort` (default `gpt-5.5` at `xhigh` -- "extra high", the deepest reasoning). The skill pins these explicitly so review quality does not silently depend on the user's Codex `config.toml` defaults. Lower `reasoning_effort` (e.g. `high`) for faster, cheaper reviews, or if the Codex build or chosen model does not support `xhigh`.
- The convergence approach is smarter than fixed iterations: simple plans may be approved in 1 round, complex plans may need 4-5.
- For security-sensitive plans, use `focus=security` to prioritize threat modeling -- with codebase access Codex can inspect the real attack surface.
- The `VERDICT:` line makes parsing deterministic -- no ambiguity about whether Codex approved or not.
