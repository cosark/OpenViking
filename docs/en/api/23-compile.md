# Compile API

Compile is OpenViking's asynchronous task API. OpenViking validates the request, persists the task, and owns its lifecycle before delegating execution to the configured Compile server. The bundled VikingBot implements the same execution protocol for local deployments.

**Code entry points**:

- `openviking/server/routers/compile.py` - Compile task creation
- `openviking/server/routers/tasks.py` - task inspection and cancellation
- `openviking/service/compile_service.py` - Compile server calls and task state convergence

## Create a task

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `from` | string[] | Yes | - | One or more source directories |
| `to` | string | Yes | - | Target Resource or Memory directory, or a supported Skill namespace |
| `skill` | string | Yes | - | Skill directory or its `SKILL.md` URI |
| `reason` | string | No | Skill-driven default | Additional instructions for this Compile run |
| `args` | object | No | - | Compile server extensions; `model_name` accepts a model endpoint ID |

The entire `args` object is optional, and the model endpoint ID is not a top-level field. Use `args.model_name` to select a model; when omitted, the Compile server uses its default model configuration.

**HTTP API**

```http
POST /api/v1/compile
```

```bash
curl -X POST http://localhost:1933/api/v1/compile \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-key" \
  -d '{
    "from": ["viking://resources/research"],
    "to": "viking://resources/research-wiki",
    "skill": "viking://user/default/skills/research-compiler",
    "reason": "Track the historical progress and preserve supporting evidence.",
    "args": {"model_name": "your-model-endpoint-id"}
  }'
```

The endpoint returns `202 Accepted` with an OV task record:

```json
{
  "status": "ok",
  "result": {
    "task_id": "cmp_01abc",
    "task_type": "compile",
    "status": "pending",
    "stage": "queued",
    "resource_id": "viking://resources/research"
  }
}
```

**CLI**

```bash
ov compile \
  --from viking://resources/research \
  --to viking://resources/research-wiki \
  --skill viking://user/default/skills/research-compiler \
  --reason "Track the historical progress and preserve supporting evidence." \
  --args '{"model_name":"your-model-endpoint-id"}'
```

`--args` must be a JSON object. The command returns a task ID immediately after submission.

**SDKs**

The Python, TypeScript, and Go SDKs pass `reason` and `args` through their Compile options:

::: code-group

```python [Python]
task = client.compile(
    ["viking://resources/research"],
    "viking://resources/research-wiki",
    "viking://user/default/skills/research-compiler",
    {"args": {"model_name": "your-model-endpoint-id"}},
)
```

```ts [TypeScript]
const task = await client.compile(
  ["viking://resources/research"],
  "viking://resources/research-wiki",
  "viking://user/default/skills/research-compiler",
  { args: { model_name: "your-model-endpoint-id" } },
);
```

```go [Go]
task, err := client.Compile(
    ctx,
    []string{"viking://resources/research"},
    "viking://resources/research-wiki",
    "viking://user/default/skills/research-compiler",
    &openviking.CompileOptions{
        Args: map[string]any{"model_name": "your-model-endpoint-id"},
    },
)
```

:::

## Get task status

A task is visible only to the principal that created it. A missing task and a task owned by another principal both return `404`.

```http
GET /api/v1/tasks/{task_id}
```

```bash
ov task status cmp_01abc
```

Terminal task responses also contain the result or error.

## Cancel a task

```http
POST /api/v1/tasks/{task_id}/cancel
```

```bash
ov task cancel cmp_01abc
```

The task first enters `cancelling`, then becomes `cancelled` after in-process work and cleanup settle. Writes that already completed are not rolled back. Repeated cancellation of an already `cancelled` task is idempotent.

| Status | Typical stages |
|--------|----------------|
| `pending` | `queued` |
| `running` | Execution stage reported by the Compile server, such as `agent` or `writing` |
| `cancelling` | Settling in-process work and resource cleanup |
| `completed` | `completed`, `salvaged` |
| `failed` | Stage where the failure occurred; the response contains `error` |
| `cancelled` | `cancelled` |

## Legacy endpoints

The following VikingBot routes are retired and return migration guidance only:

```http
POST /bot/v1/compile
GET /bot/v1/compile/{task_id}
POST /bot/v1/compile/{task_id}/cancel
```

Create tasks through `/api/v1/compile`; inspect and cancel them through `/api/v1/tasks/{task_id}`.

## Related documentation

- [Background Tasks](17-tasks.md) - generic task inspection, cancellation, and listing
- [Context Compilation](../context-compilation/01-overview.md) - Compile scenarios and examples
- [Skills API](04-skills.md) - managing the Skills used by Compile
