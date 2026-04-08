# Deploy Docker Images to Fly.io via GitHub Actions

Template for deploying Docker images to [Fly.io](https://fly.io) using GitHub Actions, without installing `flyctl` locally.

## Quick Start

1. **Fork/clone this repository**

2. **Configure** - Edit these files:
   - **`fly.json`**: Set `app` name, `primary_region` ([region code](https://fly.io/docs/reference/regions/)), `build.image`, and `http_service.internal_port`
   - **`.github/actions/fly-setup/action.yml`**: Set `FLY_ORG` to your Fly.io organization slug
   - **`.github/workflows/fly-set-secrets.yml`**: Add your app secrets (if needed)

3. **Add GitHub Secret** - In repo settings (Settings > Secrets > Actions):
   - `FLY_API_TOKEN`: Generate at [fly.io/user/personal_access_tokens](https://fly.io/user/personal_access_tokens)

4. **Bootstrap & Deploy**:
   - Run "Fly Bootstrap" workflow manually (creates app & volumes)
   - In `.github/workflows/fly-deploy.yml`, set `AUTO_RUN_ON_PUSH` to `"true"` to enable auto-deploy on push to `main`

## Configuration

### fly.json
- `app`: App name on Fly.io
- `primary_region`: 3-letter [region code](https://fly.io/docs/reference/regions/)
- `build.image`: Docker image (any registry); when set, image is pulled locally and deployed with `--local-only`
- `env`: Environment variables
- `http_service.internal_port`: Container port
- `mounts`: Persistent volumes (optional)
  - Volume names: lowercase, numbers, underscores only (max 30 chars)
  - Use `app_data` not `app-data`

See [Fly.io config reference](https://fly.io/docs/reference/configuration/) for all options.

### fly-setup action
`FLY_ORG` is set directly in `.github/actions/fly-setup/action.yml`. `FLY_APP_NAME` and `FLY_PRIMARY_REGION` are read from `fly.json`.

### Workflows
- **[fly-deploy.yml](.github/workflows/fly-deploy.yml)**: Auto-deploys on push to `main` (set `AUTO_RUN_ON_PUSH: "true"` to enable)
- **[fly-bootstrap.yml](.github/workflows/fly-bootstrap.yml)**: Manual initial setup
- **[fly-set-secrets.yml](.github/workflows/fly-set-secrets.yml)**: Secrets deployment (customize as needed)
- **[fly-cleanup-volumes.yml](.github/workflows/fly-cleanup-volumes.yml)**: Delete unattached volumes

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
