---
name: organize-dataset
display_name: 数据集组织 (按规则移到 train/val/test 目录)
description: 用户给规则分组数据 ("90% 训练 10% 验证" / "step=1→train, step=2→val") 后, AI 自动建目录 + 抽样 + 移动. 含 dry_run 安全防线.
user-invocable: true
allowed-tools: mcp__diffgram__query_files,mcp__diffgram__query_files_count,mcp__diffgram__query_directories,mcp__diffgram__directory_get_or_create,mcp__diffgram__directory_create,mcp__diffgram__move_files_to_dir,mcp__diffgram__move_files_clone_to_dir,mcp__diffgram__batch_pipeline
---

# 数据集组织

把符合规则的 file 移到指定 working_dir. 用于:
- 按 attribute 分 train / val / test
- 按 label 分阳性 / 阴性 / 边界
- 按时间窗分新 / 旧 batch

**关键**: 移动是改 wfl 表 (file ↔ wd 多对多), 默认"移" 是 add target + remove source, 如果只想保留原也复制到新 wd, 用 `clone`.

## 标准流程

### Step 1 — 解析规则

让用户描述清楚每条规则:
- **源** (project + 可选 source_wd)
- **筛选条件** (filter)
- **目标 wd** (target_wd nickname)
- **操作类型**: move (默认) / clone (复制不删源)

如果是分批 (e.g. 70/30 split), 让用户决定:
- 按什么字段 split (random / file_id mod / attribute hash 之类)
- 是否需要 stratified (按 label 比例分)

### Step 2 — 列现有 directories

`query_directories(project)` 让用户看现有目录:
```
TRAIN_1.0 (3450 files)
TRAIN_5.0 (8200 files)
EVAL_TEST (1130 files)
UNMARK     (560000 files)
```

让用户决定: 用现有目录还是新建.

### Step 3 — 建目标目录 (如不存在)

```
directory_get_or_create({project, nickname: "TRAIN_v3"})
→ {id: 12345, nickname: "TRAIN_v3", was_created: true}
```

`get_or_create` 不抛错, 安全.

### Step 4 — 估每条规则的总量

每条规则跑 `query_files_count`:
```
规则 1: filter="a.step=1 and !l.NG_H"  → 8430 files → TRAIN_v3
规则 2: filter="a.step=2 and !l.NG_H"  → 2150 files → VAL_v3
规则 3: filter="l.NG_H"                → 6700 files → both (clone)
```

总量给用户预览.

### Step 5 — 抽样 (每规则 3 条) 给用户看

`query_files({..., limit: 3, with_attrs: true})` 显示 3 条 file 样本.

让用户确认 filter 命中目标.

### Step 6 — dry_run 全部移动操作

每条规则都 dry_run:
```
move_files_to_dir({
  project,
  file_ids: [大列表],  # 或先 query_files limit=10000 拿全部
  source_wd: "UNMARK",  # 移动语义需要 source
  target_wd: "TRAIN_v3",
  dry_run: true
})
→ {request_id, preview: "8430 files will be moved from UNMARK to TRAIN_v3"}
```

或者用 `move_files_to_dir_with_query` 一步 (内部组合 query + move):
```
move_files_to_dir_with_query({
  project,
  filter: "a.step=1 and !l.NG_H",
  source_wd: "UNMARK",
  target_wd: "TRAIN_v3",
  dry_run: true
})
```

把所有规则的 dry_run preview 集中报告给用户.

### Step 7 — 用户确认 → confirm 全部

用户最终确认后, 用 `batch_pipeline` 一次性 confirm 全部 move:
```
batch_pipeline({
  steps: [
    {id: "m1", tool: "move_files_to_dir", args: {request_id: "req-1", confirm: true}},
    {id: "m2", tool: "move_files_to_dir", args: {request_id: "req-2", confirm: true}},
    {id: "m3", tool: "move_files_clone_to_dir", args: {request_id: "req-3", confirm: true}}
  ]
})
```

### Step 8 — 报告 + 验证

```
✓ TRAIN_v3: 移入 8430 files (其中 0 失败)
✓ VAL_v3:   移入 2150 files
✓ NG_H 数据 clone 到两个目录: 6700 → TRAIN_v3 (+) + VAL_v3 (+)
回滚: move_files_to_dir(target_wd=UNMARK, source_wd=TRAIN_v3, file_ids=...) 移回
```

调 `query_directories(project)` 确认目录 file_count 已更新.

## 典型场景

### 场景 A: train/val/test 三分

用户: "huakai_999 里 NG_H 数据按 70/15/15 分 train/val/test"

```
1. query_files_count(filter="l.NG_H") → 6700
2. 计算: 4690/1005/1005
3. query_files(filter="l.NG_H", limit=10000) 拿全部 file_id
4. random shuffle (用 seed=42 可复现) + 按 file_id 切 70/15/15
5. directory_get_or_create × 3 (TRAIN_NG_H / VAL_NG_H / TEST_NG_H)
6. dry_run × 3 move_files_to_dir
7. 用户确认 → batch_pipeline × 3 confirm
8. 报告
```

如果用户要 stratified by attribute, 用 `query_instance_stats group_by="attr.X"` 先看分布, 然后每组各自 70/15/15.

### 场景 B: 按时间窗分新旧

用户: "把 date >= '2026-04-01' 的标记数据分到 TRAIN_v3, 早期的进 TRAIN_v2"

```
1. query_files_count(filter="a.date >= '2026-04-01' and l.NG_H") → 1230
2. query_files_count(filter="a.date < '2026-04-01' and l.NG_H") → 5470
3. dir_get_or_create × 2
4. move_files_to_dir_with_query × 2 (dry_run)
5. 确认 → confirm
```

### 场景 C: 标 + 移 (复合)

用户: "step=2 channel=4 的全标 NG_H, 然后移到 TRAIN_v3"

→ 这是 `bulk-mark-by-rule` skill + `organize-dataset` skill 串. 用 batch_pipeline:

```json
{
  "steps": [
    {"id": "q",  "tool": "query_files", "args": {"project": "huakai_999", "filter": "a.step=2 and a.channel_id=4 and !l.NG_H", "limit": 10000}},
    {"id": "ml", "tool": "mark_set_label",
                  "args": {"project": "huakai_999", "file_ids": "$.q.results[*].file_id", "label_name": "NG_H", "dry_run": true}},
    {"id": "d",  "tool": "directory_get_or_create", "args": {"project": "huakai_999", "nickname": "TRAIN_v3"}},
    {"id": "mv", "tool": "move_files_to_dir",
                  "args": {"project": "huakai_999", "file_ids": "$.q.results[*].file_id", "source_wd": "UNMARK", "target_wd": "$.d.id", "dry_run": true}}
  ]
}
```

dry_run 全部, 给用户看 → 用户确认 → 重跑同 pipeline 但所有 dry_run 改成 confirm.

## 安全约束

1. **每条规则都要抽样 + dry_run + 用户确认**
2. **move 默认是"移" (add+remove), clone 是"复制" (只 add)** — 区分清楚再执行
3. **大批量分批**: file_ids 上限 100k 一次
4. **保留 audit log + request_id**, 万一弄错可以回滚
5. **跨项目移动不支持** (file 跨 project 是另一回事)

## 不要做

- ❌ 不预览直接 confirm
- ❌ random split 不固定 seed (不可复现)
- ❌ 跨项目 move (现在不支持)
- ❌ 用户描述含糊还往下走 (问清楚 source_wd / 是 move 还是 clone)
