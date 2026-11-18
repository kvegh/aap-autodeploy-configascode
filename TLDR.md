# TLDR

Rebuild a physical host and an AAP instance from configuration as code. This
repository is a refactor of the previous implementation, keeping what it proved
and dropping the rest. See [README.md](README.md) for the full picture.

## What exists today

Nine stages:

- `playbooks/00-provider-reinstall.yml`, fourteen tasks, reinstalls a confirmed
  dedicated server with Rocky Linux 9 and removes the managed SSH port from its
  AAP inventory host.
- `playbooks/01-deployment-version-assign.yml`, five tasks, takes the next
  deployment version (vXY) from the counter in the private repository and
  writes the number plus one back, adding the assigned version to the list of
  live instances.
- `playbooks/02-host-dns-publish.yml`, four tasks, publishes the host name for
  that version at GoDaddy and waits until public DNS resolves it.
- `playbooks/03-host-access-bootstrap.yml`, eight tasks, creates the `ansible`
  account, proves its login and sudo, and turns off root and password login
  minutes after the reinstall.
- `playbooks/04-rhel-convert.yml`, fourteen tasks, converts Rocky Linux 9 to
  RHEL 9 with `convert2rhel` and proves a Red Hat kernel runs.
- `playbooks/05-rhel-update.yml`, two tasks, updates all packages and reboots
  if anything changed.
- `playbooks/06-host-setup.yml`, 47 tasks in ten stages, configures the
  hypervisor: base, virtualization, SELinux, storage, networks, users,
  firewall, SSH (managed port on the public address, port 22 on the internal
  bridge, port recorded in the AAP inventory), web edge and tuning.
- `playbooks/07-host-reboot-verify.yml`, fifteen tasks, reboots the host and
  proves the setup survived the boot: SELinux, services, libvirt networks and
  pools, kernel values, web ports open, port 22 closed and root login refused
  from outside. Expected values are shared with the setup stages.
- `playbooks/08-images-download.yml`, nine tasks, requires 80 GiB free on
  `/opt` when an image is missing, installs rclone, downloads each missing VM
  image from Google Drive and requires its pinned SHA-256. A corrupt file fails with its name and is
  deleted by hand before a rerun.

One job outside the deployment workflow, started by hand:

- `playbooks/instance-retire.yml`, six tasks, retires the versions named in
  its survey: deletes every A record with the version as a label and removes
  the versions from the live list. Every later stage that creates something
  per instance adds its own cleanup step here.

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
ansible-core 2.20.10. Only the deployment version stage has run for real, through
its AAP job template; the reinstall stage has a job template but has not run.
The previous implementation remains the working one.
