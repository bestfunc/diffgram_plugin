---
name: bulk-mark-by-rule
display_name: 规则驱动批量打标
description: 用户给"把符合 X 条件的 wav 标 Y"规则后, AI 自动 dry_run → 给用户预览 + 样本 → 用户确认 → 真执行. 强制安全流程.
user-invocable: true
allowed-tools: mcp__diffgram__query_files_count,mcp__diffgram__query_files,mcp__diffgram__query_attributes_schema,mcp__diffgram__query_labels_schema,mcp__diffgram__mark_set_label,mcp__diffgram__mark_remove_label,mcp__diffgram__mark_set_attribute
---

# 规则驱动批量打标

你将帮用户用一句话规则给海量 wav 打标 / 改 attribute. **关键: 写操作不可逆, 必须 dry_run + 抽样验证 + 用户最终确认 3 道防线.**

## 标准流程 (一字不差按这个跑)

### Step 1 — 解析用户规则

让用户描述清楚:
- **项目** (1 个 project_string_id, 不支持跨项目, mark 是单项目操作)
- **筛选条件** (复合 filter, 见 `query-syntax` skill)
- **目标动作** (label 加/移除 / attribute 设/清)

把规则口头改写一次让用户确认:
> "你的意思是: 在 **huakai_999** 项目里, 把所有 **a.step=1 且 a.channel_id=4 但还没标 NG_H** 的 wav, 给打 **NG_H** label, 是吗?"

### Step 2 — 核对 schema (必做)

`query_attributes_schema(project)` + `query_labels_schema(project)`:
- 确认 attribute key 存在 + 取值合法
- 确认 label name 存在 (打不存在的 label 会拒)

### Step 3 — 估总量 (count)

```
query_files_count({project, filter})
→ {count: 8430}
```

如果 0 → schema 错或 filter 错, 回 Step 2.
如果 > 100000 → 让用户考虑要不要拆批 (单次 mark 上限 100k).

### Step 4 — 抽样验证 (5 条) 给用户看

```
query_files({project, filter, limit: 5, with_attrs: true, with_instances: true})
```

打印 5 条样本 file:
```
file_id=12345 | filename=2790667_*.wav | wd=[TRAIN_1.2] | attrs={step:1,channel:4} | instances=[OK 0-6.0s]
file_id=12346 | ...
```

让用户看 — 这 5 条是不是符合预期? 要打 NG_H 是不是真该打 NG_H?

### Step 5 — dry_run 拿 request_id

```
mark_set_label({
  project: "huakai_999",
  file_ids: <Step 4 query 出的全部 file_id, 或者重新 query limit=10000>,
  label_name: "NG_H",
  dry_run: true
})
→ {
  request_id: "req-abc123",
  preview: "8430 files will be marked NG_H. Of which: 230 already have (skip), 8200 new instances will be created.",
  affected_count: 8200,
  expires_at: "5min later"
}
```

把 preview 给用户看.

### Step 6 — 用户最终确认

> "我即将给 8200 个 wav 添加 NG_H label (230 个已有, 跳过). 此操作 5 分钟内有效, 请确认是否执行: [是 / 否 / 修改条件]"

只有用户**明确说"是"** (或同等确认) 才能进 Step 7. 用户回避 / 含糊不算确认.

### Step 7 — confirm 真执行

```
mark_set_label({
  request_id: "req-abc123",
  confirm: true
})
→ {instance_ids: [...], succeeded: 8200, failed: 0}
```

### Step 8 — 报告

```
✓ 完成: 8200 个 instance 创建, 230 个跳过 (已有 NG_H), 0 个失败
ckpt 文件: file_ids 列表已保存到 audit_log (request_id=req-abc123)
回滚: mark_remove_label(file_ids=..., label_name=NG_H, instance_ids=above_list) 撤销
```

## 典型对话示例

**用户**: 把 huakai_999 里 step=2 channel=4 但没标 NG_H 的, 全部标 NG_H

**你**:

1. 改写规则确认: "你想给 huakai_999 里 a.step=2 && a.channel_id=4 && !l.NG_H 的 wav 打 NG_H, 对吗?" — 用户: 对
2. `query_attributes_schema(huakai_999)` 确认 step / channel_id 存在 ✓
3. `query_labels_schema(huakai_999)` 确认 NG_H 存在 ✓
4. `query_files_count({project: "huakai_999", filter: "a.step=2 and a.channel_id=4 and !l.NG_H"})` → 8430
5. `query_files({..., limit: 5, with_attrs: true, with_instances: true})` 显示 5 条样本
6. 抽样看着对 → `mark_set_label({project, file_ids: [全部], label_name: "NG_H", dry_run: true})`
   → request_id, preview "8430 files, 0 already, 8430 new"
7. **用户确认** "是"
8. `mark_set_label({request_id, confirm: true})` → 报告完成

## 安全约束 (绝对不可省)

1. **必跑 dry_run**, 即使用户说"直接做不要确认"也要跑 — 安全防线
2. **必抽样验证 5 条**, 给用户看 schema 命中是否对
3. **必让用户口头确认**, 不能在 dry_run 之后自动 confirm
4. **file_ids 单次 ≤ 100,000**, 超分批
5. **如有失败, 报告未完成的 file_ids**, 让用户重试
6. **request_id 只有 5 分钟有效期**, 超过失效就重新 dry_run

## 不要做

- ❌ 不抽样直接 dry_run + confirm
- ❌ 用户没明确 "是" 就 confirm
- ❌ 跨项目 mark (mark 是单项目, 跨项目用 batch_pipeline 串多次)
- ❌ 规则本身有歧义还往下走 (e.g. "标 NG" — 要 NG_H 还是 NG_F? 问清楚)
- ❌ 给用户报 count = 0 时直接执行 (大概率 filter 错, 回核对 schema)

## 进阶: attribute 批量设值

把 `mark_set_label` 换成 `mark_set_attribute`, 流程一样:

```
mark_set_attribute({
  project: "huakai_999",
  file_ids: [...],
  attrs: {date: "2026-05-01", batch: "v3"},
  dry_run: true
})
```

- attrs 是 kv map
- 如 file 已有该 key 值, 会 overwrite (给用户提醒)
- attribute 必须在项目 attribute_template 里, 不存在的 key 会拒
