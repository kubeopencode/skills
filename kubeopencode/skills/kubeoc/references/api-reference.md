# KubeOpenCode API Reference

Detailed field reference for KubeOpenCode CRDs. API Group: `kubeopencode.io/v1alpha1`.

## Task (`tk`)

### TaskSpec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `agentRef` | AgentReference | Yes | Agent to use. Must be in the same namespace as the Task. |
| `description` | *string | No | Task instruction/prompt. Written to `${WORKSPACE_DIR}/task.md`. |
| `contexts` | []ContextItem | No | Additional context for the task. |

### AgentReference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Agent name. Must be in the same namespace as the Task. |

### TaskExecutionStatus

| Field | Type | Description |
|-------|------|-------------|
| `phase` | TaskPhase | Current phase: Pending, Queued, Running, Completed, Failed. |
| `agentRef` | AgentReference | Resolved agent reference. |
| `podName` | string | Pod name running the task (in the same namespace as the Task). |
| `startTime` | Time | When the task started running. |
| `completionTime` | Time | When the task finished. |
| `conditions` | []Condition | Standard Kubernetes conditions (Ready, Queued, Stopped). |

## Agent (`ag`)

### AgentSpec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `templateRef` | AgentTemplateReference | No | Reference to an AgentTemplate in the same namespace. Agent inherits template config; agent-level fields override. |
| `profile` | string | No | Brief human-readable summary of the Agent's purpose and capabilities. Visible via `-o wide`. |
| `agentImage` | string | No | OpenCode init container image. Default: `quay.io/kubeopencode/kubeopencode-agent-opencode:latest`. |
| `executorImage` | string | No | Main worker container image. Default: `quay.io/kubeopencode/kubeopencode-agent-devbox:latest`. |
| `attachImage` | string | No | Lightweight image for server-mode attach pods. Default: `quay.io/kubeopencode/kubeopencode-agent-attach:latest`. |
| `workspaceDir` | string | Yes | Working directory (must start with `/`). |
| `command` | []string | No | Entrypoint command override. Default: `["sh", "-c", "/tools/opencode run \"$(cat ${WORKSPACE_DIR}/task.md)\""]`. |
| `contexts` | []ContextItem | No | Default contexts for all tasks using this agent. |
| `config` | *string | No | OpenCode config as JSON string. Written to `/tools/opencode.json`. |
| `credentials` | []Credential | No | Secrets mounted as env vars or files. |
| `podSpec` | AgentPodSpec | No | Advanced pod configuration (labels, scheduling, resources, runtimeClass, security). |
| `serviceAccountName` | string | Yes | Kubernetes ServiceAccount for agent pods. |
| `maxConcurrentTasks` | *int32 | No | Max concurrent running tasks. nil/0 = unlimited. |
| `quota` | QuotaConfig | No | Rate limiting: `maxTaskStarts` per `windowSeconds`. |
| `caBundle` | CABundleConfig | No | Custom CA certificates for TLS verification in all containers. |
| `proxy` | ProxyConfig | No | HTTP/HTTPS proxy settings for all containers. Overrides cluster-level proxy. |
| `imagePullSecrets` | []LocalObjectReference | No | References to Secrets for pulling images from private registries. |
| `serverConfig` | ServerConfig | No | Enables persistent server mode (Deployment + Service). |

### AgentTemplateReference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Name of the AgentTemplate in the same namespace. |

### Credential

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Descriptive name. |
| `secretRef.name` | string | Yes | Secret name. |
| `secretRef.key` | *string | No | Specific key. If omitted, entire secret is used. |
| `env` | *string | No | Env var name (only with secretRef.key). |
| `mountPath` | *string | No | File mount path. |
| `fileMode` | *int32 | No | File permission (default: 0600). |

**Mounting behavior depends on whether `secretRef.key` is specified:**

1. No key + no mountPath → entire Secret as environment variables
2. No key + mountPath → entire Secret as directory (each key becomes a file)
3. key + env → single key as environment variable
4. key + mountPath → single key as file

### QuotaConfig

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `maxTaskStarts` | int32 | Yes | Max task starts within window (min: 1). |
| `windowSeconds` | int32 | Yes | Sliding window in seconds (60-86400). |

### CABundleConfig

Exactly one of `configMapRef` or `secretRef` must be specified.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `configMapRef` | CABundleReference | No* | ConfigMap containing PEM-encoded CA certificates. |
| `secretRef` | CABundleReference | No* | Secret containing PEM-encoded CA certificates. |

### CABundleReference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Name of the ConfigMap or Secret. |
| `key` | string | No | Key containing the PEM bundle. Default: `ca-bundle.crt` (ConfigMap), `ca.crt` (Secret). |

### ProxyConfig

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `httpProxy` | string | No | HTTP proxy URL. Sets `HTTP_PROXY` and `http_proxy`. |
| `httpsProxy` | string | No | HTTPS proxy URL. Sets `HTTPS_PROXY` and `https_proxy`. |
| `noProxy` | string | No | Comma-separated bypass list. `.svc` and `.cluster.local` always appended automatically. |

### AgentPodSpec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `labels` | map[string]string | No | Additional labels for agent pods (e.g., NetworkPolicy, monitoring). |
| `scheduling` | PodScheduling | No | Node selection, tolerations, affinity rules. |
| `runtimeClassName` | *string | No | RuntimeClass for enhanced isolation (e.g., `gvisor`, `kata`). |
| `resources` | ResourceRequirements | No | CPU/memory requests and limits for the agent container. |
| `securityContext` | SecurityContext | No | Container-level security options. Default: no privilege escalation, drop ALL caps, RuntimeDefault seccomp. |
| `podSecurityContext` | PodSecurityContext | No | Pod-level security attributes (runAsUser, fsGroup, etc.). |

### PodScheduling

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `nodeSelector` | map[string]string | No | Label selector for node scheduling. |
| `tolerations` | []Toleration | No | Tolerations for tainted nodes. |
| `affinity` | Affinity | No | Advanced affinity/anti-affinity rules. |

### ServerConfig

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `port` | int32 | No | Server port (default: 4096, range: 1-65535). |
| `persistence` | PersistenceConfig | No | Persistent storage for sessions and workspace. |
| `suspend` | bool | No | Scales server to 0 replicas when true. PVCs and Service retained. Tasks enter Queued phase until resumed. |

### PersistenceConfig

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sessions` | VolumePersistence | No | PVC for OpenCode session data (SQLite DB). Survives pod restarts. |
| `workspace` | VolumePersistence | No | PVC for workspace directory. Without this, workspace uses EmptyDir. |

### VolumePersistence

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `storageClassName` | *string | No | StorageClass for PVC. Cluster default if empty. |
| `size` | string | No | PVC size. Default: 1Gi (sessions), 10Gi (workspace). |

## AgentTemplate (`agt`)

Reusable base configuration for Agents. Teams maintain a single template with shared settings; Agents reference it via `spec.templateRef.name` and inherit configuration.

**Merge strategy:** Agent scalar/pointer fields override template if non-zero/non-nil. Agent list fields (contexts, credentials, imagePullSecrets) replace template lists entirely.

**Agent-only fields** (NOT in AgentTemplate): `profile`, `maxConcurrentTasks`, `quota`, `templateRef`.

### AgentTemplateSpec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `agentImage` | string | No | OpenCode init container image. |
| `executorImage` | string | No | Main worker container image. |
| `attachImage` | string | No | Lightweight image for server-mode attach pods. |
| `workspaceDir` | string | Yes | Working directory (must start with `/`). |
| `command` | []string | No | Entrypoint command override. |
| `contexts` | []ContextItem | No | Default contexts for all tasks. |
| `config` | *string | No | OpenCode config as JSON string. |
| `credentials` | []Credential | No | Secrets for the agent. |
| `podSpec` | AgentPodSpec | No | Advanced pod configuration. |
| `serviceAccountName` | string | Yes | Kubernetes ServiceAccount. |
| `caBundle` | CABundleConfig | No | Custom CA certificates. |
| `proxy` | ProxyConfig | No | HTTP/HTTPS proxy settings. |
| `imagePullSecrets` | []LocalObjectReference | No | Private registry secrets. |
| `serverConfig` | ServerConfig | No | Enables Server mode. |

## ContextItem

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Identifier for logging and XML tag generation. |
| `description` | string | No | Human-readable documentation. |
| `type` | ContextType | Yes | `Text`, `ConfigMap`, `Git`, `Runtime`, or `URL`. |
| `mountPath` | string | No* | Mount location. Relative paths prefixed with workspaceDir. |
| `fileMode` | *int32 | No | File permission for mounted file (default: 0644). |
| `text` | string | If Text | Inline text content. |
| `configMap` | ConfigMapContext | If ConfigMap | ConfigMap reference. |
| `git` | GitContext | If Git | Git repository reference. |
| `runtime` | RuntimeContext | If Runtime | No fields. Injects platform awareness. |
| `url` | URLContext | If URL | URL fetch configuration. |

*`mountPath` is required for Git context type. If empty for other types, content goes to `task.md`.

### ConfigMapContext

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | ConfigMap name. |
| `key` | string | No | Specific key. If omitted, all keys mounted as files. |
| `optional` | *bool | No | If true, ConfigMap does not need to exist. |

### GitContext

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `repository` | string | Yes | Git repository URL. |
| `ref` | string | No | Branch, tag, or commit SHA. Default: `HEAD`. |
| `path` | string | No | Path within repo. Empty = entire repo (includes `.git/`). |
| `depth` | *int | No | Clone depth. 1 = shallow (default), 0 = full. |
| `recurseSubmodules` | bool | No | Clone submodules recursively. Default: false. |
| `secretRef` | GitSecretReference | No | Secret with git credentials (`username`+`password` or `ssh-privatekey`). |

### URLContext

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `source` | string | Yes | HTTP/HTTPS URL to fetch. |
| `headers` | map[string]string | No | HTTP headers for the request. |
| `secretRef` | URLSecretReference | No | Secret with auth credentials (`token` or `username`+`password`). |
| `insecureSkipTLSVerify` | bool | No | Skip TLS certificate verification. Use only for testing. |
| `timeout` | *int32 | No | Request timeout in seconds (default: 30). |

## CronTask

Namespace-scoped. Creates Tasks on a cron schedule (analogous to CronJob creating Jobs, but without requiring kubectl Pods or RBAC for Task creation).

### CronTaskSpec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schedule` | string | Yes | Cron expression (e.g., `"0 9 * * 1"`). |
| `concurrencyPolicy` | string | No | `Forbid` (default), `Allow`, or `Replace`. |
| `maxRetainedTasks` | int32 | No | Max completed Tasks to retain (oldest deleted first). |
| `suspend` | bool | No | Pause scheduling when true. |
| `taskTemplate` | TaskTemplate | Yes | Template for Tasks created by this CronTask. |

### TaskTemplate

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `metadata` | ObjectMeta | No | Labels/annotations for created Tasks. |
| `spec` | TaskSpec | Yes | Task specification (agentRef, description, contexts). |

### CronTask Labels

Tasks created by a CronTask receive the label `kubeopencode.io/crontask=<crontask-name>`.

### Manual Trigger

Annotate with `kubeopencode.io/trigger=true` to create a Task immediately:

```bash
kubectl annotate crontask <name> -n <namespace> kubeopencode.io/trigger=true
```

## KubeOpenCodeConfig (`ktc`)

Cluster-scoped singleton. Must be named `cluster`.

### KubeOpenCodeConfigSpec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `systemImage` | SystemImageConfig | No | System image for init containers (`image`, `imagePullPolicy`). |
| `cleanup` | CleanupConfig | No | Task cleanup policies. |
| `proxy` | ProxyConfig | No | Cluster-wide proxy settings. Agent-level proxy takes precedence. |

### CleanupConfig

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ttlSecondsAfterFinished` | int32 | No | Delete tasks after N seconds from completion. |
| `maxRetainedTasks` | int32 | No | Max completed tasks per namespace (oldest deleted first). |
