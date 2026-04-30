# ansible-role-docker-swarm-stack

Generic Ansible role to deploy a Docker Swarm stack with **idempotent**, **rotation-safe** secrets and configs.

The role wraps the recurring scaffolding around `community.docker.docker_stack`:

- Optional preflight that ends the play on Swarm workers and on every manager except the first one.
- Optional install of the Python deps required by `community.docker` modules.
- Optional eager creation of overlay/external networks referenced by the stack.
- Optional `docker_login` against one or more registries.
- **Rolling-versions** management of Swarm **secrets** and **configs**: the role creates a new versioned object instead of failing with `secret/config is in use by service ...`, then rewrites the compose `secrets:` / `configs:` sections so each stable alias points at the freshly created version.
- `community.docker.docker_stack` deploy + post-deploy task info debug.

Stack-specific transformations (htpasswd hashing for Traefik, dotenv rendering for app frameworks, ...) are intentionally **not** in this role; do them in the caller playbook's `pre_tasks` and reference the resulting fact inside `swarm_stack_managed_secrets[].data` or `swarm_stack_managed_configs[].data`.

## Requirements

- Ansible >= 2.16
- `community.docker` collection on the controller
- Targets: a Docker Swarm cluster (managers + workers); the role automatically narrows execution to the first manager when `swarm_stack_run_preflight: true`.

Add the role to `roles/requirements.yml`:

```yaml
- src: https://github.com/fidanf/ansible-role-docker-swarm-stack
  name: fidanf.docker_swarm_stack
  version: v0.1.0
```

Install with `ansible-galaxy install -r roles/requirements.yml`.

## Variables

See [`defaults/main.yml`](./defaults/main.yml) for the fully commented contract. Quick reference:

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `swarm_stack_name` | string | `""` | **Required.** Name passed to `docker_stack`. |
| `swarm_stack_content` | mapping | `{}` | **Required.** Compose-style stack body. |
| `swarm_stack_managed_secrets` | list | `[]` | Secrets to create/rotate with rolling versions. |
| `swarm_stack_managed_configs` | list | `[]` | Configs to create/rotate with rolling versions. |
| `swarm_stack_managed_networks` | list | `[]` | Networks to ensure exist before deploy. |
| `swarm_stack_registry_logins` | list | `[]` | Registries to log into before deploy. |
| `swarm_stack_run_preflight` | bool | `true` | Filter execution to the first Swarm manager. |
| `swarm_stack_first_manager_only` | bool | `true` | If false, run on every manager. |
| `swarm_stack_install_pip_requirements` | bool | `true` | Install `swarm_stack_pip_packages`. |
| `swarm_stack_pip_packages` | list | `[jsondiff, pyyaml]` | Python deps for `community.docker`. |
| `swarm_stack_install_apt_requirements` | bool | `false` | Install `swarm_stack_apt_packages`. |
| `swarm_stack_apt_packages` | list | `[]` | Optional apt packages (e.g. `apache2-utils`). |
| `swarm_stack_with_registry_auth` | bool | `true` | Forwarded to `docker_stack`. |
| `swarm_stack_prune` | bool | `true` | Forwarded to `docker_stack`. |
| `swarm_stack_debug_task_info` | bool | `true` | Print `docker_stack_task_info` after deploy. |

### Secret / config item shape

Each item in `swarm_stack_managed_secrets` and `swarm_stack_managed_configs` accepts:

```yaml
- alias: my_app_secret        # required: stable name referenced by compose
  data: "{{ my_app_secret }}" # required when present; if empty/undefined the item is skipped
  versions_to_keep: 2         # optional, default 2
  tags: [secrets, my_app]     # optional, default ["secrets"] / ["configs"]
  no_log: true                # optional, default false
```

The `alias` is the **stable** key referenced by `swarm_stack_content.secrets.<alias>` / `swarm_stack_content.configs.<alias>`. The role mutates that key on the fly to set `external: true` and `name: <alias>_v<N>` so services pick up the new version on their next update.

## Example: control stack (Traefik + Portainer)

Caller playbook computes the htpasswd line in `pre_tasks` and lets the role manage Traefik basic-auth password / Cloudflare DNS API token / Portainer admin password as rolling secrets, plus an Alloy config:

```yaml
---
- name: Swarm Control
  hosts: docker_swarm
  gather_facts: true
  become: true
  vars_files:
    - "{{ inventory_dir }}/stacks/control/vars.yml"
    - "{{ inventory_dir }}/stacks/control/vault.yml"
  pre_tasks:
    - name: Compute deterministic htpasswd salt
      ansible.builtin.set_fact:
        control_basic_auth_salt: "{{ (((control_basic_auth_user ~ ':' ~ control_basic_auth_password) | hash('sha1'))[0:8]) }}"
    - name: Generate deterministic htpasswd hash for Traefik basic auth
      ansible.builtin.shell:
        cmd: openssl passwd -apr1 -salt "{{ control_basic_auth_salt }}" "{{ control_basic_auth_password }}"
        executable: /bin/bash
      register: create_htpasswd
      changed_when: false
    - name: Build basic auth usersFile line
      ansible.builtin.set_fact:
        control_basic_auth_usersfile_line: "{{ control_basic_auth_user }}:{{ create_htpasswd.stdout }}"
  roles:
    - role: fidanf.docker_swarm_stack
```

`environments/<env>/stacks/control/vars.yml` then declares the role-generic vars:

```yaml
---
swarm_stack_name: control
swarm_stack_install_apt_requirements: true
swarm_stack_apt_packages: [apache2-utils]
swarm_stack_pip_packages: [jsondiff, pyyaml, passlib]
swarm_stack_managed_networks:
  - name: t2_proxy
    driver: overlay
    attachable: true
    scope: swarm
swarm_stack_managed_secrets:
  - alias: control_basic_auth_password
    data: "{{ control_basic_auth_usersfile_line | default('') }}"
    tags: [secrets, traefik]
  - alias: control_cf_dns_api_token
    data: "{{ control_cf_dns_api_token | default('') }}"
    tags: [secrets, traefik]
  - alias: control_portainer_password
    data: "{{ control_portainer_password | default('') }}"
    tags: [secrets, portainer]
swarm_stack_managed_configs:
  - alias: control_alloy_config
    data: "{{ control_alloy_config | default('') }}"
    tags: [configs, alloy]
swarm_stack_content:
  version: "3.8"
  networks:
    t2_proxy:
      external: true
  secrets:
    control_basic_auth_password: { external: true }
    control_cf_dns_api_token: { external: true }
    control_portainer_password: { external: true }
  configs:
    control_alloy_config: { external: true }
  services:
    traefik:
      image: traefik:v3.6
      # ...
```

## Example: apps stack (private registry + Symfony dotenv)

Caller renders a list of `{key, value}` pairs into a dotenv-shaped string in `pre_tasks`, then lets the role manage the resulting config + the registry login:

```yaml
---
- name: Swarm Apps
  hosts: docker_swarm
  gather_facts: true
  become: true
  vars_files:
    - "{{ inventory_dir }}/stacks/apps/vars.yml"
    - "{{ inventory_dir }}/stacks/apps/vault.yml"
  pre_tasks:
    - name: Render Symfony dotenv content
      ansible.builtin.set_fact:
        apps_sf_env_content: >-
          {{
            (apps_sf_env | map('items') | map('first') | map('list')
              | map('join', '=') | list) | join('\n')
          }}
      no_log: true
      when: apps_sf_env is defined and apps_sf_env | length > 0
  roles:
    - role: fidanf.docker_swarm_stack
```

```yaml
---
swarm_stack_name: apps
swarm_stack_registry_logins:
  - username: "{{ apps_docker_login }}"
    password: "{{ apps_docker_password }}"
    registry_url: ghcr.io
swarm_stack_managed_secrets:
  - alias: apps_alloy_token
    data: "{{ alloy_token | default('') }}"
    tags: [secrets, alloy]
swarm_stack_managed_configs:
  - alias: apps_sf_env
    data: "{{ apps_sf_env_content | default('') }}"
    tags: [configs, backend]
    no_log: true
swarm_stack_content:
  version: "3.8"
  # ...
```

## How rotation works (under the hood)

1. The role calls `community.docker.docker_secret` / `docker_config` with `rolling_versions: true`. When the payload changes, a new object named `<alias>_v<N>` is created without touching the previous one (which is still in use by services).
2. The role builds an override map `{ <alias>: { external: true, name: <alias>_v<N> } }` from the registered results.
3. The override map is recursively merged into `swarm_stack_content.secrets` / `swarm_stack_content.configs` to produce `swarm_stack_content_effective`.
4. `community.docker.docker_stack` is called with the effective compose; services see a new secret/config name, which triggers a Swarm rolling update.

If the payload is unchanged across runs, no new version is created and `changed=0` is reported.

## Verification

- Idempotency: rerun the playbook with identical inputs; secrets/configs tasks should report `changed=0`.
- Rotation: change a payload and rerun; a new `_v<N>` object is created, services roll, the previous version stays around (kept by `versions_to_keep`).

## License

MIT — see [LICENSE](./LICENSE).
