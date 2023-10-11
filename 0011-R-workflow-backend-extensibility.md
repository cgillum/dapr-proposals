# Dapr Workflow Backend Extensibility

* Author(s): Chris Gillum (@cgillum)
* State: Draft
* Updated: 10/10/2023

## Overview

This is a proposal for allowing users to configure alternate backend implementations for Dapr Workflow, beyond the default Actors backend.

## Background

Dapr Workflow relies on the Durable Task Framework for Go (a.k.a. durabletask-go) as the core engine for executing workflows. This engine is designed to support multiple backend implementations. For example, the durabletask-go repo includes a SQLite implementation and a Postgres implementation, and the Dapr repo includes an Actors implementation.

The choice of backend implementation does NOT have any impact on the code developers write. The backend implementation instead impacts how workflow state is stored and how workflows execution is coordinated across replicas. In that sense, it is similar to Dapr's state store abstraction, except optimized for workflow.

## Motivation

The primary motivation is to allow vendors to provide proprietary backend implementations as part of their own Dapr distributions. For example, a vendor may want to provide a backend implementation that stores workflow state in a fully managed, proprietary database. This would allow users to leverage the vendor's database without having to write any code.

The secondary motivation is to allow users to choose a backend implementation that best fits their needs based on what's currently available in durabletask-go. For example, a user may want to use the existing Durable Task Postgres backend implementation so that they can run arbitrary SQL queries on their workflow data, which isn't always practical with actor state stores.

## Example: Postgres Backend

As an example, let's consider a Postgres backend implementation. This implementation would be based on the existing [durabletask-go Postgres backend](https://github.com/microsoft/durabletask-go/tree/postgres/backend/postgres) and would serve users that need to run arbitrary queries on their workflow data.

To use the Postgres implementation, users would need to provide a Postgres connection string in their Dapr configuration as part of a new `workflow` configuration section:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprConfig
spec:
  workflow:
    backendType: "postgres"
    maxConcurrentWorkflows: 255
    maxConcurrentActivities: 255
    storeConfiguration:
      connectionString: "postgres://postgres:secret@localhost:5432/workflowdb?sslmode=disable"
```

## Implementation Details

This feature requires a relatively small set of changes to the Dapr runtime:

* Add a new OPTIONAL `workflow` configuration section to the Dapr configuration schema.
* Refactor the existing `wfengine` code to decouple it from the Actors runtime.
* Update the `wfengine` code to select an alternate backend implementations based on the value of `workflow.backendType` in the Dapr configuration.
* Update the runtime startup logic to initialize internal actors only if the Actors backend (default) is configured.
* Update to the latest version of `durabletask-go`, which includes the Postgres backend implementation.

A relatively small number of documentation updates will also be required to reflect the fact that users can now configure alternate backend implementations.

## FAQ

### Will users be able to author their own backend implementations?

Yes and no. The workflow backend will not be "pluggable" in the sense that users can write their own backend implementations and plug them into Dapr. Dapr OSS will be hardcoded to only support a fixed set of backend implementations (for example, Actors and Postgres). However, users will be able to write their own backend implementations and include them in their own Dapr distributions. For example, a vendor may want to provide a proprietary backend implementation as part of their own Dapr distribution.

### Will this allow users to implement their own workflow engine?

No. The durabletask-go library will continue to be the basis for the Dapr Workflow feature. This proposal only impacts the backend used by durabletask-go.

### Are changes required for the existing Actors backend?

No. The Actors backend will continue to be the default backend implementation and does not require any changes at all. This proposal is purely additive to the existing Dapr Workflow feature.

### How does this relate to the Dapr State Store abstraction?

This proposal allows users to configure workflow backends that don't have any dependency on the State Store abstraction.

### How does this relate to the Dapr Actor runtime?

This proposal allows users to configure workflow backends that don't have any dependency on the Actor runtime. If an alternate backend implementation is configured, then the internal actors will not be registered with the Actor runtime.

### Does this impact Dapr Workflow APIs?

Yes and no. No changes are required to the existing workflow APIs to support alternate backends. However, alternate backend implementations may change the behavior of the existing APIs in subtle ways since workflow backend code is often invoked by workflow APIs. For example, a Postgres backend implementation may have different error conditions than the Actors backend implementation.

### Does this impact Dapr Workflow SDKs or existing user code?

No. Alternate backends don't require any changes to Dapr Workflow SDKs, nor do they require any changes to user code. A correctly implemented alternate backend should also not change the behavior of existing user code in any meaningful way.

### Can users migrate workflow data from one backend to another?

No. This proposal does not include any migration tooling. Users would need to write their own migration tooling if they want to migrate workflow data from one backend to another.

### Does this feature help users manage the infrastructure of the alternate backends?

No. Users must manage the infrastructure of the alternate backends themselves. For example, if a user chooses to use the Postgres backend, then they must create and manage the Postgres database themselves.
