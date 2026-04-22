# **Creating and Maintaining DDEV Add-Ons**
### with

<img src="images/ddev-logo.svg" alt="DDEV Logo" class="ddev-logo">

---

## What Are Add-Ons?

Extend DDEV with additional services, commands, and configuration

* Services: Redis, Elasticsearch, Solr, Mailpit, and more
* Custom commands, Docker images, configuration files
* Community-contributed — browse at [addons.ddev.com](https://addons.ddev.com)

```bash
ddev add-on get ddev/ddev-redis
ddev add-on list --all
```

---

## Quick Start: Use the Template

1. Go to [github.com/ddev/ddev-addon-template](https://github.com/ddev/ddev-addon-template)
2. Click **"Use this template"**
3. Name it with a `ddev-` prefix (e.g. `ddev-foobar`)
4. Add a meaningful description with keywords

Key files:
* `install.yaml` — defines what gets installed and how
* `tests/testdata/` + `tests.bats` — automated test suite
* `docker-compose.addon-template.yaml` — service definition
* GitHub Actions CI: runs tests on push and nightly

---

## install.yaml Structure

```yaml
name: my-addon
ddev_version_constraint: ">= v1.24.10"
pre_install_actions: []
project_files: []
global_files: []
post_install_actions: []
removal_actions: []
```

* `project_files` → copied to project's `.ddev/`
* `global_files` → copied to `~/.ddev/`
* Actions run on the host (or in the web container for PHP)
* Advanced: `dependencies`, `yaml_read_files`

---

## The `#ddev-generated` Directive

**Critical for proper cleanup.**

Add `#ddev-generated` to any file your add-on creates:

```yaml
# docker-compose.myservice.yaml
#ddev-generated
services:
  myservice:
    image: myimage:latest
```

* DDEV tracks these files during installation
* `ddev add-on remove` deletes them automatically
* Files **without** this comment are left in place on removal

---

## Action Types: Bash

Bash actions run on the **host system** — not inside a container

* Use for: permissions, env setup, simple file operations
* Always `#!/usr/bin/env bash` for portability
* Must be **idempotent** — safe to run multiple times
* `#ddev-description:` shows progress during install

```yaml
post_install_actions:
  - |
    #ddev-description: Configure project settings
    #!/usr/bin/env bash
    chmod +x .ddev/commands/web/mycommand
    echo "Setup complete for $DDEV_PROJECT"
```

---

## Action Types: PHP (New!)

* Built-in `php-yaml` — reliable YAML read/write
* Cross-platform — no shell scripting quirks
* Rich string processing and conditional logic
* Detected by the `<?php` opening tag; runs in web container

```yaml
post_install_actions:
  - |
    <?php
    #ddev-description: Process project configuration
    $projectType = $_ENV['DDEV_PROJECT_TYPE'];
    $projectName = $_ENV['DDEV_PROJECT'];
    echo "Setting up $projectType project: $projectName\n";
```

---

## PHP Actions: Available Env Vars

```php
<?php
// Project
$_ENV['DDEV_PROJECT']         // project name
$_ENV['DDEV_PROJECT_TYPE']    // 'drupal', 'wordpress', 'laravel', etc.
$_ENV['DDEV_APPROOT']         // '/var/www/html'
$_ENV['DDEV_DOCROOT']         // 'web', 'public', etc.
$_ENV['DDEV_TLD']             // 'ddev.site' or configured TLD
// Stack
$_ENV['DDEV_PHP_VERSION']     // '8.1', '8.2', '8.3', etc.
$_ENV['DDEV_DATABASE']        // 'mysql:8.0', 'postgres:16', etc.
$_ENV['DDEV_DATABASE_FAMILY'] // 'mysql', 'postgres'
$_ENV['DDEV_WEBSERVER_TYPE']  // 'nginx-fpm', 'apache-fpm'
// System
$_ENV['DDEV_VERSION']         // current DDEV version
$_ENV['DDEV_MUTAGEN_ENABLED'] // 'true' or 'false'
```

Working directory: `/var/www/html/.ddev`

---

## PHP Actions: Conditional Config Example

```php
<?php
#ddev-description: Generate service configuration
$projectType = $_ENV['DDEV_PROJECT_TYPE'];
$services = [];
if ($projectType === 'drupal') {
    $services['redis'] = ['image' => 'redis:7-alpine'];
} elseif ($projectType === 'wordpress') {
    $services['memcached'] = ['image' => 'memcached:alpine'];
}
if (!empty($services)) {
    $content = "#ddev-generated\n" . yaml_emit(['services' => $services]);
    file_put_contents('docker-compose.cache.yaml', $content);
    echo "Generated config for $projectType\n";
}
```

For complex logic: use separate files in `myservice/scripts/setup.php`

---

## Mixed Bash + PHP

A single add-on can use both action types — choose per-action:

```yaml
pre_install_actions:
  - |
    #ddev-description: Set file permissions
    #!/usr/bin/env bash
    chmod +x .ddev/commands/web/mycommand

post_install_actions:
  - |
    <?php
    #ddev-description: Process configuration
    $projectName = $_ENV['DDEV_PROJECT'];
    // Generate YAML config files...
```

---

## Dependencies

**Static dependencies** — declared in `install.yaml`:

```yaml
dependencies:
  - ddev/ddev-redis          # GitHub repository
  - ddev/ddev-elasticsearch
```

* Automatically installed when the add-on is installed
* Skip with: `ddev add-on get --skip-deps my-addon`
* DDEV detects and prevents circular dependencies

**Runtime dependencies** (advanced) — for add-ons that analyze project files:

```php
file_put_contents('.runtime-deps-my-addon', "ddev/ddev-redis\n");
// DDEV processes this after all installation phases complete
```

See [ddev-upsun](https://github.com/ddev/ddev-upsun) for a real-world example

---

## Testing with Bats

The template includes `tests.bats` — run on every push and nightly:

```bash
#!/usr/bin/env bats
@test "install add-on and verify service" {
  ddev add-on get .
  ddev restart
  curl -sf "http://localhost:$(ddev port myservice 8080)/health"
}
@test "remove add-on cleanly" {
  ddev add-on remove my-addon
  [ ! -f .ddev/docker-compose.myservice.yaml ]
}
```

Use `bats-support` and `bats-assert` for cleaner assertions

---

## Publishing Your Add-on

1. Add the **`ddev-get`** GitHub topic to your repository
   * Appears in `ddev add-on list --all` immediately
   * Listed on addons.ddev.com within ~24 hours
2. Create releases with **semantic versioning** (v1.0.0, v1.1.0, ...)
3. Include registry badges in your README
4. Write clear documentation with usage examples

To become an **officially supported** add-on:
* Open an issue in the DDEV repository requesting upgrade
* Commit to maintaining it and responding to issues

---

## Maintenance Practices

* **Track DDEV releases** — subscribe to ddev/ddev, read changelogs
* **Run the update checker script** periodically
* **Study official add-ons** — they set the pattern
* **Update** `ddev_version_constraint: ">= v1.24.10"` in `install.yaml`
* Watch [ddev-addon-template](https://github.com/ddev/ddev-addon-template) for template changes
* Publish a release after every substantive update

---

## Repository Settings

* Squash merge only — PR title as commit message
* Auto-delete head branches after merge
* Require pull requests before merging on `main`
* Block force pushes and restrict deletions
* Disable unused features: Wikis, Discussions, Projects
* Add issue and PR templates

---

## Advanced Features

* `dockerfile_inline` — minor Docker image tweaks without a separate Dockerfile
* `ddev dotenv set` — manage env vars in `.ddev/.env.*` files
* `x-ddev.describe-*` — customize `ddev describe` output (v1.24.10+)
* `x-ddev.ssh-shell` — set a custom shell for `ddev ssh -s my-service`
* Docker Compose `profiles:` — optional services users can enable
* `MutagenSync` annotation — correct file sync for file-modifying commands

---

## Getting Help & Resources

* **DDEV Discord** — [ddev.com/s/discord](https://ddev.com/s/discord) — #add-ons channel
* **Add-on Registry** — [addons.ddev.com](https://addons.ddev.com)
* **Template** — [github.com/ddev/ddev-addon-template](https://github.com/ddev/ddev-addon-template)
* **Docs** — [docs.ddev.com/users/extend/creating-add-ons/](https://docs.ddev.com/en/stable/users/extend/creating-add-ons/)
* **Maintenance Guide** — [ddev.com/blog/ddev-add-on-maintenance-guide/](https://ddev.com/blog/ddev-add-on-maintenance-guide/)
* **GitHub Issues** — [github.com/ddev/ddev/issues](https://github.com/ddev/ddev/issues)
