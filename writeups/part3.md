
# Part 3 — Butane Deep Dive & Cloud-Init → Butane Mapping

## Objective

The final part of the learning journey focused on understanding
Butane in depth and building the conceptual mapping required for
the Cloud-Init → Butane transpiler.

The main focus was:

- Butane schema
- `variant` and `version`
- `passwd.users`
- `storage.files`
- `storage.directories`
- `storage.links`
- `systemd.units`
- Cloud-Init → Butane field mapping
- Direct mappings
- Field renames
- Format transformations
- Smart transformations
- Unsupported directives
- Warning generation
- End-to-end configuration conversion

---

# 1. Butane Output Structure

The transpiler is expected to produce valid Butane YAML.

A basic Butane configuration starts with:

```yaml
variant: flatcar
version: 1.1.0
````

For this project, the target is Flatcar Container Linux, so the
generated configuration uses:

```yaml
variant: flatcar
version: 1.1.0
```

The `variant` identifies the target operating system while
`version` identifies the Butane schema version. 

---

# 2. Butane passwd Section

Butane uses:

```yaml
passwd:
  users:
```

for user configuration.

Example:

```yaml
passwd:
  users:
    - name: admin
      shell: /bin/bash
      uid: 1001
      home_dir: /home/admin
      groups:
        - sudo
        - docker
      ssh_authorized_keys:
        - ssh-ed25519 AAAA...
```

Important fields studied include:

* `name`
* `password_hash`
* `ssh_authorized_keys`
* `uid`
* `gecos`
* `home_dir`
* `shell`
* `groups`
* `primary_group`
* `system`
* `no_user_group`
* `should_exist`



---

# 3. Cloud-Init Users → Butane Users

The user mapping is one of the core parts of the transpiler.

| Cloud-Init            | Butane                | Transformation |
| --------------------- | --------------------- | -------------- |
| `name`                | `name`                | Direct         |
| `gecos`               | `gecos`               | Direct         |
| `homedir`             | `home_dir`            | Rename         |
| `shell`               | `shell`               | Direct         |
| `uid`                 | `uid`                 | Direct         |
| `primary_group`       | `primary_group`       | Direct         |
| `groups`              | `groups`              | Format change  |
| `ssh_authorized_keys` | `ssh_authorized_keys` | Direct         |
| `passwd`              | `password_hash`       | Rename         |
| `system`              | `system`              | Direct         |
| `no_user_group`       | `no_user_group`       | Direct         |

Some Cloud-Init fields do not have direct Butane equivalents.
These should not silently disappear; the transpiler should be able
to generate warnings for unsupported functionality. 

---

# 4. Format Transformation — groups

One important difference is the format of groups.

Cloud-Init may represent groups as:

```yaml
groups: sudo, docker
```

Butane expects a YAML list:

```yaml
groups:
  - sudo
  - docker
```

Therefore, the transpiler needs to perform an actual
transformation:

```text
Input
"sudo, docker"

      ↓

Split by comma
      ↓

Trim whitespace
      ↓

Create YAML list

      ↓

["sudo", "docker"]
```

This is an example where the transpiler is not simply copying
fields — it is transforming their representation. 

---

# 5. sudo Transformation

Cloud-Init can contain:

```yaml
sudo: ALL=(ALL) NOPASSWD:ALL
```

Butane does not provide a direct `sudo` field.

The studied approach is to represent normal sudo access through
membership in the `sudo` group:

```yaml
groups:
  - sudo
```

The original sudo configuration can also be preserved as a
comment/warning when the exact semantics cannot be represented.

Example:

```yaml
# NOTE: sudo access granted via 'sudo' group membership
groups:
  - sudo
```

This is categorized as a **smart transformation**, rather than
a direct field mapping. 

---

# 6. Cloud-Init write_files → storage.files

Cloud-Init:

```yaml
write_files:
  - path: /etc/myapp/config.yaml
    permissions: "0644"
    content: |
      app_name: myapp
```

becomes conceptually:

```yaml
storage:
  files:
    - path: /etc/myapp/config.yaml
      mode: 0644
      contents:
        inline: |
          app_name: myapp
```

Important transformations:

```text
permissions → mode
content     → contents.inline
owner       → user.name + group.name
```

These mappings preserve the actual file configuration while
converting it into the Butane representation. 

---

# 7. File Ownership

Cloud-Init may use:

```yaml
owner: root:root
```

Butane represents ownership using separate structures:

```yaml
user:
  name: root

group:
  name: root
```

Therefore:

```text
root:root
   ↓
split owner
   ↓
user.name = root
group.name = root
```

This is another example of a format transformation rather than
a direct copy. 

---

# 8. Encoded File Content

For normal content, Butane can use:

```yaml
contents:
  inline: |
    hello
```

For encoded content, the representation can instead use a source
containing encoded data.

Conceptually:

```text
Cloud-Init encoded content
          ↓
detect encoding
          ↓
transform content
          ↓
Butane contents.source
```

The important requirement is that the transpiler must not
accidentally treat encoded content as normal plaintext. 

---

# 9. storage.directories

Some Cloud-Init `runcmd` operations can be converted into native
Butane storage operations.

Example Cloud-Init:

```yaml
runcmd:
  - mkdir -p /var/www/html
```

Can become:

```yaml
storage:
  directories:
    - path: /var/www/html
      mode: 0755
```

The mapping therefore changes from an imperative shell command
into a declarative storage configuration.

```text
runcmd mkdir
      ↓
storage.directories
```



---

# 10. storage.links

Symbolic-link commands can similarly be converted.

Cloud-Init:

```yaml
runcmd:
  - ln -sf /etc/nginx/sites-available/mysite \
           /etc/nginx/sites-enabled/mysite
```

Butane:

```yaml
storage:
  links:
    - path: /etc/nginx/sites-enabled/mysite
      target: /etc/nginx/sites-available/mysite
```

Conceptually:

```text
runcmd ln
   ↓
storage.links
```



---

# 11. systemd.units

Service configuration is one of the most important transformations.

Example:

```yaml
systemd:
  units:
    - name: myapp.service
      enabled: true
      contents: |
        [Unit]
        Description=My Application

        [Service]
        ExecStart=/opt/myapp/start.sh

        [Install]
        WantedBy=multi-user.target
```

Butane's `systemd.units` can represent services directly.

The transpiler may combine information from multiple Cloud-Init
directives into a single systemd unit. 

---

# 12. Service Pattern Detection

A major lesson from the mapping stage was that some conversions
require pattern detection.

For example:

```text
write_files
    ↓
service file created

runcmd
    ↓
systemctl enable myapp

       ↓

Detect service pattern

       ↓

systemd.units
```

The resulting Butane configuration can combine the service file
and enable state:

```yaml
systemd:
  units:
    - name: myapp.service
      enabled: true
      contents: |
        ...
```

This is a **pattern combination**, not a simple one-to-one field
mapping. 

---

# 13. systemctl Commands

Some Cloud-Init commands become unnecessary after conversion.

For example:

```bash
systemctl daemon-reload
```

does not need to remain as an explicit command in the generated
configuration.

Similarly, when a service is represented with:

```yaml
enabled: true
```

the separate:

```bash
systemctl enable myapp
```

command is covered by the Butane configuration.

The studied conversion therefore removes redundant imperative
commands where the declarative Butane configuration already
provides the required behavior. 

---

# 14. runcmd → systemd oneshot

Not every `runcmd` command has a native Butane equivalent.

For arbitrary commands, one possible strategy is to represent the
operation as a systemd oneshot service.

Example:

```ini
[Service]
Type=oneshot
ExecStart=/bin/tar czf /tmp/backup.tar.gz /opt/myapp/data
```

Butane:

```yaml
systemd:
  units:
    - name: backup.service
      contents: |
        [Unit]
        Description=Backup app data

        [Service]
        Type=oneshot
        ExecStart=/bin/tar czf /tmp/backup.tar.gz /opt/myapp/data
```

The learning material also demonstrated scheduled operations using
a `.timer` unit. 

---

# 15. Unsupported Features

A transpiler must distinguish between:

```text
Supported
   ↓
Transform

Partially supported
   ↓
Transform + warning

Unsupported
   ↓
Warning / preserve information
```

A major example is:

```yaml
packages:
  - nginx
```

There is no simple direct equivalent in the target mapping.

Therefore the expected behavior is not to silently generate
incorrect Butane.

Instead:

```text
packages
   ↓
WARNING
```

This is important for safe automated migration. 

---

# 16. Six Mapping Categories

The final mapping model was divided into six categories:

### 1. Direct Mapping

Example:

```text
Cloud-Init name → Butane name
Cloud-Init shell → Butane shell
Cloud-Init uid → Butane uid
```

### 2. Rename

Example:

```text
homedir → home_dir
passwd → password_hash
```

### 3. Format Change

Example:

```text
groups: sudo, docker
        ↓
groups:
  - sudo
  - docker
```

### 4. Smart Transformation

Example:

```text
hostname → /etc/hostname file
sudo → sudo group
```

### 5. Pattern Combination

Example:

```text
service file + systemctl enable
        ↓
systemd.units
```

### 6. No Equivalent

Example:

```text
packages
   ↓
WARNING
```

These six categories form the conceptual basis for the
transpiler's transformation logic. 

---

# 17. Complete Transformation Flow

The final conceptual pipeline is:

```text
        Cloud-Init YAML
               │
               ▼
             Parse
               │
               ▼
        Detect directives
               │
               ▼
       ┌───────┴────────┐
       │                │
   Supported        Unsupported
       │                │
       ▼                ▼
   Transform          Warning
       │                │
       └───────┬────────┘
               ▼
         Build Butane
               │
               ▼
        Generate YAML
               │
               ▼
        Butane CLI
               │
               ▼
        Ignition JSON
               │
               ▼
      Flatcar Container Linux
```

The Day 7 notes summarize the core decision process as:

```text
Parse → detect → transform/warn → output
```



---

# 18. Butane Validation

The generated Butane configuration should be validated using the
Butane CLI.

Example:

```bash
butane --pretty --strict config.bu > config.ign
```

The process is:

```text
config.bu
   ↓
Butane CLI
   ↓
config.ign
```

`--pretty` produces readable JSON output.

`--strict` treats warnings as errors, making validation stricter.



---

# 19. Hands-On Verification

A minimal Butane configuration was used for validation:

```yaml
variant: flatcar
version: 1.1.0

passwd:
  users:
    - name: testuser
      shell: /bin/bash

storage:
  files:
    - path: /etc/motd
      mode: 0644
      contents:
        inline: Welcome to Flatcar!
```

It was tested with:

```bash
butane --pretty --strict test.bu
```

Expected result:

```text
Valid Ignition JSON output
```

This verifies the basic relationship:

```text
Butane YAML → Ignition JSON
```



---

# 20. Final Understanding

After completing the three-part learning process, the overall
architecture is understood as:

```text
                INPUT
                  │
                  ▼
          ┌───────────────┐
          │ Cloud-Init    │
          │ YAML          │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │ Transpiler    │
          │               │
          │ Parse         │
          │ Detect        │
          │ Transform     │
          │ Warn          │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │ Butane YAML   │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │ Butane CLI    │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │ Ignition JSON │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │ Flatcar       │
          │ Container     │
          │ Linux         │
          └───────────────┘
```

---

# Key Takeaways

* Butane is the output format of the planned transpiler.
* `variant` and `version` are mandatory for the generated config.
* `passwd.users` is the main destination for Cloud-Init users.
* `write_files` maps primarily to `storage.files`.
* `mkdir` patterns can map to `storage.directories`.
* `ln` patterns can map to `storage.links`.
* Service configurations can map to `systemd.units`.
* Some transformations require renaming fields.
* Some require changing data formats.
* Some require semantic/pattern detection.
* Unsupported directives should produce warnings instead of being
  silently discarded.
* `packages` has no simple direct equivalent in the studied mapping.
* `runcmd` may require systemd-based transformation or a warning.
* The final transformation logic can be represented as:

  `Parse → Detect → Transform/Warn → Output`


