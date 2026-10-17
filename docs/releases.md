# Releases

## Official Release Page

<https://github.com/InfraMeshLabs/infra-deploy/releases>

GitHub Releases of this repository are the official distribution channel for InfraMesh Console.

## Release Policy

- Every release is published on the Releases page of this repository.
- Standalone packages are attached to the release as **Release Assets**. They are never committed to this Git repository.
- Each package is published together with a `.sha256` checksum file.
- Docker images are published to the GitHub Container Registry as `ghcr.io/inframeshlabs/infra-console:<version>`.
- Releases are versioned by Git tag (`v<version>`, for example `v0.2.0`). Package file names and Docker image tags use the same version without the `v` prefix.
- A published release's assets are not replaced. Fixes are shipped as a new version.

## Release Assets

Each release with a Standalone Distribution contains six assets:

| Asset | Platform |
| --- | --- |
| `inframesh-console-<version>-macos-arm64.tar.gz` | macOS, Apple Silicon |
| `inframesh-console-<version>-macos-arm64.tar.gz.sha256` | Checksum |
| `inframesh-console-<version>-macos-x64.tar.gz` | macOS, Intel |
| `inframesh-console-<version>-macos-x64.tar.gz.sha256` | Checksum |
| `inframesh-console-<version>-windows-x64.zip` | Windows x64 |
| `inframesh-console-<version>-windows-x64.zip.sha256` | Checksum |

Each `.sha256` file contains one line in the standard `sha256sum` format:

```text
<sha256-hex>  inframesh-console-<version>-<platform>.<ext>
```

Verify a download with `shasum -a 256 -c <file>.sha256` on macOS, or `Get-FileHash` on Windows. See [Standalone Installation](console/standalone.md).

The Docker deployment does not use release assets. It pulls the image tag set by `INFRA_CONSOLE_VERSION` in `console/.env`. See [Upgrading](console/upgrade.md#docker).

## How Releases Are Produced

The Console is built and tested in a separate private repository. This repository only publishes the results.

```text
infra-console (private)
    └── Build / Test / Package

infra-deploy (public)
    └── Download / Installation / Deployment
```

Release assets are uploaded and published by an automated release process. This repository contains no workflow that builds Console packages or images, and none that modifies published release assets.

## Release Notes

Release notes are written in English and describe only what users need to install, configure and upgrade:

- New features and changes visible to users
- Supported platforms and assets
- Installation and upgrade notes
- Known limitations

Public release notes do not include links to private repositories, internal issues or pull requests, private infrastructure details, credentials, or internal build paths.

## Language

README, installation guides, upgrade guides, release notes and all other public documentation in this repository are written in English.
