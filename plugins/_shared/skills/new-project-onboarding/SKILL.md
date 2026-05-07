---
name: new-project-onboarding
display_name: 新项目接入 checklist
description: 新 diffgram 项目接入时, 帮检查 attribute 覆盖率 / label 一致性 / sup vs unsup 比例 / 空目录, 生成接入报告.
user-invocable: true
allowed-tools: mcp__diffgram__query_projects,mcp__diffgram__query_directories,mcp__diffgram__query_attributes_schema,mcp__diffgram__query_labels_schema,mcp__diffgram__query_files_count,mcp__diffgram__query_files,mcp__diffgram__query_instance_stats
---

# 新项目接入 checklist

新接入 diffgram 项目时, 用户常见痛点:
- attribute 标注不全 (有些 file attribute_groups 是空 / 部分缺)
- label 名跟其他项目不一致 (难合并训练)
- 目录结构混乱 (没分清 TRAIN/EVAL/UNMARK)
- sup / unsup 比例失衡 (sup 太少不够训, 或 unsup 太少没池)

这个 skill 帮一次性体检 + 给整改建议.

## 标准流程

### Step 1 — 用户给项目名

`query_projects()` → 确认项目可访问 + 拿 audio_count / role.

### Step 2 — 目录结构检查

`query_directories(project)` 列全部 wd:

```
TRAIN_5.0    (8200 files)
TRAIN_5.0_v2 (1230 files)
EVAL_TEST    (1130 files)
UNMARK_999   (185000 files)
test_temp    (45 files)         ← ⚠ 看着像临时, 待清理
```

报告:
- 目录命名是否清晰 (有没有 `tmp`, `test`, `bak` 这种)
- TRAIN / EVAL / UNMARK 是否齐全
- 空目录 / 极小目录 (< 50 files) 列表

### Step 3 — attribute 覆盖率

`query_attributes_schema(project)` 拿全部 attribute key + 用法 count.

对每个 key 跑:
```
total_files = query_files_count(project)
covered = query_files_count(project, filter=f"a.{key} != null")  # 或 "a.{key} = *"
coverage = covered / total_files
```

报告:
```
attribute 覆盖率:
- step:       coverage 99.8% (185000 / 185100)  ✓
- channel_id: coverage 95.2% (176000 / 185100)  ⚠ 9100 files 缺
- date:       coverage 100%                      ✓
- batch:      coverage 5.5%                      ⚠ 大部分 file 没 batch attr
```

≤ 90% 的 attribute 标⚠, 让用户决定:
- 这些 file 后续手动补 attribute
- 或者接受不全 (训练时这些 file 进不了 stratified sampling)

### Step 4 — label 体系检查

`query_labels_schema(project)` 拿 label 列表 + instance_count:

```
labels:
- OK:   instance=12000, file_count=11500, avg_segments=1.04
- NG_H: instance=2300,  file_count=1800,  avg_segments=1.28  ← 多段标注
- NG_F: instance=540,   file_count=320,   avg_segments=1.69  ← 多段标注更多
- NG:   instance=15,    file_count=15                        ← ⚠ 极少, 是不是误标? 跟 NG_H 重复?
- TEMP: instance=3,     file_count=3                         ← ⚠ 测试 label?
```

报告:
- 主 label (instance > 100) 列表
- 边角 label (instance < 50) 标⚠ — 让用户决定是否清理
- avg_segments > 1 的 label = 多段标注 (segment 级)
- 跟其他项目对比 (e.g. huakai 的 NG_H vs nsk 的 NG): 命名是否能统一

### Step 5 — sup vs unsup 比例

```
total = query_files_count(project)
sup   = query_files_count(project, filter="l.*")       # 有任意 label 的
unsup = total - sup
```

报告:
```
sup:   12450 (6.4%)
unsup: 182000 (93.6%)

sup 内分布:
- OK:    11500 file (52% of sup wav-level)   ← OK 类太多, 类不平衡
- NG_H:  1800
- NG_F:  320
不平衡 ratio: 11500 : (1800+320) = 5.4:1  ⚠ 训练时需类权重
```

判断:
- sup < 1000: 不够训, 建议补标
- sup 类不平衡 ratio > 10:1: 训练用 class weight 或 focal loss
- unsup > 100k: 够 contrastive 预训练
- unsup < 10k: 建议先 download 全量 raw wav 进库

### Step 6 — 跨项目语义对齐建议

如果用户说 "我打算把这个项目的数据合到 huakai 训练里", 调出 huakai 的 schema 跟新项目对比:

```
新项目 vs huakai_999:
- attribute step:        新项目用 step ✓
- attribute channel_id:  新项目用 channel_id ✓
- label OK:              新项目用 OK ✓
- label NG:              新项目用 NG  ⚠ huakai 用 NG_H 区分细类
建议:
- 统一 label: 把新项目 NG 拆成 NG_H / NG_F, 或 huakai 退回二类
```

### Step 7 — 输出 checklist 报告

给用户一份 markdown 体检报告:

```markdown
# huakai_ningbo 接入检查 (2026-05-07)

## 总览
- audio: 1,052,100 files
- 目录数: 12 (1 tmp 待清)
- attribute group: 4 (1 完整, 2 不全, 1 几乎空)
- label: 6 (3 主, 3 边角)
- sup: 8430 (0.8%)  ⚠ 太少
- unsup: 1,043,670 (99.2%)

## 阻塞项 (必修)
- [ ] 9100 files 缺 channel_id, 影响分层抽样 → 让标注员补
- [ ] 边角 label TEMP (3 files) 清理

## 警告 (建议)
- [ ] sup 不到 1% 太少, 建议补标到至少 5%
- [ ] OK 类占 sup 的 95%, 严重不平衡, 训练加权重

## 跨项目语义对齐
- 跟 huakai_999 对齐: ✓ 已对齐
- 跟 nsk- 对齐: ⚠ label NG_H 在 nsk 叫 NG, 需统一

## 整改顺序
1. 清空 tmp 目录
2. 补 channel_id 缺失
3. 边角 label 清理
4. 决定 sup 补标策略
```

## 不要做

- ❌ 不直接修改项目数据 (只做检查 + 报告), 整改需用户走 `bulk-mark-by-rule` 等
- ❌ 假设 attribute 全覆盖 (常常不全, 必查)
- ❌ 跳过跨项目对比 (新项目接入价值在能合并, 必须对齐)
- ❌ 报告写太长用户看不下 (优先给阻塞项 + 警告 + 整改顺序, 详情放后面)
