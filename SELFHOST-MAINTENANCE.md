# Private self-host maintenance

This repository is the private `roxasroot.net` deployment fork of Executor.
It keeps the upstream project recognizable so upgrades can be reviewed against
the real upstream history instead of becoming an opaque vendor copy.

## Remotes and ownership

- `origin`: `git@github.com:ramarivera/executor-selfhost.git` (private deployment fork)
- `upstream`: `https://github.com/UsefulSoftwareCo/executor.git` (source project)
- Image: `ghcr.io/ramarivera/executor-selfhost`

The old public `ramarivera/executor` fork was not the intended home for this
deployment. The private repository became canonical on 2026-07-27; the public
fork remains separate until it is deliberately archived or removed.

## Local changes carried here

1. `apps/host-selfhost/executor.config.ts` enables stdio MCP transport only when
   `EXECUTOR_ALLOW_STDIO_MCP=true`.
2. `apps/host-selfhost/Dockerfile` passes that opt-in into the container and
   includes a pinned mise-managed Node, Bun, Python, Rust, and uv toolchain for
   trusted local stdio MCPs.
3. `apps/host-selfhost/mise.toml` is the single version manifest for those
   runtime tools.

The stdio capability is intentionally not enabled by default. This image is for
the private, trusted Executor deployment only.

## Upgrading from upstream

Fetch upstream refs and inspect the incoming release before changing `main`:

```text
git fetch upstream --tags
git log --oneline main..upstream/main
git log --oneline --decorate --all -- apps/host-selfhost/Dockerfile
```

Create an upgrade branch, merge the upstream release, and reapply only the
small private deployment delta when conflicts occur:

```text
git switch -c upgrade/upstream-<version> main
git merge --no-ff upstream/main
git diff upstream/main...HEAD -- apps/host-selfhost/executor.config.ts apps/host-selfhost/Dockerfile apps/host-selfhost/mise.toml
```

Run the self-host checks before publishing an image:

```text
bun run --filter @executor-js/host-selfhost typecheck
bun run --filter @executor-js/host-selfhost test
bun run format:check
docker buildx build --platform linux/amd64 --build-arg EXECUTOR_ALLOW_STDIO_MCP=true -f apps/host-selfhost/Dockerfile .
```

Only after those checks pass should the release workflow be dispatched with an
annotated tag. The workflow publishes the private GHCR image by digest and runs
the self-host Docker E2E suite before moving its version/channel tags.
