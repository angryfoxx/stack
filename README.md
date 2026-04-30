# stack

Personal collection of agent skills for Claude Code and other agents ([skills.sh](https://skills.sh)).

## Install as Claude Code plugins

Each skill ships as its own Claude Code plugin. Add the marketplace once, then install whichever plugins you want:

```text
/plugin marketplace add angryfox/stack
/plugin install incremental-commit@stack
/plugin install ninja-from-drf@stack
```

## Install via skills.sh

Install all skills:

```bash
npx skills add angryfox/stack
```

Install a specific skill:

```bash
npx skills add angryfox/stack/skills@incremental-commit
```

## Skills

| Skill | Description |
|---|---|
| [incremental-commit](./skills/incremental-commit/SKILL.md) | Break pending changes into multiple logical atomic commits with Conventional Commit messages |
| [ninja-from-drf](./skills/ninja-from-drf/SKILL.md) | Migrate Django REST Framework APIs to django-ninja / django-ninja-extra preserving routes, auth, throttling, filtering, and tests |

> When editing a skill, update both `skills/<name>/SKILL.md` (skills.sh) and `plugins/<name>/skills/<name>/SKILL.md` (Claude plugin).