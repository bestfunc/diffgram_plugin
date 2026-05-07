---
name: mcp-tools
display_name: Diffgram MCP tool 列表
description: 25 个 tool 的 input/output schema 速查. 包含 query / mark / move / directory / task / batch_pipeline 5 大模块.
user-invocable: false
---

# Diffgram MCP tool 列表

按 scope (module) 分组. **read** 只读, **write** 涉及 dry_run + confirm 二次确认.

## query 模块 (跨项目, 只读, scope: `query`)

| tool | input | output |
|---|---|---|
| `query_projects` | (空) 或 `{role_filter: ["admin","Editor"]}` | `[{string_id, name, role, audio_count}]` |
| `query_directories` | `{project, name_pattern?}` | `[{id, nickname, audio_count, archived}]` |
| `query_attributes_schema` | `{project}` | `{groups: [{name, templates: [{name, kind, options, usage_count}]}]}` |
| `query_labels_schema` | `{project}` | `[{id, name, color, instance_count}]` |
| `query_files` | `{project_list?: [str], project?, filter, limit=100, offset=0, with_attrs=false, with_instances=false}` | `{total: int, results: [{file_id, filename, wd_list, attrs?, instances?}]}` |
| `query_files_count` | `{project_list?, project?, filter}` | `{count, breakdown_by_project: {...}}` |
| `query_file_detail` | `{project, file_id}` | `{file_id, filename, duration_sec, attrs, instances: [{id, label, start_time, end_time}], wd_list}` |
| `query_instance_stats` | `{project_list?, project?, filter, group_by: "label"\|"attr.<name>"}` | `{groups: [{key, instance_count, total_duration_sec, file_count}]}` |

**filter 语法**: 见 `query-syntax` skill.

**跨项目 vs 单项目**: `project_list` 多个 / `project` 单个; 二选一, 不传则用 `query_projects` 默认全部用户能访问的.

## mark 模块 (单项目, 写, scope: `mark`)

⚠ **强制 dry_run + confirm 模式**:
- 第 1 调: `dry_run: true` → 返回 `{request_id, preview, affected_count}`
- 第 2 调: `request_id: <id>, confirm: true` → 真执行

| tool | input | output |
|---|---|---|
| `mark_set_attribute` | `{project, file_ids, attrs: {key: value}, dry_run\|request_id+confirm}` | `{preview, request_id}` 或 `{updated: int}` |
| `mark_remove_attribute` | `{project, file_ids, attr_keys: [str], dry_run\|...}` | 同上 |
| `mark_set_label` | `{project, file_ids, label_name, start_sec?, end_sec?, dry_run\|...}` | `{preview, request_id}` 或 `{instance_ids: [int]}` |
| `mark_remove_label` | `{project, file_ids, label_name, instance_ids?, dry_run\|...}` | 同 |
| `mark_update_metadata` | `{project, file_id, filename?, custom_fields?}` | `{updated: bool}` |

**约束**: `file_ids` 单次 ≤ 100,000. 超过分批.

## move 模块 (单项目, 写, scope: `move`)

⚠ 同 mark, dry_run + confirm 强制.

| tool | input | output |
|---|---|---|
| `move_files_to_dir` | `{project, file_ids, source_wd, target_wd, dry_run\|...}` | `{moved: int}` |
| `move_files_clone_to_dir` | `{project, file_ids, target_wd, dry_run\|...}` | `{cloned: int}` (加 link 不删原) |
| `move_files_remove_from_dir` | `{project, file_ids, wd, dry_run\|...}` | `{removed: int}` (仅删 link) |
| `move_files_to_dir_with_query` | `{project, filter, source_wd, target_wd, dry_run\|...}` | `{moved: int, sample_files: [...]}` |

`wd` 字段接受 `wd_id` (int) 或 `wd_nickname` (str).

## directory 模块 (单项目, 写, scope: `directory`)

| tool | input | output |
|---|---|---|
| `directory_create` | `{project, nickname, description?}` | `{id, nickname}` (已存在抛错) |
| `directory_get_or_create` | `{project, nickname}` | `{id, nickname, was_created: bool}` |
| `directory_rename` | `{project, wd, new_nickname}` | `{id, old, new}` |
| `directory_archive` | `{project, wd}` | `{archived: bool}` |
| `directory_delete` | `{project, wd, force?}` | `{deleted: bool}` (默认非空目录拒绝, force=true 强制) |

## task 模块 (单项目, 写, scope: `task`)

| tool | input | output |
|---|---|---|
| `task_create_for_files` | `{project, file_ids, job_id, assigned_user_email?}` | `{task_ids: [int]}` |
| `task_assign` | `{task_id, user_email}` | `{assigned: bool}` |
| `task_query` | `{project?, status?, job_id?, assigned_user_email?, limit=100}` | `[{id, file_id, status, assigned}]` |

## batch_pipeline (链式调用)

```json
{
  "tool": "batch_pipeline",
  "args": {
    "steps": [
      {"id": "step_id", "tool": "<any tool>", "args": {...}}
    ]
  }
}
```

**JSON-path 引用**: 后续 step 可以引用前 step 的输出, 用 `$.<step_id>.<json_path>`:

```json
{
  "steps": [
    {"id": "q",  "tool": "query_files", "args": {"project": "huakai_999", "filter": "...", "limit": 1000}},
    {"id": "d",  "tool": "directory_get_or_create", "args": {"project": "huakai_999", "nickname": "TRAIN_v3"}},
    {"id": "mv", "tool": "move_files_to_dir", "args": {
      "project": "huakai_999",
      "file_ids": "$.q.results[*].file_id",
      "target_wd": "$.d.id",
      "dry_run": true
    }}
  ]
}
```

**返回**: `{steps: {step_id: result}}`. 任意 step 失败整个 pipeline abort 并返回已完成的 + 错误.

## 错误码

| code | 含义 | 处理 |
|---|---|---|
| 401 | Bearer token 无效 / 过期 / 吊销 | 用户重新生成 token |
| 403 | scope 不足 / 无项目权限 | 检查 token scopes / 项目 role |
| 404 | project / file / wd 不存在 | 用 `query_projects` 重新对名 |
| 409 | request_id 过期或已用 | 重跑 dry_run |
| 422 | filter 语法错误 | 看 `query-syntax` skill |
| 429 | rate limit | 等 60s 重试 |
| 500 | 服务端错误 | 看 args_summary 上报 |

## tool 命名约定

`<module>_<action>` 一律下划线小写. e.g.:
- `query_files` (不是 `queryFiles`)
- `mark_set_label` (不是 `markSetLabel`)
- `directory_get_or_create`
