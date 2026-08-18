[![eugeneivanov.dev — Infrastructure Engineering](assets/eugeneivanov-dev-logo_1280.webp)](https://eugeneivanov.dev)

# ansible

[![lint](https://github.com/eugeneivanov-dev/ansible/actions/workflows/lint.yml/badge.svg)](https://github.com/eugeneivanov-dev/ansible/actions/workflows/lint.yml)

Configuration management for my lab. This repository holds the Ansible
baseline for the lab's fleets — the same standard every VM used to get
by hand, rewritten as idempotent roles. A new VM reaches the lab standard by
running a playbook; an existing one proves it still matches by running the
same playbook in check mode. Both fleets — RHEL and Ubuntu — are fully
managed from here.

Project pages:  
[Ansible Baseline for the Lab](https://eugeneivanov.dev/projects/ansible-baseline-for-the-lab/)  
[Ubuntu Baseline for the Lab](https://eugeneivanov.dev/projects/ubuntu-baseline-for-the-lab/)

## Requirements

- ansible-core 2.21
- collections from `requirements.yml` (`ansible-galaxy collection install -r requirements.yml`)
- managed hosts: RHEL 10 and Ubuntu Server 24.04 LTS, reachable over SSH with Python present
- an Ansible Vault file created from the template (see Secrets below)

## Layout

```
ansible.cfg               defaults: inventory path, remote user
bootstrap.yml             one-time play: creates the automation user on new hosts (both fleets)
site.yml                  two plays: rhel baseline (nine roles), ubuntu baseline (eight roles)
inventory/hosts.yml       two flat groups: rhel, ubuntu — both managed
group_vars/all/           main.yml — lab facts (resolvers, search domain, gateway, monitor address, node_exporter pin)
group_vars/rhel/          rhel role inputs, referencing the lab facts; vault.yml.example — template for the encrypted vault
group_vars/ubuntu/        ubuntu role inputs, referencing the lab facts
host_vars/                per-host values: netplan address, app ports as port + source pairs
roles/                    fleet-prefixed baseline roles (rhel_*, ubuntu_*), one topic each
```

## Variables

Where values come from and which hosts see them:

```mermaid
flowchart TB
    lab_facts["group_vars/all/main.yml<br/>lab facts: resolvers, search domain, gateway, monitor IP, node_exporter pin"]

    subgraph rhel_inputs["rhel inputs"]
        vault["group_vars/rhel/vault.yml<br/>encrypted, outside git"]
        rhel_vars["group_vars/rhel/main.yml<br/>rhel role inputs, referencing lab facts"]
    end

    subgraph ubuntu_inputs["ubuntu inputs"]
        ubuntu_vars["group_vars/ubuntu/main.yml<br/>ubuntu role inputs, referencing lab facts"]
        host_vars["host_vars/&lt;fqdn&gt;.yml<br/>per-host values"]
    end

    rhel["rhel group — managed by site.yml"]
    ubuntu["ubuntu group — managed by site.yml"]

    vault -- "vault_rhsm_username, vault_rhsm_password" --> rhel
    lab_facts --> rhel_vars
    lab_facts --> ubuntu_vars
    rhel_vars --> rhel
    ubuntu_vars --> ubuntu
    host_vars --> ubuntu
```

Lab facts — values that belong to the lab itself, not to any fleet — live
once in `group_vars/all/main.yml`. Fleet role inputs reference them
(`rhel_dns_resolver_servers: "{{ lab_dns_servers }}"`), so a value changes
in one place and every consumer follows. Roles carry no site-specific
values. Secrets sit in the group that consumes them: the vault lives in
`group_vars/rhel/`, visible to the rhel play only.

A host_vars file declares only what makes the host unique. The minimal one
is a single line — the host's address. The richest one belongs to the
monitoring host: its dashboards, its own exporter, and the containers that
consume both.

Both fleets connect under one automation user — the `remote_user` default
in `ansible.cfg`. A host answers it after `bootstrap.yml` has run against
it; until then it shows up as unreachable in full-fleet runs.

## Roles and execution order

Role names and tags carry the fleet prefix: `rhel_*` and `ubuntu_*`.
`site.yml` runs the rhel play in this order:

1. **rhel_registration** — subscription-manager with credentials from vault
2. **rhel_ssh_hardening** — key-only SSH, config validated before restart, effective policy asserted via `sshd -T`
3. **rhel_selinux** — enforcing
4. **rhel_firewalld** — deny-by-default zones
5. **rhel_chrony** — time sync
6. **rhel_updates** — package updates; reboot only behind an explicit flag, off by default
7. **rhel_packages** — the lab's standard package set
8. **rhel_dns_resolver** — clients point at the lab's own nameservers
9. **rhel_monitoring_agent** — node_exporter, reporting to the lab's monitoring host

The ubuntu play follows in this order:

1. **ubuntu_ssh_hardening** — key-only SSH via a drop-in named to sort before cloud-init's; completes the transition from socket-activated sshd to the classic service, terminating the leftover socket-spawned listener; config validated before restart, effective policy asserted via `sshd -T`
2. **ubuntu_apparmor** — asserts AppArmor is active with enforcing profiles
3. **ubuntu_ufw** — deny-by-default; ssh owned by an explicit 22/tcp rule, the distro's OpenSSH profile rule removed; app ports declared per host as port + source pairs, so every rule names its consumer
4. **ubuntu_chrony** — time sync via chrony, replacing systemd-timesyncd
5. **ubuntu_updates** — unattended-upgrades removed, updates run through the playbook only; reboot behind an explicit flag, off by default
6. **ubuntu_packages** — the lab's standard package set
7. **ubuntu_netplan** — owns the host's netplan configuration; cloud-init network config disabled
8. **ubuntu_monitoring_agent** — node_exporter as a pinned binary, replacing the packaged exporter; optional textfile collector directory for hosts that publish their own metrics

The order is designed for the worst moment of interruption: security layers
run before updates, so a play that dies halfway leaves the host locked down
and unpatched rather than patched and open. Both plays follow the same
principle.

## Package baseline

The standard set both `*_packages` roles install — general-purpose tools
every host carries regardless of its service:

| Fleet  | Packages |
|--------|----------|
| rhel   | vim, bash-completion, tar, policycoreutils-python-utils, bind-utils, openssl |
| ubuntu | vim, bash-completion, tar, bind9-dnsutils, openssl, qemu-guest-agent |

The sets differ where the platforms do: SELinux tooling exists only on RHEL,
`bind-utils` is named `bind9-dnsutils` on Ubuntu, and `qemu-guest-agent`
ships preinstalled on the RHEL template but not in the Ubuntu installer.
Changing a set means changing the role and this table in the same commit.

## Secrets

The encrypted vault file never enters git. The repository carries
`group_vars/rhel/vault.yml.example` with every expected key; the real
`group_vars/rhel/vault.yml` is created from it, encrypted with Ansible Vault,
and stays on the machines that run plays. The vault passphrase lives outside
the repository too — its path is provided by the `ANSIBLE_VAULT_PASSWORD_FILE`
environment variable on each machine.

Ansible Vault here is a deliberate temporary minimum: secrets move to
HashiCorp Vault when it arrives as its own service.

## Git boundary

Development happens on the workstation; execution happens on a dedicated
control VM in the lab.

- workstation: edit, commit, push (read-write key)
- control VM: pull only (read-only deploy key), run plays from the working copy
- no edits and no commits on the control VM — code flows one way

## Usage

```
# once per new host: create the automation user
# (SSH as your initial user whose key is already on the host; -K asks for sudo password)
ansible-playbook bootstrap.yml --limit newhost.example.com -e ansible_user=youruser -K

# full baseline
ansible-playbook site.yml

# drift check against existing hosts
ansible-playbook site.yml --check

# one role only
ansible-playbook site.yml --tags rhel_chrony

# scope a run to one host
ansible-playbook site.yml --limit newhost.example.com
```

## Scope

This is the OS baseline for the lab's hosts — the layer every service stands
on. Both fleets are fully managed. Application deployment stays with each
service's own mechanism.

This is a personal lab, not a reference implementation: the code encodes my lab's
decisions — addressing, DNS names, package choices — and is published as a
working example, not a drop-in solution.

## CI

Every push runs yamllint and ansible-lint (production profile) with pinned
versions — the same commands and versions used locally.

## History

This code was developed in a private repository from day one of the lab's
Ansible project. The public repository starts with a clean history: the
private one carries encrypted secrets and early lab-specific details that
have no place in public git history, even encrypted.

The reasoning behind the transition is written up in
[Going Public: the Decisions Behind Opening My Ansible Repo](https://eugeneivanov.dev/journal/labnotes/going-public-ansible-repo-decisions/),
and the mechanics in
[Going Public: the Ansible Repo Transition, Step by Step](https://eugeneivanov.dev/journal/labnotes/going-public-ansible-repo-transition/).

## License

MIT