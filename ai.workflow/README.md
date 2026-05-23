# AI Workflow

This directory contains the operational workflow for AI-assisted development.

## Structure

```text
ai.workflow/
 ├── tasks/
 ├── prompts/
 └── reports/
```

## Workflow

1. ChatGPT creates task specifications.
2. ChatGPT creates Codex prompts.
3. User copies prompt into Codex.
4. Codex performs implementation.
5. Codex creates report inside reports/.
6. User notifies ChatGPT that Codex completed task.
7. ChatGPT reads report directly from GitHub repository.
8. ChatGPT validates implementation.
9. Approved changes are deployed to BCL.

## Important Rule

No deployment to production/BCL before validation.
