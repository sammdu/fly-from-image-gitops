# Fly.io GitOps App Template

Lightweight template for deploying apps to [Fly.io](https://fly.io) from GitHub Actions. The usual path: edit `fly.json`, add a Dockerfile or Compose file only when needed, map secrets, run bootstrap, then deploy.

For LLM agents adapting this template, read [AGENTS.md](AGENTS.md) first.

## Quick Start

1. Edit `fly.json`: set `app`, `primary_region`, deployment type, and `http_service.internal_port`.
2. Add the GitHub Actions secret `FLY_API_TOKEN`.
3. Override `FLY_ORG` in workflow `env` only when you do not deploy to `personal`.
4. Map app secrets in [`fly-set-secrets.yml`](.github/workflows/fly-set-secrets.yml).
5. Run [`Fly Bootstrap`](.github/workflows/fly-bootstrap.yml). If `release_command` needs same-app process groups that do not exist yet, pass them in `release_dependency_process_groups`.
6. Run [`Deploy App`](.github/workflows/fly-deploy.yml), or uncomment its `push` trigger.

## Deployment Types

Pick one in `fly.json`:

- Prebuilt image: `"build": { "image": "registry.example.com/app:tag" }`
- Dockerfile: `"build": { "dockerfile": "Dockerfile", "context": "." }`
- Docker Compose: `"build": { "compose": { "file": "compose.yaml" } }`
- Custom multi-container Machine: `"experimental": { "machine_config": "cli-config.json" }`

**Compose**: [`fly-merge-compose.yml`](.github/workflows/fly-merge-compose.yml) merges optional upstream `urls` and `local` override(s) into `build.compose.file` (with `env-file` for non-secret interpolation, `unescape: "true"` to turn `$$` into `$`), then uploads it as an encrypted artifact since rendered config can embed secrets. Uncomment the `compose` job and the decrypt step in bootstrap/deploy to enable it. Keep persistence in `fly.json` `mounts`, not Compose volumes; flyctl builds the services.

**Process groups**: declare `processes`, then scope `services`, `mounts`, and `vm` to each group. Uncomment [`fly-scale-processes`](.github/actions/fly-scale-processes/action.yml) to set explicit Machine counts.

## Secrets

`FLY_API_TOKEN` authenticates the workflows. Map app secrets once in [`fly-set-secrets.yml`](.github/workflows/fly-set-secrets.yml):

```yaml
- uses: ./.github/actions/fly-sync-secrets
  with:
      stage: ${{ inputs.stage }}
      secrets: |
          APP_PASSWORD=${{ secrets.APP_PASSWORD }}
```

Bootstrap imports with `stage: false`; normal deploys stage secrets before deploying.

## Volumes

Add `mounts` only for persistent storage; bootstrap creates missing volumes in `primary_region`.

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

No default path destroys Machines or volumes; destructive cleanup needs `confirm: "true"`.

## Private Apps

Uncomment the workflow `env` block and set the access mode:

```yaml
env:
    FLY_ACCESS_MODE: flycast   # .flycast via Fly Proxy, or "internal" for .internal over 6PN
```

Then uncomment [`fly-private-network`](.github/actions/fly-private-network/action.yml) in bootstrap and deploy; bootstrap passes `allocate: "true"` for Flycast. Create WireGuard access during bootstrap or with the [manual WireGuard workflow](.github/workflows/fly-wireguard.yml).

## Deploy Readiness

Service `checks` gate routing and updates after a Machine starts; `machine_checks` probe a started target via an ephemeral test Machine (rolling/canary only). Neither starts a stopped Machine — set [`fly-deploy`](.github/actions/fly-deploy/action.yml) `start-process-groups` for groups that must run deploy-time checks. `kill_signal` is `SIGTERM` (some daemons ignore Fly's `SIGINT` default). See [AGENTS.md](AGENTS.md) for the full semantics.

## Workflows And Actions

- [`fly-bootstrap.yml`](.github/workflows/fly-bootstrap.yml): create or reconcile app and volumes, sync secrets, optionally prepare release dependencies, then deploy.
- [`fly-deploy.yml`](.github/workflows/fly-deploy.yml): sync secrets, optionally prepare private networking or Compose, then deploy.
- [`fly-set-secrets.yml`](.github/workflows/fly-set-secrets.yml): reusable secret-sync helper for both workflows.
- [`fly-wireguard.yml`](.github/workflows/fly-wireguard.yml): manual WireGuard access utility.
- Optional, uncomment or dispatch when needed: [`fly-private-network`](.github/actions/fly-private-network/action.yml), [`fly-merge-compose.yml`](.github/workflows/fly-merge-compose.yml), [`fly-scale-processes`](.github/actions/fly-scale-processes/action.yml), [`fly-cleanup-volumes`](.github/workflows/fly-cleanup-volumes.yml), [`fly-cleanup-machines`](.github/actions/fly-cleanup-machines/action.yml), [`fly-restore-volume-snapshot`](.github/workflows/fly-restore-volume-snapshot.yml), [`fly-migrate-region`](.github/workflows/fly-migrate-region.yml), [`gh-artifact-encrypt-decrypt`](.github/actions/gh-artifact-encrypt-decrypt/action.yml).

Prefer uncommenting existing optional steps over adding duplicate jobs.

## Documentation

- [Fly.io Docs](https://fly.io/docs/) · [Configuration Reference](https://fly.io/docs/reference/configuration/) · [Multi-container Machines](https://fly.io/docs/machines/guides-examples/multi-container-machines/)
