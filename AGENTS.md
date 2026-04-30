# LLM Guidance for Deploying Applications to Fly.io via GitOps

Use this guidance when adapting this template for a specific application.

## Generalize Deliberately

- Keep reusable deployment mechanics in actions and workflows; keep application behavior in app config, entrypoints, and documentation.
- Before porting a fix from another deployment, decide whether it addresses a template concern or only that app's topology.
- Prefer small changes that remove a class of failures over broad synchronization from another fork.
- When the template already encodes a deployment mechanic, keep this file focused on reasoning and failure modes rather than restating the implementation.

## Reason About Fly Semantics

- Treat health checks as deploy and proxy readiness contracts, not broad dependency tests.
- Avoid deploy-time checks that wake or block on databases, queues, caches, or external APIs unless that is the intended readiness contract.
- Dependency checks can change wakeup and deploy behavior; distinguish "the app can answer traffic" from "every downstream dependency is ready."
- For scale-to-zero apps, make cost, wakeup behavior, and reliability tradeoffs explicit before keeping any Machine warm.
- Consider `suspend` for scale-to-zero services that fit Fly's constraints and benefit from faster wakeups.
- Distinguish slow cold starts, wrong bind addresses, private-network routing, and app boot failures before changing ports, addresses, or proxy settings.
- For private services that need wake-on-request behavior, verify the traffic path actually goes through Fly Proxy rather than direct private DNS.
- Reserve release commands for cases where failing before rollout is clearly better than letting the app boot and report health.

## Keep Startup Simple

- Start the externally served process as early as safely possible.
- Delay background workers or secondary processes until their own dependencies are ready.
- Prefer shallow startup sequencing over shell frameworks, strict-mode rewrites, and broad cleanup machinery.
- Polling loops should notice when the child process they depend on has exited, and should emit enough context to debug timing failures.
- Account for VM-level state that can survive process or container restarts, including stale sockets, mounted paths, pid files, and sysctl changes.
- Be skeptical of command strings that cross YAML, shell, remote exec, and container boundaries; move complex payloads into inspectable scripts.
- In multi-process apps, verify services, checks, volumes, VMs, and mounts are scoped to the intended process group.

## Preserve Operator Clarity

- Avoid duplicate deployment notices and repeated status messages.
- Avoid splitting one operational responsibility across several hidden places.
- Document any necessary manual post-deploy step when workflow automation would add fragility.

## Review And Verify

- Inspect current repo state before editing; do not assume a clean template or a clean fork.
- When asked to simplify, remove moving parts first: extra scripts, duplicated checks, repeated notices, and unnecessary preflight work.
- Verify tool behavior, runtime versions, and provider assumptions with actual source, docs, or command output.
- Recheck assumptions around registry access, GitHub Actions runtimes, IPv4/IPv6 preference, DNS behavior, and external provider configuration.
- Treat memory, cache, and timeout values as workload hypotheses; validate them against actual Machine limits and logs.

## Official Fly.io Docs

- Proactively read official Fly.io docs for deployment, Machines, networking, health checks, secrets, and workflow questions. Use them to challenge existing assumptions before planning or editing.
- Prefer raw markdown from `https://raw.githubusercontent.com/superfly/docs/main/` over rendered pages from `https://fly.io/docs/`.
- URL mapping examples:
  - Base path: `https://fly.io/docs/` -> `https://raw.githubusercontent.com/superfly/docs/main/`
  - Flyctl command page: `https://fly.io/docs/flyctl/deploy/` -> `https://raw.githubusercontent.com/superfly/docs/main/flyctl/cmd/fly_deploy.md`
  - HTML markdown page: `https://fly.io/docs/blueprints/custom-deploy-workflows/` -> `https://raw.githubusercontent.com/superfly/docs/main/blueprints/custom-deploy-workflows.html.md`
  - Markerb page: `https://fly.io/docs/reference/configuration/` -> `https://raw.githubusercontent.com/superfly/docs/main/reference/configuration.html.markerb`

### Health Checks & Deployments

- https://fly.io/docs/reference/health-checks/
- https://fly.io/docs/blueprints/seamless-deployments/
- https://fly.io/docs/blueprints/custom-deploy-workflows/
- https://fly.io/docs/reference/configuration/

### Multiple Processes

- https://fly.io/docs/app-guides/multiple-processes/
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
