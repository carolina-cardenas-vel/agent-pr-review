I want to create a custom agent using GitHub Copilot in VS Code. The agent should accept a Pull Request (PR) URL and the corresponding Jira ticket as inputs. It should automatically analyze all files changed in the PR, compare the implementation against the Jira ticket requirements and acceptance criteria, and perform a comprehensive code review.

The review should determine:

- Whether the implementation fully satisfies the Jira ticket requirements.
- Whether any acceptance criteria have been missed or only partially implemented.
- Potential bugs, edge cases, or functional issues.
- Security risks or dangerous coding patterns.
- Performance concerns.
- Maintainability and code quality issues.
- Opportunities for refactoring or improvement.
- Whether adequate tests have been added or updated.

Instead of generating a standalone review report, the agent should post review comments directly on the relevant lines of code in the PR for each issue it identifies. These comments should provide clear explanations and actionable suggestions, allowing the developer to review the feedback, reply to the comments, and engage in a discussion within the PR review process.

What would be the best approach to design and implement such an agent using GitHub Copilot in VS Code?