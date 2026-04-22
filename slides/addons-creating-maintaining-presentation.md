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

> Note: `ddev get` was deprecated in v1.23.5 — use `ddev add-on get`

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

dependencies: []
yaml_read_files: {}
```

* `project_files` → copied to project's `.ddev/`
* `global_files` → copied to `~/.ddev/`
* Actions run on the host (or in the web container for PHP)

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

Traditional bash actions run on the **host system**

Use for: file permissions, environment setup, simple file operations

```yaml
post_install_actions:
  - |
    #ddev-description: Configure project settings
    #!/usr/bin/env bash
    chmod +x .ddev/commands/web/mycommand
    echo "Setup complete for $DDEV_PROJECT"
```

Rules:
* Always use `#!/usr/bin/env bash` for portability
* Must be **idempotent** (safe to run multiple times)
* Use `#ddev-description:` for progress display

---

## Action Types: PHP (New!)

PHP actions for complex logic and configuration processing

```yaml
post_install_actions:
  - |
    <?php
    #ddev-description: Process project configuration
    $projectType = $_ENV['DDEV_PROJECT_TYPE'];
    $projectName = $_ENV['DDEV_PROJECT'];

    echo "Setting up $projectType project: $projectName\n";
    ```

Why PHP?
* Built-in `php-yaml` extension — reliable YAML read/write
* Cross-platform (no shell scripting differences between macOS/Linux/Windows)
* Rich string manipulation and data processing
* DDEV detects PHP actions by the `<?php` opening tag

---

## PHP Actions: Available Env Vars

```php
<?php
// Project information
$_ENV['DDEV_PROJECT']        // project name
$_ENV['DDEV_PROJECT_TYPE']   // 'drupal', 'wordpress', 'laravel', etc.
$_ENV['DDEV_APPROOT']        // '/var/www/html'
$_ENV['DDEV_DOCROOT']        // 'web', 'public', etc.
$_ENV['DDEV_TLD']            // 'ddev.site' or configured TLD

// Technology stack
$_ENV['DDEV_PHP_VERSION']    // '8.1', '8.2', '8.3', etc.
$_ENV['DDEV_DATABASE']       // 'mysql:8.0', 'postgres:16', etc.
$_ENV['DDEV_DATABASE_FAMILY']// 'mysql', 'postgres'
$_ENV['DDEV_WEBSERVER_TYPE'] // 'nginx-fpm', 'apache-fpm'

// System
$_ENV['DDEV_VERSION']        // current DDEV version
$_ENV['DDEV_MUTAGEN_ENABLED']// 'true' or 'false'
```

Working directory: `/var/www/html/.ddev`

---

## PHP Actions: Conditional Config Example

```php
<?php
#ddev-description: Generate service configuration
$projectType = $_ENV['DDEV_PROJECT_TYPE'];
$services = [];

switch ($projectType) {
    case 'drupal':
        $services['redis'] = ['image' => 'redis:7-alpine'];
        break;
    case 'wordpress':
        $services['memcached'] = ['image' => 'memcached:alpine'];
        break;
}

if (!empty($services)) {
    $content = "#ddev-generated\n" . yaml_emit(['services' => $services]);
    file_put_contents('docker-compose.cache.yaml', $content);
    echo "Generated cache config for $projectType\n";
}
```

For complex logic, use separate PHP script files in a namespaced directory: `myservice/scripts/setup.php`

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

**Runtime dependencies** (advanced) — for add-ons that must analyze project files to determine what's needed:

```php
// In a post_install_action, write a file:
file_put_contents('.runtime-deps-my-addon', "ddev/ddev-redis\n");
// DDEV processes this file after all installation phases complete
```

See [ddev-upsun](https://github.com/ddev/ddev-upsun) for a real-world example

---

## Testing with Bats

The template includes `tests.bats` — run on every push and nightly:

```bash
#!/usr/bin/env bats

@test "install add-on and start" {
  ddev add-on get .
  ddev restart
  health_check_url="http://localhost:$(ddev port myservice 8080)/health"
  curl -sf "$health_check_url"
}

@test "verify service responds" {
  ddev exec curl -s http://myservice:8080/
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

Keep your add-on healthy over time:

* **Track DDEV releases** — subscribe to the ddev/ddev repo, read changelogs
* **Run the update checker script** periodically
* **Study official add-ons** — they set the pattern
* **Update version constraint** in `install.yaml`:

```yaml
ddev_version_constraint: ">= v1.24.10"
```

* Watch for template changes at [ddev-addon-template](https://github.com/ddev/ddev-addon-template)
* Publish a release after every substantive update

---

## Repository Settings

Recommended GitHub repository configuration:

**Merging:**
* Enable squash merging with PR title as commit message
* Auto-delete head branches after merge
* Disable merge commits and rebase merging

**Branch protection on `main`:**
* Require pull requests before merging
* Block force pushes
* Restrict deletions

**Housekeeping:**
* Disable unused features: Wikis, Discussions, Projects
* Add issue and PR templates for contribution quality

---

## Advanced Features

```yaml
# Minimal Docker image customization (no full Dockerfile needed)
dockerfile_inline: |
  RUN apt-get install -y mypackage

# Manage environment variables
# ddev dotenv set .ddev/.env.myservice --myservice-api-key=abc123

# Customize ddev describe output (v1.24.10+)
# x-ddev.describe-myservice: "Running at https://${DDEV_HOSTNAME}"

# Custom SSH shell
# x-ddev.ssh-shell: /bin/zsh

# Docker Compose optional profiles
# profiles: [myservice]

# Mark file-modifying commands for Mutagen
# annotations:
#   com.ddev.io.mutagensync: "true"
```

---

## Getting Help & Resources

* **DDEV Discord** — [discord.gg/hCZFfAMc5k](https://discord.gg/hCZFfAMc5k) — #add-ons channel
* **Add-on Registry** — [addons.ddev.com](https://addons.ddev.com)
* **Template** — [github.com/ddev/ddev-addon-template](https://github.com/ddev/ddev-addon-template)
* **Docs** — [docs.ddev.com/users/extend/creating-add-ons/](https://docs.ddev.com/en/stable/users/extend/creating-add-ons/)
* **Maintenance Guide** — [ddev.com/blog/ddev-add-on-maintenance-guide/](https://ddev.com/blog/ddev-add-on-maintenance-guide/)
* **GitHub Issues** — [github.com/ddev/ddev/issues](https://github.com/ddev/ddev/issues)

---

## Suggested Docs Fixes

Issues found while preparing this presentation:

**docs.ddev.com/users/extend/creating-add-ons/**
* `#ddev-generated` directive not explained — critical for removal
* `removal_actions` listed but no example provided
* `yaml_read_files` mixes Go template and PHP syntax confusingly
* PHP action execution environment (web container vs host) not clearly stated
* `ddev get` deprecation not noted
* Missing: `ddev describe` customization, `dockerfile_inline`

**ddev.com/blog/ddev-add-on-maintenance-guide/**
* PHP actions not mentioned as a maintenance upgrade path
* `#ddev-generated` purpose not explained

**ddev-addon-template README:**
* Uses `pre_install_commands`/`post_install_commands` but `install.yaml` uses `pre_install_actions`/`post_install_actions` — one is outdated
