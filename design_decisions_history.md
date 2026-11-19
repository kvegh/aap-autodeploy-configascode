# Design decisions history

## AAP VM create stage, deployment version input, retire of the VM — 2026-11-19

Decided by the user on 19 November 2026 (CET; exact times in bead
bead-wd-0gnz.41), implemented 19 November 2026. Public implementation commit
`470b68d` (v0.0.42); private CaC commit `d778bd3` (v0.0.42). Job template 206 —
AutoDeploy-NGNG 09 AAP VM create created through the API (inventory
AutoDeploy-NGNG, project 193, EE ee-proj-aap-cac, Credential 62); Project 193
synced once (update 5259). No CaC apply, no job launched, no workflow changed.

User decisions:
- Image: the golden image `/opt/images/goldimg-vm-1.disk.qcow2` (step 08),
  copied into the vms pool.
- Old baseline technologies only: copy (`remote_src`), `qemu-img resize`
  (60G), `virt-customize`, virt-install (here through
  `community.libvirt.virt_install`). No cloud-init. Filesystem growth stays in
  the later installation step.
- virt-customize: FQDN as host name; the automation account with the key of
  Credential 62 (public key from `host_authorized_keys`) and its sudo rule as
  in the host access bootstrap; empty machine-id, `subscription-manager
  clean`, SELinux relabel. No host-key generation or pinning (host-key
  checking off).
- virt-install: sizes from the CaC (4 vCPU, 20480 MiB, 60G), network
  `internal`, fixed MAC read from the reservation `aap` in
  `net_internal.reservations`, autostart, RHEL 9 OS type (virtio).
- Naming: `aap27-vXY`, FQDN `aap27.vXY.<dns_zone>`; the AAP version
  (`aap_version_short`) is CaC config.
- Deployment version: optional launch variable `deployment_version`
  (neutral name, chosen by the implementing agent); set, it must be live and
  is published without drawing; empty, step 01 draws as before.
- Access: static CaC inventory host with ProxyCommand through hactar-ng and
  the same credential for both hops. Not built: a static host name cannot
  carry the version (open question to the user).
- No firewall rule VM to hypervisor.
- Retire: AAP VM (destroy and undefine) and its disk removed per retired
  version, before DNS and the live list; the play stops the whole run on
  failure.
- After create: wait for the connection through the jump and a login plus
  sudo check. Not built, depends on the inventory host above.

Implementer choices: playbook name `09-aap-vm-create.yml`; reruns skip resize
and virt-customize once `/etc/libvirt/qemu/<name>.xml` exists; disk keeps the
image's mode (`mode: preserve`); osinfo `rhel9.0` (not checked against
hactar-ng's osinfo-db); JT timeout 3600 s. Checks: ansible-lint production
strict 0/0; render tests of the version expression, the virt-customize
arguments and the CaC templates under loop variables.

## AAP host CA trust playbook removed — 2026-11-19

19 November 2026, approved by the user. The CA was added to the AAP host's
trust store once by hand over SSH (user's permission), so
`playbooks/aap-host-ca-trust.yml` (commit `0522679`) and the inventory host
`aap-platform` were never used. Removed both. Future AAPs (WF2) get the shared
CA through the installer instead; the need recurs there, but not in this form.

## Reboot check: libvirt daemons need not be running — 2026-11-19

19 November 2026, 11:30 CET, within the user's run authorization; found by reading
step 07 before its first run. RHEL 9's modular libvirt daemons are socket-activated
and exit after 120 s idle, so asserting them "running" after the reboot could fail
on a healthy host. Step 07 now requires them enabled only; its network and pool
reads already prove they answer.

## CaC apply uses a fixed async dir — 2026-11-19

Implemented 19 November 2026, 10:22 CET. Public commit `b9b6312` (v0.0.39).
Cause: JT 204 (job 5175) failed in `collect_async_status` with "could not
find job": `aap_configuration_async_dir` defaults to empty and ansible-core
2.20 in the EE did not find the async results. Fix: `aap-cac-apply.yml` sets
`aap_configuration_async_dir: /tmp/.ansible_async`.

## AAP host trusts the AAP installer CA — 2026-11-19

Decided by the user on 19 November 2026 (CET; exact time not recorded),
implemented 19 November 2026, 09:57 CET. Public commit `0522679` (v0.0.38); private CaC commit
`596dbaa` (v0.0.37). Cause: JT 204 — AutoDeploy-NGNG CaC apply (job 5146)
failed with CERTIFICATE_VERIFY_FAILED on https://aap.supercorp.at, whose
certificate the AAP installer's own CA signed; neither the AAP host's RHEL
nor the EE trusted that CA.

- The RHEL host under AAP trusts AAP's CA: `playbooks/aap-host-ca-trust.yml`
  copies the installer CA, already on the host, into the trust anchors
  (`remote_src`, so it stays out of Git) and runs `update-ca-trust` in a
  handler (no module exists for it).
- Jobs see it without change: AAP's "Paths to expose to isolated jobs"
  already holds `/etc/pki/ca-trust:/etc/pki/ca-trust:O` (installer value),
  so the planned extra read-only mount of `extracted` was not added.
- Fewest objects: host 51 — aap-platform in the existing inventory
  AutoDeploy-NGNG, in no group; WF1 plays target only `autodeploy_target`
  and `localhost`, so no WF1 job can reach it. Variables
  `ca_trust_source_path`, `ca_trust_anchor_name` are its host variables.
- Not done: the job template and SSH credential. No AAP credential is known
  to hold a key authorized for `aap_service` on the AAP VM; it must be added
  by the user first.
- Future AAPs (WF2): one shared AutoDeploy CA, certificate plain and key
  vault-encrypted in the private repository, given to the installer as
  `ca_tls_cert`/`ca_tls_key`; the VM setup trusts it before the installer
  runs. Decided, not implemented; no CA generated.
- Rejected: EE rebuild with the CA; routing AAP calls via the public address
  (Let's Encrypt certificate); copying the Let's Encrypt certificate every 90
  days; merging a CA bundle at run time from a credential.

## Remaining self-defined configuration moved to the CaC; two simplifications — 2026-11-19

Decided by the user on 19 November 2026 (CET; exact time not recorded),
implemented 19 November 2026, 06:51 CET (v0.0.37). Implementation commit
`01d5ec5`; the private repository's CaC commit is `2c0c96a` (v0.0.36). Settles the points the previous entry left open.

- Moved into the CaC (inventory AutoDeploy-NGNG, plain), no defaults in code:
  - automation account name: `host_automation_user` (`ansible`, the user of
    the AAP machine credential, unchanged); used for the account, its key
    lookup, its sudoers file, the connection switch and, as a job-time
    template, in `host_ssh_allow_users`
  - sudo rules: `host_automation_sudo_rule` (job-time template on
    `host_automation_user`) and `host_wheel_sudo_rule`
  - host naming: `host_short_name`, `host_dns_record`, `host_fqdn` as job-time
    templates from `dns_host_label`, the deployment version and `dns_zone`;
    one definition for the base and DNS stages
  - counter file path: `counter_file_path` (deployment version, instance retire)
  - internal network: `net_internal.forward_mode`, `nat_port_start`,
    `nat_port_end`
  - rclone retries: `images_rclone_retries`, `images_rclone_low_level_retries`
  - nginx catch-all ports: `host_web_edge_http_port`, `host_web_edge_https_port`
  - `cac_usecase` and `cac_repository_ref`: removed from `aap-cac-apply.yml`,
    set in the extra variables of job template "AutoDeploy-NGNG CaC apply"
    (CaC and AAP, id 204); command line runs pass them with `-e`.
- Kept in code: third-party facts (EPEL and convert2rhel repository and GPG
  key URLs).
- Simplified: the internal network's `<bridge>` drops `stp='on' delay='0'`
  (libvirt's defaults, same result); the image download's free-space check
  reads `df --output=avail -B1G /opt` instead of matching mount candidates,
  still only when an image is missing (`df` rounds up to whole GiB).
- Rollout: as before, the AAP inventory gets the new variables only when the
  CaC is applied; until then WF1 jobs of this version fail on undefined
  variables. Job template 204's extra variables were updated in AAP directly
  (only that change), so the CaC apply job itself works before the next apply.

Checks: `ansible-lint --profile production --strict` 0 failures, 0 warnings;
syntax check of all playbooks in `ee-proj-aap-cac@sha256:e6451a10…`;
`filetree_read` of the private tree: every variable the playbooks reference is
defined, the job-time templates render (`hactar-v26`, `hactar.v26.supercorp.at`),
a list element template in inventory variables renders at job time (EE test);
`net-internal.xml.j2` and `00-default.conf.j2` render with the CaC values;
job template 204 from `filetree_read` matches AAP. Not executed in AAP.

## Configuration separated from execution code; only secrets encrypted — 2026-11-19

Decided by the user on 19 November 2026 (CET; exact time not recorded),
implemented 19 November 2026, 06:20 CET (v0.0.36). Implementation commit
`e7b2bf2`; the private repository's CaC commit is `540a2e2` (v0.0.35). Supersedes the 2026-11-19 choice of a fixed
play variable for `/opt/images` in `04-storage.yml` and the shared
`playbooks/vars/` files.

- Rule: all configuration lives in the private CaC; this repository holds
  execution code only. Playbooks read CaC inventory variables (inventory
  AutoDeploy-NGNG) directly and carry no defaults for configuration values;
  a missing value fails where Jinja uses it. No extra asserts were added.
- `playbooks/vars/` is removed (4 files) with every `vars_files`
  reference; its 10 values are inventory variables now
  (`host_selinux_default_state` became `host_selinux_state`, no default).
  The post-reboot check still reads the same variables as the setup stages;
  it derives the list of boot services from them plus `firewalld` (started
  by the firewall role).
- Moved from code into the CaC as well: `host_bootstrap_user` (root),
  `images_min_free_gib` (80), `images_download_timeout` (21600), `dns_ttl`
  (600), the base, virtualization and web edge package lists, SELinux policy
  and booleans, `host_storage_plain_dirs` (`/opt/images`), the firewall's
  public services, the fail2ban jails (whole dict, including ignoreip and
  the sshd port as job-time templates) and the bridge SSH port
  (`host_ssh_internal_port`, 22): 13 more variables. `AllowUsers` is no
  longer a stored template: the bootstrap adds it from
  `host_ssh_allow_users`, as it adds Port and ListenAddress.
- Kept in code as mechanics: module parameters, retries, reboot and wait
  timeouts, API endpoints, file locations of drop-ins, port 22 of a fresh
  installation and of the firewalld ssh service, the rasdaemon workaround,
  SELinux type `virt_image_t`, modes of root-only secret files, defaults for
  optional data keys or API answers.
- Left open for the user (not decided here): the automation account name
  `ansible` and the passwordless sudo policy for it and wheel; the software
  source URLs (EPEL, convert2rhel repository and GPG keys); the nginx base
  configuration and catch-all vhost (ports 80/443) and the internal network
  template's fixed values (NAT, port range, STP); the host naming scheme
  `<label>-<version>` / `<label>.<version>.<zone>`; the rclone retry counts
  and the `/opt` mount candidates of the free-space check; the counter file
  path in the private repository; `cac_usecase` and `cac_repository_ref`
  defaults of `aap-cac-apply.yml`, which are bootstrap inputs of the job that
  applies the CaC itself.
- The private repository now encrypts only real secrets; its non-secret
  values became plain text there (same decision, recorded there).
- Rollout: the AAP inventory gets the new variables only when the CaC is
  applied; until then WF1 jobs of this version fail on undefined variables,
  because the project updates on launch.

Checks: `ansible-lint --profile production --strict` 0 failures, 0
warnings; syntax check of all playbooks in `ee-proj-aap-cac@sha256:e6451a10…`
(ansible-core 2.20.10); `filetree_read` of the private tree in the same EE:
every variable the playbooks reference is defined by inventory variables,
credential injectors, surveys, job template extra variables or the
workflow artifact, and the inventory's job-time templates render. A
script-inventory test in the EE showed templates in inventory variables
render at job time. APME not run (not installed). Not executed in AAP.

## CaC apply job template — 2026-11-19

Requested by the user on 19 November 2026 (CET; exact time not recorded),
implemented 19 November 2026, 06:05 CET (v0.0.35). No playbook change; the
job template is defined in the private repository's CaC, commit `22b887e`
(v0.0.34).

- JT 204 — AutoDeploy-NGNG CaC apply runs `playbooks/aap-cac-apply.yml`
  from project AAP AutoDeploy-NGNG in EE ee-proj-aap-cac, inventory
  AutoDeploy-NGNG (holds `localhost` with a local connection), timeout
  3600 s, no simultaneous runs. Not a WF1 step.
- Only `cac_repository` (owner/name) is set as an extra variable;
  `cac_usecase` and `cac_repository_ref` keep the playbook defaults.
- Credentials: GitHub AutoDeploy CaC Secrets-NGNG, AAP API-NGNG and the
  deploying AAP's existing vault credential "aap_autodeploy_configascode-secrets
  CaC vault cred" (vault ID `cac`), whose password decrypts the private
  `env_variables.yml`. The vault credential is not part of the CaC: its value
  would have to be stored in the vault it unlocks.

## WF1 workflow template — 2026-11-19

Requested by the user on 18 November 2026 (CET; exact time not recorded),
implemented 19 November 2026, 06:05 CET (v0.0.34). Record commit
`9088373`; no playbook change. The workflow is defined in the private
repository's CaC, commit `e44b5cf`.

- Workflow 203 — AutoDeploy-NGNG WF1 chains the job templates WF1 00 to 08
  strictly on success, no failure or always branches: no playbook header asks
  for a cleanup step, and the reinstall stage must never be retried
  automatically. Simultaneous runs are not allowed.
- The workflow carries the same three-question survey as the reinstall job
  template (server number, IPv4, `ERASE <number>`). Workflow survey answers
  reach every node as extra variables, so the operator confirms once per run;
  storing answers on the node would remove the confirmation. The reinstall
  playbook still checks them against the configured target.
- The deployment version passes from node 01 to later nodes as a `set_stats`
  artifact (`autodeploy_deployment_version`), used by the host DNS stage and
  the base host stage inside host setup. This is the documented interface:
  after node 01 the counter holds the next number and all live versions, so
  later nodes cannot tell this run's version from it. No other node
  publishes or consumes artifacts.
- Project 193 — AAP AutoDeploy-NGNG was updated to `9f3e5d3` (project update
  5145, successful) before the workflow was created. The workflow was not
  launched: node 00 erases a live server.

## WF1: login rate limit moves to WF2, storage stage creates /opt/images — 2026-11-19

Decided with the user on 19 November 2026 (CET; exact time not recorded),
implemented 19 November 2026, 06:00 CET (v0.0.33). Implementation commit
`7d270c5`. No CaC change; the private repository records the decision too.

- The nginx login rate limit leaves WF1: it protects AAP's login, so it
  comes with WF2, like starting nginx. `09-web-edge.yml` no longer renders
  `conf.d/limits.conf`, `templates/limits.conf.j2` is removed, and
  `nginx_rate_limit_zone` / `nginx_rate_limit_rate` are no longer required.
  This closes the open item that these two were not set in the CaC.
  `nginx.conf.j2` never referenced the zone; the remaining base config
  (`nginx.conf`, `00-default.conf`) passes `nginx -t` (nginx 1.24, UBI 9).
  WF2 must bring back: `/etc/nginx/conf.d/limits.conf` with
  `limit_req_zone $binary_remote_addr zone=<zone>:10m rate=<rate>;` and the
  AAP vhost's `limit_req zone=<zone> burst=<burst> nodelay;` plus
  `limit_req_status 429;` on the login locations, as the old production
  templates do. A host already set up by an earlier WF1 run keeps its
  `limits.conf`; WF1 does not delete it.
- The storage stage also creates `/opt/images` as in production: `root`,
  group `libvirt`, mode `2775`, SELinux type `virt_image_t` (label rule and
  directory, in the same two tasks as the pool directories). It is a plain
  directory, not a libvirt pool. Simplest form chosen: a fixed play variable
  in `04-storage.yml`, no new CaC setting, since the path and owner are fixed
  production layout. The libvirt group comes with the virtualization
  stage's packages. The task count is unchanged (32 tasks, 5 handlers).
  `08-images-download.yml` already says the directories, label and owner
  come from the storage stage; its CaC destinations are under `/opt/images`.
  `07-host-reboot-verify.yml` checks pools, not directories; unchanged.
- Not in scope: production's `/opt/vmbackups` (`root:root`, `0755`,
  `virt_image_t`) is not created.

Checks: `ansible-lint --profile production --strict` 0 failures, 0 warnings;
syntax check of 06, 07 and 08 inside the AAP EE (`ee-proj-aap-cac@sha256:e6451a10…`)
pass without warnings.

## Credential type renamed to "AAP CaC repo read-write token" — 2026-11-19

Decided with the user on 19 November 2026 (CET; exact time not recorded),
implemented 19 November 2026, 05:55 CET (v0.0.32). The change is in AAP and
in the private repository, commit `fe897b7` (v0.0.31); this record is
commit `ae36704`. No playbook change.

- Credential type 50 — "GitHub Counter Writer-NGNG" is now "AAP CaC repo
  read-write token". Credential 61 — GitHub AutoDeploy CaC Secrets-NGNG uses
  it. The injected variables stay `counter_git_username` and
  `counter_git_token`, so `01-deployment-version-assign.yml`,
  `aap-cac-apply.yml` and `instance-retire.yml` are unaffected.
- Older entries below keep the old type name as history.

## WF1 base host trimmed: accounts and all SSH in the bootstrap, nginx installed but not started — 2026-11-18

Decided with the user on 18 November 2026 (CET; exact times not recorded),
implemented 19 November 2026, 05:50 CET (v0.0.30). Implementation commit
`230076f`; the CaC values that go with it are in the private repository,
commit `08561d1` (v0.0.29).

- Accounts and SSH: the personal account (`host_users`, keys, `wheel` with
  passwordless sudo) is created by `03-host-access-bootstrap.yml` next to
  `ansible`. All SSH configuration lives in that stage: `AllowUsers`
  (`host_ssh_allow_users`, exactly `ansible` and the personal account) is in
  the shared SSH settings from the first sshd change, so the login proof on
  the managed port runs with it; `net.ipv4.ip_nonlocal_bind` is set before
  sshd uses any address; the final listen addresses are the public address on
  the managed port and the internal bridge address on port 22. The interim
  `0.0.0.0` listen is gone: with nonlocal bind sshd listens on the public
  address on 22 and the managed port plus the bridge during the transition;
  the dual-port proof is unchanged. The bridge sits in firewalld's libvirt
  zone (ssh allowed); the public zone still closes port 22.
- `aap_hypervisor` leaves WF1 (user, keys, `AllowUsers` entry); it comes with
  the AAP installation in WF2.
- The host setup stages `06-users.yml` and `08-sshd.yml` are removed; the
  other stage files keep their numbers. The host setup has eight stages,
  32 tasks and 5 handlers; the bootstrap has 20 tasks.
- Web edge: nginx is installed and configured (base config, shared
  dhparams) but neither enabled nor started; WF2 starts it. The reload
  handler is gone (a reload of a stopped unit starts it); `nginx -t` stays.
  The shared basic-auth file (`host_nginx_htpasswd`) is no longer installed:
  the AIOps project, its only user, installs it with its own vhost.
- fail2ban keeps only the sshd jail (journal backend, no log file needed).
  The nginx jails and the `nginx-444` filter move to the AIOps project.
  Checked before removing: the EL9 nginx package ships its log files as
  `%ghost` (not created on install), and fail2ban 1.1.0 (EPEL 9) refuses its
  configuration when a jail finds no log file, so those jails would have
  broken fail2ban on a host where nginx never ran.
- Reboot and verify: nginx is no longer an expected running service, and the
  check that ports 80 and 443 answer from outside is removed, since nothing
  listens there until WF2. It has 14 tasks.
- Open: `nginx_rate_limit_zone` and `nginx_rate_limit_rate` (nginx base
  `limits.conf`) are still not set in the CaC; a WF1 rerun after WF2 changes
  the nginx base config without reloading nginx.

Checks: strict ansible-lint (production profile) 0 failures, 0 warnings on
the whole repository against the pinned collections; syntax check of the
bootstrap, host setup, web edge and reboot verify playbooks inside
`ee-proj-aap-cac@sha256:e6451a10…`, no deprecation warnings; render check of
the CaC in the same EE (accounts, `AllowUsers`, transitional and final
listen addresses). APME was not rerun. Not executed against a server.

## WF1 job templates named `AutoDeploy-NGNG WF1 NN <step>` — 2026-11-18

Decided with the user on 18 November 2026 (CET; exact time not recorded),
implemented 19 November 2026, 05:42 CET. Version v0.0.29, no playbook change;
the job templates live in AAP and in private CaC commit `83c18a6` of the
companion repository.

- One job template per playbook, named `AutoDeploy-NGNG WF1 NN <step>`, NN
  the playbook number: 00 Provider reinstall, 01 Deployment version, 02 Host
  DNS publish, 03 Host access bootstrap, 04 RHEL convert, 05 RHEL update,
  06 Host setup, 07 Host reboot verify, 08 Images download.
- Credentials follow each playbook header: 02 the DNS credential; 03 the
  machine credential and the AAP credential; 04 the machine credential and
  the subscription credential (which injects `rhsm_org_id` and
  `rhsm_activation_key`); 05 to 07 the machine credential; 08 the machine
  credential and the rclone credential.
- The image download job template's timeout (111600 s) exceeds the per-image
  `images_download_timeout` (default 21600 s) times the number of images
  (5) plus an hour for the checksums, so it never cuts off a download that
  is still inside its own limit.
- The GitHub credential is renamed from "GitHub Counter Writer-NGNG" to
  "GitHub AutoDeploy CaC Secrets-NGNG" (user decision, 18 November 2026,
  CET): it is the general read/write token for the private repository; the
  deployment version step writes the counter with it and
  `playbooks/aap-cac-apply.yml` reads the tree with it. The playbook's
  header and the README refer to the new name (v0.0.31); private CaC commit
  `3f39469` (v0.0.30). The variables `counter_git_username` and
  `counter_git_token` keep their names.

## Managed SSH port set in the host access bootstrap, public NIC detected, hactar-ng as test host — 2026-11-18

Decided with the user on 18 November 2026 (CET; exact time not recorded,
implemented 15:56 CET). Implementation commit `1593726` (v0.0.28), private
CaC commit `9abb493` of the companion repository.

- The managed SSH port value is set in the private vault (`host_ssh_port`).
- The switch to the managed port moves from the SSH stage of the host setup
  (`playbooks/host/08-sshd.yml`) into `playbooks/03-host-access-bootstrap.yml`,
  still on Rocky Linux, right after root and password login are turned off.
  The conversion (in place, keeps the setting), update, host setup, reboot
  check and image download then run on the managed port. Reason: one port for
  every stage after the bootstrap, set where the access is created.
- Kept from the earlier design: the dual-port safety net (22 and the managed
  port, login and sudo proven on the managed port, then 22 dropped), port 22
  closed on the public zone only, the managed port open to everyone, the port
  recorded as `ansible_port` on the AAP inventory host and removed by the
  reinstall. The bootstrap's job template therefore needs the AAP API
  credential; the host setup no longer does.
- Added for Rocky Linux: SELinux enforces on the fresh installation, so the
  sshd role labels the port `ssh_port_t` (`sshd_manage_selinux`, which calls
  the certified selinux role) before sshd binds it. Whether firewalld is
  installed and running on the provider's image is not proven by the old
  repository or its analyses (the old access step accepted it active or
  absent, and the old firewall stage installed it), so the bootstrap runs the
  certified firewall role, which installs, enables and starts it, binds the
  public interface to the public zone and opens the managed port. Its default
  public zone keeps ssh open until the proof.
- In the bootstrap sshd listens on all IPv4 addresses, not on the public
  address: `net.ipv4.ip_nonlocal_bind` is set only in the host setup, and
  without it sshd could start at boot before the address is up during the
  conversion and update reboots.
- The SSH stage of the host setup keeps what needs the libvirt bridge and the
  users: `net.ipv4.ip_nonlocal_bind`, the final listen addresses (public
  address on the managed port, bridge address on 22) and `AllowUsers`; 3 tasks
  instead of 10. The firewall stage keeps the web ports only. The reboot
  check is unchanged apart from comments: its checks stay correct. The
  conversion and update stages make no port assumption; they use the
  inventory port.
- `host_public_interface` is dropped. The host has a single NIC; the public
  interface is the gathered `ansible_facts['default_ipv4']['interface']`,
  used where the public zone is bound, now in the bootstrap.
- `net_internal` (bridge, MAC, domain, address, netmask, DHCP range,
  reservations, DNS hosts, UUID) is copied unchanged from the old hypervisor
  repository's vault into the private CaC. The source holds no deployment
  version or a name derived from one; the old per-deployment renaming only
  added versioned names at run time and is not carried over.
- The test server for the WF1 end-to-end run is hactar-ng; its current AAP
  instance may be erased for it.

Open, not decided: `AllowUsers` is applied in the host setup without its own
login proof (a missing `ansible` entry would only show at the reboot check);
the SELinux port label is expected to survive the in-place conversion, which
only a real run can show.

## Collections resolved only from private automation hub — 2026-11-18

Decided with the user on 18 November 2026 (15:38 CET). The project keeps
`collections/requirements.yml` without a source: every pinned collection and
its dependencies are added to private automation hub instead, Red Hat
certified and validated content from the hub's Red Hat remotes and community
content from Galaxy through the hub's community remote. The list the hub needs
is `pah_requirements_autodeploy` in the private repository; it is merged into
the shared hub remotes, never replacing them.

## WF1 credentials carried over from the old repositories — 2026-11-18

The remaining WF1 stage secrets were taken unchanged from the frozen old
sources and added to the private `env_variables.yml`: the GoDaddy API token,
the public DNS zone and host label, and the Red Hat subscription organization
ID and activation key from the old hypervisor repository, and the AAP API
token the old CaC used for the production AAP. Nothing was regenerated.

Three credentials carry them, each injecting only the variables its stage reads:

- `GoDaddy DNS-NGNG` (custom type of the same name, secret `api_token`,
  injects `dns_godaddy_token`) for `AutoDeploy-NGNG 02 Host DNS`.
- `Red Hat Subscription-NGNG` (custom type of the same name, `org_id` and
  `activation_key`, both secret because the organization ID is an environment
  detail; injects `rhsm_org_id` and `rhsm_activation_key`) for
  `AutoDeploy-NGNG 04 RHEL convert`.
- `AAP API-NGNG` (built-in type `Red Hat Ansible Automation Platform`, host
  and OAuth token, `verify_ssl` true) for the stages that call
  `ansible.controller`: `AutoDeploy-NGNG 00 Provider reinstall` and
  `AutoDeploy-NGNG 06 Host setup` (its SSH step records the managed port).

The DNS stage's non-secret inputs are inventory variables of
`AutoDeploy-NGNG`: `dns_zone` and `dns_host_label` from vaulted values, and
`dns_host_ipv4` from the same `vaulted_robot_expected_server_ip` as
hactar-ng's `ansible_host`, so the address is stored once.

The AAP token belongs to a personal admin token of the old automation and has
an expiry date; replacing it with a dedicated service account token is a
follow-up, not part of this change.

## CaC in the Community of Practice layout, applied by aap-cac-apply.yml — 2026-11-18

Decided with the user on 18 November 2026. The CaC in the private repository
uses the layout, variable names and secret convention of the Ansible
Community of Practice (CoP) collections `infra.aap_configuration` and
`infra.aap_configuration_extended`, at the latest versions 4.11.1 and 4.13.1.
Where a layout or naming choice comes up, the CoP defaults win over earlier
conventions of this project.

Why: the CoP export (`filetree_create`) and the CoP apply (`filetree_read`
followed by `dispatch`) then use the same tree and the same names, so an
export of the live AAP compares one to one with the repository, and the
standard CoP tooling, including `object_diff` for removing objects that are no
longer in the CaC, works without translation.

It supersedes two earlier decisions: the `_<use case>` suffix on CaC
variables (now plain CoP names such as `controller_credentials` and
`controller_templates`, the name `dispatch` reads for job templates) and the
`vault_` prefix (now `vaulted_`).

The tree: each use case is one directory, `usecases/<use case>/`, laid out as
`filetree_create` writes it with `omit_id: true`.
Organization-less objects (credential types) are files at its root;
everything else is under `<organization>/controller_<type>.d/`, one file per
object named after it, with inventories, hosts and groups in a subdirectory
per inventory. `filetree_read` reads it with
`controller_configuration_filetree_read_layout: create` and `dir_orgs_vars`
set to the use case directory. That role takes one root per run, so use cases
stay separate and are applied one at a time. Only `autodeploy` exists today.

Secrets: one vault-encrypted `env_variables.yml` per use case, the CoP env
variables file, holding `vaulted_*` values. Credential inputs use the name
`filetree_create` generates, `vaulted_controller_credentials_<credential
name>_<input>`; values that feed inventory variables keep a short name with
the new prefix (`vaulted_robot_expected_server_ip`). The values were carried
over unchanged; only the names changed. `secrets_vault.yml` is gone.

`pah_requirements_autodeploy` stays as it is, in `pah_requirements.yml` at the
use case root. It is not a CoP variable: the PAH stage of
`aap_autodeploy_configascode` reads `pah_requirements_<use case>` and merges
it into shared hub remotes, while the CoP `hub_collection_remotes` would
replace a remote's whole requirements list. `state/deployment_counter.yml`
stays where it is; the counter playbooks read that path.

The apply job is the public repository's `playbooks/aap-cac-apply.yml`,
unnumbered and started by hand. It downloads the private repository as a
GitHub API tarball (`cac_repository`, `cac_repository_ref`) with the existing
"GitHub Counter Writer-NGNG" credential, because an AAP job only sees its own
project; on the command line `cac_repository_path` points at a checkout
instead. The vault password comes from an AAP vault credential, the AAP
connection from a "Red Hat Ansible Automation Platform" credential. The
playbook sets `aap_hostname` and `aap_validate_certs` from `AAP_*` or
`CONTROLLER_*` environment variables, because the CoP version check reads
only the `AAP_*` names.

## Drive access as an AAP credential — 2026-11-18

Decided with the user on 18 November 2026. The Google Drive rclone
configuration is an AAP credential, never an inventory variable: the
credential type "Google Drive rclone-NGNG" (one secret, multiline field
`rclone_conf`) injects it as the extra variable `images_rclone_conf`, and the
credential "Google Drive rclone-NGNG" holds the value from the vault
(`vault_images_rclone_conf`). AAP stores it encrypted and hands it only to
the job that uses the credential. The image download stage reads
`images_rclone_conf`; playbooks do not read `vault_` names.

The earlier wiring, an inventory variable `vault_images_rclone_conf` read
directly by the playbook, was a mistake and was corrected before the CaC was
ever applied.

## hactar-ng in the AutoDeploy-NGNG inventory — 2026-11-18

Decided with the user on 18 November 2026. The host `hactar-ng` is added to
the new inventory `AutoDeploy-NGNG`, in the group `autodeploy_target` that
the host stages target. Its `ansible_host` is the expected server address
from the vault (`vault_robot_expected_server_ip`); the SSH port is the
default 22, so no `ansible_port` is set at first.

The SSH stage records the port it manages as `ansible_port` on this host
entry, and the provider reinstall removes it again. The inventory variables
`aap_inventory_name: AutoDeploy-NGNG` and `aap_inventory_host_name:
hactar-ng` name the inventory and host for those two tasks.

The `hactar-ng` entry in the old inventory is untouched.

## Images hactar-ng downloads — 2026-11-18

Decided with the user on 18 November 2026. hactar-ng downloads exactly five
images: `/opt/images/rhel-9.8-x86_64-kvm.qcow2`,
`/opt/images/goldimg-vm-1.disk.qcow2`, `/opt/images/windows-server-2022.qcow2`,
`/opt/images/windows-server-2025.qcow2` and `/opt/vms/zabbix-disk.qcow2`.
The disks of tra, the AAP VM and the test server (`tra-disk`, `aap-disk`,
`testserver1-disk`) are not downloaded. The five images are about 45 GiB
(44.9 GiB), within the 80 GiB free space the stage requires.

The list is the plaintext CaC `images_download` in the AutoDeploy inventory
variables of the private repository. The checksums were taken from the old
capture 20261101T205024Z. The remote path keeps the old capture layout:
`<remote root>/<capture id>/<destination without the leading slash>`.

The rclone configuration was copied unchanged from the old vault into the
new vault as `vault_images_rclone_conf`: the same Drive access production
uses, kept on purpose because this is a demo environment.

## Host accounts, SSH access and fail2ban trusted addresses — 2026-11-18

Decided with the user on 18 November 2026. The host has exactly three
accounts that can log in: `ansible` (created by the bootstrap),
`aap_hypervisor` (the AAP VM logs in with it through internal port 22) and
the owner's personal account. No other accounts are created. sshd
`AllowUsers` lists exactly these three. `aap_hypervisor` and the personal
account are in `wheel`, which has passwordless sudo. The personal account
name is an environment detail and lives in the vault.

The authorized public keys of `aap_hypervisor` and the personal account were
taken from their current `authorized_keys` on production hactar, option
prefixes included, and stored in the vault.

The fail2ban trusted addresses (`host_trusted_public_ips`) reference
production hactar's and hactar-ng's addresses that are already in the vault
for the provider stages. AAP reaches the hosts through those addresses via
NAT, so they cover AAP too, and no address is stored twice.

## Let RHEL assign user IDs — 2026-11-18

Decided with the user on 18 November 2026. Fixed UIDs and GIDs only matter
when files owned by an account move between machines or rebuilds (restores,
copied data, NFS). None of that applies to the reinstalled hypervisor. The
fixed IDs were carried over from the old design without need; dropping them
leaves one value less per user. `host_ansible_uid` is gone, and `host_users`
entries are `{name, groups}`.

## New machine credential for the ansible account — 2026-11-18

Decided with the user on 18 November 2026. The old credential 30
"hactar-ng ssh key" holds a key that AAP cannot export, and its public half
is unknown, so a reinstall could not install the matching key. The new
credential "hactar-ng ansible-NGNG" is built from the vault key, so CaC can
recreate it, and its public key is already in the provider key list.
Credential 30 stays for the old repositories. The bootstrap job uses the new
credential but logs in as root with the same key.

## Drop the aap.yml dispatcher — 2026-11-18

Decided with the user on 18 November 2026. In the old implementation every
job template ran `aap.yml` first. It checked the requested playbook against
an allow-list (which even named a playbook that does not exist), repeated the
same pre-checks every stage already does, and then imported the actual
playbook. 2 of 39 rated items reached Necessity 5; both are covered because
each new job template points directly at its own playbook. There is no
dispatcher in the new design.

## Free-space check only when an image is missing — 2026-11-18

Decided with the user on 18 November 2026, in
`playbooks/08-images-download.yml`, now 9 tasks. A first task reads which
images of `images_download` already exist (no checksum). The mount facts and
the 80 GiB check on `/opt` run only if at least one is missing. Before, a
rerun after a complete download failed, because the downloaded images use
the 80 GiB the check asks for.

## Old WF1 step 15 dropped — 2026-11-18

Decided with the user on 18 November 2026. Step 15 of the old host rebuild
workflow ran the old `playbooks/03-replacement.yml` on its own as the last
node. It is not carried over.

- In the old flow it switched on VM isolation: an nft rule dropped guest
  traffic leaving the guest bridges, so the VMs could not reach the internet.
- It made 12 libvirt units depend on that firewall service, so libvirt would
  not start if the firewall service failed.
- Its identity step only set facts, which vanish when the job ends.
- None of it reached Necessity 5 in the review rating; the isolation and the
  libvirt dependency are harmful in the new design.

## Image download with pinned checksums, no worker on the host — 2026-11-18

Decided with the user on 17 and 18 November 2026, built as
`playbooks/08-images-download.yml`, 8 tasks instead of 68 rated items in the
old download job (3 tasks, two included task files, two filter plugins and a
bash worker).

- The 80 GiB free-space check on `/opt` lives here, where the space is used.
  It reads only the mount facts and checks the longest mount point that holds
  `/opt`, before anything is downloaded. The threshold is
  `images_min_free_gib`, default 80.
- rclone is installed here, where the images are pulled, not in host setup.
- The Drive configuration (`vault_images_rclone_conf`) is the same Drive
  access production uses. Kept on purpose: this is a demo environment. It is
  written to `/root/.config/rclone/rclone.conf`, mode 0600, without logging.
- Each image is downloaded on its own with `rclone copyto`, `creates:` the
  destination, async with a 30 second poll (`images_download_timeout`,
  default 21600 seconds per image). No rclone module exists.
- Every image's SHA-256 is computed and compared with the pin in the CaC list
  `images_download` (`{remote, dest, sha256}`). A mismatch fails with the file
  name and the instruction to delete it by hand and rerun, instead of an
  automatic delete: `creates:` would otherwise skip the corrupt file.
- No per-file owner or mode. The old worker set `qemu:qemu 0640`; the files
  now stay root-owned 0644, readable by qemu. The image directories, their
  `virt_image_t` label and owner come from the storage stage, new files
  inherit the label, and libvirt's dynamic ownership takes over a VM's disks
  when it starts.
- The image list and its checksums are required CaC and are not in the
  private repository yet.
- Dropped: the empty-host proof, the restore-manifest resolution, the bash
  worker on the host, job-ID and marker files, both locks, the
  save-and-reconnect machinery WF1 never used, and `set_stats`, which had no
  consumer.

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
