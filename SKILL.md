---
name: fabric-cli
description: >
  Use this skill whenever the user wants to interact with Microsoft Fabric using the `fab` CLI tool.
  Triggers include: installing or configuring the Fabric CLI, authenticating to a Fabric tenant,
  navigating workspaces or items via the command line, exporting/importing Fabric artifacts,
  deploying items between environments, scripting or automating Fabric operations, or integrating
  the CLI into a CI/CD pipeline (Azure DevOps, GitHub Actions). Use this skill even if the user
  just asks "how do I use fab", "how do I deploy to Fabric from a pipeline", or mentions fabric-cli
  in passing.
---

# Microsoft Fabric CLI (`fab`) Skill

The `fab` CLI is a cross-platform command-line tool for navigating, scripting, and automating
Microsoft Fabric environments. It exposes a file-system-style interface over workspaces and items.

## Installation

```bash
pip install ms-fabric-cli           # install
pip install --upgrade ms-fabric-cli # upgrade
```

Requires Python ≥ 3.12. Works on Windows, macOS, and Linux.

---

## Modes

| Mode | How to activate | Best for |
|------|----------------|----------|
| **Interactive** | `fab config set mode interactive` then `fab auth login` | Exploration, ad-hoc work |
| **Command-line** | Just run `fab <command> <args>` | Scripting, CI/CD |

In interactive mode the prompt shows your current path (e.g., `fab:/ws1.Workspace$`).

---

## Authentication

### Interactive (browser)
```bash
fab auth login
# prompts: "Interactive with a web browser"
```

### Service Principal — client secret
```bash
fab auth login -u <client_id> -p <client_secret> --tenant <tenant_id>
```

### Service Principal — certificate
```bash
fab auth login -u <client_id> --certificate /path/to/cert.pem --tenant <tenant_id>
# password-protected cert:
fab auth login -u <client_id> --certificate /path/to/cert.pfx -p <cert_password> --tenant <tenant_id>
```

### Service Principal — federated token (GitHub Actions / Azure Pipelines)
```bash
fab config set fab_encryption_fallback_enabled true
fab auth login -u <client_id> --federated-token <token> --tenant <tenant_id>
```

### Managed Identity
```bash
fab auth login --identity                   # system-assigned
fab auth login --identity -u <client_id>    # user-assigned
```

> **Tip:** For CI/CD, store `FAB_CLIENT_ID` and `FAB_TENANT_ID` as pipeline secrets and never
> hard-code them in scripts.

---

## File-System Commands (`fab fs`)

All workspace/item operations follow a UNIX-like path convention:
`/<workspace_name>.Workspace/<item_name>.<ItemType>`

### Navigation & Inspection

```bash
fab ls                                    # list all workspaces
fab ls ws1.Workspace                      # list items in a workspace
fab ls ws1.Workspace -l                   # detailed listing
fab cd ws1.Workspace                      # change directory (interactive mode)
fab pwd                                   # print current path
fab exists ws1.Workspace/nb1.Notebook     # check existence (exit 0 = exists)
fab get ws1.Workspace/nb1.Notebook        # get item properties (JSON)
fab get ws1.Workspace/nb1.Notebook -q "description"   # JMESPath query
fab open ws1.Workspace                    # open in browser
```

### Creating & Deleting

```bash
fab mkdir MyWorkspace.Workspace           # create workspace
fab mkdir MyWorkspace.Workspace -P "param1=val1,param2=val2"
fab rm ws1.Workspace/nb1.Notebook         # delete item (prompts for confirmation)
fab rm ws1.Workspace/nb1.Notebook -f      # force delete, no prompt
```

### Copying & Moving

```bash
# copy within workspace
fab cp ws1.Workspace/nb1.Notebook ws1.Workspace/nb1_copy.Notebook

# copy to different workspace
fab cp ws1.Workspace/nb1.Notebook ws2.Workspace/nb1.Notebook

# move to different workspace
fab mv ws1.Workspace/nb1.Notebook ws2.Workspace/nb1.Notebook

# -f flag overwrites destination without prompting; sensitivity labels are NOT copied
```

### Export (Fabric → local)

```bash
# export a single item
fab export ws1.Workspace/nb1.Notebook -o /tmp/export/

# export entire workspace
fab export ws1.Workspace -o /tmp/export/

# export all items, skip confirmation prompts
fab export ws1.Workspace -o /tmp/export/ -a -f
```

> Note: Sensitivity labels are **not** exported. On Windows, certain characters in item names
> are invalid as folder names — rename before exporting if needed.

### Import (local → Fabric)

```bash
# import a single item (overwrites if exists with -f)
fab import ws1.Workspace/nb1.Notebook -i /tmp/export/nb1.Notebook -f

# supported item types for import:
# Notebook, SparkJobDefinition, DataPipeline, Report, SemanticModel,
# KQLDatabase, KQLQueryset, KQLDashboard, Eventhouse, Eventstream,
# MirroredDatabase, Reflex, MountedDataFactory, CopyJob, VariableLibrary, Environment

# import notebook from a .py file
fab import ws1.Workspace/nb1.Notebook -i /tmp/nb1.py --format .py -f
```

### Set Properties

```bash
# update a specific property via JMESPath
fab set ws1.Workspace/nb1.Notebook -q "description" -i "New description"
```

### Shortcuts

```bash
fab ln ws1.Workspace/shortcut1 --type oneLake --target /ws2.Workspace/lh1.Lakehouse
# external types: adlsGen2, amazonS3, dataverse, googleCloudStorage, s3Compatible
```

### Start / Stop / Assign

```bash
fab start ws1.Workspace/kqldb1.KQLDatabase
fab stop  ws1.Workspace/kqldb1.KQLDatabase
fab assign  resource1 -W ws1.Workspace
fab unassign resource1 -W ws1.Workspace
```

---

## Common Flags Reference

| Flag | Meaning |
|------|---------|
| `-f` / `--force` | Skip confirmation prompts; overwrite existing |
| `-a` / `--all` | Include all items (used with `ls`, `export`, `cp`, `mv`) |
| `-l` / `--long` | Long/detailed listing |
| `-o` / `--output` | Output path (required for `export`) |
| `-i` / `--input` | Input path or value (required for `import`, `set`) |
| `-q` / `--query` | JMESPath expression to filter/target JSON fields |
| `-v` / `--verbose` | Full JSON output |
| `-P` / `--params` | `key=value` pairs, comma-separated (used with `mkdir`) |
| `-W` / `--workspace` | Workspace name (used with `assign`/`unassign`) |
| `--format` | Input format — only for Notebooks (`.ipynb` default, `.py` supported) |
| `--type` | Shortcut type (used with `ln`) |
| `--target` | Shortcut target path (required for external shortcuts) |

---

## CI/CD Integration

### Azure Pipelines example

```yaml
- task: UsePythonVersion@0
  inputs:
    versionSpec: '3.12'

- script: pip install ms-fabric-cli
  displayName: Install Fabric CLI

- script: |
    fab config set fab_encryption_fallback_enabled true
    fab auth login -u $(FAB_CLIENT_ID) -p $(FAB_CLIENT_SECRET) --tenant $(FAB_TENANT_ID)
  displayName: Authenticate

- script: |
    fab import ws_prod.Workspace/rep1.Report -i $(Pipeline.Workspace)/drop/rep1.Report -f
  displayName: Deploy Report
```

### GitHub Actions example

```yaml
permissions:
  id-token: write
  contents: read

- name: Install Fabric CLI
  run: pip install ms-fabric-cli

- name: Login (federated)
  env:
    FAB_CLIENT_ID: ${{ secrets.FAB_CLIENT_ID }}
    FAB_TENANT_ID: ${{ secrets.FAB_TENANT_ID }}
  run: |
    fab config set fab_encryption_fallback_enabled true
    FED_TOKEN=$(curl -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=api://AzureADTokenExchange" | jq -r '.value')
    fab auth login -t $FAB_TENANT_ID -u $FAB_CLIENT_ID --federated-token $FED_TOKEN

- name: Deploy
  run: |
    fab import ws_prod.Workspace/rep1.Report -i ./artifacts/rep1.Report -f
```

---

## Scripting Patterns

### Promote items across environments

```bash
#!/usr/bin/env bash
set -euo pipefail

ITEM="rep1.Report"
SRC="ws_dev.Workspace"
DST="ws_prod.Workspace"
TMPDIR=$(mktemp -d)

fab export "$SRC/$ITEM" -o "$TMPDIR" -f
fab import "$DST/$ITEM" -i "$TMPDIR/$ITEM" -f

rm -rf "$TMPDIR"
```

### Check existence before acting

```bash
if fab exists ws1.Workspace/nb1.Notebook; then
  echo "Item exists, proceeding with export"
  fab export ws1.Workspace/nb1.Notebook -o /tmp/out -f
else
  echo "Item not found"
  exit 1
fi
```

### List all items of a specific type

```bash
fab ls ws1.Workspace -l | grep "\.Report"
```

---

## Path Convention Quick Reference

```
/                                    # root (all workspaces)
/<name>.Workspace                    # a workspace
/<name>.Workspace/<item>.<Type>      # an item
/<name>.Workspace/<item>.<Type>/...  # item sub-path (files inside an item)
```

Item types: `Workspace`, `Notebook`, `Report`, `SemanticModel`, `DataPipeline`,
`SparkJobDefinition`, `Lakehouse`, `KQLDatabase`, `KQLQueryset`, `KQLDashboard`,
`Eventhouse`, `Eventstream`, `MirroredDatabase`, `Reflex`, `MountedDataFactory`,
`CopyJob`, `VariableLibrary`, `Environment`

---

## Further Reference

- Official docs & cheatsheet: https://microsoft.github.io/fabric-cli/
- Auth examples: https://microsoft.github.io/fabric-cli/examples/auth_examples
- Workspace examples: https://microsoft.github.io/fabric-cli/examples/workspace_examples
- Item examples: https://microsoft.github.io/fabric-cli/examples/item_examples