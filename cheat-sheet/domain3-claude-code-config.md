# Domain 3: Claude Code Configuration & Workflows

**Exam weight: 20%**

→ [Cheat Sheet Index](./README.md) | [Main README](../README.md)

---

## Key Concepts Table

| Concept | What to Know |
|---------|-------------|
| CLAUDE.md | Auto-loaded project context file; read at session start |
| settings.json | Machine config — hooks, permissions, MCP servers, env vars |
| Hook types | `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop` |
| Hook exit codes | 0 = allow/continue, non-zero = block tool call |
| Path prefix | Restrict Bash/file tools to specific directories |
| Permissions | `allow` list in settings.json bypasses confirmation prompts |
| Slash commands | Custom skills in `~/.claude/commands/` or `.claude/commands/` |

---

## CLAUDE.md — Project Context File

Loaded automatically when Claude Code starts in a directory (or any parent).
Controls how Claude understands the project without requiring instructions in every prompt.

```markdown
# Project: E-Commerce API

## Tech Stack
- Runtime: Node.js 20, TypeScript 5
- Framework: Express 4
- Database: PostgreSQL 15 via Prisma ORM
- Tests: Vitest

## Commands
- Dev server: `npm run dev`
- Tests: `npm test`
- Build: `npm run build`
- Lint + fix: `npm run lint:fix`
- DB migrations: `npx prisma migrate dev`

## Architecture
- `src/routes/` — Express route handlers (thin, delegate to services)
- `src/services/` — Business logic (all DB access here)
- `src/middleware/` — Auth, rate limiting, error handling
- Never write SQL directly — use Prisma models in services

## Conventions
- All API responses: `{ success: bool, data?: T, error?: string }`
- Auth required by default — opt out with `@public` JSDoc tag
- Secrets: never hardcode — use `process.env.VARIABLE_NAME`

## Key Constraints
- `src/payments/` — never modify without a second engineer review
- All PII must use `maskPii()` from `src/utils/privacy.ts` before logging
- Test coverage must stay above 80% (`npm run coverage`)
```

**CLAUDE.md loading order (highest → lowest priority):**
1. `~/.claude/CLAUDE.md` (user global)
2. `/repo/CLAUDE.md` (repo root)
3. `/repo/src/CLAUDE.md` (subdirectory)

---

## settings.json — Full Structure

Located at: `~/.claude/settings.json` (user) or `.claude/settings.json` (project)

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run test)",
      "Bash(npm run lint*)",
      "Bash(git status)",
      "Bash(git diff*)",
      "Read(**)",
      "Write(src/**)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)",
      "Write(.env*)"
    ]
  },
  "env": {
    "NODE_ENV": "development",
    "LOG_LEVEL": "debug"
  },
  "mcpServers": {
    "my-db-tools": {
      "command": "python",
      "args": ["/tools/db_server.py"],
      "env": {"DB_URL": "${DATABASE_URL}"}
    }
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": "bash ~/.claude/hooks/validate-bash.sh"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [{
          "type": "command",
          "command": "npm run lint:fix"
        }]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [{
          "type": "command",
          "command": "bash ~/.claude/hooks/log-prompt.sh"
        }]
      }
    ],
    "Stop": [
      {
        "hooks": [{
          "type": "command",
          "command": "bash ~/.claude/hooks/notify-complete.sh"
        }]
      }
    ]
  }
}
```

---

## Hook Types — When Each Fires

| Hook | Fires | Use For |
|------|-------|---------|
| `PreToolUse` | Before tool executes | Validation, auth, blocking dangerous ops |
| `PostToolUse` | After tool executes | Linting, logging, output transformation |
| `UserPromptSubmit` | When user submits prompt | Prompt logging, rate limiting, context injection |
| `Stop` | When Claude finishes turn | Notifications, cleanup, telemetry |

### Hook Exit Codes

```bash
#!/bin/bash
# PreToolUse hook — validate-bash.sh

COMMAND="$CLAUDE_TOOL_INPUT_COMMAND"

# Block destructive commands
if echo "$COMMAND" | grep -qE "^rm -rf|^git push --force|DROP TABLE"; then
    echo "Blocked: destructive command detected: $COMMAND" >&2
    exit 1  # non-zero = block tool call, show error to Claude
fi

exit 0  # allow
```

```bash
#!/bin/bash
# PostToolUse hook — runs after Write tool

# Auto-lint any modified TypeScript file
if [ "$CLAUDE_TOOL_NAME" = "Write" ]; then
    FILE="$CLAUDE_TOOL_INPUT_FILE_PATH"
    if [[ "$FILE" == *.ts || "$FILE" == *.tsx ]]; then
        npx eslint --fix "$FILE" 2>/dev/null
    fi
fi
exit 0
```

---

## Path Prefix Configuration

Restrict Bash/file tools to specific directories:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Read(src/**)",
      "Write(src/**)",
      "Read(tests/**)",
      "Write(tests/**)"
    ],
    "deny": [
      "Write(.env*)",
      "Write(*.pem)",
      "Write(secrets/**)"
    ]
  }
}
```

**Glob patterns:**
- `src/**` — matches `src/` and all subdirectories
- `*.ts` — any TypeScript file in current dir
- `**/*.ts` — any TypeScript file anywhere

---

## Custom Slash Commands

Create project-specific commands as `.md` files:

```
.claude/
  commands/
    deploy.md       ← /deploy
    test-feature.md ← /test-feature
    review-pr.md    ← /review-pr
```

```markdown
<!-- .claude/commands/test-feature.md -->
# Test Feature

Run the full test suite for the feature in $ARGUMENTS, check coverage, and report any failures.

Steps:
1. Run `npm test -- --filter $ARGUMENTS`
2. Run `npm run coverage -- --filter $ARGUMENTS`
3. If coverage drops below 80%, flag it
4. Summarize: passed, failed, coverage %
```

Usage: `/test-feature payment-flow`

---

## Exam Anti-Patterns

| Anti-Pattern | Why Wrong | Fix |
|-------------|-----------|-----|
| Putting all project rules in every prompt | Repetitive; Claude may miss or forget | Put in CLAUDE.md — auto-loaded every session |
| Security validation in CLAUDE.md | CLAUDE.md is advisory context, not enforcement | Use PreToolUse hook with exit code |
| `"deny": []` and relying on prompts to restrict | Prompts don't enforce — hooks and permissions do | Set explicit deny rules in permissions |
| Hook exits 0 even on validation failure | Tool proceeds despite error | Exit non-zero to block; echo error to stderr |
| `PostToolUse` hook modifying files then exiting non-zero | Leaves files in modified state + blocks | Keep PostToolUse hooks idempotent; only exit non-zero when modification should be undone |
| MCP server in settings with wrong `cwd` | Server fails to start silently | Verify `cwd` and use absolute paths |

---

## Quick Reference: Hook Environment Variables

| Variable | Available In | Value |
|----------|-------------|-------|
| `CLAUDE_TOOL_NAME` | Pre/PostToolUse | Tool being called (e.g. `Bash`, `Write`) |
| `CLAUDE_TOOL_INPUT_*` | PreToolUse | Tool input fields (e.g. `CLAUDE_TOOL_INPUT_COMMAND`) |
| `CLAUDE_TOOL_OUTPUT` | PostToolUse | Raw tool output |
| `CLAUDE_USER_PROMPT` | UserPromptSubmit | The user's raw prompt text |

---

👉 **[Get 360 practice questions — Free Enrollment](https://www.udemy.com/course/claude-certified-architect-cca-f-6-practice-tests-2026/?couponCode=FFA33A1350C02189AC15)**

← [Domain 2](./domain2-tool-design-mcp.md) | → [Domain 4](./domain4-prompt-engineering.md)
