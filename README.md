# AToDo — Agent-First Todo Application

An agent-first, local-first todo application built with Rust and Tauri v2. Designed to be natively consumed by AI agents via MCP (Model Context Protocol) while providing a beautiful desktop UI for humans.

## Features

- 🤖 **Agent-First**: Full MCP server integration — AI agents can create, read, update, and manage tasks and categories natively
- 📝 **Markdown**: Task descriptions, notes, and agent instructions render as Markdown (headings, lists, tables, code, links)
- 🧠 **Agent Instructions**: Give any task its own directives for an AI agent; the agent reads and applies them *before* working on the task (see [Agent Instructions](#agent-instructions))
- 📥 **Assign to Agent**: Flag tasks for an agent to act on; agents find them via MCP and report back in the task notes (see [Assigning Tasks to an Agent](#assigning-tasks-to-an-agent))
- 🔎 **Review queue**: When an agent completes a task assigned to it, the task moves to a **Waiting for review** status (a dedicated **Review** queue in the sidebar) instead of being marked done — approve it, or request changes to send it back to the agent with feedback
- 🔒 **Secure**: SQLCipher encrypted local database
- 📴 **Offline-First**: Works fully offline, syncs when connected
- 🔄 **Pluggable Sync**: OneDrive and GitHub sync (coming soon)
- 🔁 **Live Updates**: The UI auto-refreshes, so changes made by agents appear automatically
- 🗂️ **Categories**: Create, rename, and delete categories; view uncategorized tasks
- 📅 **My Day & My Week**: Focused views for today's tasks and the working week (work days are configurable); drag tasks between days in My Week to reschedule them
- ⚙️ **Settings**: Choose your work-week days and how long to keep completed tasks; settings live in the DB and are readable by the agent
- 🖥️ **Desktop App**: Beautiful Tauri v2 desktop application
- 🔕 **Minimize to Tray**: Closing the window hides to the system tray; restore from the tray icon, right-click → Quit to exit
- ⚡ **Fast**: Built with Rust for performance and safety

## Architecture

```
AToDo/
├── crates/
│   ├── atodo-core/     # Core library (models, storage, business logic)
│   └── atodo-mcp/      # MCP server for agent integration
├── src-tauri/          # Tauri v2 desktop app (Rust backend + IPC commands)
├── ui/                 # React + TypeScript + Tailwind CSS frontend
└── docs/               # Documentation
```

## Getting Started

### Prerequisites

- [Rust](https://rustup.rs/) (1.75+)
- [Node.js](https://nodejs.org/) (18+) — for the Tauri frontend

### Install Frontend Dependencies

```bash
cd ui && npm install
```

### Build Installer

Creates a production `.exe` with the frontend embedded, plus MSI/NSIS installers:

```bash
npx @tauri-apps/cli build
```

Output:
- `target/release/atodo.exe` — standalone binary (GUI + MCP)
- `target/release/bundle/nsis/AToDo_*-setup.exe` — NSIS installer
- `target/release/bundle/msi/AToDo_*.msi` — MSI installer

### Usage

```bash
atodo              # Launch the desktop app
atodo mcp          # Run the MCP server for AI agents
```

### Development

```bash
cargo test                    # Run tests
npx @tauri-apps/cli dev       # Launch app with hot-reload (run from repo root)
```

### Run MCP Server

The MCP server is built into the same binary:

```bash
# From the unified binary
atodo mcp

# Or using the standalone crate
cargo run -p atodo-mcp
```

## MCP Integration

AToDo exposes itself as an MCP server that AI agents can connect to.

### GitHub Copilot (VS Code)

1. Open VS Code Settings (`Ctrl+,`)
2. Search for `github.copilot.chat.mcp`
3. Click **Edit in settings.json**
4. Add the AToDo server to the `mcp` section:

```json
{
  "github.copilot.chat.mcp.servers": {
    "atodo": {
      "command": "atodo",
      "args": ["mcp"]
    }
  }
}
```

> **Note**: Make sure `atodo.exe` is in your `PATH`, or use the full path:
> ```json
> "command": "C:\\Program Files\\AToDo\\atodo.exe"
> ```

5. Restart VS Code or reload the window
6. In Copilot Chat, you can now ask things like:
   - *"Create a task to refactor the auth module with high priority"*
   - *"List all my pending tasks"*
   - *"Mark the deploy task as complete"*

### GitHub Copilot CLI

Add to your `~/.config/github-copilot/mcp.json`:

```json
{
  "mcpServers": {
    "atodo": {
      "command": "atodo",
      "args": ["mcp"]
    }
  }
}
```

### Claude Desktop

Add to your Claude Desktop config (`%APPDATA%\Claude\claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "atodo": {
      "command": "atodo",
      "args": ["mcp"]
    }
  }
}
```

### Available MCP Tools

| Tool | Description |
|------|-------------|
| `create_task` | Create a new task with title, priority, due date, project |
| `list_tasks` | List tasks, optionally filtered by project |
| `list_tasks_assigned_to_agent` | List tasks assigned to an agent that still need action (not completed), optionally filtered by project |
| `get_task` | Get details of a specific task |
| `update_task` | Update task fields (title, body, priority, etc.) |
| `complete_task` | Mark a task as completed. If the task is assigned to an agent, it is submitted for user review (status **Waiting for review**) instead of being marked done |
| `delete_task` | Delete a task |
| `search_tasks` | Full-text search across tasks |
| `create_project` | Create a new project/category |
| `list_projects` | List all projects/categories |
| `update_project` | Rename or recolor a project/category |
| `delete_project` | Delete a project/category (optionally delete its tasks too) |

### Available MCP Resources

| Resource | Description |
|----------|-------------|
| `atodo://tasks` | All tasks as JSON |
| `atodo://projects` | All projects as JSON |
| `atodo://settings` | User settings (work days + completed-task retention) as JSON |

### Available MCP Prompts

| Prompt | Description |
|--------|-------------|
| `act_on_task` | Given a `task_id`, returns a structured message that puts the task's **agent instructions first**, then the task details, then a finish step. See [Agent Instructions](#agent-instructions). |

## Agent Instructions

Every task has an optional **agent instructions** field — directives aimed at an
AI agent rather than at you. Set it from the task detail panel in the desktop app
(the "Agent Instructions" box), or via the `agent_instructions` argument on the
`create_task` / `update_task` MCP tools.

When an agent acts on a task through the `act_on_task` MCP prompt, AToDo builds an
ordered message so the instructions are always applied first:

1. **Step 1 — Agent instructions:** read and follow these before anything else.
   If the task has none, the agent is told to use sensible defaults.
2. **Step 2 — The task:** title, description, notes, priority, due date, and task ID.
3. **Step 3 — When finished:** write a short summary into the task's notes via
   `update_task`, then call `complete_task`. For agent-assigned tasks this
   submits the task for the user's review instead of marking it done.

This lets you steer *how* a task should be done (constraints, style, acceptance
criteria, tools to use) separately from *what* the task is.

## Assigning Tasks to an Agent

Each task has an **Assign to Agent** toggle (in the task detail panel) that marks
it as work for an AI agent. Assigned tasks show a small agent badge in task lists.

Agents discover assigned work with the `list_tasks_assigned_to_agent` MCP tool,
which returns assigned, not-yet-completed tasks (optionally filtered by project):

- *"Go check all the tasks under GDAP and act on them"* →
  `list_tasks_assigned_to_agent(project_id = <GDAP project id>)`
- *"Are there any tasks you need to act on?"* →
  `list_tasks_assigned_to_agent()`

The agent then runs the `act_on_task` prompt for each returned task id, which
applies the agent instructions, does the work, **writes a summary back into the
task's notes**, and calls `complete_task`. Because the task is assigned to an
agent, completing it does **not** mark it done — it moves the task to a
**Waiting for review** status that appears in the **Review** queue in the
sidebar. The user then either:

- **Approves** it (marks it done), or
- **Requests changes** — they type feedback, which is appended to the task's
  agent instructions and the task is re-assigned to the agent (status reset).
  The agent picks it up again via `list_tasks_assigned_to_agent`, sees the
  revised instructions plus its previous attempt in the notes, and resubmits
  for review. This loop continues until the user approves.

This entire review gate is enforced **server-side in the `complete_task` MCP
tool** — no special agent instructions are needed. Submitting for review also
clears the **Assign to Agent** flag, so the task leaves the agent pickup queue
until the user requests changes.

## License

MIT
