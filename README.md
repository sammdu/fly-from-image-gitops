# Fly.io GitOps App Template

Lightweight template for deploying apps to [Fly.io](https://fly.io) from GitHub Actions. Configure `fly.json` and secrets, run [`fly-release`](.github/workflows/fly-release.yml) to provision the app, then use [`fly-deploy`](.github/workflows/fly-deploy.yml) for further deployments.

For LLM agents adapting this template, read [AGENTS.md](AGENTS.md) first.

## Quick Start

1. Edit `fly.json`: set `app`, `primary_region`, deployment type, and `http_service.internal_port`.
2. Add the GitHub Actions secret `FLY_API_TOKEN`.
3. Set `FLY_ORG` in [`fly-release`](.github/workflows/fly-release.yml); it defaults to `personal`. Set `FLY_ACCESS_MODE` in [`fly-network`](.github/workflows/fly-network.yml): `internal`, `flycast`, or empty for public apps.
4. Map app secrets in [`fly-set-secrets`](.github/workflows/fly-set-secrets.yml).
5. Run [`fly-release`](.github/workflows/fly-release.yml). Pass process groups needed by `release_command` in `release_dependency_process_groups`.
6. Run [`fly-deploy`](.github/workflows/fly-deploy.yml), or uncomment its `push` trigger.

## Deployment Types

Pick one in `fly.json`:

- Prebuilt image: `"build": { "image": "registry.example.com/app:tag" }`
- Dockerfile: `"build": { "dockerfile": "Dockerfile", "context": "." }`
- Docker Compose: `"build": { "compose": { "file": "compose.yaml" } }`
- Custom multi-container Machine: `"experimental": { "machine_config": "cli-config.json" }`

**Compose**: [`fly-merge-compose`](.github/workflows/fly-merge-compose.yml) merges optional upstream `urls` and `local` override(s) into `build.compose.file` (with `env-file` for non-secret interpolation, `unescape: "true"` to turn `$$` into `$`), then uploads it as an encrypted artifact since rendered config can embed secrets. Enable the `compose` job and decryption in [`fly-deploy`](.github/workflows/fly-deploy.yml). Keep persistence in `fly.json` `mounts`, not Compose volumes; flyctl builds the services.

**Process groups**: declare `processes`, then scope `services`, `mounts`, and `vm` to each group. Uncomment [`fly-scale-processes`](.github/actions/fly-scale-processes/action.yml) to set explicit Machine counts.

## Secrets

`FLY_API_TOKEN` authenticates the workflows. Map app secrets once in [`fly-set-secrets`](.github/workflows/fly-set-secrets.yml):

```yaml
- uses: ./.github/actions/fly-sync-secrets
  with:
      stage: ${{ inputs.stage }}
      secrets: |
          APP_PASSWORD=${{ secrets.APP_PASSWORD }}
```

[`fly-release`](.github/workflows/fly-release.yml) imports secrets with `stage: false`; standalone [`fly-deploy`](.github/workflows/fly-deploy.yml) stages secrets before deploying.

## Volumes

Add `mounts` for persistent storage; [`fly-release`](.github/workflows/fly-release.yml) creates missing volumes in `primary_region`.

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

Run [`fly-resize-volume`](.github/workflows/fly-resize-volume.yml) with a Machine ID and size in whole GB (1-500). Growth uses native extension; follow flyctl's restart instructions if needed. Shrinking copies data over private 6PN and replaces the Machine and volume. [`fly-copy-volume`](.github/actions/fly-copy-volume/action.yml) removes its temporary workers and fork.

Shrinking requires downtime and changes the Machine ID. If copying fails, the original remains stopped and cordoned. If cloning fails, inspect any replacement before choosing which copy to resume. Adjust `mounts.initial_size` separately for future volumes. Bulk cleanup requires `confirm: "true"`; targeting a volume ID deletes it immediately.

## Private Apps

[`fly-release`](.github/workflows/fly-release.yml) creates private apps on `app-<app name>` and issues a WireGuard config. [`fly-deploy`](.github/workflows/fly-deploy.yml) checks private access and rejects public ingress IPs. Flycast provides native suspend/wakeup; direct `.internal` access requires running Machines.

Run [`fly-network`](.github/workflows/fly-network.yml) with `operation: list` for organization peers, `vend` for an app-network config, or `revoke` to remove a named peer. `peer_name` defaults to the app name; use a unique name per device. Download the `wireguard-<peer name>` artifact and import its `.conf` file. Artifacts expire after one day; CI deletes temporary configs and logs only metadata. Filenames use the first 15 peer-name characters for Linux compatibility. To replace a config, vend one with a new `peer_name`, verify access, then revoke the old peer.

Existing apps can keep their Machine network and expose Flycast on `app-<app name>`. Run [`fly-network`](.github/workflows/fly-network.yml) with `create` if that network is missing. [`fly-network`](.github/workflows/fly-network.yml) with `vend` allocates a missing Flycast IP on that network and preserves other private addresses for backend connections. Direct `internal` mode requires the app and VPN clients on the same network.

## Deploy Readiness

Service `checks` gate routing and updates after a Machine starts; `machine_checks` probe a started target via an ephemeral test Machine (rolling/canary only). Neither starts a stopped Machine; set [`fly-deploy`](.github/actions/fly-deploy/action.yml) `start-process-groups` for groups that must run deploy-time checks. `kill_signal` is `SIGTERM` (some daemons ignore Fly's `SIGINT` default). See [AGENTS.md](AGENTS.md) for the full semantics.

## Workflows And Actions

- [`fly-release`](.github/workflows/fly-release.yml): provision the app and volumes, call [`fly-deploy`](.github/workflows/fly-deploy.yml), then vend WireGuard. Run for initial setup or infrastructure changes.
- [`fly-deploy`](.github/workflows/fly-deploy.yml): sync secrets, check networking, build, and deploy application code and config changes.
- [`fly-set-secrets`](.github/workflows/fly-set-secrets.yml): secret mapping shared by releases and standalone dispatch.
- [`fly-resize-volume`](.github/workflows/fly-resize-volume.yml): resize the selected Machine's volume within its current region.
- [`fly-network`](.github/workflows/fly-network.yml): private networking and WireGuard peer management.
- Optional, uncomment or dispatch when needed: [`fly-merge-compose`](.github/workflows/fly-merge-compose.yml), [`fly-scale-processes`](.github/actions/fly-scale-processes/action.yml), [`fly-cleanup-volumes`](.github/workflows/fly-cleanup-volumes.yml), [`fly-cleanup-machines`](.github/actions/fly-cleanup-machines/action.yml), [`fly-restore-volume-snapshot`](.github/workflows/fly-restore-volume-snapshot.yml), [`fly-migrate-region`](.github/workflows/fly-migrate-region.yml), [`gh-artifact-encrypt-decrypt`](.github/actions/gh-artifact-encrypt-decrypt/action.yml).

Prefer uncommenting existing optional steps over adding duplicate jobs.

## Documentation

- [Fly.io Docs](https://fly.io/docs/) · [Configuration Reference](https://fly.io/docs/reference/configuration/) · [Multi-container Machines](https://fly.io/docs/machines/guides-examples/multi-container-machines/)
