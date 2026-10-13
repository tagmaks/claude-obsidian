# Copilot Agent Skills in VS Code Remote WSL

**Status:** Draft for user review  
**Date:** 2026-10-12

## Goal

Enable GitHub Copilot Agent in VS Code Remote WSL to discover and use this
repository's canonical Agent Skills. Keep the fork easy to update from upstream:
do not copy or restructure skills, change their behavior, or modify the Python
vault core.

The product repository remains separate from the user's vault. Vault writes
continue to use the existing inspected transaction workflow in WSL.

## Decision

Expose each canonical skill through a workspace-local symbolic link:

```text
.github/skills/<name> -> <repository>/skills/<name>
```

VS Code documents `.github/skills/` as a project Agent Skills location.
Workspace-local links keep Copilot's discovery scoped to this checkout and make
upstream skill changes visible without a second copy or synchronization step.

The existing `scripts/setup-multi-agent.sh` will add an opt-in `copilot` host.
It will require an explicit workspace and preserve the installer's existing
dry-run, apply, check, idempotency, conflict, and path-confinement behavior.
Copilot will be included in `--host all`; invoking the installer without
`--host` will retain its existing default hosts.

The setup commands will be:

```bash
bash scripts/setup-multi-agent.sh --host copilot --workspace "$PWD"
bash scripts/setup-multi-agent.sh --host copilot --workspace "$PWD" --apply
bash scripts/setup-multi-agent.sh --host copilot --workspace "$PWD" --check
```

The first command is a no-write preview. Users apply only after reviewing its
planned destinations. Generated `.github/skills/` links will be ignored by Git
and treated as local integration state.

## Scope

### Installer and local state

- Add `copilot` to the accepted host values and route it to
  `<workspace>/.github/skills`.
- Require `--workspace`, just as for other workspace-local hosts.
- Include Copilot in `--host all`; keep the no-host default unchanged.
- Create one link per canonical skill directory that contains `SKILL.md`.
- Do not overwrite existing files or links. Reject symlinked parents that
  could redirect a destination outside the workspace.
- Ignore the generated `.github/skills/` directory in the repository's
  `.gitignore`.

### Copilot instructions and documentation

- Extend `.github/copilot-instructions.md` with concise guidance to use the
  relevant discovered Agent Skill and run repository commands in Remote WSL.
- Update `README.md`, `docs/install-guide.md`, and `docs/windows-wsl.md` with
  the preview/apply/check commands and the supported VS Code Remote WSL
  workflow.
- Explain that discovery makes skills available to Copilot but does not
  guarantee automatic invocation.
- Do not add another command wrapper or duplicate any `SKILL.md` content.

### Tests

Extend `tests/test_setup_multi_agent.py` to verify that:

- Copilot requires an explicit workspace.
- Dry-run plans the expected links without creating them.
- Apply creates links for all canonical skills and each resolves to this
  checkout's `skills/<name>` directory.
- Reapplying is idempotent and check mode reports missing links.
- Existing destinations are preserved and reported as conflicts.
- Symlinked workspace parents cannot redirect link creation outside the
  workspace.
- `--host all` includes Copilot while the no-host default remains unchanged.

## User workflow and data flow

1. Open the product checkout from Ubuntu using VS Code Remote WSL, for example
   by running `code .` from the checkout in the WSL terminal.
2. Preview the Copilot setup command and inspect the planned destinations.
3. Apply the links from WSL, then verify the skills appear in VS Code's Agent
   Customizations.
4. Copilot loads an applicable skill according to its description. Users
   continue to follow `AGENTS.md`, the selected skill, and the existing
   transaction protocol for vault operations.
5. Run the product CLI and tests from WSL. Native Windows remains a
   read-only/dry-run platform for vault operations.
6. Pulling upstream changes updates the files behind the links; no skill
   copying or second synchronization process is needed.

This integration targets the VS Code Copilot Agent in Remote WSL. It does not
claim support for Copilot CLI, cloud agents, or native-Windows vault writes.

## Errors and safety

- Missing `--workspace` is an explicit usage error.
- An existing destination that is not the expected link is a conflict; it is
  never replaced automatically.
- Symlinked workspace parents are rejected to preserve workspace confinement.
- Dry-run and check modes do not create links. Apply creates only missing
  links.
- The setup script changes integration links only. It does not select, inspect,
  or mutate a user vault.
- The `.github/skills/` directory is reserved for generated links in this
  integration; canonical skill content remains under `skills/`.

## Verification and acceptance

Run the complete repository suite from WSL:

```bash
make test
```

Also verify the integration in the target VS Code environment: after setup,
Agent Customizations must show the 15 canonical skills through
`.github/skills/`. This runtime check is necessary because VS Code documents
the skill location but does not specify symbolic-link traversal. If the
checkout filesystem or VS Code does not discover the links, stop and revise
the design; do not silently fall back to duplicated skill files.

The fork adapter changes only the setup script, local ignore rules, Copilot
instructions, documentation, and related tests. It does not alter the
transaction engine, vault format, skill instructions, or upstream skill
layout.
