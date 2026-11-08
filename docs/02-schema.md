# 02 · wiki_schema.md 编译契约

## 为什么需要 Schema 契约

LLM 编译是非确定性的。同一批源文件，今天编译和明天编译可能输出不同结构。要让**多批次编译行为一致**，必须把页面模板、聚合规则、链接规则、锚定规则写成一份契约文档，**每次调用 LLM 时全文注入提示词**。

## Schema 五大块

### 1. 页面类型定义（五类）

| 类型 | 目录 | 模板字段 |
|---|---|---|
| entity | `wiki/entity/` | 标题 → 别名 → 类型 → 来源列表 → 摘要 → 关键事实（聚合）→ 关联页面 → 原文引用 |
| concept | `wiki/concept/` | 标题 → 主题标签 → 来源列表 → 摘要 → 概念描述（聚合）→ 关联页面 → 原文引用 |
| comparison | `wiki/comparison/` | 标题 → 来源列表 → 对比维度表 → 综合判断 → 关联页面 → 原文引用 |
| synthesis | `wiki/synthesis/` | 标题 → 涉及实体/概念 → 来源列表 → 跨源综合 → 策略建议 → 关联页面 → 原文引用 |
| summary | `wiki/` 根目录 | 每源一篇的摘要草稿 |

### 2. 聚合规则

- 品牌 + 业务概念 + 方法论 + 行业趋势**全量聚合**
- **同一实体/概念多源 → 合并一页**，来源列表列全部源
- 聚合页「摘要 / 关键事实」为多源综合描述，逐条标注来源

### 3. 双向链接规则

- entity ↔ concept ↔ synthesis 层级互链
- comparison 挂接两端实体页
- 每页「关联页面」段必须**对称回链**（A 链 B，B 必须链回 A）
- 链接使用**相对路径**（`../entity/xxx.md`），不用 wiki-link 语法（`[[...]]`），保证可做文件级对称校验

### 4. 编译流程

每次 ingest 固定五步：summary 草稿 → 概念聚合 → 跨页联动 → 补 index/log → lint 巡检。

### 5. 锚定规则

每页必须带「来源路径 + 原文引用」，供 lint 校验：引用必须能回溯到 raw 层真实存在的文件。

## 注入方式

```python
SCHEMA = os.path.join(KB, "_meta", "wiki_schema.md")

def load_schema():
    """读取 wiki_schema.md 全文，供注入 LLM 提示词"""
    try:
        with open(SCHEMA, encoding="utf-8") as f:
            return f.read()
    except FileNotFoundError:
        return "（未找到 wiki_schema.md，请按默认 LLM Wiki 规范输出）"
```

每次 LLM 编译调用，`load_schema()` 的返回值拼接进 system prompt。代码显式读取，不依赖模型记忆。

## 契约版本管理

- Schema 文档带版本号（v1.0 / v2.0）与终审状态
- 结构升级 = 推倒重建 + 新 Schema 版本，旧快照保留 30 天