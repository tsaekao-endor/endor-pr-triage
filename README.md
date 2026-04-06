# Endor Labs PR Triage

A GitHub Action and companion workflow that lets developers triage [Endor Labs](https://www.endorlabs.com) security findings directly from pull request comments — no platform login required.

After an Endor Labs PR scan completes, a bot comment is posted with a numbered table of findings. Developers reply with simple commands to mark findings as false positives or accepted risks. The workflow automatically creates ignore entries in `.endorignore.yaml`, commits them to the PR branch, and replies with a confirmation.

> **No scripts to copy.** The triage comment step is a single `uses:` line in your existing scan workflow. The triage handler downloads its script at runtime.

---

## How it works

```
┌─────────────────────────────────────────────────────────┐
│  1. PR opened → your endor-scan.yml runs                │
│     • Endor Labs scans the branch                       │
│     • Post Endor Labs triage comment step fires         │
│       (only when CI-blocking/warning findings exist)    │
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
│     • Downloads handle_triage_command.py at runtime     │
│     • Looks up finding UUIDs from the hidden map        │
│     • Runs endorctl ignore for each finding             │
│     • Commits .endorignore.yaml to the PR branch        │
│     • Replies with a confirmation comment               │
└─────────────────────────────────────────────────────────┘
```

---

## Example PR comment

The bot posts a comment like this after every scan (only when findings are present):

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

### Step 1 — Add the triage comment step to your scan workflow

In your existing Endor Labs PR scan job, add one step after the scan:

```yaml
- name: Post Endor Labs triage comment
  if: always()
  uses: tsaekao-endor/endor-pr-triage@main
  with:
    namespace: ${{ vars.ENDOR_NAMESPACE }}
    project_uuid: ${{ vars.ENDOR_PROJECT_UUID }}
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

That's it. No scripts to copy. See [`templates/endor-scan.yml`](templates/endor-scan.yml) for a complete example.

### Step 2 — Add the triage handler workflow

Copy [`templates/endor-triage.yml`](templates/endor-triage.yml) to `.github/workflows/endor-triage.yml` in your repository. This is a one-time setup. The script it runs is downloaded automatically at runtime from this repo — no local copy needed.

### Step 3 — Configure repository variables

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

### Step 4 — Configure workflow permissions

Go to **Settings → Actions → General → Workflow permissions** and enable:

- **Read and write permissions**
- **Allow GitHub Actions to create and approve pull requests**

---

## Authentication

Both workflows use **keyless authentication** via GitHub Actions OIDC tokens. No API keys or secrets need to be stored — Endor Labs verifies the workflow's identity automatically.

This requires:
- `id-token: write` permission on the job (already included in the templates)
- The GitHub Actions OIDC integration enabled in your Endor Labs tenant

---

## When does the triage comment appear?

The triage comment is only posted when the scan finds findings tagged as **CI Blocker** or **CI Warning** by your Endor Labs action policy. If a scan is clean, no comment is posted.

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

This appends an entry to `.endorignore.yaml` in the root of your repository and commits it to the PR branch. When the PR is merged, the ignore entry is picked up by all future scans.

---

## Repository layout

```
endor-pr-triage/
├── action.yml                       ← composite action (the uses: target)
├── scripts/
│   ├── post_triage_comment.py       ← fetches findings, builds table, posts comment
│   └── handle_triage_command.py     ← parses /endor commands, runs endorctl ignore
└── templates/
    ├── endor-scan.yml               ← sample scan workflow (PR + push)
    └── endor-triage.yml             ← sample triage handler workflow
```

---

## Requirements

- [Endor Labs](https://www.endorlabs.com) account with a project configured
- GitHub repository with Actions enabled
- `endorctl` — installed automatically by the action at runtime
- Python 3.10+ — available by default on `ubuntu-latest` GitHub Actions runners
