# Fly.io GitOps App Template

Lightweight template for deploying apps to [Fly.io](https://fly.io) from GitHub Actions. The usual path is: edit `fly.json`, add a Dockerfile or Compose file only when needed, map secrets, run bootstrap, then deploy.

For LLM agents adapting this template, read [AGENTS.md](AGENTS.md) first.

## Quick Start

1. Edit `fly.json`: set `app`, `primary_region`, deployment type, and `http_service.internal_port`.
2. Add the GitHub Actions secret `FLY_API_TOKEN`.
3. Override `FLY_ORG` in workflow `env` only when you do not deploy to `personal`.
4. Map app secrets in `.github/workflows/fly-set-secrets.yml`.
5. Run `Fly Bootstrap`.
6. Run `Deploy App`, or uncomment the `push` trigger in `.github/workflows/fly-deploy.yml`.

## Deployment Types

Single image:

```json
"build": { "image": "registry.example.com/app:tag" }
```

Single Dockerfile:

```json
"build": { "dockerfile": "Dockerfile", "context": "." }
```

Docker Compose:

```json
"build": { "compose": { "file": "compose.yaml" } }
```

Uncomment `fly-compose` before deploy when using Compose. If you use a non-secret interpolation file, pass it as `env-file`:

```yaml
- uses: ./.github/actions/fly-compose
  with:
      env-file: compose.fly.env
```

Keep Fly Volumes in `fly.json`, not Compose volumes. When every Compose service uses a prebuilt image, pin `build.image` to the primary image and skip `fly-build-image`. Fly native Compose is best when services belong on one multi-container Machine; containers share `localhost`, one service receives inbound proxy traffic, and one service should be buildable.

Custom multi-container Machine:

```json
"experimental": { "machine_config": "cli-config.json" }
```

Use this only when Fly config cannot express the Machine shape cleanly. Uncomment `fly-cleanup-machines` if replaced Machines accumulate.

Multi-machine process groups:

```json
"processes": { "web": "bin/web", "worker": "bin/worker" },
"services": [
    { "internal_port": 8080, "processes": ["web"], "protocol": "tcp" }
]
```

Scope `services`, `mounts`, and `vm` entries to the intended process groups. Uncomment `fly-scale-processes` after deploy when you want explicit process group Machine counts.

## Secrets

`FLY_API_TOKEN` authenticates the workflows. App secrets are mapped once in `.github/workflows/fly-set-secrets.yml`.

```yaml
- uses: ./.github/actions/fly-sync-secrets
  with:
      stage: ${{ inputs.stage }}
      secrets: |
          APP_PASSWORD=${{ secrets.APP_PASSWORD }}
          API_KEY=${{ secrets.API_KEY }}
```

Bootstrap imports secrets with `stage: false` after the app exists. Normal deploys stage secrets before deploying.

## Volumes

Add `mounts` only when the app needs persistent storage. Bootstrap creates missing volumes in `primary_region`.

```json
"mounts": [
    {
        "source": "app_data",
        "destination": "/app/data",
        "initial_size": "1gb",
        "auto_extend_size_increment": "1gb",
        "auto_extend_size_limit": "10gb",
        "auto_extend_size_threshold": 80,
        "scheduled_snapshots": true,
        "snapshot_retention": 7
    }
]
```

No default workflow path destroys Machines or volumes. Destructive cleanup actions require `confirm: "true"`.

## Private Apps

For private-network apps, uncomment the workflow `env` block and set the access mode:

```yaml
env:
    FLY_ACCESS_MODE: flycast
```

Use `flycast` for `.flycast` access through Fly Proxy, or `internal` for `.internal` access over 6PN. Then uncomment `fly-private-network` in bootstrap and deploy. Bootstrap should pass `allocate: "true"` for Flycast. Keep `fly-wireguard-config` enabled after bootstrap and deploy so the workflow creates or reports a reusable private-network peer automatically.

## Workflows And Actions

- `fly-bootstrap.yml`: create app and volumes, sync secrets, optionally prepare private networking or Compose, then deploy.
- `fly-deploy.yml`: sync secrets, optionally prepare private networking or Compose, then deploy.
- `fly-set-secrets.yml`: reusable helper used by both main workflows.
- `fly-wireguard.yml`: manual utility for rotating private access config.
- `fly-private-network`: Flycast/internal policy, private IPv6 allocation, and public IP checks.
- `fly-compose`: validate and optionally render Docker Compose before `flyctl deploy`.
- `fly-scale-processes`: explicit process group Machine count reconciliation after deploy.
- `fly-cleanup-volumes`, `fly-cleanup-machines`: optional maintenance actions.

Prefer uncommenting existing optional steps over adding duplicate workflow jobs.

## Documentation

- [Fly.io Docs](https://fly.io/docs/)
- [Fly.io Configuration Reference](https://fly.io/docs/reference/configuration/)
- [Multi-container Machines](https://fly.io/docs/machines/guides-examples/multi-container-machines/)
