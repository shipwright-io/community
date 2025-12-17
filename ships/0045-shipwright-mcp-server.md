<!--
Copyright The Shipwright Contributors

SPDX-License-Identifier: Apache-2.0
-->

---
- title: shipwright-mcp-server
- authors:
  - "@ayushsatyam"
- reviewers:TBD
- approvers: TBD
- creation-date: 2025-12-17
- last-updated: 2025-12-17
= status: provisional
- see-also:
- replaces: []
- superseded-by: []
---

# Shipwright MCP Server

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [docs](/docs/)

## Summary

This proposal introduces a Model Context Protocol (MCP) server for the Shipwright Build project. The MCP server exposes Shipwright Build functionality through standardized tools that can be consumed by AI assistants, IDE integrations, chatbots, and other MCP-compatible clients. This enables AI-assisted DevOps workflows where developers can interact with Shipwright resources using natural language through tools like Claude, Cursor, and other AI-powered development environments.

The MCP server provides a complete set of tools for managing Shipwright resources:
- **Build Management**: Create, list, get, and delete Build resources
- **BuildRun Management**: Create, list, get, restart, and delete BuildRun resources
- **Strategy Discovery**: List BuildStrategies and ClusterBuildStrategies

This enhancement positions Shipwright as an AI-native container image build platform, meeting developers where they are increasingly working—within AI-assisted development environments.

## Motivation

The developer experience landscape is rapidly evolving with AI-assisted development becoming mainstream. Tools like GitHub Copilot, Cursor, and Claude are fundamentally changing how developers interact with their development environments and infrastructure. The Model Context Protocol (MCP), introduced by Anthropic, provides a standardized way for AI systems to interact with external tools and data sources.

Currently, Shipwright users interact with the build system through:
- `kubectl` commands
- The `shp` CLI tool
- Direct API calls via client libraries
- GitOps workflows with declarative manifests

While these interfaces are powerful, they require developers to context-switch away from their AI-assisted workflows. As AI assistants become the primary interface for many development tasks, Shipwright needs a native integration point.

An MCP server for Shipwright enables:
- **Natural language build management**: Developers can ask their AI assistant to "create a build for my Go application using the buildah strategy"
- **Intelligent troubleshooting**: AI can query build status, analyze failures, and suggest remediations
- **Workflow automation**: AI agents can orchestrate complex build workflows without manual intervention
- **Reduced cognitive load**: Developers don't need to remember specific API fields or CLI flags

### Goals

1. Provide a standardized MCP interface for Shipwright Build resources
2. Enable AI assistants to create, manage, and monitor builds through natural language
3. Support both reference-based and inline BuildRun creation for flexible build workflows
4. Expose build strategy discovery to help users understand available build options
5. Provide comprehensive error handling with actionable feedback for AI consumption
6. Support deployment both inside and outside Kubernetes clusters
7. Maintain compatibility with the Shipwright v1beta1 API

### Non-Goals

1. Replace existing interfaces (CLI, kubectl, client libraries)
2. Implement MCP resources or prompts (only tools are in scope for initial implementation)
3. Provide build log streaming (future enhancement)
4. Implement authentication/authorization beyond Kubernetes RBAC
5. Support Shipwright v1alpha1 API (deprecated)
6. Provide direct Tekton resource management

## Proposal

Introduce a new project `shipwright-io/mcp-server` (or integrate into the main build repository) that implements an MCP server exposing Shipwright Build operations as standardized tools. I recommend a seperate reposditory.

### Example Usage

The following examples demonstrate how developers interact with Shipwright through an AI assistant using the MCP server:

**Build Creation:**
```
Developer: Create a build for my Node.js app at github.com/myorg/myapp 
           using buildah, output to quay.io/myorg/myapp:latest

AI Assistant: [Calls create_build tool]
Successfully created Build 'myapp-build' in namespace 'default'
```

**Build Troubleshooting:**
```
Developer: Why did my build fail?

AI Assistant: [Calls get_buildrun tool]
Your build failed: BuildRunTimeout - exceeded configured timeout of 10m.
Would you like me to restart with a longer timeout?
```

**Strategy Discovery:**
```
Developer: What build strategies are available?

AI Assistant: [Calls list_clusterbuildstrategies]
Available: buildah, buildpacks-v3, kaniko, source-to-image
```

### Implementation Notes

#### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MCP Clients                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐   │
│  │   Claude    │  │   Cursor    │  │ Slack/Teams │  │   Custom AI      │   │
│  │  Desktop    │  │    IDE      │  │    Bots     │  │    Agents        │   │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └────────┬─────────┘   │
└─────────┼────────────────┼────────────────┼───────────────────┼─────────────┘
          │                │                │                   │
          │         stdio/SSE Transport (MCP Protocol)          │
          │                │                │                   │
          └────────────────┴────────┬───────┴───────────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │   Shipwright MCP Server       │
                    │                               │
                    │  ┌─────────────────────────┐  │
                    │  │     Tool Handlers       │  │
                    │  │                         │  │
                    │  │ • list_builds           │  │
                    │  │ • get_build             │  │
                    │  │ • create_build          │  │
                    │  │ • delete_build          │  │
                    │  │ • list_buildruns        │  │
                    │  │ • get_buildrun          │  │
                    │  │ • create_buildrun       │  │
                    │  │ • restart_buildrun      │  │
                    │  │ • delete_buildrun       │  │
                    │  │ • list_buildstrategies  │  │
                    │  │ • list_clusterbuild...  │  │
                    │  └───────────┬─────────────┘  │
                    │              │                │
                    │  ┌───────────▼─────────────┐  │
                    │  │   Kubernetes Client     │  │
                    │  │   (controller-runtime)  │  │
                    │  └───────────┬─────────────┘  │
                    └──────────────┼────────────────┘
                                   │
                                   │ Kubernetes API
                                   │
                    ┌──────────────▼────────────────┐
                    │      Kubernetes Cluster       │
                    │                               │
                    │  ┌─────────────────────────┐  │
                    │  │  Shipwright Build APIs  │  │
                    │  │                         │  │
                    │  │  • Build                │  │
                    │  │  • BuildRun             │  │
                    │  │  • BuildStrategy        │  │
                    │  │  • ClusterBuildStrategy │  │
                    │  └─────────────────────────┘  │
                    └───────────────────────────────┘
```

#### MCP Tools Specification

The server implements 11 MCP tools organized by resource type:

##### Build Tools

| Tool | Description | Required Parameters | Optional Parameters |
|------|-------------|---------------------|---------------------|
| `list_builds` | List Builds in a namespace | `namespace` | `prefix`, `label-selector` |
| `get_build` | Get Build details | `name` | `namespace` |
| `create_build` | Create a new Build | `name`, `source-type`, `source-url`, `strategy`, `output-image` | `namespace`, `context-dir`, `revision`, `strategy-kind`, `parameters`, `timeout` |
| `delete_build` | Delete a Build | `name` | `namespace` |

##### BuildRun Tools

| Tool | Description | Required Parameters | Optional Parameters |
|------|-------------|---------------------|---------------------|
| `list_buildruns` | List BuildRuns in a namespace | `namespace` | `prefix`, `label-selector` |
| `get_buildrun` | Get BuildRun details | `name` | `namespace` |
| `create_buildrun` | Create a new BuildRun | `build-name` OR (`source-type`, `source-url`, `strategy`, `output-image`) | `name`, `namespace`, `parameters`, `timeout`, `service-account`, `context-dir`, `revision`, `strategy-kind` |
| `restart_buildrun` | Restart a BuildRun | `name` | `namespace` |
| `delete_buildrun` | Delete a BuildRun | `name` | `namespace` |

##### Strategy Tools

| Tool | Description | Required Parameters | Optional Parameters |
|------|-------------|---------------------|---------------------|
| `list_buildstrategies` | List namespace BuildStrategies | `namespace` | `prefix`, `label-selector` |
| `list_clusterbuildstrategies` | List ClusterBuildStrategies | none | `prefix`, `label-selector` |

#### Server Implementation

The MCP server is already implemented in Go, leveraging the official MCP Go SDK. 
It is currently available at [github.com/ayushsatyam146/shipwright-mcp-server](https://github.com/ayushsatyam146/shipwright-mcp-server) and I would like to transfer 
ownership to the `github.com/shipwright-io` organization. It is still a relatively new implementation with significant improvements needed but I think it still serves as a solid POC and a good v1 for the project.

Key implementation details:

```go
// Server initialization
server := mcp.NewServer(&mcp.Implementation{
    Name:    "shipwright-build-mcp-server",
    Version: "v1.0.0",
}, nil)

// Tool registration example
mcp.AddTool(server, &mcp.Tool{
    Name:        "create_build",
    Description: "Create a new Build resource",
}, tools.CreateBuild)
```

The server uses:
- **Transport**: stdio (standard input/output) for local MCP clients
- **Kubernetes Client**: controller-runtime client with typed Shipwright APIs
- **Configuration**: Automatic detection of in-cluster config or kubeconfig file

#### Error Handling

All tools return structured error responses that AI clients can interpret:

```go
if err := k8sClient.Create(ctx, build); err != nil {
    return &mcp.CallToolResultFor[any]{
        IsError: true,
        Content: []mcp.Content{&mcp.TextContent{
            Text: fmt.Sprintf("Failed to create build: %v", err),
        }},
    }, nil
}
```

Error categories include:
- **Validation Errors**: Missing required parameters, invalid formats
- **Kubernetes Errors**: Resource not found, permission denied, quota exceeded
- **Network Errors**: Connection issues, timeout errors


## Drawbacks

1. **Additional Component**: Introduces another service to deploy and maintain alongside the Shipwright controller.

2. **AI Dependency**: Effectiveness depends on AI client capabilities and prompt engineering.

3. **Security Surface**: Exposes Kubernetes operations through a new interface that must be secured.

4. **Ecosystem Maturity**: MCP is a relatively new protocol; ecosystem is still evolving.

## Alternatives

### 1. Extend the `shp` CLI with AI Features

Instead of a dedicated MCP server, we could add natural language processing to the existing CLI.

**Pros**: Single tool, familiar interface
**Cons**: Requires LLM integration in CLI, doesn't integrate with IDE-based AI assistants

## Open Questions

1. Should the MCP server be deployed as a sidecar to the Shipwright controller or as a standalone deployment?
2. What authentication mechanisms should be supported for AI clients connecting to the MCP server?
3. Should we support additional MCP features like prompts and resources beyond tools?
