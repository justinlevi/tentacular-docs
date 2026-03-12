---
title: Quickstart
description: Get up and running with Tentacular in minutes
---

## Prerequisites

- **Kubernetes cluster** — any distribution (EKS, GKE, AKS, k0s, k3s, kind)
- **kubectl** — configured to access your cluster
- **Docker** — for building tentacle images
- **Deno 2.x** — for running the engine locally
- **Go 1.22+** — if building `tntc` from source
- **Helm** — for installing the MCP server

## Install the CLI

```bash
# Option 1: Install script (recommended)
curl -fsSL https://raw.githubusercontent.com/randybias/tentacular/main/install.sh | sh

# Option 2: Build from source
git clone https://github.com/randybias/tentacular.git
cd tentacular
make install   # builds and installs to ~/.local/bin/
tntc version   # verify
```

:::note
Make sure `~/.local/bin` is on your `PATH`. You can add it by appending
`export PATH="$HOME/.local/bin:$PATH"` to your shell profile.
:::

## Install the MCP Server

The MCP server is the in-cluster control plane. All CLI commands that interact with the cluster route through it.

```bash
# Clone the MCP server repo
git clone git@github.com:randybias/tentacular-mcp.git

# Create the support namespace (used by the esm.sh module proxy)
kubectl create namespace tentacular-support

# Generate an auth token and install via Helm
TOKEN=$(openssl rand -hex 32)
helm install tentacular-mcp ./tentacular-mcp/charts/tentacular-mcp \
  --namespace tentacular-system --create-namespace \
  --set auth.token="${TOKEN}"
```

Save the token for CLI configuration:

```bash
mkdir -p ~/.tentacular
kubectl get secret tentacular-mcp-auth -n tentacular-system \
  -o jsonpath='{.data.token}' | base64 -d > ~/.tentacular/mcp-token
```

### Accessing the MCP server

The CLI connects to the MCP server over HTTP. How you expose it depends on your cluster:

**kind / local clusters** — use port-forwarding (kind doesn't expose NodePorts to the host by default):

```bash
kubectl port-forward -n tentacular-system svc/tentacular-mcp 8080:8080 &
# MCP endpoint: http://localhost:8080/mcp
```

**Cloud clusters (EKS, GKE, AKS)** — use NodePort or LoadBalancer:

```bash
helm upgrade tentacular-mcp ./tentacular-mcp/charts/tentacular-mcp \
  --namespace tentacular-system \
  --reuse-values \
  --set service.type=NodePort \
  --set service.nodePort=30080
# MCP endpoint: http://<node-ip>:30080/mcp
```

Verify the server is healthy:

```bash
curl http://localhost:8080/healthz
# {"status":"ok"}
```

## Configure the CLI

```bash
tntc configure \
  --registry <your-registry> \
  --default-namespace <your-namespace> \
  --project
```

This writes project-level defaults to `.tentacular/config.yaml`.

Next, add the MCP endpoint and token to your config. Open `.tentacular/config.yaml` and add `mcp_endpoint` and `mcp_token_path` to your environment:

```yaml
# .tentacular/config.yaml
registry: <your-registry>
namespace: <your-namespace>
runtime_class: gvisor
default_env: dev

environments:
  dev:
    namespace: <your-namespace>
    mcp_endpoint: http://localhost:8080/mcp
    mcp_token_path: ~/.tentacular/mcp-token
```

Verify cluster connectivity:

```bash
tntc cluster check --env dev
```

:::note
`tntc configure` automatically generates a cluster profile on first run.
You can regenerate it later with `tntc cluster profile --env dev --save`.
:::

## Create Your First Tentacle

### From Scratch

```bash
tntc init my-first-tentacle
cd my-first-tentacle
```

This scaffolds:
- `workflow.yaml` — tentacle definition
- `nodes/hello.ts` — a starter node
- `tests/fixtures/hello.json` — test fixture

### From a Template

```bash
# Browse available templates
tntc catalog list

# Scaffold from a template
tntc catalog init word-counter my-first-tentacle
cd my-first-tentacle
```

## Develop Locally

```bash
# Validate the workflow spec
tntc validate

# Run the local dev server with hot-reload
tntc dev

# In another terminal, trigger the tentacle
curl -X POST http://localhost:8080/run

# Run tests
tntc test
```

## Deploy to Kubernetes

```bash
# Set up secrets (if the tentacle needs them)
tntc secrets init
# Edit .secrets.yaml with your values

# Build the engine image
tntc build --push

# Deploy
tntc deploy

# Verify
tntc status my-first-tentacle
tntc logs my-first-tentacle --tail 20
```

## Trigger and Monitor

```bash
# Trigger manually
tntc run my-first-tentacle

# List all deployed tentacles
tntc list

# Check health
tntc status my-first-tentacle --detail
```

## Clean Up

```bash
tntc undeploy my-first-tentacle --yes
```

## Next Steps

- [Your First Tentacle](/tentacular-docs/guides/first-tentacle/) — detailed walkthrough of building a tentacle from scratch
- [CLI Reference](/tentacular-docs/reference/cli/) — complete command reference
- [Workflow Spec](/tentacular-docs/reference/workflow-spec/) — all workflow.yaml fields
- [Security](/tentacular-docs/concepts/security/) — understand the defense-in-depth model
- [Catalog Templates](/tentacular-docs/guides/catalog-usage/) — browse and use pre-built templates
