---
name: cross-project-explore
display_name: 跨项目数据探查
description: 用户问"N 个项目里 X 条件下有多少 wav, 分布如何"时, 用这个 skill. 跨项目核对 attribute schema + 统计 + 抽样验证.
user-invocable: true
allowed-tools: mcp__diffgram__query_projects,mcp__diffgram__query_directories,mcp__diffgram__query_attributes_schema,mcp__diffgram__query_labels_schema,mcp__diffgram__query_files_count,mcp__diffgram__query_files,mcp__diffgram__query_instance_stats
---

# 跨项目数据探查

你将帮用户跨多个项目核对数据分布. **关键: 跨项目时 attribute / label 命名可能不一致, 必须先核对 schema 再写 filter, 否则查出 0 你自己都不知道为啥.**

## 标准流程

### Step 1 — 列出可访问的项目

调 `query_projects()`. 给用户看:
- 哪些项目 + role + 总 audio_count
- 让用户选要核对哪几个

### Step 2 — 核对 attribute schema (必做)

对每个目标项目调 `query_attributes_schema(project)`, 列出每个项目的 attribute groups + templates + options:

```
huakai_999:
  - step    [select: 1, 2]
  - channel_id [select: 3, 4]
  - date    [select: 2026-03-09, 2026-03-10, ...]

bose_changchun:
  - Stage      [select: 1, 2]      # ⚠ 命名不同, huakai 是 step
  - channel    [select: A, B]      # ⚠ 取值不同, huakai 是数字
  - Date       [select: 2025-...]
```

**这一步如果跳过, 后面 filter 写错你查不出来**. 给用户看 schema 表 + 提醒命名差异 + 让用户决定如何统一.

### Step 3 — 列 label schema (如果涉及 label)

`query_labels_schema(project)` 各项目, 看 label name 是否一致 (可能 huakai 用 NG_H, brose 用 LABEL_NG).

### Step 4 — count 估总量 (低 cost)

```
query_files_count({
  project_list: ["huakai_999", "bose_changchun"],
  filter: "a.step = 1 and l.NG_H"
})
→ {count: 12450, breakdown_by_project: {huakai_999: 8230, bose_changchun: 4220}}
```

如果 count = 0, 大概率是 attribute / label 命名不一致, 回 Step 2 查.

### Step 5 — 抽样验证 (sample 5 条)

```
query_files({
  project_list: [...],
  filter: "...",
  limit: 5,
  with_attrs: true,
  with_instances: true
})
```

给用户看 5 个 file 的具体内容 (filename / wd / attrs / instances), 让用户确认 filter 真的命中目标.

### Step 6 — 分布统计 (按 attr / label 分组)

```
query_instance_stats({
  project_list: [...],
  filter: "...",
  group_by: "attr.channel_id"   # 或 "label", "attr.step", "project"
})
→ [
  {key: "huakai_999/channel=3", instance_count: 1234, total_duration_sec: 5678, file_count: 800},
  {key: "huakai_999/channel=4", ...},
  {key: "bose_changchun/A", ...},
]
```

报告 + 表格给用户.

## 典型对话示例

**用户**: 我想知道 huakai_999 和 nsk- 项目里所有 step=1 + label=NG_H 的 wav 数, 分别多少 + 一些样本.

**你**:

1. `query_projects()` → 确认两个项目都有访问权限
2. `query_attributes_schema(huakai_999)` + `query_attributes_schema(nsk-)`
   - huakai 用 `step` (1, 2)
   - nsk- 用 `step` (1, 2) ✓ 一致
3. `query_labels_schema(huakai_999)` + `query_labels_schema(nsk-)`
   - 都有 NG_H ✓
4. `query_files_count({project_list: ["huakai_999", "nsk-"], filter: "a.step = 1 and l.NG_H"})`
   → {count: 5230, breakdown: {huakai_999: 3450, nsk-: 1780}}
5. `query_files({..., limit: 5, with_attrs: true, with_instances: true})` 抽样
   - 给用户看具体 file: filename, wd, attrs={step:1, channel_id:4}, instances=[NG_H 0.5-1.2s]
6. `query_instance_stats({..., group_by: "attr.channel_id"})` 看 channel 分布
7. 总结报告:
   - huakai_999: 3450 wav (channel_id=3: 1820, channel=4: 1630)
   - nsk-: 1780 wav (channel=3: 980, channel=4: 800)
   - 总 instance 时长 NG_H 共 X 秒 / Y 小时
   - 5 条样本展示

## 关键约束

1. **跨项目前先核对 schema**, 不要直接拿一个项目的 filter 套到另一个
2. **count 优先**, 大数据集不要直接 `query_files limit=10000` 拖死服务
3. **抽样验证 5 条**, 防 filter 写错命中不存在的数据
4. **分布统计用 instance_stats, 不要拉全列表自己 group**
5. **报告时同时给数字 + 5 条样本**, 让用户能 sanity check

## 不要做

- ❌ 不调 schema 直接写 `a.step=1` 跨项目查 (可能项目根本没 `step` 这个 attr)
- ❌ 不抽样直接给用户 count 数字 (用户没法验证 filter 对不对)
- ❌ `limit > 1000` 拉全列表 (用 stats 或者 batch_pipeline 分页)
- ❌ 跨项目 query 不传 `project_list` 默认全 (可能扫海量数据)
