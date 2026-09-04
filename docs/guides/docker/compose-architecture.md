---
sidebar_position: 2
---

# The Compose Architecture

:::tip Before continuing

- [Introduction to Docker for Ignition](./intro.md)
- [Traefik Reverse Proxy](../../getting-started/traefik.md) must be set up and running before `docker compose up` will succeed.

:::

The [Project Template](https://github.com/etknorr/project-template) defines three services in its compose file. Each one exists for a specific reason. This guide walks through exactly what happens when you run `docker compose up`, service by service.

## The Three Services

### The `bootstrap` Service

`bootstrap` runs once before the gateway starts. Its job is to prepare the `ignition-data` named volume so the gateway finds a properly initialized data directory on first boot.

An empty named volume would cause the gateway to fail immediately. Ignition expects certain files to exist in its data directory (`gateway.xml_clean`, directory structure for modules, etc.) before it can initialize. The bootstrap service seeds the volume with Ignition's base files by copying them from the official image before the gateway process ever starts.

Bootstrap also generates a deterministic UUID from the `GATEWAY_NAME` environment variable using `md5sum`. This matters for licensing: Ignition ties a license activation to the gateway's UUID. By deriving the UUID from a predictable input (the gateway name), the same gateway name always produces the same UUID. If the volume is lost and recreated, the UUID is the same and the license can be restored without contacting Inductive Automation.

To avoid re-seeding on every start, bootstrap writes a `.ignition-seed-complete` sentinel file to the volume after its first run. On subsequent starts, it checks for that file and exits immediately. The `gateway` service uses `depends_on` with a `service_completed_successfully` condition, so it will not start until bootstrap exits cleanly.

### The `gateway` Service

`gateway` runs the official `inductiveautomation/ignition` image. The command arguments passed to the container determine almost everything about how the gateway behaves.

```yaml
command:
  - "-n"
  - "${GATEWAY_NAME}"
  - "-m"
  - "4096"
  - "-h"
  - "80"
  - "-s"
  - "443"
  - "-a"
  - "${GATEWAY_NAME}.localtest.me"
  - "-Dignition.config.mode=${DEPLOYMENT_MODE:-dev}"
```

`-n $GATEWAY_NAME` sets the gateway name that appears in the Status page and the gateway network. It also becomes the container's hostname inside Docker's network.

`-m 4096` sets the JVM heap size in megabytes. 4096 MB is 4 GB. Set this to roughly 85% of the memory you want to allocate to the gateway - leaving headroom for the JVM's off-heap usage. Too low and the gateway will crash under load; too high and you crowd out other containers.

`-h 80 -s 443` sets the HTTP and HTTPS ports the gateway listens on inside the container. These are container-internal ports, not host ports. Traefik routes external traffic to these ports across the `proxy` network.

`-a ${GATEWAY_NAME}.localtest.me` is the public address Ignition tells clients to connect to. This must exactly match the URL clients actually use to reach the gateway. Traefik routes `${GATEWAY_NAME}.localtest.me` to the container, and Ignition advertises that same address to Perspective clients. If these do not match, the gateway and the reverse proxy disagree about where clients should go.

`-Dignition.config.mode=${DEPLOYMENT_MODE:-dev}` activates a named resource collection. When set to `dev`, Ignition loads configuration from the `dev/` resource directory in addition to `core/`. This is how environment-specific settings (database connection strings, tag provider configurations) are separated from shared configuration. See [Gateway Resource Collections](../../reference/resource-collections.md) for a full explanation.

The gateway service also sets one JVM system property that is easy to miss:

```yaml
environment:
  - GATEWAY_SYSTEM_PROPS=gateway.useProxyForwardedHeader=true
```

This tells Ignition to trust the `X-Forwarded-For` and `X-Forwarded-Proto` headers that Traefik adds to proxied requests. Without it, Ignition sees HTTP traffic arriving at port 80 (the internal container port) and generates redirect URLs using `http://` - even though Traefik is serving HTTPS externally. The gateway and Traefik end up in a redirect loop, and Perspective sessions fail.

:::warning Perspective sessions will fail without this
`gateway.useProxyForwardedHeader=true` and `-a ${GATEWAY_NAME}.localtest.me` must both be set correctly when running behind a reverse proxy. If either is missing or wrong, opening a Perspective project produces a `MissingGatewayAddressException` error.
:::

### The `db` Service

`db` runs PostgreSQL with a healthcheck. Its purpose is to provide a persistent, network-accessible database for the Ignition gateway's external database connection.

The `gateway` service uses `depends_on` with `condition: service_healthy` pointing at `db`. Docker evaluates the healthcheck before allowing the gateway to start. Without this ordering, the gateway might initialize its database connection before PostgreSQL is ready to accept connections, producing a connection error on first boot that requires a manual gateway restart.

Inside Docker's network, services reach each other by service name. The database hostname is `db` - the name of the service in the compose file. The Ignition database connection resource stored at `services/ignition/config/resources/core/ignition/database-connection/db/` uses `db` as the JDBC hostname. This works because Docker's internal DNS resolves service names automatically within a compose project.

The database data is stored in a named volume so it survives container restarts. Stopping and restarting the `db` service does not lose your data unless you explicitly remove the volume with `docker compose down -v`.

## The External `proxy` Network

The `gateway` service is attached to an external Docker network named `proxy`. This network is created and managed by Traefik, which you set up in [Traefik Reverse Proxy](../../getting-started/traefik.md). Traffic from a browser to `${GATEWAY_NAME}.localtest.me` enters Traefik, which forwards it across the `proxy` network to the gateway container.

If Traefik is not running when you run `docker compose up`, Docker will fail immediately with:

```text
network proxy not found
```

The `db` service is not attached to the `proxy` network. It is only reachable from other services in the same compose project, on the internal network Docker creates automatically. Keeping the database off the proxy network means it is never reachable from a browser, even accidentally.

## The `.env` File

Docker Compose reads variables from a `.env` file in the project root and substitutes them into the compose file wherever `${VARIABLE}` appears. The template ships with `.env.example` containing placeholder values. Copy it to `.env` and fill in your values before running `docker compose up`.

| Variable | Purpose |
| --- | --- |
| `GATEWAY_NAME` | Gateway display name, container hostname, and Traefik route - the gateway is accessible at `${GATEWAY_NAME}.localtest.me` |
| `DB_USER` | PostgreSQL username for the Ignition database connection |
| `DB_PASSWORD` | PostgreSQL password |
| `GATEWAY_ADMIN_USERNAME` | Username for the initial gateway admin login |
| `GATEWAY_ADMIN_PASSWORD` | Password for the initial gateway admin login - change before the first start |
| `TZ` | Container timezone (e.g. `America/Los_Angeles`) - affects timestamps in Ignition logs and tag history |
| `DEPLOYMENT_MODE` | Activates a named resource collection - `dev` by default (see [Gateway Resource Collections](../../reference/resource-collections.md)) |

:::tip Full environment variable reference
For the complete list of environment variables the Ignition image supports, see the [Platform Environment Variables reference](https://docs.inductiveautomation.com/docs/8.3/appendix/reference-pages/platform-environment-variables#environment-variables) in the Ignition 8.3 docs.
:::

`.env` is listed in `.gitignore` and will not be committed. Never remove it from `.gitignore` - it may contain database credentials. If you need to share default values with your team, edit `.env.example` instead.
