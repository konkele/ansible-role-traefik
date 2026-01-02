# Ansible Role: Traefik Dynamic Configuration

This **`traefik` Ansible role** generates **Traefik v3.x dynamic configuration files** for HTTP, TCP, UDP routers and TLS options.

> **Note:** This role **does not deploy Traefik**, manage secrets, or create data directories. Its focus is **purely on rendering dynamic configuration files** in a specified config directory.

> **Intended Use:** This role can be used in combination with [ansible-role-docker_stack](https://github.com/konkele/ansible-role-docker_stack) to deploy containerized stacks, but usage of that role is **optional**.

---

## Key Capabilities

* Renders Traefik dynamic configuration files into a **config directory**:

  * `http.yml`
  * `tcp.yml`
  * `udp.yml`
  * `tls.yml`
* Supports inventory-driven router and service definitions.
* Automatically normalizes HTTP routers, hosts, and rules.
* Handles basic authentication middleware for HTTP routers.
* TLS options are hot-reloadable and fully configurable.
* Optional override of the config directory via `traefik_config_path` variable.

---

## Configuration Model

Dynamic configuration is expressed as structured variables:

```yaml
traefik:
  dynamic:
    http:
      routers:
        dashboard:
          enabled: true
          host: traefik.example.com
          service: api@internal
          entryPoints: [websecure]
          middlewares:
            - security-headers
            - internal-only
          basic_auth_users:
            - "admin:$apr1$hash"
    tcp:
      routers:
        mqtt:
          enabled: false
          entryPoints: [mqttsecure]
          rule: "HostSNI(`mqtt.example.com`)"
          service: mqtt
    udp:
      routers:
        dns:
          enabled: false
          entryPoints: [dns]
          service: dns
    tls:
      min_version: VersionTLS13
      sni_strict: true
```

### Normalization

For **enabled HTTP routers**:

* `hosts` is always a list.
* `rule` is generated automatically if not explicitly defined.
* Legacy `host` keys are removed after normalization.
* Routers with `basic_auth_users` automatically get a router-specific `*-auth` middleware attached.

---

## Directory Layout

Dynamic configuration files are rendered under the Traefik stack config directory, which can be overridden per role run using `traefik_config_path`:

```
{{ docker_stack.directories.config.path }}/
  http.yml
  tcp.yml
  udp.yml
  tls.yml
```

*Directories for data or secrets are not created.*

---

## Role Execution Flow

1. Merge inventory and default variables.
2. Normalize HTTP, TCP, and UDP routers.
3. Apply optional `traefik_config_path` override.
4. Render dynamic configuration files into the config directory.

---

## Example Playbook

```yaml
- name: Generate Traefik Dynamic Config
  hosts: swarm_managers
  become: true
  vars:
    # Optional override for config directory
    traefik_config_path: /opt/traefik/custom_config
  roles:
    - traefik
```

---

## Design Principles

* Inventory expresses **intent**, not static rendered YAML.
* Deterministic merges eliminate config drift.
* Hot-reload wherever Traefik supports it.
* Minimal filesystem footprint — only dynamic configuration is created.
* Templates tolerate future Traefik features.
* Compatible with `ansible-role-docker_stack` but not dependent on it.

---

## License

MIT License
