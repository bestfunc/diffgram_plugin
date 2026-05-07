---
name: multi-project-merge-preview
display_name: 跨项目数据合并预览
description: AI 工程师准备从多项目挑 sup 训练数据时, 帮做 attribute 标准化建议 + label 一致性检查 + 预算总量.
user-invocable: true
allowed-tools: mcp__diffgram__query_projects,mcp__diffgram__query_attributes_schema,mcp__diffgram__query_labels_schema,mcp__diffgram__query_files_count,mcp__diffgram__query_files,mcp__diffgram__query_instance_stats
---

# 跨项目数据合并预览

AI 工程师准备从 N 个项目挑 sup 训练数据时, 主要痛点:
1. 各项目 attribute schema 不一致 (huakai 用 `step`, brose 用 `Stage`)
2. label 名不一致 (NG_H vs LABEL_NG vs 异常)
3. 不知道总数 + 类别分布

这个 skill 帮做这些前置检查, 输出**合并方案 markdown** 给用户做决策, 不直接执行.

## 标准流程

### Step 1 — 用户给项目列表 + 目标用途

让用户说清楚:
- 要合并哪几个项目
- 目标用途 (e.g. "训练 v14B FT 二分类 OK / NG", "训练 v20Av3 三分类 OK / OK000 / NG_H")
- 是否需要 stratified (按 label / attribute 分层抽样)

### Step 2 — 拉每个项目的 schema

```python
for project in projects:
    schema = query_attributes_schema(project)
    labels = query_labels_schema(project)
```

输出对比表:

| 项目 | attribute keys | label names |
|---|---|---|
| huakai_999 | step, channel_id, date, batch | OK, OK000, NG_H |
| nsk- | step, channel_id, date | OK, NG |
| bose_changchun | Stage, Channel, Date | LABEL_OK, LABEL_NG |

### Step 3 — 标记差异点

自动识别:
- **attribute 命名差异**: huakai `step` ≈ bose `Stage` ≈ nsk `step` (人工核对)
- **attribute 取值差异**: huakai channel_id ∈ {3, 4}, bose Channel ∈ {A, B} (要建映射)
- **label 体系差异**:
  - huakai 三类 (OK, OK000, NG_H)
  - nsk 二类 (OK, NG)
  - bose 二类 (LABEL_OK, LABEL_NG, 名字不同)

给用户预览差异 + 提议标准化映射:

```yaml
# 标准化建议
attribute_mapping:
  step:
    huakai_999.step: as-is
    nsk-.step: as-is
    bose_changchun.Stage: rename → step

  channel_id:
    huakai_999.channel_id: as-is (3, 4)
    bose_changchun.Channel: as-is, value_map: {A: 3, B: 4}

label_mapping:
  OK:
    huakai_999.OK: as-is
    nsk-.OK: as-is
    bose_changchun.LABEL_OK: rename → OK
  NG:
    huakai_999.NG_H: rename → NG (合并 OK000 进 OK?)
    nsk-.NG: as-is
    bose_changchun.LABEL_NG: rename → NG
```

### Step 4 — 各项目分别 count

```python
for project in projects:
    total = query_files_count(project)
    sup_NG = query_files_count(project, filter="l.NG_H or l.NG or l.LABEL_NG")  # 跨命名 or
    sup_OK = query_files_count(project, filter="l.OK or l.LABEL_OK")
```

输出:

```
huakai_999:  audio=195098, sup=8986 (OK 7833, NG_H 1153)
nsk-:         audio=66087,  sup=1167 (OK 800, NG 367)
bose_chchun:  audio=36310,  sup=2230 (LABEL_OK 1800, LABEL_NG 430)
合计 sup:     12383 (OK 10433, NG 1950) → 5.4:1 不平衡
```

### Step 5 — 类别分布 + 维度分布

```python
query_instance_stats(
  project_list=projects,
  filter="l.NG_H or l.NG or l.LABEL_NG",
  group_by="attr.channel_id"  # 也可 group_by="project"
)
```

输出 NG 在 channel / project / step 各维度的分布表.

### Step 6 — 输出合并方案 markdown

把上面 attribute_mapping + label_mapping + 数量统计 + 分布写成一份方案给用户.

```markdown
## 合并方案: huakai+nsk+bose

### 标准化映射
[attribute_mapping yaml]
[label_mapping yaml]

### 总量
- audio: 297k
- sup: 12.4k (OK 10.4k, NG 1.9k)

### 数据分布 (NG)
| 维度 | huakai | nsk | bose |
|---|---|---|---|
| channel=3/A | 600 | 200 | 220 |
| channel=4/B | 553 | 167 | 210 |

### 风险点
- LABEL_NG 在 bose 里只 430 条, 比 huakai NG_H 少很多, 跨项目训练时 bose 容易 underfit
- huakai OK000 子类未合并, 决策: 合进 OK / 单独保留 / 当 NG?

### 建议下一步
方案 1: 按上面映射执行 (建议先建 SFT csv 不动 diffgram)
方案 2: 分项目训 + ensemble (避免标签语义冲突)
方案 3: 仅 huakai 单独训 (数据量够)
```

让用户选方案, 不要 AI 替用户决定.

## 重点

1. **不直接修改数据**, 只做预览 + 建议
2. **跨项目 attribute / label 命名差异是常见 bug 源**, 必要先核对
3. **count + 抽样 + 分布**, 三件套用户才能做合理决策
4. **类别不平衡时显式提醒**, 影响训练策略
5. **提议的合并不是唯一方案**, 给用户 2-3 个选项 + 各自 trade-off

## 不要做

- ❌ 不核对 schema 直接合并 (跨项目后 filter 全错)
- ❌ 假设各项目 label 体系一致 (常常不一致)
- ❌ 直接 mark / move 修改 diffgram 数据 (不是这个 skill 的职责, 转 `bulk-mark-by-rule`)
- ❌ 替用户决定标签合并 (语义合并是业务决策, 让用户决定)
