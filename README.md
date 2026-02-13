# Deploy Docker Images to Fly.io via GitHub Actions

Template for deploying Docker images to [Fly.io](https://fly.io) using GitHub Actions, without installing `flyctl` locally.

## Quick Start

1. **Fork/clone this repository**

2. **Configure environment** - Copy `.env.example` to `.env`:
   ```bash
   cp .env.example .env
   ```
   Edit `.env` with your values:
   - `FLY_APP_NAME`: Your Fly.io app name
   - `FLY_ORG`: Your Fly.io organization slug
   - `FLY_PRIMARY_REGION`: [Region code](https://fly.io/docs/reference/regions/) (e.g., `iad`, `lhr`, `syd`)

3. **Configure app** - Edit `fly.json`:
   - Set `build.image` to your Docker image (e.g., `registry.hub.docker.com/user/image:tag`)
   - Configure `env` with environment variables your app needs
   - Set `http_service.internal_port` to match your container's exposed port
   - Add `mounts` if you need persistent storage (optional)

4. **Add GitHub Secret** - In your repo settings, add:
   - `FLY_API_TOKEN`: Generate at [fly.io/user/personal_access_tokens](https://fly.io/user/personal_access_tokens)

5. **Bootstrap** - Run the "Fly Bootstrap" workflow manually to create the app and provision resources

6. **Deploy** - Push to `main` branch to trigger automatic deployment

## Workflow Files

- **[fly-deploy.yml](.github/workflows/fly-deploy.yml)**: Triggers on push to `main`
- **[fly-bootstrap.yml](.github/workflows/fly-bootstrap.yml)**: Manual workflow for initial setup
- **[fly-cleanup-volumes.yml](.github/workflows/fly-cleanup-volumes.yml)**: Manual workflow to delete unattached volumes
- **[fly.common.yml](.github/workflows/fly.common.yml)**: Shared deployment logic

## Configuration Reference

### fly.json Options

See [Fly.io configuration reference](https://fly.io/docs/reference/configuration/) for all options. Common settings:

- `build.image`: Docker image to deploy
- `env`: Environment variables
- `http_service`: HTTP service configuration
- `mounts`: Persistent volume mounts
- `vm`: VM resources (CPU/memory)
- `metrics`: Prometheus metrics endpoint (optional)

### .env Variables

- `FLY_APP_NAME`: App identifier on Fly.io
- `FLY_ORG`: Organization for billing and access control
- `FLY_PRIMARY_REGION`: Deployment region

## Features

- Automatic resource provisioning (app, volumes)
- Region migration support
- Volume management with auto-extend
- No local `flyctl` installation required
- GitOps workflow: configuration lives in git

## Documentation

- [Fly.io Docs](https://fly.io/docs/)
- [Fly.io Regions](https://fly.io/docs/reference/regions/)
- [Configuration Reference](https://fly.io/docs/reference/configuration/)
- [GitHub Actions](https://docs.github.com/actions)
