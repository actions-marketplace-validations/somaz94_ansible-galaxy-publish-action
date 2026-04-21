# ansible-galaxy-publish-action

[![CI](https://github.com/somaz94/ansible-galaxy-publish-action/actions/workflows/ci.yml/badge.svg)](https://github.com/somaz94/ansible-galaxy-publish-action/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Latest Tag](https://img.shields.io/github/v/tag/somaz94/ansible-galaxy-publish-action)](https://github.com/somaz94/ansible-galaxy-publish-action/tags)
[![Top Language](https://img.shields.io/github/languages/top/somaz94/ansible-galaxy-publish-action)](https://github.com/somaz94/ansible-galaxy-publish-action)
[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Ansible%20Galaxy%20Publish%20Action-blue?logo=github)](https://github.com/marketplace/actions/ansible-galaxy-publish-action)

A composite GitHub Action that publishes an [Ansible](https://www.ansible.com/) **collection** or **role** to [Ansible Galaxy](https://galaxy.ansible.com/). Installs Python and Ansible, then either runs `ansible-galaxy collection build && publish` (collection mode) or `ansible-galaxy role import` (role mode). Supports `dry_run` for CI validation without a live API key.

<br/>

## Features

- One action for both publish paths: **collection** (`build` + `publish`) and **role** (`import`)
- `dry_run: true` validates inputs and — for collections — still builds the tarball, without hitting Galaxy (ideal for pull-request CI)
- Automatically locates the built tarball by `<namespace>-<name>-*.tar.gz` glob and picks the highest semver match
- Version pin for Ansible (`ansible_version`); empty = latest
- Writes a per-run result to `$GITHUB_STEP_SUMMARY`
- Exposes `published_ref` (e.g., `collection/somaz94.ansible_k8s_iac_tool@1.2.0`) and `artifact_path` outputs

<br/>

## Requirements

- **Runner OS**: `ubuntu-latest` (also works on other OSes, but the rest of the Galaxy tooling is most commonly tested there).
- **Caller must run `actions/checkout`** before this action.
- **Python 3.10+** is recommended (default `3.12`).
- **Secret**: `GALAXY_API_KEY` (or any secret name you pass to `api_key`). Required unless `dry_run: true`.

<br/>

## Quick Start

### Publish an Ansible collection on tag push

```yaml
name: Publish to Ansible Galaxy
on:
  push:
    tags: ["v*"]
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: somaz94/ansible-galaxy-publish-action@v1
        with:
          type: collection
          api_key: ${{ secrets.GALAXY_API_KEY }}
          collection_namespace: somaz94
          collection_name: ansible_k8s_iac_tool
```

<br/>

### Import an Ansible role on tag push

```yaml
name: Publish to Ansible Galaxy
on:
  push:
    tags: ["v*"]
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: somaz94/ansible-galaxy-publish-action@v1
        with:
          type: role
          api_key: ${{ secrets.GALAXY_API_KEY }}
          namespace: somaz94
          role_name: ansible_kubectl_krew
```

<br/>

## Usage

### Dry-run in PR CI (no API key needed)

```yaml
- uses: actions/checkout@v6
- uses: somaz94/ansible-galaxy-publish-action@v1
  with:
    type: collection
    dry_run: true
    collection_namespace: somaz94
    collection_name: ansible_k8s_iac_tool
```

The build step still runs (so broken `galaxy.yml` / missing files fail CI), but no publish call is made. The `published_ref` output is prefixed with `dry-run:` so downstream steps can branch on it.

<br/>

### Pin the Ansible version

```yaml
- uses: somaz94/ansible-galaxy-publish-action@v1
  with:
    type: collection
    api_key: ${{ secrets.GALAXY_API_KEY }}
    ansible_version: '9.5.1'
    collection_namespace: somaz94
    collection_name: ansible_k8s_iac_tool
```

<br/>

### Publish a collection that lives in a subdirectory

```yaml
- uses: somaz94/ansible-galaxy-publish-action@v1
  with:
    type: collection
    api_key: ${{ secrets.GALAXY_API_KEY }}
    working_directory: collections/somaz94/my_collection
    collection_namespace: somaz94
    collection_name: my_collection
```

<br/>

### Consume the outputs in a follow-up step

```yaml
- id: galaxy
  uses: somaz94/ansible-galaxy-publish-action@v1
  with:
    type: collection
    api_key: ${{ secrets.GALAXY_API_KEY }}
    collection_namespace: somaz94
    collection_name: ansible_k8s_iac_tool

- name: Report
  if: always()
  run: |
    echo "Published: ${{ steps.galaxy.outputs.published_ref }}"
    echo "Artifact:  ${{ steps.galaxy.outputs.artifact_path }}"
```

<br/>

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `type` | Publish target type: `collection` or `role`. | Yes | — |
| `api_key` | Ansible Galaxy API key. Required unless `dry_run: true`. | Conditional | `''` |
| `namespace` | Galaxy namespace. Required for `role` (with `role_name`). In `collection` mode, used as a fallback when `collection_namespace` is empty. | Conditional | `''` |
| `role_name` | Role name under `namespace` (role mode, e.g., `ansible_kubectl_krew`). | Conditional (role mode) | `''` |
| `collection_namespace` | Namespace used to locate the built tarball (collection mode). Falls back to `namespace`. | Conditional (collection mode) | `''` |
| `collection_name` | Collection name used to locate the built tarball (e.g., `ansible_k8s_iac_tool`). | Conditional (collection mode) | `''` |
| `working_directory` | Directory containing `galaxy.yml` (collection) or `meta/main.yml` (role). | No | `.` |
| `python_version` | Python version for `actions/setup-python`. | No | `3.12` |
| `ansible_version` | pip pin for Ansible (e.g., `9.5.1`). Empty = latest. | No | `''` |
| `dry_run` | When `true`, build the collection (if applicable) but skip `publish`/`import`. | No | `false` |

<br/>

## Outputs

| Output | Description |
|--------|-------------|
| `published_ref` | Published reference, e.g., `collection/somaz94.ansible_k8s_iac_tool@1.2.0` or `role/somaz94.ansible_kubectl_krew`. Prefixed with `dry-run:` when `dry_run` is true. |
| `artifact_path` | Absolute path to the built collection tarball (collection mode only; empty for role mode). |

<br/>

## Permissions

The action itself needs no special permissions beyond what `actions/checkout` and `actions/setup-python` require. A typical caller:

```yaml
permissions:
  contents: read
```

The Galaxy API key is supplied via the `api_key` input (typically `${{ secrets.GALAXY_API_KEY }}`).

<br/>

## How It Works

1. **Validate inputs** — `type` must be `collection` or `role`; required fields per mode are enforced; `api_key` is required unless `dry_run: true`.
2. **`actions/setup-python`** — installs the requested Python version.
3. **pip install** — installs `ansible` (or `ansible==<version>` if `ansible_version` is set).
4. **Collection mode**:
   - `ansible-galaxy collection build --force` in `working_directory`
   - Locate the tarball `<namespace>-<name>-*.tar.gz` (highest semver wins) and expose it via `artifact_path`
   - When `dry_run` is `false`, run `ansible-galaxy collection publish <tarball> --api-key=<key>`
5. **Role mode**:
   - When `dry_run` is `false`, run `ansible-galaxy role import --api-key <key> <namespace> <role_name>`
6. **Summary & outputs** — write the published reference to `$GITHUB_STEP_SUMMARY` and `published_ref`. Dry-run results are prefixed with `dry-run:`.

<br/>

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
