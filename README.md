# KubeOpenCode Skills

Agent skills for managing [KubeOpenCode](https://github.com/kubeopencode/kubeopencode) AI tasks and agents on Kubernetes.

These skills follow the [Agent Skills specification](https://agentskills.io/specification) so they can be used by any skills-compatible agent, including Claude Code, OpenCode, and Codex CLI.

## Installation

### Marketplace

First, add the marketplace source:

```
/plugin marketplace add kubeopencode/skills
```

Then, install the plugin:

```
/plugin install kubeopencode@kubeopencode-skills
```

### npx skills

```
npx skills add git@github.com:kubeopencode/skills.git
```

### Manually

#### Claude Code

Clone this repo into your project's `.claude` directory (or your home directory's `~/.claude`):

```sh
git clone https://github.com/kubeopencode/skills.git /path/to/your/project/.claude
```

Or copy just the `skills/` directory if you already have a `.claude` folder:

```sh
git clone https://github.com/kubeopencode/skills.git /tmp/kubeopencode-skills
cp -r /tmp/kubeopencode-skills/kubeopencode/skills/ /path/to/your/project/.claude/skills/
```

See the [Claude Skills documentation](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/skills) for more details.

#### OpenCode

Clone the entire repo into the OpenCode skills directory (`~/.opencode/skills/`):

```sh
git clone https://github.com/kubeopencode/skills.git ~/.opencode/skills/kubeopencode-skills
```

Do not copy only the inner `skills/` folder — clone the full repo so the directory structure is `~/.opencode/skills/kubeopencode-skills/kubeopencode/skills/<skill-name>/SKILL.md`.

OpenCode auto-discovers all `SKILL.md` files under `~/.opencode/skills/`. No changes to `opencode.json` or any config file are needed. Skills become available after restarting OpenCode.

#### Codex CLI

Copy the `skills/` directory into your Codex skills path (typically `~/.codex/skills`):

```sh
git clone https://github.com/kubeopencode/skills.git /tmp/kubeopencode-skills
cp -r /tmp/kubeopencode-skills/kubeopencode/skills/ ~/.codex/skills/
```

See the [Agent Skills specification](https://agentskills.io/specification) for the standard skill format.

## Skills

| Skill | Description |
|-------|-------------|
| [koc](kubeopencode/skills/koc) | Manage KubeOpenCode AI tasks and agents on Kubernetes — create tasks, list agents, check status, view logs, stop tasks, monitor progress, and schedule recurring tasks |

## Configuration

The `koc` skill requires the following environment variables:

| Variable | Required | Description |
|----------|----------|-------------|
| `KUBEOPENCODE_KUBECONFIG` | **Yes** | Path to kubeconfig for the KubeOpenCode cluster |
| `KUBEOPENCODE_DEFAULT_TASK_NAMESPACE` | No | Default namespace for Tasks |
| `KUBEOPENCODE_DEFAULT_AGENT_NAMESPACE` | No | Default namespace for Agents |

Set these in your shell profile or project-level environment configuration.

## What can this skill do?

Once installed, your AI coding agent can:

- **List agents** — see what AI agents are available on the cluster
- **Create tasks** — run AI-powered tasks on Kubernetes with automatic agent matching
- **Monitor tasks** — track task progress with automatic polling
- **View logs** — fetch task output from pod logs
- **Stop tasks** — gracefully stop running tasks
- **Schedule tasks** — set up recurring tasks with natural language schedules

Example prompts:

- "What agents are available on the cluster?"
- "Create a task to fix the login bug using the go-agent"
- "What's the status of my task?"
- "Show me the logs for task fix-bug-abc1"
- "Stop the running task"
- "Schedule a daily task to check for dependency updates using the go-agent"

## License

Apache License 2.0 - see [LICENSE](LICENSE) for details.
