---
name: build
description: Build, edit, or create OPNsense MVC plugins. Use when working on plugin source, packaging, deploying to a firewall, or scaffolding a new plugin from scratch.
argument-hint: "[new <PluginName>] | [target-firewall]"
---

# OPNsense Plugin Development

Use this skill when building, editing, or creating OPNsense MVC plugins.

## Context Detection

Read the project's `Makefile` to determine `PLUGIN_NAME`, `PLUGIN_VERSION`, and other metadata.
Derive the plugin's namespace and paths from the directory structure under `src/`.

## Creating a New Plugin from Scratch

If the user asks to create a new plugin, or `$ARGUMENTS` starts with `new`:

Ask the user for:
1. **Plugin name** (PascalCase, e.g., `HelloWorld`) — used for namespace and directory names
2. **Short description** — used in Makefile `PLUGIN_COMMENT` and pkg-descr
3. **Maintainer email**
4. **Whether it needs a daemon** (background service) or is config/UI only

Then generate all required files using the templates in [plugin-reference.md](plugin-reference.md).
Use the lowercase/underscore version of the name for `PLUGIN_NAME`, filenames, and configd actions
(e.g., `HelloWorld` → `hello_world`). Use the all-lowercase version (no underscores) for URLs
(e.g., `helloworld`).

### Files to generate

**Always required:**

| File | Purpose |
|------|---------|
| `Makefile` | Plugin metadata (`PLUGIN_NAME`, `PLUGIN_VERSION`, etc.) |
| `pkg-descr` | Package description with `WWW:` line |
| `src/etc/inc/plugins.inc.d/<plugin>.inc` | Service registration & lifecycle hooks |
| `src/opnsense/mvc/app/models/OPNsense/<Plugin>/<Plugin>.php` | Model class (extends BaseModel) |
| `src/opnsense/mvc/app/models/OPNsense/<Plugin>/<Plugin>.xml` | Model schema with `enabled` BooleanField |
| `src/opnsense/mvc/app/models/OPNsense/<Plugin>/ACL/ACL.xml` | Access control patterns |
| `src/opnsense/mvc/app/models/OPNsense/<Plugin>/Menu/Menu.xml` | Menu entries under Services |
| `src/opnsense/mvc/app/controllers/OPNsense/<Plugin>/GeneralController.php` | UI settings page |
| `src/opnsense/mvc/app/controllers/OPNsense/<Plugin>/Api/GeneralController.php` | Settings API |
| `src/opnsense/mvc/app/controllers/OPNsense/<Plugin>/Api/ServiceController.php` | Service control API |
| `src/opnsense/mvc/app/controllers/OPNsense/<Plugin>/forms/general.xml` | Form field definitions |
| `src/opnsense/mvc/app/views/OPNsense/<Plugin>/general.volt` | Settings page template |
| `src/opnsense/service/conf/actions.d/actions_<plugin>.conf` | Configd actions (start/stop/status) |

**If the plugin has a daemon:**

| File | Purpose |
|------|---------|
| `src/opnsense/scripts/OPNsense/<Plugin>/generate_config.php` | Config generator (runs as root) |
| `src/opnsense/scripts/OPNsense/<Plugin>/<plugin>.php` | Main daemon script |
| `src/opnsense/service/templates/OPNsense/Syslog/local/<plugin>.conf` | Syslog filter |

**Optional (generate if user requests):**

| File | Purpose |
|------|---------|
| `src/opnsense/mvc/app/controllers/OPNsense/<Plugin>/StatusController.php` | UI status page |
| `src/opnsense/mvc/app/controllers/OPNsense/<Plugin>/Api/StatusController.php` | Status data API |
| `src/opnsense/mvc/app/views/OPNsense/<Plugin>/status.volt` | Status page template |
| `src/opnsense/scripts/OPNsense/<Plugin>/<utility>.php` | Utility scripts |
| `build.sh` | Remote build automation script |
| `.gitignore` | Ignore `tmp/`, `dist/`, `.claude/` |

### Critical rules when generating files

- All scripts in `src/opnsense/scripts/` **must** be created with execute permissions
- The `[status]` configd action must output `"<plugin> is running"` or `"<plugin> is not running"`
- `$internalModelName` in ApiGeneralController must match the `<id>` prefix in `forms/general.xml`
- Model fields go at root of `<items>` (no intermediate wrapper element)
- The `enabled` BooleanField should default to `1` (enabled on fresh install) for better UX
- Use `escapeHtml()` for all dynamic content in Volt templates
- PHP files use `<?php` opening tag, scripts use `#!/usr/local/bin/php` shebang
- BSD 2-Clause license header required in **all** PHP/inc files (mandatory for upstream submission)
- Follow PSR-12 coding standard (max line length 120 chars)
- Add `PLUGIN_DEPENDS` in Makefile if the plugin requires other packages

### After scaffolding

Tell the user:
1. Review the generated files and customize the model fields for their use case
2. Clone the plugins build system: `git clone https://github.com/opnsense/plugins.git tmp/plugins`
3. Add `tmp/`, `dist/`, and `.claude/` to `.gitignore`
4. See [plugin-reference.md](plugin-reference.md) for details on each component

## Building a Package

If `$ARGUMENTS` contains a firewall hostname or the user asks to build:

### Prerequisites

1. `tmp/plugins/` must exist with `Mk/`, `Templates/`, `Scripts/` (the OPNsense plugins repo).
   If missing: `git clone https://github.com/opnsense/plugins.git tmp/plugins`
2. `build.sh` must exist and be executable
3. All PHP scripts in `src/opnsense/scripts/` must have execute permissions (`chmod +x`)

### Build

Run `./build.sh` which uploads source to a build firewall via SSH, runs `sudo bmake package`
remotely (requires FreeBSD), and downloads the `.pkg` to `dist/`.

The build must use `sudo` because `bmake package` invokes `pkg` for dependency checking,
which requires root privileges. The cleanup step also needs `sudo rm -rf` since root-owned
files are created during the build.

Git version.sh warnings about "not a git repository" are expected and harmless.

### Deploy (if target firewall provided)

If a target firewall hostname was given in `$ARGUMENTS`:

1. Find the `.pkg` in `dist/`
2. Upload: `scp dist/<pkg-file> <firewall>:/tmp/`
3. Remove old version if installed: `ssh <firewall> "sudo pkg delete -y <pkg-name>"`
4. If upgrading from a renamed plugin, clean up old files (old .inc, old scripts dir, old actions conf, old syslog conf) and remove old model config from `/conf/config.xml`
5. Install: `ssh <firewall> "sudo pkg install -y /tmp/<pkg-file>"`
6. Start the service: `ssh <firewall> "sudo configctl <plugin_name> start"`
7. Verify: `ssh <firewall> "sudo configctl <plugin_name> status"`

## Editing an Existing Plugin

When creating new files or modifying plugin structure, follow the patterns documented in
[plugin-reference.md](plugin-reference.md). Key rules:

- Follow the OPNsense MVC directory structure exactly
- All scripts in `src/opnsense/scripts/` **must** have execute permissions
- The `[status]` configd action must return text containing `"is running"` or `"not running"`
- `$internalModelName` in the API controller must match the `<id>` prefix in `forms/*.xml`
- Model fields go at root of `<items>` unless explicitly wrapping in a container
- Always use `escapeHtml()` for dynamic content in Volt templates (XSS prevention)

## Troubleshooting

- **Error 126 from configd**: A PHP script is missing execute permission (`chmod +x`)
- **Menu/ACLs not appearing**: Run `sudo /usr/local/etc/rc.configure_plugins POST_INSTALL`
- **Status shows "not running"**: Check the `[status]` action output matches expected format
- **Config not applied after save**: Ensure the save button calls the `reconfigure` endpoint
- **`pkg: No package(s) matching`**: Build requires `sudo bmake package` for pkg dependency checks
- **`Permission denied` on cleanup**: Use `sudo rm -rf` since build creates root-owned files
- **Enabled checkbox not checked after fresh install**: Set `<Default>1</Default>` on the `enabled` field in the model XML
- **Old menu entries after rename**: Remove old plugin files from firewall and run `rc.configure_plugins POST_INSTALL`

## Reference

For complete plugin architecture details, see [plugin-reference.md](plugin-reference.md).
