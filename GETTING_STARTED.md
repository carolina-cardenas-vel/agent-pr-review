# Getting Started: GitHub Copilot PR Review Agent

This agent automatically reviews pull requests against Azure DevOps work items using GitHub Copilot in Agent Mode.

## Prerequisites

- **VS Code 1.101+** (required for remote MCP + OAuth)
- **GitHub Copilot** subscription
- **Azure DevOps** organization and access to work items
- Your ADO organization name (the part after `dev.azure.com/` in your ADO URL)

## Setup (5 minutes)

### 1. Find Your ADO Organization Name

Your ADO URL looks like: `https://dev.azure.com/my-company`

The organization name is: `my-company`

### 2. Configure VS Code MCP

The file `.vscode/mcp.json` is already configured. On first use, VS Code will prompt you for:
- **ADO Organization**: Enter the organization name (e.g., `my-company`)

That's it. Both MCP servers (GitHub and ADO) use OAuth — no PATs or API tokens to create.

### 3. Verify Setup

1. Open VS Code (1.101+)
2. Open GitHub Copilot Chat
3. Click the **Agent Mode** toggle (icon next to the chat input)
4. Click **Refresh** on the MCP tools list
5. You should see two servers appear:
   - **github** ✅
   - **azure-devops** ✅

## First Test Review

### 1. Find a PR
- Go to a GitHub PR in your browser
- Copy the PR URL: `https://github.com/owner/repo/pull/123`

### 2. Find the ADO Work Item
- Go to your Azure DevOps organization
- Find a work item
- Copy the work item ID (e.g., `4567`)

### 3. Trigger the Review
In VS Code Agent Mode chat, type:

```
Review PR: https://github.com/owner/repo/pull/123 against ADO: 4567
```

### 4. What Happens

1. **First time login** → Browser opens for ADO OAuth
   - Sign in with your Microsoft account
   - Token is cached for the session
   - No further login needed

2. **Analysis** → Agent fetches:
   - ADO work item (title, description, acceptance criteria)
   - PR files and diffs
   - Full file context
   - Runs 6 specialized review passes:
     - ADO Compliance (vs acceptance criteria)
     - Functional Bugs
     - Security Review
     - Performance Review
     - Test Coverage
     - Maintainability

3. **Review Posted** → Inline comments appear on the PR:
   - 🔴 **Critical** (blocks merge)
   - 🟡 **Warning** (should fix)
   - 🔵 **Suggestion** (nice to have)

## Review Passes Explained

| Pass | Checks | Example |
|------|--------|---------|
| **ADO Compliance** | Each acceptance criterion implemented? | ❌ "Token expiration not validated (AC #2)" |
| **Functional Bugs** | Null handling, error cases, race conditions | ❌ "User could be null (line 52)" |
| **Security** | OWASP Top 10: SQLi, XSS, secrets, auth | 🔴 "SQL injection vulnerability (line 23)" |
| **Performance** | N+1 queries, inefficient loops | 🟡 "1000 queries instead of 1 (line 56)" |
| **Test Coverage** | New code tested? Branches covered? | 🟡 "No tests for new password reset logic" |
| **Maintainability** | Duplication, clarity, tech debt | 🔵 "Extract magic value to constant" |

## Comment Format

Each inline comment includes:

```
🔴 Critical: SQL Injection in customer query (Pass 3: Security)

File: src/db/queries.ts
Line: 23

Issue: The customer ID is directly interpolated into a SQL string.

Risk: Attacker can inject malicious SQL to extract all customer data.

Suggested Fix:
Use parameterized queries:
  db.query('SELECT * FROM orders WHERE customer_id = ?', [customerId])

AC Reference: This addresses AC #1 from ADO #1089
```

## Troubleshooting

### "Tool not found" error
- Toggle Agent Mode OFF, then ON again
- Click "Refresh" on the MCP tools list
- Restart VS Code if it persists

### "ADO work item not found"
- Confirm work item ID is numeric (e.g., `4567`, not `ADO-4567`)
- Confirm you have read access to the ADO project
- Confirm org name in `.vscode/mcp.json` is spelled correctly

### "PR review failed"
- Confirm PR URL is publicly accessible (or you have access)
- Confirm you're logged into GitHub in VS Code
- Check that file lines in comments are within the diff range

### Browser login keeps appearing
- This is normal on first use
- The token is cached for the session
- Sign in once, and you're set for the rest of your session

## What's Implemented

✅ **Configuration**
- `.vscode/mcp.json` — GitHub + ADO MCP servers (remote, OAuth)
- `.env.example` — Reference for ADO organization format

✅ **Agent Instructions**
- `.github/copilot-instructions.md` — Complete workflow with 6-pass review logic

✅ **Documentation**
- `.github/plan.md` — Full implementation spec
- `implementation.md` — Design rationale
- `objective.md` — Project goals

## Next Steps

1. **Test it out** — Run a review on a real PR
2. **Iterate** — Reply to comments in the PR; re-run reviews as needed
3. **Customize** — Edit `.github/copilot-instructions.md` to adjust review criteria
4. **Integrate** — Add to CI/CD pipelines (GitHub Actions) for automatic reviews

## Files Reference

| File | Purpose |
|------|---------|
| `.vscode/mcp.json` | MCP server configuration (GitHub + ADO) |
| `.env.example` | ADO org name format reference |
| `.gitignore` | Prevent accidental commits of `.env` |
| `.github/copilot-instructions.md` | Agent workflow — the core logic |
| `.github/plan.md` | Detailed implementation spec |
| `implementation.md` | Design rationale |
| `objective.md` | Project goals |

## Support

For issues:
- **GitHub MCP**: https://github.com/github/github-mcp-server
- **ADO MCP**: https://github.com/microsoft/azure-devops-mcp
- **VS Code Agent Mode**: https://code.visualstudio.com/docs/copilot/chat/agent-mode

---

**Ready to review?** Open Agent Mode and type:

```
Review PR: <PR_URL> against ADO: <WORK_ITEM_ID>
```
