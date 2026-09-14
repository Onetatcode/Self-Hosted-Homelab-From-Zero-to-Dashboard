# 03, Immich

Immich is a self-hosted alternative to Google Photos. It handles auto-upload from your phone, ML-based search, timeline browsing, and sharing, all running on your NUC.

Estimated time is about twenty minutes.

Assumes [02, Docker and Portainer](02-portainer.md) is done.

## Choosing a storage location

Decide where photos will actually live on disk. For this guide the convention is `/srv/immich` for the app data (database, thumbnails) and `/srv/immich/library` for the actual photo files. If you have a mounted NAS drive at `/srv/mergerfs/NAS_Storage/` from OMV shared folders, use that instead. SSH to the NUC and create the paths.

```bash
mkdir -p /srv/immich/library
```

Substitute your chosen path in the compose file below.

## Deploying the Immich stack

In Portainer, go to Stacks and Add stack. Name it `immich`. Paste this compose file in the web editor.

```yaml
name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:release
    volumes:
      - /srv/immich/library:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - stack.env
    ports:
      - '2283:2283'
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:release
    volumes:
      - model-cache:/cache
    env_file:
      - stack.env
    restart: always
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: docker.io/valkey/valkey:8-bookworm
    healthcheck:
      test: redis-cli ping || exit 1
    restart: always

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - /srv/immich/db:/var/lib/postgresql/data
    healthcheck:
      test: >-
        pg_isready --dbname="$${POSTGRES_DB}" --username="$${POSTGRES_USER}" || exit 1
      interval: 5m
      start_interval: 30s
      start_period: 5m
    restart: always

volumes:
  model-cache:
```

Scroll down in the same Portainer page to the Environment variables section, switch to Advanced mode, and paste your variable set. Replace the placeholders with your actual values.

```
UPLOAD_LOCATION=/srv/immich/library
DB_DATA_LOCATION=/srv/immich/db
TZ=<TIMEZONE>
IMMICH_VERSION=release
DB_PASSWORD=<STRONG_PASSWORD>
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

The compose file references these via `${DB_PASSWORD}` and similar. Portainer wires them up automatically as `stack.env`, so do not paste `stack.env` as a file, it exists internally. Click Deploy the stack at the bottom.

## Waiting for containers to start

First deployment pulls around three gigabytes of images, so it takes five to ten minutes depending on connection. Watch progress with `docker ps`. You should eventually see four containers: `immich_server`, `immich_machine_learning`, `immich_postgres`, and `immich_redis`. All should show `(healthy)` in the STATUS column once ready.

## First-time setup

Open `http://<NUC_LAN_IP>:2283` in your browser. Immich shows a getting-started wizard. Create your admin account with your email, a strong password (save it), and your name. Sign up and you are logged in.

## Installing the mobile app

Immich has apps on both iOS and Android. Install from the App Store or Play Store. Open the app. It asks for a Server Endpoint URL. Enter `http://<NUC_LAN_IP>:2283` and log in with the same credentials you just created. Enable Backup, select your photo library, and let it start uploading. For remote access when you are not on your home WiFi, you will need Tailscale (next doc). Once Tailscale is set up you point the app at your Tailscale IP instead.

## What you have now

Immich is running at `http://<NUC_LAN_IP>:2283` with a web UI and phone app auto-upload when on your home network.

Next: [04, Tailscale](04-tailscale.md).

## Troubleshooting

If the machine learning container keeps restarting, it likely needs more than 2 GB free. On tight-memory NUCs you can disable it entirely, remove the `immich-machine-learning` service from the compose. You lose smart search but everything else still works.

If uploads fail with a network error, it is usually the endpoint URL. Open `http://<NUC_LAN_IP>:2283` in the phone's browser first. If the web UI loads there, the app should too. If not, the firewall or IP is wrong.

If photos upload but do not appear in the web UI, Immich needs to run its job queue to generate thumbnails. Go to Administration and Jobs in the web UI, and run all jobs.
