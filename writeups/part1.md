
# Part 1 — YAML & Cloud-Init Fundamentals

## Objective

This part covers the fundamentals required before implementing the
Cloud-Init → Butane transpiler.

The focus was on:

- YAML
- Linux provisioning
- Cloud-Init
- Cloud-Init users
- write_files
- packages
- runcmd
- Provisioning concepts relevant to the transpiler

---

# 1. Project Context

The target project is a Cloud-Init → Butane YAML configuration
transpiler for Flatcar Container Linux.

The problem is that existing Cloud-Init configurations used for
traditional Linux systems cannot be directly used with Flatcar.

The intended workflow is:

    Cloud-Init YAML
          ↓
    Cloud-Init → Butane Transpiler
          ↓
    Butane YAML
          ↓
    Butane CLI
          ↓
    Ignition JSON
          ↓
    Flatcar Container Linux

The transpiler therefore needs to understand Cloud-Init as its
INPUT format and Butane as its OUTPUT format.

---

# 2. YAML Fundamentals

YAML is a human-readable data serialization format commonly used
for configuration files.

Important concepts studied:

- Key-value pairs
- Lists
- Nested structures
- Indentation
- Strings
- Booleans
- Numbers
- Multi-line strings
- Comments

Example:

```yaml
server:
  name: web-server
  port: 8080

packages:
  - nginx
  - curl
  - git
````

YAML indentation is significant.

For example:

```yaml
user:
  name: admin
  shell: /bin/bash
```

Here `name` and `shell` belong to the `user` object.

---

# 3. YAML Validation

Because the project processes YAML configurations, validating
the input is important.

Tools used during the learning process included:

```bash
yamllint
yq
jq
```

Example:

```bash
yamllint config.yaml
```

`yamllint` can detect formatting and syntax-related YAML problems.

`yq` can be used to query and manipulate YAML data.

`jq` is useful when working with JSON output.

These tools are useful for validating and debugging configuration
files during transpiler development.

---

# 4. Linux Provisioning

Provisioning means preparing a machine automatically instead of
manually configuring every component.

A provisioning configuration may define:

* Users
* SSH access
* Files
* Permissions
* Software
* Services
* Startup commands

Conceptually:

```
Configuration
      ↓
Provisioning System
      ↓
Configured Linux Machine
```

This is the foundation behind both Cloud-Init and the
Cloud-Init → Butane project.

---

# 5. Cloud-Init Introduction

Cloud-Init is a system used to initialize and configure cloud
instances during their startup process.

A basic Cloud-Init configuration starts with:

```yaml
#cloud-config
```

Example:

```yaml
#cloud-config

users:
  - name: admin
    shell: /bin/bash

packages:
  - nginx
  - curl

runcmd:
  - systemctl enable nginx
  - systemctl start nginx
```

Cloud-Init provides many directives that allow a machine to be
configured automatically.

For this project, the most important directives are:

* `users`
* `write_files`
* `packages`
* `runcmd`

---

# 6. Cloud-Init Users

The `users` directive is used to define users during provisioning.

Example:

```yaml
users:
  - name: admin
    shell: /bin/bash
    groups:
      - sudo
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...
```

Important user properties studied include:

* `name`
* `shell`
* `uid`
* `gecos`
* `homedir`
* `groups`
* `primary_group`
* `ssh_authorized_keys`
* `passwd`
* `system`
* `no_user_group`

The important idea for the transpiler is that Cloud-Init user
fields must eventually be translated into their corresponding
Butane fields.

---

# 7. SSH Keys & Security

SSH public keys are preferred over passwords for production-style
server access.

Example:

```yaml
ssh_authorized_keys:
  - ssh-ed25519 AAAA...
```

The transpiler must preserve SSH key information when a compatible
Butane representation exists.

Security-related user options were also examined, including
password hashes, groups and account configuration.

---

# 8. Cloud-Init write_files

The `write_files` directive allows files to be created automatically
during provisioning.

Example:

```yaml
write_files:
  - path: /etc/myapp/config.yaml
    permissions: "0644"
    content: |
      app_name: myapp
      port: 8080
```

A file definition can contain:

* `path`
* `permissions`
* `owner`
* `content`
* `encoding`

The transpiler must eventually map these concepts to Butane's
storage configuration.

---

# 9. File Permissions

Linux file permissions are important during automated provisioning.

Common examples:

```text
0644 → normal configuration files
0600 → sensitive files/secrets
0755 → executable scripts
```

Example:

```yaml
write_files:
  - path: /opt/app/.env
    permissions: "0600"
    content: |
      PASSWORD=secret
```

The permission information must not be lost during conversion.

This becomes especially important because Butane represents the
permission value differently.

---

# 10. Encoded File Content

Cloud-Init can represent file contents using different encodings.

For example, base64-encoded content can be specified using an
encoding field.

This is important for the transpiler because encoded Cloud-Init
content may need to become a Butane `contents.source` value rather
than a normal inline string.

The Day 3 notes identify this as a critical mapping:

```text
Cloud-Init:
write_files + encoding: b64

        ↓

Butane:
contents.source:
  data:;base64,...
```

---

# 11. Cloud-Init Packages

The `packages` directive is used to install software.

Example:

```yaml
packages:
  - nginx
  - git
  - curl
```

Cloud-Init also provides:

```yaml
package_update: true
package_upgrade: true
package_reboot_if_required: true
```

Conceptually:

```text
package_update
      ↓
refresh package database

package_upgrade
      ↓
upgrade installed packages

packages
      ↓
install requested software
```

However, packages become a major compatibility issue when
translating to Flatcar because Flatcar follows an immutable OS
model and does not provide the same traditional package-management
workflow.

Therefore, `packages` cannot simply be copied into Butane.

---

# 12. Cloud-Init runcmd

The `runcmd` directive executes commands during provisioning.

Example:

```yaml
runcmd:
  - systemctl enable nginx
  - systemctl start nginx
```

It can also execute scripts:

```yaml
runcmd:
  - /usr/local/bin/app-init.sh
```

`runcmd` is powerful because it can perform arbitrary actions.

However, this also makes it one of the hardest directives to
translate automatically.

---

# 13. Idempotency

An important provisioning concept studied was idempotency.

An idempotent operation can safely be executed multiple times
without producing unwanted results.

Good example:

```bash
mkdir -p /opt/myapp
```

Potentially problematic example:

```bash
mkdir /opt/myapp
```

The first command does not fail simply because the directory
already exists.

Idempotent provisioning is important because configuration
operations should be predictable and safe.

---

# 14. write_files + runcmd Pattern

A common Cloud-Init pattern is:

```yaml
write_files:
  - path: /usr/local/bin/init.sh
    permissions: "0755"
    content: |
      #!/bin/bash
      echo "Initializing..."

runcmd:
  - /usr/local/bin/init.sh
```

The idea is:

```text
write_files
    ↓
Place the script

runcmd
    ↓
Execute the script
```

This pattern becomes important later because Butane does not have
a direct equivalent to arbitrary Cloud-Init `runcmd`.

The migration strategy can instead use:

```text
Cloud-Init:

write_files + runcmd

        ↓

Butane:

storage.files + systemd.units
```

---

# 15. Why packages and runcmd Are Difficult

Two of the most important migration challenges identified were:

### packages

Flatcar is immutable and does not follow the traditional
package-manager model used by systems such as Ubuntu.

Therefore:

```text
Cloud-Init packages
        ↓
No simple direct Butane equivalent
        ↓
Alternative / warning required
```

### runcmd

Butane does not provide a generic arbitrary-shell-command field
equivalent to Cloud-Init `runcmd`.

Depending on the command, it may be possible to translate it into:

```text
systemd oneshot service
```

or specialized Butane structures such as:

```text
storage.directories
storage.links
```

Complex commands may instead require a warning.

---

# 16. Example Cloud-Init Configuration

A simplified configuration combining the concepts studied:

```yaml
#cloud-config

users:
  - name: appuser
    shell: /bin/bash
    groups:
      - sudo

write_files:
  - path: /opt/myapp/config.yaml
    permissions: "0644"
    content: |
      app_name: myapp
      port: 8080

  - path: /opt/myapp/start.sh
    permissions: "0755"
    content: |
      #!/bin/bash
      echo "Starting myapp..."

packages:
  - curl

runcmd:
  - /opt/myapp/start.sh
```

This example demonstrates the four major Cloud-Init areas studied:

```text
users
  ↓
Identity

write_files
  ↓
Files

packages
  ↓
Software

runcmd
  ↓
Actions
```

---

# 17. Initial Mapping Understanding

The first mapping model developed during this part was:

| Cloud-Init               | Future Butane Direction     |
| ------------------------ | --------------------------- |
| `users`                  | `passwd.users`              |
| `write_files`            | `storage.files`             |
| `runcmd mkdir`           | `storage.directories`       |
| `runcmd ln`              | `storage.links`             |
| service-related commands | `systemd.units`             |
| `packages`               | No simple direct equivalent |
| arbitrary `runcmd`       | systemd oneshot / warning   |

This mapping is only the initial conceptual model.
Detailed field-by-field mapping is handled in the later Butane
deep-dive stage.

---

# 18. Tools & Environment

The learning environment used:

* Kali Linux VM
* VMware Workstation Player
* Python
* Git
* YAML
* `yamllint`
* `yq`
* `jq`
* Butane CLI

The project work was performed in a Linux-based lab environment
to practice the same configuration and provisioning concepts used
by the target project.

---

# Key Learnings

* YAML is the human-readable configuration layer.
* YAML indentation and structure are critical.
* Cloud-Init automates Linux provisioning.
* `users` handles account configuration.
* `write_files` creates and configures files.
* File permissions must be preserved during translation.
* Encoded content requires special handling.
* `packages` handles software installation in Cloud-Init.
* `runcmd` executes provisioning commands.
* Idempotency is important for reliable provisioning.
* `packages` and `runcmd` are major Cloud-Init → Butane challenges.
* The transpiler must understand semantics, not simply rename YAML
  fields.


