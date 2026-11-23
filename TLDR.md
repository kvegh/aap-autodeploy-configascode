# TLDR

Rebuild a physical host and an AAP instance from configuration as code. This
repository is a refactor of the previous implementation, keeping what it proved
and dropping the rest. See [README.md](README.md) for the full picture.

Rule: configuration is separated from execution code. This repository holds
execution code only; all configuration lives in the private companion
repository's CaC (inventory variables of AutoDeploy-NGNG), with no defaults in
the playbooks: accounts, sudo rules, host naming, ports, retry counts, the
counter path and the CaC apply job's use case and ref included. Only
third-party facts (software source URLs) stay in code. There only real
secrets are vault encrypted.

## What exists today

Seven parts in two workflows; playbooks `PS-<subject>-<action>.yml`, job
templates `AutoDeploy-NGNG PS <what>` (P part, S order).

Workflow "AutoDeploy-NGNG Hypervisor deploy", once per hypervisor:

1. Hypervisor install: `10-provider-reinstall` (erases and reinstalls Rocky
   Linux 9 after the survey confirmation), `11-dns-record-publish` (host name
   at GoDaddy), `12-hypervisor-access-bootstrap` (accounts, root and password
   login off, SSH on the managed port), `13-rhel-convert`, `14-rhel-update`.
2. Hypervisor OS config: `20-hypervisor-setup` (eight stages in
   `playbooks/hypervisor/`: base, virtualization, SELinux, storage, networks,
   firewall, web edge, tuning), `21-hypervisor-reboot-verify`,
   `22-images-download` (pinned images incl. the golden image).

Workflow "AutoDeploy-NGNG AAP deploy", once per AAP instance (survey
`deployment_version`, empty draws the next number):

3. AAP VM deploy: `30-deployment-version-assign`, `31-aap-vm-create` (golden
   image copy, virt-customize, fixed MAC, inventory host, login via jump).
4. AAP VM OS config: `40-aap-vm-os-config` (registration, update, snapshot
   before the install, shared CA, root filesystem, packages).
5. AAP software deployment: `11-dns-record-publish` with the AAP name (job
   template 50), `51-aap-certificate-request` (Let's Encrypt),
   `52-aap-install` (containerized installer with vault secrets, snapshot
   after the install, vhost, public URL proof).
6. AAP platform config: `60-aap-platform-config` (hub and MCP tokens into the
   vault, subscription, credential type and credentials).
7. AAP base content config: `70-aap-base-content-config` (hub remotes and
   repositories, default EE, Galaxy and registry credentials, syncs), ending
   with the use case deployer: job template "Deploy UseCases to AAP"
   (`usecase-deploy.yml`) and per use case a workflow of the three generic
   nodes infra → config → content (`usecase-aap-infra/-config/-content.yml`):
   infra runs the use case's own playbook for what lies outside AAP, if it
   declares one; config applies what changes AAP itself (`aap-config/`);
   content applies what AAP uses (`aap-content/`, incl. EDA, activations
   enabled). No use case job template is launched.

Parts 3–4 use generic `vm_` variables; the AAP values sit in the CaC.

Started by hand: `instance-retire.yml` (removes the named versions' VM,
snapshots, vhost, certificate, inventory host, DNS records, vault tokens and
live list entry) and `aap-cac-apply.yml` (applies one use case's CaC tree to
the deploying AAP).

## Before you run it

- **It erases the target server.** There is no undo.
- It refuses to start unless the survey confirms the server number, its primary
  IP and the phrase `ERASE <server number>`, and unless the target is absent
  from the protected production address list.
- Reaching SSH afterwards does not prove the installation is fresh, because the
  provider's rescue system also answers on SSH. The stage itself proves the
  installation finished: it waits until the server presents exactly the host
  keys the provider reports for the last installation. There is no separate
  identity stage.
- Variables and confirmations are listed in the playbook header. Secrets belong
  in the private companion repository, encrypted.

## State of verification

Strict lint and syntax check pass, also inside the AAP execution environment on
ansible-core 2.20.10. Both workflows have run end to end from bare metal on the
test hypervisor (v30, 22 November 2026).
