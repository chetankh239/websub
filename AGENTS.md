# AGENTS.md

This file provides guidance to AI agents when working with code in this repository.

## Project Overview

MOSIP WebSub is a Kafka-backed publish-subscribe hub implementing the [W3C WebSub](https://www.w3.org/TR/websub/) specification. It is part of the [MOSIP](https://docs.mosip.io/1.2.0) identity platform, which manages identity lifecycle events (registration, authentication, updates) that flow between services via this hub. The repo contains two Ballerina services and one Java/Maven utility library.

## Build Commands

### Hub Service
```bash
cd hub
bal build                          # compile
bal run target/bin/hub.jar         # run locally (requires Config.toml)
bal test                           # run tests
```

### Consolidator Service
```bash
cd consolidator
bal build
bal run target/bin/consolidator.jar
bal test
```

### kafka-admin-client (Java/Maven)
```bash
cd kafka-admin-client
mvn clean install                  # build and run unit tests
mvn test -Dtest=ClassName#method   # run a single test
mvn test -DskipTests               # skip tests
```

### Docker
Both services use multi-stage Dockerfiles. The base image is `mosipid/openjdk-17-jre:17.0.11`. The build artifact (`.jar`) must exist before building the Docker image.

```bash
docker build -t websub-service hub/
docker build -t consolidator-websub-service consolidator/
```

### Kubernetes / Helm
```bash
cd deploy
./install.sh [kubeconfig]          # install both charts into `websub` namespace
./restart.sh [kubeconfig]          # rolling restart
./delete.sh [kubeconfig]           # teardown
```
Chart version is `0.0.1-develop` on the develop branch.

## Architecture

### Two-Service Design

```
MOSIP services
     │  publish / subscribe / register HTTP calls
     ▼
┌─────────────┐   websub events (Kafka)   ┌──────────────────────┐
│  Hub        │ ────────────────────────▶ │  Consolidator        │
│  :9191/hub  │                           │  :9192/consolidator  │
└─────────────┘                           └──────────────────────┘
      │                                            │
      │         Kafka (persistence + messaging)    │
      └────────────────────────────────────────────┘
```

**Hub** (`hub/`) — the WebSub endpoint. Handles topic registration/deregistration, subscriptions/unsubscriptions, and content distribution to subscribers. Each subscriber gets its own Kafka consumer goroutine (`@strand {thread: "any"}`). The hub does NOT own the consolidated state — it reads it from Kafka topics written by the Consolidator.

**Consolidator** (`consolidator/`) — stateful aggregator. Consumes raw WebSub events from the hub, maintains the single source of truth for registered topics and active subscribers, and writes consolidated snapshots to Kafka. Runs on port 9192. The hub's health check depends on the Consolidator being healthy.

### Kafka Topic Layout

| Topic | Owner | Purpose |
|-------|-------|---------|
| `registered-websub-topics` | Hub writes | Raw topic registration events |
| `consolidated-websub-topics` | Consolidator writes | Canonical topic list (snapshot) |
| `registered-websub-subscribers` | Hub writes | Raw subscription events |
| `consolidated-websub-subscribers` | Consolidator writes | Canonical subscriber list (snapshot) |
| `<topic-name>` (dynamic) | Hub writes | Actual published messages per topic |

The four meta-topics must exist before either service starts. `kafka-admin-client` is the Java utility that creates/manages these Kafka topics.

### State Management Pattern

Both services hold an in-memory `isolated map<>` cache guarded by Ballerina `lock` blocks. On startup, state is bootstrapped by polling the `consolidated-*` Kafka topics. Continuous background strands (`syncRegisteredTopicsCache`, `syncSubscribersCache`) keep the hub's cache updated from Consolidator snapshots. Consolidator persists state by producing a full JSON snapshot to Kafka on every mutation (not event-sourced deltas).

### Security

Authorization is enforced in `hub/modules/security/security.bal`. The hub validates an `Authorization` cookie against the MOSIP IdP (`MOSIP_AUTH_BASE_URL`). Role naming convention for topics uses the pattern `<PREFIX>_<TOPIC>[_GENERAL | _ALL_INDIVIDUAL | _INDIVIDUAL]`. Hub secrets for subscribers can be AES-GCM encrypted (prefix `cipher{`, suffix `}`); the hub decrypts them at subscriber start using `HUB_SECRET_ENCRYPTION_KEY`.

## Configuration

All config is Ballerina `configurable` variables loaded from `Config.toml` at runtime. Key variables for the Hub:

| Variable | Default | Notes |
|----------|---------|-------|
| `KAFKA_BOOTSTRAP_NODE` | `localhost:9092` | |
| `HUB_PORT` | `9191` | |
| `SECURITY_ON` | `true` | Set `false` for local dev without IdP |
| `SERVER_ID` | `server-1` | Must be unique per hub instance |
| `MOSIP_AUTH_BASE_URL` | — | IdP base URL |
| `HUB_SECRET_ENCRYPTION_KEY` | — | 32-byte base64-encoded AES key |
| `META_TOPICS` | (4 topics) | Must pre-exist in Kafka |

Consolidator uses a subset of the same variables (no security config, adds `CONSOLIDATOR_PORT=9192`).

## CI / CD

- `.github/workflows/push-trigger.yml` — builds both services and pushes Docker images; uses `mosip/kattu/.github/workflows/docker-build.yml@master-java21`.
- `.github/workflows/chart-lint-publish.yml` — lints and publishes Helm charts.
- Helm charts live in `helm/websub/` and `helm/websub-consolidator/`. Image registry for dev: `mosipqa`; tag: `develop`.

## Versioning

- Ballerina packages (`hub/Ballerina.toml`, `consolidator/Ballerina.toml`) and `kafka-admin-client/pom.xml` share a single version string.
- Current develop version: `1.4.0-SNAPSHOT`.
- The Ballerina `path` field in `Ballerina.toml` must reference the exact JAR filename matching the `kafka-admin-client` version (e.g. `kafka-admin-client-1.4.0-SNAPSHOT.jar`).
