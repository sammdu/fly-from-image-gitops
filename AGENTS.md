# LLM Guidance for Deploying Applications to Fly.io via GitOps

Use this guidance when adapting this template for a specific application.

The template's goal is 100% GitOps through GitHub Actions, from initial Fly.io app setup to further deployments, operation, and maintenance.

Keep items brief. Split independent ideas into bullets and related aspects into sub-bullets.

## Generalize Deliberately

- Aim for reusable deployment mechanics without adopting overly specific setups from other apps.
    - Use other Fly.io deployment setups as references; avoid wholesale copying that imports unrelated topology.
    - Avoid adding non-general behaviors & patterns specific to one deployment into shared mechanics; it makes other deployments inherit app assumptions.
    - Consider app config, entrypoints, or docs for app-specific behavior, while sharing the deployment mechanics.
- Help users act on their goals through the operator interface.
    - Avoid workflows, inputs, or processes without an actionable purpose; they add ceremony without helping the user.
    - Consider independently useful workflows so users can pursue a goal without unrelated steps.
- Before porting a fix from another deployment, decide whether it addresses a template concern or only that app's topology.
- Abstract cross-repo lessons into failure modes before porting a command, config shape, or workaround.
- Prefer small changes that remove a class of failures over broad synchronization from another fork.
- Keep AGENTS.md focused on reusable reasoning and failure modes.
    - Avoid workflow- and action-specific documentation; changing implementation details make the guidance stale and less portable.
- Avoid carrying over branch-specific checks, scripts, or workarounds unless the same failure mode exists in the target app.
- Avoid duplicating deployment facts or mechanics; consider existing sources and reusable interfaces.
    - Consider native config, platform state, or action outputs for deployment facts.
    - Consider the owning workflow or app config for user settings; users should not need to edit action internals.
    - Keep creation policy separate from observed state; defaults may not describe existing resources.
    - Share settings through native GitHub Actions inputs, outputs, env, and contexts; verify their scope.
    - Aim for cohesive Fly.io actions with declared inputs and outputs.
    - Avoid scraping another workflow's YAML or calling another action's private scripts.
    - Consider shared image routing, staged secrets, checkout/setup ordering, networking, Compose, and cleanup.
- Keep non-secret architectural settings explicit and checked in.
    - For example, `FLY_ORG` and `FLY_ACCESS_MODE` describe deployment design; repository variables or secrets would hide their values.

## Start From Simplicity

- Approach planning and implementation with simplicity and minimalism as first principles.
- Prefer the least custom solution that satisfies the deployment goal and preserves operator clarity.
- Avoid rebuilding behavior provided by the platform, template, upstream image, framework, or project conventions.
    - Consider flyctl's lifecycle, state, wait, and validation features to avoid duplicate orchestration.
    - Avoid unnecessary state records, repeated cross-checks, and error handling already supplied by the platform.
- Treat every addition as a cost that needs a concrete requirement or failure mode.
    - This includes scripts, checks, services, workflow steps, overrides, abstractions, input parameters, output fields, and artifacts.
- For nontrivial fixes, compare a few simple options, test uncertain assumptions when practical, then choose the one with fewer moving parts and fewer hidden dependencies.
- Before adding custom logic, ask whether the same outcome can be reached by removing, simplifying, reusing, or slightly adjusting existing behavior.
- Avoid duplicating template mechanics in app-specific code; keep app behavior near the app so ownership stays clear.
- Aim for workflow glue that operators can read and maintain.
    - Avoid tiny helper files or Bash functions that add indirection; consider inlining short YAML/Bash steps.
    - Only consider introducing a script when it makes substantial complexity clearer than an inline block.
- Avoid growing the template without reducing repeated app setup or addressing a demonstrated platform failure.
- When revising an existing change, prefer reverting, amending, or shrinking it over adding compatibility layers or follow-up cleanup.
- Avoid accumulating conditions, branches, and states during iterative troubleshooting or implementation.
    - Revisit the failure layer and core assumptions before adding another workaround.
    - Consider broader architectural solutions that address the issue generally and simplify the logic.
- Avoid speculative rewrites, broad render transforms, custom test harnesses, helper dependencies, and one-off scripts unless a concrete platform gap requires them.

## Reason About Fly Semantics

- Treat health checks as deploy and proxy readiness contracts, not broad dependency tests.
- Service HTTP/TCP checks gate routing and updates after a Machine starts; they do not start stopped Machines.
- Machine checks use an ephemeral test Machine.
    - They probe a started deployment target through `FLY_TEST_MACHINE_IP`.
    - They run only for rolling and canary deploys and do not start stopped targets.
- Use `release_command` for one-off setup or migrations without mounted volumes.
    - It replaces `CMD` and preserves `ENTRYPOINT`.
    - It must remain compatible with the currently deployed release.
- Distinguish deploy success, proxy routing, public listener availability, and internal dependency readiness before changing checks or startup behavior.
- Keep readiness checks scoped to the intended serving contract.
    - Avoid waking or blocking on databases, queues, caches, or external APIs unless that is the intended contract.
    - Distinguish "the app can answer traffic" from "every downstream dependency is ready."
- For scale-to-zero apps, make cost, wakeup behavior, and reliability tradeoffs explicit before keeping any process warm.
    - Verify the first real cold-start request against the app's readiness requirement.
- Before changing ports, checks, or proxy settings, distinguish the failure:
    - Cold starts, `connection refused`, and app-level `502`.
    - Wrong bind addresses, private-network routing, and app boot failures.
- Persistent Fly Volume state can outlive config changes; consider mounted data, generated files, sockets, and pid files before changing startup logic.
- VM-wide kernel settings can be configured via `vm[].kernel_args` in `fly.json`.
    - For Redis, `sysctl.vm.overcommit_memory=1` applies its recommended overcommit policy before containers start.
    - Check compatibility with all containers sharing the VM and verify effective settings after deployment.
- Map upstream image, Dockerfile, Compose, and script assumptions onto Fly.io limits before editing.
    - Build/runtime boundaries and global runtime secrets.
    - Ignored Compose volumes and release commands without volumes.
    - Proxy target selection and app-injected `FLY_*` variables.
- For private services that need wake-on-request behavior, verify the traffic path before changing networking or readiness logic.
- Public CNAMEs to `<app>.flycast` require private DNS and networking; recursion can return `NXDOMAIN` while direct `.flycast` lookup succeeds.
    - For example, Socket.IO `connect_error` with `Unauthorized: TypeError: fetch failed` after HTTP `101` may indicate callback DNS failure.
    - Test DNS and callbacks from the calling container; WebSocket upgrade does not prove authentication or event delivery.
    - Configure internal callbacks separately from canonical URLs and tenant identity; verify supported env/config inputs.
    - Flycast IPs survive Machine replacement; update DNS/hosts mappings after reallocation.
- Use pre-release or deploy-blocking hooks only when failing before rollout is clearly better than letting the app boot and report health.
- Find the application's graceful-shutdown signal handler and configure `kill_signal` accordingly.
    - Do not assume Fly.io's default `SIGINT` is appropriate for the app.
    - The configured signal is sent to each container on stop and deploy.
    - Verify the main process shuts down cleanly before `kill_timeout`; otherwise it receives `SIGKILL`.

## Keep Startup Simple

- Start the externally served process as early as safely possible; Fly service checks should own external readiness when practical.
- Dependency gates should not prevent the only public listener from starting unless intentionally blocking rollout.
- Delay background workers or secondary processes until their own dependencies are ready.
- One-shot setup services must be idempotent and fast when the target state already exists.
- When an app needs file-backed secrets or initial files on a mounted path, prefer an idempotent one-shot service that writes only the missing target state.
- Avoid shell frameworks, strict-mode rewrites, or broad cleanup without a concrete need; consider shallow startup sequencing to keep execution inspectable.
- Polling loops should notice when the child process they depend on has exited, and should emit enough context to debug timing failures.
- Account for VM-level state that can survive process or container restarts, including stale sockets, mounted paths, pid files, and sysctl changes.
- Aim to make commands understandable across config, shell, remote exec, and container boundaries.
    - Avoid nested shells that obscure expansion; consider separate installation, configuration, or startup calls when clearer.
- In multi-process apps, verify services, checks, volumes, VMs, and mounts are scoped to the intended process group.

## Manage Render Boundaries

- Treat GitHub expressions, YAML, JSON, Compose interpolation, Dockerfile parsing, shell expansion, and app config as separate interpreters.
    - Decide which layer consumes each variable, quote, escape, newline, and path.
    - Verify the rendered artifact.
- Prefer structured fields, argv arrays, env maps, config files, `jq`, and native validators over strings that must survive several interpreters.
- Keep secrets out of rendered config and logs; pass them through runtime secret or file mechanisms until the final consumer.

## Compose Guidance

- Consider Fly native Compose when services belong on one Machine and Compose is the clearest source of container definitions.
    - Treat it as a translation path, not Docker Compose running in the Machine.
    - Use localhost between containers; verify support before relying on Docker networks, service DNS, or `extra_hosts`.
- Keep Fly-specific persistence in `fly.json`; Compose volumes do not replace Fly Volumes.
    - A Fly Volume mounted over an image path hides the image's original files at that path.
- Check rendered Compose output for flyctl-supported shapes, especially `volumes`, `ports`, `depends_on`, `healthcheck`, `secrets`, and `build`.
    - To clear inherited lists in shared anchors, prefer `!override []`; some Compose versions lose `!reset []` through aliases.
- Distinguish Docker Compose interpolation, `$$` escaping, `.env`, `--env-file`, `env_file`, and `environment`.
    - Verify which layer consumes and exports each value before rewriting templates.
- Fly secrets are runtime Machine secrets; they are not automatically available to Compose rendering unless the workflow explicitly passes them into that step.
- Keep Compose interpolation defaults non-secret.
- Pass runtime secrets through the reusable secret-setting mechanism.
    - Materialize a secret file inside a volume only when a concrete app setup step requires it.
- Avoid dummy Dockerfiles or image handoffs for prebuilt Compose services; consider the primary image in `fly.json` `build.image` to keep selection declarative.
- Compare upstream Compose files at the pinned ref first.
    - Preserve upstream service commands and dependency graphs unless the Fly translation path demonstrably requires an override.
- Keep app-specific Compose preprocessing in the app repo; avoid template-level generators unless they remove repeated setup across apps.

## Preserve Operator Clarity

- Avoid duplicate deployment notices and repeated status messages.
- Do not require, download, install, or configure flyctl or other vendor tools on users' computers.
    - Run required vendor tooling and handle its setup in GitHub Actions.
- Keep operator inputs clear and minimal.
    - Avoid ambiguous free-text options and hidden defaults; consider native choices to make supported operations visible.
    - Avoid redundant switches; consider deriving behavior from existing inputs to reduce operator decisions.
- Use concise prose and exact, hyperlinked workflow/action names so operators can find the relevant interface.
    - Name the responsible component: flyctl, the Machines API, Fly Proxy, or Fly.io.
- Preserve established cleanup and confirmation behavior unless explicitly changed or a verified failure requires it.
- Avoid resource churn during routine deploys; keep credential rotation and auxiliary-network replacement as explicit operator goals.
- Avoid spreading one operational responsibility across hidden locations; consider a clear owner to keep behavior discoverable.
- Avoid retaining temporary diagnostics without purpose; remove them or document the uncertainty they still resolve.
- Diagnostic logs should answer a specific uncertainty, and rendered config logging must avoid exposing secrets or secret-derived values.
- Avoid fragile post-deploy automation; consider documenting a manual step when it makes the operation more reliable.
- Avoid automating persistent-state resets without a reliable flow; consider documenting the manual reset procedure.

## Review And Verify

- Avoid assuming a clean template or fork; inspect current repo state to preserve intentional changes.
- Preserve established guidance unless repeated use contradicts it or a newer explicit user decision supersedes it.
- Honor requested design and UX review before implementation, then complete the authorized scope.
- When asked to simplify, remove moving parts first: extra scripts, duplicated checks, repeated notices, and unnecessary preflight work.
- Aim to identify the failing layer before changing deployment behavior.
    - Avoid stacking speculative fixes; they obscure which assumption failed.
    - Consider the last known-good diff, official docs, and logs to choose one hypothesis change, then verify it.
- Inspect both source and rendered output when generated config is involved.
    - Verify the actual runtime consumer and environment scope.
    - Explicit exports in a temporary diagnostic or release Machine do not prove runtime inheritance.
- Verify tool behavior, runtime versions, and provider assumptions with actual source, docs, or command output.
- Recheck assumptions around registry access, CI runtimes, network behavior, DNS behavior, and external provider configuration.
- If local `docker`, `flyctl`, or network access is unavailable, rely explicitly on CI artifacts and state what remains unverified.
- One deploy fix should generally map to one commit or one isolated hypothesis.
- Treat memory, cache, and timeout values as workload hypotheses; validate them against actual Machine limits and logs.

## Official Docs And Sources

- Proactively read official Fly.io docs for deployment, Machines, networking, health checks, secrets, and workflows.
    - Use them to challenge existing assumptions before planning or editing.
- Prefer direct Markdown at `https://docs.fly.io/<path>.md` for Fly.io documentation.
- Pair Fly.io docs with flyctl source when behavior depends on current parser or translator support.
    - Shallow-clone source repositories into a temporary directory and navigate them with `ast-grep`.
- Treat Docker Compose, GitHub Actions, upstream image, and upstream app docs as first-class sources for interpolation, workflows, image behavior, and setup.
- When a provided URL matches the patterns below, transform it proactively and read the Markdown before reasoning from the rendered page.
- Fly.io docs URL mapping patterns:
    - Map `https://fly.io/docs/<path>/` to `https://docs.fly.io/<path>.md`.
    - If direct Markdown is unavailable:
        - Use the rendered page.
        - Shallow-clone `https://github.com/superfly/docs` to inspect source; filenames may differ from public paths.
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
