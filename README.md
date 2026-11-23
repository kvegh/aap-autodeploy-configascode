# AAP AutoDeploy, Configuration as Code

Rebuild a physical host and an Ansible Automation Platform instance from
configuration as code, through AAP job templates and workflows.

Quick overview: [TLDR.md](TLDR.md).

## This repository is a refactor

It replaces [`aap_autodeploy_configascode`](https://github.com/kvegh/aap_autodeploy_configascode)
and the host automation in `hactarconfig`. Those work, and the rebuilt host and
platform they produce are real. They are also much larger than the task needs.

A review of the host rebuild workflow on 17 November 2026 counted 439 rated
task lines. 67 of them, 15 percent, did essential work. 284, 65 percent, were
repeated assertions, evidence bookkeeping, or defensive checks of state that
Ansible and AAP already guarantee. The same check appeared up to seven times in
one playbook. Around 500 lines of orchestration lived in custom Python, and two
helper scripts were copied onto the managed host and executed there.

Every failure during the rebuild on 16 November 2026 came from that extra
machinery passing values between workflow nodes, not from the work itself.
Provider reinstall, conversion, host setup, DNS and image staging each
succeeded on the first attempt.

This repository keeps what the old one proved and drops the rest.

### Principles

- Certified and maintained collections first. Custom code needs a stated gap.
- No helper scripts copied onto managed hosts. Facts come from `setup`,
  `service_facts` and `package_facts`.
- One check in one place. A condition is asserted where it is acted on.
- Values that pass between workflow nodes get names that nothing else uses,
  and are never named after a variable a later play registers.
- Safety requirements are kept: production is excluded by address, the exact
  target is confirmed before an erase, and a fresh installation is verified
  before anything trusts it.
- Configuration is separated from execution code. This repository holds
  execution code only; every configuration value (packages, services,
  sysctl and KSM values, SELinux mode and booleans, sshd and fail2ban
  settings, paths, ports, sizes and timeouts that are design choices) lives in
  the private companion repository's CaC, as variables of the AAP inventory
  AutoDeploy-NGNG. Playbooks carry no defaults for configuration values: a
  missing value fails where it is used. Secrets come from AAP credentials.

### Parts

Seven parts, two workflows. Playbooks are `PS-<subject>-<action>.yml` and
job templates `AutoDeploy-NGNG PS <what>`, where P is the part and S the
order inside it.

| Part | Playbooks (job template number) |
|---|---|
| 1 Hypervisor install | `10-provider-reinstall`, `11-dns-record-publish`, `12-hypervisor-access-bootstrap`, `13-rhel-convert`, `14-rhel-update` |
| 2 Hypervisor OS config | `20-hypervisor-setup` (stages in `playbooks/hypervisor/`), `21-hypervisor-reboot-verify`, `22-images-download` |
| 3 AAP VM deploy | `30-deployment-version-assign`, `31-aap-vm-create` |
| 4 AAP VM OS config | `40-aap-vm-os-config` |
| 5 AAP software deployment | `11-dns-record-publish` with the AAP name (50), `51-aap-certificate-request`, `52-aap-install` |
| 6 AAP platform config | `60-aap-platform-config` |
| 7 AAP base content config | `70-aap-base-content-config`, ending with the use case deployer |

Workflow "AutoDeploy-NGNG Hypervisor deploy" runs parts 1–2 (10 → 22,
behind the erase confirmation survey), once per hypervisor. Workflow
"AutoDeploy-NGNG AAP deploy" runs parts 3–7 (30 → 70, survey
`deployment_version`), once per AAP instance. Data passes between parts only
through the version artifact `autodeploy_deployment_version`, the AAP
inventory AutoDeploy-NGNG and the private vault (part 6 stores the new AAP's
tokens, part 7 reads them). Parts 3–4 use generic `vm_` variables; the AAP
values sit in the CaC.

### Status

Refactored, reviewed and committed:

| Stage | Tasks before | Tasks after |
|---|---|---|
| Provider reinstall | 24 | 14 |
| Deployment version | 34 | 5 |
| Host DNS | about 45 | 3 |
| Host access bootstrap | part of host setup | 20 |
| Conversion to RHEL | about 120 | 14 |
| RHEL update | about 20 | 2 |
| Host setup (8 stages) | about 368 | 32 + 5 handlers |
| Reboot and verify | about 78 | 14 |
| Image download | 68 | 9 |
| AAP VM create | about 147 | 8 |
| AAP DNS and certificate | about 50 | 2 (DNS reuses the host DNS stage) |
| AAP VM OS config | part of AAP install | 7 + snapshot task file + 1 handler, plus the RHEL update stage |
| AAP install | 49 | 11 + snapshot and download task files + 2 handlers |
| AAP platform config | part of AAP base config | 10 + download task file |
| AAP base content config | about 300 (74 + about 230 in task files) | 10 + download task file |
| Instance retire | 69 | 12 + 1 handler |

The reinstall stage removes `ansible_port` from the target's AAP inventory
host before the erase, so later jobs reach the rebuilt host on port 22, and
ends by proving the server presents the host keys the provider reports for the
last installation. The deployment version stage reads
the plaintext counter in the private repository, publishes it as
`autodeploy_deployment_version` and writes the next number back, with the
assigned version added to the list of live instances, through the GitHub
contents API. A launch may instead name a live version in
`deployment_version` (for example to rerun a deployment); it is published
as it is and the counter is not changed. The host DNS stage creates or corrects the name
`host_fqdn` (in the CaC: `<label>.<zone>`, without the deployment version) at GoDaddy and waits until public DNS returns the host
address. It never deletes names: older versions may still be live instances;
the instance retire job removes them.

The host access bootstrap runs as root on port 22, minutes after the
reinstall, because the host is exposed to the internet from the start. It
creates the automation account `host_automation_user` (the AAP machine
credential user, `ansible` in the CaC) with its keys and sudo rule, and the
personal accounts with their keys (admins in `wheel`, which gets its own sudo
rule; both rules come from the CaC), proves login and sudo as the automation
account,
then turns off root, password and keyboard-interactive login and limits login
to the declared accounts (`AllowUsers`). All SSH configuration lives in this
stage. Still on Rocky Linux, it also moves SSH to the managed port: the sshd
role labels the port `ssh_port_t` (SELinux enforces on the fresh
installation), the firewall role makes sure firewalld runs, binds the public
interface (the interface of the default IPv4 route) to the public zone and
opens the managed port to everyone; with `net.ipv4.ip_nonlocal_bind` set,
sshd listens on the public address on port 22 and the managed port and on the
internal libvirt bridge address on port 22 (the bridge appears with the host
setup's libvirt network; nonlocal bind also covers boot), the run proves login
and sudo on the managed port, records it as `ansible_port` on the AAP
inventory host, then drops the public port 22 and closes it on the public
zone. The bridge sits in firewalld's libvirt zone, which allows ssh. Every
later stage runs as the automation account on the managed port; the conversion is in place
and keeps it.

The conversion stage updates Rocky Linux, runs `convert2rhel -y` with
temporary root-only credentials and proves the running kernel is Red Hat's.

The RHEL update stage updates all packages and reboots if anything changed.

The hypervisor setup (`20-hypervisor-setup.yml`) runs eight stages from `playbooks/hypervisor/`:
base (EPEL, names, timezone, admin tools, hardware monitors), virtualization,
SELinux (permissive), VM storage (the pool directories and the plain image
directory `/opt/images`, `root:libvirt` 2775), libvirt networks, firewall
(web ports), the web edge (nginx base, certbot, fail2ban) and tuning. The stage files keep
their numbers; 06 (users) and 08 (SSH) moved into the host access bootstrap.
fail2ban never bans our own public addresses. File templates live in the
public repository; their values come from the private repository's CaC. nginx is installed and configured but neither enabled nor started;
the AAP deployment starts it and brings the AAP login rate limit
(`limit_req_zone` in `conf.d`, used by the AAP vhost). fail2ban runs with the
sshd jail only.
Use cases add their own nginx vhosts, basic-auth files and fail2ban jails for
their nginx logs.

The reboot and verify stage (`21-hypervisor-reboot-verify.yml`) reboots the finished
host and proves that the host setup survives a boot: SELinux mode, services
running and enabled, the certificate renewal timer, libvirt networks and
storage pools in their configured state with autostart, sysctl and KSM values,
then from outside: port 22 is closed on the public address and a real root
login on the managed port is refused. The web ports are not checked, since
nginx runs only after the AAP deployment. It runs as the automation account on the inventory port with no
fallback to port 22 or root. The old workflow ran a separate verify before
the reboot as well; it is dropped, since the host access bootstrap proves
login and sudo itself and only the run after the reboot shows something new.
Every value that a setup stage sets and this stage checks is defined once, as
a CaC inventory variable, and read by both.

The instance retire job (`instance-retire.yml`) is not part of any deployment
workflow. Several instances may be live at once, so a person names the
versions to retire in its survey, for example `v24, v25`. Only versions in the
live list are accepted, never the counter's next number. Everything an
instance owns is found by the version in its name, without a separate
registry. It deletes every A record in the zone that has the version as a
whole label (`host.v24`, `aap.v24`, never `host.v240`), then removes the
versions from the live list with the same SHA-guarded write as the deployment
version stage. Before DNS it deletes each version's AAP VM snapshots, stops,
undefines and deletes the AAP VM and disk on the hypervisor (names from the
CaC templates), removes the AAP vhost and deletes the
certificate of the AAP name with `certbot delete`, then removes the VM's host
from the AAP inventory. VMs, hosts and DNS go before the list, so a failed delete leaves the version live and
a rerun finds its names again. After a reinstall the previous version stays
live and in DNS until it is retired by hand; this is intended. Extension rule:
every later stage that creates something per instance (VMs, vhosts,
certificates, AAP inventory entries, generated secrets, monitoring) adds its
own cleanup step to this job when it is built. Removing the Red Hat
registration is a planned follow-up that needs a Red Hat API token.

The image download stage (`22-images-download.yml`) runs as the automation
account on the inventory port. If at least one image of the CaC list is
missing, it first requires the configured free space (`images_min_free_gib`)
on `/opt`, read with `df`; with every image present the check is skipped, so a
rerun after a complete download does not fail on the space the images use.
It then installs rclone, writes the Google Drive rclone configuration from
the AAP credential "Google Drive rclone-NGNG" (the same Drive access
production uses, kept on purpose for the demo environment), downloads each
image of the CaC list `images_download` that is not there yet (`rclone
copyto` with the CaC retry counts, `creates:`, async with a 30 second poll) and requires the pinned SHA-256 of every image.
A file with a wrong checksum is not deleted automatically: the failure names
it and says to delete it by hand and run the job again, because a rerun skips
files that exist. Files keep root ownership and mode 0644; the image
directories, their SELinux label and owner come from the storage stage, and
libvirt takes over a VM's disks when it starts. The image list and its
checksums are required CaC in the private repository.

The AAP VM stage (`31-aap-vm-create.yml`), the first step of the AAP
deployment, copies the golden image `vm_base_image` into the vms pool,
grows it with `qemu-img resize` and prepares it offline with `virt-customize`:
the FQDN `vm_fqdn` (`aap27.vXY.<zone>`, the AAP version from the CaC) as
host name, the automation account with exactly its declared keys and its sudo
rule, an empty machine-id, `subscription-manager clean` and a SELinux
relabel. No cloud-init. `community.libvirt.virt_install` then imports it as
`vm_name` (`aap27-vXY`) with the CaC sizes, the RHEL 9 OS type (virtio),
autostart and the fixed MAC of the reservation `aap` in
`net_internal.reservations`. Once the VM is defined, a rerun touches neither
the disk nor the VM. It then records the VM as host `vm_name` in the AAP
inventory with the host variables of the CaC dict `vm_connection`
(`ansible_host` from the reservation `aap`, `ansible_fqdn`, and
`ansible_ssh_common_args` with a ProxyCommand through the hypervisor's managed
port), adds it to the run with `add_host` from the same dict, waits for the
connection and proves login and sudo with `ping`. The machine credential
covers both hops. The filesystem is grown in the installation stage. The
instance retire job removes the VM, its disk and its inventory host. Only the
first AAP VM (fixed reservation) is built; further VMs are future work.

The AAP name `vm_fqdn` is published by the host DNS stage itself:
`11-dns-record-publish.yml` runs unchanged under its own job template, whose
extra variables set `host_dns_record` and `host_fqdn` to the AAP name. The
record points to the hypervisor's public address, like the host name.
`51-aap-certificate-request.yml` then enables and starts nginx on the
hypervisor (the web edge stage left it stopped) and runs `certbot certonly
--webroot` for the AAP name into the web edge stage's webroot
`/var/lib/letsencrypt`, with the account address `host_certbot_email` from the
CaC; `creates:` skips it once the certificate exists. No temporary nginx
configuration: the AAP vhost is written once, after the AAP install. The
instance retire job deletes the certificate.

Step 15 of the old host rebuild workflow, the old replacement playbook run on
its own as the last node, is dropped: it switched on VM network isolation and
made libvirt depend on that firewall, which the new design does not want.
The old `aap.yml` dispatcher, which every job template ran first, is dropped
as well: each job template points directly at its own playbook.

The VM OS configuration stage (`40-aap-vm-os-config.yml`) takes the
deployment version as an extra variable and targets the VM's inventory host
(`aap*-<version>`) and the hypervisor group. It registers the VM with `rhc`
(the activation key of the conversion stage, no AAP repository), updates it
by importing `14-rhel-update.yml` with `rhel_update_hosts` set to the VM's
host pattern, takes a cold libvirt snapshot (shut down,
`virsh snapshot-create-as`, start, wait for SSH; names from
`vm_snapshot_names`; a rerun skips an existing snapshot without shutting
the VM down), adds the shared AutoDeploy CA to the
VM's trust store, grows the root partition and XFS, resolves the AAP name to
the VM in `/etc/hosts` and installs `ansible-core`. No vault is needed.

The installation stage (`52-aap-install.yml`) downloads the private
repository with the GitHub credential (task file shared with the CaC apply
job), decrypts the installer secrets (`vaulted_aap_installer`) with the vault
credential, renders the installer inventory (growth topology, bundle install,
MCP server, `ca_tls_cert`/`ca_tls_key` from the shared CA, no hub seeding)
into the bundle directory and runs the containerized installer as
`aap_install_user`. A Gateway ping on the VM against the shared CA proves the
install. The cold snapshot after the install follows, then the hypervisor gets the AAP
vhost (Let's Encrypt certificate, upstream verified with the shared CA, login
rate limit; `nginx -t` before the reload), and a ping of the public URL proves
the path end to end.

The platform configuration stage (`60-aap-platform-config.yml`) runs on the deploying
AAP's execution environment against the new AAP's public URL, logged in as
`aap_base_user` (admin) with the installer's admin password from the vault.
When the vault has no tokens for the AAP VM yet, it creates two Gateway
tokens (read scope for the AAP's own hub, write scope for MCP clients) and
writes them into the vault file through the GitHub contents API: decrypted
with the vault password kept in the vault itself, the AAP's key added under
`vaulted_aap_instance_tokens`, encrypted again, sent with the blob SHA. It
attaches the subscription with the CoP role `controller_license` (Red Hat
service account, filter from the CaC), then reads the private repository's
AAP config tree (`aap_config_cac_dir`, `aap-config/`, outside `usecases/` so the CaC apply
job never applies it to the deploying AAP) with `filetree_read` and applies it
with `dispatch`: what changes AAP itself, namely organization Default with
three Galaxy credentials on the AAP's own hub (hub token), the collection
remotes rh-certified, validated and community (requirements files next to
them) and the deploying AAP's hub as an anonymous container registry.

The base content stage (`70-aap-base-content-config.yml`) logs in the same
way, takes the AAP's tokens from the vault and applies the base content tree
(`aap_content_cac_dir`, `aap-content/base/`): the collection repositories
and distributions (no sync), the repository of the default EE, the registry
credential on the AAP's own hub, the default EE pinned by digest, the Git
write credential type and the vault, Git write and own-AAP (MCP token)
credentials the deployer uses. Then it syncs the EE repository itself (the CoP sync role
stops waiting after about 50 s), sets `DEFAULT_EXECUTION_ENVIRONMENT` and
syncs the three collection repositories, whose remotes carry requirements
files with exactly the collections the deployer project needs
(`collections/requirements.yml` plus dependencies). Last, it reads the use
case deployer tree (`aap_deployer_cac_dir`, `aap-content/usecase-deployer/`, a
sibling of the base content tree) and
applies only its inventory, project, job templates and workflows; the
project's first sync installs the collections from the new hub (6 to 14 min; waited for up to 25 min).
Route timeouts keep their defaults.

The use case deployer (job template "Deploy UseCases to AAP",
`usecase-deploy.yml`) runs, for each use case chosen in its survey, that use
case's workflow on the AAP it lives on, passing the target AAP chosen in the
second survey question (an AAP VM name with tokens in the vault). Each
workflow has the same three generic nodes, infra → config → content
(`usecase-aap-infra.yml`, `usecase-aap-config.yml`, `usecase-aap-content.yml`);
only the workflow variable `usecase_name` differs. Each node downloads the
private repository (vault and use case catalog), then the use case's -secrets
repository with the GitHub token the catalog names (one vault variable per
repository) and loads the repository's vault (`env_variables.yml` at its
root). The sections follow the user's rule of 23 November 2026:

- infra = outside AAP (hosts, VMs, hypervisor, RHEL under AAP, client
  machines): the node runs the use case's own playbook if its catalog entry
  declares `infra_playbook` (a path in its -secrets repository, run with the
  use case vault), otherwise it does nothing. None declares one yet.
- config = AAP itself, from `aap-config/`: settings, organizations, teams,
  local users, authenticators and maps, OAuth applications, hub collection
  remotes and EE registries (sync sources and tokens), credential types that
  several use cases share.
- content = what AAP uses with external systems, from `aap-content/`: first
  hub collection remotes and repositories and EE repositories, synced
  explicitly (no 50 s limit); then credential types of this use case only,
  credentials, EEs, labels, inventories, hosts, groups, projects; then the
  projects the job templates use are updated and job templates (with
  surveys), workflows, schedules and the EDA objects follow (credentials,
  projects, event streams, decision environments, rulebook activations).

Both AAP nodes read their section with `filetree_read` and apply it with
`dispatch`, logged in to the target with the target's MCP token. Deploy
means ready to demonstrate: rulebook activations are created as their tree
says (enabled), no job template or workflow of a use case is launched.
Deployed to v31 (23 November 2026): AIOps self-healing, RHEL sniper patching,
CCNG, Disconnected Windows, Assistant team, Automation Orchestrator.

The web edge's port-80 catch-all serves `/.well-known/acme-challenge/` from
the certbot webroot for every name and answers everything else with 444, so
Let's Encrypt issuance and renewal need no per-name port-80 server.

Both workflows have run end to end from bare metal on the test hypervisor
(v30 on 22 November 2026, before the restructure into parts).

## Conventions

- Playbooks are `PS-subject-action.yml` (part, order inside the part; see
  Parts). A playbook reused by a later part keeps its number; the reusing job
  template gets its own. Jobs outside the deployment workflows, started by
  hand, are unnumbered
  (`instance-retire.yml`, `aap-cac-apply.yml`), and so are the use case
  deployer's (`usecase-*.yml`).
- Variables use `snake_case` with a prefix naming their source: `robot_` for
  provider API inputs and confirmations, `provider_` for values this automation
  derives at run time, `counter_` for the deployment counter, `dns_` for the host DNS stage, `rhsm_` for subscription inputs,
  `host_` for host configuration, `aap_` for the AAP inventory the stages
  update, `vm_` for the VM of parts 3–4 (generic names, AAP values in the CaC), `retire_` for the instance retire job, `cac_` for the CaC apply
  job, `usecase_` for the use case deployer. Variables the CoP roles define keep their CoP names (`dir_orgs_vars`,
  `aap_hostname`). Registered
  values carry the stage name, for example `rhel_convert_`. Values published
  for later workflow nodes start with `autodeploy_`.
- Task names start with a capital and an imperative verb, and say what the task
  achieves rather than which module it calls.

## Idempotency

Reinstalling a server is inherently not idempotent: it destroys the running
system. Running the reinstall stage twice reinstalls twice. This is why the
provider writes are guarded by the target confirmations and the production
exclusion, and why a workflow must not retry this stage automatically. The
stages that follow are written to be repeatable.

Check mode is not supported for the provider writes. The query and the
assertions are read-only and safe to run at any time.

## Layout

    playbooks/                one playbook per stage
    playbooks/hypervisor/     hypervisor setup stages, imported by 20-hypervisor-setup.yml
    playbooks/hypervisor/templates/ file templates, structure only (values come from the
                              private repo's CaC)
    playbooks/hypervisor/files/ static files (public, no secrets)
    playbooks/tasks/          task files imported by more than one play
    playbooks/templates/      file templates of the AAP stages
    collections/  collection requirements

## Applying the CaC

`playbooks/aap-cac-apply.yml` applies the CaC of one use case (`cac_usecase`,
for example `autodeploy`) with the Ansible Community of Practice collections
`infra.aap_configuration` 4.11.1 and `infra.aap_configuration_extended`
4.13.1. The private repository holds each use case as the tree the CoP export
`filetree_create` writes, under `usecases/<use case>/`; the playbook loads its
encrypted `env_variables.yml`, reads the tree with `filetree_read`
(`controller_configuration_filetree_read_layout: create`) and applies it with
`dispatch`. Export and apply therefore use the same layout and variable names.

What the job template needs (in AAP: "AutoDeploy-NGNG CaC apply"):

- **Private repository.** An AAP job sees only its own project, so a second
  AAP project for the private repository is no use. The playbook downloads
  the repository itself: the GitHub contents API tarball of `cac_repository`
  (owner/name) at `cac_repository_ref`, with the account and
  token of the existing credential "GitHub AutoDeploy CaC Secrets-NGNG"
  (`counter_git_username`, `counter_git_token`), into a scratch directory it
  removes at the end. On the command line, `cac_repository_path` points at a
  local checkout instead and nothing is downloaded. The playbook has no
  defaults: the job template's extra variables set `cac_repository`,
  `cac_repository_ref` (`main`) and `cac_usecase` (`autodeploy`); command
  line runs pass them with `-e`.
- **Vault password.** An AAP vault credential with the private repository's
  vault password; AAP passes it to `ansible-playbook`, which decrypts
  `env_variables.yml`.
- **AAP connection.** A "Red Hat Ansible Automation Platform" credential. The
  `ansible.controller` modules take the login from its environment variables
  (`CONTROLLER_*`, `AAP_*` or `TOWER_*`). The playbook sets `aap_hostname` and
  `aap_validate_certs` from `AAP_HOSTNAME`/`AAP_VALIDATE_CERTS` or
  `CONTROLLER_HOST`/`CONTROLLER_VERIFY_SSL`, because the CoP version check
  reads only the `AAP_*` names and otherwise targets 127.0.0.1.
- **Collections.** `dispatch` always runs the CoP version check, which uses
  `ansible.platform`; it and `ansible.hub` are dependencies of the CoP
  collections and come from private automation hub with them.

- **Trust in AAP's certificate.** The job calls AAP by its name, whose
  certificate the AAP installer's own CA signed. The AAP host's RHEL trusts
  that CA (added once by hand on 2026-11-19), and AAP's job setting "Paths to
  expose to isolated jobs" mounts the host's `/etc/pki/ca-trust` into every
  job container, so the EE trusts it too. Future AAPs (AAP deploy workflow) get one shared
  AutoDeploy CA through the installer (decided, not built).

Command line example:

    ansible-playbook playbooks/aap-cac-apply.yml \
      -e cac_usecase=autodeploy \
      -e cac_repository_path=../aap-autodeploy-configascode-secrets \
      --vault-password-file <file>

## Verification

| Check | Result |
|---|---|
| `ansible-lint --profile production --strict` | pass, 0 failures, 0 warnings |
| `ansible-playbook --syntax-check` | pass |
| CaC apply: `filetree_read` of the private tree inside the EE (CoP 4.11.1 / 4.13.1) | object counts equal the previous CaC files; vaulted references render; `!unsafe` templates stay literal |
| CaC apply: run without network, AAP address pointed at a closed port | reads and decrypts the tree, stops at the CoP version check before any change |
| Syntax check inside the AAP EE (ansible-core 2.20.10, Python 3.12.15, image `ee-proj-aap-cac@sha256:e6451a10…`) | pass, no deprecation warnings |
| `reset_type: hardware` valid in community.hrobot 2.8.0 | yes (`software`, `hardware`, `power`, `manual`) |
| APME (commit `9c4372f`, core 2.20; native, opa and ansible validators) | no defects; one lead open, deviations accepted below |
| Executed against a real server | both workflows end to end on the test hypervisor (v30, 22 November 2026) |
| AAP job templates and workflows | all parts' job templates, both workflows and the job template "AutoDeploy-NGNG CaC apply" exist in AAP and the CaC |

Accepted APME deviations:

- `L081` numbered playbooks: the number is the execution order and is the
  documented naming convention.
- `L005` community collection: no certified collection covers the Hetzner Robot
  API; `community.hrobot` is the maintained one.
- `L039` possibly undefined variables: supplied by inventory and credentials,
  listed in the playbook header. `attribute` is a filter argument.
- `L049` loop variable prefix and `L078` bracket notation: style only; the
  current form is more readable and has no conflicting names.
- `R101` parameterized command on the host key scan: the only value is the
  target IP, already asserted and passed through `quote`; runs on the controller.

- `R105` outbound transfer with parameterized destination (deployment
  version): the destination is the configured private repository on GitHub.

Deployment version, tested offline only: inside the EE, a simulated contents
API response built from the real counter file rendered the expected write
(assigns v26, writes 27, keeps the file comments). Then run for real on
17 November 2026 through AAP job template "AutoDeploy-NGNG 01 Deployment
version": assigned v26, committed the counter at 27 with live list [25, 26],
no token in the job output.

Host DNS, tested with GoDaddy replaced by a local stand-in inside the EE and
the public lookup run for real against an existing name: creates a missing
record, leaves a correct one alone, corrects a wrong one. The real GoDaddy
read and write have not run. The GoDaddy v3 API takes the token only as a
bearer header, which Ansible does not hide; run the job template at normal
verbosity, where module arguments are not printed.

Conversion to RHEL, accepted APME findings: `R108` privilege escalation (the
stage changes the operating system); `R104`/`R106` downloads from a URL
variable (the official Red Hat repository file, fixed in the play); `R109`
signing key from a variable (the official Red Hat key URL); `R101` command with
a parameter (the kernel version from facts, read only); `L071`/`L072` inline
content without backup (a three-line credentials file that is removed again);
`L004` deprecated `rpm_key`: a false positive, ansible-core 2.20.10 shows no
deprecation for it. Not executed yet.

Host setup, accepted APME findings: `R108` privilege escalation, `R113`/`R114`/
`R109` package names, paths and keys from variables (inherent to host setup);
`L038` unresolved certified roles (APME cannot see them;
ansible-lint and the EE resolve them); `L068` `lineinfile` on `/etc/hosts` (keeps
the installer's other lines); `L071` inline content for the password file;
`L004` `rpm_key` false positive. Tested by lint, EE syntax check and a render
test of the firewall rules, sshd settings, jail file and network XML. Not
executed yet. The end-to-end test host is the test hypervisor (decided 18 November
2026); its current AAP instance may be erased for it.

Host access bootstrap with the move to the managed port, and the SSH listen
addresses, accepted APME findings: `L032`/`L033`/`L101` re-defining
`ansible_user` and `ansible_port` in the bootstrap (the deliberate connection
switch), `L039` the managed port and AAP inventory name (inventory
variables), `R108` and `L038` as above. APME cannot
download `ansible.controller` from public Galaxy and aborts the scan, so the
scanned copy used its upstream `awx.awx`, which has the same `host` module and
`controller_api` lookup; lint and the EE syntax check used `ansible.controller`
4.8.11 itself. Tested by a render test inside the EE of the bootstrap's
transitional and final sshd settings, the SSH stage's settings, the firewall
port item, the fail2ban `ignoreip` line and the inventory variable merge
and removal (full, port-less and empty host variables). Not executed yet.

Reboot and verify, accepted APME findings: `R101` parameterized command on the
root login attempt (the only values are the host address and managed port from
the inventory, passed as separate arguments; it runs in the execution
environment), `L005` for `community.libvirt` (no certified collection reads
libvirt networks and pools), `R108`, `L039`, `L049`, `L078` as above. Also
reported, on the conversion and update stages: `L012` `state: latest` (a full
update is the purpose; marked `noqa package-latest`), `L021` file mode on a
file removal (meaningless for `state: absent`), `R401` an informational list of
download sources. Tested by lint, EE syntax check and a render test inside the
EE of every assertion with simulated results, including the root login
outcome (refused passes; connection refused, timeout and a successful login
fail), and of the tuning stage's KSM file (byte-identical to before). Not
executed yet.

Image download, accepted APME findings: `R101` parameterized command (rclone
with the remote and destination from the CaC list, passed as separate
arguments; no rclone module exists); `R114` file changes with a parameterized
path (the rclone configuration path, fixed in the play); `L071`/`L072` inline
content without backup (the configuration comes from the AAP credential
and is rewritten every run); `R108`, `L039`, `L049`, `L078`, `L081` as above. Tested
inside the EE against the container itself, with rclone replaced by a stand-in
that copies a local file and the `dnf` backend stubbed (the EE has no dnf):
a fresh run downloads both test files and passes the checksums; a rerun
downloads nothing and passes; a corrupted file fails the checksum check with
its name in the message; a free-space requirement larger than the disk fails
at the free-space check before any download when one image is missing, and is
skipped, with the run passing and downloading nothing, when all images are
present. Not executed on a host yet.

Instance retire, accepted APME findings: `R105` outbound transfer (the
configured private repository on GitHub, as for the deployment version);
`L033` re-defined variables (the `counter_` facts carry the same names and
meaning as in the deployment version stage, in a separate play); `L051` Jinja
spacing, a false positive on the `|` alternation inside the regular
expression string, gone when that string is removed; `L039`, `L049`, `L078` as
above; `M005` as the open lead below. Tested inside the EE against local
stand-ins for the GitHub contents API and GoDaddy: retiring `v24` deletes
`host.v24`, `aap.v24`, `v24` and `HOST.V24` but not `host.v240`,
`v240.host` or `host.v24x`, and writes the live list without it, comments
and the counter line unchanged, against the read SHA; `24 ,V25` is normalised
to `v24, v25`; a live version without a record deletes nothing and still
updates the list; a version not live, the next number, a malformed or an empty
input, and a rerun after a successful retire fail at the assertion before any
write; a multi-page record list fails before any delete. The real GoDaddy and
GitHub calls have not run.

Open: `M005` flags a registered value used in a template as possibly untrusted
on ansible-core 2.19 or later. The same lead appeared for the counter file
read; the real AAP run of that stage passed, so it is closed there. The values are plain strings, so this is expected to be harmless; only a real run can confirm it.
APME's gitleaks, collection health and dependency audit validators did not run.

Review requirements come from
[Agent_generated_code_review](https://github.com/kvegh/Agent_generated_code_review).
No review report exists yet; a stage cannot be accepted before it has run
end to end through AAP.

All configuration belongs in the private companion repository's CaC, never
here. There only real secrets (passwords, private keys, tokens) are Ansible
Vault encrypted; other values are plain text in that private repository.
