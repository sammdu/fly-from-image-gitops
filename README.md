# Deploy Docker Images to Fly.io with GitHub Actions

Template for deploying a Docker image to [Fly.io](https://fly.io) from GitHub Actions without installing `flyctl` locally.

For LLM agents adapting this template, read [AGENTS.md](AGENTS.md) first. It captures Fly.io deployment reasoning, common pitfalls, and when to verify behavior against current docs.

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

- `fly-bootstrap.yml`: creates the app, provisions volumes, syncs secrets, builds when needed, deploys, and prints WireGuard access when applicable.
- `fly-deploy.yml`: syncs secrets, builds when needed, and deploys. Push deploys are disabled until the commented trigger is enabled.
- `fly-wireguard.yml`: recreates or prints private access config when needed.

The workflows stay thin; reusable behavior lives in composite actions under `.github/actions/fly-*`. Optional maintenance steps are commented out in `fly-deploy.yml`.

## Configuration

- `fly.json` is the Fly app config source.
- `FLY_ORG` defaults to `personal` in `fly-setup`; override it in workflow `env` when needed.
- Deploys default to rolling with `max_unavailable=1`.
- `enable-ha-with-no-volumes` controls whether Fly creates spare Machines for service groups without volumes.

For private apps, set `FLY_ACCESS_MODE` once in the workflow `env`:

- `flycast`: use `.github/actions/fly-private-network` with `allocate: "true"` during bootstrap.
- `internal`: use `.github/actions/fly-private-network` only to fail if public IPs are attached.

`FLY_API_TOKEN` is required. Map app secrets once in `.github/workflows/fly-set-secrets.yml`.

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

Add `mounts` to `fly.json` when the app needs persistent storage. Bootstrap creates missing volumes in `primary_region`.

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

Maintenance actions are available but inactive by default. Destructive actions require `confirm: "true"`.

- `fly-scale-processes`: enforces process group Machine counts such as `web=1`; useful after disabling deploy HA for service groups without volumes.
- `fly-cleanup-volumes`: lists unattached volumes and deletes them when confirmed. Set `orphaned: "true"` to also snapshot and remove attached volumes whose names are no longer declared in `fly.json.mounts`; this also destroys the attached machine.
- `fly-cleanup-machines`: for apps using `fly.json.experimental.machine_config`, removes replaced or duplicate machines.

## Documentation

- [Fly.io Docs](https://fly.io/docs/)
- [Fly.io Regions](https://fly.io/docs/reference/regions/)
- [Fly.io Configuration Reference](https://fly.io/docs/reference/configuration/)
- [GitHub Actions](https://docs.github.com/actions)
