# Harbor Task Dev Container Template

A minimal Dev Container Template for developing [Harbor](https://github.com/laude-institute/harbor) tasks locally with VS Code.

The template is published to GitHub Container Registry (GHCR) and can be applied through:

> Dev Containers: Add Dev Container Configuration Files...

→ **Add configuration to user data folder**

→ **enter a custom template id**

Current Template ID:

```text
ghcr.io/krave1986/harbor-task-devcontainer-template/harbor-task:1.0.1
```

## USAGE

### Using this template

This repository publishes the Harbor Task Dev Container Template to GitHub Container Registry (GHCR).

The published template package is:

```text
ghcr.io/krave1986/harbor-task-devcontainer-template/harbor-task
```

Current version:

```text
ghcr.io/krave1986/harbor-task-devcontainer-template/harbor-task:1.0.1
```

GHCR package page:

### How to determine a template's GHCR address

For a Dev Container Template, the GHCR reference can normally be determined from the template metadata and repository:

```text
ghcr.io/<publisher>/<repository>/<template-id>:<version>
```

Where:

* `<publisher>` comes from `publisher` in `devcontainer-template.json`
* `<repository>` is the GitHub repository name
* `<template-id>` comes from `id` in `devcontainer-template.json`
* `<version>` comes from `version` in `devcontainer-template.json`

For example, this repository contains:

```json
{
  "id": "harbor-task",
  "version": "1.0.1",
  "publisher": "krave1986"
}
```

and the repository is:

```text
krave1986/harbor-task-devcontainer-template
```

Therefore its published GHCR reference is:

```text
ghcr.io/krave1986/harbor-task-devcontainer-template/harbor-task:1.0.1
```

The GHCR reference is an OCI container registry reference, not a web URL. To browse the published package, use the GitHub Packages page above.

In normal Dev Container usage, users generally do not need to manually construct or pull this GHCR reference. Dev Container Templates are normally discovered and consumed through the Dev Container Templates registry.


---

## Why this repository exists

Harbor tasks already define their execution environment through:

```text
task/
└── environment/
    └── Dockerfile
```

That Dockerfile is part of the task itself and is the environment definition that Harbor will eventually use.

However, developing the task locally with VS Code is much more convenient with Dev Containers.

The problem is that we do **not** want to add VS Code-specific configuration to every Harbor task repository.

We also want the local development container to use the **exact same Dockerfile that Harbor will use**.

Therefore this repository provides a reusable Dev Container Template.

---

# Design

The intended Harbor task structure is:

```text
task/
├── environment/
│   └── Dockerfile
├── tests/
├── solution/
├── instruction.md
└── task.toml
```

The Dev Container Template adds only local development configuration:

```text
task/
├── environment/
│   └── Dockerfile
├── tests/
├── solution/
├── instruction.md
├── task.toml
└── .devcontainer/
    └── devcontainer.json
```

The `.devcontainer` directory does **not** need to be committed to the Harbor task repository.

Instead, VS Code can store it in its user data directory.

This keeps Harbor task repositories clean.

---

# Why not modify the Harbor task?

## Answer

Because the Dev Container configuration is a development convenience, not part of the Harbor task itself.

Harbor already has its own environment definition:

```text
environment/Dockerfile
```

Adding:

```text
.devcontainer/devcontainer.json
```

to every task creates unnecessary repository changes.

It also creates a maintenance problem:

* every task needs the same configuration;
* configuration changes have to be copied to every task;
* task repositories contain files that Harbor itself does not need.

A reusable Template solves this once.

---

# Why use "Add configuration to user data folder"?

## Answer

VS Code provides this option specifically so that Dev Container configuration does not have to be written into the workspace.

The important consequence is:

```text
Harbor task repository
        │
        ├── environment/
        ├── tests/
        ├── solution/
        └── ...
        
VS Code user data
        │
        └── .devcontainer/
```

The Harbor repository therefore remains unchanged.

This is particularly useful for benchmark/task repositories where the repository contents themselves are part of the evaluation.

---

# Why use the task's Dockerfile?

## Answer

The Harbor task's:

```text
environment/Dockerfile
```

is the authoritative environment definition.

We do not want to create another Dockerfile for local development.

Otherwise we could accidentally develop against:

```text
Local Dockerfile
```

while Harbor evaluates against:

```text
Task Dockerfile
```

That can produce a subtle "works locally, fails in Harbor" problem.

The Template therefore points directly at:

```text
${localWorkspaceFolder}/environment/Dockerfile
```

---

# Why is `${localWorkspaceFolder}` important?

## Answer

This was one of the main problems encountered while developing this Template.

The Template itself lives in a repository such as:

```text
harbor-task-devcontainer-template/
└── src/
    └── harbor-task/
```

But when the Template is applied, VS Code does **not** execute it from that repository.

Instead, the generated Dev Container configuration belongs to the target workspace.

For example:

```text
C:\...\exam_001\
├── environment\
│   └── Dockerfile
├── solution\
└── ...
```

Therefore paths in the resulting `devcontainer.json` must be resolved relative to the **target workspace**, not relative to the Template repository.

The working configuration is:

```json
{
  "name": "Harbor Task",
  "build": {
    "dockerfile": "${localWorkspaceFolder}/environment/Dockerfile",
    "context": "${localWorkspaceFolder}"
  },
  "workspaceMount": "source=${localWorkspaceFolder}/solution,target=/app,type=bind",
  "workspaceFolder": "/app"
}
```

---

# Why did `../environment/Dockerfile` fail?

## Answer

Our first version used:

```json
"dockerfile": "../environment/Dockerfile"
```

This looks correct if you imagine the final workspace as:

```text
task/
├── environment/
└── .devcontainer/
    └── devcontainer.json
```

because:

```text
.devcontainer/../environment/Dockerfile
```

does point to the Dockerfile.

However, we selected:

> Add configuration to user data folder

When VS Code applied the Template, the generated configuration was stored under its user data area.

The error showed:

```text
c:\Users\krave\AppData\Roaming\Code\User\globalStorage\
ms-vscode-remote.remote-containers\configs\exam_001\
environment\Dockerfile
```

did not exist.

The relative path was therefore being resolved from the generated configuration location, not from the Harbor task's `.devcontainer` directory.

Using:

```json
"${localWorkspaceFolder}/environment/Dockerfile"
```

solves this because the path explicitly starts from the actual target workspace.

---

# Why mount only `solution/`?

## Answer

The Dockerfile contains:

```dockerfile
WORKDIR /app
```

The actual source we are editing is:

```text
task/solution/
```

Therefore:

```json
"workspaceMount": "source=${localWorkspaceFolder}/solution,target=/app,type=bind"
```

makes:

```text
Windows:
task/solution/

        ↓ bind mount

Container:
/app/
```

This is exactly what we want.

The rest of the Harbor task remains available through the container filesystem because it was already included by the Docker build context where appropriate, while the editable solution is directly mounted from the host.

---

# Why not mount the entire task directory?

## Answer

Because `/app` is the working directory defined by the task Dockerfile.

Mounting the entire task directory onto `/app` would turn:

```text
/app
```

into a host bind mount and could obscure files that were created by the Dockerfile.

More importantly, the actual artifact we are developing is:

```text
solution/
```

So mounting only `solution/` keeps the container environment controlled by the task's Dockerfile while making the solution directly editable from Windows.

---

# Why not use Docker Compose?

## Answer

There is only one container.

The Harbor task already provides the Dockerfile.

Adding Compose would introduce another layer of configuration without solving a real problem.

The intended configuration is deliberately minimal:

```text
Dockerfile
    ↓
one container
    ↓
/app
    ↑
solution/ bind mount
```

---

# Why not use Dev Container Features?

## Answer

Because the task environment is supposed to come from:

```text
environment/Dockerfile
```

Adding Features would install additional software that is not necessarily present in Harbor's evaluation environment.

That creates another possible source of:

> Works in VS Code, fails in Harbor.

Therefore this Template intentionally adds:

* no Features;
* no additional packages;
* no Compose;
* no startup script;
* no extra development dependencies.

The Dockerfile remains authoritative.

---

# Why is the Template stored under `src/`?

## Answer

This follows the official Dev Container Template repository structure.

The repository uses:

```text
src/
└── harbor-task/
    ├── devcontainer-template.json
    └── .devcontainer/
        └── devcontainer.json
```

`harbor-task` is the Template ID.

The GitHub Action publishes Templates found below `src/` to GHCR.

---

# Why use GHCR?

## Answer

Dev Container Templates are distributed as OCI artifacts.

GitHub Container Registry provides a natural distribution mechanism for them.

The published Template is:

```text
ghcr.io/krave1986/harbor-task-devcontainer-template/harbor-task:1.0.1
```

This means the Template can be consumed directly by VS Code without cloning this repository.

---

# Why does GHCR show two packages?

## Answer

After the first publication, GitHub showed:

```text
harbor-task-devcontainer-template
```

and:

```text
harbor-task-devcontainer-template/harbor-task
```

The latter is the actual Template package:

```text
ghcr.io/krave1986/harbor-task-devcontainer-template/harbor-task
```

The Template name is the final `harbor-task` component.

Therefore this is the ID to use in VS Code:

```text
ghcr.io/krave1986/harbor-task-devcontainer-template/harbor-task:1.0.1
```

---

# Why use a version instead of `latest`?

## Answer

Dev Container Templates have their own version in:

```text
devcontainer-template.json
```

For example:

```json
"version": "1.0.1"
```

Using an explicit version makes the development environment reproducible.

A Harbor task that was tested with:

```text
1.0.1
```

does not unexpectedly change because a newer Template was published later.

When the Template changes, increment the version:

```text
1.0.1
→
1.0.2
```

and publish again.

---

# Why was `1.0.0` changed to `1.0.1`?

## Answer

The first published version contained:

```json
"dockerfile": "../environment/Dockerfile"
```

That path failed when the Template was applied through VS Code's user-data configuration.

After changing it to:

```json
"dockerfile": "${localWorkspaceFolder}/environment/Dockerfile"
```

the Template itself changed.

Therefore a new Template version was required.

The successful version is:

```text
1.0.1
```

---

# Why is the release workflow so small?

## Answer

The original Dev Container Template starter workflow also supports automatic documentation generation and creation of documentation PRs.

Those features are unnecessary here.

We only need:

```text
checkout
    ↓
devcontainers/action
    ↓
publish Template
    ↓
GHCR
```

The current workflow is therefore intentionally minimal:

```yaml
name: Publish Dev Container Template

on:
  workflow_dispatch:

jobs:
  publish:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    permissions:
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Publish Template
        uses: devcontainers/action@v1
        with:
          publish-templates: "true"
          base-path-to-templates: "./src"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

# Why is publishing manual?

## Answer

The workflow uses:

```yaml
on:
  workflow_dispatch:
```

instead of automatically publishing on every push.

This prevents an ordinary repository change from automatically creating a new published Template.

The current process is:

```text
change Template
    ↓
increment version
    ↓
commit + push
    ↓
GitHub Actions
    ↓
Run workflow
    ↓
GHCR
```

This is safer for a development utility that is used by other repositories.

---

# Why did the first GitHub Action run show warnings?

## Answer

The first successful run reported:

```text
Node.js 20 is deprecated
```

and:

```text
ubuntu-latest will migrate to Ubuntu 26
```

These were GitHub Actions runner warnings/notices.

They did not prevent publication.

The Action successfully published the Template to GHCR.

It also reported that it could not automatically create a repository tag:

```text
Failed to automatically add repo tag
```

This did not prevent the Template from being published.

The actual GHCR package was visible after the successful run, confirming publication.

---

# Current repository structure

The intended repository structure is:

```text
harbor-task-devcontainer-template/
├── .github/
│   └── workflows/
│       └── release.yaml
│
└── src/
    └── harbor-task/
        ├── devcontainer-template.json
        └── .devcontainer/
            └── devcontainer.json
```

---

# Current Template configuration

## `src/harbor-task/devcontainer-template.json`

```json
{
  "id": "harbor-task",
  "version": "1.0.1",
  "name": "Harbor Task",
  "description": "Minimal Dev Container for Harbor tasks using the task environment Dockerfile.",
  "documentationURL": "https://github.com/krave1986/harbor-task-devcontainer-template",
  "publisher": "krave1986",
  "licenseURL": "https://github.com/krave1986/harbor-task-devcontainer-template/blob/main/LICENSE",
  "platforms": [
    "Linux"
  ],
  "keywords": [
    "harbor",
    "devcontainer",
    "task"
  ]
}
```

## `src/harbor-task/.devcontainer/devcontainer.json`

```json
{
  "name": "Harbor Task",
  "build": {
    "dockerfile": "${localWorkspaceFolder}/environment/Dockerfile",
    "context": "${localWorkspaceFolder}"
  },
  "workspaceMount": "source=${localWorkspaceFolder}/solution,target=/app,type=bind",
  "workspaceFolder": "/app"
}
```

---

# How to use it

For a Harbor task such as:

```text
task/
├── environment/
│   └── Dockerfile
├── tests/
├── solution/
├── instruction.md
└── task.toml
```

open the `task` directory in VS Code.

Then:

1. Press `Ctrl+Shift+P`.

2. Run:

   ```text
   Dev Containers: Add Dev Container Configuration Files...
   ```

3. Select:

   ```text
   Add configuration to user data folder
   ```

4. Select:

   ```text
   enter a custom template id
   ```

5. Enter:

   ```text
   ghcr.io/krave1986/harbor-task-devcontainer-template/harbor-task:1.0.1
   ```

6. Apply the Template.

7. Reopen the project in the container.

The container will use:

```text
task/environment/Dockerfile
```

and mount:

```text
task/solution/
```

to:

```text
/app
```

---

# The core principle

The entire Template exists to enforce one simple rule:

> **Use the Harbor task's own Dockerfile as the local development environment, while keeping VS Code-specific configuration outside the Harbor task repository.**

That gives us:

```text
Harbor
  ↓
environment/Dockerfile
  ↓
authoritative environment

VS Code
  ↓
GHCR Template
  ↓
uses the same Dockerfile
  ↓
solution/ → /app
```

The task repository remains clean, the development environment is reusable, and the local container stays aligned with the environment Harbor will evaluate.
