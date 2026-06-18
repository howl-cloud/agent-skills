# Hostess Skills

Agent skills for [Hostess](https://hostess.sh), following the [Agent Skills](https://agentskills.io/) format.

Skills are packaged instructions that extend AI coding agent capabilities. This repository contains the `hostess` skill, which helps agents create, deploy, and operate applications on Hostess — directly from the user's repository.

## Available Skills

### hostess

Deploy and operate applications on Hostess. The skill helps agents write `hostess.yml` configs, run deploys, manage secrets, check service status, stream logs, connect to databases, and set up CI/CD.

**Use when:**

- Creating or editing a `hostess.yml`
- Deploying an app or migrating from Docker Compose
- Configuring managed Postgres or Redis
- Setting up GitHub Actions for preview and production deploys
- Checking service status, streaming logs, or connecting to a database locally
- Managing secrets, custom domains, backups, or jobs

**References loaded on demand:**

- `configuration.md` — full `hostess.yml` schema: service types, magic variables, resources, replicas, backups, jobs
- `deploy.md` — deploy workflow, `hostess init`, validation, CI/CD setup
- `operate.md` — status, logs, `hostess connect`, secrets, domains, backups, jobs
- `examples.md` — copyable starter configs for common stacks

## Installation

### Skill

```bash
npx skills add howl-cloud/agent-skills --skill hostess --agent claude-code codex cursor
```

Add `-g` to install the skill globally.

#### All agents

```bash
npx skills add howl-cloud/agent-skills --skill hostess
```

### Plugin

<!-- Plugin install snippets

## Claude Code

```
/plugin marketplace add howl-cloud/agent-skills
/plugin install hostess@agent-skills
```

## Codex

`codex plugin marketplace add howl-cloud/agent-skills`

## Cursor

1. Open Settings → Plugins.
2. Paste https://github.com/howl-cloud/agent-skills in the search field.
3. Select the Hostess plugin and click Add to Cursor.
-->

## Links

- Get started deploying at [hostess.sh](https://hostess.sh)
- Documentation: [docs.hostess.sh](https://docs.hostess.sh)

## Skill Structure

```
hostess/
  SKILL.md              # Skill definition and routing instructions
  references/
    configuration.md    # hostess.yml schema reference
    deploy.md           # Deploy workflow and CI/CD
    operate.md          # Operational commands
    examples.md         # Starter config patterns
```
