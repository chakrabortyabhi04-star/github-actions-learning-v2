# GitHub Actions — Days 1–5 Notes

## Day 1 — CI/CD Mental Model and Setup

### What is CI/CD?
CI/CD stands for Continuous Integration and Continuous Deployment. Before CI/CD existed, teams manually checked code, ran tests by hand, and deployed by copying files to servers. This caused bugs that only appeared in production, broken deployments, and the classic "it works on my machine" problem.

CI/CD solves this by automating the entire process — every time code is pushed, a machine automatically validates, tests, and deploys it without human intervention.

- **CI (Continuous Integration):** Every push triggers automated checks — linting, testing, building
- **CD (Continuous Deployment):** Once checks pass, code is automatically deployed to staging or production

### GitHub Actions Architecture — 5 Core Components

| Component | What it is |
|-----------|-----------|
| **Workflow** | The entire automation file (`.yml`) — the top-level container |
| **on:** | The event that triggers the workflow (push, PR, schedule, manual) |
| **jobs:** | A collection of work to be done — can run in parallel or in sequence |
| **steps:** | Individual tasks inside a job — run commands or use pre-built actions |
| **runs-on:** | Tells GitHub which machine type to spin up for the job |

### Runners
A runner is a machine that executes your workflow. Key facts:
- **Ephemeral by design** — every job gets a fresh machine, discarded after completion. No configuration drift.
- **GitHub-hosted runners** — managed by GitHub, available immediately: `ubuntu-latest`, `windows-latest`, `macos-latest`
- **Self-hosted runners** — your own machines, used when workflows need access to internal networks or private infrastructure

### Repository Setup
- Repo: `github-actions-learning-v2`
- Workflow files live exclusively at: `.github/workflows/` — the dot before `github` is mandatory
- Files anywhere else are ignored by GitHub Actions

### Git Workflow
```bash
git add .           # Stage all changes (dot = everything from current directory downward)
git commit -m "message"   # Snapshot with a description
git push origin main      # Send to GitHub
```

---

## Day 2 — Workflow Anatomy

### First Complete Workflow
Every GitHub Actions workflow needs these six fields at minimum:

```yaml
name: my first workflow        # Label shown in Actions tab
on: push                       # What triggers it

jobs:
  build:                       # Job name (you choose this)
    runs-on: ubuntu-latest     # Which machine to use
    steps:
      - name: Printing Message # Step label
        run: echo "Hello world" # Shell command to execute
```

### Key Concepts

**Why does `steps:` live inside a job?**
Because each job owns its steps. Different jobs can run on different machines with completely different steps. A job is like a maintenance task assigned to a specific server — the checklist (steps) belongs to that task, not to the whole department.

**Set up job and Complete job — where do they come from?**
You never write these. GitHub Actions injects them automatically into every job:
- **Set up job** — provisions the runner, sets permissions, prepares the workspace
- **Complete job** — cleans up processes and discards the runner

**YAML rule: space after colon is mandatory**
```yaml
name:my workflow    # WRONG — breaks YAML
name: my workflow   # CORRECT
```

### Hands-on Result
- Pushed `day1.yml` with `echo "Hello world"`
- Watched it execute live in the Actions tab
- Confirmed "Hello world" printed in the Printing Message step

---

## Day 3 — Triggers In Depth

### Multiple Triggers
`on:` accepts multiple triggers as sibling keys:

```yaml
on:
  push:
    branches:
      - main
  pull_request:
```

**Common mistake:** Nesting `pull_request` under `push` makes GitHub read it as a branch name. They must be at the same indentation level.

### Branch Filtering
```yaml
on:
  push:
    branches:
      - main    # Only fires on pushes to main, not other branches
```

YAML list items always start with `- ` (dash + space).

### pull_request Trigger
Fires when a PR event happens — NOT on direct pushes.

**Activity types:**
- `opened` — PR is first created
- `synchronize` — new commits pushed to the PR branch
- `closed` — PR is merged or closed

Default behaviour (no types specified): fires on `opened` and `synchronize`.

### Production PR Workflow
This is how real teams work:
1. Developer pushes to a feature branch
2. Developer opens a Pull Request
3. **GitHub Actions fires immediately** — automated checks run before any human reviews
4. Team lead sees "All checks passed" → reviews code → approves → merges
5. Merge to main triggers deployment pipeline

> Key insight: By the time a human reviews the PR, the machine has already verified it. Automated checks are your pre-flight checklist before the CAB review.

### Hands-on Result
- Added `pull_request:` trigger to `day1.yml`
- Created `feature.test-pr` branch, made a change, opened a PR
- Confirmed "All checks have passed — 1 successful check" appeared on the PR automatically

---

## Day 4 — workflow_dispatch and Inputs

### Manual Trigger
`workflow_dispatch` adds a "Run workflow" button in the GitHub Actions tab:

```yaml
on:
  workflow_dispatch:
```

No code push needed — anyone with repo access can trigger it on demand.

**Production use cases:**
- On-demand deployments to production
- Manual rollbacks when something breaks
- Running maintenance jobs (database cleanup, cache purge)
- Emergency hotfix deployments

### Inputs — Making Manual Triggers Dynamic
```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - production
```

**Input types:** `string`, `boolean`, `choice`, `number`

**`required: true`** — workflow refuses to run if not filled in
**`default:`** — used when input is optional and user leaves it blank

### Accessing Input Values
```yaml
run: echo ${{ inputs.environment }}
```

`${{ }}` is GitHub Actions expression syntax. This is how you read inputs, secrets, contexts, and variables anywhere in your workflow.

### Cron Schedule
```yaml
on:
  schedule:
    - cron: '0 9 * * *'    # Every day at 9:00 AM UTC
```

**Cron field order — memorise this:**
```
minute  hour  day-of-month  month  day-of-week
  0      9         *           *        *
```
Memory trick: **M**y **H**amster **D**oes **M**ath **W**eekly → Minute, Hour, Day, Month, Weekday

### Git Tip — Single Quotes in Commit Messages
If your commit message contains `${{ }}`, use single quotes to prevent bash from trying to interpret it:
```bash
git commit -m 'add inputs.environment to echo step'   # CORRECT
git commit -m "add ${{ inputs.environment }}"          # WRONG — bash substitution error
```

### Hands-on Result
- Added `workflow_dispatch:` with `type: choice` input (staging/production)
- Confirmed dropdown appeared in GitHub Actions UI
- Triggered manually with "production" → `production` printed in logs

---

## Day 5 — Jobs, needs, and Sequential Pipelines

### Why Multiple Jobs?
In a real pipeline, work is broken into stages. Running everything in one job means:
- A security failure doesn't stop the build from running
- You can't see clearly which stage failed
- You can't run independent stages in parallel

### needs: — Creating Job Dependencies
```yaml
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Validate code
        run: echo "validate successful"

  security:
    needs: validate          # Won't start until validate passes
    runs-on: ubuntu-latest
    steps:
      - name: Security scan
        run: echo "security successful"

  build:
    needs: security          # Won't start until security passes
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: echo "build successful"
```

**What happens when a job fails?**
If `validate` fails → `security` is skipped → `build` is skipped. GitHub Actions stops the chain immediately. No wasted runner minutes.

### Real Production Pipeline Stages

| Stage | What it does | Tools used in production |
|-------|-------------|--------------------------|
| **Validate** | Syntax check, linting | eslint, pylint, terraform validate |
| **Security** | Secrets detection, vulnerability scan | trufflehog, snyk, trivy |
| **Build** | Compile, package, containerise | docker build, npm build, maven |
| **Test** | Unit tests, integration tests | jest, pytest, junit |
| **Deploy** | Push to staging or production | kubectl, terraform apply, ansible |

### Multiple Workflow Files
Each `.yml` file in `.github/workflows/` is a separate workflow. When multiple workflows have the same `on: push` trigger, **all of them fire on every push**.

To limit a workflow to only fire when its own file changes:
```yaml
on:
  push:
    branches:
      - main
    paths:
      - '.github/workflows/day5.yml'
```

### VS Code Extensions for GitHub Actions
Install these to catch YAML errors automatically:
- **YAML** by Red Hat — real-time YAML syntax validation
- **GitHub Actions** by GitHub — autocomplete and validation specific to Actions syntax

### Hands-on Result
- Created `day5.yml` with three sequential jobs: `validate` → `security` → `build`
- Pushed to GitHub and confirmed the visual pipeline showing all three jobs connected by dots
- Observed both `day1.yml` and `day5.yml` firing on the same push event

---

## Quick Reference — YAML Rules

| Rule | Wrong | Right |
|------|-------|-------|
| Space after colon | `name:value` | `name: value` |
| List item dash | `-item` | `- item` |
| Step name and run together | `- name: X` then `- run: Y` | `- name: X` then `  run: Y` (same item) |
| No space before colon | `branches :` | `branches:` |

---

## Interview Answer Bank

**"What is GitHub Actions?"**
> GitHub Actions is a CI/CD platform built into GitHub. You define workflows in YAML files inside `.github/workflows/`. Workflows are triggered by events — push, pull request, schedule, or manual — and run jobs on ephemeral machines called runners. Each job contains steps that execute shell commands or reuse pre-built actions from the marketplace.

**"What is a runner?"**
> A runner is an ephemeral machine that executes a job. GitHub provides hosted runners for Ubuntu, Windows, and macOS. Every job gets a fresh machine — discarded after completion — so there's no configuration drift between runs. Self-hosted runners are used when workflows need access to internal networks or private infrastructure.

**"What is needs: used for?"**
> `needs:` creates a dependency between jobs. If job B has `needs: jobA`, it won't start until jobA completes successfully. If jobA fails, jobB is skipped automatically. This prevents wasting compute and ensures failures are caught early — no point building code that failed validation.

**"How do you trigger a workflow manually?"**
> Using `workflow_dispatch`. It adds a "Run workflow" button in the Actions tab. You can define inputs with types — string, boolean, choice, number — so the person triggering can pass values in at runtime. Teams use this for on-demand deployments and manual rollbacks.

**"What is the difference between push and pull_request triggers?"**
> `push` fires when commits are pushed directly to a branch. `pull_request` fires when a PR event happens — opened, updated, or merged. In a real team workflow, developers push to feature branches and open PRs. The `pull_request` trigger runs automated checks on the PR before the team lead reviews it.

---

*Notes maintained by Claude | Last updated: Day 5*