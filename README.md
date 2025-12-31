# Ansible Role: Traefik

This **`traefik` Ansible role** deploys and manages **Traefik v3.x** as a Docker-based reverse proxy with a strong focus on **Docker Swarm**, **inventory-driven configuration**, and **hot‑reloadable dynamic routing**.

The role is designed to be **deterministic, layered, and reusable** across multiple clusters and environments. All Traefik configuration—static, dynamic, and deployment—is expressed as structured data and rendered consistently at runtime.

---

## Key Capabilities

> **Dependency:** This role delegates all Docker Compose / Swarm stack creation to the shared role:
>
> **ansible-role-docker_stack** – [https://github.com/konkele/ansible-role-docker_stack](https://github.com/konkele/ansible-role-docker_stack)
>
> The Traefik role is responsible for *intent, normalization, and rendering*; the `docker_stack` role is responsible for *materializing* that intent into Docker resources.

### Deployment

* Deploys Traefik via a shared **`docker_stack` role**
* Supports **Docker Swarm** (primary target)
* Compose-compatible data model (Compose v2 semantics)
* Safe rolling updates (`start-first`, rollback on failure)
* Manager-only placement by default

### Configuration Model

* **Strict layered merge** of configuration sources
* Inventory expresses *intent*, not rendered YAML
* Canonical variables always exist after merge

Merge precedence (lowest → highest):

```
<var>_defaults
<var>_group
docker_stack_group
<var>_host
docker_stack_host
<var>_override
```

Merged outputs are exposed as:

* `traefik`
* `docker_stack`

---

## Traefik Configuration Scope

### Static Configuration (Container Arguments)

This role does **not** render a standalone `traefik.yml` file.

All **static Traefik configuration** is expressed via **container command arguments** defined in the Docker Compose / Swarm service specification:

```
docker_stack.services.controller.command
```

This includes:

* Providers (Swarm, file)
* EntryPoints and redirects
* TLS defaults and certificate resolvers
* ACME / Let’s Encrypt configuration
* Logging and access logs
* Global Traefik flags

Changes to static configuration require a **service update / restart**, which is handled safely by Docker Swarm rolling updates.

---

### Dynamic Configuration (Hot‑Reloadable)

Rendered into individual files under:

```
{{ docker_stack.directories.dynamic.path }}
```

Dynamic files:

* `http.yml`
* `tcp.yml`
* `udp.yml`
* `tls.yml`

Dynamic changes are picked up automatically by Traefik without a container restart.

---

## HTTP Routers (Dynamic)

HTTP routers are fully inventory-driven and normalized automatically.

### Supported Inputs

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
```

### Normalization Guarantees

For **enabled routers**, the role guarantees:

* `hosts` is always a list
* `rule` is always generated or honored if explicitly defined
* `Host()` rules are OR‑joined automatically
* Legacy `host` key is removed after normalization

Disabled routers pass through untouched.

---

## Services (HTTP / TCP / UDP)

* Services are derived automatically from router definitions
* HTTP services support load‑balanced URL backends
* TCP / UDP services support address backends

No duplicated service definitions are required.

---

## Middlewares

### Built‑in Defaults

Defined in `defaults/main.yml`:

* `security-headers`

  * SSL redirect
  * HSTS (preload)
  * Frame deny
  * XSS protection
  * Content type sniff prevention

* `internal-only`

  * RFC1918 allowlist
  * Loopback

### Per‑Router Auth

If `basic_auth_users` is present on a router:

* A router‑specific `*-auth` middleware is generated
* Automatically attached to the router

---

## TLS Options

TLS options are rendered separately and hot‑reloadable:

```yaml
traefik:
  dynamic:
    tls:
      min_version: VersionTLS13
      sni_strict: true
```

Rendered via:

```
roles/traefik/templates/dynamic/tls.yml.j2
```

---

## Directory Layout

All filesystem paths are computed safely and deterministically.

Default layout:

```
/opt/stacks/
  traefik/
    config/
      dynamic/
    data/
    secrets/
```

### Design Notes

* Directories are **computed first**, then merged
* No incremental mutation of `docker_stack`
* Extra directories may be defined freely

---

## Secrets

Secrets are defined declaratively and injected into Swarm.

```yaml
docker_stack:
  secrets:
    cf_dns_api_token:
      value: "super-secret"
```

* Stored securely on the host
* Mounted via Docker secrets
* Secret hash is exposed via label (`docker.secrets.hash`)
* Changing secrets triggers a controlled rolling update

---

## Networks

* Uses an **external overlay network** by default (`proxy`)
* Attachable for downstream stacks

```yaml
docker_stack:
  networks:
    proxy:
      external: true
      driver: overlay
      attachable: true
```

---

## Role Execution Flow

1. Load optional vars file (`traefik_vars_file`)
2. Merge layered configuration (`merge.yml`)
3. Normalize and finalize directory structure
4. Normalize HTTP routers
5. Ensure directories exist
6. Render dynamic configuration files
7. Delegate deployment to `docker_stack` role

---

## Example Playbook

```yaml
- name: Deploy Traefik
  hosts: swarm_managers
  become: true
  vars:
    traefik_vars_file: group_vars/traefik.yml
  roles:
    - traefik
```

---

## Design Principles

* Inventory expresses **intent**, not rendered YAML
* Deterministic merges eliminate config drift
* Hot‑reload wherever Traefik supports it
* No hidden defaults
* Templates tolerate future Traefik features
* Safe rolling updates by default

---

## Notes / Updates in v3.x

* Updated role to support **Traefik v3.x**
* Dynamic configuration templates now fully compatible with v3 routing syntax
* Handlers updated to use **community.docker.docker_compose_v2** for safe restarts
* Secrets and TLS configuration aligned with Swarm deployment best practices

---

## License

MIT License