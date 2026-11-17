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

### Status

Refactored, reviewed and committed:

| Stage | Tasks before | Tasks after |
|---|---|---|
| Provider reinstall | 24 | 13 |
| Deployment version | 34 | 5 |
| Host DNS | about 45 | 4 |
| Conversion to RHEL | about 120 | 14 |
| RHEL update | about 20 | 2 |
| Host setup (10 stages + connection probe) | about 368 | 53 + 6 handlers |

The reinstall stage ends by proving the server presents the host keys the
provider reports for the last installation. The deployment version stage reads
the plaintext counter in the private repository, publishes it as
`autodeploy_deployment_version` and writes the next number back, with the
assigned version added to the list of live instances, through the GitHub
contents API. The host DNS stage creates or corrects the name
`<label>.vXY.<zone>` at GoDaddy and waits until public DNS returns the host
address. It never deletes names: older versions may still be live instances.

The conversion stage updates Rocky Linux, runs `convert2rhel -y` with
temporary root-only credentials and proves the running kernel is Red Hat's.

The RHEL update stage updates all packages and reboots if anything changed.

The host setup (`05-host-setup.yml`) runs ten stages from `playbooks/host/`:
base (EPEL, names, timezone, admin tools, hardware monitors), virtualization,
SELinux (permissive), VM storage, libvirt networks, users with an admin login
proof, firewall, SSH hardening with a safe move to the managed port, the web
edge (nginx base, certbot, fail2ban) and tuning. `host-connect.yml` runs first
in every job that can come before the SSH stage: it uses the managed port if it
answers, otherwise port 22 as the bootstrap user. File templates live in the
public repository; their values come from the private repository's CaC and
vault. Use cases add their own nginx vhosts and fail2ban jails.

Not yet started: host setup, retiring old DNS names, image staging. Nothing here has run against a real
server yet. The old repositories remain the working implementation until a
stage has been proven here.

## Conventions

- Playbooks are `NN-subject-action.yml`, numbered in execution order.
- Variables use `snake_case` with a prefix naming their source: `robot_` for
  provider API inputs and confirmations, `provider_` for values this automation
  derives at run time, `counter_` for the deployment counter, `dns_` for the host DNS stage, `rhsm_` for subscription inputs. Registered
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
    playbooks/host/           host setup stages, imported by 05-host-setup.yml
    playbooks/host/templates/ file templates (values come from the private repo)
    playbooks/host/files/     static files (public, no secrets)
    collections/  collection requirements

## Verification

| Check | Result |
|---|---|
| `ansible-lint --profile production --strict` | pass, 0 failures, 0 warnings |
| `ansible-playbook --syntax-check` | pass |
| Syntax check inside the AAP EE (ansible-core 2.20.10, Python 3.12.15, image `ee-proj-aap-cac@sha256:e6451a10…`) | pass, no deprecation warnings |
| `reset_type: hardware` valid in community.hrobot 2.8.0 | yes (`software`, `hardware`, `power`, `manual`) |
| APME (commit `9c4372f`, core 2.20; native, opa and ansible validators) | no defects; one lead open, deviations accepted below |
| Executed against a real server | not done |
| AAP job template and end-to-end test | deployment version: job template run successful; others not done |

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
`L101`/`L033` setting `ansible_user` and `ansible_port` (the deliberate
connection switch); `L038` unresolved certified roles (APME cannot see them;
ansible-lint and the EE resolve them); `L068` `lineinfile` on `/etc/hosts` (keeps
the installer's other lines); `L071` inline content for the password file;
`L004` `rpm_key` false positive. Tested by lint, EE syntax check and a render
test of the firewall rules, sshd settings, jail file and network XML. Not
executed yet; there is no test host free (the current one runs a live AAP).

Open: `M005` flags a registered value used in a template as possibly untrusted
on ansible-core 2.19 or later. The same lead appeared for the counter file
read; the real AAP run of that stage passed, so it is closed there. The values are plain strings, so this is expected to be harmless; only a real run can confirm it.
APME's gitleaks, collection health and dependency audit validators did not run.

Review requirements come from
[Agent_generated_code_review](https://github.com/kvegh/Agent_generated_code_review).
No review report exists yet; a stage cannot be accepted before it has run
end to end through AAP.

Populated configuration and secrets belong in the private companion repository
as Ansible Vault encrypted files, never here.
