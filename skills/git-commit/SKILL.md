---
name: git-commit
description: >
  Create git commits with high-quality messages that capture intent, rationale, and trade-offs.
  Use when the user asks to commit, make a commit, save changes, or invokes /commit.
  Replaces the default commit flow with a structured approach that ensures commit messages
  include: purpose (conventional commit title), why this approach was chosen,
  why alternatives were rejected, and relevant constraints.
---

# Commit Skill

Create git commits with messages that capture not just *what* changed, but *why*.

## Workflow

1. Run `git status` (no `-uall`), `git diff --staged`, `git diff`, and `git log --oneline -10` in parallel
2. Review conversation context for decision rationale — why this approach, what alternatives were considered, what constraints applied
3. If context is insufficient for any of the following, ask the user before proceeding:
   - **Why** this change is being made
   - **Why this approach** over alternatives (if non-obvious)
   - **Constraints** that shaped the implementation (if any)
4. Stage relevant files (prefer explicit file names over `git add -A`)
5. Write the commit message and commit

## Commit Message Format

```
<type>(<scope>): <concise summary>

<Body: free-form paragraphs covering the points below as relevant>

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

### Title

Follow Conventional Commits: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `ci`, `build`, `style`.
Keep under 72 characters. Use imperative mood.

### Body

Write in free-form prose or bullet points — no rigid template. Cover these points **when relevant and non-obvious**:

- **Purpose**: What problem does this solve or what capability does it add?
- **Rationale**: Why was this approach chosen?
- **Alternatives rejected**: What other approaches were considered and why were they not used? (Skip if the approach is straightforward)
- **Constraints**: What technical, business, or time constraints shaped this? (Skip if none)

Do not pad the body with filler. If the change is trivial (typo fix, version bump), a title-only message is fine.

### Examples

Trivial change — title only:
```
fix: correct typo in error message
```

Non-trivial change — with rationale:
```
feat(auth): add JWT refresh token rotation

Rotate refresh tokens on each use to limit the window of a leaked token.
Considered using short-lived access tokens only (no refresh), but that
would force re-login every 15 minutes which is unacceptable for the
mobile app's UX. Redis-based token blacklisting was also considered but
adds an infrastructure dependency we want to avoid at this stage.
```

## Rules

- Never skip hooks (`--no-verify`) or bypass signing unless the user explicitly asks
- Never amend unless the user explicitly asks — always create new commits
- If a pre-commit hook fails, fix the issue and create a **new** commit
- Do not commit files that likely contain secrets (`.env`, credentials, etc.)
- Pass commit messages via HEREDOC for proper formatting
- Do not push unless the user explicitly asks
