# CLAUDE.md

<br/>

## Project Structure

- Composite GitHub Action (no Docker image — `runs.using: composite`)
- One action covers both Ansible Galaxy publish paths:
  - **collection** → `ansible-galaxy collection build && publish`
  - **role**       → `ansible-galaxy role import`
- `dry_run: true` validates inputs + (for collections) still builds the tarball, but skips the Galaxy call — used by `ci.yml` and `use-action.yml` so neither needs a live API key.

<br/>

## Key Files

- `action.yml` — composite action (10 inputs, 2 outputs). Branch logic lives in a single inline `shell: bash` step ("Publish to Galaxy") that handles both modes and dry-run.
- `tests/fixtures/sample_collection/` — minimal buildable collection (`galaxy.yml` + one role). CI dry-runs the collection path against it.
- `tests/fixtures/sample_role/` — minimal role (`meta/main.yml` + `tasks/main.yml`). CI dry-runs the role path against it.
- `cliff.toml` — git-cliff config for release notes.
- `Makefile` — `lint` (dockerized yamllint), `test` (`test-collection` + `test-role` locally), `fixtures`, `clean`.

<br/>

## Build & Test

There is no local "build" — composite actions execute on the GitHub Actions runner.

```bash
make lint              # yamllint action.yml + workflows + fixtures
make test              # local dry-run: build fixture collection + list fixture role
make test-collection   # only build the fixture collection
make test-role         # only sanity-check the fixture role metadata
make clean             # remove built collection tarballs
```

Local `make test` requires `ansible` on PATH (the CI-side install is handled by the action's `setup-python` + `pip install ansible` steps).

<br/>

## Workflows

- `ci.yml` — `lint` (yamllint + actionlint) + `test-collection` (dry-run against `tests/fixtures/sample_collection`) + `test-role` (dry-run against `tests/fixtures/sample_role`) + `ci-result` aggregator.
- `release.yml` — git-cliff release notes + `softprops/action-gh-release` + `somaz94/major-tag-action` for `v1` sliding tag.
- `use-action.yml` — post-release smoke test: `uses: somaz94/ansible-galaxy-publish-action@v1` against the same fixtures in both modes (dry-run).
- `gitlab-mirror.yml`, `changelog-generator.yml`, `contributors.yml`, `dependabot-auto-merge.yml`, `issue-greeting.yml`, `stale-issues.yml` — standard repo automation.

<br/>

## Release

Push a `vX.Y.Z` tag → `release.yml` runs → GitHub Release published → `v1` major tag updated → `use-action.yml` smoke-tests the published version against the fixture collection and role.

<br/>

## Action Inputs

Required: `type` (`collection` or `role`).

Conditional: `api_key` (required unless `dry_run: true`), `namespace` / `role_name` (role mode), `collection_namespace` / `collection_name` (collection mode).

Tuning: `working_directory` (default `.`), `python_version` (default `3.12`), `ansible_version`, `dry_run` (default `false`).

See [README.md](README.md) for the full table.

<br/>

## Internal Flow

1. **Validate inputs** — `type` in {`collection`, `role`}; required fields per mode; `api_key` required unless `dry_run=true`; `working_directory` exists.
2. **`actions/setup-python`** — installs the requested Python version.
3. **`pip install ansible`** — optional version pin via `ansible_version`.
4. **Collection mode**:
   - `ansible-galaxy collection build --force` in `working_directory`
   - Locate `<namespace>-<name>-*.tar.gz` (highest semver wins) → expose via `artifact_path`; derive `version` from tarball name → build `published_ref = collection/<ns>.<name>@<version>`
   - When `dry_run=false`: `ansible-galaxy collection publish <tarball> --api-key=<key>`
5. **Role mode**:
   - `published_ref = role/<namespace>.<role_name>`; `artifact_path=""`
   - When `dry_run=false`: `ansible-galaxy role import --api-key <key> <namespace> <role_name>`
6. **Summary & outputs** — `published_ref` and (collection only) `artifact_path` written to `$GITHUB_OUTPUT`; dry-run prefixes the ref with `dry-run:`. Step summary appended to `$GITHUB_STEP_SUMMARY`.
