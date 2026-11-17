# TLDR

Rebuild a physical host and an AAP instance from configuration as code. This
repository is a refactor of the previous implementation, keeping what it proved
and dropping the rest. See [README.md](README.md) for the full picture.

## What exists today

One stage: `playbooks/00-provider-reinstall.yml`, thirteen tasks, reinstalling a
confirmed dedicated server with Rocky Linux 9.

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

Strict lint and syntax check pass. It also passes a syntax check inside the
AAP execution environment on ansible-core 2.20.10. It has never been executed
and has no AAP job template yet. The previous implementation remains the working one.
