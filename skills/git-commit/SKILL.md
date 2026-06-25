---
name: git-commit
description: >
  按照约定式提交（Conventional Commits）规范自动提交Git代码。
disable-model-invocation: true
---

# git-commit 技能

## 概述

按照 [Conventional Commits v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) 规范自动提交 Git 代码。

AI 自行分析变更内容、生成规范的提交信息，用户只需确认即可。

此技能仅在用户主动输入 `/commit` 时触发，不会自动调用。

## 约定式提交规范摘要

### 提交格式

```
type(scope): description

[optional body]

[optional footer(s)]
```

### 类型（type）

| 类型 | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 bug |
| `build` | 构建系统或外部依赖变更 |
| `chore` | 杂项（构建过程或辅助工具的变动） |
| `ci` | CI 配置变更 |
| `docs` | 仅文档变更 |
| `style` | 不影响代码含义的格式变更 |
| `refactor` | 重构 |
| `perf` | 性能优化 |
| `test` | 添加或修正测试 |
| `revert` | 回退提交 |

### scope（可选）

影响范围的简短描述，如 `(auth)`、`(api)`、`(ui)`。

### description（必填）

简洁的变更描述，祈使句、小写开头、末尾不加句号。从变更内容中归纳总结。

### body（可选）

当变更需要额外背景说明时补充。每行不超过 72 字符。

### footer（可选）

- 关联 Issue：`Closes #123`
- BREAKING CHANGE：`feat(api)!: ...` 或 footer 中写 `BREAKING CHANGE: ...`

### 示例

```
feat(auth): add JWT-based authentication

Implement JWT token generation and validation with refresh token support.
Closes #234
```

```
fix(api): handle null response in user lookup
```

```
refactor(core): extract validation logic into separate module
```

```
docs(readme): update installation instructions
```

## 工作流程

### Step 1: 检查 Git 状态

执行 `git status` 和 `git diff --cached --stat` 查看暂存区。

**情况 A — 暂存区有变更**：进入 Step 2。

**情况 B — 工作区有变更但未暂存**：自动执行 `git add` 暂存所有变更（或询问用户是否暂存所有），然后进入 Step 2。如果用户明确表示只提交部分文件，按用户要求 add 指定文件。

**情况 C — 无任何变更**：告知用户当前没有变更需要提交，结束流程。

### Step 2: 分析变更并生成提交信息

分析暂存区的 diff 内容，理解变更的实质。按以下逻辑自动生成提交信息：

1. **判断 type**：根据变更内容自动判断最合适的类型
   - 新增功能/接口 → `feat`
   - 修复问题 → `fix`
   - 代码结构调整但不改功能 → `refactor`
   - 仅文档 → `docs`
   - 性能相关 → `perf`
   - 测试相关 → `test`
   - CI 配置 → `ci`
   - 依赖/构建 → `build`
   - 格式化 → `style`
   - 杂项 → `chore`

2. **判断 scope（可选）**：如果变更集中在某个特定模块（如 auth、api、ui、core），自动提取作为 scope

3. **生成 description**：通览 diff 后归纳总结。如果变更涉及多个维度，以主要变动为准；如果范围刚好涵盖一个完整功能，可以用更抽象的表述。

4. **生成 body（可选）**：当 diff 包含值得说明的技术决策、动机或影响范围时，补充简要 body

5. **生成 footer（可选）**：检测到 BREAKING CHANGE 时自动添加 `BREAKING CHANGE:` footer

### Step 3: 展示并确认

调用 `AskUserQuestion` 工具展示提交预览，询问用户是否确认。在 `question` 字段中嵌入预览内容和暂存文件列表。提供两个选项：

1. **确认提交** → 执行 `git commit`
2. **修改提交信息** → 用户可手动输入修改意见（如"把 type 改成 refactor"、"加个 scope"等），按反馈调整后重新调用 `AskUserQuestion` 再次预览

格式参考：

```
┌─ 提交预览 ─────────────────────────────────────
│ feat(auth): add JWT-based authentication
│
| Implement login and refresh token flow.
│ ───────────────────────────────────────────────
│ 暂存文件:
│   M  src/auth/login.ts
│   A  src/auth/jwt.ts
└────────────────────────────────────────────────
```

### Step 4: 提交

提交完成后展示提交摘要（commit hash、变更摘要）。**不执行 `git push`**。

## 关键提醒

- **AI 自主分析**：AI 完全自主分析 diff 内容并生成提交信息，不需要向用户提问"type 选什么？"、"description 怎么写？"这类问题
- **让用户瞟一眼**：提交前展示预览让用户确认，避免意外提交
- **用户不满意可调整**：如果生成的信息不准确，根据用户反馈修正后重新预览
- **只 commit 不 push**：只执行 `git commit`，不要执行 `git push`
- **不要用 `git commit -a`**：只提交暂存区内容（或用户明确指定的文件）
- **多行提交信息**：body 有多行时使用 heredoc 方式提交
