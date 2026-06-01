# beamdrop Agent Skills

This repository contains a collection of agent skills for beamdrop. These skills are designed to enhance the capabilities of agents by providing them with specialized functionalities.

## What are Agent Skills?

Agent Skills are folders of instructions, scripts, and resources that agents can discover and use to do things more accurately and efficiently. They work across any AI agent that supports the [open Agent Skills standard](https://agentskills.io).

## Available Skills
<!-- START:Available-Skills -->
| Skill | Description |
| ----- | ----------- |
| [beamdrop](./skills/beamdrop) | Interact with a Beamdrop file storage server — upload, download, and manage f... |
<!-- END:Available-Skills -->

## Installation

### Skills

Use [skills](https://skills.sh/) to install skills directly:

```bash
# Install all skills
npx skills add beamdrop/agent-skills

# Install specific skills
npx skills add beamdrop/agent-skills --skill beamdrop

# List available skills
npx skills add beamdrop/agent-skills --list
```

### Claude Code Plugin

Install via Claude Code's plugin system:

```bash
# Add the plugin (includes all skills)
/plugin add beamdrop/agent-skills
```

> Claude Code plugins are also supported in Factory's [Droid](https://docs.factory.ai/cli/configuration/plugins#claude-code-compatibility).

### Other Installation Methods

Agent skills can also be installed by using the below commands from [Playbooks](https://playbooks.com/skills) or [Context7](https://context7.com/docs/skills):

```bash
# Playbooks
npx playbooks add skill beamdrop/agent-skills

# Context7
npx ctx7 skills install /beamdrop/agent-skills
```

## Adding New Skills

Use the included script to add new skills:

```bash
node scripts/add-skill.js <skill-name> "<description>"
```

Example:

```bash
node scripts/add-skill.js beamdrop "Interact with a Beamdrop file storage server — upload, download, and manage files via the S3-compatible API. Use when the user wants to store files, create buckets, generate presigned URLs, or manage API keys on a Beamdrop instance."
```

This will create the skill structure and automatically update manifest.json, platform plugin files, skills/index.json, and this README.

## Scripts

| Script | Description |
| ------ | ----------- |
| `node scripts/add-skill.js` | Add a new skill to the repository |
| `node scripts/sync-skills.js` | Sync manifest.json, platform plugin files, skills/index.json, and README with skills directory |

## Resources

- [Agent Skills Specification](https://agentskills.io/specification)
- [npx skills](https://skills.sh/)
- [Validate Agent Skill](https://github.com/marketplace/actions/validate-skill)
- [Playbooks](https://playbooks.com/skills)
- [Context7 Skills](https://context7.com/docs/skills)

## Contributing

Contributions are welcome! Please read our [Contributing Guide](.github/CONTRIBUTING.md) for more information.

## License

MIT
# beamdrop-skills
