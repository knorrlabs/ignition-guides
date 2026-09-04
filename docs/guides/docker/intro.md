---
sidebar_position: 1
---

# Introduction to Docker for Ignition

:::tip Before continuing

- [Workstation Setup](../../getting-started/workstation-setup.md) must be complete - Docker Desktop must be installed and running.
- Familiarity with the [Version Control guide](../version-control/intro.md) is helpful but not required.

:::

Docker changes how you install, run, and share Ignition gateways. Instead of running an installer on your workstation and managing a system service, you declare a gateway in a compose file and start it with a single command. This guide explains the concepts you need before working with the [Project Template](https://github.com/etknorr/project-template).

## What Docker Solves for Ignition Developers

The most immediate benefit is running multiple Ignition versions simultaneously. If you support a client on 8.3.4 and another on __IGNITION_VERSION__, you can run both gateways on the same machine without any manual installer switching. Each gateway lives in its own container, on its own ports, behind its own Traefik route.

Beyond version isolation, Docker gives every developer on a project an identical environment. The gateway a new team member starts on day one behaves the same as the one that has been running in a CI pipeline for six months. There is no "works on my machine" because the image and compose configuration define the environment completely.

Networking between multiple gateways is handled by Traefik, the reverse proxy that routes `*.localtest.me` hostnames to the correct container. Each gateway gets a clean URL without port numbers - no more remembering whether the staging gateway is on port 8088 or 8089.

Teardown and reset are also cleaner. Removing a gateway installed on a host requires uninstalling the service, deleting leftover files across several system directories, and sometimes rebooting. With Docker, `docker compose down -v` removes the containers and their volumes in seconds, and `docker compose up` starts fresh. Nothing persists on the host.

## Containers vs. VMs: the Ignition Mental Model

A virtual machine runs a complete operating system on top of a hypervisor. Starting a VM means booting a kernel, initializing services, and waiting for a full OS to come up - this is why Ignition VMs can take minutes to reach the gateway login screen.

A container shares the host OS kernel. There is no second operating system to boot. The Ignition process inside a container starts the same Java runtime on the same kernel as everything else on your machine. Gateway startup time drops from minutes to seconds.

From Ignition's perspective, nothing has changed. The gateway process runs exactly as it would on a Linux server - same Java runtime, same port bindings, same file paths. The gateway does not know or care that it is inside a container. The web interface is at port 8088, the modules load from the same paths, and scripting functions behave identically.

The critical difference is filesystem lifetime. Everything written inside a container's filesystem is discarded when the container stops, unless it was written to a volume or bind mount. A fresh container is a fresh slate. For Ignition this matters a great deal - the gateway's internal database, installed modules, and license activation would all disappear between restarts if not explicitly configured to persist. The next section explains how volumes and bind mounts solve this.

## Named Volumes vs. Bind Mounts: Why Ignition Needs Both

Ignition's data falls into two categories that need different persistence strategies. Named volumes handle runtime state (the internal database, installed modules, license activation) that must survive restarts but does not belong in Git. Bind mounts handle the git-tracked project files and gateway configuration that you edit in your repository.

See [Volume Strategy](./volume-strategy.md) for the full breakdown of what lives where and why.

## The Official `inductiveautomation/ignition` Image

Inductive Automation publishes official Docker images at `inductiveautomation/ignition` on Docker Hub. Images are tagged by Ignition version:

```text
inductiveautomation/ignition:__IGNITION_VERSION__
```

The image packages the same Ignition installer used for bare-metal deployments. The gateway inside the image is configured to accept command-line arguments and environment variables for headless operation - no installer wizard, no manual configuration screen. You pass the gateway name, port assignments, JVM memory, and other settings through the compose file.

On first start, if the gateway finds no existing data in its data directory, it initializes a fresh gateway automatically. If data already exists (from a named volume that was seeded), the gateway loads that existing state instead. The project-template takes advantage of this by running a bootstrap service that seeds the named volume before the gateway starts, ensuring the gateway always finds a properly initialized data directory rather than starting completely cold. This is covered in detail in [The Compose Architecture](./compose-architecture.md).

## Licensing in Containers

Docker does not change how Ignition licensing works - it changes how licensing is stored and activated. The license is tied to the gateway's UUID, which is stored in the named volume. If the volume is deleted, the activation is lost and must be restored. The bootstrap service in the project-template generates a deterministic UUID from the gateway name so that activation can be recovered predictably.

See [Licensing in Containers](./licensing.md) for the full picture before running a licensed gateway in Docker.

## Further Reading

These guides cover Docker through the lens of running Ignition. If you want to go deeper on Docker and Compose themselves:

- [Docker Compose Documentation](https://docs.docker.com/compose/) - official reference for compose file syntax, commands, and concepts
- [Docker Documentation](https://docs.docker.com/) - the broader Docker engine, images, networking, and storage
- [Observability for Ignition](../observability/intro.md) - attaching the OTel Java agent to a Docker Compose gateway for metrics, traces, and logs
