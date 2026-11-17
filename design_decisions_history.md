# Design decisions history

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

Not yet validated against a real server.
