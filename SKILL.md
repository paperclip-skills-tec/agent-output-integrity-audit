---
name: agent-output-integrity-audit
description: "Post-incident audit workflow for investigating suspected fabricated or inaccurate agent sign-offs. Use this skill whenever the board, a user, or another agent reports a verification comment that may contain fabricated artifact claims — non-existent git SHAs, branches that never existed, file paths that weren't created, or test commands with invented exit codes. Also invoke when a previously-accepted completion is being questioned and you need to establish the blast radius. Covers the full pipeline: pulling the agent's comment history, extracting artifact claims, spot-checking each claim against real git/filesystem state, categorizing findings by veracity (accurate / inaccurate / unverifiable / fabricated), drafting per-issue remediation recommendations, writing a structured findings document, and escalating confirmed fabrications to the board. Invoke this skill proactively on any integrity concern — even if the reported case looks minor, a single confirmed fabrication often signals a broader pattern worth auditing."
---

# Agent Output Integrity Audit

This skill governs post-incident investigation when an agent's comments may contain fabricated or inaccurate artifact claims. The goal is a consistent, evidence-based findings document that the board can use to assess blast radius and decide on remediation.

Fabricated sign-offs are particularly damaging because they look exactly like real completions — a QA engineer who writes "merged to SHA abc123 on branch feature/x with all tests passing" when none of those things exist has poisoned the audit trail. Without a structured investigation, it is easy to miss the full scope, apply inconsistent evidence standards, or make remediation decisions on incomplete information.

---

## Step 1 — Identify the Agent and Time Window

Before pulling data, establish:

- **Agent ID** — from the report, from the Paperclip issue, or via `GET /api/companies/$PAPERCLIP_COMPANY_ID/agents`
- **Date range** — start/end in ISO 8601 (e.g., `2026-01-01T00:00:00Z` to `2026-01-15T23:59:59Z`)
- **Issues in scope** — the specific issues named in the report, plus any parent or sibling issues that may have relied on the suspect sign-offs

Document this scope at the top of your working notes before continuing. Do not expand the scope mid-audit without noting why.

---

## Step 2 — Pull Agent Comment History

Fetch all comments by the suspect agent across the scoped issues:

```bash
# Comments on a specific issue, filtered to the suspect agent
curl -s "$PAPERCLIP_API_URL/api/issues/{issueId}/comments" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  | jq '[.[] | select(.authorAgentId == "{agentId}") | {id, body, createdAt}]'
```

For a broader sweep when the full issue scope is uncertain:

```bash
# All recently completed issues by the agent
curl -s "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/issues?assigneeAgentId={agentId}&status=done" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  | jq '[.[] | {id, identifier, title, completedAt}]'
```

Collect full comment bodies — do not rely on summaries or truncated excerpts.

---

## Step 3 — Extract Artifact Claims

For each comment, parse every verifiable claim and record it in a table:

| Comment ID | Date | Claim Type | Claimed Value |
|------------|------|------------|---------------|
| … | … | SHA | `abc1234` |
| … | … | Branch | `feature/auth-fix` |
| … | … | File path | `/src/auth/middleware.ts` |
| … | … | Test command + result | `npm test — exit 0, 47 passing` |
| … | … | Issue/PR ref | `TEC-990 merged` |

**Claim types to watch for:**

- Git commit SHAs (full or short — `abc1234`, `abc1234def5678`)
- Branch names (`feature/`, `fix/`, `release/`, `hotfix/`)
- File paths claimed to have been created, modified, or deleted
- Test commands and their claimed output (exit codes, pass/fail counts, coverage %)
- Issue or PR references cited as "completed", "merged", or "closed"
- Timestamps for events ("tests passed at 14:32")

Err on the side of extracting more claims rather than fewer. A claim you skip cannot be verified.

---

## Step 4 — Spot-Check Each Artifact

Run the appropriate check for each claim type:

**Git SHAs:**

```bash
git rev-parse --verify {sha} 2>&1
# Pass: outputs the full 40-char SHA
# Fail: "fatal: unknown revision or path not in the working tree"

# Also check full history in case it was rewritten
git log --all --oneline | grep "^{short-sha}"
```

**Branch existence:**

```bash
git branch -a | grep "{branch-name}"
# Also check recent history for any trace
git log --all --oneline --decorate | grep "{branch-name}"
```

**File paths:**

```bash
ls -la {file-path} 2>&1
stat {file-path} 2>&1
# For claimed-deleted files: check git history
git log --all --oneline -- {file-path}
```

**Test commands (verify the script/command exists):**

```bash
which {command} 2>&1
ls {script-path} 2>&1
```

**Referenced issues/PRs:**

```bash
curl -s "$PAPERCLIP_API_URL/api/issues/{issueId}" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  | jq '{status, identifier, title, completedAt}'
```

Record the raw command output as evidence alongside each claim — do not paraphrase the results.

---

## Step 5 — Categorize Findings

Classify each claim using this four-category scheme:

| Category | Label | Meaning |
|----------|-------|---------|
| **(a)** | Verifiable + Accurate | Artifact exists and matches the claim exactly |
| **(b)** | Verifiable + Inaccurate | Artifact exists but does not match (wrong branch name, different SHA, file exists but was not modified as claimed) |
| **(c)** | Unverifiable | Artifact may have existed but cannot be confirmed now (deleted branch, ephemeral container, cleaned workspace) |
| **(d)** | Fabricated | Artifact demonstrably never existed — no trace in git history, filesystem, or issue tracker, and the supposed creation window is well past |

**Be conservative with (d).** Only use "Fabricated" when you have positive evidence of non-existence (e.g., `git log --all` returns nothing for the SHA, and the commit was supposedly made in this repo days ago). If there is genuine ambiguity, use (c).

---

## Step 6 — Per-Issue Remediation Recommendation

For each issue where you found **(b)** or **(d)** findings, recommend one of:

- **Re-verify** — The issue may still be correctly completed; request the assignee supply real evidence. Use when (b) findings could be honest mistakes or transposition errors.
- **Rollback** — Mark the issue not-done and return it to the appropriate prior state. Use when (d) findings indicate the claimed work was never performed.
- **Mark moot** — Accept that the work cannot be verified retroactively, but document the gap. Use when (c) findings dominate and the verification window has passed.
- **Escalate** — Surface to the board with your recommendation if findings suggest a systemic pattern rather than an isolated incident.

---

## Step 7 — Write Findings Document

Create the findings as an `audit` document on the audit issue:

```bash
curl -s -X POST "$PAPERCLIP_API_URL/api/issues/{auditIssueId}/documents" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Integrity Audit Findings — {AgentName} — {DateRange}",
    "kind": "audit",
    "content": "..."
  }'
```

**Required document structure:**

```markdown
# Integrity Audit Findings

**Agent:** {agent name} ({agent ID})
**Period:** {start} to {end}
**Issues in scope:** {list with identifiers}
**Audit date:** {today}

## Summary

| Category | Count |
|----------|-------|
| (a) Verifiable + Accurate | N |
| (b) Verifiable + Inaccurate | N |
| (c) Unverifiable | N |
| (d) Fabricated | N |
| **Total claims checked** | N |

## Findings by Issue

### {Issue Identifier} — {Issue Title}

**Claims checked:** N
**Categories:** (a) N, (b) N, (c) N, (d) N

| Claim | Verification command | Category | Raw evidence |
|-------|---------------------|----------|--------------|
| SHA `abc1234` | `git rev-parse --verify abc1234` | (d) | `fatal: unknown revision` |
| Branch `feature/auth` | `git branch -a \| grep feature/auth` | (d) | *(no output)* |

**Remediation recommendation:** {Rollback / Re-verify / Mark moot / Escalate}
**Rationale:** {One sentence}

## Overall Assessment

{1-2 paragraphs: what the evidence shows, likely scenario (honest mistake vs. systematic
fabrication), and recommended next step.}
```

---

## Step 8 — Escalation

If findings include **confirmed (d) Fabricated claims**, request a board decision before recommending agent pause or rollback:

```bash
curl -s -X POST "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/approvals" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "request_board_approval",
    "requestedByAgentId": "'"$PAPERCLIP_AGENT_ID"'",
    "issueIds": ["{auditIssueId}"],
    "payload": {
      "title": "Agent integrity: fabricated claims confirmed — {AgentName}",
      "summary": "N fabricated artifact claims confirmed across M issues in the period start–end. Findings document attached to {auditIssueId}.",
      "recommendedAction": "Suspend / Warn / Rollback affected issues — {your recommendation with rationale}",
      "risks": ["List affected issues and any downstream tasks that relied on fabricated completions"]
    }
  }'
```

If findings are only **(b)** or **(c)** — no confirmed fabrication — document the issues, notify relevant assignees, and recommend re-verification. No board escalation is required unless the pattern appears systematic.

Set the audit issue to `in_review` while waiting for the board's `approval_approved` or `approval_rejected` wake.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
