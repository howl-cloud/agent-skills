# Hostess Deploy Workflow

Use this reference for first deploys, Docker Compose migration, preview deploys, production deploys, and CI/CD. Full documentation is at [docs.hostess.sh](https://docs.hostess.sh).

## Authentication and install

Check for the CLI:

```bash
command -v hostess
hostess --version
```

Install:

```bash
curl -fsSL https://hostess.sh/install.sh | sh
```

Update:

```bash
hostess update
```

Interactive login (opens a browser automatically):

```bash
hostess login
```

If the browser does not open or you are in a headless environment, use `--no-browser` to print the login URL instead:

```bash
hostess login --no-browser
```

For CI/CD, use `HOSTESS_TOKEN` from a Hostess personal access token — do not run `hostess login` in CI.

## First deploy decision flow

1. If `hostess.yml` exists:
   - Inspect it.
   - Check whether it matches the repo layout and runtime.
   - Run `hostess validate`.
2. If no `hostess.yml` exists but Docker Compose exists:
   - Run `hostess init`.
   - Review the generated config.
   - Replace raw Postgres/Redis images with managed `type: postgres` or `type: redis` if not already converted.
   - Move sensitive env values into Hostess secrets.
   - Run `hostess validate`.
3. If neither exists:
   - Infer services from repo structure.
   - Create a minimal `hostess.yml`.
   - Add Dockerfiles or use public images where appropriate.
   - Run `hostess validate`.
4. Deploy only after validation succeeds.

## Docker Compose migration

Typical mapping:

| Docker Compose                     | Hostess                                               |
| ---------------------------------- | ----------------------------------------------------- |
| `services.<name>.build.context`    | `services.<name>.build.source`                        |
| `services.<name>.build.dockerfile` | `services.<name>.build.dockerfile`                    |
| `environment`                      | `env`                                                 |
| `image: postgres:*`                | `type: postgres`                                      |
| `image: redis:*`                   | `type: redis`                                         |
| service-to-service hostnames       | magic variables like `${database.url}`                |
| named database volumes             | `retention: permanent` and database resources         |
| bind-mounted config files          | `files.source`                                        |
| durable custom app data            | `persistence`                                         |
| published ports                    | `ports`, usually omitted for Next.js/FastAPI defaults |

After `hostess init`, make production choices explicit:

- Add `retention: permanent` for data that must survive redeploys.
- Add `backups: daily` for important Postgres/Redis data.
- Replace hardcoded connection URLs with magic variables.
- Replace hardcoded API keys or passwords with `${secret:NAME}`.
- Replace local browser URLs such as `http://localhost:8000` with `${api.external_url}` where browser clients need a public URL.

## Validation

Always run:

```bash
hostess validate
```

Validation checks required fields, service names, supported service types, build/image rules, dependencies, resources, ports, domains, and magic variable references.

Fix validation errors in `hostess.yml`; do not deploy around them.

## Deploy commands

Local interactive deploy:

```bash
hostess deploy
```

Target an environment explicitly:

```bash
hostess deploy --env production
hostess deploy --env preview
hostess deploy --env staging
```

Deploy selected services:

```bash
hostess deploy -s api
hostess deploy -s api -s worker --env production
```

Use a full deploy when `hostess.yml`, database configuration, Redis configuration, dependencies, domains, backups, or shared service wiring changed.

Use per-service deploys for code-only changes to one or more app services.

Remote config deploys:

```bash
hostess deploy --repo org/repo
hostess deploy --url https://example.com/hostess.yml
```

Remote sources are best for configs that use public `image:` services. For services with `build:`, clone the repo and deploy from the checkout.

## Environment targeting

Use `--env` explicitly in CI and scripts.

New projects normally have `production` and `preview` environments. If the user targets a custom environment such as `staging` or `qa`, make sure it exists before deploying.

Preview deployment automation belongs in CI branch or pull request filters, not inside `hostess.yml`.

## Secrets before deploy

Create secrets referenced by config before deploying:

```bash
hostess secrets add JWT_SECRET --value "..."
hostess secrets add STRIPE_KEY --envs production --value "sk_live_..."
hostess secrets add STRIPE_KEY --envs preview --value "sk_test_..."
```

To sync from a local env file:

```bash
hostess secrets sync push --env production --file .env.production
```

Do not print secret values back to the user.

## GitHub Actions

Basic production deploy:

```yaml
name: Deploy to Hostess

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: howl-cloud/setup-hostess@v1
      - run: hostess validate
      - run: hostess deploy --env production --no-interactive
        env:
          HOSTESS_TOKEN: ${{ secrets.HOSTESS_TOKEN }}
```

Preview deploy for pull requests:

```yaml
name: Deploy Preview

on:
  pull_request:
    branches:
      - main

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: howl-cloud/setup-hostess@v1
      - run: hostess validate
      - run: hostess deploy --env preview --no-interactive
        env:
          HOSTESS_TOKEN: ${{ secrets.HOSTESS_TOKEN }}
```

Per-service deploys in CI:

```yaml
- run: hostess deploy -s api --env production --no-interactive
  env:
    HOSTESS_TOKEN: ${{ secrets.HOSTESS_TOKEN }}
```

If CI cannot prompt and the project does not exist, create the project first with `hostess projects create <name>` or do the first deploy interactively.

## Post-deploy verification

After deploy, report:

- Deployment ID.
- Environment.
- Service URLs.
- Any service still building or unhealthy.
- Next command if action is needed.

Useful checks:

```bash
hostess inspect --env production
hostess ps --env production
hostess logs api --tail 100 --env production
```

## Live build log streaming

During `hostess deploy`, the TUI streams live build phase labels for each service as it builds:

```
→ Preparing
→ Pulling base image
→ Restoring cache
→ Installing
→ Building
→ Pushing image
✓ built
```

For `source:` (Railpack) services this includes package install output and build step details. For cached builds the phases flash by quickly.

The same log stream is available in the Studio under **Deployments → [service] → Build Logs**. Build logs are also written to the deployment record for post-deploy inspection.

## Troubleshooting deploys

Common checks:

1. `hostess validate` for config errors.
2. `hostess inspect --env <env>` for deployment status.
3. `hostess ps --env <env>` for service status and URLs.
4. `hostess logs <service> --tail 100 --env <env>` for runtime failures.
5. Confirm all `${secret:NAME}` values exist in the target environment.
6. Confirm apps listen on the expected port and health path.
7. Confirm Docker build context includes files needed by the Dockerfile.
