# Running multiple isolated Hermes agents with Docker

One container per project, fully isolated state, resources, image pin, and network.
Use this when you need hard isolation between projects — separate blast radius,
separate upgrade cadence, separate resource ceilings — rather than the
single-container/multi-profile pattern from the user guide.

## Isolation guarantees

| Boundary | How it's enforced |
|---|---|
| State (`SOUL.md`, sessions, memories, skills, `.env`) | Distinct host volume per container (`~/.hermes-<project>`) |
| CPU / RAM | Per-container `--memory` and `--cpus` (or compose `deploy.resources.limits`) |
| Image version | Each service pins its own tag — upgrade one project without touching the others |
| Network | Per-project bridge network; no cross-project DNS or routing |
| API surface | Each container publishes its gateway / dashboard on a distinct host port |
| Failure blast radius | One project's crash, OOM, or bad config can't take the others down |

> **Never** point two containers at the same `~/.hermes` directory. Session
> files and memory stores are not designed for concurrent writers.

## Layout

```
~/.hermes-juno/        # project A state
~/.hermes-anima/       # project B state
~/.hermes-daodao/      # project C state

/workspace/juno/       # project A code
/workspace/anima/      # project B code
/workspace/dao-dao/    # project C code
```

## Port allocation

Pick a base port per project so the mapping stays memorable:

| Project | Gateway API | Dashboard |
|---|---|---|
| juno   | 8642 | 9119 |
| anima  | 8742 | 9219 |
| daodao | 8842 | 9319 |

## `docker-compose.yml`

```yaml
services:
  hermes-juno:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-juno
    restart: unless-stopped
    command: gateway run
    ports:
      - "8642:8642"
      - "9119:9119"
    volumes:
      - ~/.hermes-juno:/opt/data
      - /workspace/juno:/work
    environment:
      - HERMES_DASHBOARD=1
      - HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
      - HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=${JUNO_DASH_PASSWORD}
    shm_size: 1g
    networks:
      - juno-net
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: "2.0"

  hermes-anima:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-anima
    restart: unless-stopped
    command: gateway run
    ports:
      - "8742:8642"
      - "9219:9119"
    volumes:
      - ~/.hermes-anima:/opt/data
      - /workspace/anima:/work
    environment:
      - HERMES_DASHBOARD=1
      - HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
      - HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=${ANIMA_DASH_PASSWORD}
    shm_size: 1g
    networks:
      - anima-net
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "1.0"

  hermes-daodao:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-daodao
    restart: unless-stopped
    command: gateway run
    ports:
      - "8842:8642"
      - "9319:9119"
    volumes:
      - ~/.hermes-daodao:/opt/data
      - /workspace/dao-dao:/work
    environment:
      - HERMES_DASHBOARD=1
      - HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
      - HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=${DAODAO_DASH_PASSWORD}
    shm_size: 1g
    networks:
      - daodao-net
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "1.0"

networks:
  juno-net:
  anima-net:
  daodao-net:
```

Each service has its own bridge network — no project's container can resolve
or reach another's by name. Combine with a sidecar (e.g. a local vLLM) by
attaching the sidecar to that project's network only.

## One-time setup per project

Run the interactive setup wizard against each empty data dir before first
`docker compose up`:

```bash
mkdir -p ~/.hermes-juno ~/.hermes-anima ~/.hermes-daodao

docker run -it --rm -v ~/.hermes-juno:/opt/data \
  nousresearch/hermes-agent setup
docker run -it --rm -v ~/.hermes-anima:/opt/data \
  nousresearch/hermes-agent setup
docker run -it --rm -v ~/.hermes-daodao:/opt/data \
  nousresearch/hermes-agent setup
```

The wizard writes `SOUL.md`, `config.yaml`, and `.env` into each volume — give
each project a distinct persona at this step.

## Daily ops

```bash
# Start everything
docker compose up -d

# Per-project lifecycle
docker compose restart hermes-juno
docker compose stop hermes-anima
docker compose logs -f hermes-daodao

# Talk to one project's gateway
docker exec -it hermes-juno hermes
docker exec -it hermes-anima hermes session list
```

## Independent upgrades

Pin tags so upgrading one project never touches the others:

```yaml
hermes-juno:
  image: nousresearch/hermes-agent:0.42.0
hermes-anima:
  image: nousresearch/hermes-agent:0.41.3
```

Upgrade workflow for a single project:

```bash
docker compose pull hermes-juno
docker compose up -d hermes-juno
```

The container runs non-interactive config-schema migrations on startup; API
keys and settings in the volume are preserved.

## When **not** to use this pattern

If you only need separate personas / memories / skills and don't care about
resource caps, image pins, or network isolation, use one container with multiple
profiles instead — it's lighter and the recommended default in the docs:

```bash
docker exec hermes hermes profile create juno
docker exec hermes hermes -p juno gateway start
```

Choose isolation when at least one of these is true:

- A project does browser automation or heavy tool use and you don't want it
  starving the others
- Projects need different versions of the image (testing a release candidate
  against one project before rolling it out)
- A project has untrusted skills or MCP servers and you want them on their own
  bridge network
- You need to wipe / migrate / back up one project's state without coordinating
  with the others

## Gotchas

- **Host port collisions**: each project must pick a unique host port. If you
  add a fourth project, extend the allocation table — don't reuse.
- **UID/GID**: the image runs as UID/GID 10000 by default. If host-side tools
  need to read `~/.hermes-<project>`, set `PUID` / `PGID` to your host user, or
  `chmod 755` the dir.
- **Volume backups**: snapshot each `~/.hermes-<project>` dir independently;
  there's no shared state to coordinate.
- **Browser tools**: `shm_size: 1g` is required for Playwright stability under
  load. Add it to every service that uses browser skills.
- **Dashboard auth**: when binding the dashboard to a non-loopback address
  (i.e. publishing it on the host), the auth gate requires basic-auth, OAuth,
  or OIDC env vars — it will refuse to start otherwise.
