# Hostess Examples

Use these as starter patterns. Adjust service names, paths, ports, secrets, and health checks to match the user's repository. More examples and guides are at [docs.hostess.sh](https://docs.hostess.sh).

## Next.js with Postgres

```yaml
version: "1.0"
name: my-app

services:
  web:
    type: nextjs
    build:
      source: .
    env:
      DATABASE_URL: ${database.url}
    depends_on:
      - database

  database:
    type: postgres
    resources: small
```

## Next.js, FastAPI, Postgres, Redis

```yaml
version: "1.0"
name: full-stack-app

services:
  frontend:
    type: nextjs
    build:
      source: ./frontend
    env:
      NEXT_PUBLIC_API_URL: ${api.external_url}
    depends_on:
      - api

  api:
    type: fastapi
    build:
      source: ./backend
    env:
      DATABASE_URL: ${database.url}
      REDIS_URL: ${cache.url}
      JWT_SECRET: ${secret:JWT_SECRET}
    depends_on:
      - database
      - cache

  database:
    type: postgres
    resources: medium
    retention: permanent
    backups: daily

  cache:
    type: redis
    resources: small
```

## Production domains and staging domains

```yaml
version: "1.0"
name: my-app

environments:
  staging:
    domains:
      frontend: [staging.myapp.com]
      api: [api-staging.myapp.com]

services:
  frontend:
    type: nextjs
    build:
      source: ./frontend
    domains:
      - myapp.com
      - www.myapp.com
    env:
      NEXT_PUBLIC_API_URL: ${api.external_url}

  api:
    type: fastapi
    build:
      source: ./backend
    domains:
      - api.myapp.com
```

Service-level `domains` apply to production. Use `environments.<env>.domains` for staging, preview, and other non-production targets.

## Autoscaled API

```yaml
version: "1.0"
name: api-app

services:
  api:
    type: fastapi
    build:
      source: ./api
    resources: medium
    replicas:
      min: 2
      max: 10
      target_cpu: 70
    env:
      DATABASE_URL: ${database.url}
    depends_on:
      - database

  database:
    type: postgres
    resources:
      preset: large
      storage: 100Gi
    retention: permanent
    backups:
      schedule: daily
      retention: 14
```

## Custom public service from image

```yaml
version: "1.0"
name: custom-app

services:
  web:
    type: custom
    image: ghcr.io/acme/public-web:1.2.3
    ports: [8080]
    health:
      http: /health
    env:
      NODE_ENV: production
```

Use `image:` only when the image is public. If the image is private or built from this repo, use `build:` instead.

## Custom worker with fixed replicas

```yaml
version: "1.0"
name: worker-app

services:
  worker:
    type: custom
    build:
      source: ./worker
    command: ["python", "-m", "worker"]
    replicas: 3
    env:
      DATABASE_URL: ${database.url}
      REDIS_URL: ${cache.url}
    depends_on:
      - database
      - cache

  database:
    type: postgres
    retention: permanent

  cache:
    type: redis
```

## Persistent custom service

```yaml
version: "1.0"
name: n8n-like-app

services:
  app:
    type: custom
    image: docker.n8n.io/n8nio/n8n:stable
    ports: [5678]
    persistence:
      mount: /home/node/.n8n
      size: 10Gi
      owner: "1000:1000"
    env:
      DATABASE_URL: ${database.url}
      REDIS_URL: ${cache.url}
      N8N_ENCRYPTION_KEY: ${secret:N8N_ENCRYPTION_KEY}
    depends_on:
      - database
      - cache

  database:
    type: postgres
    retention: permanent
    backups: daily

  cache:
    type: redis
    retention: permanent
```

## Config file mount

```yaml
version: "1.0"
name: gateway-app

services:
  gateway:
    type: custom
    build:
      source: ./gateway
    ports: [8080]
    files:
      - name: gateway-config
        mount: /etc/gateway/config.yml
        source: ./gateway/config.yml
      - name: signing-key
        mount: /etc/gateway/signing.key
        secret: GATEWAY_SIGNING_KEY
        mode: "0400"
```

## Migration job gates API startup

```yaml
version: "1.0"
name: api-with-migrations

services:
  api:
    type: fastapi
    build:
      source: ./api
    env:
      DATABASE_URL: ${database.url}
    depends_on:
      database:
        condition: healthy
      migrate-db:
        condition: completed

  database:
    type: postgres
    retention: permanent
    backups: daily

jobs:
  migrate-db:
    build:
      source: ./api
    command: ["alembic", "upgrade", "head"]
    run: on_deploy
    env:
      DATABASE_URL: ${database.url}
    depends_on:
      database:
        condition: healthy
```

## Manual job

```yaml
version: "1.0"
name: reporting-app

services:
  api:
    type: fastapi
    build:
      source: ./api

jobs:
  rebuild-search:
    build:
      source: ./api
    command: ["python", "-m", "scripts.rebuild_search"]
    run: manual
```

Trigger it:

```bash
hostess jobs run rebuild-search --project reporting-app --env production
```

## Cron job

```yaml
version: "1.0"
name: reports

services:
  api:
    type: fastapi
    build:
      source: ./api

jobs:
  daily-report:
    build:
      source: ./api
    command: ["python", "-m", "reports.daily"]
    run: cron
    schedule: daily
```
