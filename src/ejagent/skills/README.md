# Local skill configuration

[中文](README_zh-CN.md) · [Module index](../README.md)

`SkillCatalog(skills_root)` discovers application-supplied instruction files.
`SkillsContextPipeline` injects their index and explicitly selected content into
Actor context. Skills do not add executable tools or a separate model role.

## Files to provide

The root must exist. Discovery scans its immediate child directories, in sorted
order, for `SKILL.md`:

```text
my-skills/
  review/
    SKILL.md
    template.md          # optional
    examples/sample.md   # optional
```

An example `SKILL.md`:

```markdown
---
name: review
description: Review a proposed code change and report actionable findings.
---
Read the requested diff. Explain each finding with its affected behavior.
```

YAML frontmatter is optional. `name` defaults to the directory name and
`description` to an empty string. Duplicate names or non-mapping frontmatter fail
discovery. The recognized optional template and sample are included with a selected
skill; arbitrary files are not recursively loaded.

## Discovery and selection

```python
from ejagent.context import SkillsContextPipeline


def skill_context(skills_root):
    return SkillsContextPipeline(skills_root, base=None)
```

Supply this as Harness `context`; startup calls `catalog.discover()`. `base=None`
uses identity projection; provide a base pipeline to combine compaction or other
behavior. Direct users must start the pipeline before `build()` and shut it down
afterward.

The catalog is discovered once per instance; the index and loaded skill content
are cached. Recreate the catalog/pipeline to pick up changed files reliably.
The pipeline includes an available-skills index and examines the latest user
message for `$review` or `skill:review`. Only the first matching catalog entry is
injected, not every mentioned skill; `skill:` matching is case-insensitive, while
`$name` matching is case-sensitive. Descriptions alone do not automatically select
and load a skill.

Selection adds `TransientInstruction` values with sources `skills:index` and
`skills:<name>`. They are model context, not committed conversation messages.
The application must separately register any file-reading or other tools needed
by a skill. See [context composition](../context/README.md) and
[tool registration](../tools/README.md).
