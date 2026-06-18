# Hostess Configuration

Use this reference when creating or editing `hostess.yml`. Full schema documentation is available at [docs.hostess.sh](https://docs.hostess.sh) — the index map is at https://docs.hostess.sh/llms.txt.

## Minimal shape

```yaml
version: "1.0"
name: my-app

services:
  web:
    type: nextjs
    build:
      source: .
```

Only `version` and `services` are required. `name`, `description`, `environments`, and top-level `jobs` are optional.

## Services

Every service lives under `services.<name>`. Service names must be lowercase DNS-safe names: lowercase letters, digits, and hyphens; start with a letter; end with a letter or digit; 2-63 characters.

Supported service types:

| Type       | Use for             | Defaults                        |
| ---------- | ------------------- | ------------------------------- |
| `nextjs`   | Next.js apps        | Port 3000, health `GET /`       |
| `fastapi`  | FastAPI apps        | Port 8000, health `GET /health` |
| `postgres` | Managed PostgreSQL  | Port 5432, managed credentials  |
| `redis`    | Managed Redis       | Port 6379, managed credentials  |
| `custom`   | Any other container | Set ports and health explicitly |

`type` is optional but recommended. When omitted, the service behaves as `custom` and all behavior must be configured explicitly.

Non-database services need exactly one of `build`, `image`, or `source`:

```yaml
build:
  source: ./path
  dockerfile: Dockerfile
  args:
    NODE_ENV: production
```

```yaml
image: nginx:1.25
```

```yaml
source: ./path
```

`source` is the zero-config path: Hostess builds the service using [Railpack](https://railpack.com) — no Dockerfile required. Railpack auto-detects the language and framework (Node, Python, Go, etc.). Use it when you have no Dockerfile and want the shortest path to a working image. `source` requires the BuildKit backend.

Use `image` only for public images that can be pulled without credentials. For private registries, prefer `build` from source so deploy-time auth stays in the build path. Build args are literal strings; secret references like `${secret:NAME}` are not resolved inside `build.args`.

The three keys are mutually exclusive — set exactly one.

## Environment variables

Service env values can be literals, magic variables, or secret references:

```yaml
env:
  LOG_LEVEL: info
  DATABASE_URL: ${database.url}
  REDIS_URL: ${cache.url}
  JWT_SECRET: ${secret:JWT_SECRET}
```

Use literals only for non-sensitive values. Use `${secret:NAME}` for keys, passwords, tokens, signing secrets, and provider credentials.

To load variables from a file, use `env_file`:

```yaml
env_file:
  - .env
  - .env.production
```

Files are read in order; `env` values take precedence over file values.

## Magic variables

Magic variables resolve at deploy time and wire services together automatically.

Postgres:

| Variable                    | Meaning                          |
| --------------------------- | -------------------------------- |
| `${database.url}`           | Internal Postgres connection URL |
| `${database.external_url}`  | External Postgres connection URL |
| `${database.host}`          | Internal host                    |
| `${database.external_host}` | External host                    |
| `${database.port}`          | Port, usually 5432               |
| `${database.user}`          | Username                         |
| `${database.database}`      | Database name                    |
| `${database.password}`      | Managed password                 |

Redis:

| Variable                 | Meaning            |
| ------------------------ | ------------------ |
| `${cache.url}`           | Internal Redis URL |
| `${cache.external_url}`  | External Redis URL |
| `${cache.host}`          | Internal host      |
| `${cache.external_host}` | External host      |
| `${cache.port}`          | Port, usually 6379 |
| `${cache.password}`      | Managed password   |

HTTP services:

| Variable               | Meaning                        |
| ---------------------- | ------------------------------ |
| `${api.url}`           | Internal service URL           |
| `${api.external_url}`  | Public generated or custom URL |
| `${api.host}`          | Internal host                  |
| `${api.external_host}` | Public host                    |
| `${api.port}`          | Port                           |

Use internal URLs for service-to-service calls from the server. Use external URLs for browser-facing values, webhooks, or third-party callbacks.

## Dependencies

Use `depends_on` when startup order or job gating matters:

```yaml
services:
  api:
    type: fastapi
    build:
      source: ./api
    depends_on:
      - database
    env:
      DATABASE_URL: ${database.url}

  database:
    type: postgres
```

Object form supports explicit conditions:

```yaml
depends_on:
  database:
    condition: healthy
  migrate-db:
    condition: completed
```

Compact mapping form is also valid:

```yaml
depends_on:
  database: healthy
  migrate-db: completed
```

Service dependencies use `healthy`. Job dependencies use `completed`. Avoid dependency cycles and self-references.

## Ports and health

Default ports:

| Type      | Default port |
| --------- | ------------ |
| `nextjs`  | 3000         |
| `fastapi` | 8000         |
| `custom`  | None         |

Set `ports` when a service listens on a different port or exposes multiple ports:

```yaml
ports: [8080]
```

Default health checks:

| Type       | Default health           |
| ---------- | ------------------------ |
| `nextjs`   | `GET /`                  |
| `fastapi`  | `GET /health`            |
| `postgres` | Postgres readiness check |
| `redis`    | Redis ping               |
| `custom`   | None                     |

For custom services, configure health when possible:

```yaml
health:
  http: /health
  interval: 10s
  timeout: 5s
  retries: 3
```

or:

```yaml
health:
  command: "pgrep -f worker"
```

## Resources

Presets:

| Preset   |       CPU | Memory | Database storage |
| -------- | --------: | -----: | ---------------: |
| `small`  | 0.5 cores |  512Mi |             10Gi |
| `medium` |    1 core |    1Gi |             20Gi |
| `large`  |   2 cores |    2Gi |             50Gi |
| `xlarge` |   4 cores |    4Gi |            100Gi |

Default is `medium`.

For databases, override storage:

```yaml
resources:
  preset: large
  storage: 200Gi
```

Do not set `storage` on non-database services.

## Replicas and autoscaling

Default is one replica with no autoscaling.

Fixed replicas:

```yaml
replicas: 3
```

Autoscaling:

```yaml
replicas:
  min: 2
  max: 10
  target_cpu: 70
```

Rules:

- Fixed `replicas` and autoscaling `min` must be at least 1.
- Scale-to-zero is not supported in `hostess.yml`.
- If `target_cpu` is omitted, it defaults to 70.
- Use fixed replicas for workers with predictable load.
- Use autoscaling for web frontends and APIs with variable traffic.

## Retention and persistence

Use `retention` for `postgres`, `redis`, and stateful `custom` services:

```yaml
retention: permanent
```

`permanent` keeps data across redeploys. `ephemeral` means data can be discarded with the environment. The default is `permanent` for production environments and `ephemeral` for preview environments.

For custom services needing durable files, use `persistence`:

```yaml
persistence:
  mount: /home/node/.n8n
  size: 10Gi
  owner: "1000:1000"
```

Multiple mounts:

```yaml
persistence:
  - name: data
    mount: /var/lib/app
    size: 20Gi
  - name: uploads
    mount: /app/uploads
    size: 100Gi
```

Do not use `persistence` on non-custom services.

## Files

Use `files` for read-only config files or secret-backed files:

```yaml
files:
  - name: gateway-config
    mount: /etc/gateway/config.yml
    source: ./gateway/config.yml # project-relative file path
  - name: kong-config
    mount: /usr/local/kong/kong.yml
    content: | # inline content
      _format_version: "3.0"
      services: []
  - name: signing-key
    mount: /etc/gateway/signing.key
    secret: GATEWAY_SIGNING_KEY # Hostess secret key
    mode: "0400"
```

Exactly one of `source`, `content`, or `secret` must be set. Mount paths must be absolute file paths. Use `secret` for private keys, certificates, and other sensitive values.

## Insights and logs

Insights (metrics and distributed traces) are **on by default** for all services. The `insights` and `logs` keys exist only to turn behavior off or adjust it.

```yaml
services:
  api:
    type: fastapi
    source: ./api
    insights: disabled # disables server metrics + traces + browser analytics

  web:
    type: nextjs
    source: ./web
    logs: plain # structured (JSON) is the default for app services
    insights:
      browser: disabled # turn off browser analytics only; keep server metrics
```

`insights` fields:

| Key                | Values                  | Meaning                                                                             |
| ------------------ | ----------------------- | ----------------------------------------------------------------------------------- |
| `insights`         | `enabled` \| `disabled` | Shorthand to enable/disable all insights for this service                           |
| `insights.server`  | `enabled` \| `disabled` | OTel env injection and server-side metrics/trace collection                         |
| `insights.browser` | `enabled` \| `disabled` | Browser analytics ingest on this service's ingresses                                |
| `logs`             | `structured` \| `plain` | Log format — `structured` (JSON) is default for app services; `plain` for databases |

Omitting `insights` entirely is equivalent to `insights: enabled`. Ingress-tier traffic metrics (request rate, latency) are always collected regardless of `insights.server`.

The former `observability` key is removed. Migrate by renaming `observability.metrics` → `insights.server`, `observability.logs` → top-level `logs`, and dropping `observability.traces` (traces now follow `insights.server`).

## Domains

Production shorthand:

```yaml
services:
  frontend:
    type: nextjs
    build:
      source: ./frontend
    domains:
      - myapp.com
      - www.myapp.com
```

Per-environment domains:

```yaml
environments:
  production:
    domains:
      frontend: [myapp.com, www.myapp.com]
      api: [api.myapp.com]
  staging:
    domains:
      frontend: [staging.myapp.com]

services:
  frontend:
    type: nextjs
    build:
      source: ./frontend
```

Environment-level domains override service-level domains for that environment. Database services cannot have custom domains.

DNS target:

```txt
CNAME app.example.com -> ingress.hostess.run
```

For apex domains, use the provider's CNAME flattening, ALIAS, or ANAME option if needed. With Cloudflare, use DNS-only mode for Hostess-managed TLS.

## Backups

For Postgres and Redis:

```yaml
backups: daily
```

Full form:

```yaml
backups:
  schedule: daily
  retention: 14
```

Schedules: `daily`, `weekly`, `biweekly`, `monthly`, or a 5-field cron expression. In Hostess v1, `biweekly` is accepted but resolves like weekly.

For custom services:

```yaml
backups:
  schedule: daily
  retention: 7
  command: ["mongodump", "--archive=/backup/dump.gz", "--gzip"]
  restore_command:
    ["mongorestore", "--drop", "--archive=/backup/dump.gz", "--gzip"]
```

Custom backup commands must write to `/backup/`.

## Jobs

Use top-level `jobs:` for finite workloads:

```yaml
jobs:
  migrate-db:
    build:
      source: ./api
    command: ["alembic", "upgrade", "head"]
    run: on_deploy
    depends_on:
      database:
        condition: healthy

  daily-report:
    image: ghcr.io/acme/reports:latest
    command: ["python", "-m", "reports.daily"]
    run: cron
    schedule: daily
```

Job fields:

| Field              | Description                                                                                 |
| ------------------ | ------------------------------------------------------------------------------------------- |
| `build` or `image` | Container source (one required)                                                             |
| `command`          | Command to run (required)                                                                   |
| `run`              | `on_deploy` (default), `once`, `manual`, `cron`, `lifecycle`                                |
| `schedule`         | Required when `run: cron`. Preset (`daily`, `weekly`, `monthly`) or 5-field cron expression |
| `env` / `env_file` | Env vars; same semantics as services. Magic variables supported                             |
| `files`            | File mounts; same syntax as service `files:`                                                |
| `depends_on`       | Service or job names with `healthy` / `completed` conditions                                |
| `resources`        | Same shape as service resources                                                             |
| `timeout`          | Max run time, default `5m`                                                                  |
| `retries`          | Retry count on failure, default `1`                                                         |

The `lifecycle` run mode is used when a service's `lifecycle.*.job:` field references this job by name, letting one job definition be shared across services. Use `on_deploy`, `once`, `manual`, or `cron` for directly scheduled or triggered jobs.

Use jobs for migrations, bootstrap tasks, reports, cleanup tasks, and manual operations that should not be long-running services.
