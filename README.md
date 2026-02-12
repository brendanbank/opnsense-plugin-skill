# OPNsense Plugin Skill for Claude Code

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for building, editing, and creating [OPNsense](https://opnsense.org/) MVC plugins.

This skill gives Claude deep knowledge of the OPNsense plugin architecture: directory structure, MVC framework, configd actions, service registration, packaging, and deployment. Use it when developing OPNsense plugins with Claude Code.

## Installation

Claude Code skills are just files — clone this repo and copy the skill directory to where you need it.

### As a personal skill (available across all your projects)

```sh
git clone https://github.com/brendanbank/opnsense-plugin-skill.git /tmp/opnsense-plugin-skill
cp -r /tmp/opnsense-plugin-skill/skills/build ~/.claude/skills/opnsense-plugin
```

### As a project skill (shared with contributors via git)

From your plugin project root:

```sh
git clone https://github.com/brendanbank/opnsense-plugin-skill.git /tmp/opnsense-plugin-skill
mkdir -p .claude/skills
cp -r /tmp/opnsense-plugin-skill/skills/build .claude/skills/
```

Then commit `.claude/skills/` to your repo so all contributors can use it.

### Verify installation

Start Claude Code and type `/build` — the skill should appear in autocomplete.

## Usage

### Build and deploy

```
/build                    # Build the plugin package
/build fw.example.com     # Build and deploy to a firewall
```

### Automatic context

When editing OPNsense plugin source files, Claude will automatically apply the plugin
conventions and patterns from this skill — correct directory structure, naming conventions,
security patterns, and framework integration.

### Ask questions

Claude can reference the skill's knowledge to answer questions like:

- "Create a new configd action for reloading config"
- "Add a status page to the plugin"
- "Why is my menu not appearing?"

## What's Included

- **`SKILL.md`** — Main skill with build/deploy instructions and plugin editing guidelines
- **`plugin-reference.md`** — Comprehensive reference covering:
  - Complete plugin directory structure
  - Makefile and pkg-descr format
  - Model XML schema (fields, validation, mount points)
  - All controller types (UI, API settings, API service, custom)
  - Forms and Volt template patterns
  - Configd action format and types
  - Service registration via `plugins.inc.d`
  - Menu and ACL XML structure
  - Daemon two-phase security pattern (root config generator + unprivileged daemon)
  - Signal handling and atomic file writes
  - Package build system and remote builds
  - Manual deployment sequence
  - Common pitfalls and troubleshooting

## Requirements

For building packages:
- An OPNsense/FreeBSD host accessible via SSH (for `bmake` and `pkg create`)
- The OPNsense plugins repo cloned locally: `git clone https://github.com/opnsense/plugins.git tmp/plugins`
- A `build.sh` script in your plugin repo (see the [gateway-exporter](https://github.com/brendanbank/opnsense-gateway-exporter) for an example)

## License

BSD-2-Clause
