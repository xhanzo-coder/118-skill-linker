# 118 Skill Linker

`118-skill-linker` is a Codex skill for managing Agent skills with a central skills directory and project-level symlinks.

It is designed for users who work with multiple agents, such as Codex and Claude Code, and want one clean way to inspect, clone, link, migrate, update, and version skills without scattering duplicate copies across projects.

## What It Does

- Inspect user-level and project-level skill directories.
- Detect real directories, symlinks, broken symlinks, duplicate skill names, and likely central skill repositories.
- Initialize project-level skill entry points such as `.agents/skills`, `.codex/skills`, and `.claude/skills`.
- Link one or more skills from a central directory into the current project.
- Clone likely skills repositories into a central parent directory after user confirmation.
- Migrate copied skill directories into a central directory and replace the original location with a symlink.
- Check git status, detect update candidates, update downloaded skill repositories, and checkout fixed versions.
- Provide guidance for when to fork someone else's skill repository.

## Safety Model

The skill is intentionally conservative.

- It starts with read-only inspection when state is unknown.
- It asks the user to confirm the central skills directory before treating it as authoritative.
- It uses dry-run behavior by default for write operations.
- It does not overwrite real directories with symlinks.
- It does not delete real skill directories unless the user explicitly asks for that exact deletion.
- It explains sync, migration, clone, git update, and checkout plans before execution.

## Layout

```text
118-skill-linker/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── scripts/
    └── skill_manager.py
```

## Example Commands

Inspect the current project:

```bash
python3 118-skill-linker/scripts/skill_manager.py inspect --project .
```

Initialize project skill entry points as a dry run:

```bash
python3 118-skill-linker/scripts/skill_manager.py init --project . --agents claude,codex
```

Link a skill as a dry run:

```bash
python3 118-skill-linker/scripts/skill_manager.py link --project . --source ~/GitHub/my-skills/skills/write-blog
```

Check downloaded skill repositories for update candidates:

```bash
python3 118-skill-linker/scripts/skill_manager.py updates --central ~/GitHub/my-skills/skills
```

## Windows Notes

The workflow is cross-platform in concept, but Windows symlink creation can require Developer Mode or an administrator terminal. Directory junctions can be an alternative for directory-based skills. The skill should guide the user rather than silently elevating privileges.

## Installation

For project-level use, link or copy this folder into a project's skill directory:

```bash
.agents/skills/118-skill-linker -> /path/to/118-skill-linker
```

If Codex or Claude Code expects a different entry point, link that entry point to `.agents/skills` after confirming the plan:

```bash
.codex/skills  -> .agents/skills
.claude/skills -> .agents/skills
```

