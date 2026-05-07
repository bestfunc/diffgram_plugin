---
name: query-syntax
display_name: Diffgram 复合查询语法
description: 跨项目查询 file / instance 时, 复合 grammar `a.xxx and l.xxx and instance.xxx` 完整语法 + 常见用例
user-invocable: false
---

# Diffgram 复合查询语法 (a.xxx / l.xxx / instance.xxx)

跨项目数据探查的核心 — `query_files` / `query_files_count` 两个 tool 的 `filter` 字段都用此语法.

## EBNF

```
filter      ::= expr
expr        ::= term (OR term)*
term        ::= factor (AND factor)*
factor      ::= "(" expr ")"
              | "!" factor                  # NOT
              | compare_expr
              | bare_label
compare_expr::= NAME COMPARE_OP VALUE
NAME        ::= entity "." field
entity      ::= "a" | "l" | "instance" | "file" | "dataset"
COMPARE_OP  ::= "=" | "!=" | ">" | "<" | ">=" | "<=" | "in" | "like"
VALUE       ::= NUMBER | STRING | array
array       ::= "[" VALUE ("," VALUE)* "]"
bare_label  ::= "l." LABEL_NAME             # 简写: 等价 l.<name> = true
```

## entity 含义

| entity | 字段来源 | 例 |
|---|---|---|
| `a.<name>` | file 的 `attribute_groups` JSONB (项目级 attribute_template_group 字段) | `a.step = 1`, `a.channel_id = 4`, `a.date = "2026-03-09"` |
| `l.<name>` | file 含某 label 的 instance (whole-file 或 segment) | `l.NG_H` (含 NG_H 的 instance), `!l.OK` (没有 OK label) |
| `instance.<field>` | instance 表字段 (start_time / end_time / soft_delete) | `instance.start_time > 0.5`, `instance.end_time - instance.start_time < 1.0` |
| `file.<field>` | file 表字段 (filename / type / state) | `file.original_filename like "%huakai%"`, `file.state = "available"` |
| `dataset.<name>` | working_dir nickname (项目内目录) | `dataset.TRAIN_5_0`, `dataset = "EVAL_TEST"` |

## 操作符

| op | 含义 | 适用 |
|---|---|---|
| `=` | 等于 | 全部 |
| `!=` | 不等于 | 全部 |
| `>`, `<`, `>=`, `<=` | 大小比较 | 数值 / 时间字段 |
| `in [a, b, c]` | 在集合 | 全部 |
| `like "%pattern%"` | SQL LIKE | 字符串字段 |
| `!` (NOT 前缀) | 否定 | 整个 factor |
| `and` / `or` | 逻辑 | 任意复合 |

## 常见用例

```
# 单条件
a.step = 1
l.NG_H
file.original_filename like "%2026-03%"

# 双 attribute
a.step = 1 and a.channel_id = 4

# attribute + label
a.step = 1 and a.channel_id = 4 and l.NG_H

# attribute + 没标 (negative filter)
a.step = 1 and !l.NG_H

# 多 label OR
l.NG_H or l.NG_F

# 数组 + 范围
a.step in [1, 2] and a.channel_id >= 3

# segment 时长筛选
l.NG_H and instance.end_time - instance.start_time > 0.5

# 目录 + 条件 (跨 wd 筛选)
dataset = "TRAIN_5_0" and a.step = 1 and !l.NG_H

# 复合括号
(a.step = 1 or a.step = 2) and l.NG_H

# 文件名模糊匹配
file.original_filename like "%channel_id_4%" and l.NG_H
```

## 注意事项 (容易踩坑)

1. **attribute 字段名要跟项目 attribute_template_group 一致**
   - 调 `query_attributes_schema(project)` 看项目实际有哪些 attribute key + 每 key 的取值范围
   - 不同项目可能 attribute 命名不一致 (huakai_999 用 `step`, 别的项目可能用 `Stage`)
   - 跨项目查询前先各自核对

2. **`a.date` 字段的值是字符串不是时间戳**
   - diffgram attribute 的 value 都按 select 选项存, 字符串比较
   - `a.date = "2026-03-09"` 行, `a.date > "2026-03-01"` 行 (字符串比较, ISO 8601 格式可)

3. **`l.<name>` 要求 instance 上 label 不是 file 上**
   - whole-file label 也是一条 instance (覆盖整个 wav)
   - segment label 是带 start_time/end_time 的 instance

4. **`!l.X` 语义**
   - 整个 file 都没有 X label 的 instance, 才匹配
   - 注意 file 可能有多条 instance, 只要任一 instance 是 X 都算"有 X"

5. **跨项目时 attribute / label 命名差异**
   - 同一字段在不同项目可能不同名 (huakai 用 NG_H, 客户方 brose 可能用 LABEL_NG)
   - 跨项目复合查询前先确认或用 `or` 兜两边

6. **性能**
   - `like "%pattern%"` 走全表扫, 慎用; 优先用 attribute / label 这种走索引
   - 复合 and 比 or 快 (and 早期剪枝)
   - 跨项目 + 海量 file 时建议先 `query_files_count` 估一下

## 一行验证

```
query_files_count({project: "huakai_999", filter: "a.step = 1 and l.NG_H"})
```

返回数字, 用来 sanity check 你的 filter 语法对不对.
