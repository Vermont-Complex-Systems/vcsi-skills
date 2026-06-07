# VCSI-skills

AI coding skills maintained by [VCSI](https://vermontcomplexsystems.org/) at the University of Vermont. Each skill is a markdown file that teaches coding agents best practices for specific tools and workflows.

## Available Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [svelte-d3-charting](skills/svelte-d3-charting/) | Svelte 5 + D3 charting best practices: reactive scales, responsive sizing, animations | See below |

## Install

### Copy-paste (any agent)

Copy the `SKILL.md` file into your project's skills directory:

```bash
# Claude Code
mkdir -p .claude/skills/svelte-d3-charting
curl -o .claude/skills/svelte-d3-charting/SKILL.md \
  https://raw.githubusercontent.com/Vermont-Complex-Systems/vcsi-skills/main/skills/svelte-d3-charting/SKILL.md
```

### Svelte CLI (SvelteKit projects)

```bash
npx sv add @the-vcsi/svelte-d3-charting
```

## Contributing

Each skill lives in `skills/{name}/SKILL.md`. The SKILL.md frontmatter must include `name` and `description` fields.

See existing skills for the format.
