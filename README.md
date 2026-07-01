# 118 Skill Linker

`118-skill-linker` 是一个用于管理 Agent skills 的 Codex skill。它的核心思路是：**把 skills 原件集中放在一个中央目录里，每个项目只通过软链接引用自己需要的 skills**。

它适合同时使用 Codex、Claude Code 或其他 Agent 的用户，帮助你检查、克隆、链接、迁移、更新和固定 skills 版本，避免每个项目里散落多份重复 skill。

如果你已经确定了中央 skills 目录，可以用 `.skill-linker.json` 记住它。这样在新的对话或新的项目里，Agent 不需要每次重新猜测中央目录。

## 它能做什么

- 检查用户级和项目级 skills 目录。
- 读取项目级或用户级 `.skill-linker.json`，识别已经确认过的中央 skills 目录。
- 识别真实目录、软链接、失效软链接、重复 skill 名称，以及可能的中央 skills 仓库。
- 初始化项目级入口，例如 `.agents/skills`、`.codex/skills`、`.claude/skills`。
- 把中央目录里的一个或多个 skills 链接到当前项目。
- 从当前项目停用某个 skill，只删除入口链接，不删除中央原件。
- 在用户确认后，把可能的 skills 仓库克隆到中央目录父级。
- 把已经复制到全局或项目里的 skill 迁移到中央目录，并在原位置创建软链接。
- 检查 git 状态、发现可更新的下载仓库、更新 skill 仓库、切换到指定版本。
- 在你长期修改别人 skill 时，提示是否应该 fork 一份自己的仓库。

## 安全原则

这个 skill 默认很保守。

- 状态不明确时，先只读检查。
- 在把某个路径当作中央 skills 目录前，会先让用户确认。
- 写入类操作默认 dry-run。
- 不会用软链接覆盖已有真实目录。
- 用户说“删除 skill”时，默认只从当前项目停用，不删除真实 skill 目录，除非用户明确要求删除那个具体目录。
- 同步、迁移、克隆、git 更新和版本切换前，会先说明计划。

## 输出要求

在整理 skills 前，skill 应该明确区分：

- **用户级**：例如 `~/.codex/skills`、`~/.claude/skills`、`~/.agents/skills`
- **项目级**：例如当前项目里的 `.codex/skills`、`.claude/skills`、`.agents/skills`
- **中央目录**：用户确认后的 skills 原件目录

执行计划中应写清楚每一步的来源路径、目标路径、动作和风险，避免只写 `.codex/skills` 这种容易混淆的相对路径。

## 配置文件

配置文件名固定为 `.skill-linker.json`。

读取优先级：

1. 当前项目：`当前项目/.skill-linker.json`
2. 用户级：`~/.skill-linker.json`
3. 如果两者都不存在，再扫描候选目录并询问用户

推荐先写用户级配置：

```json
{
  "central_skills_dir": "/Users/you/.agents/skills",
  "default_mode": "centralize"
}
```

`default_mode` 可选值：

- `centralize`：默认把项目里的真实 skill 同步或迁移到中央目录，项目里只保留链接入口。
- `project-local`：允许当前项目 `.agents/skills` 作为项目内部中心目录。
- `ask`：每次列出选项，由用户决定。

查看当前生效配置：

```bash
python3 118-skill-linker/scripts/skill_manager.py config --project .
```

以 dry-run 方式写入用户级配置：

```bash
python3 118-skill-linker/scripts/skill_manager.py config --scope user --central ~/.agents/skills --mode centralize
```

用户确认后实际写入：

```bash
python3 118-skill-linker/scripts/skill_manager.py config --scope user --central ~/.agents/skills --mode centralize --execute
```

## 目录结构

```text
118-skill-linker/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── scripts/
    └── skill_manager.py
```

## 常用命令

检查当前项目：

```bash
python3 118-skill-linker/scripts/skill_manager.py inspect --project .
```

以 dry-run 方式初始化项目 skills 入口：

```bash
python3 118-skill-linker/scripts/skill_manager.py init --project . --agents claude,codex
```

以 dry-run 方式链接一个 skill：

```bash
python3 118-skill-linker/scripts/skill_manager.py link --project . --source ~/GitHub/my-skills/skills/write-blog
```

检查下载的 skill 仓库是否可能有更新：

```bash
python3 118-skill-linker/scripts/skill_manager.py updates --central ~/GitHub/my-skills/skills
```

以 dry-run 方式从当前项目停用某个 skill：

```bash
python3 118-skill-linker/scripts/skill_manager.py unlink --target .agents/skills/write-blog
```

用户确认后更新某个仓库：

```bash
python3 118-skill-linker/scripts/skill_manager.py update --repo ~/GitHub/my-skills --execute
```

## Windows 说明

这套管理方式在概念上是跨平台的，但 Windows 创建软链接时可能需要开启 Developer Mode，或者使用管理员权限终端。

Windows 上常见的链接方式：

- `mklink /D link target`：目录 symlink，最接近 macOS/Linux 的软链接。
- `mklink /J link target`：目录 junction，常用于目录，很多时候权限要求更友好。
- `.lnk` 快捷方式：不推荐用于 skills，因为 Agent 和脚本通常不会把它当真实目录读取。

脚本默认使用 `--link-type auto`。在 Windows 上，auto 模式会先尝试 symlink；如果失败，会尝试 junction。你也可以明确指定 junction：

```powershell
python 118-skill-linker\scripts\skill_manager.py link --project . --source C:\Users\you\GitHub\my-skills\skills\write-blog --link-type junction
python 118-skill-linker\scripts\skill_manager.py init --project . --agents claude,codex --link-type junction
```

AI Agent 可以帮你检测和解释权限问题，但不应该静默提权、绕过 UAC，或者输入管理员密码。

## 安装方式

项目级使用时，可以把这个目录复制或链接到项目的 skills 目录：

```bash
.agents/skills/118-skill-linker -> /path/to/118-skill-linker
```

如果 Codex 或 Claude Code 需要不同入口，可以在确认计划后，把对应入口链接到 `.agents/skills`：

```bash
.codex/skills  -> .agents/skills
.claude/skills -> .agents/skills
```
