# LLM Guidance for Deploying Applications to Fly.io via GitOps

Use this guidance when adapting this template for a specific application.

## Generalize Deliberately

- Keep reusable deployment mechanics in actions and workflows; keep application behavior and app-specific topology in app config, entrypoints, and documentation.
- Before porting a fix from another deployment, decide whether it addresses a template concern or only that app's topology.
- Abstract cross-repo lessons into failure modes before porting a command, config shape, or workaround.
- Use sibling deployments as references, not sources to copy wholesale.
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
- For nontrivial fixes, compare a few simple options, test uncertain assumptions when practical, then choose the one with fewer moving parts and fewer hidden dependencies.
- Before adding custom logic, ask whether the same outcome can be reached by removing, simplifying, reusing, or slightly adjusting existing behavior.
- Keep app-specific behavior close to the app, and avoid duplicating responsibilities already handled by reusable template mechanics.
- Template updates should shrink future app setup; add a new action or workflow step only when it removes repeated setup or isolates a real Fly failure mode.
- When revising an existing change, prefer reverting, amending, or shrinking it over adding compatibility layers or follow-up cleanup.
- After repeated failures, stop and reclassify the failure layer before editing again.
- Avoid speculative rewrites, broad render transforms, custom test harnesses, helper dependencies, and one-off scripts unless a concrete platform gap requires them.

## Reason About Fly Semantics

- Treat health checks as deploy and proxy readiness contracts, not broad dependency tests.
- Distinguish deploy success, proxy routing, public listener availability, and internal dependency readiness before changing checks or startup behavior.
- Avoid checks that wake or block on databases, queues, caches, or external APIs unless that is the intended readiness contract; distinguish "the app can answer traffic" from "every downstream dependency is ready."
- For scale-to-zero apps, make cost, wakeup behavior, and reliability tradeoffs explicit before keeping any process warm.
- Distinguish cold starts, `connection refused`, app-level `502`, wrong bind addresses, private-network routing, and app boot failures before changing ports, checks, or proxy settings.
- Persistent Fly Volume state can outlive config changes; consider mounted data, generated files, sockets, and pid files before changing startup logic.
- Map upstream image, Dockerfile, Compose, and script assumptions onto Fly limits before editing: build/runtime boundaries, global runtime secrets, ignored Compose volumes, release commands without volumes, proxy target selection, and app-injected `FLY_*` variables.
- For private services that need wake-on-request behavior, verify the traffic path before changing networking or readiness logic.
- Use pre-release or deploy-blocking hooks only when failing before rollout is clearly better than letting the app boot and report health.

## Keep Startup Simple

- Start the externally served process as early as safely possible; Fly service checks should own external readiness when practical.
- Dependency gates should not prevent the only public listener from starting unless intentionally blocking rollout.
- Delay background workers or secondary processes until their own dependencies are ready.
- One-shot setup services must be idempotent and fast when the target state already exists.
- When an app needs file-backed secrets or initial files on a mounted path, prefer an idempotent one-shot service that writes only the missing target state.
- Prefer shallow startup sequencing over shell frameworks, strict-mode rewrites, and broad cleanup machinery.
- Polling loops should notice when the child process they depend on has exited, and should emit enough context to debug timing failures.
- Account for VM-level state that can survive process or container restarts, including stale sockets, mounted paths, pid files, and sysctl changes.
- Be skeptical of command strings that cross config, shell, remote exec, and container boundaries; simplify them first, and only move unavoidable complexity into inspectable scripts.
- In multi-process apps, verify services, checks, volumes, VMs, and mounts are scoped to the intended process group.

## Manage Render Boundaries

- Treat GitHub expressions, YAML, JSON, Compose interpolation, Dockerfile parsing, shell expansion, and app config as separate interpreters; decide which layer should consume each variable, quote, escape, newline, and path, then verify the rendered artifact.
- Prefer structured fields, argv arrays, env maps, config files, `jq`, and native validators over strings that must survive several interpreters.
- Keep secrets out of rendered config and logs; pass them through runtime secret or file mechanisms until the final consumer.

## Compose Guidance

- Use Fly native Compose when services belong on one Machine and a Compose file is the clearest source of container definitions; treat it as a translation path, not Docker Compose running in the Machine.
- Keep Fly-specific persistence in `fly.json`; Compose volumes do not replace Fly Volumes, and a Fly Volume mounted over an image path hides the image's original files at that path.
- Check rendered Compose output for flyctl-supported shapes, especially `volumes`, `ports`, `depends_on`, `healthcheck`, `secrets`, and `build`.
- Docker Compose interpolation, `$$` escaping, `.env`, `--env-file`, `env_file`, and `environment` operate at different layers; verify which layer consumes and exports each value before rewriting templates.
- Fly secrets are runtime Machine secrets; they are not automatically available to Compose rendering unless the workflow explicitly passes them into that step.
- Use `compose.fly.env` or `fly-compose` `env-file` only for non-secret interpolation defaults. Keep runtime secrets in `fly-set-secrets` unless a concrete app setup step must materialize a file inside a volume.
- For all-prebuilt Compose services, prefer pinning `fly.json` `build.image` to the primary image over adding a dummy Dockerfile or image handoff step.
- Compare pinned upstream Compose files at the pinned ref first, and preserve upstream service commands and dependency graphs unless the Fly translation path demonstrably requires an override.
- Keep app-specific Compose preprocessing in the app repo; avoid template-level generators unless they remove repeated setup across apps.

## Preserve Operator Clarity

- Avoid duplicate deployment notices and repeated status messages.
- Avoid splitting one operational responsibility across several hidden places.
- Remove temporary diagnostics before finalizing, or document why they must remain.
- Diagnostic logs should answer a specific uncertainty, and rendered config logging must avoid exposing secrets or secret-derived values.
- Document any necessary manual post-deploy step when workflow automation would add fragility.
- Document manual reset steps for persistent state when automating the reset would be fragile.

## Review And Verify

- Inspect current repo state before editing; do not assume a clean template or a clean fork.
- When asked to simplify, remove moving parts first: extra scripts, duplicated checks, repeated notices, and unnecessary preflight work.
- For broken deployments, classify the failure layer, diff against the last known-good commit, inspect official docs or source, make one hypothesis change, then verify with logs or CI-rendered config.
- Inspect both source and rendered output when generated config is involved.
- Verify tool behavior, runtime versions, and provider assumptions with actual source, docs, or command output.
- Recheck assumptions around registry access, CI runtimes, network behavior, DNS behavior, and external provider configuration.
- If local `docker`, `flyctl`, or network access is unavailable, rely explicitly on CI artifacts and state what remains unverified.
- One deploy fix should generally map to one commit or one isolated hypothesis.
- Treat memory, cache, and timeout values as workload hypotheses; validate them against actual Machine limits and logs.

## Official Docs And Sources

- Proactively read official Fly.io docs for deployment, Machines, networking, health checks, secrets, and workflow questions. Use them to challenge existing assumptions before planning or editing.
- Prefer raw markdown from `https://raw.githubusercontent.com/superfly/docs/main/` over rendered pages from `https://fly.io/docs/`.
- Pair Fly docs with flyctl source when behavior depends on current parser or translator support.
- Treat Docker Compose, GitHub Actions, upstream image, and upstream app documentation as first-class sources when interpolation, workflows, image behavior, or app setup is involved.
- When a provided URL matches the patterns below, transform it proactively and read the raw source before reasoning from the rendered page.
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
