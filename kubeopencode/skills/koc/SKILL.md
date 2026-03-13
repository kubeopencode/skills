---
name: koc
description: >
  Manage KubeOpenCode AI tasks and agents on Kubernetes. This skill handles
  anything related to running AI-powered tasks on a Kubernetes cluster via
  KubeOpenCode — creating tasks, listing agents, checking task status, viewing
  logs, stopping tasks, monitoring task progress, and scheduling recurring tasks.
  Trigger this skill when the user wants to: create or run an AI task on the
  cluster, see what agents are available, check on a running task, view task
  output or logs, stop a task, use a specific agent for a job, schedule a
  recurring task, list schedules, create a cronjob, set up a daily/weekly task,
  suspend/resume a schedule, or anything involving KubeOpenCode resources
  (Task, Agent). Also trigger when users say things like "run this
  on the cluster", "use agent X to do Y", "what's my task doing", "show me the
  logs", "is it done yet", "schedule a daily task", "run this every hour",
  or refer to kubeopencode, tk, or ag resources.
  Supports namespace flag: --namespace/--ns to override default namespace.
---

# KubeOpenCode Skill

Interact with KubeOpenCode Kubernetes resources (Task, Agent) using kubectl.

## Prerequisites

Requires `kubectl` configured with access to a cluster running KubeOpenCode.

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `KUBEOPENCODE_KUBECONFIG` | **Yes** | Path to kubeconfig for the KubeOpenCode cluster |
| `KUBEOPENCODE_DEFAULT_NAMESPACE` | No | Default namespace for all operations (if unset, use `--all-namespaces` for list, ask on create) |

**CRITICAL**: Every `kubectl` command MUST include `--kubeconfig "$KUBEOPENCODE_KUBECONFIG"`. If this env var is not set, ask the user to set it before proceeding — do NOT fall back to the default kubeconfig. Check this env var in each Bash call since shell state does not persist between calls.

### Namespace Flag

Users can pass a namespace flag as a skill argument to override the default namespace for the current invocation:

| Flag | Short Form | Description |
|------|------------|-------------|
| `--namespace <ns>` | `--ns <ns>` | Override `KUBEOPENCODE_DEFAULT_NAMESPACE` for this invocation |

**Examples:**
```
/koc --ns production list tasks
/koc --namespace monitoring list agents
```

## API Quick Reference

- **API Group**: `kubeopencode.io/v1alpha1`
- **Resources & Short Names**: Task (`tk`), Agent (`ag`), KubeOpenCodeConfig (`ktc`)
- **Task Phases**: `Pending` -> `Queued` -> `Running` -> `Completed` | `Failed`
- **Stop Annotation**: `kubeopencode.io/stop=true`
- **Pod Naming**: `<task-name>-pod`
- **Pod Location**: Pod runs in the same namespace as the Task
- **Scheduled Tasks**: Standard K8s CronJob with label `kubeopencode.io/scheduler=true`
- **Scheduler ServiceAccount**: `kubeopencode-task-scheduler` (auto-created with RBAC if missing)
- **Schedule Name**: `schedule-<short-slug>-<4-random-hex>`

For detailed field reference, read: `references/api-reference.md`

## Operations

**Namespace flag behavior for list/get operations**: When `--namespace`/`--ns` is provided, use it for all operations. If no flag or env var is set, fall back to `--all-namespaces` for list operations.

### 1. List Agents

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

Summarize: profile, executorImage, workspaceDir, maxConcurrentTasks, quota, serverConfig presence, credentials (names only — never show secret values).

### 3. List Tasks

```bash
# All namespaces
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk --all-namespaces

# Specific namespace
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk -n <namespace>
```

Show: name, phase, agent, pod, age.

### 4. Create Task

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

After successful creation, **automatically start monitoring** (see Operation 8).

### 5. Get Task Status

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

### 6. Get Task Logs

First resolve the pod name from the task status:
```bash
# Get pod name and fetch logs — pod is in the same namespace as the task
POD_NAME=$(kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get tk <task-name> -n <namespace> -o jsonpath='{.status.podName}')
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" logs "$POD_NAME" -n <namespace> -c agent
```

If the task is still running, add `--tail=100` to show recent logs.
If the pod is not found, the task may have been cleaned up — inform the user.

### 7. Stop Task

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" annotate task <name> -n <namespace> kubeopencode.io/stop=true
```

**Always warn the user first**: Stopping a task deletes the pod, and **logs will be lost** unless an external log aggregation system is in place. Ask for confirmation before executing.

### 8. Delete Task

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" delete task <name> -n <namespace>
```

Deleting a Task cascades to its Pod and ConfigMap via OwnerReference. Ask for confirmation — this is irreversible.

Only use delete when the user explicitly wants to remove the Task resource. For stopping a running task, prefer the stop annotation (Operation 7).

### 9. Monitor Task

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
- **Completed**: Fetch logs (Operation 6), summarize the output.
- **Failed**: Fetch logs + conditions, report the error details.
- **Timeout reached**: Report current status and ask user whether to continue monitoring or stop the task.

**On completion, present a summary:**
> Task `my-task` completed in 3m 42s.
> [Summary of the agent's output from logs]

### 10. Schedule Task

Create a Kubernetes CronJob that periodically generates KubeOpenCode Tasks. This enables recurring/scheduled task execution without a custom CRD.

**Natural Language → Cron Reference:**

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
2. Resolve the agent (reuse Operation 4 agent resolution logic)
3. Resolve the namespace (reuse Operation 4 namespace resolution logic)
4. Check if ServiceAccount `kubeopencode-task-scheduler` exists in the namespace:
   ```bash
   kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get sa kubeopencode-task-scheduler -n <namespace> 2>/dev/null
   ```
5. If the ServiceAccount does **not** exist, generate RBAC resources (ServiceAccount + Role + RoleBinding) as YAML
6. Generate the CronJob YAML
7. Show all generated YAML to the user for confirmation, then apply

**RBAC Resources** (only created if ServiceAccount does not exist):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubeopencode-task-scheduler
  namespace: <namespace>
  labels:
    kubeopencode.io/scheduler: "true"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kubeopencode-task-scheduler
  namespace: <namespace>
  labels:
    kubeopencode.io/scheduler: "true"
rules:
  - apiGroups: ["kubeopencode.io"]
    resources: ["tasks"]
    verbs: ["create", "get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubeopencode-task-scheduler
  namespace: <namespace>
  labels:
    kubeopencode.io/scheduler: "true"
subjects:
  - kind: ServiceAccount
    name: kubeopencode-task-scheduler
    namespace: <namespace>
roleRef:
  kind: Role
  name: kubeopencode-task-scheduler
  apiGroup: rbac.authorization.k8s.io
```

**CronJob YAML:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: schedule-<slug>-<hex>
  namespace: <namespace>
  labels:
    kubeopencode.io/scheduler: "true"
    kubeopencode.io/agent-name: "<agent>"
  annotations:
    kubeopencode.io/task-description: "<description>"
    kubeopencode.io/created-by: "koc-skill"
spec:
  schedule: "<cron>"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 0
      ttlSecondsAfterFinished: 3600
      template:
        metadata:
          labels:
            kubeopencode.io/scheduler: "true"
        spec:
          serviceAccountName: kubeopencode-task-scheduler
          restartPolicy: Never
          containers:
            - name: task-creator
              image: bitnami/kubectl:1.31
              command: ["/bin/sh", "-c"]
              args:
                - |
                  TASK_NAME="task-<slug>-$(head -c 2 /dev/urandom | od -A n -t x1 | tr -d ' \n')"
                  cat <<'TASKEOF' | sed "s/__KUBEOPENCODE_TASK_NAME__/$TASK_NAME/" | kubectl apply -f -
                  apiVersion: kubeopencode.io/v1alpha1
                  kind: Task
                  metadata:
                    name: __KUBEOPENCODE_TASK_NAME__
                    namespace: <namespace>
                    labels:
                      kubeopencode.io/scheduled-by: schedule-<slug>-<hex>
                  spec:
                    agentRef:
                      name: <agent>
                    description: |
                      <user's task description>
                  TASKEOF
```

**Key design points:**
- CronJob runs in-cluster using the ServiceAccount's credentials — no `KUBEOPENCODE_KUBECONFIG` needed inside the Job
- `concurrencyPolicy: Forbid` prevents overlapping executions
- Each created Task gets label `kubeopencode.io/scheduled-by` for traceability back to the CronJob
- `<slug>` is a short kebab-case summary of the task description (3-4 words max)
- `<hex>` is 4 random hex characters for uniqueness

**Apply the resources:**
```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" apply -f /tmp/kubeopencode-schedule-<name>.yaml
```

### 11. List Scheduled Tasks

```bash
# Specific namespace
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get cronjobs \
  -l kubeopencode.io/scheduler=true \
  -n <namespace> \
  -o custom-columns=NAME:.metadata.name,SCHEDULE:.spec.schedule,AGENT:.metadata.labels.kubeopencode\.io/agent-name,SUSPENDED:.spec.suspend,LAST-SCHEDULE:.status.lastScheduleTime

# All namespaces (when no namespace is specified or configured)
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get cronjobs \
  -l kubeopencode.io/scheduler=true \
  --all-namespaces \
  -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,SCHEDULE:.spec.schedule,AGENT:.metadata.labels.kubeopencode\.io/agent-name,SUSPENDED:.spec.suspend,LAST-SCHEDULE:.status.lastScheduleTime
```

Use the same namespace resolution as list operations: if a namespace flag or env var is set, use it; otherwise use `--all-namespaces`.

### 12. Delete Scheduled Task

```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" delete cronjob <name> -n <namespace>
```

**Always ask for confirmation before deleting.** Show the CronJob's schedule and description from its annotations.

After deletion, check if this was the last KubeOpenCode-managed CronJob in the namespace:
```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" get cronjobs \
  -l kubeopencode.io/scheduler=true \
  -n <namespace> --no-headers | wc -l
```

If the count is 0 (no remaining schedules), ask the user if they want to clean up the RBAC resources:
```bash
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" delete sa,role,rolebinding kubeopencode-task-scheduler -n <namespace>
```

### 13. Suspend/Resume Scheduled Task

```bash
# Suspend a schedule
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" patch cronjob <name> -n <namespace> \
  -p '{"spec":{"suspend":true}}'

# Resume a schedule
kubectl --kubeconfig "$KUBEOPENCODE_KUBECONFIG" patch cronjob <name> -n <namespace> \
  -p '{"spec":{"suspend":false}}'
```

Show the current state after patching to confirm the change took effect.

## Important Notes

- **Never expose secret values** — only show credential names when describing agents.
- **Same namespace**: Task and Agent must be in the same namespace. The Pod also runs in this namespace.
- **Server mode**: If an agent has `serverConfig`, tasks use `--attach` to connect to a persistent server. No special handling needed from the user's perspective.
- **Cleanup**: Tasks may be automatically cleaned up by `KubeOpenCodeConfig` TTL/retention policies. If a task or its pod is not found, it may have been cleaned up.
