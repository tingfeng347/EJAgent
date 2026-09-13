# 本地 Skills 配置

[English](README.md) · [模块索引](../README_zh-CN.md)

`SkillCatalog(skills_root)` 发现应用提供的指令文件，`SkillsContextPipeline` 将索引和
显式选中的内容注入 Actor 上下文。Skills 不会增加可执行工具或独立模型角色。

## 需要提供的文件

根目录必须存在。发现过程按排序后的直接子目录查找 `SKILL.md`：

```text
my-skills/
  review/
    SKILL.md
    template.md          # 可选
    examples/sample.md   # 可选
```

`SKILL.md` 示例：

```markdown
---
name: review
description: Review a proposed code change and report actionable findings.
---
Read the requested diff. Explain each finding with its affected behavior.
```

YAML frontmatter 可省略，`name` 默认目录名，`description` 默认空字符串。
名称重复或 frontmatter 不是映射时发现失败。选中 skill 时会一起注入上述可选模板和示例，
不会递归加载任意文件。

## 发现与选择

```python
from ejagent.context import SkillsContextPipeline


def skill_context(skills_root):
    return SkillsContextPipeline(skills_root, base=None)
```

将其作为 Harness `context`，启动时调用 `catalog.discover()`。`base=None` 默认直接投影；
组合压缩等行为时传入 base pipeline。直接使用者需在 `build()` 前启动，结束后关闭。

每个实例只发现一次，索引和已读取的 skill 内容会缓存。需要可靠加载文件变化时重新创建 catalog
或 pipeline。pipeline 注入可用 Skills 索引，并检查最新用户消息中的 `$review` 或
`skill:review`。只注入目录顺序中首个匹配项，不会加载所有被提及的 skill；
`skill:` 匹配不区分大小写，`$name` 匹配区分大小写。仅描述匹配不会自动选择并加载完整 skill。

选中后添加来源为 `skills:index`、`skills:<name>` 的 `TransientInstruction`。
这些是模型上下文，不是已提交会话消息。skill 所需的文件读取等工具由应用另行注册。
参见[上下文组合](../context/README_zh-CN.md)与[工具注册](../tools/README_zh-CN.md)。
