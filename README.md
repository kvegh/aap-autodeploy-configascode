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
| Provider reinstall | 24 | 9 |

Not yet started: host identity, deployment version, conversion to RHEL,
updates, host setup, DNS, image staging. Nothing here has run against a real
server yet. The old repositories remain the working implementation until a
stage has been proven here.

## Conventions

- Playbooks are `NN-subject-action.yml`, numbered in execution order.
- Variables use `snake_case` with a prefix naming their source: `robot_` for
  provider API inputs and confirmations, `provider_` for values this automation
  derives at run time.
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

    playbooks/    one playbook per stage
    collections/  collection requirements

## Verification

| Check | Result |
|---|---|
| `ansible-lint --profile production --strict` | pass, 0 failures, 0 warnings |
| `ansible-playbook --syntax-check` | pass |
| Checked against the AAP execution environment's ansible-core | not done |
| Executed against a real server | not done |
| AAP job template and end-to-end test | not done |

Review requirements come from
[Agent_generated_code_review](https://github.com/kvegh/Agent_generated_code_review).
No review report exists yet; a stage cannot be accepted before it has run
end to end through AAP.

Populated configuration and secrets belong in the private companion repository
as Ansible Vault encrypted files, never here.
