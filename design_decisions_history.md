# Design decisions history

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
How the counter job writes it is still to be decided.

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
