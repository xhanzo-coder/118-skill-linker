---
name: 118-skill-linker
description: 使用中央 skills 目录、.skill-linker.json 配置和项目级软链接管理 Agent skills。当用户需要检查、初始化、下载/克隆、链接、批量链接、取消链接、停用、删除入口、同步、迁移、更新、切换版本、去重或排查 Codex、Claude Code、.agents、.codex、.claude、用户级 skills 目录或项目级 skills 目录时使用。也适用于处理失效软链接、中央 skills 仓库、多 Agent 共享 skills、检查下载的 skills 是否有更新、通过 git 更新 skills、判断是否应该 fork 别人的 skill，以及把复制出来的 skills 安全迁移为软链接。
---

# 118 Skill Linker

## 概览

使用这个 skill，把中央来源中的 Agent skills 链接到项目级 skills 目录中。默认只做只读检查；在任何写入、同步、替换、删除或 git 更新前，先解释计划并获得用户确认。

如果已经存在 `.skill-linker.json` 配置，优先使用配置中的中央 skills 目录，不要在新对话里重新把当前项目目录猜成中央目录。

## 安全规则

- 除非用户明确要求某个安全的写入操作，否则先从只读检查开始。
- 在把某个目录视为中央 skills 目录前，必须先让用户确认。
- 在同步或迁移前，列出所有计划创建、替换、跳过、备份、删除或 git 操作，并等待用户确认。
- 不要用软链接覆盖已有的真实目录。
- 用户说“删除 skill”时，默认理解为“从当前项目停用”，只删除入口链接；不要删除真实 skill 目录，除非用户明确要求删除那个具体目录。
- 移除项目 skill 时，先确认目标是软链接，再只删除软链接本身。
- 通过软链接编辑文件前要提醒用户：这会修改中央原件，并影响所有指向它的项目。
- 对共享或团队项目，要提醒用户：`/Users/name/...` 这类绝对软链接在别人电脑上可能变成失效链接。
- 在 Windows 上创建 symlink 可能需要 Developer Mode 或管理员终端；不要让 Agent 静默提权、绕过 UAC 或输入管理员密码。

## 工作流程

1. 判断用户请求属于检查、初始化、下载/克隆、链接、批量链接、取消链接/停用、同步、迁移、更新、切换版本、fork 建议或解释。
2. 在状态未知时先执行检查，并读取 `.skill-linker.json` 配置。
3. 识别项目级目录：
   - `.agents/skills`
   - `.codex/skills`
   - `.claude/skills`
4. 识别用户级目录：
   - `~/.agents/skills`
   - `~/.codex/skills`
   - `~/.claude/skills`
5. 识别中央目录候选：
   - 当前项目 `.skill-linker.json` 中的 `central_skills_dir`
   - 用户级 `~/.skill-linker.json` 中的 `central_skills_dir`
   - 用户提供的路径
   - `~/GitHub/*/skills`
   - `~/GitHub/*/.agents/skills`
   - `~/Skills`
   - `~/.agents/skills`
6. 汇报当前目录结构、重复 skill 名、软链接、失效软链接、复制出来的 skills，以及可能的中央仓库。
7. 如果配置中已有中央目录，围绕该中央目录给出建议方案；如果没有配置，再要求用户确认中央目录。
8. 只执行已经确认的操作。
9. 使用 `ls -l`、`readlink` 或 `scripts/skill_manager.py inspect` 验证结果。

## 配置文件

支持用 `.skill-linker.json` 记住用户已经确认过的中央 skills 目录。

读取优先级：

1. 当前项目 `.skill-linker.json`
2. 用户级 `~/.skill-linker.json`
3. 没有配置时，再扫描候选目录并询问用户

推荐的用户级配置位置：

```text
macOS/Linux: ~/.skill-linker.json
Windows: C:\Users\you\.skill-linker.json
```

配置示例：

```json
{
  "central_skills_dir": "/Users/name/.agents/skills",
  "default_mode": "centralize"
}
```

字段含义：

- `central_skills_dir`：已经确认过的中央 skills 原件目录。
- `default_mode: "centralize"`：默认把项目里真实 skill 迁入或同步到中央目录，再把项目入口改为软链接。
- `default_mode: "project-local"`：允许当前项目 `.agents/skills` 作为本项目内部中心目录。
- `default_mode: "ask"`：每次列出选项，让用户选择。

如果配置存在且 `default_mode` 是 `centralize`，当当前项目 `.agents/skills` 里出现真实 skill 目录时，不要默认把它当作项目中心原件长期保留；应建议同步或迁移到配置的中央目录，然后在项目中留下软链接。执行前仍必须列出计划并等待用户确认。

## 首次触发流程

用户第一次要求“管理 skills”“统一 skills”“同步 Codex 和 Claude 的 skills”或类似任务时，按以下顺序处理：

1. 先运行只读检查，不创建目录、不创建软链接、不删除文件。
2. 先读取配置文件：
   - 如果当前项目 `.skill-linker.json` 存在，使用它。
   - 否则如果 `~/.skill-linker.json` 存在，使用它。
   - 如果配置中的中央目录存在，明确写出 `[已配置中央目录]`，不要重新猜测中央目录。
3. 同时扫描用户级目录和当前项目目录：
   - 用户级：`~/.agents/skills`、`~/.codex/skills`、`~/.claude/skills`
   - 项目级：`.agents/skills`、`.codex/skills`、`.claude/skills`
4. 汇总发现：
   - 哪些目录存在
   - 哪些目录不存在
   - 哪些条目是真实目录
   - 哪些条目是软链接
   - 哪些软链接已经失效
   - 哪些 skill 名在多个位置重复
   - 哪些路径可能是中央 skills 目录
5. 如果已有有效配置，围绕配置的中央目录生成方案；如果没有配置，向用户说明候选中央目录，并请用户确认要使用哪一个。
6. 在没有配置且用户未确认中央目录前，不要执行迁移、同步、替换或 git 更新。

如果没有发现明显的中央目录，询问用户希望把中央 skills 目录放在哪里；不要自行创建中央目录，除非用户明确同意。

## 同步前确认

同步、迁移、替换、备份、删除软链接、创建软链接、克隆仓库、运行 `git fetch` 或运行 git 更新前，必须先给用户一份计划。计划至少包含：

- 将创建哪些目录
- 将创建哪些软链接，以及每个软链接指向哪里
- 将跳过哪些已有目录或冲突项
- 是否会备份任何内容
- 是否会删除软链接
- 是否会克隆仓库，以及克隆到哪里
- 是否会运行 `git fetch`、`git pull` 或 `git checkout`
- 明确说明不会删除真实 skill 目录，除非用户明确要求

用户确认后再执行。执行完成后，再运行检查命令验证结果。

## 输出格式规则

向用户汇报检查结果和整理方案时，必须把“用户级”和“项目级”分开写清楚。不要只写 `.agents/skills`、`.codex/skills`、`.claude/skills`，因为这些名称在用户级和项目级都可能出现。

路径必须同时尽量提供：

- 层级标签：`用户级`、`项目级`、`中央候选`、`备份目录`
- Agent 标签：`通用 .agents`、`Codex .codex`、`Claude .claude`
- 相对路径：例如 `.codex/skills`
- 绝对路径：例如 `/Users/name/project/.codex/skills`
- 状态：`真实目录`、`软链接`、`junction`、`失效链接`、`不存在`

推荐汇报格式：

```text
项目根目录：
/abs/path/to/project

项目级 skills：
- [项目级][通用 .agents][真实目录] .agents/skills
  绝对路径：/abs/path/to/project/.agents/skills
  数量：41
- [项目级][Codex .codex][真实目录] .codex/skills
  绝对路径：/abs/path/to/project/.codex/skills
  数量：47

用户级 skills：
- [用户级][Codex .codex][真实目录] ~/.codex/skills
  绝对路径：/Users/name/.codex/skills
```

推荐执行计划格式：

```text
我建议执行以下变更：

| 动作 | 来源 | 目标 | 影响 | 是否删除真实目录 |
| --- | --- | --- | --- | --- |
| 迁移 | [项目级][Codex] /abs/project/.codex/skills/foo | [项目级][中央 .agents] /abs/project/.agents/skills/foo | foo 会进入中央目录 | 否 |
| 备份 | [项目级][Claude] /abs/project/.claude/skills | /abs/project/.skill-linker-backup/.../.claude/skills | 保留原目录副本 | 否 |
| 建立软链接 | [项目级][Claude] /abs/project/.claude/skills | ../.agents/skills | Claude 入口指向中央目录 | 否 |
```

确认口令也要包含作用范围，避免用户误解。例如：

```text
如果你确认要整理“当前项目级 skills”，请回复：确认整理当前项目
```

不要使用含糊口令，例如只写“确认整理”。如果操作会影响用户级目录，确认口令必须明确写出“用户级”。

## 脚本

使用 `scripts/skill_manager.py` 进行确定性的检查和安全的软链接操作。除非传入 `--execute`，默认命令都是只读或 dry-run。

检查当前状态：

```bash
python3 scripts/skill_manager.py inspect --project .
```

查看当前生效配置：

```bash
python3 scripts/skill_manager.py config --project .
```

以 dry-run 方式写入用户级中央目录配置：

```bash
python3 scripts/skill_manager.py config --scope user --central ~/.agents/skills --mode centralize
```

用户确认后写入：

```bash
python3 scripts/skill_manager.py config --scope user --central ~/.agents/skills --mode centralize --execute
```

以 dry-run 方式初始化项目入口目录：

```bash
python3 scripts/skill_manager.py init --project . --agents claude,codex
```

用户确认后执行：

```bash
python3 scripts/skill_manager.py init --project . --agents claude,codex --execute
```

以 dry-run 方式链接单个 skill：

```bash
python3 scripts/skill_manager.py link --project . --source ~/GitHub/my-skills/skills/write-blog
```

用户确认后执行：

```bash
python3 scripts/skill_manager.py link --project . --source ~/GitHub/my-skills/skills/write-blog --execute
```

只检查链接：

```bash
python3 scripts/skill_manager.py check --project .
```

以 dry-run 方式从当前项目停用某个 skill。此命令只删除软链接或 junction，不删除中央原件：

```bash
python3 scripts/skill_manager.py unlink --target .agents/skills/write-blog
```

用户确认后执行停用：

```bash
python3 scripts/skill_manager.py unlink --target .agents/skills/write-blog --execute
```

检查单个下载仓库的 git 状态：

```bash
python3 scripts/skill_manager.py git-status --repo ~/GitHub/my-skills
```

检查中央目录下哪些下载的 skill 仓库可能落后远端。默认只使用本地已有的远端跟踪信息，不联网：

```bash
python3 scripts/skill_manager.py updates --central ~/GitHub/my-skills/skills
```

用户确认后，获取最新远端信息再判断哪些仓库有更新：

```bash
python3 scripts/skill_manager.py updates --central ~/GitHub/my-skills/skills --execute
```

用户确认后，更新某个下载仓库：

```bash
python3 scripts/skill_manager.py update --repo ~/GitHub/my-skills --execute
```

以 dry-run 方式克隆 skill 仓库到中央仓库父目录：

```bash
python3 scripts/skill_manager.py clone --repo-url https://github.com/user/some-skills.git --dest-parent ~/GitHub
```

用户确认后执行克隆，并在克隆后识别仓库中包含哪些 skills：

```bash
python3 scripts/skill_manager.py clone --repo-url https://github.com/user/some-skills.git --dest-parent ~/GitHub --execute
```

以 dry-run 方式切换某个下载仓库到指定版本：

```bash
python3 scripts/skill_manager.py checkout --repo ~/GitHub/my-skills --ref v1.2.0
```

以 dry-run 方式把多个 skills 链接到当前项目：

```bash
python3 scripts/skill_manager.py link-many --project . --sources ~/GitHub/my-skills/skills/a,~/GitHub/my-skills/skills/b
```

以 dry-run 方式把已有真实 skill 目录迁移到中央目录，并在原位置创建软链接：

```bash
python3 scripts/skill_manager.py migrate --source ~/.claude/skills/write-blog --central ~/GitHub/my-skills/skills
```

Windows 上如果 symlink 权限不足，可以显式使用 junction：

```powershell
python scripts/skill_manager.py link --project . --source C:\Users\you\GitHub\my-skills\skills\write-blog --link-type junction
python scripts/skill_manager.py init --project . --agents claude,codex --link-type junction
python scripts/skill_manager.py migrate --source C:\Users\you\.claude\skills\write-blog --central C:\Users\you\GitHub\my-skills\skills --link-type junction
```

## 操作指引

首次使用时，先运行 `inspect`，总结发现的问题，然后询问用户哪个目录应作为中央 skills 目录。不要仅凭某个路径存在就把它推断为中央目录。

链接 skill 时，要求源目录有效且包含 `SKILL.md`。如果目标名称已经存在：
- 如果它已经是同一个软链接，报告无需修改。
- 如果它是失效软链接，询问是否替换。
- 如果它是真实目录或文件，不要覆盖；建议备份、手动迁移或跳过。

删除或停用 skill 时，默认只从当前项目停用，不删除中央原件。先检查目标路径属于用户级还是项目级，以及它是软链接、junction、真实目录、失效链接还是不存在。如果目标是项目级软链接或 junction，只删除链接本身。如果目标是真实目录，停止并说明删除会删除 skill 内容本身，建议迁移、备份或只取消引用。如果用户明确要求删除中央原件，必须列出影响范围，说明所有指向它的项目可能变成失效链接，并要求明确确认口令，例如 `确认删除中央原件 write-blog`。

批量链接多个 skills 时，先逐个验证源目录都包含 `SKILL.md`，再列出计划创建的所有软链接。只在用户确认后执行 `link-many --execute`。

当用户要求下载、克隆或安装一个仓库时，先判断它是否可能是 skills 仓库。判断依据包括：仓库名或路径包含 `skill`/`skills`，用户描述中提到 agent skills，或克隆/下载后能发现 `SKILL.md`、`skills/*/SKILL.md`、`.agents/skills/*/SKILL.md`。如果判断为 skills 仓库，询问用户是否要克隆到中央 skills 仓库父目录，例如 `~/GitHub`。不要默认克隆到当前项目里。

克隆完成后，识别仓库内的 skill 位置：
- 仓库根目录存在 `SKILL.md`：整个仓库可能就是一个 skill。
- 仓库内存在 `skills/<name>/SKILL.md`：这些子目录是可链接 skills。
- 仓库内存在 `.agents/skills/<name>/SKILL.md`：这些子目录是可链接 skills。

迁移已有真实目录时，适用于“全局目录或项目目录里已经复制了一份 skill，现在想改成中央目录 + 软链接”的场景。迁移计划必须说明：真实目录会被移动到中央目录，原位置会变成软链接；不会删除真实 skill 内容。目标中央路径已存在时，默认停止并让用户选择。

同步多个 Agent 时，优先使用 `.agents/skills` 作为项目级中心目录，并且只在用户确认后把 `.claude/skills` 和 `.codex/skills` 链接到它。如果其中任一目录已经是真实目录，报告冲突并让用户选择处理方式。

更新时，把 git 作为版本管理来源。先用 `git-status` 或 `updates` 报告仓库路径、当前分支、上游分支、本地改动、ahead/behind 状态。默认不要联网检查；如果需要知道远端是否有新提交，先说明会运行 `git fetch --prune`，获得用户确认后再执行。运行 `git pull` 前，报告仓库路径以及是否有本地改动；有本地改动时默认不要更新。如果用户想固定到某个版本，只在确认目标 ref 后使用 `git checkout`。

如果用户问“哪个下载的 skill 更新了”，先确认中央 skills 目录，然后运行 `updates --central <目录>`。如果结果可能过期，说明它只是本地远端跟踪信息；询问用户是否允许获取远端信息，确认后再运行 `updates --central <目录> --execute`。只在用户明确确认后，才对具体仓库运行 `update --repo <仓库> --execute`。

如果用户需要固定某个版本，先运行 `git-status` 查看本地改动和当前分支，再说明将执行 `git checkout <ref>`。有本地改动时默认不要切换；用户确认风险后才能使用 `--allow-dirty`。

## Fork 决策

fork 的意思是：把别人的 GitHub 仓库复制成你自己账号下的一份仓库。适用于你需要长期修改别人的 skill，同时还希望以后能从原作者那里同步更新。

触发 fork 建议的典型情况：

- 用户说“我经常要改这个别人的 skill”。
- git 状态显示中央仓库有本地改动，同时用户又想更新远端。
- 用户担心自己的修改被 `git pull` 覆盖、冲突或难以维护。
- 用户想把本地修复提交到自己的 GitHub。

处理流程：

1. 先确认当前仓库是不是用户自己的仓库；如果不确定，查看 `git remote -v` 并向用户说明。
2. 如果这是别人的仓库且用户要长期改，建议用户 fork 一份到自己的 GitHub。
3. fork 后，把本地仓库的 `origin` 指向用户自己的 fork，把原作者仓库作为 `upstream`。
4. 后续本地修改提交到 `origin`；需要同步原作者更新时，从 `upstream` 拉取。
5. 不要自动执行 fork、改 remote 或推送，除非用户明确确认。

对团队仓库，尽量使用相对软链接；如果不合适，就提供安装脚本或说明，避免提交绝对软链接。

不要为定时任务添加自动化逻辑；如果用户要求定时更新，先解释该 skill 当前不内置定时任务，再建议手动运行更新检查命令。

## Windows 兼容

Windows 可以使用这套中央目录 + 项目链接方案，但链接机制和权限与 macOS/Linux 不同。

优先顺序：

1. 先尝试普通目录 symlink。
2. 如果失败，说明可能需要开启 Windows Developer Mode 或使用管理员权限终端。
3. 如果只是链接 skill 目录，可以建议使用 junction：`mklink /J`。
4. 不要自动提权，不要绕过 UAC，不要代替用户输入管理员密码。

概念说明：

- symlink：最接近 macOS/Linux 的软链接，命令类似 `mklink /D link target`。
- junction：Windows 的目录联接，命令类似 `mklink /J link target`，常用于目录，权限要求通常更友好。
- `.lnk` 快捷方式：不要用于 skills；很多 Agent 和脚本不会把 `.lnk` 当真实目录读取。

脚本行为：

- 默认 `--link-type auto`。
- 在 Windows 上，`auto` 会先尝试 symlink；如果权限不足，会尝试 junction。
- 如果 junction 也失败，向用户说明 Developer Mode、管理员终端和手动 `mklink /J` 命令。
- 用户明确想使用 junction 时，传入 `--link-type junction`。

Agent 引导用户时可以这样说明：

```text
当前 Windows 环境可能没有权限创建目录 symlink。
你可以选择开启 Developer Mode、用管理员权限终端重试，或改用 junction。
我不会自动提权；你确认后我可以继续生成或执行对应命令。
```
