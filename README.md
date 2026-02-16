[![Build Status](https://dev.azure.com/ybend/peex_01_2025/_apis/build/status%2Fpeex_01_2025?branchName=master)](https://dev.azure.com/ybend/peex_01_2025/_build/latest?definitionId=8&branchName=master)
[![codebeat badge](https://codebeat.co/badges/0e006c74-a2f9-4f34-9cf4-2378fb7d995a)](https://codebeat.co/projects/github-com-edonosotti-ci-cd-tutorial-sample-app-master)
[![Maintainability](https://api.codeclimate.com/v1/badges/e14a2647843de209fd5e/maintainability)](https://codeclimate.com/github/edonosotti/ci-cd-tutorial-sample-app/maintainability)

# CD/CI Tutorial Sample Application

## Description

This sample Python REST API application was written for a tutorial on implementing Continuous Integration and Delivery pipelines.

It demonstrates how to:

 * Write a basic REST API using the [Flask](http://flask.pocoo.org) microframework
 * Basic database operations and migrations using the Flask wrappers around [Alembic](https://bitbucket.org/zzzeek/alembic) and [SQLAlchemy](https://www.sqlalchemy.org)
 * Write automated unit tests with [unittest](https://docs.python.org/2/library/unittest.html)

Also:

 * How to use [GitHub Actions](https://github.com/features/actions)

Related article: https://medium.com/rockedscience/docker-ci-cd-pipeline-with-github-actions-6d4cd1731030

## Requirements

 * `Python 3.8`
 * `Pip`
 * `virtualenv`, or `conda`, or `miniconda`

The `psycopg2` package does require `libpq-dev` and `gcc`.
To install them (with `apt`), run:

```sh
$ sudo apt-get install libpq-dev gcc
```

## Installation

With `virtualenv`:

```sh
$ python -m venv venv
$ source venv/bin/activate
$ pip install -r requirements.txt
```

With `conda` or `miniconda`:

```sh
$ conda env create -n ci-cd-tutorial-sample-app python=3.8
$ source activate ci-cd-tutorial-sample-app
$ pip install -r requirements.txt
```

Optional: set the `DATABASE_URL` environment variable to a valid SQLAlchemy connection string. Otherwise, a local SQLite database will be created.

Initalize and seed the database:

```sh
$ flask db upgrade
$ python seed.py
```

## Running tests

Run:

```sh
$ python -m unittest discover
```

## Running the application

### Running locally

Run the application using the built-in Flask server:

```sh
$ flask run
```

### Running on a production server

Run the application using `gunicorn`:

```sh
$ pip install -r requirements-server.txt
$ gunicorn app:app
```

To set the listening address and port, run:

```
$ gunicorn app:app -b 0.0.0.0:8000
```

## Running on Docker

Run:

```
$ docker build -t ci-cd-tutorial-sample-app:latest .
$ docker run -d -p 8000:8000 ci-cd-tutorial-sample-app:latest
```

## Deploying to Heroku

Run:

```sh
$ heroku create
$ git push heroku master
$ heroku run flask db upgrade
$ heroku run python seed.py
$ heroku open
```

or use the automated deploy feature:

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy)

For more information about using Python on Heroku, see these Dev Center articles:

 - [Python on Heroku](https://devcenter.heroku.com/categories/python)

## CI/CD Pipeline

This project uses **Azure DevOps Pipelines** for build, test, and deploy. The **source repository** can be either **Azure Repos** or **GitHub**; when using GitHub, native GitHub scanners (CodeQL, secret scanning, Dependabot) are used for code security, and the pipeline runs in Azure DevOps. See [GitHub and Azure DevOps security setup](my_files/GitHub-and-Azure-DevOps-security-setup.md) for the hybrid setup (repo on GitHub, pipeline on Azure DevOps).

### Pipeline Stages

 * **Preparing and Testing** - secret scan (gitleaks), installs dependencies, dependency scan (pip-audit), unit tests with coverage, SonarCloud analysis
 * **Build and Push** - builds Docker image, pushes to ACR, container scan (Trivy), cleanup old images
 * **Deploy to Dev** - deploys to development environment
 * **Deploy to Prod** - deploys to production environment

### SonarCloud Integration

[SonarCloud](https://sonarcloud.io) performs static code analysis on every pull request and merge to master.

Configuration file: `sonar-project.properties`

#### Quality Gate

The pipeline enforces a Quality Gate that checks:

 * Code coverage >= 80% on new code
 * No new bugs or vulnerabilities
 * Code duplication <= 3%

If the Quality Gate fails, the pipeline stops and does not proceed to build/deploy stages.

#### Interpreting Results

 * **Passed** - code meets quality standards, pipeline continues
 * **Failed** - quality issues detected, check SonarCloud dashboard for details

To view detailed analysis:
1. Go to [SonarCloud](https://sonarcloud.io)
2. Navigate to your project
3. Check **Issues** tab for specific problems
4. Check **Measures** tab for metrics

### Versioning

#### Version Format

Images use semantic versioning: `Major.Minor.BuildId`

 * `Major` - manually set, incremented for breaking changes
 * `Minor` - manually set, incremented for new features
 * `BuildId` - auto-incremented by Azure DevOps for each build

Example: `1.0.392`

#### Docker Tags

Each build creates three tags pointing to the same image:

 * `1.0.392` - semantic version (for production/rollback)
 * `392` - build ID (quick reference)
 * `latest` - always points to newest image

#### Configuration

Version variables are stored in Azure DevOps Variable Group `general-config`:

 * `Major` - major version number
 * `Minor` - minor version number
 * `retentionDays` - delete images older than N days
 * `retentionCount` - minimum images to keep

### Azure Container Registry

Docker images are stored in Azure Container Registry (ACR).

 * Registry: `peexcicdwebappacr.azurecr.io`
 * Image: `peex_01_2025`
 * Retention: images older than 10 days are cleaned up, minimum 10 kept

### Deploying a Specific Version

To deploy a specific version manually:

1. Go to Azure DevOps → Pipelines
2. Select the pipeline → Run pipeline
3. In "Deploy specific version" field, enter version (e.g., `1.0.370`)
4. Click Run

If left empty, the pipeline deploys the current build version.

### Rollback Procedure

To rollback to a previous version:

1. Find the target version in ACR (Azure Portal → Container Registry → Repositories)
2. Run pipeline manually with the target version in "Deploy specific version" field
3. Verify deployment in Azure Portal → Web App → Deployment Center

Example: To rollback from `1.0.392` to `1.0.385`:
 * Run pipeline with `deployVersion` = `1.0.385`

### Troubleshooting

#### SonarCloud Quality Gate Failed

1. Check SonarCloud dashboard for specific issues
2. Fix reported bugs, vulnerabilities, or code smells
3. Ensure new code has sufficient test coverage
4. Re-run the pipeline

#### Unit Tests Failing

1. Check pipeline logs for test output
2. Run tests locally: `python -m unittest discover`
3. Verify dependency versions in `requirements.txt`

#### Docker Build Failed

1. Check Dockerfile syntax
2. Verify base image availability
3. Check ACR credentials and permissions

#### Pipeline Stuck or Slow

1. Verify self-hosted agent is online
2. Check agent has required tools (Python, Docker, Java)
3. Review pipeline logs for specific errors

## Security

### Security Features Enabled

| Feature | Tool | What it checks |
|---------|------|----------------|
| Secret scanning | gitleaks (Azure pipeline); when repo is on GitHub also **GitHub secret scanning** (Settings → Security) | Git history and current files for exposed secrets |
| Dependency scanning | pip-audit (Azure pipeline); when repo is on GitHub also **Dependabot** (`.github/dependabot.yml`) | Python packages for known CVEs; Dependabot alerts and PRs |
| Code scanning | SonarCloud (Azure pipeline); when repo is on GitHub also **CodeQL** (`.github/workflows/codeql.yml`) | Static analysis, bugs, vulnerabilities, code smells |
| Container image scanning | Trivy | Docker image (OS packages, app dependencies, misconfigurations) |
| .gitignore | - | Prevents committing sensitive files |

Scans run on every pipeline execution. pip-audit and Trivy use `continueOnError: true` so the pipeline continues even when vulnerabilities are found; findings are visible in logs.

### Vulnerability Management Process

1. **Review** - Check pipeline logs for pip-audit and Trivy output
2. **Triage** - Prioritize by severity (CRITICAL > HIGH > MEDIUM > LOW)
3. **Remediate** - Update vulnerable packages or base image
4. **Verify** - Re-run pipeline; scans should show fewer or no findings

For Python: update `requirements.txt` with fix versions from pip-audit output.
For Docker: update base image in Dockerfile or apply security patches.

### RBAC: Who Can Modify, Run, and Access the Pipeline

Access to the pipeline and its outputs is controlled by **Azure DevOps RBAC**. There is no pipeline definition in this repo that sets these permissions; they are configured in the **Azure DevOps portal** for the project. When the **source repo is on GitHub**, repository RBAC and branch protection are configured in **GitHub** (Settings → Collaborators and teams, Branches); the pipeline itself and its permissions remain in Azure DevOps.

| What is controlled | Where it is configured | Typical roles |
|--------------------|------------------------|---------------|
| **Modify pipeline definitions** (edit `azure-pipelines.yml` and pipeline settings) | **Repos → Security** (for the repo that contains `azure-pipelines.yml`): who can push to branches that change the pipeline. **Pipelines →** select pipeline → **⋮** → **Security**: who has "Edit pipeline" / "Manage permissions". | Contributors can push to branches; only certain users/groups can edit pipeline security or delete the pipeline. |
| **Trigger pipeline executions** (run the pipeline, cancel runs) | **Pipelines →** select pipeline → **⋮** → **Security**. Permissions: "Queue builds", "View builds", "Manage build queue". | Contributors (or Build Service) can queue and view builds; restricted by branch policies (e.g. only `master` runs full deploy). |
| **Access pipeline artifacts** (build outputs, run history, logs) | Same **Pipelines → … → Security** for "View builds" / "View pipeline". **Container images**: access to ACR is separate — **Azure Portal → Container Registry → Access control (IAM)** (e.g. AcrPull, AcrPush per identity). | Viewers can see runs and logs; Contributors can download artifacts; container images are protected by ACR IAM. |

**How to review or change RBAC:**

1. **Azure DevOps** → your project (e.g. `peex_01_2025`).
2. **Repos** → **Project Settings** (bottom-left) → **Repositories** → select repo → **Security** tab: manage who can push/force-push/edit policies (controls who can change the pipeline file).
3. **Pipelines** → select the pipeline → **⋮** (three dots) → **Security**: add users/groups and set **View**, **Edit**, **Queue builds**, **Manage permissions**, etc.
4. **Azure Portal** → **Container registries** → your ACR → **Access control (IAM)**: manage who can pull/push/delete images (pipeline uses a service connection with minimal roles, e.g. AcrPush, Website Contributor on Web App RGs).

This way, only authorized identities can modify the pipeline, trigger runs, or access build artifacts and container images.

### Optional: Azure Key Vault (auditable secrets)

Secrets can be stored in **Azure Key Vault** instead of (or in addition to) Variable Groups. The pipeline then fetches them via the **AzureKeyVault** task; access to secrets is logged in Key Vault for audit.

**To enable:**

1. **Azure Portal** — create a Key Vault (e.g. `peex-01-2025-kv`) in the same subscription as the pipeline.
2. **Key Vault → Secrets** — create secret **SONAR-TOKEN** with your SonarCloud token (same value as in Variable Group `sonarcloud-config`).
3. **Key Vault → Access control (IAM)** — add role assignment: **Key Vault Secrets User** for the pipeline’s service principal (the one used by service connection `peex-cicd`).
4. **Azure DevOps** — In variable group **general-config** or **sonarcloud-config**: add variable **keyVaultName** = `peex-01-2025-kv` (your Key Vault name).

When `keyVaultName` is set, the pipeline runs the "Get secrets from Key Vault" step and uses `SONAR_TOKEN` from Key Vault for SonarCloud; otherwise it uses the Variable Group as before. Key Vault access appears under **Key Vault → Monitoring → Diagnostic settings** (enable logging to Log Analytics or Storage for full audit).

### Secure Development Practices

 * No secrets in code or config; use Variable Groups (or Key Vault) for sensitive values
 * Branch protection and required reviews (configure in Azure DevOps)
 * Regular dependency updates to reduce exposure to known vulnerabilities