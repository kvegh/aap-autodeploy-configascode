# TODO

Open points, not yet designed or approved. Each needs a decision before work starts.

## CaC lessons learned (user, 2026-11-23)

- Bootstrap challenge of credentials: credentials needed before the CaC that
  would create them can run.
- Tokens: creation, storage and rotation of the API and Git tokens the CaC needs.
- External changes: objects changed outside the CaC (UI, API, other jobs) drift
  from the CaC.
- Add a way to delete objects (CaC today only creates and updates).
- Projects set to `state: absent` when they leave the CaC.
- Clean up the Git repositories.
- Use cases: an AAP deployment together with the use case's own project.
