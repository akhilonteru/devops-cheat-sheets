# Docker Cheat Sheet — CLI Commands Reference

> Source: [docker.how](https://docker.how/) — Every Docker command with options and details.

---

## Table of Contents

- [Docker Run](#1-docker-run)
- [Container Lifecycle](#2-container-lifecycle)
- [Listing Containers (docker ps)](#3-listing-containers-docker-ps)
- [Container Logs](#4-container-logs)
- [Inspect, Stats, Top](#5-inspect-stats-top)
- [Executing Commands in Containers](#6-executing-commands-in-containers)
- [Copy Files & Import/Export](#7-copy-files--importexport)
- [Managing Images](#8-managing-images)
- [Image History & Save/Load](#9-image-history--saveload)
- [Building Images (docker build)](#10-building-images-docker-build)
- [Dockerfile Basics (FROM, LABEL, ARG, ENV)](#11-dockerfile-basics-from-label-arg-env)
- [Dockerfile Files & Commands (WORKDIR, COPY, ADD, RUN)](#12-dockerfile-files--commands-workdir-copy-add-run)
- [Dockerfile Ports, Volumes & Health (EXPOSE, VOLUME, HEALTHCHECK)](#13-dockerfile-ports-volumes--health-expose-volume-healthcheck)
- [Dockerfile Entry (CMD, ENTRYPOINT, STOPSIGNAL, SHELL)](#14-dockerfile-entry-cmd-entrypoint-stopsignal-shell)
- [Multi-Stage Builds](#15-multi-stage-builds)
- [Managing Volumes](#16-managing-volumes)
- [Mounting Storage (named volumes, bind mounts, tmpfs)](#17-mounting-storage)
- [Managing Networks](#18-managing-networks)
- [Networking Examples](#19-networking-examples)
- [Docker Compose Commands](#20-docker-compose-commands)
- [Compose Service Definition](#21-compose-service-definition)
- [Compose Build Configuration](#22-compose-build-configuration)
- [Compose Startup Order & Healthchecks](#23-compose-startup-order--healthchecks)
- [Compose Networks, Resources & Scale](#24-compose-networks-resources--scale)
- [Compose Env Vars & YAML Anchors](#25-compose-env-vars--yaml-anchors)
- [System Cleanup (prune)](#26-system-cleanup-prune)
- [System Info & Registry](#27-system-info--registry)
- [Useful Command Combinations](#28-useful-command-combinations)
- [Debugging & Troubleshooting](#29-debugging--troubleshooting)
- [Security](#30-security)

---

## 1. Docker Run

Create and start a new container from an image with various configuration options.

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

docker run -d --name web -p 80:80 nginx
docker run -it --rm ubuntu bash
docker run -d -v $(pwd):/app -w /app python:3.11 python main.py
docker run --gpus all -it tensorflow/tensorflow:latest-gpu bash
```

| Option | Description |
| --- | --- |
| `-d, --detach` | Run in background |
| `-it` | Interactive with TTY |
| `--name NAME` | Assign container name |
| `--rm` | Auto-remove on exit |
| `-p HOST:CONTAINER` | Port mapping |
| `-v HOST:CONTAINER` | Bind mount volume |
| `-e KEY=VALUE` | Set environment variable |
| `--network NET` | Connect to network |
| `--restart POLICY` | `no` \| `on-failure` \| `always` \| `unless-stopped` |
| `--cpus N` | CPU limit (e.g., 1.5) |
| `-m, --memory SIZE` | Memory limit (e.g., 512m) |
| `--gpus all` | GPU access |

---

## 2. Container Lifecycle

Start, stop, restart, pause, and remove containers.

```
docker start CONTAINER
docker stop CONTAINER
docker stop -t 30 CONTAINER
docker restart CONTAINER
docker kill CONTAINER
docker kill -s SIGTERM CONTAINER
docker pause CONTAINER
docker unpause CONTAINER
docker rm CONTAINER
docker rm -f CONTAINER
docker rm -v CONTAINER
docker rename OLD NEW
```

---

## 3. Listing Containers (docker ps)

View running and stopped containers with filtering and formatting options.

```
docker ps
docker ps -a
docker ps -q
docker ps -l
docker ps --filter "status=exited"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

---

## 4. Container Logs

View and follow container output logs with timestamps and filtering.

```
docker logs CONTAINER
docker logs -f CONTAINER
docker logs --tail 100 CONTAINER
docker logs --since 1h CONTAINER
docker logs -t CONTAINER
```

---

## 5. Inspect, Stats, Top

Get detailed metadata, resource usage, and process information.

```
docker inspect CONTAINER
docker inspect -f '{{.State.Status}}' CONTAINER
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' CONTAINER

docker top CONTAINER
docker stats
docker stats --no-stream
docker diff CONTAINER
docker port CONTAINER
docker wait CONTAINER
```

---

## 6. Executing Commands in Containers

Run commands inside a running container, interactively or detached.

```
docker exec [OPTIONS] CONTAINER COMMAND

docker exec -it web bash
docker exec web cat /etc/hosts
docker exec -u root web whoami
docker exec -w /app web ls
docker exec -e DEBUG=1 web ./run

docker attach CONTAINER
```

| Option | Description |
| --- | --- |
| `-it` | Interactive with TTY |
| `-d` | Detached (background) |
| `-u USER` | Run as user |
| `-w DIR` | Working directory |
| `-e KEY=VALUE` | Environment variable |

---

## 7. Copy Files & Import/Export

Copy files between host and container, import/export filesystems.

```
docker cp CONTAINER:SRC DEST
docker cp SRC CONTAINER:DEST
docker cp -a CONTAINER:SRC DEST

docker export CONTAINER > file.tar
docker import file.tar IMAGE
```

---

## 8. Managing Images

List, pull, push, tag, and remove Docker images.

```
docker images
docker images -a
docker images -q
docker images --filter "dangling=true"

docker pull IMAGE[:TAG]
docker pull --all-tags IMAGE
docker push IMAGE[:TAG]
docker tag SOURCE TARGET
docker rmi IMAGE
docker rmi -f IMAGE
docker image prune
docker image prune -a
```

---

## 9. Image History & Save/Load

View image history, metadata, and save/load images as tar archives.

```
docker history IMAGE
docker inspect IMAGE

docker save IMAGE > file.tar
docker save -o file.tar IMG1 IMG2
docker load < file.tar
docker load -i file.tar
```

---

## 10. Building Images (docker build)

Build Docker images from a Dockerfile with caching and multi-platform support.

```
docker build [OPTIONS] PATH

docker build -t myapp:1.0 .
docker build -f Dockerfile.prod -t myapp:prod .
docker build --build-arg VERSION=1.2.3 -t myapp .
docker build --target builder -t myapp:builder .
docker build --platform linux/amd64 -t myapp .
docker buildx build --platform linux/amd64,linux/arm64 -t myapp --push .
docker build --no-cache --pull .
docker build --progress plain .
```

| Option | Description |
| --- | --- |
| `-t NAME:TAG` | Name and tag |
| `-f FILE` | Specify Dockerfile |
| `--no-cache` | Don't use cache |
| `--pull` | Always pull base image |
| `--build-arg KEY=VALUE` | Build-time variables |
| `--target STAGE` | Multi-stage target |
| `--platform` | Single target platform (multi-arch needs buildx + `--push`) |
| `--progress plain` | Plain output |

---

## 11. Dockerfile Basics (FROM, LABEL, ARG, ENV)

`FROM` sets base image, `LABEL` adds metadata, `ARG`/`ENV` define variables.

```dockerfile
FROM ubuntu:22.04
FROM python:3.11-slim AS builder
FROM scratch

LABEL org.opencontainers.image.authors="you@email.com"
LABEL org.opencontainers.image.version="1.0"

ARG VERSION=latest
ARG DEBIAN_FRONTEND=noninteractive

ENV APP_HOME=/app
ENV PATH="$APP_HOME/bin:$PATH"
```

---

## 12. Dockerfile Files & Commands (WORKDIR, COPY, ADD, RUN)

`WORKDIR`, `COPY`, `ADD` for files; `RUN` for build commands.

```dockerfile
WORKDIR /app

COPY . .
COPY --chown=user:group src/ dest/
COPY --from=builder /app/bin /usr/local/bin/
ADD archive.tar.gz /app/
ADD https://example.com/file /app/

RUN apt-get update && apt-get install -y \
    package1 \
    package2 \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash appuser
USER appuser
```

---

## 13. Dockerfile Ports, Volumes & Health (EXPOSE, VOLUME, HEALTHCHECK)

`EXPOSE` documents ports, `VOLUME` defines mount points, `HEALTHCHECK` monitors.

```dockerfile
EXPOSE 8080
EXPOSE 443/tcp 53/udp

VOLUME ["/data", "/logs"]

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

---

## 14. Dockerfile Entry (CMD, ENTRYPOINT, STOPSIGNAL, SHELL)

Define default commands and entry points for containers.

```dockerfile
CMD ["python", "app.py"]
CMD python app.py

ENTRYPOINT ["python", "app.py"]
ENTRYPOINT ["./entrypoint.sh"]

ENTRYPOINT ["python"]
CMD ["app.py"]

STOPSIGNAL SIGTERM
SHELL ["/bin/bash", "-c"]
```

---

## 15. Multi-Stage Builds

Build in one stage, copy artifacts to minimal final image.

```dockerfile
FROM golang:1.27 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o main .

FROM alpine:3.22
COPY --from=builder /app/main /usr/local/bin/
CMD ["main"]
```

---

## 16. Managing Volumes

Create, list, inspect, and remove Docker volumes.

```
docker volume create NAME
docker volume ls
docker volume inspect NAME
docker volume rm NAME
docker volume prune
docker volume prune -a
```

---

## 17. Mounting Storage

Named volumes, bind mounts, tmpfs, and read-only options.

```bash
docker run -v myvolume:/app/data IMAGE

docker run -v /host/path:/container/path IMAGE
docker run -v $(pwd):/app IMAGE

docker run --tmpfs /tmp IMAGE

docker run -v myvolume:/data:ro IMAGE

docker run --mount type=volume,source=myvolume,target=/data IMAGE
docker run --mount type=bind,source=/host,target=/container,readonly IMAGE
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=100m IMAGE
```

---

## 18. Managing Networks

Create, list, inspect, and connect containers to networks.

```
docker network create NAME
docker network create --driver bridge NAME
docker network create --driver overlay NAME
docker network create --subnet 172.20.0.0/16 NAME

docker network ls
docker network inspect NAME
docker network rm NAME
docker network prune

docker network connect NAME CONTAINER
docker network disconnect NAME CONTAINER
```

---

## 19. Networking Examples

Connect containers by name, static IPs, and network drivers.

```bash
docker network create mynet
docker run -d --name db --network mynet postgres
docker run -d --name web --network mynet -e DB_HOST=db myapp

docker network create --subnet 172.20.0.0/16 mynet
docker run --network mynet --ip 172.20.0.10 IMAGE
```

---

## 20. Docker Compose Commands

Complete docker compose command reference for managing multi-container applications.

```bash
docker compose up [SERVICE]
docker compose up -d [SERVICE]
docker compose up --build [SERVICE]
docker compose up --force-recreate [SERVICE]
docker compose up --no-deps [SERVICE]
docker compose up --scale SERVICE=N

docker compose down
docker compose down -v
docker compose down --rmi local
docker compose down --rmi all
docker compose down --remove-orphans

docker compose start [SERVICE]
docker compose stop [SERVICE]
docker compose restart [SERVICE]
docker compose pause [SERVICE]
docker compose unpause [SERVICE]
docker compose kill [SERVICE]
docker compose rm [SERVICE]

docker compose ps
docker compose ps -a
docker compose top [SERVICE]
docker compose images

docker compose logs [SERVICE]
docker compose logs -f [SERVICE]
docker compose logs --tail=100 [SERVICE]

docker compose exec SERVICE COMMAND
docker compose exec -it SERVICE sh
docker compose run --rm SERVICE COMMAND
docker compose run --rm --entrypoint sh SERVICE

docker compose pull [SERVICE]
docker compose build [SERVICE]
docker compose build --no-cache [SERVICE]
docker compose push [SERVICE]

docker compose config
docker compose config --services
docker compose config --volumes
docker compose ls
docker compose events
```

---

## 21. Compose Service Definition

Define a service with image, ports, volumes, and environment.

```yaml
services:
  web:
    image: nginx:alpine
    container_name: web_container
    ports:
      - "80:80"
      - "443:443"
      - "127.0.0.1:3000:3000"
    volumes:
      - ./app:/app
      - data:/var/lib/data
    environment:
      - DEBUG=true
      - DATABASE_URL=postgres://db:5432/app
    env_file:
      - .env
    restart: unless-stopped

volumes:
  data:
```

---

## 22. Compose Build Configuration

Build image from source with args, target, and caching options.

```yaml
services:
  app:
    build:
      context: ./dir
      dockerfile: Dockerfile
      args:
        VERSION: "1.0"
      target: production
      cache_from:
        - myapp:cache
    image: myapp:latest
```

---

## 23. Compose Startup Order & Healthchecks

Control startup order with `depends_on` and healthchecks.

```yaml
services:
  web:
    image: myapp
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:alpine
```

---

## 24. Compose Networks, Resources & Scale

```yaml
services:
  web:
    image: nginx
    scale: 3
    cpus: 0.5
    mem_limit: 512M
    mem_reservation: 256M
    networks:
      - frontend
      - backend

networks:
  frontend:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

---

## 25. Compose Env Vars & YAML Anchors

Environment variable substitution and YAML anchors for DRY configs.

```yaml
services:
  web:
    image: ${IMAGE_NAME:-nginx}:${TAG:-latest}
    environment:
      - DB_HOST=${DB_HOST:?DB_HOST required}
```

```yaml
x-common: &common
  restart: unless-stopped
  logging:
    driver: local
    options:
      max-size: "10m"

services:
  web:
    <<: *common
    image: nginx

  api:
    <<: *common
    image: myapi
```

---

## 26. System Cleanup (prune)

Remove unused containers, images, volumes, and networks.

```
docker system df
docker system df -v

docker system prune
docker system prune -a
docker system prune -a --volumes
docker system prune -f

docker container prune
docker image prune -a
docker volume prune
docker network prune
docker builder prune
```

---

## 27. System Info & Registry

Docker version, system info, and registry authentication.

```
docker info
docker version
docker context ls
docker context use NAME

docker login
docker login registry.example.com
docker logout

docker search NAME
docker search --filter "is-official=true" nginx

docker tag myapp:latest registry.example.com/myapp:latest
docker push registry.example.com/myapp:latest
```

---

## 28. Useful Command Combinations

Useful command combinations for common tasks.

```bash
docker rm $(docker ps -aq -f status=exited)

docker rmi $(docker images -q -f dangling=true)

docker stop $(docker ps -q)

docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' CONTAINER

docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

docker logs -f -t --tail 100 CONTAINER
```

---

## 29. Debugging & Troubleshooting

Techniques for troubleshooting containers and builds.

```bash
docker build --progress=plain --no-cache .

docker commit CONTAINER debug-image
docker run -it debug-image sh

docker inspect CONTAINER --format='{{.State.ExitCode}}'
docker logs CONTAINER

docker run --rm --network NETWORK nicolaka/netshoot
docker exec CONTAINER ping other-container
docker exec CONTAINER nslookup other-container

docker export CONTAINER | tar -tvf -
```

---

## 30. Security

Run containers securely with limited privileges.

```bash
docker run --user 1000:1000 IMAGE

docker run --read-only --tmpfs /tmp IMAGE

docker run --cap-drop ALL --cap-add NET_BIND_SERVICE IMAGE

docker run --security-opt no-new-privileges IMAGE

docker scout cves IMAGE
```

---

*Compiled from https://docker.how/*
