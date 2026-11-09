# Fly.io GitOps App Template

Lightweight template for deploying apps to [Fly.io](https://fly.io) from GitHub Actions. Configure [fly.json](fly.json) and secrets, run [Fly.io Infra](.github/workflows/fly-infra.yml) to provision the app, then use [Fly.io Deploy](.github/workflows/fly-deploy.yml) for further deployments.

For LLM agents adapting this template, read [AGENTS.md](AGENTS.md) first.

## Quick Start

1. Edit [fly.json](fly.json): set `app`, `primary_region`, deployment type, and networking services and ports.
2. Add the GitHub Actions secret `FLY_API_TOKEN` in [Actions secrets settings](../../settings/secrets/actions).
3. Set these values in [Fly.io Infra](.github/workflows/fly-infra.yml):
    - `FLY_ORG`: `personal` by default.
    - `FLY_ACCESS_MODE`: `internal`, `flycast`, or `public` (default); unset preserves existing access.
4. Add app secrets to [Fly.io Secrets](.github/workflows/fly-secrets.yml).
5. Run [Fly.io Infra](.github/workflows/fly-infra.yml). Pass process groups needed by `release_command` in `release_dependency_process_groups`.

### Subsequent Deployments

- Run [Fly.io Deploy](.github/workflows/fly-deploy.yml) after changing app code, images, configuration, or secrets.
- To deploy on pushes, uncomment its `push` trigger and set the deployment branch.

### Optional Deployment Steps

In [Fly.io Deploy](.github/workflows/fly-deploy.yml), uncomment the steps you need:

- [Fly.io Scale Processes](.github/actions/fly-scale-processes/action.yml) sets Machine counts, such as `counts: web=2 worker=1`.
- [Fly.io Cleanup Machines](.github/actions/fly-cleanup-machines/action.yml) removes extra Machines for single-Machine apps using `experimental.machine_config`, keeping the oldest non-replaced Machine.
- [Fly.io Cleanup Volumes](.github/actions/fly-cleanup-volumes/action.yml) removes unattached or stuck volumes after deployment.

Cleanup steps preview changes unless `confirm: "true"` enables deletion.

## Deployment Types

Pick one in [fly.json](fly.json):

- Prebuilt image: `"build": { "image": "registry.example.com/app:tag" }`
- Dockerfile: `"build": { "dockerfile": "Dockerfile", "context": "." }`
- Docker Compose: `"build": { "compose": { "file": "compose.yaml" } }`
- Custom multi-container Machine: `"experimental": { "machine_config": "cli-config.json" }`

### Merge Multiple Compose Files

- Configure [Fly.io Merge Compose](.github/workflows/fly-merge-compose.yml) with upstream `urls` and `local` overrides. The file name in `output` should match `build.compose.file` in [fly.json](fly.json).
- Use `env-file` (default `compose.fly.env`) to configure environment variables and `unescape: "true"` to convert `$$` to `$`.
- In [Fly.io Deploy](.github/workflows/fly-deploy.yml), enable the `compose` job, add it to `deploy.needs`, and enable the artifact download/decryption steps. The merged file is encrypted between workflow steps in case you choose to inject secrets.
- Declare persistent storage in [fly.json](fly.json) `mounts`. Compose volumes do not provision Fly.io Volumes; delete existing service `volumes` entries or clear them in a local override with `volumes: !override []`.

### Process Groups

If your Docker image contains multiple programs you want to run simultaneously on separate Machines, declare them in the `processes` section of [fly.json](fly.json). Each group runs its own command from the same image and can scale independently. Read more about [multiple processes on Fly.io](https://docs.fly.io/launch/processes).

For example, an image containing `./bin/web` and `./bin/worker` could run both while routing HTTP requests only to `web`:

```json
{
    "processes": {
        "web": "./bin/web",
        "worker": "./bin/worker"
    },
    "http_service": {
        "internal_port": 8080,
        "processes": ["web"]
    }
}
```

Use `processes` lists in `services`, `mounts`, and `vm` to select which groups receive those settings.

### Deploy Readiness

- Service `checks` control traffic routing and deployment updates after a Machine starts.
- `machine_checks` probe a started target from an ephemeral test Machine during rolling and canary deployments.
- Checks do not start stopped Machines. Set `start-process-groups` in [Fly.io Deploy](.github/actions/fly-deploy/action.yml) to start groups before deployment checks.
- This template sets `kill_signal` to `SIGTERM`; Fly.io defaults to `SIGINT`.

See [AGENTS.md](AGENTS.md) for readiness and shutdown guidance.

## Secrets

Go to [Actions secrets settings](../../settings/secrets/actions) to configure your secrets (`FLY_API_TOKEN` is required), then add app secrets to [Fly.io Secrets](.github/workflows/fly-secrets.yml):

```yaml
- uses: ./.github/actions/fly-sync-secrets
  with:
      stage: ${{ inputs.stage }}
      secrets: |
          APP_PASSWORD=${{ secrets.APP_PASSWORD }}
```

[Fly.io Infra](.github/workflows/fly-infra.yml) imports secrets with `stage: false`; standalone [Fly.io Deploy](.github/workflows/fly-deploy.yml) stages secrets before deploying.

## Volumes

Add `mounts` in [fly.json](fly.json) for persistent storage; [Fly.io Infra](.github/workflows/fly-infra.yml) creates missing volumes in `primary_region`.

```json
"mounts": [
    {
        "source": "app_data",
        "destination": "/app/data",
        "initial_size": "1gb",
        "scheduled_snapshots": true,
        "snapshot_retention": 7
    }
]
```

## Private Apps

Set `FLY_ACCESS_MODE` in [Fly.io Infra](.github/workflows/fly-infra.yml) before provisioning:

| Mode | Access | Address |
| --- | --- | --- |
| `public` (default) | Public internet through configured services | `<app>.fly.dev` |
| `flycast` | Private services through the Fly.io proxy, with automatic stop and start when configured | `<app>.flycast` |
| `internal` | Direct private IPv6 (6PN) connections to running Machines | `<app>.internal:<port>` |

- For [Flycast](https://docs.fly.io/networking/flycast), configure `http_service` or `services` in [fly.json](fly.json) and bind the listener to `0.0.0.0`.
- For [internal 6PN access](https://docs.fly.io/networking/private-networking), bind the listener to `fly-local-6pn` or `::` and connect to its listening port. Direct connections do not wake stopped Machines.
- New private apps use the `app-<app name>` network. [Fly.io Deploy](.github/workflows/fly-deploy.yml) preserves existing IP allocations.

### Connect Through WireGuard

Use a WireGuard VPN to access private apps from your computer:

1. Run [Fly.io Infra](.github/workflows/fly-infra.yml) with `flycast` or `internal` access. It vends a config when the app's client network has no peer other than `interactive-` peers.
2. Open the completed run in [GitHub Actions](../../actions/workflows/fly-infra.yml), download `wireguard-<peer name>` under **Artifacts**, and extract the `.conf` file. Artifacts expire after one day.
3. Import the file into your WireGuard client, activate the tunnel, and open the app's private address above.

If a peer already exists, use its saved config. For additional configs or peer changes, see [Private Networking](#private-networking).

## Maintenance Workflows

### Resize Volumes

To resize a volume, run [Fly.io Resize Volume](.github/workflows/fly-resize-volume.yml) with a Machine ID and size in whole GB (1-500).

- Growing extends storage; follow any restart instructions.
- Shrinking replaces the Machine and volume, causing downtime and changing Machine IDs. Inspect failures before retrying.

### Clean Up Volumes

To reclaim storage, run [Fly.io Cleanup Volumes](.github/workflows/fly-cleanup-volumes.yml):

- `confirm`: delete; default: preview.
- `volume`: delete immediately, ignoring `confirm`.
- `orphaned`: snapshot, then delete volumes absent from [fly.json](fly.json) and their Machines.

### Restore Volume Snapshots

To recover data, run [Fly.io Restore Volume Snapshot](.github/workflows/fly-restore-volume-snapshot.yml) with `snapshot_id`. Restoration creates a volume; `replace` also replaces the original Machine.

### Migrate Regions

To relocate Machines and volumes, set `primary_region` in [fly.json](fly.json), then run [Fly.io Migrate Region to Match fly.json](.github/workflows/fly-migrate-region.yml). Preview first; enable `confirm` to migrate. Migration requires downtime and changes Machine IDs.

### Private Networking

To inspect networks or manage WireGuard peers, run [Fly.io Private Networking](.github/workflows/fly-private-networking.yml).

- Choose a network or peer operation. Set `peer_name` to name or revoke a peer; leave blank to generate a name when vending.
- To replace a config, vend another, verify access, then revoke the old peer.

To change access mode, set `FLY_ACCESS_MODE` in [Fly.io Infra](.github/workflows/fly-infra.yml) and rerun it. Private modes remove public IPs; `internal` requires the `app-<app name>` Machine network, which cannot be changed.

### Update Secrets

To sync secrets, run [Fly.io Secrets](.github/workflows/fly-secrets.yml). `stage` enabled: next deployment; disabled: apply immediately and restart Machines.

## Documentation

- [Fly.io Docs](https://fly.io/docs/) · [Configuration Reference](https://fly.io/docs/reference/configuration/) · [Multi-container Machines](https://fly.io/docs/machines/guides-examples/multi-container-machines/)
