---
sidebar_position: 3
---

# Traefik Reverse Proxy

**GitHub**: [etknorr/traefik-reverse-proxy](https://github.com/etknorr/traefik-reverse-proxy)

Traefik is a lightweight reverse proxy that runs alongside your Docker stacks and gives
each service a friendly local URL instead of a port number.

| Without Traefik | With Traefik |
| --- | --- |
| `http://localhost:8088` | `http://my-gateway.localtest.me` |
| `http://localhost:9088` | `http://other-gateway.localtest.me` |

When you are running multiple Ignition gateways locally (common in more advanced setups),
managing port numbers gets messy fast. Traefik solves this by listening on port 80 and
routing requests by hostname.

:::note Required for the labs
The [Docker Lab](../labs/docker-ignition-lab.md) and [Version Control Lab](../labs/version-control-lab.md)
both reach the gateway at `https://<GATEWAY_NAME>.localtest.me`, which requires Traefik to
route the hostname to the right container.
:::

## How It Works

`*.localtest.me` is a public wildcard DNS record that always resolves to `127.0.0.1`
(your own machine). Traefik listens on port 80 and routes requests based on the hostname.
Docker services opt in by adding a `traefik.hostname` label.

## Setup

Traefik runs as its own Docker Compose stack, separate from your project stacks. Set it up
once and leave it running.

1. Clone the Traefik repo into a utilities directory (not inside a project):

   ```shell
   mkdir -p ~/projects/utilities
   cd ~/projects/utilities
   git clone https://github.com/etknorr/traefik-reverse-proxy.git traefik
   cd traefik
   ```

2. Start Traefik:

   ```shell
   docker compose up -d
   ```

   Traefik binds to **port 80**. If something else is using port 80, stop it first.

3. Verify Traefik is running:

   <Terminal title="bash — ~/projects/utilities/traefik" lines={[
     "$ docker compose ps",
     "NAME       IMAGE            COMMAND                  SERVICE   CREATED        STATUS                   PORTS",
     "traefik    traefik:latest   \"/entrypoint.sh --pr…\"   traefik   3 seconds ago  Up 2 seconds (healthy)   0.0.0.0:80->80/tcp",
   ]} />

   Then open [http://proxy.localtest.me](http://proxy.localtest.me) in your browser.
   You should see the Traefik dashboard.

## Using Traefik with an Ignition Stack

Add these labels to your gateway service in `docker-compose.yml`:

```yaml
services:
  gateway:
    labels:
      - traefik.enable=true
      - traefik.hostname=${GATEWAY_NAME}
```

With `GATEWAY_NAME=my-gw` in your `.env`, the gateway becomes accessible at
`http://my-gw.localtest.me`.

## Stopping Traefik

```shell
cd ~/projects/utilities/traefik
docker compose down
```

Traefik does not affect other running containers when stopped - they just lose their
friendly URLs and fall back to direct port access.
