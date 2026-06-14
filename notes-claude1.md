# GitHub Actions — Days 1–6 Notes
> Written by Claude | Repository: `github-actions-learning-v2` | github.com/chakrabortyabhi04-star

---

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
git add .                      # Stage all changes
git commit -m "message"        # Snapshot with a description
git push origin main           # Send to GitHub
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

## Day 6 — Environment Variables and Contexts

### Why Variables Exist
Three reasons variables exist in GitHub Actions:

1. **Single source of truth** — define once, use everywhere. Change `VERSION: 1.0.0` to `VERSION: 2.0.0` in one place and every job picks it up automatically.
2. **Separation of config from code** — workflow logic stays the same, only values change between environments.
3. **Security** — sensitive values like passwords and API keys never get hardcoded. They live in GitHub Secrets and get injected as variables at runtime.

### Three Levels of env: Scope

```yaml
name: scoped variables example

env:                          # WORKFLOW LEVEL — available in every job and step
  APP_NAME: my-app
  VERSION: 1.0.0

jobs:
  security:
    runs-on: ubuntu-latest
    env:                      # JOB LEVEL — available only in this job's steps
      SCAN_LEVEL: deep
    steps:
      - name: step level example
        env:                  # STEP LEVEL — available only in this one step
          TEMP_TOKEN: abc123
        run: echo "$APP_NAME $SCAN_LEVEL $TEMP_TOKEN"
```

**Scope rule:** Variables only flow downward — a job-level variable is invisible to other jobs. A step-level variable is invisible to other steps in the same job.

### Accessing env: Variables in run: Commands
```yaml
run: echo "$APP_NAME $VERSION"    # Shell syntax — $ prefix
```

Shell syntax works because GitHub Actions injects all `env:` variables into the runner's shell environment before executing `run:` commands.

### GitHub Contexts — Built-in Variables
GitHub automatically provides information about every run. You don't define these — they just exist.

```yaml
steps:
  - name: Print context info
    run: |
      echo "Triggered by: ${{ github.actor }}"
      echo "Branch: ${{ github.ref }}"
      echo "Event: ${{ github.event_name }}"
```

| Context | What it contains | Real example |
|---------|-----------------|--------------|
| `github.actor` | Username who triggered the run | `chakrabortyabhi04-star` |
| `github.ref` | Full branch reference | `refs/heads/main` |
| `github.event_name` | Type of event that fired | `push`, `pull_request`, `workflow_dispatch` |
| `github.sha` | Exact commit hash | `a83e2e7f...` |
| `github.repository` | Repo name | `chakrabortyabhi04-star/github-actions-learning-v2` |

### run: | — Multi-line Shell Commands
```yaml
run: |
  echo "line one"
  echo "line two"
  echo "line three"
```

The `|` (pipe character) after `run:` tells YAML to treat everything indented below as a multi-line string. Each line runs as a separate shell command in sequence.

### Real Production Workflow Using Variables and Contexts

```yaml
name: Production Deployment Pipeline

env:
  APP_NAME: customer-portal
  AZURE_REGION: eastus
  DOCKER_REGISTRY: mycompany.azurecr.io

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy target'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Print deployment info
        run: |
          echo "App: $APP_NAME"
          echo "Triggered by: ${{ github.actor }}"
          echo "Branch: ${{ github.ref }}"
          echo "Event: ${{ github.event_name }}"

  security:
    needs: validate
    runs-on: ubuntu-latest
    env:
      SCAN_LEVEL: deep
    steps:
      - name: Scan for secrets
        run: echo "Running $SCAN_LEVEL scan on $APP_NAME"

  build:
    needs: security
    runs-on: ubuntu-latest
    steps:
      - name: Build Docker image
        run: |
          echo "Building $DOCKER_REGISTRY/$APP_NAME:${{ github.sha }}"
          echo "Pushing to $AZURE_REGION registry"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to environment
        run: |
          echo "Deploying $APP_NAME to ${{ inputs.environment }}"
          echo "Image tag: ${{ github.sha }}"
```

**What each part does in real life:**
- `APP_NAME`, `DOCKER_REGISTRY` — defined once, used in 3+ jobs. Change in one place, updates everywhere.
- `github.actor` — audit trail. Every deployment log shows who triggered it.
- `github.sha` — the exact commit hash becomes the Docker image tag. You always know which exact code is running in production.
- `SCAN_LEVEL: deep` scoped to security job — other jobs don't need this.

### Hands-on Result
- Added workflow-level `env:` with `APP_NAME: my-app` and `VERSION: 1.0.0`
- Confirmed `my-app 1.0.0` printed in all three jobs via `$APP_NAME $VERSION`
- Added `SCAN_LEVEL: deep` at job level in `security` — confirmed `deep` printed only there
- Added `github.actor`, `github.ref`, `github.event_name` context step — live values printed in logs
- Observed schedule trigger firing automatically at 9:00 AM UTC (2:30 PM IST)

---

## Quick Reference — YAML Rules

| Rule | Wrong | Right |
|------|-------|-------|
| Space after colon | `name:value` | `name: value` |
| List item dash | `-item` | `- item` |
| Step name and run together | `- name: X` then `- run: Y` | `- name: X` then `  run: Y` (same item) |
| No space before colon | `branches :` | `branches:` |
| Multi-line run command | `run: echo a echo b` | `run: |` then indented lines |

---

## Two Ways to Access Variables — Summary

| Syntax | Used for | Example |
|--------|----------|---------|
| `$VARIABLE_NAME` | `env:` variables in `run:` shell commands | `echo "$APP_NAME"` |
| `${{ context.property }}` | GitHub contexts, inputs, secrets | `${{ github.actor }}` |

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

**"How do you use environment variables in GitHub Actions?"**
> Environment variables are defined using `env:` at three levels — workflow, job, or step. Workflow-level variables are available everywhere. Job-level variables are scoped to that job only. They're accessed in `run:` commands using shell syntax like `$APP_NAME`. GitHub also provides built-in context variables like `${{ github.actor }}` and `${{ github.ref }}` that contain information about the run — who triggered it, which branch, what event.

**"What are GitHub Actions contexts?"**
> Contexts are built-in objects GitHub provides automatically for every workflow run. The most useful is the `github` context — it contains `github.actor` (who triggered the run), `github.ref` (which branch), `github.event_name` (push, PR, or manual), and `github.sha` (the exact commit hash). Teams use these for audit trails, conditional logic, and tagging Docker images with the exact commit that built them.

---

*Notes maintained by Claude | Last updated: Day 6*

---

## Day 7 — Secrets

### Why Secrets Exist
Hardcoding sensitive values like passwords and API keys in YAML files is a critical security risk:
- The file lives in Git history permanently — even after deletion, every previous commit still contains it
- Automated bots scan GitHub for accidentally committed secrets within seconds of a push
- Anyone with repo access can read the YAML file and extract credentials

GitHub Secrets solves this by storing sensitive values encrypted on GitHub's servers, injecting them at runtime, and masking them in logs automatically.

### How GitHub Secrets Work
- Stored encrypted at rest on GitHub's servers
- Never visible after saving — not even to the person who created them
- Automatically masked in logs — if the value appears in output, GitHub replaces it with `***`
- Accessed in workflows via `${{ secrets.SECRET_NAME }}`
- Never appear in the YAML file itself

### Adding a Secret
Go to: **Repo → Settings → Secrets and variables → Actions → New repository secret**

Name your secret in CAPS_WITH_UNDERSCORES convention (e.g. `MY_API_KEY`, `DB_PASSWORD`, `AZURE_CLIENT_SECRET`)

### Using Secrets in Workflows

```yaml
jobs:
  secret-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Use secret directly
        run: echo "API Key is ${{ secrets.MY_API_KEY }}"

      - name: Use secret via env variable
        env:
          DB_PASS: ${{ secrets.DB_PASSWORD }}
        run: echo "DB Password is $DB_PASS"
```

Both approaches mask the value in logs as `***`. The `env:` approach is preferred when passing secrets to scripts or tools that read environment variables.

### Three Types of Secrets — Scope Comparison

| Type | Scope | Where set | Production use |
|------|-------|-----------|----------------|
| **Repository secret** | One repo only | Repo → Settings → Secrets | App-specific API keys |
| **Environment secret** | One environment (staging/prod) | Repo → Settings → Environments | Different DB passwords per environment |
| **Organisation secret** | All repos in the org | Org → Settings → Secrets | Azure SP credentials shared across all repos |

> Production insight: In a real company, Azure Service Principal credentials live as **organisation secrets** so every team's repo can deploy to Azure without storing credentials separately in each repo. Individual teams never see the actual values.

### Day 7 Workflow File — day7.yml
```yaml
name: Secret Example
on: push

jobs:
  secret-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Use Secret
        run: echo "API Key is ${{ secrets.MY_API_KEY }}"

      - name: Print actor
        run: echo "${{ github.actor }}"
```

### Hands-on Result
- Stored `MY_API_KEY` as a repository secret with value `abc123-fake-key`
- Pushed `day7.yml` and confirmed `API Key is ***` in logs — value masked automatically
- Confirmed `github.actor` prints in plain text — contexts are not masked, only secrets are

---

### Day 7 — Interview Questions

**Q: What is the difference between GitHub Secrets and env: variables?**
> `env:` variables store non-sensitive configuration values — they appear in plain text in your YAML file and in logs. GitHub Secrets store sensitive values encrypted on GitHub's servers — they never appear in YAML, are automatically masked in logs as `***`, and cannot be read back after saving. Use `env:` for things like app names and versions; use Secrets for passwords, API keys, and tokens.

**Q: If a developer accidentally runs `echo ${{ secrets.MY_SECRET }}` in a workflow, what happens?**
> GitHub automatically detects that the output matches a known secret value and replaces it with `***` in the logs. This masking happens at the runner level before logs are uploaded to GitHub. It's a safety net — but it's not foolproof. If the secret is transformed (e.g. base64 encoded), the masking may not catch it. Best practice is to never echo secrets intentionally.

**Q: What is the difference between repository secrets, environment secrets, and organisation secrets?**
> Repository secrets are scoped to a single repo. Environment secrets are scoped to a specific deployment environment (staging or production) within a repo — useful for having different database passwords per environment. Organisation secrets are shared across all repos in the organisation — ideal for credentials like Azure Service Principals that every team needs for deployments. Organisation secrets are managed centrally, so rotating credentials only needs to happen in one place.

**Q: How would you pass a secret to a Docker build command?**
> Never pass secrets as build arguments — they get baked into the image layers and can be extracted. Instead, use Docker BuildKit secrets or pass them as environment variables scoped to the specific step using `env:` at step level, so they're only available during that one command and not inherited by other steps.

**Q: Can a pull request from a forked repo access repository secrets?**
> No — by default, GitHub does not pass secrets to workflows triggered by pull requests from forks. This is a deliberate security measure to prevent a malicious fork from opening a PR specifically to exfiltrate your secrets. You can explicitly allow this in Settings, but it's strongly discouraged for production repos.

---

*Notes maintained by Claude | Last updated: Day 7*

---

## Day 8 — Actions and the Marketplace

### Why the Marketplace Exists
Every team in the world needs to: checkout code, login to Azure, build Docker images, run tests. If every team writes their own shell commands for these tasks — when GitHub changes how authentication works or Docker changes its CLI, every team has to rewrite their YAML files.

The Actions Marketplace solves this — pre-built, reusable steps maintained by GitHub, cloud providers, and the community. When the underlying tool changes, the action maintainer updates it and your workflow picks up the fix automatically.

### Two Types of Steps

```yaml
steps:
  # Type 1 — you write the shell command
  - name: Print something
    run: echo "hello"

  # Type 2 — use a pre-built action from the marketplace
  - name: Checkout code
    uses: actions/checkout@v4
```

### Understanding the uses: Syntax
```
actions/checkout@v4
│       │        │
│       │        └── Version tag — pins to a specific release
│       └────────── Action name
└────────────────── GitHub organisation that maintains it
```

**Always pin to a version** (`@v4` not `@latest`) — prevents your workflow from breaking if the action releases a major update with breaking changes.

### actions/checkout@v4 — The Most Used Action

Without checkout, the runner starts as a completely blank machine — it has no knowledge of your files. `actions/checkout@v4` clones your repository onto the runner so every subsequent step can access your code.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: List files (proves checkout worked)
        run: ls -la
```

**What checkout does automatically:**
- Syncs the repository from GitHub
- Sets up Git authentication
- Fetches the specific commit that triggered the workflow
- Makes all your files available at the working directory

**Checkout is almost always the first step in any real pipeline.**

### paths: Filter — Scoping Workflow Triggers
```yaml
on:
  push:
    branches:
      - main
    paths:
      - '.github/workflows/day8.yml'  # Only fire when this file changes
```

**Known behaviour:** `paths:` filters are unreliable when only the workflow file itself is the changed path on first creation. Subsequent pushes work correctly.

**Production use:** Use `paths:` to prevent unnecessary runs — for example, only run the frontend pipeline when frontend files change:
```yaml
paths:
  - 'src/frontend/**'
  - 'package.json'
```

### Common Marketplace Actions You'll Use Daily

| Action | What it does | When to use |
|--------|-------------|-------------|
| `actions/checkout@v4` | Clones repo onto runner | Always — first step in every pipeline |
| `actions/setup-node@v4` | Installs Node.js on runner | Before `npm install` or `npm run build` |
| `actions/setup-python@v5` | Installs Python on runner | Before `pip install` or `pytest` |
| `azure/login@v2` | Authenticates to Azure | Before any Azure CLI or Terraform commands |
| `docker/build-push-action@v5` | Builds and pushes Docker images | CI pipelines building containers |
| `actions/upload-artifact@v4` | Saves files between jobs | Passing build output to deploy job |

### Real Production Pipeline with Checkout

```yaml
name: Node.js CI Pipeline

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test

      - name: Build application
        run: npm run build
```

### Hands-on Result
- Created `day8.yml` with `actions/checkout@v4` and `ls -la`
- Confirmed repo files visible on runner after checkout: `.git`, `.github`, `notes-claude.md`
- Understood that without checkout, runner has zero knowledge of your code

---

### Day 8 — Interview Questions

**Q: What is the difference between `run:` and `uses:` in a GitHub Actions step?**
> `run:` executes a shell command you write yourself — useful for simple tasks like `echo`, `ls`, or calling scripts. `uses:` references a pre-built action from the GitHub Marketplace or another repository — useful for complex tasks like checking out code, logging into cloud providers, or building Docker images. Pre-built actions encapsulate complexity so you don't have to write and maintain the underlying shell commands yourself.

**Q: Why do you always pin actions to a specific version like `@v4` instead of `@latest`?**
> Pinning to a specific version like `@v4` ensures your workflow doesn't break when the action maintainer releases a major update with breaking changes. Using `@latest` means your workflow could silently change behaviour overnight. In production pipelines, stability is more important than always having the newest version. When you're ready to upgrade, you explicitly change the version and test the change.

**Q: Why is `actions/checkout@v4` almost always the first step in a pipeline?**
> GitHub-hosted runners start as completely blank machines — they have no knowledge of your repository or code. `actions/checkout@v4` clones the repository onto the runner so every subsequent step can access your files. Without it, commands like `npm install`, `terraform init`, or `docker build` would fail because none of your files exist on the machine.

**Q: How would you find and evaluate a marketplace action before using it in production?**
> I go to github.com/marketplace and search for the action. I check: who maintains it (GitHub official actions are safest), how many stars and recent commits it has, whether it's marked "Verified Creator", what version is current, and I read the README for required inputs. For production, I also check the action's source code to understand what it does — you should never blindly trust third-party code running in your pipeline with access to your secrets.

**Q: What is the `with:` keyword used for in a uses: step?**
> `with:` passes inputs to a marketplace action — similar to how `inputs:` works in `workflow_dispatch`. Each action defines its own accepted inputs. For example, `actions/setup-node` accepts `node-version` to specify which Node.js version to install. Without `with:`, the action uses its default values.

---

*Notes maintained by Claude | Last updated: Day 8*