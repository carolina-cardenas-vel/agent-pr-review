# GitHub Copilot PR Review Agent
## Presentation Outline

---

## Slide 1: Title Slide
**GitHub Copilot PR Review Agent**
- Automating Code Reviews with AI
- Building My First Custom Agent
- 3 months with Copilot

---

## Slide 2: Project Overview - The Problem

**Why we built this:**
- Code reviews are time-consuming and repetitive
- Teams use both GitHub (repos) and Azure DevOps (work items)
- Need to verify requirements are actually implemented
- Security, performance, and test coverage often get missed
- Developers want actionable feedback, not just "fix this"

**The Gap:**
- Existing tools only check one thing at a time
- No tool connects ADO requirements to GitHub code
- Manual reviews slow down PR cycles

---

## Slide 3: Project Overview - The Solution

**What we built:**
- A VS Code Agent that automatically reviews PRs
- Analyzes code against Azure DevOps acceptance criteria
- Runs 6 specialized review passes
- Posts inline comments directly to PRs

**In plain English:**
- You type: "Review PR: <url> against ADO: <work item ID>"
- Copilot analyzes the code
- Comments appear on the PR with findings and fixes

**Outcome:**
- Faster reviews, better code quality, fewer bugs shipped

---

## Slide 4: Architecture Overview

**High-Level Flow:**
1. User triggers review in VS Code Agent Mode
2. Agent fetches ADO work item (requirements)
3. Agent fetches PR files and diffs (code changes)
4. Agent runs 6 analysis passes
5. Agent posts comments to GitHub PR

**Key Components:**
- VS Code Agent Mode (the orchestrator)
- GitHub Copilot (the AI brain)
- Two MCP Servers (integrations):
  - GitHub MCP (for PR analysis and comments)
  - Azure DevOps MCP (for work items and requirements)
- No custom backend needed—everything runs locally in VS Code

---

## Slide 5: The 6 Review Passes

**Pass 1: ADO Compliance**
- Does each acceptance criterion have code?
- Is the logic correct?
- Nothing is missing?

**Pass 2: Functional Bugs**
- Null pointer exceptions?
- Missing error handling?
- Race conditions?

**Pass 3: Security**
- SQL injection vulnerabilities?
- Hardcoded secrets?
- Missing authentication checks?
- (OWASP Top 10)

**Pass 4: Performance**
- N+1 query patterns?
- Inefficient loops?
- Unbounded queries?

**Pass 5: Test Coverage**
- New functions have tests?
- All branches covered?
- Error cases tested?

**Pass 6: Maintainability**
- Code duplication?
- Magic values (hardcoded numbers)?
- Unclear naming?
- Tech debt (TODOs, FIXMEs)?

---

## Slide 6: Example Comment

**What the reviewer sees on the PR:**

```
🔴 Critical: SQL Injection vulnerability (Pass 3: Security)

File: src/db/queries.ts
Line: 23

Issue:
Customer ID is directly interpolated into SQL string.

Risk:
Attacker can inject SQL to steal customer data.

Suggested Fix:
Use parameterized queries:
  db.query('SELECT * FROM orders WHERE customer_id = ?', [customerId])

AC Reference: AC #1 from ADO #1089
```

**Color-coded severity:**
- 🔴 Critical (must fix before merge)
- 🟡 Warning (should fix)
- 🔵 Suggestion (nice to have)

---

## Slide 7: How I Leveraged AI - Discovery Phase

**My journey with Copilot:**
- Started with a problem: "I need a PR review tool"
- Asked Copilot: "Create objective.md for this project"
- Reviewed the output together
- Asked: "What's the implementation approach?"
- Copilot suggested using MCP servers
- Asked: "Create a detailed plan"

**Key insight:**
- Copilot helped me think through the problem
- Each request built on the previous answer
- It was like having a smart colleague brainstorm with me

---

## Slide 8: How I Leveraged AI - Implementation Phase

**Copilot generated:**
1. `.vscode/mcp.json` — Server configuration
2. `.github/copilot-instructions.md` — Agent workflow (400+ lines)
3. `.github/plan.md` — Detailed spec
4. Configuration & setup documentation

**What I didn't have to write:**
- No backend server code
- No API integration code
- No OAuth handling code
- Copilot generated the entire 6-pass review logic

**Total setup time:**
- ~2 hours from idea to working agent
- Most of that was understanding MCP servers
- Actual coding: ~30 minutes

---

## Slide 9: How I Leveraged AI - Quality & Refinement

**Using Copilot for iteration:**
- "Check this for bugs" → Copilot reviewed the instructions
- "Simplify this pass" → Made review logic clearer
- "Add examples" → Generated example comments and workflows
- "Update docs" → Created README, GETTING_STARTED guide

**The power:**
- No trial-and-error coding
- No debugging loops
- Copilot understood the full context from the start
- Could ask for refinements, not rewrites

---

## Slide 10: Key Learnings - MCP Servers

**What I learned about MCP (Model Context Protocol):**
- MCP servers are pre-built integrations (GitHub, ADO, etc.)
- Official ones exist for major platforms
- Check what's available before building custom
- They handle OAuth automatically
- No API tokens to manage or expose

**Key takeaway:**
- MCP servers eliminated an entire layer of complexity
- Would have taken weeks to build custom integrations
- Using existing ones saved massive time

---

## Slide 11: Key Learnings - Agent Mode

**What I learned about VS Code Agent Mode:**
- It's a way to run Copilot with specific instructions
- Activates on trigger text patterns
- Can access MCP tools automatically
- Instructions file (.instructions.md) is incredibly powerful
- No UI coding needed—all in prompts

**Key takeaway:**
- Agent Mode is like giving Copilot a job description
- Clear instructions = predictable behavior
- The agent can orchestrate complex workflows

---

## Slide 12: Key Learnings - Iterative AI-Human Collaboration

**What I learned about working with AI:**

✓ **Start with the problem, not the solution**
- "Here's what I want to build"
- Not: "Build this exact architecture"

✓ **Review each artifact**
- Copilot's output is a starting point, not the final answer
- Small adjustments unlock better results

✓ **Use AI for thinking, not just code**
- Copilot helped me reason through requirements
- It validated my assumptions

✓ **Build on previous outputs**
- Each answer informed the next question
- Compound effect (objective → plan → implementation)

---

## Slide 13: Key Learnings - Prompting Matters

**What makes good prompts:**
- Context is everything (what problem are we solving?)
- Specific examples help (show the desired output format)
- Edge cases matter (mention constraints)
- Iteration beats perfection (good → great → excellent)

**My best results came from:**
- Asking Copilot to review its own work
- Providing feedback on what worked/didn't work
- Refining instructions iteratively
- Being specific about the use case

---

## Slide 14: Gotchas - MCP Server Authentication

**Gotcha #1: OAuth vs API Tokens**

*What I thought:*
- "Need to set up Azure DevOps API tokens"
- "Need to manage GitHub PATs securely"

*What actually happened:*
- MCP servers use OAuth (browser login)
- No tokens stored locally
- Session-based authentication
- First-time setup is just: sign in once

**Lesson:**
- Read the MCP docs first!
- Authentication can be simpler than expected

---

## Slide 15: Gotchas - Line Number Mapping

**Gotcha #2: Diff Line Numbers Are Confusing**

*What I thought:*
- "Just post comments at any line in the file"

*What actually happened:*
- GitHub API requires line numbers from the diff itself
- Can't comment on lines outside the changed section
- Had to map logic to nearest changed line
- Took trial-and-error to get right

**Lesson:**
- Understand the platform APIs early
- Test with real examples
- Comments failing silently is a real problem

---

## Slide 16: Gotchas - Prompt Clarity

**Gotcha #3: Vague Instructions = Confused Agent**

*What I thought:*
- "Just tell Copilot to review the code"

*What actually happened:*
- Generic instructions produced generic comments
- Had to write detailed step-by-step logic
- 6 passes needed to be explicit in order
- Formatting rules had to be exact

**Lesson:**
- Be extremely specific with agent instructions
- Number the steps
- Show examples of desired output
- Test edge cases

---

## Slide 17: Gotchas - Token Limits

**Gotcha #4: Context Gets Expensive**

*What I thought:*
- "Just fetch all the context"

*What actually happened:*
- Large PRs with full file context can exceed token limits
- Copilot has limits on how much context to process
- Had to be selective about which files to fully fetch

**Lesson:**
- Token limits are real constraints
- Prioritize the most important context
- Trade-offs: depth vs. breadth

---

## Slide 18: What I'd Do Differently - Architecture

**If I could start over:**

1. **Start with a real PR**
   - Don't design in a vacuum
   - Test assumptions against actual code early

2. **Limit scope initially**
   - Maybe start with 3 passes, not 6
   - Add complexity incrementally
   - Easier to enhance than simplify

3. **Focus on one problem first**
   - Either "compliance" OR "security" OR "performance"
   - Master one before adding others

---

## Slide 19: What I'd Do Differently - Process

**If I could start over:**

1. **Document assumptions explicitly**
   - "We only review PRs against open work items"
   - "We only support TypeScript"
   - Would save debugging later

2. **Test the feedback loop early**
   - Post real comments to a test PR
   - Get user feedback before refining
   - Currently only tested in theory

3. **Build a feedback mechanism**
   - How do we know if comments are helpful?
   - Track which findings devs actually fix
   - Iterate on what works

---

## Slide 20: What I'd Do Differently - Using Copilot

**If I could start over:**

1. **Ask more verification questions**
   - "Is this the right approach?"
   - "What could go wrong?"
   - Copilot catches issues I miss

2. **Request specific trade-offs**
   - "Fast review vs. deep analysis—which matters more?"
   - "Should we catch more bugs or have fewer false positives?"
   - Would shape the 6 passes differently

3. **Prototype with real MCP responses**
   - Ask Copilot: "What does GitHub MCP actually return?"
   - Test against real API responses early
   - Current design is based on guesses

---

## Slide 21: What I'd Do Differently - Documentation

**If I could start over:**

1. **Architecture diagrams earlier**
   - Visual flow would've caught issues
   - Easier to communicate with team

2. **Use cases & examples upfront**
   - "Here are 3 real PRs we want to review"
   - Design from use cases, not theory

3. **Failure scenarios**
   - "What if MCP server times out?"
   - "What if ADO work item is closed?"
   - Build robustness proactively

---

## Slide 22: Key Wins

**What went really well:**

✅ **MCP servers just worked**
- Official integrations saved weeks
- OAuth was seamless

✅ **Copilot understood the full scope**
- Even complex workflows translated to instructions
- Didn't need to explain context repeatedly

✅ **Rapid iteration was possible**
- From idea to working agent: 2 hours
- From implementation to polish: 1 hour

✅ **No infrastructure needed**
- Everything runs in VS Code
- Zero deployment complexity

---

## Slide 23: Impact & Next Steps

**Current capabilities:**
- Review any PR against any ADO work item
- 6 specialized analysis passes
- Inline comments with actionable fixes
- Supports GitHub + Azure DevOps ecosystem

**Immediate next steps:**
- Deploy and test with real team
- Gather feedback on comment quality
- Measure impact (bugs caught, time saved)
- Iterate on review criteria

**Future vision:**
- Auto-run on every PR (CI/CD integration)
- Custom rules per team
- Metrics dashboard
- Multi-language support
- Learn from fixes (continuous improvement)

---

## Slide 24: Lessons for Your Copilot Journey

**My 3 key takeaways:**

1. **Use Copilot for thinking, not just coding**
   - Best results come from collaboration
   - AI helps you reason through problems
   - Not a replacement for judgment

2. **Understand your tools (MCP, APIs, platforms)**
   - 80% of the work is understanding integrations
   - 20% is actual implementation
   - Invest upfront in learning

3. **Iterate with real feedback loops**
   - Theory beats practice initially
   - Real data changes everything
   - Build to learn, not just to build

---

## Slide 25: Q&A

**GitHub Copilot PR Review Agent**

Questions?

- How does it handle edge cases?
- Can it integrate with other work item systems?
- What's the accuracy of findings?
- How do you customize review criteria?
- Can it run on every PR automatically?

---

## Slide 26: Thank You

**Thank you!**

Presentation Materials:
- README.md — Project overview
- GETTING_STARTED.md — 5-minute setup guide
- implementation.md — Technical architecture
- .github/copilot-instructions.md — Agent logic (400+ lines)

Next: Test it on a real PR and give feedback!
