# LLM Guidance for Deploying Applications to Fly.io via GitOps

Use this guidance when adapting this template for a specific application.

## Generalize Deliberately

- Keep reusable deployment mechanics in actions and workflows; keep application behavior in app config, entrypoints, and documentation.
- Before porting a fix from another deployment, decide whether it addresses a template concern or only that app's topology.
- Prefer small changes that remove a class of failures over broad synchronization from another fork.
- When the template already encodes a deployment mechanic, keep this file focused on reasoning and failure modes rather than restating the implementation.
- Avoid carrying over branch-specific checks, scripts, or workarounds unless the same failure mode exists in the target app.
- Favor inheriting or inferring from `fly.json`, workflow `env`, and action outputs over manually repeating app names, ports, image names, regions, access mode, or process lists.
- Keep actions independent: deploy logic belongs in `fly-deploy`, private network policy in `fly-private-network`, Compose preparation in `fly-compose`, and secret mapping in `fly-set-secrets`.
- Keep image routing, staged secrets, checkout/setup ordering, private networking, Compose preparation, and cleanup as reusable mechanics instead of copied workflow blocks.

## Start From Simplicity

- Approach planning and implementation with simplicity and minimalism as first principles.
- Prefer the least custom solution that satisfies the deployment goal and preserves operator clarity.
- Use what the platform, template, upstream image, framework, and existing project conventions already provide before adding new logic.
- Treat every new script, check, service, workflow step, override, or abstraction as a cost that must be justified by a concrete failure mode or requirement.
- When several approaches could work, choose the one with fewer moving parts, fewer places to update, and less hidden coordination.
- Before adding custom logic, ask whether the same outcome can be reached by removing, simplifying, reusing, or slightly adjusting existing behavior.
- Keep app-specific behavior close to the app, and avoid duplicating responsibilities already handled by reusable template mechanics.
- Template updates should shrink future app setup; add a new action or workflow step only when it removes repeated setup or isolates a real Fly failure mode.
- When revising an existing change, prefer amending and shrinking it over layering on follow-up cleanup.

## Reason About Fly Semantics

- Treat health checks as deploy and proxy readiness contracts, not broad dependency tests.
- Avoid deploy-time checks that wake or block on databases, queues, caches, or external APIs unless that is the intended readiness contract.
- Dependency checks can change wakeup and deploy behavior; distinguish "the app can answer traffic" from "every downstream dependency is ready."
- For scale-to-zero apps, make cost, wakeup behavior, and reliability tradeoffs explicit before keeping any process warm.
- Distinguish cold starts, wrong bind addresses, private-network routing, and app boot failures before changing ports, addresses, or proxy settings.
- For private services that need wake-on-request behavior, verify the traffic path before changing networking or readiness logic.
- Use pre-release or deploy-blocking hooks only when failing before rollout is clearly better than letting the app boot and report health.

## Keep Startup Simple

- Start the externally served process as early as safely possible.
- Delay background workers or secondary processes until their own dependencies are ready.
- Prefer shallow startup sequencing over shell frameworks, strict-mode rewrites, and broad cleanup machinery.
- Polling loops should notice when the child process they depend on has exited, and should emit enough context to debug timing failures.
- Account for VM-level state that can survive process or container restarts, including stale sockets, mounted paths, pid files, and sysctl changes.
- Be skeptical of command strings that cross config, shell, remote exec, and container boundaries; simplify them first, and only move unavoidable complexity into inspectable scripts.
- In multi-process apps, verify services, checks, volumes, VMs, and mounts are scoped to the intended process group.

## Compose Guidance

- Use Fly native Compose when services belong on one Machine and a Compose file is the clearest source of container definitions.
- Keep Fly-specific persistence in `fly.json`; Compose volumes do not replace Fly Volumes.
- Avoid adding app-specific Compose generators to this template. Put upstream fetches, framework-specific overrides, and custom normalization in the app repo.
- If Compose needs preprocessing, prefer a small app-local action before `fly-compose` and `fly-deploy`.

## Preserve Operator Clarity

- Avoid duplicate deployment notices and repeated status messages.
- Avoid splitting one operational responsibility across several hidden places.
- Document any necessary manual post-deploy step when workflow automation would add fragility.

## Review And Verify

- Inspect current repo state before editing; do not assume a clean template or a clean fork.
- When asked to simplify, remove moving parts first: extra scripts, duplicated checks, repeated notices, and unnecessary preflight work.
- Verify tool behavior, runtime versions, and provider assumptions with actual source, docs, or command output.
- Recheck assumptions around registry access, CI runtimes, network behavior, DNS behavior, and external provider configuration.
- Treat memory, cache, and timeout values as workload hypotheses; validate them against actual Machine limits and logs.

## Official Fly.io Docs

- Proactively read official Fly.io docs for deployment, Machines, networking, health checks, secrets, and workflow questions. Use them to challenge existing assumptions before planning or editing.
- Prefer raw markdown from `https://raw.githubusercontent.com/superfly/docs/main/` over rendered pages from `https://fly.io/docs/`.
- Fly.io docs URL mapping patterns:
  - `https://fly.io/docs/<path>/` -> `https://raw.githubusercontent.com/superfly/docs/main/<path>.md`
  - `https://fly.io/docs/<path>/` -> `https://raw.githubusercontent.com/superfly/docs/main/<path>.html.md`
  - `https://fly.io/docs/<path>/` -> `https://raw.githubusercontent.com/superfly/docs/main/<path>.html.markerb`
  - `https://fly.io/docs/flyctl/<command>/` -> `https://raw.githubusercontent.com/superfly/docs/main/flyctl/cmd/fly_<command>.md`
- GitHub docs URL mapping patterns:
  - `https://github.com/<owner>/<repo>/blob/<branch>/<path>` -> `https://raw.githubusercontent.com/<owner>/<repo>/refs/heads/<branch>/<path>`
  - `https://github.com/<owner>/<repo>/blob/<tag>/<path>` -> `https://raw.githubusercontent.com/<owner>/<repo>/refs/tags/<tag>/<path>`

### Health Checks & Deployments

- https://fly.io/docs/reference/health-checks/
- https://fly.io/docs/blueprints/seamless-deployments/
- https://fly.io/docs/blueprints/custom-deploy-workflows/
- https://fly.io/docs/reference/configuration/

### Multiple Processes

- https://fly.io/docs/app-guides/multiple-processes/
- https://fly.io/docs/machines/guides-examples/multi-container-machines/
- https://supervisord.org/configuration.html
- https://fly.io/docs/launch/processes/

### Private Networking & Flycast

- https://github.com/fly-apps/privatenet
- https://fly.io/docs/networking/private-networking/
- https://fly.io/docs/networking/custom-private-networks/
- https://fly.io/docs/networking/flycast/
- https://fly.io/docs/blueprints/private-applications-flycast/
- https://fly.io/docs/blueprints/connect-private-network-wireguard/
- https://fly.io/docs/blueprints/autostart-internal-apps/

### docker-compose in Docker on fly.io single machine

- https://www.zachblume.com/blog/2024-10-21-docker-compose-in-docker
- https://gist.github.com/rubys/ae5f24a4dc72936a67c6c44ee3c2a0d6
- https://github.com/zachblume/docker-compose-on-fly-io/blob/main/Dockerfile
