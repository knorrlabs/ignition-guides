---
sidebar_position: 2
---

# Stoker Operator

**GitHub**: [knorrlabs/stoker-operator](https://github.com/knorrlabs/stoker-operator)  
**Docs**: [knorrlabs.github.io/stoker-operator](https://knorrlabs.github.io/stoker-operator/)

## What It Does

Stoker is a Kubernetes operator that continuously syncs Ignition SCADA gateway configuration from Git. It uses a controller + agent sidecar architecture:

- The **controller** watches `GatewaySync` custom resources, resolves git refs, and discovers gateway pods
- The **agent** runs as a native sidecar inside gateway pods, clones the repo, and syncs files to the gateway's data directory

No shared PVC required. Each `GatewaySync` CR gets two ConfigMaps: the controller writes one that the agent reads (control metadata: repo URL, ref, credentials reference), and the agent writes a second that the controller reads (sync status). The agent clones the Git repo into a local `emptyDir` volume and calls the gateway's scan REST API to apply changes, so no shared PVC is needed.

## When to Use It

Stoker is for teams running Ignition on Kubernetes who want GitOps-style config management - treat your Ignition project files as code, stored in Git, automatically deployed to gateways when changes land on a branch.

## Key Features

- Watches a git branch and syncs on commit (or via webhook trigger)
- Supports GitHub App, SSH key, and API token authentication
- Mutating webhook injects the agent sidecar automatically with namespace/pod labels
- Webhook receiver supports GitHub releases, ArgoCD notifications, Kargo promotions, and generic payloads
- Helm chart published to `oci://ghcr.io/knorrlabs/charts/stoker-operator`

## Quick Links

- [Quickstart](https://knorrlabs.github.io/stoker-operator/quickstart)
- [Installation](https://knorrlabs.github.io/stoker-operator/installation)
- [Helm Values Reference](https://knorrlabs.github.io/stoker-operator/reference/helm-values)

## End-to-End Config Sync Guide

For a walkthrough of setting up config sync in a real Ignition deployment - including the `GatewaySync` resource, SSH auth wiring, profile mappings, and the fallback git-sync approach - see [Config Sync](../guides/kubernetes/config-sync.md).

## See Also

- [External Secrets](../guides/kubernetes/external-secrets.md): how the SSH key Stoker uses for Git authentication is provisioned from a cloud secret store into a Kubernetes Secret
