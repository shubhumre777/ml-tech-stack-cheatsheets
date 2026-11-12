# Docker 2026 — Master Reference

**Documentation checked:** 12 November 2026  
**Docker Engine baseline:** 29.9.3 (released 3 November 2026)

> Docker Engine, Docker CLI, Buildx, Compose, and Docker Desktop are versioned independently. Check `docker version`, `docker buildx version`, and `docker compose version` on the machine running the workflow.

## 1. Install / verify / help

```bash
docker version
docker info
docker context ls
docker context inspect default
docker context use default
docker login
docker logout
docker init
docker help
docker <command> --help
```

A Docker CLI talks to a Docker Engine through its active context. Check client and server versions separately.

## 2. Image lifecycle

```bash
docker pull nginx:alpine
docker build -t acme/api:1.0 .
docker image ls --digests
docker image inspect acme/api:1.0
docker image history acme/api:1.0
docker tag acme/api:1.0 ghcr.io/acme/api:1.0
docker push ghcr.io/acme/api:1.0
docker image save -o api.tar acme/api:1.0
docker image load -i api.tar
docker image rm acme/api:1.0
```

Use immutable digests for repeatable deployments. Tags can be moved to a different image later.

## 3. Container lifecycle

```bash
docker run --name web -d --init --restart unless-stopped -p 8080:80 --env-file .env nginx:alpine
docker run --rm -it --entrypoint sh alpine
docker ps
docker ps -a
docker start web
docker stop web
docker restart web
docker kill web
docker pause web
docker unpause web
docker rename web web-v2
docker update --memory 512m --cpus 1.5 web
docker rm web
docker rm -f web
```

- `-d`: detached; `-it`: interactive terminal; `--rm`: auto-remove after exit.
- `--init`: run a tiny init process to handle signals and reap zombie processes.
- `stop` sends SIGTERM, then SIGKILL after the configured timeout.

## 4. Logs, inspection, and debug

```bash
docker logs --tail 100 --follow --timestamps web
docker exec -it web sh
docker inspect web
docker inspect -f '{{.State.Status}}' web
docker inspect -f '{{.State.Health.Status}}' web
docker stats
docker top web
docker port web
docker cp ./file.txt web:/tmp/file.txt
docker events --since 10m
docker system df
```

Debug in this order: state and exit code → logs → healthcheck result → events → mounts, environment, network, and resource limits.

## 5. Runtime resource and health controls

| Option | Purpose |
|---|---|
| `--memory=512m` | Hard memory limit |
| `--cpus=1.5` | CPU quota |
| `--pids-limit=200` | Maximum process count |
| `--shm-size=128m` | Shared memory size |
| `--restart=unless-stopped` | Restart policy |
| `--read-only` | Read-only root filesystem |
| `--health-cmd` | Container health probe |
| `--health-interval` | Probe interval |
| `--health-timeout` | Probe timeout |
| `--health-retries` | Failures before unhealthy |
| `--health-start-period` | Startup grace period |

Health status: `starting` → `healthy` or `unhealthy`. Use `docker inspect` to read probe output.

---

## 6. Dockerfile instruction reference

| Instruction | Use |
|---|---|
| `FROM` | Base image or new build stage |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files from build context or another stage |
| `ADD` | Copy with remote URL / Git / archive handling |
| `RUN` | Execute build-time commands |
| `ARG` | Build-time variable |
| `ENV` | Persist environment variable in image config |
| `USER` | Select runtime user/group |
| `EXPOSE` | Document listening port |
| `HEALTHCHECK` | Define health probe |
| `ENTRYPOINT` | Main executable |
| `CMD` | Default command or arguments |
| `LABEL` | Image metadata |
| `VOLUME` | Declare mount point |
| `STOPSIGNAL` | Configure stop signal |
| `SHELL` | Change shell for shell-form commands |
| `ONBUILD` | Register trigger for downstream builds |

Prefer `COPY` for ordinary local files; use `ADD` only when its extra behavior is needed. Use exec/JSON form for `ENTRYPOINT` and `CMD` so the application receives OS signals directly.

## 7. Multi-stage Python image

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.14-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install -r requirements.txt

FROM python:3.14-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY --from=build /install /usr/local
COPY --chown=10001:10001 --link . .
USER 10001:10001
EXPOSE 8000
CMD ["python", "-m", "uvicorn", "app.main:app",
     "--host", "0.0.0.0", "--port", "8000"]
```

The build stage may contain compilers and build tools. Copy only required artifacts into the runtime stage. Use a non-root runtime identity and pin/update base images deliberately.

## 8. BuildKit mounts and safe builds

```dockerfile
# Package cache
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Build secret
RUN --mount=type=secret,id=pipconf,target=/etc/pip.conf \
    pip install -r requirements.txt

# SSH agent
RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git

# Bind mount (read-only by default)
RUN --mount=type=bind,source=.,target=/src make -C /src build
```

Build mount types: `bind`, `cache`, `tmpfs`, `secret`, and `ssh`.

```bash
docker buildx build --secret id=pipconf,src=$HOME/.config/pip/pip.conf .
docker buildx build --ssh default .
```

Never pass credentials through `ARG` or `ENV`. Do not copy `.env`, private keys, local caches, or credentials into the build context. Use `.dockerignore`.

## 9. Layer cache and build checks

```bash
docker buildx build --check .
docker buildx build --target runtime -t acme/api:dev .
docker buildx build --pull -t acme/api:latest .
docker buildx build --no-cache -t acme/api:test .
docker buildx build --progress=plain .
```

- Copy stable dependency manifests before frequently edited source files.
- Put package installation and temporary package-index cleanup in the same layer.
- Use named stages (`AS build`, `AS runtime`) and explicit `--target`.
- Use `COPY --link` where its destination/symlink behavior fits.
- Keep the build context small with `.dockerignore`.

---

## 10. Docker Compose — current specification

Use `compose.yaml` or `compose.yml` with the current Compose Specification. Do not add the obsolete top-level `version:` key.

```yaml
name: inference-stack
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports: ["8000:8000"]
    env_file: [".env"]
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    networks: [app]

  db:
    image: postgres:18
    environment:
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets: [db_password]
    volumes: [db_data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "app", "-d", "app"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  db_data: {}
networks:
  app: {}
secrets:
  db_password:
    file: ./secrets/db_password.txt
```

## 11. Compose command reference

```bash
docker compose config --quiet
docker compose config
docker compose pull
docker compose build
docker compose up -d --build
docker compose up --wait --wait-timeout 60
docker compose ps
docker compose logs -f --tail 100 api
docker compose exec api sh
docker compose run --rm api python -m pytest
docker compose restart api
docker compose stop
docker compose down
docker compose down --volumes  # destructive
```

Use `docker compose config` to validate and render the merged configuration before deployment. `down --volumes` removes volumes and can destroy persistent data.

## 12. Compose service options and features

- Dependencies: `depends_on`, `condition: service_healthy`.
- Health: `healthcheck.test`, `interval`, `timeout`, `retries`, `start_period`.
- Runtime: `command`, `entrypoint`, `user`, `restart`, `init`.
- Networking: `ports`, `expose`, `networks`, `network_mode`, `extra_hosts`.
- Storage: `volumes`, `tmpfs`, `configs`, `secrets`.
- Configuration: `environment`, `env_file`, `labels`.
- Resources/security: `mem_limit`, `cpus`, `pids_limit`, `read_only`, `cap_drop`, `security_opt`.
- Optional workloads: `profiles`; development: `develop.watch`.

### Compose Watch and profiles

```bash
docker compose up --watch
docker compose --profile debug up -d
docker compose -f compose.yaml -f compose.prod.yaml up -d
docker compose --env-file .env.staging up -d
```

Watch actions: `sync` copies changed files into a running service; `sync+restart` syncs then restarts; `rebuild` rebuilds when dependencies or image inputs change. Profiles keep optional debug/admin/test services opt-in. Compose Watch is a development workflow; production should deploy the immutable image that was tested.

---

## 13. Networks

```bash
docker network ls
docker network create app-net
docker network inspect app-net
docker network connect app-net api
docker network disconnect app-net api
docker network rm app-net
docker network prune
```

User-defined bridge networks provide service/container-name DNS. Keep databases private and publish only ports required by clients.

## 14. Volumes and mounts

```bash
docker volume ls
docker volume create db_data
docker volume inspect db_data
docker volume rm db_data
docker volume prune
docker run --mount type=volume,src=db_data,dst=/var/lib/postgresql/data postgres:18
docker run --mount type=bind,src="$PWD",dst=/app,readonly alpine
```

Named volumes persist beyond one container. Bind mounts expose host paths. Confirm mounts before deleting volumes or pruning resources.

## 15. Production security baseline

```bash
docker run --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop=ALL \
  --security-opt=no-new-privileges:true \
  --pids-limit=200 --memory=512m --cpus=1.5 \
  --user 10001:10001 acme/api:1.0
```

- Run as non-root; avoid `--privileged`.
- Drop unneeded Linux capabilities; keep seccomp enabled.
- Use a read-only root filesystem and explicit writable mounts.
- Set memory, CPU, process and shared-memory limits.
- Restrict access to the Docker socket; it grants control of the Engine.
- Use secrets instead of embedding credentials in image layers or source.
- Use trusted, regularly updated base images and scan them.
- Consider rootless mode when its prerequisites and workload compatibility fit.

Rootless mode runs both the daemon and containers without root privileges inside a user namespace.

## 16. Buildx, multi-platform images, and attestations

```bash
docker buildx ls
docker buildx create --name ci --driver docker-container --use
docker buildx inspect --bootstrap
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/acme/api:1.0 --push \
  --cache-from type=registry,ref=ghcr.io/acme/api:buildcache \
  --cache-to type=registry,ref=ghcr.io/acme/api:buildcache,mode=max \
  --sbom=true --provenance=mode=max .
docker buildx bake -f docker-bake.hcl
```

- `--platform`: build multiple CPU architectures.
- `--cache-from` / `--cache-to`: import/export reusable build cache.
- `--sbom=true`: attach a Software Bill of Materials.
- `--provenance=mode=max`: attach detailed build provenance.
- `docker buildx bake`: build multiple targets from HCL/Compose Bake definitions.

BuildKit attestations are metadata attached to the image. Verify registry and image-store support so attestations persist when pushed.

## 17. Docker Scout

```bash
docker scout quickview acme/api:1.0
docker scout cves acme/api:1.0
docker scout recommendations acme/api:1.0
docker scout sbom acme/api:1.0
```

Scout can summarize an image, list CVEs, recommend base-image updates, and generate/display an SBOM. Treat findings as inputs to remediation and release policy, not as a substitute for patching and rebuilding.

## 18. Cleanup and troubleshooting

```bash
docker system df
docker builder prune
docker system prune
docker system prune -a
docker system prune -a --volumes  # destructive
```

| Symptom | First checks |
|---|---|
| Container exits | `docker ps -a`, `docker inspect`, `docker logs` |
| Unhealthy container | health status and probe output in `docker inspect` |
| Service DNS fails | shared network membership and service name |
| Data missing | volume name, mount target, and container path |
| Build slow | context size, `.dockerignore`, cache hits |
| Port unreachable | publish mapping, app bind address, host firewall |

Prune removes unused resources. Adding `--volumes` can delete persistent data; inspect before confirming.

## Official references

- [Docker Engine 29 release notes](https://docs.docker.com/engine/release-notes/29/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Buildx build reference](https://docs.docker.com/reference/cli/docker/buildx/build/)
- [Build attestations](https://docs.docker.com/build/metadata/attestations/)
- [SBOM attestations](https://docs.docker.com/build/metadata/attestations/sbom/)
- [Compose CLI reference](https://docs.docker.com/reference/cli/docker/compose/)
- [Compose Specification](https://docs.docker.com/reference/compose-file/)
- [Compose services reference](https://docs.docker.com/reference/compose-file/services/)
- [Compose Watch quickstart](https://docs.docker.com/compose/gettingstarted/)
- [Rootless mode](https://docs.docker.com/engine/security/rootless/)
- [docker container run](https://docs.docker.com/reference/cli/docker/container/run/)
- [Docker Scout CLI](https://docs.docker.com/reference/cli/docker/scout/)
- [Docker Scout CVEs](https://docs.docker.com/reference/cli/docker/scout/cves/)
- [Docker system prune](https://docs.docker.com/reference/cli/docker/system/prune/)

> Version-sensitive features and flags should be verified against the installed Engine, Buildx and Compose versions. Adapt image references, user IDs, paths, ports, secrets and limits to your app.
