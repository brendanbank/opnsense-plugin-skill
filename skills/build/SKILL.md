---
name: build
description: Build, edit, or create OPNsense MVC plugins. Use when working on plugin source, packaging, or deploying to a firewall.
argument-hint: "[target-firewall]"
---

# OPNsense Plugin Development

Use this skill when building, editing, or creating OPNsense MVC plugins.

## Context Detection

Read the project's `Makefile` to determine `PLUGIN_NAME`, `PLUGIN_VERSION`, and other metadata.
Derive the plugin's namespace and paths from the directory structure under `src/`.

## Building a Package

If `$ARGUMENTS` is provided or the user asks to build:

### Prerequisites

1. `tmp/plugins/` must exist with `Mk/`, `Templates/`, `Scripts/` (the OPNsense plugins repo).
   If missing: `git clone https://github.com/opnsense/plugins.git tmp/plugins`
2. `build.sh` must exist and be executable
3. All PHP scripts in `src/opnsense/scripts/` must have execute permissions (`chmod +x`)

### Build

Run `./build.sh` which uploads source to a build firewall via SSH, runs `bmake package`
remotely (requires FreeBSD), and downloads the `.pkg` to `dist/`.

Git version.sh warnings about "not a git repository" are expected and harmless.

### Deploy (if target firewall provided)

If a target firewall hostname was given in `$ARGUMENTS`:

1. Find the `.pkg` in `dist/`
2. Upload: `scp dist/<pkg-file> <firewall>:/tmp/`
3. Remove old version if installed: `ssh <firewall> "sudo pkg delete -y <pkg-name>"`
4. Install: `ssh <firewall> "sudo pkg install -y /tmp/<pkg-file>"`
5. Start the service: `ssh <firewall> "sudo configctl <plugin_name> start"`
6. Verify: `ssh <firewall> "sudo configctl <plugin_name> status"`

## Creating or Editing a Plugin

When creating new files or modifying plugin structure, follow the patterns documented in
[plugin-reference.md](plugin-reference.md). Key rules:

- Follow the OPNsense MVC directory structure exactly
- All scripts in `src/opnsense/scripts/` **must** have execute permissions
- The `[status]` configd action must return text containing `"is running"` or `"not running"`
- `$internalModelName` in the API controller must match the `<id>` prefix in `forms/*.xml`
- Model fields go at root of `<items>` unless explicitly wrapping in a container
- Always use `escapeHtml()` for dynamic content in Volt templates (XSS prevention)
- Copyright: Brendan Bank, year 2026

## Troubleshooting

- **Error 126 from configd**: A PHP script is missing execute permission (`chmod +x`)
- **Menu/ACLs not appearing**: Run `sudo /usr/local/etc/rc.configure_plugins POST_INSTALL`
- **Status shows "not running"**: Check the `[status]` action output matches expected format
- **Config not applied after save**: Ensure the save button calls the `reconfigure` endpoint

## Reference

For complete plugin architecture details, see [plugin-reference.md](plugin-reference.md).
