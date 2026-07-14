# GitHub Copilot PR Review Agent — Implementation Plan

## Overview

This document provides the complete implementation specification for a GitHub Copilot Agent that automatically reviews pull requests against Azure DevOps (ADO) work item requirements. The agent is always-on in Agent Mode and activates when users provide a PR URL and an ADO work item ID.

## Architecture

```
User in VS Code (Agent Mode)
    ↓
    ├─→ sends: "Review PR: https://github.com/org/repo/pull/123 against ADO: 4567"
    ↓
VS Code Agent Mode + copilot-instructions.md
    ↓
    ├─→ work_item_get(4567)  [Azure DevOps MCP tool]
    │   ├─→ title, description, acceptance criteria
    │   └─→ linked work items, tags, state
    ↓
    ├─→ get_pull_request(org, repo, 123)
    │   └─→ PR metadata, branch info, author
    ↓
    ├─→ get_pull_request_files(org, repo, 123)
    │   ├─→ changed files list
    │   ├─→ diff patches with line numbers
    │   └─→ additions/deletions summary
    ↓
    ├─→ get_file_contents(org, repo, path) × N
    │   └─→ full file context for each changed file
    ↓
    ├─→ 6-Pass Review Analysis (in-context, no tool calls)
    │   ├─→ Pass 1: ADO Compliance
    │   ├─→ Pass 2: Functional Bugs
    │   ├─→ Pass 3: Security Review
    │   ├─→ Pass 4: Performance Review
    │   ├─→ Pass 5: Test Coverage Review
    │   └─→ Pass 6: Maintainability Review
    ↓
    └─→ create_pull_request_review(...)
        ├─→ all findings as inline comments
        ├─→ one batched API call
        └─→ event: REQUEST_CHANGES or COMMENT
```

## Files Created

### 1. `.vscode/mcp.json`
**Purpose**: Configure MCP servers for GitHub and Azure DevOps

**Content**:
- GitHub server: remote HTTP, OAuth at `https://api.githubcopilot.com/mcp/`
- ADO server: remote HTTP, OAuth at `https://mcp.dev.azure.com/{org}` (Microsoft-hosted)
- VS Code input: prompt for ADO organization name only (no secrets)
- Allowed tools from GitHub: `get_pull_request`, `get_pull_request_files`, `get_file_contents`, `create_pull_request_review`, `list_pull_requests`

**Why**: Both servers are remote and OAuth-based — no credentials to store, no local processes to run.

### 2. `.env.example`
**Purpose**: Document that the ADO remote MCP server uses OAuth (no secrets needed)

**Content**:
- ADO organization name format explanation
- Note that authentication is browser-based OAuth (Microsoft account) on first use
- No API tokens or passwords required

### 3. `.gitignore`
**Purpose**: Prevent accidental credential commits

**Changes**: Add `.env` line (create if missing)

### 4. `.github/copilot-instructions.md`
**Purpose**: Agent workflow and review logic (the core deliverable)

**Key sections**:
- Role definition: Staff-level code reviewer
- Input parsing: extract PR URL and ADO work item ID (numeric)
- Workflow: sequential tool calls
- 6 review passes: detailed prompts for each pass
- Comment format: severity emoji + explanation + fix, references as `ADO #<id>, AC #<n>`
- Line mapping: diff-only lines or nearest changed line
- Review event logic: REQUEST_CHANGES vs COMMENT

### 5. `.github/plan.md`
**Purpose**: This file — complete spec for future reference and implementation

---

## Phase 1: Prerequisites (Manual Check)

Before running the agent, verify:

1. **VS Code**: Version 1.101 or later (required for remote MCP + OAuth)
   ```bash
   code --version
   ```

2. **ADO organization name**: Find it in your ADO URL — `https://dev.azure.com/<org-name>`
   - This is the only configuration value you need
   - No API tokens or passwords are required
   - Authentication happens via Microsoft account OAuth on first use

3. **GitHub access**: Ensure your GitHub account has read access to the PR repository and `pull_request:write` for posting comments

---

## Phase 2: MCP Configuration

### File: `.vscode/mcp.json`

Configures two MCP servers:

**GitHub Server** (Remote):
- URL: `https://api.githubcopilot.com/mcp/`
- Authentication: OAuth (automatic, first-time browser login)
- Tools enabled: `get_pull_request`, `get_pull_request_files`, `get_file_contents`, `create_pull_request_review`

**Azure DevOps Server** (Remote, official Microsoft-hosted):
- URL: `https://mcp.dev.azure.com/{organization}` (organization name prompted via VS Code input)
- Authentication: **Microsoft account OAuth** — browser opens on first use; no PAT or password required
- Tools: All ADO work item tools available (work items, boards, repos, wikis, pipelines, etc.)
- Relevant tool: `work_item_get` or equivalent — verify exact name from VS Code tools list

### File: `.env.example`

Documents that the ADO remote server requires no secrets. Authentication is handled via browser-based OAuth using the user’s existing Microsoft/ADO account.

### File: `.gitignore`

Add `.env` to prevent credential leaks.

---

## Phase 3: Agent Instructions

### File: `.github/copilot-instructions.md`

This file contains the complete agent workflow. Key sections:

#### Role

```
You are a Staff-level Software Engineer and Code Reviewer with expertise in:
- Security (OWASP Top 10, secure coding patterns)
- Performance (databases, APIs, algorithms)
- Testing strategies (coverage, edge cases)
- Maintainability (SOLID principles, architectural patterns)
- Requirement validation (business logic, acceptance criteria)
```

#### Input Parsing

```
When a user provides:
"Review PR: https://github.com/org/repo/pull/123 against ADO: 4567"

Extract:
- owner: org
- repo: repo
- pullNumber: 123
- adoWorkItemId: 4567
```

#### Workflow (Sequential)

1. **Fetch ADO Work Item Requirements**
   ```
   Tool: work_item_get(id=4567)  [or equivalent ADO MCP tool name]
   Extract:
     - title
     - description
     - acceptanceCriteria (from Microsoft.VSTS.Common.AcceptanceCriteria field)
     - linked work items
     - tags, state
   ```

2. **Fetch PR Metadata**
   ```
   Tool: get_pull_request(owner, repo, pullNumber)
   Extract:
     - title, description
     - author, base branch
     - created_at, updated_at
   ```

3. **Fetch Changed Files + Diffs**
   ```
   Tool: get_pull_request_files(owner, repo, pullNumber)
   For each file:
     - filename
     - patch (raw diff with @@ line numbers @@)
     - additions, deletions
   ```

4. **Fetch Full File Context**
   ```
   Tool: get_file_contents(owner, repo, path) for each changed file
   Purpose: Understand full context beyond just diff hunks
   ```

5. **Run 6 Review Passes** (in-context, no tool calls)

   **Pass 1: ADO Compliance**
   ```
   For each acceptance criterion from the ADO work item:
     1. Find implementation in changed code
     2. Determine: Fully Implemented / Partially Implemented / Missing
     3. If not fully implemented:
        - Identify exact file and line
        - Explain the gap
        - Suggest fix

   Output: List of findings with file, line, severity
   ```

   **Pass 2: Functional Bugs**
   ```
   Look for:
     - Null pointer handling
     - Unhandled error cases
     - Race conditions
     - Off-by-one errors
     - Broken loop conditions
     - Invalid state transitions
     - Resource leaks

   Output: Finding per issue (file, line, explanation, fix)
   ```

   **Pass 3: Security Review**
   ```
   Look for:
     - SQL Injection (string interpolation in queries)
     - XSS (unsanitized user input in responses)
     - Path Traversal (user input in file paths)
     - CSRF (missing token validation)
     - Broken Authentication (weak credential checks)
     - Insecure Deserialization
     - Hardcoded Secrets (API keys, passwords, tokens)
     - Insecure Logging (sensitive data)
     - Privilege Escalation (unauthorized access checks)

   Output: Finding per vulnerability (file, line, OWASP category, fix)
   ```

   **Pass 4: Performance Review**
   ```
   Look for:
     - N+1 Query patterns (loops with database calls)
     - Repeated API calls in loops
     - Large object allocations
     - Inefficient sorting/filtering
     - Missing indexes on queries
     - Unbounded data loads

   Output: Finding per issue (file, line, impact, optimization)
   ```

   **Pass 5: Test Coverage Review**
   ```
   Compare code changes with test changes:
     - New functions: are they tested?
     - New branches: are all paths covered?
     - New error cases: are failures tested?
     - AC-specific logic: is it tested?

   Output: Finding per gap (file, line, test suggestion)
   ```

   **Pass 6: Maintainability Review**
   ```
   Look for:
     - Code duplication (same logic > 2 places)
     - Long methods (> 50 lines)
     - Magic values (hardcoded numbers/strings)
     - Poor naming (unclear intent)
     - Layer violations (UI calling DB directly)
     - Technical debt markers (TODO, FIXME, HACK)

   Output: Finding per issue (file, line, suggestion)
   ```

6. **Create PR Review**
   ```
   Tool: create_pull_request_review(
     owner=org,
     repo=repo,
     pullNumber=123,
     comments=[...],  // all findings from passes 1-6
     event="REQUEST_CHANGES" or "COMMENT"
   )

   Comment format for each finding:
   ----
   🔴 Critical: [Title] (Pass 2: Functional Bugs)
   File: src/auth/service.ts
   Line: 45

   **Issue**: [Explanation]
   
   **Risk**: [Business/security/performance impact]
   
   **Suggested Fix**:
   ```
   [Code snippet showing the fix]
   ```
   
   **Acceptance Criterion**: This addresses AC #3 from ADO #4567
   ----

   Or for warnings:
   
   🟡 Warning: [Title] (Pass 4: Performance)
   
   Or for suggestions:
   
   🔵 Suggestion: [Title] (Pass 6: Maintainability)
   ----

   Event logic:
     - If any 🔴 Critical finding: event = "REQUEST_CHANGES"
     - Else if any 🟡 Warning: event = "COMMENT"
     - Else: event = "COMMENT"
   ```

#### Line Mapping Rule

```
GitHub review comments require exact line numbers from the diff.

Rule:
1. Use only lines present in the patch (between @@ markers)
2. If a comment target is outside the diff:
   - Find the nearest changed line within the function/block
   - Reference that line instead
   - Explain: "Related to change on line X"

Example:
  Diff shows lines 40-50 changed.
  Issue is on line 35 (calling the changed function).
  → Reference line 40 (nearest changed line in the diff)
```

---

## Phase 4: Verification Checklist

After implementation, verify:

### 1. MCP Server Setup
```bash
# No installation needed for remote servers.
# Both GitHub and ADO MCP servers are hosted remotely by their respective companies.
# Just confirm VS Code 1.101+ is installed:
code --version
```

### 2. VS Code Agent Mode
- Confirm `.vscode/mcp.json` exists
- Open VS Code (1.101+)
- Toggle Agent Mode (icon in Copilot Chat input)
- Click "MCP" or tools button
- Should see two servers: **github**, **azure-devops**

### 3. ADO OAuth Login
- First time using Agent Mode with an ADO tool
- VS Code triggers browser-based login
- Sign in with your **Microsoft account** (same as your ADO account)
- No tokens to manage — OAuth session is kept in memory

### 4. Test Review
- Go to a real PR on GitHub
- Copy PR URL: `https://github.com/owner/repo/pull/123`
- Copy ADO work item ID: `4567`
- In VS Code Agent Mode, type:
  ```
  Review PR: https://github.com/owner/repo/pull/123 against ADO: 4567
  ```
- Agent should:
  1. Fetch ADO work item (shows in chat)
  2. Fetch PR files (shows in chat)
  3. Analyze code (shows 6 passes in chat)
  4. Create review comments (shows API call output)
  5. Refresh GitHub PR — inline comments appear

### 5. Check PR Comments
- Navigate to GitHub PR
- Scroll through changed files
- Inline comments should appear with:
  - Severity emoji (🔴 / 🟡 / 🔵)
  - File + line number
  - Explanation + fix suggestion
  - Link to ADO work item AC (if applicable)

---

## Configuration: Azure DevOps OAuth (Remote Server)

The remote ADO MCP server uses Microsoft account OAuth. On first use:

1. User opens Agent Mode and triggers an ADO tool call
2. VS Code opens a browser login page
3. User signs in with their **Microsoft account** (the same account they use for Azure DevOps)
4. Token is kept in memory for the session

**No PAT or API token is required.** If the user is already signed in to VS Code with a Microsoft account linked to their ADO organization, the flow is seamless.

**Scopes required** (granted automatically via OAuth):
- Read access to ADO work items and projects

**Organization name**: The only configuration needed is the ADO organization name (the part after `dev.azure.com/` in the ADO URL), set as a VS Code input in `.vscode/mcp.json`.

---

## Troubleshooting

### "MCP server 'azure-devops' failed to start"
- Check: VS Code 1.101+
- Check: Agent Mode is toggled ON
- Check: ADO organization name in `.vscode/mcp.json` is correct (no spaces, matches your ADO URL)
- Check: Microsoft account OAuth completed successfully (browser login)

### "Tool 'get_pull_request' not found"
- Check: `.vscode/mcp.json` has GitHub server configured
- Check: VS Code 1.101+
- Check: Agent Mode is toggled ON
- Check: GitHub OAuth login succeeded (first time)

### "Pull request review failed"
- Check: PR URL is valid and public (or you have access)
- Check: File lines in comments are within diff range
- Check: GitHub token has `pull_request:write` scope

### "ADO work item not found"
- Confirm the work item ID is numeric and correct
- On first use, a browser window will open — sign in with your Microsoft account
- Confirm the ADO organization name in `.vscode/mcp.json` matches `dev.azure.com/<org>`
- Confirm you have read access to the ADO project

---

## Future Enhancements

1. **Two-Agent System**: Run separate "Requirement Reviewer" and "Technical Reviewer" agents, then merge findings
2. **Cache PR Analysis**: Store PR diff analysis in workspace for quick re-reviews
3. **Custom Rules**: Allow teams to define org-specific security/performance rules
4. **Metrics**: Track review time, issues found per category, approval rate
5. **CI/CD Integration**: Trigger review automatically on PR creation via GitHub Actions
6. **Multi-Language**: Extend security/performance rules by detected language

---

## Support

For issues or questions:
- GitHub: https://github.com/github/github-mcp-server
- Jira MCP: https://github.com/sooperset/mcp-atlassian
- VS Code MCP Docs: https://code.visualstudio.com/docs/copilot/chat/mcp-servers
