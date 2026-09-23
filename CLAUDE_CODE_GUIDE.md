# Claude Code: A Practical Guide (with Multi-Agent Workflows)

This guide covers what Claude Code is, how to use the web interface you're in right now,
how to set up a project so Claude works well in it, and how to run **many agents** at once.

Official docs: https://code.claude.com/docs

---

## 1. What Claude Code is

Claude Code is an AI coding agent. Unlike a chat bot, it can **act**: it reads and edits files,
runs shell commands, runs tests, uses git, opens pull requests, and calls external tools
(GitHub, Google Drive, Slack, databases…) through connectors called **MCP servers**.

You can run it in several places. It's the same agent everywhere:

| Where | How you start it | Where the code lives |
|---|---|---|
| **Web / mobile / desktop app** (claude.ai/code) | Pick a repo and type a task | A fresh cloud container that clones your GitHub repo |
| **Terminal (CLI)** | `npm install -g @anthropic-ai/claude-code`, then `claude` in a project folder | Your own machine |
| **IDE** | VS Code / JetBrains extensions | Your own machine |
| **GitHub Actions / Slack** | Mention `@claude` in an issue, PR, or Slack thread | CI runner or cloud |

---

## 2. The web interface (what you're using now)

### How a session works
1. You pick a **repository** and an **environment**, then type a task.
2. Claude gets an **isolated cloud container** with your repo freshly cloned.
3. It works on its own **branch** (this session's is `claude/code-multi-agent-guide-e0n9r3`).
4. When it's done it **commits and pushes** to that branch. You can then ask it to open a **pull request**.
5. The container is thrown away after inactivity. **Anything not pushed is lost.**

### Things you can do from the interface
- **Run several sessions at once.** Every session is independent: its own container, its own
  branch. This is the simplest way to run "many agents": open 3 sessions, give each one a task.
- **Watch it work live.** You see every file it reads, command it runs, and edit it makes.
- **Interrupt and redirect** at any time by typing a new message ("stop, do X instead").
- **Review the diff** and create a PR from the session.
- **Ask it to babysit a PR.** Say "watch this PR and fix CI failures / review comments". It
  subscribes to GitHub events and wakes up by itself when CI fails or someone comments.
- **Schedule work ("Routines").** Say "every weekday at 9am, check open issues and triage them",
  or "remind me in 2 hours to re-check the deploy". It creates a scheduled trigger.
- **Use connectors.** If you've connected Gmail, Google Drive, Calendar, etc. in claude.ai
  settings, Claude can use them inside the session.
- **Get pages back (Artifacts).** Ask for a dashboard, report, or HTML page and Claude publishes
  it as a private link you can open and share.
- **Move to your terminal.** A web session can be continued locally in the CLI (see the
  "Claude Code on the web" docs page for the current command/button).

### Environments
An **environment** controls what the container can do:
- **Network access policy** (none / limited allowlist / full internet)
- **Environment variables and secrets** (API keys, etc.)
- **Setup script** (e.g. `npm install`, install a database) that runs when the container starts

Configure them in the Claude Code web settings. Docs:
https://code.claude.com/docs/en/claude-code-on-the-web

---

## 3. Talking to Claude effectively

- **Be specific about the outcome.** "Add a `/health` endpoint that returns `{status:"ok"}` and a
  test for it" beats "add health check".
- **Say how to verify.** "Run `npm test` and make sure it passes." Claude works best when it can
  check its own work.
- **Point at files.** "Look at `src/api/users.ts` first." (In the CLI you can type `@` to
  autocomplete a file path.)
- **Ask for a plan first** on big tasks: "Plan this before writing code." (CLI: **Shift+Tab**
  cycles into Plan Mode.)
- **Iterate.** Treat it like a teammate: review, then say what to change.
- **Commit/PR only when you want.** Claude won't open a PR unless you ask.

---

## 4. Setting up a project so Claude works well

Everything lives in the repo, so it works for everyone and in every session (web or CLI).

```
your-repo/
├── CLAUDE.md                     # Project memory: read at the start of every session
└── .claude/
    ├── settings.json             # Permissions, hooks, env vars (shared with team)
    ├── settings.local.json       # Your personal overrides (git-ignore this)
    ├── agents/                   # Custom subagents (see §5)
    │   └── reviewer.md
    └── skills/                   # Reusable workflows / slash commands
        └── deploy/SKILL.md
```

### CLAUDE.md: project memory
Plain markdown that Claude reads automatically. Put in it:
- How to install, build, test, lint (`npm test`, `pytest -q`, …)
- Code style rules and conventions
- Architecture notes ("API in `server/`, UI in `web/`")
- Things to never do ("don't edit generated files in `gen/`")

Run `/init` to have Claude write a first draft by scanning the repo.
You can also have a personal one at `~/.claude/CLAUDE.md` (CLI only) that applies to all projects.

### Skills: reusable workflows
A skill is a folder with a `SKILL.md` that teaches Claude a repeatable procedure. Claude loads it
when relevant, or you invoke it by name as a slash command (`/deploy`).

```markdown
---
name: deploy
description: Deploy the app to staging. Use when the user asks to deploy or ship.
---
1. Run `npm run build` and make sure it succeeds.
2. Run `npm test`.
3. Run `./scripts/deploy.sh staging` and report the URL.
```

### Hooks: automatic actions
Hooks are shell commands the *harness* runs on events (before/after a tool, when Claude stops,
at session start). Example: auto-format after every file edit. They live in
`.claude/settings.json`. Ask Claude "add a hook that runs prettier after every edit" and it will
write it. A **SessionStart** hook is especially useful on the web to install dependencies.

### MCP servers: connect external tools
MCP servers give Claude new tools (GitHub, Sentry, Postgres, Figma, …).
- CLI: `claude mcp add <name> -- <command>`, and `/mcp` to see status.
- Web/app: connect them as **connectors** in claude.ai settings.

### Permissions
Claude asks before risky actions. You can pre-allow safe ones in `settings.json`:
```json
{ "permissions": { "allow": ["Bash(npm test:*)", "Bash(git status)"] } }
```
Permission modes (CLI: Shift+Tab to cycle): **default** (asks), **acceptEdits** (auto-accepts
file edits), **plan** (read-only, proposes a plan), **auto** (approves automatically within
safety rules).

---

## 5. Many agents: the three levels

### Level 1: Subagents inside one session
The main Claude can spawn **subagents**, which are helper instances with their **own fresh
context window**. They do a focused job and report back a summary. Why use them:
- **Parallelism:** several run at the same time.
- **Clean context:** a big search doesn't clutter the main conversation.
- **Specialization:** each has its own instructions, tools, and even model.

**Built-in subagents**

| Agent | What it's for |
|---|---|
| `general-purpose` | Any multi-step task, research, code changes |
| `Explore` | Fast read-only searching of the codebase |
| `Plan` | Designing an implementation plan (read-only) |

**How to trigger them**: just ask:
> "Use 3 subagents in parallel: one to find every place we call the payments API, one to
> list all failing tests, one to check which dependencies are outdated. Then summarize."

> "Spawn a subagent to review my diff for security issues while you write the tests."

Claude will not spawn agents unprompted for small tasks (each one starts cold and costs
tokens), so **explicitly ask** when you want them.

**Custom subagents**: create `.claude/agents/<name>.md`:

```markdown
---
name: code-reviewer
description: Reviews code changes for bugs, security issues, and style. Use after writing code.
tools: Read, Grep, Glob, Bash
model: sonnet
---
You are a senior code reviewer. For the current diff:
1. Look for correctness bugs and edge cases.
2. Look for security issues (injection, secrets, auth).
3. Report findings ranked by severity with file:line references.
Do not edit files.
```

Frontmatter fields:
- `name`: how you refer to it
- `description`: tells Claude **when** to use it (write this carefully; it drives auto-selection)
- `tools`: which tools it may use (omit = all). Restricting tools makes agents safer.
- `model`: e.g. `sonnet`, `opus`, `haiku` (cheap and fast for simple jobs), or omit to inherit

Where they live:
- `.claude/agents/`: project agents, committed and shared with your team
- `~/.claude/agents/`: your personal agents, available in every project (CLI)

In the CLI, `/agents` opens an interactive manager to create/edit them.
Then: *"Use the code-reviewer agent on my changes."*

**Useful subagent patterns**
- **Fan-out research:** N agents each explore one area → main agent merges results.
- **Writer + reviewer:** one agent implements, a separate one (fresh eyes) reviews.
- **Tester:** an agent whose only job is to write and run tests for the change.
- **Isolated worktree:** an agent can work in its own git worktree so parallel edits don't
  collide.
- **Background agents:** long-running agents keep going while you keep chatting; you can send
  follow-up messages to a running agent instead of starting a new one.

### Level 2: Many parallel sessions
For truly independent tasks, run **separate sessions**, each a full Claude with its own
container and branch:
- **Web:** start several sessions from claude.ai/code (e.g. one fixing a bug, one writing docs,
  one upgrading dependencies). Each produces its own branch/PR.
- **Orchestration from a session:** a session can itself **create child sessions** and hand
  them tasks. Ask: *"Create 3 new sessions: one per microservice, each should upgrade to
  Node 22 and open a PR."*
- **CLI:** open several terminals. Use **git worktrees** so they don't step on each other:
  ```bash
  git worktree add ../myrepo-feature-a -b feature-a
  git worktree add ../myrepo-feature-b -b feature-b
  cd ../myrepo-feature-a && claude     # terminal 1
  cd ../myrepo-feature-b && claude     # terminal 2
  ```

### Level 3: Scripted / programmatic agents
- **Headless mode:** `claude -p "fix the lint errors in src/" --output-format json`
  runs one task non-interactively, which is great for scripts and CI. Loop it over many
  files/repos to run many agents from a shell script.
- **GitHub Actions:** install the Claude GitHub app and mention `@claude` on issues/PRs.
- **Claude Agent SDK** (Python/TypeScript): build your own agents on the same engine as
  Claude Code, with your own tools, subagents, and permissions.
  https://code.claude.com/docs/en/sdk
- **Routines:** scheduled agents (cron-style) that run a prompt on a schedule, in a fresh
  session each time or continuing an existing one.

### Choosing the right level

| You want… | Use |
|---|---|
| Speed up one task by splitting research/review | Subagents (Level 1) |
| Several unrelated tasks done at once, each with its own PR | Parallel sessions (Level 2) |
| Automation in CI, scripts, or your own product | Headless / SDK (Level 3) |
| Something to happen every day/hour | Routines |

---

## 6. CLI cheat sheet

| Command / key | What it does |
|---|---|
| `claude` | Start interactive session in current folder |
| `claude -p "task"` | Run one task headlessly and print the result |
| `claude -c` / `claude -r` | Continue last conversation / pick one to resume |
| `/init` | Generate a CLAUDE.md |
| `/agents` | Manage subagents |
| `/mcp` | Manage MCP servers |
| `/model` | Switch model |
| `/config`, `/permissions`, `/hooks` | Settings |
| `/clear` | Start a fresh conversation |
| `/compact` | Summarize the conversation to free up context |
| `/help` | List all commands (including your skills) |
| **Shift+Tab** | Cycle permission modes (incl. Plan Mode) |
| **Esc** | Interrupt Claude |
| **Esc Esc** | Rewind to an earlier message |
| `@path` | Reference a file |
| `!cmd` | Run a shell command directly |

---

## 7. A hands-on exercise in this repo

Try these prompts, in order, in a session on this repo:

1. *"Create a small Python CLI todo app with tests, and a CLAUDE.md explaining how to run it."*
2. *"Create a custom subagent in `.claude/agents/test-writer.md` that only writes tests."*
3. *"Use 2 subagents in parallel: the test-writer to add edge-case tests, and a reviewer
   to review the code. Then apply the fixes."*
4. *"Commit, push, and open a PR."*
5. *"Watch the PR and fix anything CI or reviewers flag."*

---

## 8. Tips & gotchas

- **Context is finite.** Long sessions get summarized automatically. Use subagents for big
  searches, and `/clear` between unrelated tasks.
- **More agents ≠ always better.** Each subagent starts from zero and re-reads things.
  Use them for parallelizable or context-heavy work, not tiny edits.
- **Give agents a way to verify** (tests, linters, a build). That's the biggest quality lever.
- **Web sessions are ephemeral:** make sure work is pushed.
- **Review before merging.** Claude is strong but not infallible.
