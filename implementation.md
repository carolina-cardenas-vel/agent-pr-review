
# GitHub Copilot PR Review Agent Design

For this use case, I would **not build it as a traditional GitHub Copilot "prompt-only agent."** Instead, I would use Copilot in VS Code as the orchestration layer and combine it with:

- GitHub APIs (PR analysis and review comments)
- Azure DevOps APIs (work item / requirements retrieval)
- An LLM-powered review engine
- Existing MCP servers (no custom server needed)
- GitHub Review APIs for inline comments

This produces a much more reliable solution than trying to fit everything into a single Copilot prompt.

---

# High-Level Architecture

```text
┌─────────────────────┐
│ User in VS Code     │
│                     │
│ PR URL              │
│ ADO Work Item ID    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Copilot Agent       │
│ (Custom Agent Mode) │
└──────────┬──────────┘
           │
           ▼
┌────────────────────────────┐
│ Review Orchestrator        │
├────────────────────────────┤
│ Parse PR URL               │
│ Get PR Files               │
│ Get Diff/Hunks             │
│ Get ADO Work Item          │
│ Extract AC                 │
│ Chunk Code                 │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│ Multi-Step Review Engine   │
├────────────────────────────┤
│ Requirement Validation     │
│ Security Review            │
│ Bug Detection              │
│ Performance Review         │
│ Test Coverage Review       │
│ Maintainability Review     │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│ Issue Mapper               │
├────────────────────────────┤
│ Maps findings to           │
│ exact PR diff lines        │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│ GitHub Review API          │
├────────────────────────────┤
│ Inline Review Comments     │
│ Request Changes            │
│ Approve                    │
└────────────────────────────┘
```

---

# Recommended Implementation Pattern

## Input

The agent receives:

```json
{
  "prUrl": "https://github.com/org/repo/pull/123",
  "adoWorkItemId": 4567
}
```

or

```text
Review PR:

https://github.com/org/repo/pull/123

Against ADO:

4567
```

---

# Step 1: Retrieve ADO Work Item Requirements

Use the **Azure DevOps remote MCP server** (`microsoft/azure-devops-mcp`, hosted at `https://mcp.dev.azure.com/{org}`). Authentication is Microsoft account OAuth — no PAT or API token needed.

Retrieve:

```text
Title
Description
Acceptance Criteria  (Microsoft.VSTS.Common.AcceptanceCriteria field)
Repro Steps / Story Notes
Linked Work Items
Tags / State
```

The ADO MCP tool (e.g., `work_item_get`) returns the full work item. The agent then converts it into structured requirements:

```json
{
  "requirements": [
    "User can reset password",
    "Email contains token",
    "Token expires after 30 minutes"
  ],
  "acceptanceCriteria": [
    "...",
    "...",
    "..."
  ]
}
```

This becomes the source of truth.

---

# Step 2: Retrieve PR Data

Use GitHub APIs.

## Pull Request

```http
GET /repos/{owner}/{repo}/pulls/{number}
```

## Changed Files

```http
GET /repos/{owner}/{repo}/pulls/{number}/files
```

Returns:

```json
[
  {
    "filename":"auth/service.ts",
    "patch":"@@ ..."
  }
]
```

The patch information is critical because GitHub inline comments require exact diff locations.

---

# Step 3: Build Understanding of the Change

The biggest mistake is reviewing only diff hunks.

The agent should gather:

```text
Changed file
Related classes
Imported services
Test files
Configuration changes
```

For example:

```text
src/auth/password-reset.service.ts

+
+
+
```

The reviewer should also load:

```text
src/auth/token.service.ts
src/auth/email.service.ts
```

Otherwise it lacks context.

A Retrieval-Augmented workflow works well here.

---

# Step 4: Multi-Pass Review

Instead of asking the LLM:

```text
Review this PR
```

Run several specialized review passes.

---

## Pass 1: ADO Compliance

Prompt:

```text
You are a business analyst.

Given:
- Azure DevOps work item
- Acceptance criteria
- Changed code

Determine:

1. Fully implemented
2. Partially implemented
3. Missing

For each issue produce:
- file
- line
- reason
- suggested fix
```

Example finding:

```text
AC #3 requires token expiry after 30 minutes.

Implementation creates token but no expiration validator exists.
```

---

## Pass 2: Functional Bugs

Prompt:

```text
Act as a senior engineer.

Look for:
- Null handling
- Error handling
- Race conditions
- Broken logic
- Invalid assumptions
```

---

## Pass 3: Security Review

Look for:

```text
SQL Injection
Path Traversal
SSRF
XSS
Secrets
Weak Authentication
Unsafe Deserialization
Insecure Logging
Privilege Escalation
```

Example:

```ts
const sql = `SELECT * FROM users WHERE id=${userId}`;
```

Generated comment:

```text
Potential SQL injection vulnerability.

Use parameterized queries instead of string interpolation.

Suggested:

db.query(
  'SELECT * FROM users WHERE id = ?',
  [userId]
)
```

---

## Pass 4: Performance Review

Look for:

```text
N+1 queries
Repeated API calls
Inefficient loops
Large object loading
Excessive allocations
```

---

## Pass 5: Test Coverage

Compare code changes against test changes.

Questions:

```text
Were new branches tested?

Are error cases covered?

Are acceptance criteria tested?
```

Example:

```text
Password expiration logic added.

No unit tests validate expiration behavior.
```

---

## Pass 6: Maintainability Review

Check:

```text
Duplicated logic
Large methods
Magic values
Naming quality
Layer violations
Technical debt
```

---

# Step 5: Map Findings to Exact Lines

This is one of the hardest parts.

GitHub review comments need:

```json
{
  "path":"src/auth/service.ts",
  "line":104
}
```

The LLM should produce:

```json
{
  "file":"src/auth/service.ts",
  "line":104,
  "severity":"high",
  "comment":"Token expiration is not validated."
}
```

A mapper component then verifies:

```text
Does line exist?
Is line within diff?
```

If not:

```text
Find closest modified line.
```

---

# Step 6: Publish Inline Comments

Use GitHub Review APIs.

Example:

```http
POST /repos/{owner}/{repo}/pulls/{number}/reviews
```

```json
{
  "event":"REQUEST_CHANGES",
  "comments":[
    {
      "path":"src/auth/service.ts",
      "line":104,
      "body":"Acceptance criterion #3 requires token expiration validation..."
    }
  ]
}
```

Result:

- Comments appear directly in PR
- Developer can reply
- Thread created automatically
- Resolution workflow preserved

Exactly what you described.

---

# Using GitHub Copilot Agent Mode

If you're specifically targeting the newer GitHub Copilot Agent experience in VS Code, the following tools are provided by the two existing MCP servers — no custom server code needed:

```text
# From microsoft/azure-devops-mcp (remote, OAuth)
work_item_get(id)

# From github/github-mcp-server (remote, OAuth)
get_pull_request(owner, repo, number)
get_pull_request_files(owner, repo, number)
get_file_contents(owner, repo, path)
create_pull_request_review(owner, repo, number, comments, event)
```

Then the agent workflow becomes:

```text
User:
Review PR 123 against ADO: 4567

Agent:
  -> fetch ADO work item (ADO MCP)
  -> fetch PR + files (GitHub MCP)
  -> analyze code
  -> identify issues
  -> create comments
  -> publish review (GitHub MCP)
```

This is much more reliable than giving the model direct access to raw REST endpoints without structured tools.

---

# MCP-Based Design (Chosen Approach)

We use two **existing, officially maintained MCP servers** — no custom server code to write or maintain:

| Server | Hosted by | Auth | Purpose |
|--------|-----------|------|---------|
| `github/github-mcp-server` | GitHub | OAuth (Microsoft/GitHub account) | PR data, file contents, post review |
| `microsoft/azure-devops-mcp` | Microsoft | OAuth (Microsoft account) | ADO work item requirements |

Both servers are remote (HTTP), configured in `.vscode/mcp.json`. Authentication is browser-based OAuth on first use — no PATs or API tokens to manage.

## MCP Tools Used

```text
# Azure DevOps MCP
work_item_get(id)

# GitHub MCP
get_pull_request(owner, repo, number)
get_pull_request_files(owner, repo, number)
get_file_contents(owner, repo, path)
create_pull_request_review(owner, repo, number, comments, event)
```

The agent prompt (in `.github/copilot-instructions.md`) becomes the only deliverable:

```text
You are a Staff Software Engineer.

Review the pull request against the Azure DevOps work item.

Validate:

- requirements and acceptance criteria
- security
- performance
- test coverage
- maintainability

For every issue:
- determine exact diff location
- create review comment
- provide actionable fix

Publish comments through GitHub review APIs.
```

This keeps the intelligence in the model and the execution in deterministic tools.

---

# Additional Enhancement: Two-Agent Review System

For enterprise-grade accuracy, run two reviewers.

## Agent 1

### Requirement Reviewer

Focuses only on:

```text
Azure DevOps Work Item
Acceptance Criteria
Business Logic
```

## Agent 2

### Technical Reviewer

Focuses only on:

```text
Security
Performance
Tests
Architecture
Code Quality
```

Then merge findings before posting comments.

This significantly reduces missed issues and hallucinated requirements.

---

# My Recommended Stack

```text
VS Code + GitHub Copilot Agent Mode
            +
github/github-mcp-server  (remote, OAuth)
            +
microsoft/azure-devops-mcp  (remote, OAuth)
            +
.github/copilot-instructions.md  (agent prompt)
            +
GitHub Copilot model
```

This architecture gives you a review agent that behaves almost like a senior engineer: it understands the Azure DevOps requirements, analyzes the complete PR context, identifies implementation gaps, and posts actionable inline comments directly into the pull request discussion threads rather than generating a separate report.
