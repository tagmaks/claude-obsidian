# Copilot Agent Skills in VS Code Remote WSL Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the 15 canonical Agent Skills discoverable by GitHub Copilot Agent in VS Code Remote WSL without changing the skills or vault core.

**Architecture:** Extend the existing safe multi-agent installer with an opt-in workspace-local Copilot host that links `.github/skills/<name>` to `skills/<name>`. Update the existing Copilot instructions and product documentation; validate behavior with hermetic tests and the full WSL test suite.

**Tech Stack:** POSIX Bash, Python `unittest`, Markdown, VS Code Agent Skills, Ubuntu WSL2.

**Spec:** [../specs/2026-10-12-copilot-wsl-agent-skills-design.md](../specs/2026-10-12-copilot-wsl-agent-skills-design.md)

## Global Constraints

- Keep `skills/<name>/SKILL.md` as the only canonical skill source; frontmatter remains exactly `name` and single-line `description`.
- Create workspace-local symbolic links at `.github/skills/<name>` to `<repository>/skills/<name>`.
- Target the VS Code Copilot Agent in Remote WSL only; do not claim Copilot CLI, cloud-agent, or native-Windows vault-write support.
- Require `--workspace` for Copilot; preserve the installer's existing no-host defaults and include Copilot in `--host all`.
- Never overwrite existing destinations or follow symlinked workspace parents.
- Treat `.github/skills/` as generated local integration state and ignore it in Git.
- Do not modify the Python core, vault format, transaction protocol, skill contents, or canonical skill layout.
- Run `make test` from WSL after behavioral changes.
- Git checkpoints are separate and explicit; do not commit unless the owner separately approves a checkpoint.

## Review Focus

- Missing `--workspace`: Copilot setup must return a usage error without creating paths; cover in `test_copilot_workspace_host_requires_workspace`.
- Existing skill destination: a conflicting file or link must remain unchanged; cover in `test_copilot_conflict_is_preserved`.
- Symlinked `.github` or `.github/skills` parent: applying must fail without writing outside the workspace; cover in `test_copilot_workspace_parent_symlinks_cannot_redirect_apply`.
- Dry-run and check modes: neither may create links, and check mode must distinguish missing links; cover in `test_copilot_dry_run_and_check_do_not_create_links`.
- Host expansion and source correctness: all 15 links must resolve to this checkout, `all` must include Copilot, and the no-host defaults must remain unchanged; cover in `test_copilot_host_links_canonical_skills_idempotently`, `test_all_includes_copilot`, and the existing `test_apply_is_idempotent_for_global_hosts`.

---

### Task 1: Add the safe Copilot workspace host

**Files:**
- Modify: `scripts/setup-multi-agent.sh`
- Modify: `tests/test_setup_multi_agent.py`
- Modify: `.gitignore`

**Interfaces:**
- Consumes: the existing `SKILLS_DIR`, `inspect_link(host, source, destination, confinement_root)`, and workspace-host confinement logic in `scripts/setup-multi-agent.sh`.
- Produces: accepted `--host copilot`, with links rooted at `<workspace>/.github/skills`; `--workspace` is required, and the host is included in `--host all`.

- [ ] **Step 1: Write failing Copilot host tests**

Add tests named:

- `test_copilot_workspace_host_requires_workspace`: invoke `--dry-run --host copilot` without a workspace; expect exit code `2`, an error stating that `--workspace` is required, and no created paths.
- `test_copilot_dry_run_and_check_do_not_create_links`: in a temporary workspace, expect dry-run to report 15 planned destinations with no `.github/skills` directory; expect check mode to return `1` for missing links and also create nothing.
- `test_copilot_host_links_canonical_skills_idempotently`: apply into a temporary workspace; expect 15 discovered `SKILL.md` files under `.github/skills`, each parent to be a symlink resolving to `ROOT / "skills" / <name>`; apply again and run check mode, expecting exit code `0` and 15 `READY` results from each.
- `test_copilot_conflict_is_preserved`: pre-create a conflicting destination with marker content; expect exit code `2` and unchanged marker content.
- `test_copilot_workspace_parent_symlinks_cannot_redirect_apply`: exercise symlinked `.github` and `.github/skills` parents; expect preview/apply to return `2` and the outside directory to remain empty.
- `test_all_includes_copilot`: invoke `--host all --workspace <temp-workspace> --apply`; expect the Copilot skill links under `.github/skills`.

Keep the existing default-host test passing to prove that invoking the installer without `--host` does not add Copilot to the default destinations.

- [ ] **Step 2: Run the focused test file and verify the new tests fail**

Run from WSL:

```bash
python3 -m unittest discover -s tests -p test_setup_multi_agent.py -v
```

Expected: the existing tests pass, and new Copilot cases fail because the host is not yet supported.

- [ ] **Step 3: Implement the Copilot host in `scripts/setup-multi-agent.sh`**

Add `copilot` to the accepted host list and help text; add it to the `all` expansion; route it to `$WORKSPACE/.github/skills`; and include it in workspace-required and confinement handling. Preserve existing dry-run, check, conflict, idempotency, and no-host default behavior.

- [ ] **Step 4: Ignore generated workspace links**

Add `.github/skills/` to the existing `.gitignore` section for workspace-local Agent Skills links. Confirm it is effective with `git check-ignore .github/skills/wiki`.

- [ ] **Step 5: Run the focused tests and shell syntax check**

Run:

```bash
python3 -m unittest discover -s tests -p test_setup_multi_agent.py -v
bash -n scripts/setup-multi-agent.sh
git check-ignore .github/skills/wiki
```

Expected: all setup tests pass, Bash reports no syntax errors, and Git reports the generated skill-link path as ignored.

### Task 2: Document Copilot and Remote WSL usage

**Files:**
- Modify: `.github/copilot-instructions.md`
- Modify: `README.md`
- Modify: `docs/install-guide.md`
- Modify: `docs/windows-wsl.md`

**Interfaces:**
- Consumes: the Copilot host and command behavior implemented in Task 1.
- Produces: a consistent documented workflow for previewing, applying, and checking Copilot skill links from VS Code Remote WSL.

- [ ] **Step 1: Add concise Copilot-specific workspace guidance**

In `.github/copilot-instructions.md`, tell Copilot to use the task-relevant discovered Agent Skill and run repository commands in Remote WSL while continuing to follow `AGENTS.md`. Do not claim that discovery guarantees automatic invocation.

- [ ] **Step 2: Document the commands and link layout**

Update `README.md` and `docs/install-guide.md` with the opt-in Copilot host and:

```bash
bash scripts/setup-multi-agent.sh --host copilot --workspace "$PWD"
bash scripts/setup-multi-agent.sh --host copilot --workspace "$PWD" --apply
bash scripts/setup-multi-agent.sh --host copilot --workspace "$PWD" --check
```

Document `.github/skills/<name> -> <repository>/skills/<name>`, the dry-run-first behavior, and that pulling upstream changes updates the linked canonical skills without copying.

- [ ] **Step 3: Document the Remote WSL boundary**

Add a focused section to `docs/windows-wsl.md`: open the checkout from Ubuntu with VS Code Remote WSL, run setup and product commands from WSL, keep vault operations on the existing transaction workflow, and do not describe native Windows writes as supported.

- [ ] **Step 4: Review documentation consistency**

Check that the README, installation guide, Windows/WSL guide, Copilot instructions, and setup-script usage text agree on the host name, required workspace, command flags, link destination, and discovery limitation. Do not modify the local, untracked `wsl2.md`.

### Task 3: Verify package and runtime discovery

**Files:**
- No additional product files; validation only.

**Interfaces:**
- Consumes: the installer, tests, and documentation from Tasks 1 and 2.
- Produces: evidence that repository contracts pass and a clear Copilot discovery result in the target WSL environment.

- [ ] **Step 1: Run the full repository suite from WSL**

Run:

```bash
make test
```

Expected: exit code `0`; all Python and shell suites, product/capability/package/hook/manifest contracts pass.

- [ ] **Step 2: Preview setup in the target WSL checkout**

Run:

```bash
bash scripts/setup-multi-agent.sh --host copilot --workspace "$PWD"
```

Expected: 15 `PLANNED` destinations under `.github/skills`; no link is created. Review the output before applying.

- [ ] **Step 3: Apply only with explicit owner approval**

Do not apply links to the current checkout unless the owner explicitly approves that workspace mutation. If approved, run the corresponding `--apply` command, verify all 15 links resolve to the canonical `skills/<name>` directories, then run `--check` and expect exit code `0` with 15 `READY` results.

- [ ] **Step 4: Verify discovery in VS Code**

In the Remote WSL session, inspect Agent Customizations and confirm the 15 skills appear. If VS Code does not discover the symlinked skills, stop and report the result; do not copy skill files or silently change the design.
