# Design decisions history

## Instance retire as a separate job, explicit versions only — 2026-11-17

Decided with the user on 17 November 2026, built as
`playbooks/instance-retire.yml`, 6 tasks instead of 69 rated task lines in the
old host DNS retire.

- Retiring is a separate job template, started by hand, never part of a
  deployment workflow: several instances may be live in parallel, and only a
  person decides which one is finished. The playbook is unnumbered for that
  reason.
- Explicit versions only: the survey field `retire_versions` takes text such
  as `v24, v25`. Every version must be in the live list, at least one is
  needed, and the counter's next number is never accepted.
- The automatic wiped-server mode is dropped. After a reinstall the previous
  version stays live and in DNS until it is retired by hand; this is intended.
- Version 1 cleans two things: every A record in the zone with the version as
  a whole label (`hactar.v24`, later also `aap.v24`, never `hactar.v240`), then
  the version in the live list of `state/deployment_counter.yml`, written with
  the same SHA-guarded contents API call as the counter job, comments and the
  next number unchanged. DNS goes first, so a failed delete leaves the version
  live and a rerun finds its names again. Missing records are fine.
- Every later stage that creates something per instance (VMs, vhosts,
  certificates, AAP inventory entries, generated secrets, monitoring) adds its
  own cleanup step to this job when it is built.
- Removing the Red Hat registration is planned once a Red Hat API token
  exists.
- Everything is found by the version in its name; there is no extra registry.
- The list write has no skip condition: the assertion already guarantees that
  every named version is live, so the list always changes. A rerun after a
  successful retire is refused at the assertion, before any write.

## Reboot and verify in one playbook, no pre-reboot verify — 2026-11-17

Decided with the user on 17 November 2026, built as
`playbooks/07-host-reboot-verify.yml`, 15 tasks instead of about 78 (the old
verify, 66 task lines run twice and counted once, plus the old reboot, 12).

- The verify before the reboot is dropped. Host setup proves login and sudo
  itself, and every module reports its own success; reading the same state
  back a minute later adds nothing. Only the run after the reboot shows
  something new.
- The final reboot is kept as the real persistence test. Service enablement,
  the rasdaemon drop-in, sysctl.d, tmpfiles.d, libvirt autostart, the sshd
  drop-in, permanent firewall rules and the SELinux configuration only take
  effect at boot. The rasdaemon start race was only visible after a reboot.
- Reboot and check are one playbook. It runs as `ansible` on the port the SSH
  stage wrote to the AAP inventory, without a fallback to port 22 or root: a
  fallback would hide a broken SSH stage.
- Checks after the boot: SELinux mode, services running and enabled, the
  certificate renewal timer, libvirt networks and storage pools in their
  configured state with autostart, sysctl and KSM values, then from the
  execution environment: ports 80 and 443 answer, port 22 is closed on the
  public address, and a real root login on the managed port is refused (only
  "Permission denied" passes; a successful login, a closed port or a timeout
  fail). The root attempt runs last; our own public addresses are in
  fail2ban's `ignoreip`, so it cannot get AAP banned.
- Expected values are shared so setup and check cannot drift: sysctl and KSM
  values, the services and timers, and the default SELinux mode moved from the
  setup stages into `playbooks/vars/host-tuning.yml`, `host-services.yml` and
  `host-selinux.yml`, read by the setup stages and the check. Networks and
  pools are already inventory variables. The setup stages behave as before;
  the web edge starts nginx, fail2ban and the renewal timer in one task, so
  host setup has 48 tasks instead of 50.
- The 80 GiB free-space check on `/opt` moves to the image download step,
  where the space is used.

## Generate an SSH keypair for the ansible account — 2026-11-17

Decided with the user on 17 November 2026.

- No keypair for the automation account existed in the new repositories. The
  old AAP machine credential's key cannot be exported and belongs to the old
  setup, so a new keypair is generated.
- Type ed25519.
- The private and the public key are stored in `secrets_vault.yml`
  (`vault_ansible_ssh_private_key`, `vault_ansible_ssh_public_key`).
- The public key is also in the provider key list (`vault_robot_ssh_keys`), so
  the reinstall installs it for root and the bootstrap stage can connect.
- The same key later fills the AAP machine credential for user `ansible` and
  the `ansible` user's authorized keys.

## Host access bootstrap, two SSH addresses, port in the AAP inventory — 2026-11-17

Decided with the user on 17 November 2026.

- New stage `03-host-access-bootstrap.yml` right after the reinstall; the
  conversion, update and host setup move to 04, 05 and 06. It runs as root on
  port 22 (keys only), creates the `ansible` account (fixed uid, exclusive
  keys, `NOPASSWD` sudo), proves login and sudo as `ansible`, then turns off
  root, password and keyboard-interactive login. Reason: the host is exposed to
  the internet from the first minute, and the old WF1 ran as root through most
  of its run. Everything later runs as `ansible`, the AAP machine credential
  user. `NOPASSWD`, because with key-only access a sudo password adds little
  and would be one more secret to store and inject.
- `host-connect.yml` is removed. The SSH stage records the managed port as
  `ansible_port` on the AAP inventory host (`ansible.controller.host`, merged
  with the existing host variables read through the `controller_api` lookup),
  and the reinstall stage removes it again before the erase, after the
  confirmations. Reason: the inventory states the port instead of every job
  probing for it.
- The users stage no longer proves an admin login (the bootstrap did);
  `host_admin_user` is gone.
- sshd listens on exactly two addresses: the public address on the managed
  port and the internal libvirt bridge on port 22. Reason: VMs on the internal
  network, AAP's among them, reach the hypervisor on internal port 22, which
  production shows in active use. The dual-port safety net stays: public 22 is
  dropped and closed on the public zone only after login on the managed port is
  proven. The libvirt firewalld zone allows ssh by default.
  `net.ipv4.ip_nonlocal_bind = 1` lets sshd bind the bridge address when it
  starts at boot before libvirt has created the bridge. The hardening settings
  are defined once and shared by the bootstrap and the SSH stage.
- The managed SSH port is open to everyone (the user logs in from changing
  addresses); `host_management_sources` is gone.
- fail2ban ignores our own public addresses (`host_trusted_public_ips`).
  Reason: a real check that root login is refused attempts that login from AAP
  and must not get AAP banned.
- `ansible.controller` 4.8.11, the version proven with the previous
  implementation, is added to the collection requirements.
- The post-reboot verification is still under discussion and not built.

## Host setup in ten native stages — 2026-11-17

Decided with the user on 17 November 2026, stage by stage, and built as
`playbooks/05-host-setup.yml` importing `playbooks/host/01-base.yml` to
`10-tuning.yml`, 53 tasks and 6 handlers instead of about 368 tasks.

- Dropped: the replacement safeguards stage (its data became variables), every
  repeated profile, target and empty-host proof, the old rclone stage (rclone
  moves to the image download), CodeReady Builder (no package needs it),
  the outbound production block, the production-only half of nginx,
  `python3-certbot-nginx`, rsync and zstd, the timesync role (RHEL default) and
  host-side Python for libvirt networks.
- Base: EPEL, short hostname `<label>-vXY`, `/etc/hosts`, timezone, SELinux
  tools, admin tools (the user is their use case), sysstat, smartd and
  rasdaemon. The rasdaemon start is serialized with a drop-in because RHEL 9
  still ships 0.6.7 with the start race of upstream issue 74, which was seen on
  the rebuilt host.
- SELinux stays permissive (demo environment). Storage pools and the internal
  libvirt network come from plain variables and templates; the Windows network
  and NTFS packages move to the Windows use case; internal DNS names carry no
  version.
- Users: the run switches to the admin user and proves login and sudo before
  SSH is hardened. SSH: sshd first listens on 22 and the managed port, the run
  proves login and sudo on the managed port, then 22 is removed and closed.
  The managed port is private and comes from the vault. `host-connect.yml`
  chooses port and user per run, so nothing passes between workflow nodes.
- Web edge: nginx base, certbot and fail2ban in one playbook; use cases bring
  their own vhosts and jails. One shared public Diffie-Hellman file.
- Tuning: byte-based write limits (2026-10-29 incident) and KSM through
  tmpfiles.d, not TuneD, so nothing can override the write limits.
- Firewall rules removed from CaC are not removed from hosts yet; handling
  removals is a separate to-do.

## RHEL update in two tasks, as its own playbook — 2026-11-17

Decided with the user on 17 November 2026. The update stage updates all
packages from fresh metadata and reboots through a handler if anything changed.
It is its own playbook so it can be reused as a plain update job. Dropped: the
facts re-read and the final RHEL, kernel vendor and address check (the
conversion stage has just proved RHEL 9 with a Red Hat kernel, and an update
from Red Hat repositories cannot change that; a failed reboot fails the reboot
module itself), three more empty-host proofs, two more registration checks and
set_stats bookkeeping.

## Conversion to RHEL in fourteen native tasks — 2026-11-17

Decided with the user on 17 November 2026. The stage checks for Rocky Linux 9
or RHEL 9 on x86_64 (the check carried over from the dropped preflight), ends
on a host that already runs RHEL, updates Rocky and reboots if anything
changed, adds the official convert2rhel repository and signing key, installs
convert2rhel, runs `convert2rhel -y` with credentials in a temporary root-only
file that is always removed, reboots, reads facts again and requires RHEL 9
with a Red Hat kernel.

Dropped: the separate `convert2rhel analyze` run (the conversion runs the same
checks first), the EULA acceptance flag (the project's own gate; convert2rhel
has no such argument, and launching the job template is the decision), the
subscription registration check (convert2rhel fails if registration fails, and
the following update stage needs the RHEL repositories anyway), and around a
hundred task lines of repeated empty-host proofs, VM isolation code WF1 never
ran, cleanup of files a reinstall had already wiped, and saved-approval logic.
`infra.convert2rhel` 1.2.1 selects only EL7 and EL8 repositories, so native
tasks are used.

Kept on purpose: the kernel vendor check. Facts report RedHat as soon as the
release files are swapped, but the kernel is a separate package that only
takes effect after the reboot. A conversion that broke in between, or a boot
menu still defaulting to the old kernel, would show RHEL 9 in facts while the
Rocky kernel runs. The kernel version cannot tell them apart, since both end in
`.el9`; only the vendor of the running kernel's package can. It is one
read-only query that catches a half-finished conversion where it happens,
instead of as odd behaviour in later stages. convert2rhel normally sets the
RHEL kernel as default and rolls back on failure, so this is rare.

## The counter also lists the live instances — 2026-11-17

Decided with the user on 17 November 2026. Several instances may be live at
once, and DNS is not to be the single source of truth for which ones. The
counter file therefore has two fields: `autodeploy_deployment_counter_next` and
`autodeploy_deployment_instances_live`, seeded with `[25]`. The deployment
version job adds the number it assigns to the list in the same write that
advances the counter, so no value passes between jobs. The separate job that
retires an instance removes it. A version stays listed if a deployment fails
after the counter ran, until it is retired. The check task refuses to assign a
number the list already contains.

This replaces the earlier rule that the counter is the only runtime state
with no stored history.

## Host DNS publishes only; deleting old names is separate — 2026-11-17

Decided with the user on 17 November 2026. The host DNS stage has four tasks:
build `<label>.vXY.<zone>` from the deployment version, read the A record at
GoDaddy, create or update it when address or TTL differ, and wait until Google
public DNS returns exactly the host address. It never deletes a name. Several
instances may be live at once, so deleting an older name is a separate job and
must never assume that the previous version is obsolete.

Dropped from the old stage: three repeated derivations of the name, four
repeated target checks, a second provider read after the write, the wait for
the authoritative name servers (a 425-line script; the retrying public lookup
covers it) and the refusal to overwrite a record pointing elsewhere (the
counter never reuses a version). GoDaddy has no maintained Ansible module, so
`ansible.builtin.uri` calls the v3 API.

## The counter job writes its own counter file — 2026-11-17

Decided with the user on 17 November 2026. The deployment version job reads
`state/deployment_counter.yml` in the private repository through the GitHub
contents API, publishes the number as `autodeploy_deployment_version` and
writes the number plus one back in the same job. It does not hand the number to
the "Store generated secret" job template: the only-one-writer rule covers
generated secrets, and the counter is not a secret. One job avoids passing a
value between workflow nodes, where every failure on 16 November 2026 occurred.
The cost is that the Git write credential is attached to a second job template.

Five tasks: read the file and its blob SHA, take the number, require 1 to 99
and the expected line format, write the next number against the read SHA, then
publish the version. The version is published only after the write succeeded.
The SHA makes GitHub reject a concurrent change instead of losing an update. A
rerun after a successful write skips a number, which is harmless. A missing
file fails the read; the playbook never recreates the counter.

## Separate secrets from plain CaC definitions — 2026-11-17

Decided with the user on 17 November 2026. Each use case keeps all secrets and
environment details in one encrypted `secrets_vault.yml`, with `vault_`
prefixed names. All other CaC files are plaintext and reference those names.
Environment details (IP addresses, hostnames, server numbers, usernames) count
as sensitive under the owner's rules and are encrypted even though they are not
secrets in the narrow sense.

Reasons: plaintext definitions can be read and reviewed without decrypting,
encrypted files stay small, and Git stores readable deltas instead of a full
new copy on every change, which is what bloated the previous repositories. The
cost is one indirection per sensitive value; Ansible decrypts vault files
transparently, so applying the CaC is unchanged. A leak check for addresses and
server numbers in plaintext files runs before commits.

Converting the files exposed a latent bug that encryption had hidden:
credential type injectors and a runtime host variable used `{{ }}` templates
meant for AAP, which applying the CaC would have evaluated. They are now marked
`!unsafe`.

The deployment counter is plaintext too, in `state/deployment_counter.yml`,
seeded with 26: v25 was taken by the last deployment and the old counter
reads 26. The number is public anyway through DNS names, and keeping it plain
removes the need to expose the vault password to a playbook for re-encryption.
How the counter job writes it was decided later the same day (see above).

## Prove the installation finished by matching host keys — 2026-11-17

Decided with the user on 17 November 2026. The old trust stage
(`wf1-trust.yml`) and prerequisites stage (`wf1-prerequisites.yml`) were
analysed and dropped entirely, and with them the planned identity stage.
Standard Ansible and AAP do not pin host keys; AAP jobs run with host key
checking disabled by default. No known_hosts pinning or key publication is
therefore carried over.

Instead the reinstall stage ends with three tasks: read the host keys the
provider reports for the last installation, read the host keys the server
presents, and assert that both sets of SHA256 fingerprints are equal. The
rescue system has different host keys, so this also definitively tells the
rescue system from the finished installation.

`ansible.builtin.uri` is used because no `community.hrobot` module returns the
host keys of the last installation; the call is read-only.
`ansible.builtin.shell` runs `ssh-keyscan | ssh-keygen -lf -` because no module
reads a server's host key without connecting to it. Both run on the
controller; nothing runs on the managed node.

## Wait for the provider to consume the installation request — 2026-11-17

Decided with the user on 17 November 2026. The provider installs from its
rescue system, which also runs OpenSSH on port 22, so the `wait_for` on SSH
could succeed while the installer was still running. Hetzner documents the
automatic installation only as activate, reset, completes automatically; the
manual installimage flow boots the rescue system, installs, then reboots, and
the previous implementation guarded against exactly this. In the last real run
the server was ready about six minutes after the reset.

A tenth task now polls `community.hrobot.boot` with `regular_boot: true` in
check mode until it reports no change, meaning no installation configuration is
still active. Check mode makes no provider change. No helper code was needed.

Whether the provider clears the configuration when the installer starts or when
it finishes is not documented. The identity stage therefore remains the real
proof: it must compare the server's host keys with the keys the provider
reports for the last installation, retrying until they match.

## Agree the repository structure and deployment layers — 2026-11-17

Decided with the user on 17 November 2026. This is a demo environment;
safeguards are added only where they fix an actual problem.

**Layers.** An AAP deployment is: bootstrap RHEL, configure that RHEL from
CaC, bootstrap-install AAP, configure the AAP base from CaC, then deploy use
cases onto that AAP from CaC. Use cases include the jobs that run the demos.

**Two repositories.**

- Public `aap-autodeploy-configascode`: playbooks, execution environment
  definitions, examples. No values.
- Private `aap-autodeploy-configascode-secrets`: all CaC, meaning AAP objects
  together with their values, as many small Ansible Vault files split by
  target and use case rather than a few large ones, plus the deployment counter.

**Naming link.** A use case uses one short name in both repositories, for
example `usecases/aiops/` in each, and the same suffix on its CaC variables.

**State.** The only runtime state kept is the deployment counter (vXY), marked
as written by automation. The current host keys and machine ID are
automation-maintained but belong to CaC. Everything else is derived. The old
install journal is not carried over: the survey ERASE confirmation guards the
wipe, and destructive workflow nodes are never retried automatically.

**PAH and collections.** PAH configuration is AAP CaC, split into base and
per-use-case parts. Collections are not stored in any repository. Certified
collections from Red Hat Automation Hub are preferred, synced into PAH;
community collections only where no certified one fits. Base collections sync
at base configuration; use-case collections come from that use case CaC.

**Execution environments.** EE definitions live in the public repository with
documented build instructions. Images are built ahead of time, stored next to
the VM images on the shared drive, downloaded with pinned checksums, and
imported into the PAH container registry once PAH is up. Nothing is built at
deployment time. Where the build runs is not decided yet; a later use case will
rebuild all EEs.

**Generated secrets.** Base configuration holds the vault credential and a Git
write credential for the secrets repository. Only one job template, "Store
generated secret", uses the write credential; use cases call it rather than
pushing themselves. The active AAP is the only writer; a newly built AAP is a
passive configuration target until it takes over.

**AutoDeploy is a use case.** Deploying an AAP is itself a use case, installed
like any other, so every new AAP can build its successor.

## Refactor the provider reinstall stage — 2026-11-17

Decided with the user on 17 November 2026, after a rated review of all 439 task
lines in the existing host rebuild workflow.

The provider reinstall was 24 tasks across `playbooks/00-hetzner-robot.yml` and
two included files. Seven did essential work. The rewrite in
`playbooks/00-provider-reinstall.yml` has nine tasks and uses only
`community.hrobot` and `ansible.builtin`.

Removed, with the reason for each:

- The rescue and regular-boot branches. The workflow only ever used
  `install_linux`, so both were always skipped.
- The supported-operation and credential assertions. They validated a menu of
  four actions that no longer exists, and emptiness the modules report anyway.
- Seven of the nine write guards. Key syntax and duplicate-name checks tested
  static vault data that the provider API validates. The server number was
  confirmed three separate times.
- The installation readback and its eleven assertions. They re-read what the
  preceding module call had just set and returned.
- The closing debug message. The job result already reports the outcome.

Kept and merged: the ERASE phrase confirmation became a third condition on the
target confirmation assertion rather than its own task. A `when` condition was
rejected for this, because a false condition skips an assertion silently and
would fail open.

Hardcoded: `reset_type: hardware`. The variable defaulted to `software` in
group variables, which this hardware does not support, and a run failed on
exactly that. With the value fixed in the task, the mistake cannot recur, and
the separate reset-method availability check was dropped with it.

Added: an explicit `wait_for` on SSH. The old implementation hid this wait
inside a custom Python script in a later stage. Answering on SSH does not prove
the installation is fresh; the identity stage still has to verify the changed
host keys and machine ID.

Implementation commit: `6678bd1`.

Collection requirements name no source, so private automation hub resolves them
through AAP rather than the repository pinning a public endpoint.

Not yet validated against a real server. Strict lint passes against local
ansible-core 2.16.19. A syntax check passes inside the AAP execution
environment used by the old workflow, `ee-proj-aap-cac` at digest
`sha256:e6451a10…`, on ansible-core 2.20.10 with Python 3.12.15, with
deprecation warnings enabled and none reported. `reset_type: hardware` is a
valid choice in community.hrobot 2.8.0.
