# Endor Labs PR Triage

A GitHub Actions workflow that lets developers triage [Endor Labs](https://www.endorlabs.com) security findings directly from pull request comments — no platform login required.

After an Endor Labs PR scan completes, a bot comment is posted with a numbered table of findings. Developers reply with simple commands to mark findings as false positives or accepted risks. The workflow automatically creates ignore entries in `.endorignore.yaml`, commits them to the PR branch, and replies with a confirmation.

---

## How it works

```
┌─────────────────────────────────────────────────────────┐
│  1. PR opened → endor-scan.yml runs                     │
│     • Endor Labs scans the branch                       │
│     • post_triage_comment.py posts a findings table     │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  2. Developer comments /endor fp 1,2 on the PR          │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  3. endor-triage.yml fires                              │
│     • Looks up finding UUIDs from the hidden map        │
│     • Runs endorctl ignore for each finding             │
│     • Commits .endorignore.yaml to the PR branch        │
│     • Replies with a confirmation comment               │
└─────────────────────────────────────────────────────────┘
```

---

## Example PR comment

The bot posts a comment like this after every scan:

> **View PR scans on Endor Labs →**
>
> | Finding | Severity | Package | Vulnerability |
> |---------|----------|---------|---------------|
> | **1** | 🔴 Critical | `npm://lodash@4.17.4` | [GHSA-p6mc-m468-83gw](https://github.com/advisories/GHSA-p6mc-m468-83gw) Prototype Pollution in lodash |
> | **2** | 🟠 High | `npm://marked@0.3.5` | [GHSA-x5pg-88wf-qq4p](https://github.com/advisories/GHSA-x5pg-88wf-qq4p) Regular Expression Denial of Service in marked |

Developers can then reply:

```
/endor fp 1
/endor accept-risk 2
/endor fp 1,2 --comment="Not reachable in prod" --expires=2026-12-31 --expire-if-fix
```

---

## Triage commands

| Command | Effect |
|---------|--------|
| `/endor fp <numbers>` | Mark findings as **false positive** |
| `/endor accept-risk <numbers>` | Mark findings as **accepted risk** |

Finding numbers are comma-separated (e.g. `1,3,5`) and match the **Finding** column in the bot comment.

### Optional flags

| Flag | Description |
|------|-------------|
| `--comment="<text>"` | Add a note explaining the triage decision |
| `--expires=YYYY-MM-DD` | Auto-expire the ignore entry on this date |
| `--expire-if-fix` | Auto-expire the entry when a fix becomes available |

Flags can be combined:

```
/endor fp 1,2 --comment="Low risk for our environment" --expires=2026-06-30 --expire-if-fix
```

---

## Setup

### 1. Copy the files into your repository

```
your-repo/
├── .github/
│   └── workflows/
│       ├── endor-scan.yml       ← triggers on pull_request
│       └── endor-triage.yml     ← triggers on issue_comment
└── scripts/
    ├── post_triage_comment.py   ← posts the findings table
    └── handle_triage_command.py ← handles /endor commands
```

Copy the `.github/workflows/` and `scripts/` directories from this repo into your repository.

### 2. Configure repository variables

Go to **Settings → Secrets and variables → Actions → Variables** and add:

| Variable | Description | Example |
|----------|-------------|---------|
| `ENDOR_NAMESPACE` | Your Endor Labs tenant namespace | `my-company` |
| `ENDOR_PROJECT_UUID` | UUID of this project in Endor Labs | `abc123def456` |

To find your project UUID, run:

```bash
endorctl api list \
  --namespace=<your-namespace> \
  --resource=Project \
  --output-type=json \
  --filter='meta.name=="https://github.com/<org>/<repo>.git"' \
  | jq -r '.list.objects[0].uuid'
```

### 3. Configure workflow permissions

Go to **Settings → Actions → General → Workflow permissions** and enable:

- **Read and write permissions**
- **Allow GitHub Actions to create and approve pull requests**

### 4. Update the build step

In `endor-scan.yml`, replace the `npm install` step with whatever your project uses to install dependencies (e.g. `pip install`, `mvn package`, `gradle build`). This step is required so Endor Labs can resolve your dependency tree.

```yaml
# For Node.js
- name: Install dependencies
  run: npm install

# For Python
- name: Install dependencies
  run: pip install -r requirements.txt

# For Java (Maven)
- name: Build
  run: mvn package -DskipTests
```

### 5. Update the branch filter

In `endor-scan.yml`, update the `branches` filter to match your default branch:

```yaml
on:
  pull_request:
    branches: [main]   # or 'master', 'develop', etc.
```

---

## Authentication

Both workflows use **keyless authentication** via GitHub Actions OIDC tokens. No API keys or secrets need to be stored — Endor Labs verifies the workflow's identity automatically.

This requires:
- `id-token: write` permission on the job (already included in the templates)
- The GitHub Actions OIDC integration enabled in your Endor Labs tenant

---

## How ignore entries work

When a developer runs `/endor fp 1`, the workflow calls:

```bash
endorctl ignore \
  --namespace=<namespace> \
  --finding-uuid=<uuid> \
  --path=.endorignore.yaml \
  --prefix=<github-username> \
  --reason=false-positive \
  --username=<github-username>@github \
  --comments="Triaged via PR comment by <github-username>"
```

This appends an entry to `.endorignore.yaml` in the root of your repository:

```yaml
version: 1.0.0
ignore:
  - id: alice-1
    username: alice@github
    finding_name: 'GHSA-p6mc-m468-83gw: Prototype Pollution in lodash'
    parent_name: npm://my-app@1.0.0
    dependency_name: npm://lodash@4.17.4
    vuln_id: GHSA-p6mc-m468-83gw
    reason: false-positive
    comments: Triaged via PR comment by alice
```

The entry is committed to the PR branch. When the branch is merged to your default branch, the ignore entry is picked up by all future scans.

### Multiple teams, one ignore file

All ignore entries go to `.endorignore.yaml` at the repository root. Entries are logically scoped by their content (`parent_name`, `dependency_name`, `vuln_id`) — not by file location. The `--prefix` flag (set to the commenter's GitHub username) ensures entry IDs are unique across contributors.

---

## Files

| File | Purpose |
|------|---------|
| `.github/workflows/endor-scan.yml` | Runs the Endor Labs PR scan and posts the triage comment |
| `.github/workflows/endor-triage.yml` | Handles `/endor` commands posted in PR comments |
| `scripts/post_triage_comment.py` | Fetches findings, builds the sorted findings table, posts to PR |
| `scripts/handle_triage_command.py` | Parses commands, runs `endorctl ignore`, commits, replies |

---

## Requirements

- [Endor Labs](https://www.endorlabs.com) account with a project configured
- GitHub repository with Actions enabled
- `endorctl` — installed automatically by the workflow at runtime
- Python 3.10+ — available by default on `ubuntu-latest` GitHub Actions runners
