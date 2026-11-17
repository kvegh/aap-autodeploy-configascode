# TLDR

Rebuild a physical host and an AAP instance from configuration as code. This
repository is a refactor of the previous implementation, keeping what it proved
and dropping the rest. See [README.md](README.md) for the full picture.

## What exists today

One stage: `playbooks/00-provider-reinstall.yml`, nine tasks, reinstalling a
confirmed dedicated server with Rocky Linux 9.

## Before you run it

- **It erases the target server.** There is no undo.
- It refuses to start unless the survey confirms the server number, its primary
  IP and the phrase `ERASE <server number>`, and unless the target is absent
  from the protected production address list.
- Reaching SSH afterwards does not prove the installation is fresh. The identity
  stage, not yet written, verifies the changed host keys and machine ID.
- Variables and confirmations are listed in the playbook header. Secrets belong
  in the private companion repository, encrypted.

## State of verification

Strict lint and syntax check pass. It also passes a syntax check inside the
AAP execution environment on ansible-core 2.20.10. It has never been executed
and has no AAP job template yet. The previous implementation remains the working one.
