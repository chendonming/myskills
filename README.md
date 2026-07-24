# Skills

个人 Skills 集合，用于 [Reasonix](https://reasonix.ai) 等 AI 编码代理的技能库。

## 技能列表

| 名称 | 描述 |
|------|------|
| [git-commit](./skills/git-commit/SKILL.md) | 按照约定式提交规范自动提交 Git 代码并以 rebase 方式推送，无需用户确认 |
| [technical-decision-document-generator](./skills/technical-decision-document-generator/SKILL.md) | 生成可审计的技术决策文档，涵盖架构决策、工程权衡、实现选择和技术调研 |

## 使用方法

### Reasonix

本技能目录已注册到 `~/.reasonix/config.toml` 的 `[skills].paths` 中：

```toml
[skills]
paths = ["/path/to/myskills/skills"]
```

Reasonix 启动时会自动发现并加载这些技能，可以通过 `/skill-name` 或 `run_skill` 调用。

### 其他 AI 代理

将本目录的路径添加到对应代理的技能发现路径中即可使用。

## 添加新技能

在 `skills/` 下新建文件夹并创建 `SKILL.md`，格式示例：

```yaml
---
name: skill-name
description: 简短描述（≤120 字符）
---
## 功能

技能具体做什么。

## 使用方式

如何调用。
```

## 项目结构

```
.
├── README.md
└── skills/
    ├── git-commit/
    │   └── SKILL.md
    └── technical-decision-document-generator/
        └── SKILL.md
```
