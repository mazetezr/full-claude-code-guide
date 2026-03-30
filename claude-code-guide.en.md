# Claude Code — Features & Workflow Guide

> Compiled from the official CHANGELOG, public community materials, and bcherny's thread (March 2026).
> Claude Code version at the time of compilation: **2.1.78**

---

## 🏷️ Priority System

Each section is tagged with a flag indicating how important it is when setting up Claude Code:

| Flag | Level | Meaning |
|------|-------|---------|
| 🔴 | **Essential** | Set this up first. Without it, the agent performs significantly worse — you lose quality on every prompt. Applicable to any project and stack. |
| 🟡 | **Situational** | A powerful tool, but not for everyone. Adopt it when your project reaches a certain scale or the task matches the feature's profile. Specific situations are described below. |
| 🟢 | **Nice-to-have** | A convenient feature, but doesn't affect code quality. Saves some people 5 minutes a day, others don't need it at all. Try it — decide for yourself. |

> **Recommendation:** start with the 🔴 sections, then go through the 🟡 ones and mark those relevant to your project.

---

## 1. 🔴 Agent Code Quality

### Never typecast (`as`)

**Why:** A single line in `AGENTS.md` that improves code quality more than any skill. Banning `as` forces the agent to solve type problems correctly instead of silencing them.

**How to implement:**

Add to `AGENTS.md`:
```markdown
## Code Standards
- Never typecast. Never use `as`
```

For automated enforcement — Biome plugin:
```
https://github.com/albertodeago/biome-plugin-no-type-assertion
```

📌 Real-world example

**Situation:** You're building a Next.js app with Prisma ORM. The agent receives data from an API and needs to convert `unknown` to the `User` type. Without the ban, it would write:
```typescript
const user = data as User; // silently breaks if the API returns a different structure
```

With the `as` ban, the agent is forced to write runtime validation:
```typescript
const user = UserSchema.parse(data); // Zod — fails explicitly if the structure doesn't match
```

**Result:** instead of a hidden bug in production — an error during development.

**Who needs this:** all TypeScript projects — from a Next.js landing page to an enterprise monolith. The larger the codebase, the more critical it is. For Python/Go/Rust projects — formulate analogous rules for your stack (e.g., `Never use # type: ignore` for Python).

---

## 2. 🔴 Skills Architecture (harness optimization)

**Why:** A strict Skills structure provides ~90% improvement in result stability. A mediocre agent inside a rigid constraint system performs better than a powerful agent in a poorly structured environment. This isn't prompt engineering — it's system design.

**Principle:** one Skill = one function. The result = a combination of atomic Skills.

**SKILL.md structure:**

```markdown
---
name: my-skill
description: Short description for triggering
---

## Skill Purpose
One paragraph. The essence of the task. Only the main idea — details go below.

## Instructions
1. Exact step with exact path
   Execute: ~/scripts/run.py
2. Next step
3. ...

> Rule: if there are more than 3 steps — split into separate Skills.

## Non-Negotiable Acceptance Criteria
- [ ] Criterion 1 — the agent does not produce a result until this is met
- [ ] Criterion 2
- [ ] Criterion 3

> Use "Non-Negotiable" specifically, not "Rules" — the latter leaves the agent room for interpretation.

## Output
Exact response format. Required for stable Skills chaining.
```

**Unstable results = weak harness.** If the agent behaves unpredictably — the problem is in the structure, not the model.

📌 Real-world example

**Situation:** You have a SaaS product. Every week you need to: create a DB migration → update API endpoints → update frontend types → write tests. Without Skills, the agent does this slightly differently each time — sometimes forgets tests, sometimes writes the migration in the wrong format.

With Skills, you create:
```
.claude/skills/
├── db-migration.md      # Skill: create a migration in Prisma format
├── api-endpoint.md      # Skill: CRUD endpoint from template
├── generate-types.md    # Skill: update TypeScript types from schema
└── integration-test.md  # Skill: test for a new endpoint
```

Each Skill has Non-Negotiable criteria:
```markdown
## Non-Negotiable Acceptance Criteria
- [ ] Migration includes rollback (down migration)
- [ ] All fields have explicit types, no `any` or `unknown`
- [ ] Ran `prisma migrate dev` and migration completed without errors
```

**Result:** a repeatable, predictable pipeline. The same quality every time.

**Who needs this:** everyone who uses Claude Code regularly (not as a one-off). Especially teams where multiple people use the agent — Skills standardize the approach. Projects with recurring patterns (CRUD, migrations, code generation) benefit the most.

---

## 3. 🟡 Parallel Work: Git Worktrees

**Why:** Run dozens of agents in a single repository without conflicts. Each agent gets its own isolated worktree and works independently.

**How to implement:**

```bash
# New session in an isolated worktree
claude -w

# or
claude --worktree
```

In the Desktop app — check the "worktree" checkbox when creating a session.

For non-git VCS — a `WorktreeCreate` hook for your own worktree creation logic.

For agents in `.claude/agents/*.md`:
```yaml
---
isolation: worktree
---
```

Documentation: `https://git-scm.com/docs/git-worktree`

📌 Real-world example

**Situation:** You're a lead on a team of 3 frontend developers. You have a React app with 50+ components. You need to simultaneously: refactor authentication, build a new cart feature, and fix bugs in the catalog. Without worktrees — three tasks in one file tree, conflicts are inevitable.

With worktrees:
```bash
# Terminal 1 — auth refactoring
claude -w
> "Refactor the auth module, migrate from context to zustand"

# Terminal 2 — new feature
claude -w
> "Create a cart module with Cart, CartItem, CartSummary components"

# Terminal 3 — bugfix
claude -w
> "Fix the pagination bug in the catalog — items.length on page 2 is always 0"
```

Each agent works in an isolated copy of the repo. No conflicts. At the end — three PRs for review.

**Who needs this:**
- Projects with multiple parallel tasks (features + bugfixes + refactoring)
- Teams where multiple people use Claude Code on the same repo
- Monoliths with long PR cycles
- Situations where you want to try two approaches to the same task and compare

**When you don't need it:** solo development with one task at a time.

---

## 4. 🟡 Mass Changes: /batch

**Why:** Large code migrations and other parallelizable tasks. `/batch` interviews you, then splits the work and launches as many worktree agents as needed — dozens, hundreds, thousands.

**How to implement:**

```
/batch
```

Claude will ask clarifying questions, determine the decomposition itself, and launch agents on its own.

Use for: migrations, refactorings, mass renames, dependency updates.

📌 Real-world example

**Situation:** The company decided to migrate from Material UI v4 to v5. 200+ components, each one requiring import API changes, replacing `makeStyles` with `styled`, updating props. Manually — a week of work. One file at a time with an agent — still slow.

```
/batch
> Claude: What's the task?
> You: Migrate all components from MUI v4 to MUI v5.
>       Replace makeStyles with styled/sx, update imports from @material-ui to @mui.
> Claude: Found 217 files. Splitting into 43 groups by module. Launching agents...
```

Claude automatically:
1. Finds all affected files
2. Groups them by logical modules (so related files change together)
3. Launches parallel agents (each in its own worktree)
4. Collects results

**Result:** 200+ files migrated in 20 minutes instead of a week.

**Who needs this:**
- Large framework/library migrations (React Router v5→v6, Vue 2→3, Angular upgrade)
- Mass refactorings (renaming an entity across the entire codebase, changing a pattern)
- API contract updates when 50+ files change
- Monorepos with dozens of packages

**When you don't need it:** targeted changes in 1-5 files — a regular session handles that faster.

---

## 5. 🟡 Automation: /loop and /schedule

**Why:** Turn Claude Code from a tool into a background process. The agent works on its own on a schedule while you focus on other things.

**How to implement:**

```bash
# Basic syntax
/loop <interval> <command>

# Practical examples from bcherny:
/loop 5m /babysit          # auto code review + rebase + shepherd PRs
/loop 30m /slack-feedback  # PR for Slack feedback every 30 min
/loop /post-merge-sweeper  # PRs for missed review comments
/loop 1h /pr-pruner        # closing stale PRs
```

Scheduled Tasks via Desktop sidebar: Schedule → New task → prompt + time + frequency.

Or in natural language: *"set up a daily code review every morning at 9am"* — Claude will configure it itself.

**Limitation:** the machine must be awake, Desktop must be open.

📌 Real-world example

**Situation:** You're a tech lead at a startup. A team of 5 developers, 8-12 PRs every day. You physically can't review everything, PRs sit for 2-3 days. Quality drops, conflicts grow.

You set up two automated processes:
```bash
# Every 5 minutes — check open PRs, rebase if needed, leave a review
/loop 5m /babysit

# Every day at 9 AM — report on stale PRs older than 48 hours
# Via Desktop: Schedule → "Review stale PRs older than 48h, ping authors in comments"
```

Or a more specific workflow:
```bash
# Every hour — check that tests pass on all open PRs
/loop 1h "Check all open PRs, if tests fail — leave comment with diagnosis"
```

**Result:** PRs get reviewed within minutes. Stale branches don't pile up. You focus on architectural decisions, not routine.

**Who needs this:**
- Teams with active PR flow (5+ PRs/day)
- Tech leads who can't keep up with reviews
- Projects with CI/CD that need constant monitoring
- Open-source maintainers with high contribution volume

**When you don't need it:** solo projects, projects without a PR process, early prototypes.

---

## 6. 🟡 Hooks — Deterministic Logic in the Agent Loop

**Why:** Hooks let you attach your own logic to any point in the agent's workflow. This isn't a prompt — it's guaranteed execution.

**Events:**

| Event | When it triggers |
|---|---|
| `SessionStart` | Context loading at startup |
| `PreToolUse` | Before every tool call |
| `PostToolUse` | After every tool call |
| `Stop` | When the agent stops |
| `PermissionRequest` | On permission request |
| `SessionEnd` | Session termination |
| `WorktreeCreate/Remove` | Creating/removing a worktree |
| `StopFailure` | API error (rate limit, auth) |

**Usage examples:**

```bash
# Dynamic context loading at startup
SessionStart → script reads current data and passes it to the prompt

# Logging all bash commands
PreToolUse(Bash) → logs the command

# Approving dangerous operations from your phone
PermissionRequest → POST to WhatsApp/Telegram → waits for response

# Preventing the agent from stopping
Stop → pings the agent to continue working
```

Documentation: `https://code.claude.com/docs/en/hooks`

📌 Real-world example

**Situation:** You have a fintech project with a microservices architecture. Each service has its own environment variables, its own DB schema, its own dependencies. When the agent starts a session — it needs current context: which services are running, which migrations are applied, which feature flags are active.

You set up a `SessionStart` hook:
```json
// .claude/settings.json
{
  "hooks": {
    "SessionStart": [{
      "command": "python scripts/claude-context.py",
      "timeout": 10000
    }]
  }
}
```

```python
# scripts/claude-context.py
# Collects current context and outputs to stdout — Claude will read it
import subprocess, json

# Which services are running
services = subprocess.check_output(["docker", "compose", "ps", "--format", "json"])
# Latest migration
migration = subprocess.check_output(["prisma", "migrate", "status"])
# Active feature flags
flags = json.load(open("config/features.json"))

print(f"## Current Environment State")
print(f"Running services: {services.decode()}")
print(f"DB migration status: {migration.decode()}")
print(f"Active feature flags: {json.dumps(flags, indent=2)}")
```

Or a simpler example — logging for audit:
```json
{
  "hooks": {
    "PreToolUse": [{
      "command": "python scripts/log-action.py",
      "timeout": 5000
    }]
  }
}
```

**Result:** the agent always works with up-to-date context. All actions are logged. Dangerous operations require confirmation.

**Who needs this:**
- Projects with dynamic context (microservices, feature flags, multi-tenant)
- Teams with audit and compliance requirements (fintech, medtech)
- Projects where the agent works with production data and needs a safety layer
- CI/CD integrations where the agent needs to know the pipeline state

**When you don't need it:** simple projects with a single service and static configuration.

---

## 7. 🟢 Mobile App + Remote Control

**Why:** Write and review code without a laptop. Control local sessions from your phone.

**How to implement:**

Mobile app:
- iOS/Android → Code tab

Session management:
```bash
# Transfer a cloud session to the terminal
claude --teleport
# or from within a session:
/teleport

# Control a local session from phone/web
/remote-control
```

Recommendation: enable "Enable Remote Control for all sessions" in `/config` — then every session is automatically available remotely.

📌 Real-world example

**Situation:** You launched a long migration from your phone on the way to work. Or you're on a call and want to quickly check the agent's status on your computer — without switching away from Zoom.

```bash
# On your laptop you started a heavy task
claude> "Refactor all API endpoints to the new authorization scheme"
# Agent is working...

# From your phone you open the Claude Code app → see the session →
# type: "Show progress, which files have been changed?"
# or: "/btw don't forget to update tests for /api/users"
```

Or the reverse scenario — started on phone, continued on computer:
```bash
# On phone: "Create a skeleton for a new microservice notification-service"
# 10 minutes later at your desk:
claude --teleport  # picked up the session from phone in the terminal
```

**Who needs this:**
- Developers who are often on the go or in meetings
- Tech leads who want to monitor agents from their phone
- Those who launch long tasks and want to check status without switching context

**Who probably doesn't need this:** those who work exclusively at a desktop in the office.

---

## 8. 🟢 Session Branching

**Why:** Try an alternative approach without losing the current context. Or give one task to two branches and compare the result.

**How to implement:**

```bash
# From within a session
/branch

# From CLI
claude --resume <session-id> --fork-session
```

📌 Real-world example

**Situation:** You're discussing with the agent the architecture of a new notification module. The agent suggested using Redis pub/sub. You're thinking — maybe WebSocket directly would be better? You don't want to lose the current discussion context.

```bash
# Current session — Redis approach
claude> "Let's implement it via Redis pub/sub"
# Agent starts...

# Create a branch — WebSocket approach
/branch
claude> "Now let's try via WebSocket directly, without Redis"
# Agent implements the alternative

# Compare both results, choose the better one
```

Another scenario — "what if":
```bash
# Agent wrote a component with CSS Modules
# You want to see how a Tailwind version would look
/branch
claude> "Rewrite this component using Tailwind instead of CSS Modules"
```

**Who needs this:**
- Architectural decisions where you want to compare two approaches
- "What if" situations without risking losing current work
- Experimenting with different technologies/libraries for the same task

**Who probably doesn't need this:** linear development without experiments.

---

## 9. 🟢 /btw — Parallel Questions

**Why:** Ask a quick question while the agent is working, without interrupting it.

**How to implement:**

```
/btw what is X?
```

The agent will answer on the side and continue the main task.

📌 Real-world example

**Situation:** The agent is writing a complex CSV file parsing module. You're reading its code in parallel and see an unfamiliar construct. You don't want to interrupt — it's in the middle of something.

```bash
claude> "Write a CSV parser with support for custom delimiters and escaping"
# Agent is working, writing code...

/btw why are you using a generator instead of a regular array?
# Agent quickly explains: "A generator doesn't load the entire file into memory —
# for 500MB files this is critical" — and continues the main task

/btw is TextDecoder supported in Node 16?
# Quick answer, main work is not interrupted
```

**Who needs this:**
- Those who learn from the agent's code and want to understand decisions in real time
- Long sessions where the agent works for 5-10 minutes and you want to ask a clarification
- "Just remembered something" situations in the middle of generation

**Who probably doesn't need this:** short sessions where it's easier to wait until the end and ask.

---

## 10. 🟡 Chrome Extension for Frontend

**Why:** Give the agent "eyes" — the ability to see the result of its work and iterate. Key principle: an agent without a browser can't produce good frontend, just like an engineer without a browser.

**How to implement:**

Install the extension: `https://code.claude.com/docs/en/chrome`

Works in Chrome and Edge. After installation — the agent launches a server on its own, opens the browser, sees the result, and iterates.

📌 Real-world example

**Situation:** You ask the agent to build a landing page from a Figma mockup. Without the extension — the agent writes HTML/CSS blindly, you manually check and say "the button is too big, the padding is wrong." With the extension — the agent sees the result itself.

```bash
claude> "Build the hero section: heading, subheading, CTA button, background gradient.
        Mockup: https://figma.com/file/xxx"
```

What happens with the extension:
1. Agent writes HTML + CSS
2. Launches `npm run dev`
3. Opens Chrome → takes a screenshot of the page
4. Analyzes: "Button is not centered, adding `margin: 0 auto`"
5. Reloads → takes a new screenshot
6. "Now it's OK, but the heading font is too small on mobile"
7. Adds a media query → checks → done

**Result:** instead of 5 iterations of "you → agent → you → agent" — the agent polishes it to the pixel on its own.

**Who needs this:**
- Any frontend development: React, Vue, Svelte, plain HTML
- Building layouts from mockups (Figma, Sketch)
- Fixing visual bugs ("it shifts on mobile", "looks different in Safari")
- CSS animations and interactive elements

**When you don't need it:** backend projects, CLI utilities, libraries without UI.

---

## 11. 🟡 Desktop: Web App Preview

**Why:** A built-in browser in the Desktop app for automatically launching and testing web servers — no manual setup required.

Documentation: `https://code.claude.com/docs/en/desktop#preview-your-app`

📌 Real-world example

**Situation:** You're working in Claude Code Desktop and building a React app. Instead of switching between Desktop and Chrome — the built-in preview shows the result right in the Claude window.

```bash
claude> "Create a registration form with field validation"
# Agent writes code → automatically launches dev server →
# built-in preview shows the result right in the Desktop app →
# agent sees the result and iterates
```

Works like the Chrome Extension (#10), but without installing an extension — everything is inside Desktop.

**Who needs this:**
- Claude Code Desktop (not CLI) users with frontend projects
- Those who want "everything in one window" without switching
- Rapid prototyping — see the result instantly

**Alternative:** if you work in CLI — use the Chrome Extension (#10) instead.

---

## 12. 🟡 --bare for SDK (up to 10x speedup)

**Why:** By default, every `claude -p` invocation searches for CLAUDE.md, settings, MCPs — that's unnecessary work for non-interactive usage. `--bare` disables auto-discovery.

**How to implement:**

```bash
claude -p --bare --system-prompt "..." --mcp-config ./mcp.json "your task"
```

Anthropic plans to make `--bare` the default in future versions. For now — it's opt-in.

📌 Real-world example

**Situation:** You've integrated Claude Code into a CI/CD pipeline. On every PR, a code review is automatically run via `claude -p`. The problem: each invocation spends 5-10 seconds searching for and loading CLAUDE.md, settings, connecting MCP servers — that's unnecessary for a scripted call.

```bash
# Before (slow — 12 sec startup):
claude -p "Review this diff: $(git diff main...HEAD)"

# After (fast — 1-2 sec startup):
claude -p --bare \
  --system-prompt "You are a code reviewer. Focus on bugs, security, performance." \
  "Review this diff: $(git diff main...HEAD)"
```

A more complex example — documentation generation pipeline:
```bash
#!/bin/bash
# generate-docs.sh — runs in CI on every merge to main
for file in src/api/*.ts; do
  claude -p --bare \
    --system-prompt "Generate JSDoc for the given TypeScript file. Output only the updated file." \
    "$(cat $file)" > "${file}.documented"
done
```

**Result:** the CI step takes 20 seconds instead of 2 minutes. A clean scripted call with no overhead.

**Who needs this:**
- CI/CD integrations (GitHub Actions, GitLab CI, Jenkins)
- Scripts and pipelines that call Claude Code dozens/hundreds of times
- SDK usage: custom tools built on top of the Claude Code CLI
- Batch file processing where each call needs to be fast

**When you don't need it:** interactive terminal work — that's where CLAUDE.md and MCPs are needed.

---

## 13. 🟡 --add-dir for Multi-Repo Workflow

**Why:** Give the agent access to multiple repositories in a single session.

**How to implement:**

```bash
# At launch
claude --add-dir ../other-repo

# From within a session
/add-dir ../other-repo
```

For a team — add to `settings.json`:
```json
{
  "additionalDirectories": ["../shared-lib", "../api"]
}
```

Documentation: `https://code.claude.com/docs/en/cli-reference`

📌 Real-world example

**Situation:** A typical startup architecture — three repositories:
```
~/projects/
├── frontend/     # React SPA
├── backend/      # Node.js API
└── shared-types/ # TypeScript types shared by both
```

You're changing an API endpoint in `backend/` and need to update types in `shared-types/` and the call in `frontend/`. Without `--add-dir` — three separate sessions, manual coordination.

```bash
cd ~/projects/backend
claude --add-dir ../shared-types --add-dir ../frontend

claude> "Add the `avatar_url` field to User:
        1. Update the type in shared-types/src/user.ts
        2. Add the field to the API response in backend/src/routes/users.ts
        3. Display the avatar in frontend/src/components/UserProfile.tsx"
```

The agent sees all three repos and makes coordinated changes in a single session.

Another example — a monorepo with Turborepo/Nx:
```bash
cd packages/web
claude --add-dir ../api --add-dir ../shared
```

**Who needs this:**
- Multi-repo architectures (frontend + backend + shared)
- Monorepos with interdependent packages
- Situations where a change in one place requires an update in another
- Microservices with a shared contract (protobuf, OpenAPI schema)

**When you don't need it:** a single repository, self-contained projects.

---

## 14. 🟡 --agent for Custom Agents

**Why:** Launch Claude Code with a custom system prompt and a specific set of tools. A powerful primitive for creating specialized agents for specific tasks.

**How to implement:**

Create a file `.claude/agents/my-agent.md`:
```markdown
---
name: my-agent
description: What this agent does
tools: Bash, Read, Write
model: sonnet
---

You are a specialized agent for...
```

Launch:
```bash
claude --agent=my-agent
```

Documentation: `https://code.claude.com/docs/en/sub-agents`

📌 Real-world example

**Situation:** Your team has recurring tasks: reviewing SQL migrations, generating API documentation, creating components from templates. Explaining the context to the agent each time is a waste of time. Custom agents solve this.

Example — an agent for reviewing SQL migrations:
```markdown
# .claude/agents/sql-reviewer.md
---
name: sql-reviewer
description: Review SQL migrations for safety and performance
tools: Read, Bash
model: sonnet
---

You are a PostgreSQL migration expert. When you receive a migration:

1. Check reversibility (is there a DOWN migration)
2. Check for locks: ALTER TABLE on large tables without CONCURRENTLY — red flag
3. Check indexes: new columns with WHERE queries should have an index
4. Check defaults: NOT NULL without DEFAULT on an existing table = downtime

## Non-Negotiable
- If a blocking ALTER is found — output a WARNING with a safe version suggestion
- Always show estimated lock time for each operation
```

Usage:
```bash
claude --agent=sql-reviewer
> "Review the migrations in db/migrations/2026-03-*"
```

Another example — an agent for creating React components:
```markdown
# .claude/agents/component-factory.md
---
name: component-factory
description: Creates a React component following the project template
tools: Read, Write, Bash
---

Create a React component. Required:
1. TypeScript, named export, no default export
2. Styles via CSS Modules (.module.css)
3. Storybook story alongside (ComponentName.stories.tsx)
4. Unit test alongside (ComponentName.test.tsx)
5. Barrel export via index.ts
```

**Who needs this:**
- Teams with standardized processes (reviews, generation, checks)
- Projects with templates: "every new module looks like this"
- Companies with domain-specific rules (fintech, medtech, gamedev)
- Onboarding: a new developer launches an agent instead of reading 50 pages of docs

**When you don't need it:** one-off tasks, prototyping, small projects without templates.

---

## 15. 🟢 /voice — Voice Input

**How to implement:**

- CLI: `/voice` → hold spacebar
- Desktop: voice button
- iOS: enable dictation in system settings

📌 Real-world example

**Situation:** You're on the couch with your laptop, thinking about architecture. It's easier to speak your thoughts aloud than to type a long prompt.

```bash
/voice
# Hold spacebar and say:
# "I need a notification module. It should support three channels:
#  email via SendGrid, push via Firebase, and SMS via Twilio.
#  Each channel — a separate adapter behind a single interface.
#  Queue on Redis, retry with exponential backoff."
```

The agent receives the text and starts working. Faster than typing 5 sentences.

Also convenient for describing bugs:
```bash
/voice
# "On the profile page, if the user hasn't uploaded an avatar,
#  a broken image shows instead of a default icon.
#  This only happens on mobile resolution, on desktop it's fine."
```

**Who needs this:**
- Those who prefer to think out loud
- Long, descriptive prompts (bug reports, architectural decisions)
- Working from the mobile app

**Who probably doesn't need this:** those who type faster than they speak, working in a noisy office.

---

## 16. 🟡 gstack — Specialized Cognitive Modes

**Why:** Instead of one generic agent — a team of specialists. Each skill — strictly one role. Key principle: explicit "cognitive modes" produce predictable results.

**Installation** (one command in Claude Code):
```
Install gstack: run `git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup`
```

**Modes:**

| Command | Role | What it does |
|---|---|---|
| `/plan-ceo-review` | Founder/CEO | Finds the 10-star product inside the request |
| `/plan-eng-review` | Eng Manager | Architecture, diagrams, edge cases |
| `/review` | Paranoid Staff Eng | Bugs that pass CI but break prod |
| `/ship` | Release Engineer | sync + tests + PR in one command |
| `/browse` | QA Engineer | Headless Chromium, ~100ms, sees UI |
| `/qa` | QA Lead | Diff-aware: reads git diff, tests affected pages |
| `/retro` | Eng Manager | Team retrospective with metrics |

Repository: `https://github.com/garrytan/gstack`

📌 Real-world example

**Situation:** You're building a B2B SaaS. New feature — an analytics dashboard. You want to go through the full cycle from idea to PR, using specialized roles.

**Step 1 — Product review:**
```bash
/plan-ceo-review
> "Analytics dashboard: revenue, churn, MRR charts. Filters by date and segment."
# CEO-mode: "Remove churn from the main screen — it scares people.
#  Add 'Net Revenue Retention' — it shows growth.
#  The first screen should evoke 'wow, we're growing'."
```

**Step 2 — Engineering review:**
```bash
/plan-eng-review
# Eng Manager: draws architecture, finds edge cases:
# "What if there's no data for the period? Empty chart or a message?
#  Date filter: client-side or server-side? With 100k records — server-side only."
```

**Step 3 — Implementation + review:**
```bash
# You write code...

/review
# Paranoid Staff Eng: "You're computing MRR on the client by summing all subscriptions.
#  With 10k subscriptions that's 2 seconds of UI blocking.
#  Move it to a SQL aggregation or Web Worker."
```

**Step 4 — QA + Ship:**
```bash
/qa     # Tests affected pages in headless browser
/ship   # Builds, runs tests, creates PR
```

**Result:** a complete pipeline from product thinking to PR, each stage — a specialized agent.

**Who needs this:**
- Product teams where both UX and code quality matter
- Startups where one person = CEO + CTO + IC: gstack provides a "virtual team"
- Projects with frontend where visual QA matters (headless browser)
- Teams practicing a structured review process

**When you don't need it:** pure backend without UI, small utilities, one-off scripts.

---

## 17. 🔴 Thinking — Show the Reasoning Process

**Why:** Since v2.0.0, thinking is hidden by default. To debug and understand the agent's decisions — enable it explicitly.

**How to enable:**

In `settings.json`:
```json
{
  "alwaysThinkingEnabled": true
}
```

Or in a session: `Ctrl+O` → transcript mode.

Controlling thinking depth:
```
/effort low|medium|high
```

📌 Real-world example

**Situation:** The agent is doing something strange — refactoring a file you didn't ask about, or choosing an incomprehensible solution. Without thinking — it's a black box, unclear why.

```bash
# Without thinking:
claude> "Optimize the DB query in users.service.ts"
# Agent silently rewrites 3 files, changes ORM calls, adds caching.
# You: "Why caching? I only asked to optimize the query!"

# With thinking (alwaysThinkingEnabled: true):
claude> "Optimize the DB query in users.service.ts"
# 💭 Thinking: "The query at users.service.ts:42 is doing N+1 — for each user
#   it loads orders with a separate query. I could solve this with a JOIN, but
#   this same query is used in the cache layer (users.cache.ts),
#   which expects a flat user object without orders. If I add a JOIN,
#   I'll break the cache. Two options:
#   1. JOIN + update the cache layer
#   2. Eager loading via ORM relation
#   Option 2 is simpler and doesn't break the contract."
# Now you understand WHAT and WHY it's doing.
```

Controlling thinking depth based on the task:
```bash
/effort low     # Quick tasks: "rename variable", "add import"
/effort medium  # Standard: features, refactoring, bugs
/effort high    # Complex tasks: architecture, security audit, debugging race conditions
```

**Who needs this:** everyone. This is a fundamental setting for understanding and controlling the agent:
- **Debugging unexpected behavior** — understand why the agent made a specific decision
- **Learning** — learn from the agent's reasoning (especially for junior developers)
- **Review** — ensure the agent understood the task correctly before it writes 500 lines of code
- **Complex tasks** — architectural decisions, security, performance optimization

`/effort` is useful for everyone: don't spend compute on trivial tasks, invest maximally in complex ones.

---

## 18. 🔴 CLAUDE.md as the Primary Control Document

**Why:** CLAUDE.md is the main tool for configuring agent behavior. It loads automatically.

**Structure:**

```markdown
## Project Overview
Brief project description

## Code Standards
- Never typecast. Never use `as`
- [other standards]

## Architecture
Key decisions that don't need to be reinvented

## Workflow
How we work in this project
```

HTML comments `<!-- -->` in CLAUDE.md are hidden from the agent during auto-loading, but visible when explicitly read via the Read tool. Use them for notes to yourself.

📌 Real-world example

**Situation:** You have a Next.js e-commerce project. Without CLAUDE.md, every session starts from scratch — the agent doesn't know the project structure, conventions, or architectural decisions. You explain the same things over and over.

With CLAUDE.md — the agent starts with full context:

```markdown
# CLAUDE.md

## Project Overview
E-commerce on Next.js 14 (App Router) + Prisma + PostgreSQL + Stripe.
Turborepo monorepo: apps/web, apps/admin, packages/shared.

## Code Standards
- Never typecast. Never use `as` — use Zod for runtime validation
- All API routes via Route Handlers (app/api/), not Pages API
- Styles: Tailwind CSS, no CSS-in-JS
- State: Zustand for client, React Query for server data
- Forms: react-hook-form + zod resolver

## Architecture
- Auth: NextAuth.js with JWT strategy, middleware in middleware.ts
- Payments: Stripe Checkout → webhook at /api/webhooks/stripe
- Images: S3 + CloudFront, optimization via next/image
- Emails: React Email + Resend

## Workflow
- Branches: feature/<name>, fix/<name>
- Commits: Conventional Commits (feat:, fix:, chore:)
- Before PR: `pnpm lint && pnpm test && pnpm build`
- Deploy: Vercel, preview on every PR

## Known Issues
<!-- Note to self: don't show to agent automatically -->
<!-- TODO: migrate from next-auth v4 to v5 after stabilization -->
- Stripe webhook sometimes duplicates events — there's an idempotency check in lib/stripe.ts
- On Safari the cart animation lags — known CSS transform bug
```

**Result:** every new session — the agent already knows:
- What the stack is and why exactly this one
- Which patterns to use (Zod, not as; Zustand, not Redux)
- How authentication and payments are structured
- Which bugs are known (so it doesn't try to "fix" them again)

**Who needs this:** absolutely everyone who uses Claude Code for more than one session:
- **Any project** — from pet project to enterprise: CLAUDE.md saves 2-5 minutes per session
- **Teams** — unified standards for everyone who runs the agent
- **Complex architectures** — the agent won't "reinvent" what's already been decided
- **Onboarding** — a new developer reads CLAUDE.md and understands key decisions

This is **the most important file** in Claude Code setup. Everything else is optimization on top.

---

## 19. 🔴 Harness Engineering — Theory and Resources

**Main thesis:** weak agent results are almost always a harness (environment) problem, not a model problem. This is the red thread throughout the entire guide — `Never use as`, Skills structure with Non-Negotiable criteria, worktrees, hooks — all of this is harness engineering in different forms.

**Definition:** harness engineering — the practice of shaping the environment around the agent so that it works predictably. Overlaps with context engineering, evaluation, observability, orchestration, and safe autonomy.

📌 Real-world example

**Situation:** You're complaining: "The agent generates bad code, doesn't follow standards, forgets tests." Intuition says — switch the model or the prompt. Harness engineering says: the problem isn't the model.

**Before (bad harness):**
```
You: "Write authentication"
Agent: *writes something... different every time*
You: "No, not like that! Use JWT, not sessions!"
Agent: *rewrites... forgets refresh token*
You: "Add refresh token!"
# 5 iterations... mediocre result
```

**After (good harness):**
```markdown
# CLAUDE.md
## Auth
- JWT with access + refresh tokens
- Access token: 15 min, refresh: 7 days
- Storage: httpOnly cookies, not localStorage
- Middleware: app/middleware.ts, checks access token

# .claude/skills/auth-module.md
## Non-Negotiable Acceptance Criteria
- [ ] Access token expires in 15 min
- [ ] Refresh token expires in 7 days
- [ ] Tokens stored in httpOnly secure cookies
- [ ] /auth/refresh endpoint exists and tested
- [ ] Middleware rejects expired tokens with 401
```

```
You: "Write authentication"
Agent: *reads CLAUDE.md + skill → knows exactly WHAT and HOW*
# 1 iteration, predictable result
```

**Analogy:** harness engineering = road markings. Without markings — an experienced driver will get there, but on a different route every time. With markings — even a beginner drives predictably.

**Who needs this:** absolutely everyone — this is a foundation, not a feature:
- This isn't a separate tool, it's a **mindset**: "if the agent is doing the wrong thing — the problem is in my environment"
- The articles below are required reading for anyone who wants to get the most out of AI agents
- Applicable to any stack, any project size

### Primary Sources from Anthropic

| Article | Summary |
|---|---|
| [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) | Initializer agents, `init.sh`, self-verification, handoff artifacts across multiple context windows |
| [Claude Code: Best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices) | Repo structure, checkpoints, validation, delegation |
| [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents) | Tool interfaces that the model calls correctly and safely |
| [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Context window as a working memory budget, not a junk drawer |
| [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | Workflows, agents, tools — when structure beats a raw prompt |

### Practical Guides

| Article | Summary |
|---|---|
| [Writing a good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) | Durable repo-local instructions that the agent follows repeatedly |
| [Skill Issue: Harness Engineering](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents) | A concrete argument: weak results = harness problem |
| [12 Factor Agents](https://www.humanlayer.dev/blog/12-factor-agents) | Principles for production agents: explicit prompts, state ownership, pause-resume |
| [Context Engineering for AI Agents (Manus)](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) | KV-cache locality, tool masking, filesystem memory |

### Awesome-list

Full curated list: https://github.com/walkinglabs/awesome-harness-engineering

---

## 20. Official Resources

| Resource | Link |
|---|---|
| CHANGELOG (v2.1.78) | https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md |
| Hooks | https://code.claude.com/docs/en/hooks |
| Sub-agents | https://code.claude.com/docs/en/sub-agents |
| CLI Reference | https://code.claude.com/docs/en/cli-reference |
| Chrome Extension | https://code.claude.com/docs/en/chrome |
| Remote Control | https://code.claude.com/docs/en/remote-control |
| Scheduled Tasks | https://code.claude.com/docs/en/scheduled-tasks |
| gstack | https://github.com/garrytan/gstack |
| Biome no-type-assertion | https://github.com/albertodeago/biome-plugin-no-type-assertion |
| Official Skills repo | https://github.com/anthropics/skills |
| Awesome Harness Engineering | https://github.com/walkinglabs/awesome-harness-engineering |
