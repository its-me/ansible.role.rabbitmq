# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Ansible role for installing and configuring RabbitMQ (v3.6.x) with Erlang (v19.x) on Debian/Ubuntu systems via apt. It supports single-node and multi-node cluster deployments.

## Role Dependencies

Declared in `meta/main.yml`, the role depends on three other roles (must exist in the playbook's roles path):
- `apt` — installs packages and manages apt repos/pins (called three times: for `python-requests`, pinned Erlang, and pinned RabbitMQ)
- `limits` — sets systemd `LimitNOFILE` for `rabbitmq-server`
- `shorewall` — configures firewall rules (called conditionally via `tags: shorewall`)
- `facts` — stores persistent Ansible local facts (used to track cluster join state in `ansible_local.rabbitmq.cluster_ready`)

## Task Flow (`tasks/main.yml`)

Three tagged blocks run in order:

1. **`prepare`** — Writes Erlang cookie, env file, config file, enables plugins, restarts service if any changed.
2. **`shorewall`** — Delegates to the `shorewall` role with `shorewall_fact: rabbitmq_cluster`.
3. **`init`** — Handles cluster formation: non-master nodes join the master via `rabbitmqctl join_cluster`, then sets HA policy (`ha-mode: all`, `ha-sync-mode: automatic`) via `rabbitmq_policy` module.

Cluster join is idempotent via the `rabbitmq_cluster_ready` fact stored in `ansible_local.rabbitmq`. The master node is always `groups["rabbitmq_cluster"][0]`.

## Key Variables (`defaults/main.yml`)

- `rabbitmq_cluster_master` — first host in the `rabbitmq_cluster` inventory group
- `rabbitmq_cluster_ready` — read from `ansible_local.rabbitmq.cluster_ready`; prevents re-joining an already-formed cluster
- `rabbitmq_erlang_cookie` — must be identical across all cluster nodes (default is a placeholder, override in vault)
- `rabbitmq_ssl` — boolean gate; SSL listener and cert options are only rendered when `True`
- `rabbitmq_plugins` — list iterated with `with_items`; defaults to `rabbitmq_management`
- Variables prefixed `rabbitmq_env__` map to `rabbitmq-env.conf` environment variables

## Templates

- `templates/etc/rabbitmq/rabbitmq.config.j2` — renders `rabbitmq.conf` (new-style sysctl format, not classic Erlang terms). Most settings are conditionally rendered only when the corresponding variable is defined.
- `templates/etc/rabbitmq/rabbitmq.env.j2` — renders `rabbitmq-env.conf` from `rabbitmq_env__*` variables.

## Running the Role

```bash
# Full role
ansible-playbook site.yml --tags rabbitmq

# Only update config and restart
ansible-playbook site.yml --tags prepare

# Only cluster init tasks
ansible-playbook site.yml --tags init

# Only HA policy
ansible-playbook site.yml --tags init:policy

# Lint with ansible-lint
ansible-lint roles/rabbitmq/tasks/main.yml
```

## Inventory Requirements

The cluster tasks require an inventory group named `rabbitmq_cluster`. The first host in that group becomes the cluster master.

```ini
[rabbitmq_cluster]
rabbit1.example.com
rabbit2.example.com
rabbit3.example.com
```
