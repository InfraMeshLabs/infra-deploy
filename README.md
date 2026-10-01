# InfraMesh Deployment

Deployment configurations and guides for running the **InfraMesh platform**.

This repository serves as the public deployment entry point for InfraMesh and will provide the resources required to deploy and operate InfraMesh components without requiring access to their internal source repositories.

## Overview

InfraMesh is a distributed AI inference platform designed to connect and utilize heterogeneous GPU and CPU resources across multiple nodes.

The platform is composed of the following components:

```text
                         InfraMesh
                             │
                          Console
                             │
                    ┌────────┴────────┐
                    │                 │
                 Router            Worker
                    │                 │
                    └────────┬────────┘
                             │
                         AI Runtime
```

- **Console** — Control plane and inference orchestration layer.
- **Node SDK** — Common contracts and integrations used by Router and Worker nodes.
- **Router** — Selects an appropriate Worker for an inference request.
- **Worker** — Executes AI inference using the configured runtime.

This repository is intended to provide deployment resources for running these components together.

## Repository Status

> **Deployment resources are currently under development.**

Docker images, Docker Compose configurations, and production deployment guides have not yet been published.

The repository is being prepared as the public deployment and installation entry point for InfraMesh.

## Planned Deployment Support

Future deployment resources may include:

- Docker images
- Docker Compose configurations
- Environment configuration examples
- PostgreSQL configuration
- Redis configuration
- Console deployment
- Router and Worker deployment examples
- Upgrade and migration guides
- Production deployment recommendations

Additional deployment options may be introduced as the platform evolves.

## Planned Structure

```text
infra-deploy
├── README.md
├── .env.example
├── docker-compose.yml
└── docs/
    ├── installation.md
    ├── configuration.md
    └── upgrade.md
```

The actual structure may change as deployment support is implemented.

## Related Projects

| Project | Description |
| --- | --- |
| `infra-node` | Common SDK and contracts for InfraMesh nodes |
| `infra-router` | Reference implementation for Router nodes |
| `infra-worker` | Reference implementation for Worker nodes |

## Documentation

Detailed installation and deployment documentation will be added as deployment support becomes available.

For general information about InfraMesh, see the InfraMesh organization repositories and project documentation.

## License

Licensed under the **Apache License 2.0**.
