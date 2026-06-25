# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Infrastructure-only repository for the HandsUp project's Kafka. No application code — only Docker Compose configurations for running Apache Kafka in KRaft mode.

## Commands

```bash
# Dev (local/dev EC2): start broker + kafka-ui
docker compose -f docker-compose.dev.yml --env-file .env.dev up -d

# Dev: stop
docker compose -f docker-compose.dev.yml --env-file .env.dev down

# Dev: stop and delete all data
docker compose -f docker-compose.dev.yml --env-file .env.dev down -v

# Check broker health
docker inspect --format='{{.State.Health.Status}}' hands-up-kafka
```

## Architecture

- **Image**: `apache/kafka` in KRaft mode (no Zookeeper)
- **Environment separation**: `docker-compose.dev.yml` (dev) / `docker-compose.prod.yml` (prod), each with its own `.env.*` file
- **Listeners (both envs)**: CONTROLLER(29093), INTERNAL(9092), EXTERNAL(9094 → host:KAFKA_EXTERNAL_PORT)
- **Dev extras**: `provectuslabs/kafka-ui` at `0.0.0.0:${KAFKA_UI_PORT}` (public access)
- **Prod extras**: `provectuslabs/kafka-ui` at 127.0.0.1:8080 (SSH tunnel only), memory limits (768M container / 512M heap), `restart: unless-stopped`, health-check verification in CD

## Deployment

- `dev` branch push → `.github/workflows/cd-dev.yml` → dev EC2
- `main` branch push → `.github/workflows/cd-prod.yml` → prod EC2

Both pipelines SCP the compose file + env file to EC2 and run `docker compose up -d`. Each includes a health-check verification step (polls for up to 120s).

GitHub Secret `ENV_DEV_CONTENT` / `ENV_PROD_CONTENT` contains the full env file contents. The `CLUSTER_ID` per environment must never change after first deployment — it is baked into KRaft metadata.

## Key Constraints

- `CLUSTER_ID` is fixed per environment. Changing it makes existing volume data inaccessible.
- `KAFKA_LOG_DIRS` is explicitly set to `/var/lib/kafka/data` (not the default `/tmp/kraft-combined-logs`) to survive container restarts with volume mounts.
- Environment files (`.env.dev`, `.env.prod`) are gitignored. Only `.example` templates are committed.
- `KAFKA_ADVERTISED_HOST`: set to `localhost` for local dev, EC2 IP for dev/prod servers.
