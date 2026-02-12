---
title: "Configuring Resolvers"
description: "How to configure resolvers to map component versions to OCM repositories."
icon: "🔍"
weight: 56
toc: true
---

## Overview

When working with OCM component versions, the CLI needs to know **which repository** stores a given component. 
By default, you specify the repository directly in the command:

```bash
ocm get cv ghcr.io/open-component-model/ocm//ocm.software/ocmcli:0.23.0
```

This works well for simple scenarios. But in practice, components are often spread across multiple registries — for example, 
third-party components in a public registry and internal components in a private one. **Resolvers** let you configure this 
mapping once so the CLI automatically routes each component to the correct repository.

Resolvers use the configuration type `resolvers.config.ocm.software/v1alpha1` and replace the deprecated priority-based 
fallback resolvers from earlier OCM versions with glob-based pattern matching.

**This guide is for users who want to:**

- Understand what resolvers are and when to use them
- Configure resolvers with glob-based component name matching
- Set up multi-registry environments with a single configuration file
- Use resolvers for recursive component resolution across repositories

## Prerequisites

- The [OCM CLI](https://github.com/open-component-model/open-component-model) installed
- Access to at least one OCI registry (e.g., `ghcr.io`, Docker Hub, or a private registry)

## What Are Resolvers?

A resolver maps a **component identity** (name + version) to an **OCM repository** where it is stored. When the CLI needs 
to find a component version, it consults the configured resolvers to determine which repository to query.

This is particularly useful when:

- Components are distributed across multiple OCI registries
- You want to avoid specifying the full repository path in every command
- You need recursive resolution of component references that span multiple registries

## Configuration

Resolvers are configured in the OCM configuration file. By default, the CLI searches for configuration in the following locations (in order):

1. The path specified by the `OCM_CONFIG` environment variable
2. XDG / Home directories:
   - `$XDG_CONFIG_HOME/ocm/config`
   - `$XDG_CONFIG_HOME/.ocmconfig`
   - `$HOME/.config/ocm/config`
   - `$HOME/.config/.ocmconfig`
   - `$HOME/.ocm/config`
   - `$HOME/.ocmconfig`
3. Current working directory:
   - `$PWD/ocm/config`
   - `$PWD/.ocmconfig`

You can also specify a configuration file explicitly with the `--config` flag.

{{<callout context="tip">}}
If multiple configuration files are found, they are merged. This allows you to layer configurations — for example, 
a global config in your home directory and a project-specific config in the working directory.
{{</callout>}}

### Basic Configuration

The resolver configuration uses the type `resolvers.config.ocm.software/v1alpha1` inside a generic OCM configuration file:

```yaml
type: generic.config.ocm.software/v1
configurations:
  - type: resolvers.config.ocm.software/v1alpha1
    resolvers:
      - repository:
          type: OCIRegistry/v1
          baseUrl: ghcr.io
          subPath: open-component-model/ocm
```

This tells the CLI: "When looking for a component version, check the OCI registry at `ghcr.io/open-component-model/ocm`."

With this configuration in place, you can omit the repository from commands:

```bash
# Instead of specifying the full repository path:
ocm get cv ghcr.io/open-component-model/ocm//ocm.software/ocmcli:0.23.0

# You can simply reference the component by name:
ocm get cv ocm.software/ocmcli:0.23.0
```

### Repository Types

The `repository` field accepts any OCM repository specification. The most common types are:

**OCI Registry** — for OCI-based registries (e.g., ghcr.io, Docker Hub):

```yaml
repository:
  type: OCIRegistry/v1
  baseUrl: ghcr.io
  subPath: open-component-model/ocm
```

| Field     | Required | Description                                                                                     |
|-----------|----------|-------------------------------------------------------------------------------------------------|
| `type`    | Yes      | Repository type. Accepted values include `OCIRegistry/v1` and the canonical `OCIRepository/v1`. |
| `baseUrl` | Yes      | Registry host and optional port (e.g., `ghcr.io`, `localhost:5000`).                            |
| `subPath` | No       | Repository prefix path within the registry.                                                     |

**Common Transport Format (CTF)** — for file-based archives:

```yaml
repository:
  type: CommonTransportFormat/v1
  filePath: /path/to/ctf-archive
```

| Field        | Required | Description                                                                                      |
|--------------|----------|--------------------------------------------------------------------------------------------------|
| `type`       | Yes      | Repository type. Accepted values include `CommonTransportFormat/v1` and the short form `CTF/v1`. |
| `filePath`   | Yes      | Path to the CTF archive file or directory.                                                       |
| `accessMode` | No       | Access mode: `readonly`, `readwrite`, or `create`.                                               |

### Component Name Patterns

Each resolver entry can include a `componentNamePattern` field that uses **glob patterns** to match component names. Only components matching the pattern will be routed to that repository.

```yaml
type: generic.config.ocm.software/v1
configurations:
  - type: resolvers.config.ocm.software/v1alpha1
    resolvers:
      - repository:
          type: OCIRegistry/v1
          baseUrl: ghcr.io
          subPath: open-component-model/ocm
        componentNamePattern: "ocm.software/*"
```

Supported glob patterns:

| Pattern                      | Matches                                                 |
|------------------------------|---------------------------------------------------------|
| `ocm.software/*`            | Any component directly under `ocm.software/`            |
| `ocm.software/core/*`       | Any component under `ocm.software/core/`                |
| `*.software/*/test`         | Components named `test` in any `*.software` namespace   |
| `ocm.software/core/[tc]est` | `ocm.software/core/test` or `ocm.software/core/cest`   |
| `*`                         | All components (wildcard catch-all)                     |

{{<callout context="note">}}
Resolvers are evaluated **in the order they are defined**. The first matching resolver wins. Place more specific patterns before broader ones.
{{</callout>}}

## Examples

### Multiple Registries

A common scenario is routing components to different registries based on their namespace. For example, OCM core components are stored in one registry, while your organization's components are in another:

```yaml
type: generic.config.ocm.software/v1
configurations:
  - type: resolvers.config.ocm.software/v1alpha1
    resolvers:
      # OCM core components
      - repository:
          type: OCIRegistry/v1
          baseUrl: ghcr.io
          subPath: open-component-model/ocm
        componentNamePattern: "ocm.software/*"
      # Internal components
      - repository:
          type: OCIRegistry/v1
          baseUrl: myregistry.example.com
          subPath: my-org/components
        componentNamePattern: "mycompany.io/*"
```

With this configuration:

- `ocm get cv ocm.software/ocmcli:0.23.0` resolves to `ghcr.io/open-component-model/ocm`
- `ocm get cv mycompany.io/my-app:1.0.0` resolves to `myregistry.example.com/my-org/components`

### Wildcard Fallback

You can add a catch-all resolver at the end to handle any component that does not match a specific pattern:

```yaml
type: generic.config.ocm.software/v1
configurations:
  - type: resolvers.config.ocm.software/v1alpha1
    resolvers:
      # Specific pattern first
      - repository:
          type: OCIRegistry/v1
          baseUrl: ghcr.io
          subPath: open-component-model/ocm
        componentNamePattern: "ocm.software/*"
      # Catch-all fallback
      - repository:
          type: OCIRegistry/v1
          baseUrl: myregistry.example.com
          subPath: components
        componentNamePattern: "*"
```

### Mixed Repository Types

You can combine different repository types in the same configuration. For example, use a local CTF archive for development and an OCI registry for production components:

```yaml
type: generic.config.ocm.software/v1
configurations:
  - type: resolvers.config.ocm.software/v1alpha1
    resolvers:
      # Local development components from a CTF archive
      - repository:
          type: CommonTransportFormat/v1
          filePath: ./local-components.ctf
        componentNamePattern: "dev.mycompany.io/*"
      # Production components from OCI registry
      - repository:
          type: OCIRegistry/v1
          baseUrl: myregistry.example.com
          subPath: production/components
        componentNamePattern: "mycompany.io/*"
```

### Combining Resolvers with Credentials

Resolver and credential configurations can coexist in the same configuration file. Simply add both configuration types:

```yaml
type: generic.config.ocm.software/v1
configurations:
  - type: credentials.config.ocm.software
    repositories:
      - repository:
          type: DockerConfig/v1
          dockerConfigFile: "~/.docker/config.json"
    consumers:
      - identity:
          type: OCIRegistry
          hostname: myregistry.example.com
        credentials:
          - type: Credentials
            properties:
              username: my-user
              password: my-token
  - type: resolvers.config.ocm.software/v1alpha1
    resolvers:
      - repository:
          type: OCIRegistry/v1
          baseUrl: ghcr.io
          subPath: open-component-model/ocm
        componentNamePattern: "ocm.software/*"
      - repository:
          type: OCIRegistry/v1
          baseUrl: myregistry.example.com
          subPath: my-org/components
        componentNamePattern: "mycompany.io/*"
```

This gives the CLI both the **routing** (which registry to use) and the **authentication** (how to log in) it needs.

For more details on configuring credentials, see [Credentials in an .ocmconfig File]({{< relref "creds-in-ocmconfig.md" >}}).

## Recursive Resolution

Resolvers are especially valuable when working with components that **reference other components** stored in different registries. The `--recursive` flag on commands like `ocm get cv` or `ocm transfer cv` follows these references, and resolvers ensure each referenced component is looked up in the correct repository.

For example, consider a component `mycompany.io/my-app:1.0.0` that references `ocm.software/ocmcli:0.23.0`. With the multi-registry configuration above, the CLI will:

1. Look up `mycompany.io/my-app:1.0.0` in `myregistry.example.com/my-org/components`
2. Discover the reference to `ocm.software/ocmcli:0.23.0`
3. Automatically look up `ocm.software/ocmcli:0.23.0` in `ghcr.io/open-component-model/ocm`

```bash
ocm get cv mycompany.io/my-app:1.0.0 --recursive -1
```

Without resolvers, you would need to specify each repository manually.

{{<callout context="note">}}
For `ocm get cv`, the `--recursive` flag accepts a depth value: `0` for no recursion (default), `-1` for unlimited depth, or any positive integer for a specific depth limit.
{{</callout>}}

## CLI and Resolver Interaction

When you provide both a repository reference on the command line and have resolvers configured, the CLI uses the following priority order:

1. **Command-line reference** (highest priority) — the repository specified directly in the command
2. **Configured resolvers** — resolvers from the configuration file, matched by component name pattern
3. **Command-line repository as fallback** — if no resolver matches, the repository from the command line acts as a catch-all

This means you can always override resolver behavior by specifying a repository explicitly:

```bash
# Uses resolvers from config
ocm get cv ocm.software/ocmcli:0.23.0

# Explicitly targets a specific repository, ignoring resolvers
ocm get cv ghcr.io/my-mirror//ocm.software/ocmcli:0.23.0
```

## Transferring Components

Resolvers work with transfer commands as well. When transferring component versions, the resolver is used to locate the **source** components:

```bash
# The resolver routes ocm.software/* to the correct source registry
ocm transfer cv ocm.software/ocmcli:0.23.0 myregistry.example.com/target
```

With recursive transfers, the resolver ensures all referenced components are discovered across registries:

```bash
ocm transfer cv mycompany.io/my-app:1.0.0 myregistry.example.com/target --recursive
```

## Configuration Reference

The resolver configuration is defined by the `resolvers.config.ocm.software/v1alpha1` type in the [OCM specification](https://github.com/open-component-model/open-component-model/tree/main/bindings/go/configuration/resolvers/v1alpha1/spec).

### Config Schema

| Field       | Type   | Required | Description                                        |
|-------------|--------|----------|----------------------------------------------------|
| `type`      | string | Yes      | Must be `resolvers.config.ocm.software/v1alpha1`.  |
| `resolvers` | array  | No       | List of resolver entries.                          |

### Resolver Schema

| Field                  | Type   | Required | Description                                                                                  |
|------------------------|--------|----------|----------------------------------------------------------------------------------------------|
| `repository`           | object | Yes      | An OCM repository specification (must include a `type` field).                               |
| `componentNamePattern` | string | No       | Glob pattern for matching component names. If omitted, the resolver matches all components.  |
