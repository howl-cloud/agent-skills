# Hostess Operations

Use this reference for status checks, logs, local connections, backups, restores, jobs, and operational troubleshooting.

## Deployments

List deployments:

```bash
hostess deployments list
hostess ls
hostess ls --env production
hostess ls --format json
```

Inspect the latest deployment:

```bash
hostess deployments inspect
hostess inspect
hostess inspect --env staging
```

Inspect a specific deployment:

```bash
hostess inspect dep_abc123
```

Remove a deployment only after explicit confirmation:

```bash
hostess deployments remove dep_abc123
hostess rm dep_abc123
```

Removing a deployment removes associated running resources. Data for database services with `retention: permanent` is preserved; data with `retention: ephemeral` can be deleted.

## Services and logs

List services:

```bash
hostess services list
hostess ps
hostess ps --env staging
hostess ps --detailed
```

View logs:

```bash
hostess services logs api
hostess logs api
hostess logs api --tail 50
hostess logs api --since 1h
hostess logs api --follow --tail 20
hostess logs api --timestamps
hostess logs api --env staging --follow
```

When troubleshooting, start with the failing service and the most recent 100-200 log lines. Use `--follow` only when watching an active deploy or reproducing a failure.

## Connect locally

Use `hostess connect` for private database/cache access, port forwarding, and shell sessions.

Postgres:

```bash
hostess connect database
hostess connect database --local-port 5433
```

Redis:

```bash
hostess connect cache
hostess connect cache --local-port 6379
```

Any service:

```bash
hostess connect api
hostess connect api --port 8000
```

Scoped database group:

```bash
hostess connect database --database analytics
```

Specific environment:

```bash
hostess connect database --env production
hostess connect database --env staging --local-port 5433
```

Interactive shell:

```bash
hostess connect shell api
hostess connect shell api --command /bin/bash
hostess connect shell api --command /bin/sh
```

One-off command:

```bash
hostess connect shell api --command "python --version"
hostess connect shell web --command "python manage.py createsuperuser"
hostess connect shell api --command "env | sort"
```

Be careful with production database access. Prefer staging or preview for development, and avoid modifying production data unless the user explicitly intends it.

## Secrets

Add or update:

```bash
hostess secrets add JWT_SECRET --value "..."
hostess secrets edit JWT_SECRET --value "..."
hostess secrets add STRIPE_KEY --envs production --value "sk_live_..."
hostess secrets add STRIPE_KEY --envs preview --value "sk_test_..."
```

List or compare:

```bash
hostess secrets get --envs production
hostess secrets diff production staging
hostess secrets diff production staging --format json
```

Sync:

```bash
hostess secrets sync push --env production --file .env.production
hostess secrets sync pull --env staging --file .env.staging
```

After editing a secret, redeploy affected services for the new value to take effect.

## Domains

Add domains through config when they should be versioned with the app. Use CLI commands for operational changes:

```bash
hostess domains add --service frontend --env production myapp.com
hostess domains add --service frontend --env staging staging.myapp.com
hostess domains list
hostess domains remove myapp.com
```

DNS target:

```txt
ingress.hostess.run
```

Use CNAME for subdomains. For apex domains, use the provider's hostname-targeting option. With Cloudflare, use DNS-only mode.

## Backups and restores

List backups:

```bash
hostess backups list
hostess backups list --service database
hostess backups list --env staging
```

Create manual backup:

```bash
hostess backups create database
hostess backups create database --env staging
```

Restore only after explicit confirmation because it overwrites current data:

```bash
hostess backups restore database
hostess backups restore database --backup database-20250326-020003.sql.gz
```

Hostess creates a safety backup before restore and prints the restore job and safety backup names. Verify afterward:

```bash
hostess backups list --service database
hostess jobs list --project my-app --env production
```

Delete backup only after explicit confirmation:

```bash
hostess backups delete database --backup database-20250325-143022.sql.gz
```

## Jobs

List and inspect:

```bash
hostess jobs list --project my-app --env production
hostess jobs inspect migrate-db --project my-app --env production
hostess jobs runs daily-report --project my-app --env production
```

Run a manual job:

```bash
hostess jobs run rebuild-search --project my-app --env production
```

Override values for one run:

```bash
hostess jobs run daily-report \
  --project my-app \
  --env production \
  --set-env REPORT_DATE=2026-05-12
```

Force-run a non-manual job only when the user explicitly intends it:

```bash
hostess jobs run migrate-db --project my-app --env production --force
```

Re-run a `once` job on the next deploy:

```bash
hostess jobs run setup --project my-app --env production --next-deploy
```

## Troubleshooting patterns

Build failure:

1. Check Dockerfile path and build source path.
2. Confirm files copied by Dockerfile are inside the build context.
3. Confirm `image:` is public if used.
4. Use `build:` for app source and private registry needs.

Service starts then fails:

1. `hostess logs <service> --tail 200 --env <env>`.
2. Check env values, missing secrets, command/entrypoint, health path, and port.
3. Confirm app listens on `0.0.0.0`, not only localhost, inside the container.

Service cannot reach database/cache:

1. Use `${database.url}` or `${cache.url}` in the app service env.
2. Add `depends_on` if startup ordering matters.
3. Confirm target service name matches the magic variable prefix.
4. Use `hostess connect <service>` to inspect data manually if needed.

Custom domain not working:

1. Confirm domain is configured for the target service and environment.
2. Confirm DNS points to `ingress.hostess.run`.
3. For Cloudflare, confirm DNS-only mode.
4. Wait for DNS propagation, then re-check with `hostess domains list`.
