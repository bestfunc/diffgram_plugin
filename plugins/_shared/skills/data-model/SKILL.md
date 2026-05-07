---
name: data-model
display_name: Diffgram 数据模型速查
description: file / instance / label / working_dir / attribute 关系图 + 字段含义 + 跨表 query 模式
user-invocable: false
---

# Diffgram 数据模型速查

## 实体关系

```
                   ┌──────────────┐
                   │   project    │  project_string_id 全局唯一
                   └──────┬───────┘
                          │ 1:N
        ┌─────────────────┼──────────────────┐
        ▼                 ▼                  ▼
  ┌──────────┐    ┌────────────────┐   ┌──────────────────────┐
  │working_  │    │attribute_      │   │label  (label_file)   │
  │ dir      │    │template_group  │   │  字典文件 (file.type=│
  │(目录)    │    │(attribute 字段)│   │     'label')         │
  └────┬─────┘    └────────┬───────┘   └──────────┬───────────┘
       │ N:M              │              引用       │
       │ wfl              │ N:M                    │
       ▼                  ▼                        │
  ┌────────────────────────────┐                  │
  │  file (audio/image/video)  │                  │
  │  - id                      │                  │
  │  - original_filename       │                  │
  │  - state (added/available) │                  │
  │  - type ('audio')          │                  │
  │  - attribute_groups JSONB  │←─attribute kv 存这里
  └────────────┬───────────────┘                  │
               │ 1:N                              │
               ▼                                  │
        ┌──────────────────────────┐              │
        │  instance (segment 标注) │              │
        │  - file_id               │              │
        │  - label_file_id ────────┼──────────────┘
        │  - start_time, end_time  │
        │  - attribute_groups JSONB│
        │  - soft_delete           │
        └──────────────────────────┘
```

## 表字段

### project
| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | int | 主键 |
| `project_string_id` | str | 全局唯一字符串 ID, 跨服务引用用这个 (如 "huakai_999") |
| `name` | str | 显示名 |
| `is_public` | bool | 是否公开项目 |

### working_dir (目录, project 内)
| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | int | 主键 |
| `project_id` | int | 所属项目 |
| `nickname` | str | 显示名 (如 "TRAIN_5_0", "EVAL_TEST", "UNMARK") |
| `state` | str | active / archived |

### file (核心实体, audio/image/video/label/compound)
| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `original_filename` | str | 文件名 |
| `state` | str | added / available / deleted |
| `type` | str | audio / image / video / label / compound |
| `attribute_groups` | JSONB | **重要** — file 级 attribute kv, 如 `{"step": 1, "channel_id": 4}` |
| `hash` | str | 内容 hash (去重用) |

**多对多关系** (file ↔ working_dir): `workingdir_file_link` (wfl)
- 一个 file 可以同时在多个 working_dir 里
- 删除 file 不删 wfl link, 改 file.state='deleted'

### instance (segment / whole-file 标注)
| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | bigint | 主键 |
| `file_id` | bigint | 被标注的文件 |
| `label_file_id` | bigint | 引用 label 字典文件 (file.type='label' 那一行) |
| `start_time` | float | 段开始秒 (audio/video) |
| `end_time` | float | 段结束秒 |
| `creation_ref_id` | str | 同一标注会话内多 instance 共享 |
| `attribute_groups` | JSONB | instance 级 attribute (segment 级 confidence 之类) |
| `soft_delete` | bool | 软删除 (查询要带 `WHERE soft_delete IS NOT TRUE`) |

**Whole-file label vs Segment label**:
- Whole-file: `start_time = NULL or 0, end_time = NULL or file_duration`
- Segment: 有具体 `start_time, end_time` 范围

### label (字典)
- 在 diffgram, label 不是单独表; 而是 `file` 表里 `type='label'` 的特殊行 + `label` 表 join
- `label.id` = `file.label_id` (label 字典 file 的关联)
- `label.name` 是字符串 (如 "NG_H", "OK", "OK1")

查 file 上某 label 的 instance:
```sql
SELECT i.*
FROM instance i
JOIN file lf ON lf.id = i.label_file_id AND lf.type = 'label'
JOIN label l ON l.id = lf.label_id
WHERE i.file_id = ? AND l.name = 'NG_H' AND i.soft_delete IS NOT TRUE
```

### attribute_template_group / attribute_template
| 表 | 说明 |
|---|---|
| `attribute_template_group` | 项目级 attribute group (如 "实验 metadata"), 含多个 template |
| `attribute_template` | 单 attribute 字段 (name, kind=numeric/text/select, options JSONB) |

attribute 真实值存在 file/instance 的 `attribute_groups` JSONB 里, 不在 template 表.

## 跨表 query 模式

### 1. 列项目某 attribute 全部取值
```
query_attributes_schema(project)
→ 返回每个 attribute_template + options + 出现次数
```

### 2. 找 file 含某 label
```sql
SELECT f.id
FROM file f
JOIN instance i ON i.file_id = f.id
JOIN file lf ON lf.id = i.label_file_id AND lf.type='label'
JOIN label l ON l.id = lf.label_id
WHERE l.name = 'NG_H'
  AND i.soft_delete IS NOT TRUE
  AND f.state IN ('added','available')
```
→ MCP 用 `query_files(filter: "l.NG_H")` 一行搞定.

### 3. 跨 project 统计 audio 数
```sql
SELECT p.project_string_id, COUNT(DISTINCT f.id)
FROM project p
JOIN working_dir wd ON wd.project_id = p.id
JOIN workingdir_file_link wfl ON wfl.working_dir_id = wd.id
JOIN file f ON f.id = wfl.file_id
WHERE f.type='audio' AND f.state IN ('added','available')
GROUP BY p.project_string_id
```
→ MCP 用 `query_projects` 一行搞定.

### 4. 一个 file 在哪些 working_dir 里
```sql
SELECT wd.nickname
FROM workingdir_file_link wfl
JOIN working_dir wd ON wd.id = wfl.working_dir_id
WHERE wfl.file_id = ?
```
→ `query_file_detail(project, file_id)` 返回里含 `wd_list`.

## 关键约束 (做 MCP 时要遵守)

1. **`soft_delete` 必须带**: instance 软删后表里还在, 查询时必须 `AND i.soft_delete IS NOT TRUE`
2. **`file.state` 也要过滤**: 'added' / 'available' 才是有效, 'deleted' 是软删
3. **跨 project 用 project_string_id, 不用 id**: 内部 id 不稳定 (导出导入会变), string_id 全局唯一
4. **attribute key 跨项目可能不同名**: 不同客户方/项目可能用不同 attribute schema, 跨项目查询前 `query_attributes_schema` 各自核对
5. **whole-file label 用整段 instance**: start=0, end=file_duration. 不是单独类型, 跟 segment 同表

## 计数 vs 列表

- `query_files_count` — 仅 SELECT COUNT, 适合海量 (huakai_999 几十万)
- `query_files` — SELECT 列表, **必须** 带 `limit` (默认 100, 上限 10000)
- 想完整列表不限 limit → 用 batch_pipeline + 多次 query_files (offset 分页)
