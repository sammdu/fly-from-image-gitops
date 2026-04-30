# Deploy Docker Images to Fly.io with GitHub Actions

Template for deploying a Docker image to [Fly.io](https://fly.io) from GitHub Actions without installing `flyctl` locally.

## Quick Start

1. Fork or clone this repository.
2. Edit `fly.json`:
   - Set `app` to your Fly app name.
   - Set `primary_region` to a Fly region code.
   - Set `build.image` to an existing image, or replace it with `build.dockerfile` for local builds.
   - Set `http_service.internal_port` to the port your container listens on.
3. Override `FLY_ORG` in the workflow `env` block if you do not deploy to your personal org.
4. Add the GitHub Actions secret `FLY_API_TOKEN`.
5. Run the `Fly Bootstrap` workflow manually.
6. Run `Deploy App` manually, or uncomment the `push` trigger in `.github/workflows/fly-deploy.yml`.

## Workflows

- `.github/workflows/fly-bootstrap.yml` creates the Fly app, creates configured volumes, syncs configured secrets, builds an image when needed, and deploys.
- `.github/workflows/fly-deploy.yml` syncs configured secrets, builds an image when needed, and deploys. Push deploys are disabled until the commented `push` trigger is enabled.

The workflows are intentionally thin. Implementation lives in composite actions under `.github/actions/fly-*`.

Bootstrap and deploy create a WireGuard peer named from the app prefix plus short commit hash when no matching app-prefixed peer exists. The recommended config filename is printed as `<peer>.conf`. Run the `Fly WireGuard Config` workflow manually with `recreate=true` to delete matching peers and print a fresh config; set `peer_name` or `match_prefix` to override the defaults.

Optional maintenance actions are included but not active by default. Uncomment the example steps in `fly-deploy.yml` only for apps that need them.

## Configuration

Non-secret configuration is committed in this repo:

- `fly.json` is the Fly app config source.
- `FLY_ORG` defaults to `personal` in `fly-setup`; override it in workflow `env` when needed.
- `deploy.strategy` is `rolling` with `max_unavailable=1` by default.
- `enable-ha-with-no-volumes` is set on the `fly-deploy` action; keep it `false` for lowest idle cost, or set it `true` for Fly's default spare Machine behavior.

For private apps, set `FLY_ACCESS_MODE` once in the workflow `env`:

- `flycast`: use `.github/actions/fly-private-network` with `allocate: "true"` during bootstrap.
- `internal`: use `.github/actions/fly-private-network` only to fail if public IPs are attached.

The access URL action auto-detects existing Flycast or public IPs when `FLY_ACCESS_MODE` is not set; otherwise it falls back to `.internal`.

Secrets stay in GitHub Actions secrets:

- `FLY_API_TOKEN` is required.
- App secrets are optional. Map them once in `.github/workflows/fly-set-secrets.yml`.

Example app secret mapping:

```yaml
- uses: ./.github/actions/fly-sync-secrets
  with:
      stage: ${{ inputs.stage }}
      secrets: |
          APP_PASSWORD=${{ secrets.APP_PASSWORD }}
          API_KEY=${{ secrets.API_KEY }}
```

## Image Modes

Use an existing image:

```json
"build": {
    "image": "your-registry/your-image:tag"
}
```

Build and push from this repository:

```json
"build": {
    "dockerfile": "Dockerfile",
    "context": "."
}
```

## Optional Volumes

Add `mounts` to `fly.json` when the app needs persistent storage. The bootstrap workflow creates missing volumes in `primary_region`.

```json
"mounts": [
    {
        "destination": "/app/data",
        "initial_size": "1gb",
        "scheduled_snapshots": true,
        "snapshot_retention": 7,
        "source": "app_data"
    }
]
```

No default workflow path destroys machines or volumes.

## Optional Maintenance

The template keeps cleanup components for apps that need them. They are commented out in `fly-deploy.yml` and each destructive action requires `confirm: "true"`.

- `fly-scale-processes`: enforces process group Machine counts such as `web=1`; useful after disabling deploy HA for service groups without volumes.
- `fly-cleanup-volumes`: lists unattached volumes and deletes them when confirmed. Set `orphaned: "true"` to also snapshot and remove attached volumes whose names are no longer declared in `fly.json.mounts`; this also destroys the attached machine.
- `fly-cleanup-machines`: for apps using `fly.json.experimental.machine_config`, keeps the oldest active machine and removes replaced or duplicate machines.

For a Fly-volume app, keep `fly-provision` and consider `fly-cleanup-volumes`. For a multi-container or machine-config app, consider `fly-cleanup-machines`. Delete unused optional actions from downstream app repos if they are not relevant.

## Documentation

- [Fly.io Docs](https://fly.io/docs/)
- [Fly.io Regions](https://fly.io/docs/reference/regions/)
- [Fly.io Configuration Reference](https://fly.io/docs/reference/configuration/)
- [GitHub Actions](https://docs.github.com/actions)
