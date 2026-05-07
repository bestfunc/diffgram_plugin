---
name: pretrain-unsup-sync
display_name: 预训练 unsup 数据同步
description: 把 diffgram 项目的 unsup wav 同步到训练机. 支持分层抽样 (按 attribute) + 跳过 sup train 防 leakage. 接 TRAINV2_diffgram_pretrain_pull skill 的下游.
user-invocable: true
allowed-tools: mcp__diffgram__query_projects,mcp__diffgram__query_directories,mcp__diffgram__query_attributes_schema,mcp__diffgram__query_files,mcp__diffgram__query_files_count,mcp__diffgram__batch_pipeline
---

# 预训练 unsup 数据同步

v14Bv2 contrastive 预训练需要海量 unsup wav. 流程:
1. diffgram 上每个项目都有 `UNMARK` (无标注) + `TRAIN_*` (有标注) 等目录
2. unsup 池 = 全部项目 file − sup train 的 file (防 contrastive leakage)
3. 用 attribute 分层抽样保证 unsup 池 cover 各 channel / step / date 组合
4. 把抽样结果的 file_id 列表写入 csv → 下游 `TRAINV2_diffgram_pretrain_pull` skill 拉到训练机

这个 skill 帮做 1-3, 输出 file_id csv 给 4 用.

## 标准流程

### Step 1 — 用户给参数

让用户描述:
- **项目列表** (1 个或多个)
- **目标 unsup 总量** (e.g. 100k 条)
- **分层维度** (按哪些 attribute 分层抽样, e.g. channel_id × step)
- **要排除的 sup train file_id 列表** (避免 leakage)

### Step 2 — 列项目 + 看 UNMARK / TRAIN 目录

`query_directories(project)` 看每个项目有哪些目录:
- UNMARK / UNMARK_xxxx — 无标注池 (主要 unsup 来源)
- TRAIN_x.x — 有标注训练集 (排除)
- EVAL_TEST / EVAL_xxxx — 评估集 (排除)

### Step 3 — 核对 attribute schema (分层依据)

`query_attributes_schema(project)` 列项目 attribute key + 取值. 用户选哪些字段当分层依据.

### Step 4 — 各 stratum count

按 (project × attr1 × attr2) 笛卡尔展开各 stratum, 每个 stratum 跑 count:

```python
for project, channel_id, step in itertools.product(projects, [3, 4], [1, 2]):
    n = query_files_count({
      project,
      filter: f"a.channel_id={channel_id} and a.step={step} and !l.* and dataset = 'UNMARK'"
    })
    table.append((project, channel_id, step, n))
```

输出:

| project | channel | step | unsup_n |
|---|---|---|---|
| huakai_999 | 3 | 1 | 45230 |
| huakai_999 | 3 | 2 | 38420 |
| huakai_999 | 4 | 1 | 50100 |
| huakai_999 | 4 | 2 | 41250 |
| ... | ... | ... | ... |

### Step 5 — 分层抽样 (proportional to count, 或 equal)

用户决定:
- proportional (按 stratum 大小成比例) — 保留分布
- equal (每 stratum 抽相同数) — 平衡分布
- top-up (小 stratum 全取, 大 stratum 抽满) — 保证小类不被忽略

按抽样数, 每个 stratum 跑 `query_files`:
```python
files = query_files({
  project, filter, limit: n_sample, with_attrs: true
})
file_ids.extend([f["file_id"] for f in files["results"]])
```

### Step 6 — 防 leakage: 减去 sup train file_ids

```python
sup_train_ids = set(load_csv(用户给的 sup csv).file_id)
unsup_ids = [fid for fid in file_ids if fid not in sup_train_ids]
print(f"after leakage filter: {len(unsup_ids)}")
```

### Step 7 — 输出 csv

写出 csv (project / file_id / wav_path / attribute_groups), 让用户保存到本地, 然后用
`TRAINV2_diffgram_pretrain_pull` skill 在训练机上拉 wav.

```csv
project,file_id,filename,channel_id,step,date
huakai_999,12345,2790667_*.wav,3,1,2026-03-09
...
```

## 典型场景

### 场景 A: 单项目 unsup 抽样

用户: "huakai_999 的 UNMARK 目录里 NG_H 抽样, 每 channel 各 25k, 共 50k"

```
1. query_directories(huakai_999) → 确认 UNMARK 存在
2. query_attributes_schema(huakai_999) → 看 channel_id 取值
3. query_files_count(filter="a.channel_id=3 and dataset='UNMARK'") → 95430
4. query_files_count(filter="a.channel_id=4 and dataset='UNMARK'") → 99680
5. query_files(filter="a.channel_id=3 ...", limit=25000)  # channel 3 抽 25k
6. query_files(filter="a.channel_id=4 ...", limit=25000)  # channel 4 抽 25k
7. dedup against sup train ids
8. 输出 csv
```

### 场景 B: 多项目 unsup 分层抽样 100k 总量

用户: "huakai_999 + nsk- + bose, 各项目按 audio 比例分配, 保 channel × step 平衡"

```
1. 各项目 audio_count → 比例
2. 比例 × 100k → 各项目目标
3. 每项目内按 channel × step 4 stratum (or equal)
4. 各 stratum query_files 抽样
5. dedup
6. csv
```

## 重点

1. **dataset = 'UNMARK'** filter 限定无标注池 (排除 TRAIN/EVAL)
2. **`!l.*` 进一步保 unsup** (有任何 label 的不要)
3. **防 leakage**: 必拿用户提供的 sup train file_ids 列表去重
4. **分层维度选 channel_id + step + date 比较稳**
5. **csv 输出 + 转交下游 skill** (这个 skill 不直接拉 wav)

## 不要做

- ❌ 不去重直接下载 (会污染 contrastive 训练)
- ❌ 用户没指定分层维度直接 random sample (分布偏)
- ❌ 单 stratum 大于 50k 一次拉 (用 batch_pipeline 分页)
- ❌ 这个 skill 不调 `mark_*` / `move_*` (那是其它 skill 职责)
