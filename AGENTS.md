# myskills — Reasonix 个人技能集合

个人 Skills 集合，用于 [Reasonix](https://reasonix.ai) 等 AI 编码代理的技能库。每个技能是一个独立文件夹下的 `SKILL.md`，通过 YAML frontmatter 声明元信息。

## 项目

- **语言/工具**: Reasonix skill 规范 (YAML frontmatter + Markdown)
- **入口**: `skills/<skill-name>/SKILL.md`
- **无构建/测试/运行脚本** — 纯文档型项目

## 目录结构

```
.
├── AGENTS.md
├── README.md
├── skills/                  # 用户自制的技能（主要目录）
│   ├── git-commit/
│   │   └── SKILL.md
│   └── technical-decision-document-generator/
│       └── SKILL.md
└── .reasonix/skills/        # 内置技能（不要手动修改）
    └── skill-creator/
        ├── SKILL.md
        ├── agents/          # 子代理提示
        ├── scripts/         # Python 评测脚本
        ├── eval-viewer/     # 评测结果查看器
        └── assets/          # 评测模板
```

## 命令

该项目不涉及构建/测试/运行。相关操作仅限 Reasonix 技能调用：

| 操作 | 命令 |
|------|------|
| 列出可用技能 | 通过 Reasonix 技能索引查看 |
| 调用技能 | `/skill-name` 或 `run_skill({name: "..."})` |

## 架构

- **`skills/`** — 用户自主编写的技能，每个子目录是一个独立技能包
- **`.reasonix/skills/skill-creator/`** — 内置技能创作工具（含评测框架和脚本），不应手动修改
- **新技能流程**: skill-creator 创建 → 将生成的 `SKILL.md` 放入 `skills/<name>/` → 注册路径后生效

## 约定

### 技能格式
- 每个技能放在 `skills/<kebab-case-name>/SKILL.md`
- 必须包含 YAML frontmatter:
  ```yaml
  ---
  name: skill-name
  description: ≤120 字符描述
  ---
  ```
- 名称使用 kebab-case（如 `git-commit`、`technical-decision-document-generator`）
- 描述用中文或英文均可，需准确反映技能用途

### 技能创建/安装
- **通过 skill-creator 创建的新技能必须放到 `skills/` 目录下，不要安装到全局（global scope）**
- 技能安装模式用 `mode: auto`（自动复制）或 `mode: copy`，不要用 `link`（symlink 在技能多目录场景不稳定）
- 如需注册外部技能库（如 NPM 包中的技能），用 `register` 模式添加路径到 `[skills].paths`，不要复制文件

## 注释

（留空，后续可快速补充项目相关的备注。）
