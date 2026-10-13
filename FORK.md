# About this fork

`InfraManiacs/turnstone` — a fork of [turnstonelabs/turnstone](https://github.com/turnstonelabs/turnstone),
Apache-2.0, used by [kube_ai_data_platform](https://github.com/InfraManiacs/kube_ai_data_platform).

**There is no source divergence.** The application code is upstream's, unchanged.
This fork exists for one reason: to publish an image for an architecture
upstream does not build.

## What it changes

| File | Change |
|---|---|
| `.github/workflows/docker-publish-multiarch.yml` | New. Builds `linux/amd64` and `linux/arm64` natively and joins them into one manifest list at `ghcr.io/inframaniacs/turnstone`. |
| `.github/workflows/docker-publish.yml` | One line: a `github.repository == 'turnstonelabs/turnstone'` guard, so upstream's single-arch publish does not fire in this fork. |

Nothing else. No Python, no Dockerfile changes.

## Why

Upstream calls `docker/build-push-action` without a `platforms:` input, so it
builds only the runner's architecture and publishes an index containing
`linux/amd64` alone — verified against every `1.8.x` and `1.9.x` tag.

A pod scheduled onto an arm64 node then dies in `CrashLoopBackOff` with
`exec format error`, *intermittently*, in proportion to how much of the pool is
arm64. That presents as a flaky deploy rather than an image problem. The
platform works around it today with an amd64 `nodeSelector` in
`charts/turnstone`; an arm64 image is what lets that selector go away, which
matters on bare metal and Talos where mixed-architecture pools are ordinary.

## Why the guard on upstream's workflow

Upstream's job sets `IMAGE_NAME: ${{ github.repository }}`, which in this fork
lowercases to exactly the image the multi-arch workflow publishes. Both would
fire on the same `v*` tag, and whichever finished last would win — regularly
replacing a real manifest list with an amd64-only one. The failure is silent,
because the result still pulls fine on amd64.

The guard is a repository check rather than a deletion so the file stays
byte-comparable with upstream and a future merge is a no-op.

## Building a version

Tags on this fork mirror upstream's exactly — `v1.8.3` here is upstream's
`v1.8.3`, same commit, same source. No fork-specific version suffix, because
there is no fork-specific code; the registry namespace is what distinguishes
the builds.

Upstream's tags already exist in this fork, so a tag push will not re-trigger
them. Build one on demand:

```
gh workflow run docker-publish-multiarch.yml \
  --repo InfraManiacs/turnstone -f ref=v1.8.3
```

New tags pushed to this fork trigger it automatically.

The workflow fails if the published manifest list does not contain both
architectures — the one bug it exists to prevent is also the one that would
otherwise go unnoticed.

Only the exact version is tagged. No `:latest` or `:stable`: this fork adds an
architecture, it does not assert a release line, and a moving alias invites the
floating-tag failure the platform repo forbids.

## The better ending

The real fix is one line in upstream's workflow — `platforms:` on the build
step, or a native runner matrix if emulation proves too slow for `uv sync` and
`compileall`. Upstream already uses `ubuntu-24.04-arm` runners elsewhere in its
own CI, so the capability is there.

Note that turnstonelabs requires a CLA (`CLA.md`) for contributions, which is a
step beyond the DCO this organization uses. That is the path to retiring this
fork entirely.
