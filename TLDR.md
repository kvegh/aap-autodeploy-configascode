# TLDR

Rebuild a physical host and an AAP instance from configuration as code. This
repository is a refactor of the previous implementation, keeping what it proved
and dropping the rest. See [README.md](README.md) for the full picture.

## What exists today

Five stages:

- `playbooks/00-provider-reinstall.yml`, thirteen tasks, reinstalls a confirmed
  dedicated server with Rocky Linux 9.
- `playbooks/01-deployment-version-assign.yml`, five tasks, takes the next
  deployment version (vXY) from the counter in the private repository and
  writes the number plus one back, adding the assigned version to the list of
  live instances.
- `playbooks/02-host-dns-publish.yml`, four tasks, publishes the host name for
  that version at GoDaddy and waits until public DNS resolves it.
- `playbooks/03-rhel-convert.yml`, fourteen tasks, converts Rocky Linux 9 to
  RHEL 9 with `convert2rhel` and proves a Red Hat kernel runs.
- `playbooks/04-rhel-update.yml`, two tasks, updates all packages and reboots
  if anything changed.

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
ansible-core 2.20.10. No stage has run for real; only the reinstall stage has an AAP job
template. The previous implementation remains the working one.
