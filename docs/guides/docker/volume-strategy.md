---
sidebar_position: 3
---

# Volumes and Bind Mounts

:::tip Before continuing
Read [The Compose Architecture](./compose-architecture.md) before this page.
:::

The [Project Template](https://github.com/etknorr/project-template) combines two storage mechanisms based on the [IA Version Control Guide](https://docs.inductiveautomation.com/docs/8.3/tutorials/version-control-guide#advanced-version-control-considerations): a Docker-managed named volume for runtime state, and bind mounts from your repository for the files Git should track. The bind mounts layer on top of the volume at the same paths, so Ignition sees a unified `/data` directory while Git only sees the specific subdirectories you care about.

## What the named volume holds

The `ignition-data` volume is managed by Docker and holds everything that should persist across restarts but should NOT be tracked in Git:

- The internal database: alarm journal, tag history for short-term storage, audit log
- Installed modules and their state
- The Java keystore: TLS certificates, GAN certificates
- License activation records
- Logs
- Anything Ignition writes at runtime

The volume survives `docker compose down` but is deleted by `docker compose down -v`. Think of it as Ignition's installed files - analogous to what lives under `C:\Program Files\Inductive Automation\Ignition\data` on a Windows host install.

## What the bind mounts hold

Three directories from your repository are mounted into the container on top of the volume:

| Repository path | Container path | What it contains |
| --- | --- | --- |
| `services/ignition/projects/` | `/data/projects/` | Ignition projects: Perspective views, tags, scripts, etc. |
| `services/ignition/config/resources/core/` | `/data/config/resources/core/` | Shared gateway config: database connections, OPC-UA settings, themes, etc. |
| `services/ignition/config/resources/dev/` | `/data/config/resources/dev/` | Dev environment overrides: dev database URL, dev tag provider, etc. |

Git only sees these three directories. Everything else in the container lives in the volume and is invisible to Git.

## How the layering works

The bind mounts apply on top of the named volume at the same path. If a file exists in the volume at `/data/config/resources/core/ignition/database-connection/db/resource.json`, the bind mount at `/data/config/resources/core/` makes the repository's version of that file take precedence. The volume provides the base; the bind mounts add and override specific paths.

In practice this means:

- First start: the volume is seeded by the bootstrap process, then bind mounts layer on top
- Subsequent starts: the volume already exists, bind mounts apply on top again
- `git pull` with new config files: immediately visible to the running gateway on next restart

No container rebuild is required when you add or change files in the mounted directories. The gateway reads them from the host filesystem on startup.

## Why you cannot mount `/data` directly

:::danger
Mounting a fresh empty volume - or an empty host directory - at `/data` causes `CrashLoopBackOff`. The Ignition container expects certain files to already exist at startup (`gateway.xml_clean`, `metro-keystore`, etc.). An empty mount hides the files that the container image itself provides, and the gateway cannot start.

This is the most common cause of immediate container crashes when running Ignition on Kubernetes without an init container. The project-template avoids this entirely by mounting only subdirectories, leaving the top-level `/data` structure from the image's own filesystem intact.
:::

## Resource collections and deployment modes

The `config/resources/` directory supports multiple named collections as subdirectories. The JVM argument `-Dignition.config.mode=dev` activates the `dev` collection. Both `core` and `dev` are active simultaneously when the gateway starts: `core` applies to all environments, `dev` overrides or supplements it for local development.

```text
services/ignition/config/resources/
├── core/          # applies to all environments
│   └── ignition/
│       └── database-connection/db/   # shared DB config
└── dev/           # applies when DEPLOYMENT_MODE=dev
    └── ignition/
        └── database-connection/db/   # dev-specific DB URL override
```

Adding a production environment means creating `services/ignition/config/resources/prod/`, adding a bind mount for it in `docker-compose.yml`, and setting `DEPLOYMENT_MODE=prod` in the environment. See [Gateway Resource Collections](../../reference/resource-collections.md) for the full reference.
