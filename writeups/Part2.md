
# Part 2 — Ignition & Butane

## Objective

The goal of Part 2 was to understand how Flatcar Container Linux
uses Ignition for first-boot provisioning and why Butane is used
as the human-friendly configuration layer.

---

## 1. What is Ignition?

Ignition is a first-boot provisioning system designed for
immutable and container-optimized Linux distributions such as
Flatcar Container Linux and Fedora CoreOS.

Key characteristics:

- Configuration format: JSON
- Runs only during the first boot
- Applies configuration atomically
- Designed for immutable operating systems
- Intended to be consumed by the machine rather than manually written by humans
- Butane can generate the Ignition configuration automatically

Ignition is therefore different from traditional configuration
systems such as Cloud-Init.

---

## 2. Why Does Ignition Use JSON?

Ignition uses JSON because its configuration is consumed by the
boot process rather than being written directly by humans.

JSON provides:

- Strict structure
- Explicit data types
- Predictable parsing
- No indentation-based ambiguity

YAML is much more convenient for humans, while JSON is well suited
for machine-readable configuration.

---

## 3. Atomic Provisioning

One of the most important concepts in Ignition is **atomic
provisioning**.

Atomic means:

> All required configuration should be applied successfully,
> otherwise the provisioning process fails rather than leaving
> the system in a partially configured state.

For example, an Ignition configuration may:

1. Create a user
2. Create configuration files
3. Configure systemd services
4. Set file permissions

If the configuration cannot be applied correctly, Ignition does
not intentionally leave the system in an unknown half-configured
state.

This makes provisioning deterministic and predictable.

---

## 4. First-Boot-Only Behavior

Ignition runs during the initial provisioning of the system.

Conceptually:

    First Boot
        ↓
    Read Ignition JSON
        ↓
    Validate configuration
        ↓
    Apply configuration
        ↓
    System becomes ready
        ↓
    Ignition does not provision again

This differs from Cloud-Init, which can perform configuration
during multiple stages of the system lifecycle.

The first-boot-only design supports the immutable infrastructure
model and reduces configuration drift.

---

## 5. Ignition Configuration Example

A simplified Ignition configuration can look like:

```json
{
  "ignition": {
    "version": "3.4.0"
  },
  "passwd": {
    "users": [
      {
        "name": "admin",
        "sshAuthorizedKeys": [
          "ssh-ed25519 AAAA... admin@lab"
        ]
      }
    ]
  },
  "storage": {
    "files": [
      {
        "path": "/etc/hostname",
        "contents": {
          "source": "data:,flatcar-server"
        },
        "mode": 420
      }
    ]
  }
}
````

This example demonstrates how Ignition represents users and files
using a deeply nested JSON structure.

File permissions are represented as numeric values. For example:

```
420 decimal = 0644 octal
```

Because Ignition JSON can become difficult for humans to construct
manually, another configuration layer is used.

---

# 6. Why Butane Exists

Butane provides a human-friendly YAML configuration format that
can be converted into Ignition JSON.

The overall flow is:

```
Human
  ↓
Butane YAML
  ↓
Butane CLI
  ↓
Ignition JSON
  ↓
Flatcar Linux
```

Therefore:

**Butane is the human-friendly configuration layer, while Ignition
is the machine-oriented provisioning format.**

---

## 7. Basic Butane Structure

A minimal Butane configuration contains the target variant and
schema version.

Example:

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

Important sections include:

* `variant`
* `version`
* `passwd`
* `storage`
* `systemd`

---

## 8. Butane → Ignition Conversion

The Butane CLI can convert the YAML configuration into Ignition
JSON.

Example:

```bash
butane --pretty --strict config.bu > config.ign
```

Where:

* `--pretty` produces readable JSON
* `--strict` treats warnings as errors

The resulting file can then be consumed by the Flatcar
provisioning process.

---

## 9. Hands-On Verification

A test configuration was created:

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

It was tested using:

```bash
butane --pretty --strict test.bu
```

Expected result:

```text
Valid Ignition JSON output
```

This verified the basic Butane → Ignition workflow.

---

# 10. Cloud-Init vs Ignition

| Feature             | Cloud-Init        | Ignition                    |
| ------------------- | ----------------- | --------------------------- |
| Main format         | YAML              | JSON                        |
| Human-friendly      | Yes               | No                          |
| Provisioning model  | Multi-stage       | First boot                  |
| Configuration style | Flexible          | Deterministic               |
| Partial failure     | Possible          | Atomic                      |
| Target              | Traditional Linux | Immutable OS                |
| Package management  | Supported         | Not directly                |
| Configuration layer | Direct            | Usually generated by Butane |

---

# 11. Relevance to the Transpiler Project

The target project is:

```
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
```

This means the transpiler does not need to manually construct
Ignition JSON.

Instead, its responsibility is to produce valid Butane YAML.

The Butane CLI then handles the conversion from Butane YAML to
Ignition JSON.

---

## Key Learnings

* Ignition is a first-boot provisioning system.
* Ignition uses JSON as its machine-oriented configuration format.
* Ignition follows an atomic provisioning model.
* Ignition is designed for immutable operating systems.
* Ignition does not need to be written manually.
* Butane provides a human-friendly YAML layer.
* Butane converts YAML configuration into Ignition JSON.
* `variant` and `version` are mandatory in a basic Butane config.
* `passwd`, `storage`, and `systemd` are important Butane sections.
* The Cloud-Init → Butane transpiler should generate Butane YAML,
  not raw Ignition JSON.

---

## Verification

Tools used during this part:

* Butane CLI
* YAML
* JSON
* `yamllint`
* `jq`
* Kali Linux VM

The hands-on Butane configuration was successfully converted
into Ignition JSON using the Butane CLI.

---


