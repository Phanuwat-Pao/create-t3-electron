# Deploying through MiniIDP

This repository is a MiniIDP template: generating a project from it on Gitea
gives a repo that builds the T3 Code server into a container and deploys it
through the platform's GitOps flow. Three files carry the contract.

## idp.yaml

One service, `web`. It is the `t3` server (apps/server) with the web client
bundled in and served from the same origin on port 3773. `develop` maps to the
`dev` environment; `main` maps to `production`, which starts disabled.

Keep `replicas: 1`. The server owns a SQLite database, pairing state and the
provider subprocesses for every thread. Two replicas behind one ingress would
split WebSocket sessions between two databases.

The chart mounts no volume, so `T3CODE_HOME` (`/data`) is lost on every
rollout. After a restart the server has no paired devices: read the pairing
URL from the pod log and pair again, or mint one inside the pod with
`node dist/bin.mjs pair`. The URL is printed with the bind address; swap the
scheme and host for the ingress domain before opening it.

## docker/Dockerfile.server

Mirrors what the release archive stages: the production web build, the server
bundled by `vp pack`, and a production install of the native packages the
bundle leaves external. Debian rather than Alpine because node-pty,
`@napi-rs/keyring` and `@ff-labs/fff-node` ship glibc builds.

```bash
docker build -f docker/Dockerfile.server -t t3-server .
docker run --rm -p 3773:3773 t3-server
```

The image ships `git` and `openssh-client`. The Claude provider works out of
the box because the agent SDK bundles its CLI; other provider CLIs are not
installed.

## .gitea/workflows

`ci.yml` runs on `develop`, `release/*`, `feature/*` and pull requests. Quality
(lint, format, typecheck) is a non-gating backstop. On deploy branches it waits
for MiniIDP's `idp.yaml` check, decides whether the image inputs changed since
the last green build, builds and pushes `<repo>/web` to Harbor, and notifies
MiniIDP. `build-push.yml` is the manual version of that build. `deploy.yml`
deploys or rolls back a specific tag.

Pushes to `main` do not build until the `main` entry in `ci.yml` is
uncommented, which is the switch for shipping production.

Repository secrets the workflows read:

| Secret                  | Used by                                |
| ----------------------- | -------------------------------------- |
| `TELECORP_PUBLIC_TOKEN` | the `telecorp/actions-*` mirrors       |
| `REGISTRY_URL`          | Harbor host, image name prefix         |
| `REGISTRY_USERNAME`     | Harbor login and retention             |
| `REGISTRY_PASSWORD`     | Harbor login and retention             |
| `IDP_URL`, `IDP_TOKEN`  | MiniIDP image-update notify (optional) |

The GitHub workflows under `.github/` are upstream's release pipeline and
expect its runners and secrets. They are not part of the MiniIDP contract.
