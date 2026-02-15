# OPNsense Plugin Development Reference

Complete reference for building OPNsense MVC plugins, based on the official development
documentation at https://docs.opnsense.org/develop.html and practical implementation experience.

Throughout this document:
- `<Plugin>` = PascalCase plugin name (e.g., `HelloWorld`, `DnsBlocklist`)
- `<plugin>` = lowercase/underscore plugin name matching `PLUGIN_NAME` in Makefile (e.g., `hello_world`)
- `<pluginlowercase>` = all-lowercase version used in URLs (e.g., `helloworld`, `dnsblocklist`)

## Directory Structure

The `src/` directory maps directly to `/usr/local/` on the firewall:

```
project-root/
├── Makefile                          # Plugin metadata for build system
├── pkg-descr                         # Package description text
├── build.sh                          # Remote build automation (optional)
├── src/                              # → /usr/local/ on firewall
│   ├── etc/inc/plugins.inc.d/
│   │   └── <plugin>.inc             # Service registration & lifecycle hooks
│   └── opnsense/
│       ├── mvc/app/
│       │   ├── controllers/OPNsense/<Plugin>/
│       │   │   ├── GeneralController.php       # UI page controller
│       │   │   ├── Api/
│       │   │   │   ├── GeneralController.php   # Settings API (CRUD)
│       │   │   │   ├── ServiceController.php   # Service control API
│       │   │   │   └── StatusController.php    # Custom API endpoints (optional)
│       │   │   └── forms/
│       │   │       └── general.xml             # Form field definitions
│       │   ├── models/OPNsense/<Plugin>/
│       │   │   ├── <Plugin>.php                # Model class (extends BaseModel)
│       │   │   ├── <Plugin>.xml                # Model schema & validation
│       │   │   ├── ACL/ACL.xml                 # Access control rules
│       │   │   └── Menu/Menu.xml               # UI menu entries
│       │   └── views/OPNsense/<Plugin>/
│       │       ├── general.volt                # Settings page template
│       │       └── status.volt                 # Status page template (optional)
│       ├── scripts/OPNsense/<Plugin>/
│       │   ├── <daemon>.php                    # Main daemon script (optional)
│       │   ├── generate_config.php             # Config generator (optional)
│       │   └── <utility>.php                   # Utility scripts (optional)
│       └── service/
│           ├── conf/actions.d/
│           │   └── actions_<plugin>.conf       # Configd action definitions
│           └── templates/OPNsense/Syslog/local/
│               └── <plugin>.conf               # Syslog filter (optional)
└── tmp/plugins/                      # OPNsense plugins repo (in .gitignore)
    ├── Mk/plugins.mk                # Build system
    ├── Templates/                    # Package script templates
    └── Scripts/                      # Build helper scripts
```

## Makefile

Minimal plugin Makefile:

```makefile
PLUGIN_NAME=        <plugin>
PLUGIN_VERSION=     1.0
PLUGIN_COMMENT=     Short description of the plugin
PLUGIN_MAINTAINER=  email@example.com

.include "../../Mk/plugins.mk"
```

To declare package dependencies, add `PLUGIN_DEPENDS`:

```makefile
PLUGIN_DEPENDS=     os-some_dependency${PLUGIN_PKGSUFFIX}
```

`${PLUGIN_PKGSUFFIX}` resolves to `-devel` for development builds and is empty for release builds.

The build system derives:
- Package name: `os-<PLUGIN_NAME>` (prefix `os-` is automatic)
- Package file: `work/pkg/os-<PLUGIN_NAME>-<VERSION>.pkg`
- `PLUGINSDIR` defaults to `../../` from the Makefile directory

## pkg-descr

Multi-line package description. Must end with a `WWW:` line:

```
Description of the plugin functionality.

WWW: https://github.com/user/repo
```

## Model (Data Layer)

### PHP Class

Minimal model class — all logic is in XML schema:

```php
<?php
namespace OPNsense\<Plugin>;
use OPNsense\Base\BaseModel;

class <Plugin> extends BaseModel
{
}
```

### XML Schema (`<Plugin>.xml`)

Defines fields, validation, and where config is stored in `config.xml`:

```xml
<model>
    <mount>//OPNsense/<Plugin></mount>
    <description>Plugin description</description>
    <version>1.0.0</version>
    <items>
        <enabled type="BooleanField">
            <Default>1</Default>
        </enabled>
        <myfield type="TextField">
            <Default>some default</Default>
            <Mask>/^regex$/</Mask>
            <ValidationMessage>Error message shown to user</ValidationMessage>
        </myfield>
        <mynumber type="IntegerField">
            <Default>15</Default>
            <MinimumValue>5</MinimumValue>
            <MaximumValue>300</MaximumValue>
        </mynumber>
    </items>
</model>
```

Key points:
- `<mount>` defines the XPath in `config.xml` where data is stored
- Fields go at the root of `<items>` (no intermediate wrapper needed)
- Available field types: `TextField`, `BooleanField`, `IntegerField`, `EmailField`,
  `NetworkField`, `ArrayField`, `BaseListField`, and custom types
- `<Mask>` uses regex for validation
- `volatile="true"` on a field prevents persistence (useful for diagnostics)
- `<mount>:memory:</mount>` creates an in-memory-only model

### Migrations

When the `<version>` in the XML changes, create a migration class. Run migrations with:
```
/usr/local/opnsense/mvc/script/run_migrations.php OPNsense/<Plugin>
```

## Controllers

### UI Controller (page rendering)

Renders Volt templates for the web UI:

```php
<?php
namespace OPNsense\<Plugin>;
use OPNsense\Base\IndexController;

class GeneralController extends IndexController
{
    public function indexAction()
    {
        $this->view->pick('OPNsense/<Plugin>/general');
        $this->view->generalForm = $this->getForm("general");
    }
}
```

URL routing: `/ui/<pluginlowercase>/general/index`

### API Settings Controller (model CRUD)

Provides `get` and `set` endpoints for the model:

```php
<?php
namespace OPNsense\<Plugin>\Api;
use OPNsense\Base\ApiMutableModelControllerBase;

class GeneralController extends ApiMutableModelControllerBase
{
    protected static $internalModelName = 'general';
    protected static $internalModelClass = 'OPNsense\<Plugin>\<Plugin>';
}
```

Auto-generates:
- `GET /api/<pluginlowercase>/general/get` — retrieve settings
- `POST /api/<pluginlowercase>/general/set` — save settings

`$internalModelName` is the key under which settings appear in API responses.

### API Service Controller (start/stop/status)

Provides service lifecycle endpoints:

```php
<?php
namespace OPNsense\<Plugin>\Api;
use OPNsense\Base\ApiMutableServiceControllerBase;

class ServiceController extends ApiMutableServiceControllerBase
{
    protected static $internalServiceClass = '\OPNsense\<Plugin>\<Plugin>';
    protected static $internalServiceEnabled = 'enabled';
    protected static $internalServiceName = '<plugin>';
}
```

Auto-generates:
- `POST /api/<pluginlowercase>/service/start`
- `POST /api/<pluginlowercase>/service/stop`
- `POST /api/<pluginlowercase>/service/restart`
- `POST /api/<pluginlowercase>/service/reconfigure`
- `GET /api/<pluginlowercase>/service/status`

The `status` action expects the configd status command to return text containing
`"is running"` or `"not running"`.

### Custom API Controller

For custom endpoints (e.g., fetching live data):

```php
<?php
namespace OPNsense\<Plugin>\Api;
use OPNsense\Base\ApiControllerBase;
use OPNsense\Core\Backend;

class StatusController extends ApiControllerBase
{
    public function dataAction()
    {
        $backend = new Backend();
        $response = json_decode(trim($backend->configdRun('<plugin> <action>')), true);
        return $response;
    }
}
```

Endpoint: `GET /api/<pluginlowercase>/status/data`

## Forms (`forms/general.xml`)

Maps model fields to UI form elements:

```xml
<form>
    <field>
        <id>general.enabled</id>
        <label>Enabled</label>
        <type>checkbox</type>
    </field>
    <field>
        <id>general.myfield</id>
        <label>My Field</label>
        <type>text</type>
        <help>Help text shown as tooltip.</help>
    </field>
</form>
```

The `<id>` uses the pattern `<modelName>.<fieldName>` matching `$internalModelName` in the
API controller. Supported types: `text`, `password`, `textbox`, `checkbox`, `dropdown`,
`select_multiple`, `hidden`, `info`.

## Views (Volt Templates)

### Settings Page (`general.volt`)

Standard pattern for a settings page with service controls:

```html
<script>
    $(document).ready(function() {
        mapDataToFormUI({'frm_general': "/api/<pluginlowercase>/general/get"}).done(function() {
            updateServiceControlUI('<pluginlowercase>');
        });

        $("#saveAct").SimpleActionButton({
            onPreAction: function() {
                const dfObj = new $.Deferred();
                saveFormToEndpoint("/api/<pluginlowercase>/general/set", 'frm_general', function() {
                    dfObj.resolve();
                });
                return dfObj;
            },
            onAction: function(data, status) {
                updateServiceControlUI('<pluginlowercase>');
            }
        });
    });
</script>

<div class="content-box" style="padding-bottom: 1.5em;">
    {{ partial("layout_partials/base_form", ['fields': generalForm, 'id': 'frm_general']) }}
    <div class="col-md-12">
        <button class="btn btn-primary" id="saveAct"
                data-endpoint="/api/<pluginlowercase>/service/reconfigure"
                data-label="{{ lang._('Save') }}"
                data-error-title="{{ lang._('Error') }}"
                type="button">
        </button>
    </div>
</div>
```

Key framework functions:
- `mapDataToFormUI()` — loads model data into form fields
- `saveFormToEndpoint()` — validates and saves form data via API
- `updateServiceControlUI()` — shows service start/stop/status controls

### Status/Data Pages

Fetch data via AJAX and render dynamically:

```html
<script>
    $(document).ready(function() {
        ajaxCall("/api/<pluginlowercase>/status/data", {}, function(data) {
            // Build HTML table from response
        });
    });

    function escapeHtml(str) {
        return $('<span>').text(str).html();
    }
</script>
```

Always use `escapeHtml()` for dynamic content to prevent XSS.

## Configd Actions (`actions_<plugin>.conf`)

Defines commands callable via `configctl <plugin> <action>`:

```ini
[start]
command:/usr/local/opnsense/scripts/OPNsense/<Plugin>/generate_config.php && /usr/sbin/daemon -u nobody -f -p /var/run/<plugin>.pid /usr/local/opnsense/scripts/OPNsense/<Plugin>/<daemon>.php && chmod 644 /var/run/<plugin>.pid
parameters:
type:script
message:Starting <Plugin>

[stop]
command:/bin/pkill -F /var/run/<plugin>.pid 2>/dev/null; rm -f /var/run/<plugin>.pid; exit 0
parameters:
type:script
message:Stopping <Plugin>

[restart]
command:/bin/pkill -F /var/run/<plugin>.pid 2>/dev/null; rm -f /var/run/<plugin>.pid; sleep 1; /usr/local/opnsense/scripts/OPNsense/<Plugin>/generate_config.php && /usr/sbin/daemon -u nobody -f -p /var/run/<plugin>.pid /usr/local/opnsense/scripts/OPNsense/<Plugin>/<daemon>.php && chmod 644 /var/run/<plugin>.pid
parameters:
type:script
message:Restarting <Plugin>

[reconfigure]
command:/usr/local/opnsense/scripts/OPNsense/<Plugin>/generate_config.php && /bin/pkill -HUP -F /var/run/<plugin>.pid 2>/dev/null; exit 0
parameters:
type:script
message:Reconfiguring <Plugin>

[status]
command:if [ -f /var/run/<plugin>.pid ] && /bin/pgrep -qF /var/run/<plugin>.pid 2>/dev/null; then echo "<plugin> is running as pid $(cat /var/run/<plugin>.pid)."; else echo "<plugin> is not running."; fi
parameters:
type:script_output
message:Checking <Plugin> status
```

Action types:
- `script` — returns only exit status
- `script_output` — returns command stdout (used for status checks and data queries)
- `stream_output` — returns result in streaming mode

After modifying actions, restart configd: `service configd restart`

Test with: `configctl <plugin> <action>`

## Service Registration (`plugins.inc.d/<plugin>.inc`)

Registers the plugin with OPNsense's service framework:

```php
<?php

function <plugin>_services()
{
    $services = array();
    $mdl = new \OPNsense\<Plugin>\<Plugin>();

    if ((string)$mdl->enabled !== "1") {
        return $services;  // Don't register if disabled
    }

    $services[] = array(
        'description' => 'Plugin Description',
        'configd' => array(
            'restart' => array('<plugin> restart'),
            'start'   => array('<plugin> start'),
            'stop'    => array('<plugin> stop'),
        ),
        'name' => '<plugin>',
        'pidfile' => '/var/run/<plugin>.pid',
    );
    return $services;
}

function <plugin>_configure()
{
    return array(
        '<plugin>_start:0',  // Bootstrap hook: run at boot priority 0
    );
}

function <plugin>_syslog()
{
    return array(
        '<plugin>' => array('facility' => array('<plugin>')),
    );
}

function <plugin>_start()
{
    $mdl = new \OPNsense\<Plugin>\<Plugin>();
    if ((string)$mdl->enabled === "1") {
        configd_run('<plugin> start');
    }
}
```

Key functions:
- `<plugin>_services()` — registers service in UI Services widget; returns empty array if disabled
- `<plugin>_configure()` — registers bootstrap hook for autostart
- `<plugin>_syslog()` — registers syslog facility for log filtering in UI
- `<plugin>_start()` — called at boot; starts service if enabled
- The `pidfile` entry enables `pluginctl -s` to check service status

## Menu (`Menu/Menu.xml`)

Defines where the plugin appears in the web UI:

```xml
<menu>
    <Services>
        <<Plugin> VisibleName="Plugin Name" cssClass="fa fa-icon-name">
            <Settings order="10" url="/ui/<pluginlowercase>/general/index"/>
            <Status order="20" url="/ui/<pluginlowercase>/status/index"/>
            <LogFile VisibleName="Log File" order="30"
                     url="/ui/diagnostics/log/<plugin>"/>
        </<Plugin>>
    </Services>
</menu>
```

- Top-level tags: `System`, `Services`, `Interfaces`, `Firewall`, `VPN`, etc.
- `order` controls sort position within the section
- `VisibleName` overrides the tag name for display
- `cssClass` uses FontAwesome icon classes (e.g., `fa fa-line-chart`)
- Log viewer URL pattern: `/ui/diagnostics/log/<facility>` where underscores in the facility name become slashes (e.g., facility `hello_world` → URL `/ui/diagnostics/log/hello/world`)

## ACL (`ACL/ACL.xml`)

Defines access control permissions for non-admin users:

```xml
<acl>
    <page-services-<plugin>>
        <name>Services: Plugin Name</name>
        <patterns>
            <pattern>ui/<pluginlowercase>/*</pattern>
            <pattern>api/<pluginlowercase>/*</pattern>
        </patterns>
    </page-services-<plugin>>
</acl>
```

## Syslog Template

Optional syslog-ng filter for the log viewer:

```
filter f_local_<plugin> {
    program("<plugin>");
};
```

Place at: `src/opnsense/service/templates/OPNsense/Syslog/local/<plugin>.conf`

## Daemon Pattern

### Two-phase architecture (security)

1. **Root phase** (`generate_config.php`): Runs as root via configd, reads OPNsense model,
   validates config, writes a JSON config file readable by the daemon
2. **Unprivileged phase** (`<daemon>.php`): Runs as `nobody` via `daemon(8)`, reads the
   JSON config file, does the actual work

### Privilege separation

The daemon should run as `nobody` using `daemon -u nobody`. This requires three things:

1. **`-u nobody` in configd actions**: Add the flag to `[start]` and `[restart]` commands
2. **`chmod 644` on the pidfile**: `daemon(8)` creates the pidfile as root before switching
   user, so the web UI (running as `www`) can't read it for status checks. Fix by appending
   `&& chmod 644 /var/run/<plugin>.pid` after the daemon command in the configd action.
3. **Ownership fix in `generate_config.php`**: When upgrading from a version that ran as root,
   existing output files will be owned by root and the `nobody` daemon can't overwrite them.
   Fix ownership in the config generator (which runs as root):

```php
// Fix ownership of existing output files for upgrade from root-daemon version
foreach (glob($outputdir . '*.prom') as $file) {
    chown($file, 'nobody');
    chgrp($file, 'nobody');
}
```

### Config generator (`generate_config.php`)

```php
#!/usr/local/bin/php
<?php
require_once("config.inc");

$mdl = new \OPNsense\<Plugin>\<Plugin>();
$config = [];

// Read each model field, validate, and add to config array
// Example for a text field:
$myfield = (string)$mdl->myfield;
if (!empty($myfield)) {
    $config['myfield'] = $myfield;
}

$config_path = '/usr/local/etc/<plugin>.conf';
file_put_contents($config_path, json_encode($config, JSON_PRETTY_PRINT) . "\n");
chmod($config_path, 0644);
```

Must have execute permission (`chmod +x`). Without it, configd returns error 126.

### Daemon script

The daemon runs as `nobody` and cannot access `config.xml`, so it does not require `config.inc`.
It reads configuration from the JSON file written by `generate_config.php`.

```php
#!/usr/local/bin/php
<?php

openlog("<plugin>", LOG_DAEMON, LOG_LOCAL4);

$running = true;
$reconfigure = true;

pcntl_signal(SIGTERM, function() use (&$running) { $running = false; });
pcntl_signal(SIGINT,  function() use (&$running) { $running = false; });
pcntl_signal(SIGHUP,  function() use (&$reconfigure) { $reconfigure = true; });

while ($running) {
    pcntl_signal_dispatch();

    if ($reconfigure) {
        $config = json_decode(file_get_contents('/usr/local/etc/<plugin>.conf'), true);
        $reconfigure = false;
    }

    // Do work here...
    // Use syslog() for logging: syslog(LOG_NOTICE, 'message');

    // If writing output files, use atomic write (prevents partial reads):
    // $tmp = $path . '.tmp.' . getmypid();
    // file_put_contents($tmp, $data);
    // chmod($tmp, 0644);
    // rename($tmp, $path);  // Atomic on POSIX

    sleep($config['interval'] ?? 60);
}

closelog();
```

Signal handling:
- `SIGHUP` — reload config without restart (sent by `[reconfigure]` action)
- `SIGTERM`/`SIGINT` — graceful shutdown

### Minimal autoloader for unprivileged scripts

When the daemon runs as `nobody`, it can't `require_once("config.inc")` because that needs
read access to `/conf/config.xml` (root-only). If your daemon or its modules need OPNsense
framework classes (e.g., `OPNsense\Core\Backend` for configd calls), use a minimal PSR-4
autoloader instead:

```php
<?php
// lib/autoload.php — Minimal autoloader for OPNsense MVC library classes.
// Replaces config.inc bootstrap so scripts can run as nobody.
spl_autoload_register(function ($class) {
    $path = '/usr/local/opnsense/mvc/app/library/'
        . str_replace('\\', '/', $class) . '.php';
    if (file_exists($path)) {
        require_once $path;
    }
});
```

This loads only the classes actually used (typically just `OPNsense\Core\Backend` and its
dependencies) without pulling in the full OPNsense bootstrap.

### Using configd Backend from scripts

Unprivileged scripts can call configd actions through the `OPNsense\Core\Backend` class,
which communicates with configd via a Unix socket. This provides a privilege boundary —
scripts don't need root to gather system data.

```php
require_once __DIR__ . '/lib/autoload.php';  // Minimal autoloader (see above)

$backend = new \OPNsense\Core\Backend();

// Run a configd action and get output
$raw = trim($backend->configdRun('interface gateways status'));
$data = json_decode($raw, true);

// Run with parameters
$result = $backend->configdpRun('filter rule', ['param1', 'param2']);
```

Key points:
- `configdRun('<plugin> <action>')` — run an action, return stdout
- `configdpRun('<plugin> <action>', [params])` — run with positional parameters
- The configd socket is accessible by `nobody`, so no root needed
- Use this pattern when daemon modules need live system data (gateway status, firewall
  counters, interface info, etc.)

### Modular script architecture

For plugins with multiple data sources (e.g., a metrics exporter with per-subsystem collectors),
use auto-discovery to load modules:

```php
// lib/collector_loader.php
function load_collectors(string $dir): array
{
    $collectors = [];
    foreach (glob($dir . '/*Collector.php') as $file) {
        $class = basename($file, '.php');
        $type = strtolower(preg_replace('/Collector$/', '', $class));
        require_once $file;
        if (class_exists($class, false)) {
            $collectors[$type] = $class;
        }
    }
    return $collectors;
}
```

Each module follows an implicit interface with static methods:

```php
class GatewayCollector
{
    public static function name(): string { return 'Gateways'; }
    public static function defaultEnabled(): bool { return true; }
    public static function collect(): string { /* return output */ }
    public static function status(): array { /* return status data */ }
}
```

To make modules individually enable/disable-able via the UI, store per-module settings as a
JSON-encoded string in a `TextField` model field:

```xml
<collectors type="TextField">
    <Default>{}</Default>
</collectors>
```

The config generator merges discovered modules with user overrides:

```php
$all_collectors = load_collectors(COLLECTORS_DIR);
$overrides = json_decode($mdl->collectors->__toString(), true) ?: [];
$config = [];
foreach ($all_collectors as $type => $class) {
    $config[$type] = isset($overrides[$type]) ? (bool)$overrides[$type] : $class::defaultEnabled();
}
```

## Packaging

### Build system

The OPNsense plugin build system (`plugins.mk`) provides a `package` target that:

1. **install** — copies `src/` into staging dir (`work/src/usr/local/...`), writes version JSON
2. **metadata** — generates `+MANIFEST`, `+DESC`, `plist`, and install/deinstall scripts
3. **pkg create** — calls `pkg create -v -m <staging> -r <staging> -p <plist> -o <pkgdir>`

### Auto-generated post-install script

The `+POST_INSTALL` script is generated automatically based on what directories exist in `src/`:

- If `src/opnsense/service/conf/actions.d/` exists → restarts configd
- If `src/opnsense/mvc/app/models/` exists → runs model migrations and reloads plugin config
- If `src/opnsense/service/templates/` exists → reloads templates (e.g., syslog)

### Remote build with `build.sh`

Since `bmake` and `pkg create` require FreeBSD, the build typically runs on an OPNsense
firewall via SSH. A `build.sh` script automates this:

1. Uploads `Mk/`, `Templates/`, `Scripts/` from `tmp/plugins/` plus plugin source
2. Creates directory structure so `../../Mk/plugins.mk` resolves from the plugin Makefile
3. Runs `sudo bmake package` via SSH (sudo needed for `pkg` dependency checks)
4. Downloads `.pkg` to `dist/`
5. Cleans up with `sudo rm -rf` (build creates root-owned files)

### Installing a package

```sh
pkg install ./os-<plugin>-<version>.pkg    # Install with dependency checks
pkg add ./os-<plugin>-<version>.pkg        # Install without dependency checks
pkg delete -y os-<plugin>                  # Remove
```

### Package hook scripts

The build system supports hook scripts that run during package install/remove. Place these
files at the plugin root (same level as Makefile):

| File | When it runs |
|------|-------------|
| `+PRE_INSTALL.pre` | Before file extraction (runs on both fresh install and upgrade) |
| `+POST_INSTALL.post` | After file extraction, appended to auto-generated post-install |
| `+PRE_DEINSTALL.pre` | Before file removal on `pkg delete` |

Critical: `pkg install` over an existing installation does NOT run deinstall hooks — only
`+PRE_INSTALL.pre` and `+POST_INSTALL.post` run. Handle upgrade cleanup in `+PRE_INSTALL.pre`,
not `+PRE_DEINSTALL.pre`.

### Plugin registration

For plugins installed outside the OPNsense UI (via `pkg install` on the command line or from
a custom repo), the plugin must register itself with the firmware system. Without registration,
the plugin shows as "(misconfigured)" in Firmware > Plugins.

**`+POST_INSTALL.post`** — register on install:

```sh
REGISTER="/usr/local/opnsense/scripts/firmware/register.php"
VERSION_INFO="/usr/local/opnsense/version/<plugin>"
if [ -f "$REGISTER" ] && [ -f "$VERSION_INFO" ]; then
    PLUGIN_ID=$(sed -n 's/.*"product_id"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$VERSION_INFO")
    if [ -n "$PLUGIN_ID" ]; then
        $REGISTER install "$PLUGIN_ID"
    fi
fi
```

**`+PRE_DEINSTALL.pre`** — deregister on removal:

```sh
REGISTER="/usr/local/opnsense/scripts/firmware/register.php"
VERSION_INFO="/usr/local/opnsense/version/<plugin>"
if [ -f "$REGISTER" ] && [ -f "$VERSION_INFO" ]; then
    PLUGIN_ID=$(sed -n 's/.*"product_id"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p' "$VERSION_INFO")
    if [ -n "$PLUGIN_ID" ]; then
        $REGISTER remove "$PLUGIN_ID"
    fi
fi
```

The `VERSION_INFO` file is auto-generated by the build system at
`/usr/local/opnsense/version/<plugin>` and contains the `product_id`.

### Hosting a package repository

The Firmware > Plugins "i" (info) button uses `pkg rquery` to fetch package details from a
remote repository. Without a hosted repo, clicking "i" shows "details not available".

**Build step** — copy the `.pkg` to a repo directory and index it with `pkg repo`:

```sh
PAGES_REPO="$REPO_ROOT/docs/repo"
rm -f "$PAGES_REPO"/os-<plugin>*.pkg
cp "$BUILT_PKG" "$PAGES_REPO/"
pkg repo "$PAGES_REPO/"
```

After `pkg repo`, the directory contains: `meta.conf`, `packagesite.pkg`, `packagesite.tzst`,
`data.pkg`, `data.tzst`, `filesite.yaml`, plus the `.pkg` file itself.

**GitHub Pages** — serve `docs/repo/` via GitHub Pages (master branch, `/docs` path).
Requires a `.nojekyll` file in `docs/` root so directories starting with `_` are served.

**`+POST_INSTALL.post`** — add the repo config on the firewall:

```sh
REPO_CONF="/usr/local/etc/pkg/repos/<plugin>.conf"
if [ ! -f "$REPO_CONF" ]; then
    cat > "$REPO_CONF" <<REPOEOF
<plugin>: {
  url: "https://<user>.github.io/<repo>/repo",
  enabled: yes
}
REPOEOF
    pkg update -f -r <plugin>
fi
```

**`+PRE_DEINSTALL.pre`** — remove the repo config:

```sh
rm -f /usr/local/etc/pkg/repos/<plugin>.conf
```

This is the same pattern used by community repos (mimugmail, repo-mihak).

### PLUGIN_WWW

The `PLUGIN_WWW` Makefile variable sets the pkg `www` metadata field (distinct from the
`WWW:` line in `pkg-descr`, which is for display text). If not set, it defaults to
`https://opnsense.org/`.

```makefile
PLUGIN_WWW=         https://github.com/user/repo
```

## Deployment (Manual, without package)

For development iteration without rebuilding the package:

```sh
# Upload source
scp -r src/ firewall:/tmp/deploy/

# Copy to /usr/local/
ssh firewall "sudo cp -r /tmp/deploy/src/* /usr/local/"

# Full post-install sequence (all steps required):
ssh firewall "sudo service configd restart"
ssh firewall "sudo /usr/local/opnsense/mvc/script/run_migrations.php OPNsense/<Plugin>"
ssh firewall "sudo /usr/local/etc/rc.configure_plugins POST_INSTALL"
ssh firewall "sudo configctl <plugin> start"
```

Step 3 (`rc.configure_plugins POST_INSTALL`) is **critical** — without it, menus and ACLs
won't appear even if you manually delete cache files.

## Coding Standards

- **BSD 2-Clause license header** required in all PHP/inc files (mandatory for OPNsense upstream submission)
- **PSR-12** coding standard recommended (max line length 120 chars)
- Check with: `phpcs --standard=PSR12 src/`
- Daemon scripts will have one unavoidable PSR-12 warning (side effects + declarations in same file)
- **"OPN" prefix** is reserved for OPNsense business edition plugins — do not use it

## Common Pitfalls

- **Error 126 from configd**: Script missing execute permission (`chmod +x`)
- **Menu not appearing**: Run `rc.configure_plugins POST_INSTALL` to flush caches
- **Status shows "not running" in UI**: The `[status]` configd action must return text
  containing exactly `"is running"` or `"not running"`
- **API returns empty model**: Check `$internalModelName` matches form field `<id>` prefix
- **Service not in Services widget**: `<plugin>_services()` returns empty array when disabled;
  enable the plugin first
- **Config not applied after save**: The save button must call the `reconfigure` endpoint,
  which regenerates config and sends SIGHUP to the daemon
- **Daemon not writing output**: Check file/directory permissions; daemon runs as `nobody`
- **Package builds but service won't start after install**: Verify all scripts under
  `src/opnsense/scripts/` have `+x` in the source repo before building
- **`pkg: No package(s) matching`**: Build needs `sudo bmake package` for dependency resolution
- **Enabled checkbox unchecked on fresh install**: Set `<Default>1</Default>` on the `enabled` field
- **Old plugin files after rename**: Manually remove old .inc, scripts dir, actions conf, syslog conf,
  and old model entry from `/conf/config.xml`, then run `rc.configure_plugins POST_INSTALL`
- **Pidfile not readable by web UI**: Status shows "not running" even when running. Fix:
  `chmod 644 /var/run/<plugin>.pid` after daemon start in the configd action
- **Plugin shows "(misconfigured)" in Firmware > Plugins**: Missing `register.php install`
  call in `+POST_INSTALL.post` (see Plugin Registration section)
- **Daemon can't write output files after upgrade**: Files owned by root from a previous
  version that ran as root. Fix: `chown`/`chgrp` to `nobody` in `generate_config.php`
- **`pkg install` over existing doesn't run deinstall hooks**: Only `+PRE_INSTALL.pre` and
  `+POST_INSTALL.post` run on upgrade. Handle upgrade cleanup in `+PRE_INSTALL.pre`
