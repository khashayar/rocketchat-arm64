# Rocket.Chat ARM64 Docker Image

This repository builds a production-ready [Rocket.Chat](https://github.com/RocketChat/Rocket.Chat) bundle that runs on ARM64/AArch64. The `Dockerfile` pulls the official Rocket.Chat release (`RC_VERSION`), recompiles server dependencies (including `sharp` for ARM), installs Deno, and wires the container to run `node main.js` from the shipped bundle.

## Build Locally

```bash
docker build \
  --build-arg ARCH=aarch64 \
  --build-arg DENO_SHA=<sha for ARCH/version> \
  -t your-registry/rocketchat-arm64:local .
```

Key build arguments / env vars:

- `ARCH` – target architecture for the Deno binary (defaults to `aarch64`).
- `DENO_SHA` – SHA256 that matches the chosen Deno archive; must be updated when `ARCH` or `DENO_VERSION` changes.
- `RC_VERSION` – Rocket.Chat release to download.
- `SHARP_VERSION` – npm version for the server-side `sharp` dependency.

## Publish with GitHub Actions

- The `Docker Publish` workflow (`.github/workflows/publish.yml`) runs on every `v*` tag push and pushes exactly one image tag: the git tag without the `v` (`v8.9.0` → `8.9.0`, `v8.9.0-1` → `8.9.0-1`). The git tag must equal the Dockerfile's `RC_VERSION`, optionally followed by `-N`, or the run fails.
- **Docker Hub tags are immutable** (rule `.*`): a pushed tag can never be overwritten. That's why there are no `latest` or branch tags. To fix an image for a Rocket.Chat version that's already published (base image, Node, Deno, sharp), push a build-numbered tag such as `v8.9.0-1` and deploy that.
- A manual run (Actions tab → "Run workflow") on a branch builds without pushing, as a CI smoke test. A manual run on a tag ref publishes like a tag push.
- To push to a registry, configure the `DOCKER_HUB_USER` and `DOCKER_HUB_TOKEN` (or analogous) secrets before running the workflow. The job will fail fast if they’re missing.

## Automated Release Flow

1. Update `RC_VERSION` (and matching Deno/sharp args) in the `Dockerfile`, test locally, and merge the change into `main`.
2. Create an annotated Git tag that matches the release, e.g. `git tag -a v7.10.7 -m "Rocket.Chat 7.10.7"` followed by `git push origin v7.10.7`.
3. Tag pushes trigger the `Release Rocket.Chat` workflow (`.github/workflows/release.yml`), which publishes a GitHub release using the tag name.
4. The tag push also starts the `Docker Publish` workflow, which builds the image and pushes `7.10.7`.

Before tagging, check the new release's runtime requirements against the Dockerfile. Rocket.Chat enforces the **Node major version at startup**, and it can be stricter than `package.json` `engines`: 8.9.0 refuses Node 22 even though `engines` allows it. Also check the Deno version in the release notes, and `sharp` in `apps/meteor/package.json`. Production runs a single replica, so an image that won't start means an outage.

## Runtime Configuration

The image runs as the unprivileged `rocketchat` user, exposes port `3000`, and defines:

- `MONGO_URL=mongodb://db:27017/meteor`
- `ROOT_URL=http://localhost:3000`
- `PORT=3000`
- `DEPLOY_METHOD=docker-official`

Mount `/app/uploads` if you need persistent file storage:

```bash
docker run -d \
  -p 3000:3000 \
  -v rocketchat_uploads:/app/uploads \
  -e MONGO_URL=mongodb://mongo/rocketchat \
  your-registry/rocketchat-arm64:latest
```

## Updating Versions

1. Edit `RC_VERSION`, `DENO_VERSION`, `DENO_SHA`, or `SHARP_VERSION` in the `Dockerfile`.
2. Rebuild locally or trigger the workflow to publish a new image.
3. Verify Rocket.Chat boots successfully against your MongoDB instance.

Refer to [Rocket.Chat's Version Durability guide](https://docs.rocket.chat/docs/version-durability) when picking which upstream release to package.
