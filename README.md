# GitHub Copilot PR Review Agent

A **VS Code Agent** that automatically reviews GitHub pull requests against Azure DevOps work items using GitHub Copilot's AI. The agent performs comprehensive 6-pass code reviews and posts inline comments directly to PRs.

## Why This Project?

Code reviews are critical for quality, security, and maintainability — but they're time-consuming. This agent:

- ✅ **Verifies acceptance criteria** — Ensures every AC from the ADO work item is actually implemented
- 🔒 **Catches security issues** — OWASP Top 10 vulnerabilities, hardcoded secrets, injection attacks
- ⚡ **Identifies performance bugs** — N+1 queries, inefficient loops, memory leaks
- 🐛 **Finds functional bugs** — Null handling, error paths, race conditions
- 📊 **Validates test coverage** — New code, branches, and edge cases tested
- 🎨 **Checks maintainability** — Code duplication, naming, SOLID principles

All while **integrating seamlessly with Azure DevOps** (no separate Jira setup needed).

---

## How It Works

### Activation
In **VS Code Agent Mode**, you type:

```
Review PR: https://github.com/owner/repo/pull/123 against ADO: 4567
```

### The Agent Then:

1. **Fetches ADO work item** (title, description, acceptance criteria)
2. **Fetches PR metadata** (files, diffs, author, base branch)
3. **Analyzes code** through 6 specialized passes:
   - **Pass 1: ADO Compliance** — Maps each AC to implementation
   - **Pass 2: Functional Bugs** — Null handling, error paths, race conditions
   - **Pass 3: Security** — OWASP Top 10 vulnerabilities
   - **Pass 4: Performance** — Inefficient queries and algorithms
   - **Pass 5: Test Coverage** — New code and branches tested
   - **Pass 6: Maintainability** — Duplication, naming, architecture

4. **Posts inline comments** directly to the PR with:
   - 🔴 **Critical** findings (must fix before merge)
   - 🟡 **Warning** findings (should fix)
   - 🔵 **Suggestion** findings (nice to have)

### Example Comment

```
🔴 Critical: SQL Injection vulnerability (Pass 3: Security)

File: src/db/queries.ts
Line: 23

Issue:
The customer ID is directly interpolated into a SQL string, allowing SQL injection.

Risk:
An attacker could inject malicious SQL to extract all customer data or modify records.

Code:
const sql = `SELECT * FROM orders WHERE customer_id = ${customerId}`;

Suggested Fix:
Use parameterized queries:
  db.query('SELECT * FROM orders WHERE customer_id = ?', [customerId])

AC Reference: This addresses AC #1 (Secure customer data) from ADO #1089
```

---

## Key Features

| Feature | Details |
|---------|---------|
| **6-Pass Review** | Specialized analysis for compliance, bugs, security, performance, tests, and code quality |
| **ADO Integration** | Reads work item title, description, and acceptance criteria |
| **GitHub Integration** | Analyzes PRs and posts inline comments at exact diff line numbers |
| **OAuth Authentication** | No tokens to manage; uses Microsoft OAuth for ADO and GitHub OAuth for PRs |
| **AI-Powered** | Leverages GitHub Copilot for intelligent code analysis and actionable feedback |
| **Agent Mode** | Runs directly in VS Code; no setup of separate services or APIs |
| **Lineage Tracking** | Each comment references the specific acceptance criterion it addresses |

---

## Architecture

```
User Types in VS Code Agent Mode
         ↓
    Copilot Agent
         ↓
    ├─→ MCP Tool: Fetch ADO Work Item #4567
    │   (Microsoft OAuth, pulls AC list)
    │
    ├─→ MCP Tool: Fetch PR Files
    │   (GitHub OAuth, pulls diff patches)
    │
    ├─→ MCP Tool: Fetch Full File Context
    │   (GitHub OAuth, ensures architecture understanding)
    │
    └─→ Run 6 Review Passes (in-context LLM analysis)
        ├─ Pass 1: ADO Compliance (AC mapping)
        ├─ Pass 2: Functional Bugs (null checks, error paths)
        ├─ Pass 3: Security (OWASP Top 10)
        ├─ Pass 4: Performance (N+1, algorithms)
        ├─ Pass 5: Test Coverage (unit tests, branches)
        └─ Pass 6: Maintainability (duplication, clarity)
             ↓
        Consolidate findings by severity
             ↓
        Format as inline PR comments
             ↓
    MCP Tool: Post Comments to PR
    (GitHub Review API, posts at exact diff lines)
        ↓
    Comments appear in PR thread
    ↓
    Developer replies and fixes issues
```

---

## Technology Stack

| Component | Technology |
|-----------|-----------|
| **IDE** | VS Code (1.101+) with Agent Mode |
| **AI** | GitHub Copilot LLM |
| **ADO Integration** | microsoft/azure-devops-mcp (official Microsoft MCP server) |
| **GitHub Integration** | github/github-mcp-server (official GitHub MCP server) |
| **Authentication** | OAuth 2.0 (Microsoft for ADO, GitHub for PRs) |
| **Communication** | MCP (Model Context Protocol) over HTTP |
| **Configuration** | `.vscode/mcp.json` (local, no backend) |

---

## Requirements

- ✅ **VS Code 1.101+** (remote MCP + OAuth support)
- ✅ **GitHub Copilot** subscription (AI analysis)
- ✅ **GitHub account** with access to the PR's repo
- ✅ **Azure DevOps account** with access to the work item
- ✅ ADO **organization name** (from `https://dev.azure.com/<org-name>`)

---

## Quick Start

### 1. Clone or Open This Repository

```bash
git clone https://github.com/your-org/agent-pr-review.git
cd agent-pr-review
```

### 2. Open in VS Code (1.101+)

VS Code will automatically detect `.vscode/mcp.json` and register the MCP servers.

### 3. Find a PR and ADO Work Item

- **PR URL**: `https://github.com/owner/repo/pull/123`
- **ADO Work Item ID**: `4567`

### 4. Enable Agent Mode and Trigger Review

1. Open **GitHub Copilot Chat** (⌘ + L)
2. Click the **Agent Mode** toggle (icon next to chat input)
3. Type:
   ```
   Review PR: https://github.com/owner/repo/pull/123 against ADO: 4567
   ```

### 5. First Time Setup

- **Microsoft OAuth**: Browser opens to sign into Azure DevOps
  - Sign in with your Microsoft account
  - Token is cached for your session
- **GitHub OAuth**: Browser opens to authorize GitHub access
  - Authorize Copilot's GitHub access (if not already done)

### 6. Review Posted

Comments appear as inline threads on the PR. You can:
- Reply to discuss the finding
- View the suggested fix
- Push changes and request a re-review

---

## Project Structure

```
agent-pr-review/
├── .vscode/
│   └── mcp.json                    # MCP server configuration
├── .github/
│   ├── copilot-instructions.md     # Complete agent workflow & 6-pass logic
│   └── plan.md                     # Detailed implementation specification
├── .env.example                    # ADO org name format reference
├── .gitignore                      # Prevents committing .env or secrets
├── README.md                       # This file
├── GETTING_STARTED.md              # Quick-start guide
├── objective.md                    # Project goals & scope
├── implementation.md               # Design rationale & architecture
└── LICENSE                         # MIT or Apache 2.0
```

---

## Configuration

The agent is configured in `.vscode/mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "url": "https://api.githubcopilot.com/mcp/",
      "auth": "oauth"
    },
    "azure-devops": {
      "url": "https://mcp.dev.azure.com/${input:ado_org}",
      "auth": "oauth",
      "inputs": {
        "ado_org": {
          "description": "Azure DevOps organization name"
        }
      }
    }
  }
}
```

On first use, VS Code prompts for your **ADO organization name** (the part after `dev.azure.com/` in your ADO URL).

**Note**: No API tokens needed — OAuth handles authentication.

---

## What Gets Reviewed?

### Pass 1: ADO Compliance
- ✅ Each acceptance criterion is implemented
- ✅ AC logic is correct
- ✅ No ACs are missing

**Example Finding**:
```
❌ AC #2: Token expires after 30 minutes
Status: NOT Implemented
Missing: Token expiration validation in validateToken()
```

### Pass 2: Functional Bugs
- ✅ Null/undefined handling
- ✅ Error paths (try-catch)
- ✅ Race conditions (concurrent access)
- ✅ Off-by-one errors
- ✅ Broken logic and invalid assumptions
- ✅ State transitions

**Example Finding**:
```
🔴 CRITICAL: Null pointer in password reset
File: src/auth/reset.service.ts, line 52
Code: const user = await getUser(email);
      return user.email; // ❌ user could be null
Fix: if (!user) throw new UserNotFoundError();
```

### Pass 3: Security
- ✅ SQL Injection (OWASP A3)
- ✅ XSS / Cross-Site Scripting (OWASP A7)
- ✅ Path Traversal / Directory Traversal (OWASP A1)
- ✅ Broken Authentication (OWASP A7)
- ✅ Hardcoded Secrets (API keys, credentials)
- ✅ Insecure Logging (sensitive data)
- ✅ Missing Authorization Checks (privilege escalation)

**Example Finding**:
```
🔴 CRITICAL: SQL Injection vulnerability
File: src/db/queries.ts, line 23
Risk: Attacker can inject SQL to extract/modify data
```

### Pass 4: Performance
- ✅ N+1 query patterns
- ✅ Repeated API calls
- ✅ Large object allocations
- ✅ Inefficient algorithms (O(n²) vs O(n log n))
- ✅ Unbounded queries (no LIMIT)

**Example Finding**:
```
🟡 WARNING: N+1 query in user lookup
File: src/services/cart.service.ts, line 56
For 1000-item cart: fires 1000 queries instead of 1
```

### Pass 5: Test Coverage
- ✅ New functions have unit tests
- ✅ All code branches covered (if/else/try-catch)
- ✅ Error cases tested
- ✅ AC-specific logic validated in tests
- ✅ Edge cases tested

**Example Finding**:
```
🟡 WARNING: Missing tests for new password reset logic
New function: initiateReset(email)
Not tested:
  ✗ User not found
  ✗ Email sending fails
  ✗ Token already active
```

### Pass 6: Maintainability
- ✅ Code duplication
- ✅ Long methods (> 50 lines)
- ✅ Magic values (hardcoded numbers/strings)
- ✅ Poor naming (unclear variable names)
- ✅ Layer violations (UI calling DB directly)
- ✅ Technical debt (TODO, FIXME, HACK comments)
- ✅ SOLID principle violations

**Example Finding**:
```
🔵 SUGGESTION: Extract magic value to constant
File: src/auth/token.service.ts, line 15
Code: const tokenExpiry = 30 * 60 * 1000; // 30 minutes in ms
Fix: const TOKEN_EXPIRY_MS = 30 * 60 * 1000;
```

---

## Severity Levels

| Icon | Level | When to Act | Example |
|------|-------|------------|---------|
| 🔴 | **Critical** | Must fix before merge | SQL injection, data loss, crashes |
| 🟡 | **Warning** | Should fix before merge | Performance bug, missing edge case |
| 🔵 | **Suggestion** | Can fix or defer | Naming clarity, code duplication |

---

## Usage Examples

### Example 1: Simple Bug Fix PR

```
You: Review PR: https://github.com/acme/auth-service/pull/42 against ADO: 1089
Agent: ✅ Review Complete
  - 1 Critical (null pointer)
  - 2 Warnings (test coverage, performance)
  - 1 Suggestion (naming)
Comments posted: Fix the critical issue, then re-run review.
```

### Example 2: Feature Implementation

```
You: Review PR: https://github.com/acme/api/pull/78 against ADO: 2034
Agent: ✅ Review Complete
  - All 5 ACs implemented ✅
  - 2 Security issues found (hardcoded secret, missing auth check)
  - 3 Warnings (test coverage, N+1 query, performance)
  - 5 Suggestions (code clarity, duplication)
Comments posted: Address security issues first, then performance.
```

### Example 3: No Issues

```
You: Review PR: https://github.com/acme/docs/pull/15 against ADO: 999
Agent: ✅ Review Complete
  - All ACs implemented ✅
  - No critical or warning issues
  - 2 Suggestions (code style preference)
Comments posted: Clean code! Two optional improvements offered.
```

---

## Troubleshooting

### "Tool not found"
- Toggle Agent Mode OFF, then ON again
- Click "Refresh" on the MCP tools list
- Verify VS Code is 1.101+

### "ADO work item not found"
- Confirm work item ID is numeric (e.g., `4567`, not `ADO-4567`)
- Confirm you have read access to that ADO project
- Check ADO org name is spelled correctly in `.vscode/mcp.json`

### "Authentication failed"
- This is normal on first use — a browser window will open
- Sign in with your Microsoft account (for ADO) or GitHub account
- Token is cached for your session; no repeat logins needed

### PR review failed
- Confirm PR URL is correct and you have access
- Confirm repository is public or you're logged into GitHub
- Verify file lines in comments are within the diff range

---

## How to Customize

Edit `.github/copilot-instructions.md` to:
- Adjust which OWASP categories to check
- Add custom security rules for your organization
- Change severity thresholds
- Add domain-specific review criteria
- Modify comment formatting

The 6-pass logic is fully configurable—no code changes needed.

---

## Security & Privacy

✅ **No credentials stored** — OAuth tokens are session-only  
✅ **No data sent externally** — MCP servers communicate directly with GitHub/ADO  
✅ **No API keys** — OAuth eliminates PAT/token management  
✅ **Local configuration** — `.vscode/mcp.json` is local; not shared  
✅ **GitHub Copilot standard** — Review analysis uses Copilot's privacy practices

---

## Future Enhancements

- 📊 **Metrics dashboard** — Track review findings by category, severity, repo
- 🔄 **CI/CD integration** — Auto-review all PRs via GitHub Actions
- 🧠 **Learning** — Remember custom rules and apply them to future reviews
- 🌍 **Multi-language** — Review Python, Go, Rust, Java, etc.
- 🔗 **Work item linking** — Auto-link related ADO work items from PR analysis
- ⚙️ **Custom rules** — Define organization-specific security/performance rules

---

## Contributing

Contributions welcome! Areas for improvement:
- Additional review passes (accessibility, documentation, etc.)
- Custom rule definitions
- Performance optimizations
- Better error messages
- Documentation improvements

---

## License

MIT (or Apache 2.0 — choose based on your organization)

---

## Support

- **GitHub Copilot**: https://github.com/features/copilot
- **GitHub MCP Server**: https://github.com/github/github-mcp-server
- **Azure DevOps MCP**: https://github.com/microsoft/azure-devops-mcp
- **VS Code Agent Mode**: https://code.visualstudio.com/docs/copilot/chat/agent-mode

---

## Authors

Built with ❤️ for better code reviews.

---

**Ready to review?** See [GETTING_STARTED.md](GETTING_STARTED.md) for a 5-minute setup guide.
