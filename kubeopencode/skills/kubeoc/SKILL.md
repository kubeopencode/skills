---
name: kubeoc
description: >
  Manage KubeOpenCode AI tasks and agents on Kubernetes. This skill handles
  anything related to running AI-powered tasks on a Kubernetes cluster via
  KubeOpenCode — creating tasks, listing agents, checking task status, viewing
  logs, stopping tasks, monitoring task progress, scheduling recurring tasks
  with CronTasks, and attaching to server-mode agents interactively.
  Trigger this skill when the user wants to: create or run an AI task on the
  cluster, see what agents or agent templates are available, check on a running
  task, view task output or logs, stop a task, use a specific agent for a job,
  attach to a server-mode agent, schedule a recurring task with CronTask, list
  schedules, set up a daily/weekly task, suspend/resume a schedule, or anything
  involving KubeOpenCode resources (Task, Agent, AgentTemplate, CronTask).
  Also trigger when users say things like "run this on the cluster",
  "use agent X to do Y", "what's my task doing", "show me the logs",
  "is it done yet", "attach to agent", "connect to agent", "schedule a daily
  task", "run this every hour", or refer to kubeopencode, kubeoc, tk, ag, agt,
  or crontask resources.
  Supports namespace flag: --namespace/--ns to override default namespace.
---

# KubeOpenCode Skill

Interact with KubeOpenCode Kubernetes resources (Task, Agent, AgentTemplate, CronTask) using `kubectl` and the `kubeoc` CLI.

## Prerequisites

- `kubectl` configured with access to a cluster running KubeOpenCode
- `kubeoc` CLI (optional, for interactive agent sessions): `go install github.com/kubeopencode/kubeopencode/cmd/kubeoc@latest`

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `KUBEOPENCODE_KUBECONFIG` | **Yes** | Path to kubeconfig for the KubeOpenCode cluster |
| `KUBEOPENCODE_DEFAULT_NAMESPACE` | No | Default namespace for all operations (if unset, use `--all-namespaces` for list, ask on create) |

**CRITICAL**: Every `kubectl` command MUST include `--kubeconfig "$KUBEOPENCODE_KUBECONFIG"`. If this env var is not set, ask the user to set it before proceeding — do NOT fall back to the default kubeconfig. Check this env var in each Bash call since shell state does not persist between calls.

The `kubeoc` CLI resolves kubeconfig in the same priority order: `KUBEOPENCODE_KUBECONFIG` > `KUBECONFIG` > `~/.kube/config`.

### Namespace Flag

Users can pass a namespace flag as a skill argument to override the default namespace for the current invocation:

| Flag | Short Form | Description |
|------|------------|-------------|
| `--namespace <ns>` | `--ns <ns>` | Override `KUBEOPENCODE_DEFAULT_NAMESPACE` for this invocation |

**Examples:**
```
/kubeoc --ns production list tasks
/kubeoc --namespace monitoring list agents
```

## API Quick Reference

- **API Group**: `kubeopencode.io/v1alpha1`
- **Resources & Short Names**: Task (`tk`), Agent (`ag`), AgentTemplate (`agt`), CronTask, KubeOpenCodeConfig (`ktc`)
- **Task Phases**: `Pending` -> `Queued` -> `Running` -> `Completed` | `Failed`
- **Stop Annotation**: `kubeopencode.io/stop=true`
- **Pod Naming**: `<task-name>-pod`
- **Pod Location**: Pod runs in the same namespace as the Task
- **Scheduled Tasks**: Native CronTask CRD (`kubeopencode.io/v1alpha1`)
- **Manual Trigger Annotation**: `kubeopencode.io/trigger=true`
- **CronTask Label on Tasks**: `kubeopencode.io/crontask=<crontask-name>`

For detailed field reference, read: `references/api-reference.md`

## Operations

**Namespace flag behavior for list/get operations**: When `--namespace`/`--ns` is provided, use it for all operations. If no flag or env var is set, fall back to `--all-namespaces` for list operations.

### 1. List Agents

**Using kubeoc CLI** (preferred — shows mode and status):
```bash
kubeoc get agents
kubeoc get agents -n <namespace>
```

Shows: NAMESPACE, NAME, PROFILE, MODE (Server/Pod), STATUS (Ready/Not Ready).

**Using kubectl** (more detail with `-o wide`):
```bash
# All namespaces
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get ag --all-namespaces

# Specific namespace
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get ag -n <namespace>

# Detailed view (shows profile, executorImage, maxConcurrentTasks)
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get ag -n <namespace> -o wide
```

Show: name, namespace, serviceAccount, age. With `-o wide`: profile, executorImage, maxConcurrentTasks.

### 2. Get Agent Details

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get ag <name> -n <namespace> -o yaml
```

Summarize: profile, templateRef, executorImage, workspaceDir, maxConcurrentTasks, quota, serverConfig presence, caBundle/proxy presence, credentials (names only — never show secret values).

### 3. Attach to Server-Mode Agent

Opens an interactive OpenCode terminal session connected to a server-mode agent. Requires `kubeoc` and `opencode` binaries.

```bash
kubeoc agent attach <agent-name> -n <namespace>
```

**Options:**

| Flag | Default | Description |
|------|---------|-------------|
| `-n, --namespace` | `default` | Agent namespace |
| `--local-port` | Agent's server port | Local port for the proxy connection |
| `--use-port-forward` | `false` | Use kubectl port-forward instead of service proxy (legacy mode) |
| `--server-namespace` | `kubeopencode-system` | Namespace where kubeopencode server is deployed |
| `--server-service` | `kubeopencode-server` | Name of kubeopencode server Service |
| `--server-port` | `2746` | Port of kubeopencode server Service |

**Examples:**
```bash
kubeoc agent attach server-agent -n test
kubeoc agent attach my-agent -n production --local-port 5000
kubeoc agent attach my-agent -n test --use-port-forward
```

**Prerequisites:**
- Agent must be in Server mode (`serverConfig` present in spec)
- Agent server must be Ready (`status.serverStatus.ready: true`)
- `opencode` binary must be in PATH

**Connection modes:**
1. **Service Proxy (default)**: Connects via kube-apiserver's built-in service proxy. No port-forward needed.
2. **Port-Forward (legacy)**: Uses `kubectl port-forward`. Enable with `--use-port-forward`.

If the agent is not in Server mode or not ready, the command will fail with an actionable error message.

### 4. List AgentTemplates

```bash
# All namespaces
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get agt --all-namespaces

# Specific namespace
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get agt -n <namespace>

# Detailed view
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get agt -n <namespace> -o wide
```

Show: name, namespace, serviceAccount, age. With `-o wide`: executorImage, workspaceDir.

### 5. Get AgentTemplate Details

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get agt <name> -n <namespace> -o yaml
```

Summarize: executorImage, workspaceDir, serviceAccountName, serverConfig presence, credentials (names only — never show secret values). Note which Agents reference this template if possible.

### 6. List Tasks

```bash
# All namespaces
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk --all-namespaces

# Specific namespace
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk -n <namespace>
```

Show: name, phase, agent, pod, age.

### 7. Create Task

**Agent Resolution Order:**
1. User-specified agent name
2. Auto-match by profile: run `kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get ag --all-namespaces -o wide` to get all agents
   with their `profile` field. Compare each agent's profile against the user's task and:
   - **Single best match**: Propose it to the user for confirmation, e.g.:
     > Based on your task, I recommend **go-agent** ("Go development with GitHub access"). Use this agent? (y/n)
   - **Multiple matches**: Present a numbered list and let the user pick, e.g.:
     > Multiple agents match your task:
     > 1. **go-agent** — "Go development with GitHub access"
     > 2. **fullstack-agent** — "Full-stack dev with Node.js and Go"
     > Which agent would you like to use? (1/2)
   - **No match**: Show all available agents with profiles and ask the user to choose.

**Namespace Resolution Order:**
1. `--namespace` / `--ns` flag
2. User-specified namespace in natural language
3. `KUBEOPENCODE_DEFAULT_NAMESPACE` env var
4. Ask user

**Task Name:** User-specified or auto-generate as `task-<short-slug>-<4-random-hex>`.

**Generate YAML and show to user for confirmation before applying.**

#### Basic Task (with agentRef)

```yaml
apiVersion: kubeopencode.io/v1alpha1
kind: Task
metadata:
  name: <task-name>
  namespace: <namespace>
spec:
  agentRef:
    name: <agent-name>
  description: |
    <user's task description in natural language>
```

#### Task with Inline Contexts

```yaml
apiVersion: kubeopencode.io/v1alpha1
kind: Task
metadata:
  name: <task-name>
  namespace: <namespace>
spec:
  agentRef:
    name: <agent-name>
  description: |
    <task description>
  contexts:
    - name: extra-context
      type: Text
      text: |
        Additional instructions here
    - name: source-code
      type: Git
      git:
        repository: https://github.com/org/repo.git
        ref: main
      mountPath: source
```

**Apply the Task:**
```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" apply -f /tmp/kubeopencode-task-<name>.yaml
```

After successful creation, **automatically start monitoring** (see Operation 12).

### 8. Get Task Status

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk <name> -n <namespace> -o yaml
```

Extract and present:
- **Phase**: `.status.phase`
- **Pod**: `.status.podName`
- **Agent**: `.status.agentRef.name`
- **Start Time**: `.status.startTime`
- **Completion Time**: `.status.completionTime`
- **Conditions**: `.status.conditions` (type, status, reason, message)

Present a concise summary, e.g.:
> Task `my-task` is **Running** (started 5m ago). Pod: `my-task-pod`.

### 9. Get Task Logs

First resolve the pod name from the task status:
```bash
# Get pod name and fetch logs — pod is in the same namespace as the task
POD_NAME=$(kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk <task-name> -n <namespace> -o jsonpath='{.status.podName}')
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" logs "$POD_NAME" -n <namespace> -c agent
```

If the task is still running, add `--tail=100` to show recent logs.
If the pod is not found, the task may have been cleaned up — inform the user.

### 10. Stop Task

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" annotate task <name> -n <namespace> kubeopencode.io/stop=true
```

**Always warn the user first**: Stopping a task deletes the pod, and **logs will be lost** unless an external log aggregation system is in place. Ask for confirmation before executing.

### 11. Delete Task

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" delete task <name> -n <namespace>
```

Deleting a Task cascades to its Pod and ConfigMap via OwnerReference. Ask for confirmation — this is irreversible.

Only use delete when the user explicitly wants to remove the Task resource. For stopping a running task, prefer the stop annotation (Operation 10).

### 12. Monitor Task

After creating a task, poll periodically until completion. Since shell state doesn't persist between calls, run each check as a standalone command.

**Polling strategy:**

| Complexity | Poll Interval | Timeout |
|------------|--------------|---------|
| Simple (typo fix, small change) | 30s | 10min |
| Medium (feature, refactor) | 60s | 30min |
| Complex (large feature, multi-file) | 120s | 60min |

Default to **Medium** unless the user's description clearly indicates otherwise.

**Each check:**
```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk <name> -n <namespace> -o jsonpath='{.status.phase}'
```

- **Pending/Queued**: Report status, wait, check again. If Queued, mention the agent may be at capacity.
- **Running**: Report elapsed time, wait, check again.
- **Completed**: Fetch logs (Operation 9), summarize the output.
- **Failed**: Fetch logs + conditions, report the error details.
- **Timeout reached**: Report current status and ask user whether to continue monitoring or stop the task.

**On completion, present a summary:**
> Task `my-task` completed in 3m 42s.
> [Summary of the agent's output from logs]

### 13. Create CronTask

Create a KubeOpenCode CronTask that periodically generates Tasks. The CronTask controller handles Task creation directly — no RBAC or ServiceAccount setup needed.

**Natural Language -> Cron Reference:**

| Natural Language | Cron Expression | Notes |
|---|---|---|
| every hour | `0 * * * *` | |
| every day / daily | `0 0 * * *` | Midnight UTC |
| every day at 9am | `0 9 * * *` | |
| every weekday | `0 0 * * 1-5` | |
| every Monday at 9am | `0 9 * * 1` | |
| every week / weekly | `0 0 * * 0` | Sunday midnight |
| every month / monthly | `0 0 1 * *` | |

If the user specifies a schedule interval shorter than 15 minutes, warn them about resource consumption and confirm before proceeding.

**Flow:**

1. Parse the user's natural language schedule into a cron expression using the reference table above
2. Resolve the agent (reuse Operation 7 agent resolution logic)
3. Resolve the namespace (reuse Operation 7 namespace resolution logic)
4. Generate the CronTask YAML
5. Show YAML to the user for confirmation, then apply

**CronTask YAML:**

```yaml
apiVersion: kubeopencode.io/v1alpha1
kind: CronTask
metadata:
  name: <name>
  namespace: <namespace>
spec:
  schedule: "<cron>"
  concurrencyPolicy: Forbid
  maxRetainedTasks: 5
  taskTemplate:
    metadata:
      labels:
        app.kubernetes.io/component: scheduled
    spec:
      agentRef:
        name: <agent-name>
      description: |
        <user's task description>
```

**Key design points:**
- `concurrencyPolicy`: `Forbid` (default, prevents overlapping), `Allow`, or `Replace`
- `maxRetainedTasks`: Number of completed Tasks to retain (oldest deleted first). Default to 5
- Tasks created by a CronTask receive label `kubeopencode.io/crontask=<crontask-name>`
- `<name>` should be user-specified or auto-generated as `<short-slug>-<4-random-hex>`

**Apply the CronTask:**
```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" apply -f /tmp/kubeopencode-crontask-<name>.yaml
```

### 14. List CronTasks

```bash
# All namespaces
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get crontask --all-namespaces

# Specific namespace
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get crontask -n <namespace>
```

Show: name, schedule, suspend status, age.

### 15. Get CronTask Details

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get crontask <name> -n <namespace> -o yaml
```

Summarize: schedule, concurrencyPolicy, maxRetainedTasks, suspend, taskTemplate (agent, description summary).

Also show tasks created by this CronTask:
```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk -n <namespace> -l kubeopencode.io/crontask=<name>
```

### 16. Delete CronTask

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" delete crontask <name> -n <namespace>
```

**Always ask for confirmation before deleting.** Show the CronTask's schedule and description.

### 17. Suspend/Resume CronTask

```bash
# Suspend
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" patch crontask <name> -n <namespace> \
  --type merge -p '{"spec":{"suspend":true}}'

# Resume
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" patch crontask <name> -n <namespace> \
  --type merge -p '{"spec":{"suspend":false}}'
```

Show the current state after patching to confirm the change took effect.

### 18. Manually Trigger CronTask

Create a Task immediately outside the normal schedule:

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" annotate crontask <name> -n <namespace> \
  kubeopencode.io/trigger=true
```

After triggering, check the created Task:
```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk -n <namespace> -l kubeopencode.io/crontask=<name> --sort-by=.metadata.creationTimestamp
```

## Important Notes

- **Never expose secret values** — only show credential names when describing agents.
- **Same namespace**: Task and Agent must be in the same namespace. The Pod also runs in this namespace.
- **Server mode**: If an agent has `serverConfig`, tasks use `--attach` to connect to a persistent server. No special handling needed from the user's perspective.
- **Cleanup**: Tasks may be automatically cleaned up by `KubeOpenCodeConfig` TTL/retention policies. If a task or its pod is not found, it may have been cleaned up.
- **AgentTemplate**: Agents can reference an AgentTemplate via `templateRef` for shared configuration. Template values are used as defaults; agent-level fields override.

### Migration from CronJob-based Scheduling

Previous versions of KubeOpenCode used standard Kubernetes CronJobs with label `kubeopencode.io/scheduler=true` to schedule Tasks. This approach has been replaced by the native CronTask CRD.

If the cluster still has legacy CronJob-based schedules, they can be listed with:

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get cronjobs \
  -l kubeopencode.io/scheduler=true --all-namespaces
```

To migrate, create an equivalent CronTask and delete the old CronJob and its RBAC resources (ServiceAccount, Role, RoleBinding named `kubeopencode-task-scheduler`).
