# Compile API

Compile 是 OpenViking 的异步任务接口。OpenViking 负责校验请求、持久化任务和管理生命周期，再把执行交给配置的 Compile Server；内置 VikingBot 也实现了同一执行协议，可用于本地部署。

**代码入口**：

- `openviking/server/routers/compile.py` - 创建 Compile 任务
- `openviking/server/routers/tasks.py` - 查询和取消任务
- `openviking/service/compile_service.py` - Compile Server 调用与任务状态收敛

## 创建任务

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `from` | string[] | 是 | - | 一个或多个来源目录 |
| `to` | string | 是 | - | 目标 Resource 或 Memory 目录，或受支持的 Skill namespace |
| `skill` | string | 是 | - | Skill 目录或其 `SKILL.md` URI |
| `reason` | string | 否 | Skill 驱动的默认值 | 本次 Compile 的补充指令 |
| `args` | object | 否 | - | Compile Server 扩展参数；`model_name` 可传模型 Endpoint ID |

`args` 整体可省略，模型 Endpoint ID 也不是顶层字段。需要指定模型时使用 `args.model_name`；不传时由 Compile Server 使用其默认模型配置。

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
    "reason": "追踪历史进展，并保留支撑证据。",
    "args": {"model_name": "your-model-endpoint-id"}
  }'
```

接口返回 `202 Accepted` 和 OV TaskRecord：

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
  --reason "追踪历史进展，并保留支撑证据。" \
  --args '{"model_name":"your-model-endpoint-id"}'
```

`--args` 必须是 JSON object。命令提交后立即返回 Task ID。

**SDK**

Python、TypeScript 和 Go SDK 都通过各自的 Compile options 传递 `reason` 和 `args`：

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

## 查询任务

任务仅对创建它的 principal 可见；任务不存在或属于其他 principal 时均返回 `404`。

```http
GET /api/v1/tasks/{task_id}
```

```bash
ov task status cmp_01abc
```

任务进入终态后，响应还会包含结果或错误。

## 取消任务

```http
POST /api/v1/tasks/{task_id}/cancel
```

```bash
ov task cancel cmp_01abc
```

任务会先进入 `cancelling`，待当前进程内工作和清理完成后进入 `cancelled`；已经完成的写入不会回滚。重复取消已经 `cancelled` 的任务是幂等的。

| Status | 常见 Stage |
|--------|------------|
| `pending` | `queued` |
| `running` | Compile Server 返回的执行 Stage，例如 `agent`、`writing` |
| `cancelling` | 收敛当前进程内工作和清理资源 |
| `completed` | `completed`、`salvaged` |
| `failed` | 失败发生时的 Stage；响应包含 `error` |
| `cancelled` | `cancelled` |

## 旧接口

以下 VikingBot 路由已经停用，只返回迁移提示：

```http
POST /bot/v1/compile
GET /bot/v1/compile/{task_id}
POST /bot/v1/compile/{task_id}/cancel
```

创建任务使用 `/api/v1/compile`，查询和取消统一使用 `/api/v1/tasks/{task_id}`。

## 相关文档

- [后台任务](17-tasks.md) - 通用任务查询、取消和列表接口
- [上下文编译](../context-compilation/01-overview.md) - Compile 使用场景和示例
- [Skills API](04-skills.md) - 管理 Compile 使用的 Skill
