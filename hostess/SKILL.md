---
name: hostess
description: Deploy and operate applications on Hostess from a user's app repository. Use when the user mentions Hostess, hostess.yml, deploying an app, migrating from Docker Compose, configuring services, managed Postgres or Redis, secrets, environments, custom domains, backups, service logs, local service connections, CI/CD with howl-cloud/setup-hostess, or troubleshooting a Hostess deployment.
---

# Use Hostess

Hostess deploys an application stack from a `hostess.yml` file in the user's repository. The skill's job is to help the agent create, validate, deploy, and operate that stack without guessing Hostess behavior from the web.

## Resource model

- **Project**: One application stack, normally named by `name:` in `hostess.yml`.
- **Environment**: A deployment target such as `production`, `preview`, `staging`, or `qa`.
- **Service**: A long-running component under `services:` such as a web app, API, database, cache, worker, or container.
- **Job**: A finite workload under top-level `jobs:` that runs on deploy, once, manually, or on a schedule.
- **Deployment**: A specific release of the project into an environment.
- **Secret**: An encrypted value referenced from config with `${secret:NAME}`.

## First actions

1. Inspect the user's repo before writing config:
   - Look for `hostess.yml`, `docker-compose.yml`, `compose.yml`, framework files, Dockerfiles, package manifests, migrations, workers, and `.env.example`.
   - Identify service names, build roots, ports, health endpoints, data stores, secrets, and environment targets.
2. Check the CLI path before relying on it:
   ```bash
   command -v hostess
   hostess --version
   ```
   If missing, tell the user to install it:
   ```bash
   curl -fsSL https://hostess.sh/install.sh | sh
   ```
3. Prefer `hostess init` when the repo has Docker Compose:
   ```bash
   hostess init
   ```
   Then review and edit the generated `hostess.yml`.
4. If there is no Compose file, create or update `hostess.yml` directly.
5. Always validate before deploying:
   ```bash
   hostess validate
   ```

## Routing

Load only the reference needed for the user's intent.

| Intent | Reference | Use for |
|---|---|---|
| Create or edit `hostess.yml` | [configuration.md](references/configuration.md) | Service types, fields, magic variables, secrets, domains, resources, replicas, backups, jobs |
| Deploy, migrate, or set up CI | [deploy.md](references/deploy.md) | `hostess init`, validation, deploy commands, preview/production targeting, GitHub Actions |
| Check status, logs, shell, database access, backups, or jobs | [operate.md](references/operate.md) | Inspect deployments, list services, stream logs, connect locally, run jobs, restore backups |
| Need a starter config | [examples.md](references/examples.md) | Copyable `hostess.yml` patterns for common stacks |

If a request spans two areas, load both references and give one unified answer. Do not make the user invoke separate steps.

## Execution rules

1. Keep Hostess config portable: paths are relative to `hostess.yml`; avoid local absolute paths.
2. Prefer `build:` for app code and private registry needs. Use `image:` only for public images that can be pulled without credentials.
3. Use managed `type: postgres` and `type: redis` instead of raw database images when the service is Postgres or Redis.
4. Use magic variables for service wiring: `${database.url}`, `${cache.url}`, `${api.external_url}`.
5. Put sensitive values in Hostess secrets and reference them as `${secret:NAME}`. Never commit real secret values into `hostess.yml`.
6. Pass `--env` explicitly for scripts and CI. For local interactive deploys, ask the user which environment if target matters.
7. Confirm before destructive actions such as deleting projects/deployments, deleting secrets, deleting backups, or restoring over current data.
8. After mutations, verify with a read command such as `hostess inspect`, `hostess ps`, `hostess logs`, or `hostess backups list`.

## Response format

For operational work, answer with:

1. What changed or what was run.
2. The result: validation status, deployment ID, service URLs, or the relevant error.
3. The next action, if any.

Keep user-facing output concise. Include commands when the user needs to run them or when command evidence explains a failure.
