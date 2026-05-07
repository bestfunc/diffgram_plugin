---
name: mark-rule-validation
display_name: AI 标记规则验证
description: 用户给一组规则后, AI 先**全规则 dry_run** + 给详细对比报告 + 标记冲突点, 让用户决策后再分步执行. 适合复杂规则集.
user-invocable: true
allowed-tools: mcp__diffgram__query_files,mcp__diffgram__query_files_count,mcp__diffgram__query_attributes_schema,mcp__diffgram__query_labels_schema,mcp__diffgram__mark_set_label,mcp__diffgram__mark_remove_label,mcp__diffgram__mark_set_attribute,mcp__diffgram__batch_pipeline
---

# AI 标记规则验证

跟 `bulk-mark-by-rule` 区别: 那个是**单规则**流程, 这个是**多规则集体验证** — 用户一次给多条规则, AI 找出规则间冲突 / 重复覆盖 / 漏覆盖, 给详细 diff 报告.

## 标准流程

### Step 1 — 用户给规则集

让用户用 yaml / 表格写出规则集:

```yaml
project: huakai_999
rules:
  - id: r1
    filter: "a.step=1 and a.channel_id=4 and !l.*"  # 没标过的
    action: mark_set_label
    label: NG_H
  - id: r2
    filter: "a.step=2 and a.channel_id=4 and l.OK"
    action: mark_remove_label  # 删 OK
    label: OK
  - id: r3
    filter: "a.step=2 and a.channel_id=4 and !l.*"
    action: mark_set_label
    label: NG_F
```

### Step 2 — 核对 schema

`query_attributes_schema` + `query_labels_schema` 确保:
- 所有 filter 用的 attribute key / value 存在
- 所有 action 用的 label 存在

### Step 3 — 各规则单独 count

```python
for rule in rules:
    rule.count = query_files_count(filter=rule.filter)
```

报告:
```
r1: 8430 files (mark NG_H)
r2: 1230 files (remove OK)
r3: 5670 files (mark NG_F)
```

### Step 4 — 规则冲突检测 (核心)

**重叠检测**: 找两两规则 filter 的交集:

```python
for ri, rj in combinations(rules, 2):
    overlap = query_files_count(filter=f"({ri.filter}) and ({rj.filter})")
    if overlap > 0:
        report_conflict(ri, rj, overlap)
```

报告:
```
⚠ r1 ∩ r2 = 0   OK 无重叠
⚠ r1 ∩ r3 = 0   OK 无重叠 (a.step=1 vs step=2 互斥)
⚠ r2 ∩ r3 = 230 file_ids  ← 这些 file 既匹配 r2 (有 OK) 又匹配 r3 (没 label)?
                          应该不可能, 检查 filter 逻辑
```

如果有重叠, 给用户标红 + 让用户决定:
- 改规则消除重叠
- 接受重叠 (后规则 overwrite 前规则)
- 取消其中一条

### Step 5 — 漏覆盖检测 (可选)

如果用户说"全部 step=2 channel=4 都得有标", 验证:
```
total = query_files_count(filter="a.step=2 and a.channel_id=4")
covered = query_files_count(filter="a.step=2 and a.channel_id=4 and (l.NG_H or l.NG_F or l.OK)")
uncovered = total - covered
if uncovered > 0:
    print(f"⚠ {uncovered} files 没被任何规则覆盖")
```

### Step 6 — 各规则抽样

每条规则跑 `query_files limit=3 with_attrs=true with_instances=true`, 给用户看每条规则命中的 3 个样例.

### Step 7 — 全规则 dry_run

每条规则跑 dry_run, 集中报告:
```
r1 dry_run: 8430 files → +NG_H instance (request_id=req-1)
r2 dry_run: 1230 files → -OK instance (request_id=req-2)
r3 dry_run: 5670 files → +NG_F instance (request_id=req-3)
全部预期总变更: +14100 instance creations, +1230 instance deletions
```

### Step 8 — 用户最终确认 + 分步执行

让用户**逐条**确认 (不是 batch 一次性, 因为可能改一两条):
> "r1 (8430 files mark NG_H) 执行吗? Y/N/skip"
> "r2 ... 执行吗?"
> "r3 ... 执行吗?"

每条 confirm 后调对应 mark tool 真执行, 报告完成. 如果用户中途反悔, 后续规则不执行.

### Step 9 — 总结报告

```
执行总结:
✓ r1: 8430 files marked NG_H (8200 new, 230 already had)
✓ r2: 1230 files OK label removed
✗ r3: 用户取消

回滚 (如需要):
- r1 回滚: mark_remove_label(file_ids=r1_files, label_name=NG_H, instance_ids=r1_instances)
- r2 回滚: mark_set_label(file_ids=r2_files, label_name=OK)
```

## 重点

1. **冲突检测是核心**, 多规则容易隐含重叠, 用户写规则时不容易看出
2. **漏覆盖可选** (如果用户期望全覆盖, 必查; 否则可跳)
3. **逐条确认而不是一键 batch** (规则错了至少不会全部执行错)
4. **保留 audit log + 回滚命令**, 即使错了也能撤销

## 典型场景

### 场景 A: 规则集 + 检查重叠

用户: "我要给 huakai_999 按这 5 条规则打标, 帮我看下规则间有没有冲突"

→ 你照 Step 1-9 跑, 重点 Step 4 冲突检测. 如果发现重叠, **不要往下推执行**, 让用户先解决冲突.

### 场景 B: 规则集 + 漏覆盖

用户: "我希望 step=2 channel=4 的全有 label, 这 3 条规则够吗?"

→ Step 5 漏覆盖检测, 给用户报 uncovered count + 提议补充规则.

## 不要做

- ❌ 不查冲突直接执行 (后规则可能 overwrite 前规则的标注)
- ❌ 一键 batch 执行所有规则 (中间错一条难回滚)
- ❌ 用户没确认就 confirm
- ❌ 用户中途取消还继续后续规则 (尊重用户决策)
