---
sidebar_position: 5
---

# Day-Two Operations

This guide covers everything you need to operate an Ignition gateway running in Docker after the initial setup. It assumes the gateway is already running from the [Project Template](https://github.com/etknorr/project-template).

:::tip Before continuing
Read [The Compose Architecture](./compose-architecture.md) to understand how the services, volumes, and bind mounts fit together before running any of the commands below.
:::

## Common Operations

All commands run from the project directory - the folder that contains your `docker-compose.yml` file.

| Command | What it does |
| --- | --- |
| `docker compose up -d` | Start all services in the background |
| `docker compose down` | Stop and remove containers (volume and data preserved) |
| `docker compose down -v` | Stop and remove containers AND delete all volumes (full reset) |
| `docker compose ps` | Show running services and their status |
| `docker compose logs -f gateway` | Stream gateway logs (Ctrl+C to stop) |
| `docker compose restart gateway` | Restart only the gateway (preserves the volume) |
| `docker compose exec gateway bash` | Open a shell inside the running gateway container |
| `docker compose pull` | Pull the latest version of all images (does not restart) |

## First-Start Commissioning

When you run `docker compose up -d` for the first time against a fresh project-template clone, the services start in a defined order:

1. `bootstrap` runs first: seeds the `ignition-data` volume, generates a UUID from `GATEWAY_NAME`, writes the sentinel file, then exits.
2. `db` starts and passes its healthcheck (PostgreSQL ready to accept connections).
3. `gateway` starts: Ignition initializes, reads the bind-mounted config from `services/ignition/config/resources/`, connects to the database, and loads projects from `services/ignition/projects/`.

The gateway takes 60-120 seconds to finish starting. Watch progress with:

```shell
docker compose logs -f gateway
```

When you see a line containing `Gateway successfully started`, the gateway is ready. Open `https://${GATEWAY_NAME}.localtest.me` in your browser and accept the self-signed certificate warning (or add the Traefik CA to your trusted roots).

:::note Gateway login
The gateway login is whatever you set via `GATEWAY_ADMIN_USERNAME` and `GATEWAY_ADMIN_PASSWORD` in your `.env` file. To change either value, edit `.env` and restart the gateway with `docker compose up -d` (or `docker compose restart gateway`).
:::

## Restarting the Gateway

Two options depending on what you need:

- `docker compose restart gateway` - restarts the gateway process without touching the volume. Use this after changing gateway configuration in the Designer or after a `.env` change that does not require a volume reset.
- `docker compose down && docker compose up -d` - stops all services and starts them again. The volume is preserved. Use this for a clean restart of the full stack.

## Resetting to a Clean State

:::warning This deletes all gateway data
`docker compose down -v` deletes the `ignition-data` volume. Any changes made inside the gateway - Designer changes that were not committed to git, installed modules, license activation - are permanently lost.
:::

Use a full reset when:

- Upgrading to a new Ignition version
- The gateway is in an unrecoverable state
- Starting fresh for a clean lab environment

```shell
docker compose down -v
docker compose up -d
```

The bootstrap service re-seeds the volume on the next start.

## Upgrading the Ignition Version

1. Open `docker-compose.yml` and update the image tag:

   ```yaml
   image: inductiveautomation/ignition:8.3.7   # was __IGNITION_VERSION__
   ```

2. Pull the new image:

   ```shell
   docker compose pull
   ```

3. Restart the stack:

   ```shell
   docker compose down
   docker compose up -d
   ```

The volume is preserved across the restart. Ignition reads the existing data directory on startup and applies any necessary migrations automatically.

## Getting a Shell in the Gateway

```shell
docker compose exec gateway bash
```

From inside the container you can inspect files, check the Ignition data directory at `/usr/local/bin/ignition/data/`, or run Ignition's command-line tools. Exit with `exit` or Ctrl+D.

## Reading Gateway Logs

Ignition writes structured logs. Follow them live:

```shell
docker compose logs -f gateway
```

Filter for errors only:

```shell
docker compose logs gateway | grep -i error
```

The gateway also writes logs to `/usr/local/bin/ignition/logs/` inside the container. These are not bind-mounted, so they live in the volume and are lost on a `down -v` reset.

## Installing Third-Party Modules

Third-party modules (MQTT Engine, Azure Injector, etc.) installed through the gateway UI do not survive a `docker compose down -v` reset. On Kubernetes, the recommended approach is to mount `.modl` files from a shared read-only volume backed by an S3 bucket, so every gateway pod picks them up automatically without baking them into the image; see [External Modules from S3](../kubernetes/external-modules-s3.md) for the full setup.
