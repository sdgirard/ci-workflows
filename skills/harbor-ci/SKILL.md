---
name: harbor-ci
description: Set up or migrate a repo's CI to Shawn's standard Harbor release blueprint - test job, then the shared sdgirard/ci-workflows harbor-build-push reusable workflow (VERSION-based semver, multi-arch push to docker.thecameraeye.ca, git tag). Use when asked to "push to Harbor", "set up CI like todo-app/ofx-ingest", "add the release pipeline", "publish the image to my private registry", or when onboarding a new containerized repo.
---

# Harbor CI blueprint

Every containerized repo releases the same way: a repo-specific `test` job, then a `build-and-push`
job that calls the shared reusable workflow. The release logic lives only in
`sdgirard/ci-workflows/.github/workflows/harbor-build-push.yml`; never copy it into a repo. If the
blueprint needs to change, change it there (see that repo's README for moving the `v1` tag).

Reference implementations: `sdgirard/ofx-ingest` (single image), `sdgirard/todo-app-code-challenge`
(two images, per-service prefixes, path filters).

## 1. Gather

Look these up in the repo before asking; ask only for what can't be inferred:

- **Images**: one per Dockerfile that ships. Each needs a `version-prefix` (usually the image name).
- **Harbor project** and **image name**: default both to the repo/app name.
- **Test commands**: from CLAUDE.md, README, or the existing CI.
- **Path filters**: only for monorepos with several independently released images.

## 2. Repo files

1. `VERSION` at the repo root: `0.1` (Major.Minor only). If the repo already has release tags or a
   chart pinned to a version, match that line instead.
2. **Dockerfile**: declare the version args immediately before the publish/build `RUN`, never at
   the top of the stage (a new commit SHA would bust the restore cache):

   ```dockerfile
   ARG APP_VERSION=0.0.0-dev
   ARG APP_COMMIT_SHA=unknown
   ARG APP_BUILD_DATE=unknown
   RUN dotnet publish ... \
       -p:Version=${APP_VERSION} \
       -p:InformationalVersion="${APP_VERSION}+${APP_COMMIT_SHA}.${APP_BUILD_DATE}"
   ```

   For non-.NET images, use the args the stack's way (e.g. write a `version.json` in a frontend build).
   For multi-arch speed when the build output is arch-neutral (.NET framework-dependent, JS bundles),
   use `FROM --platform=$BUILDPLATFORM` on the build stage.
3. **`.github/workflows/ci.yml`**:

   ```yaml
   name: CI

   on:
     push:
       branches: [main]
     pull_request:
       branches: [main]

   jobs:
     test:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         # repo-specific setup, build, test, lint

     build-and-push:
       needs: test
       if: github.event_name == 'push' && github.ref == 'refs/heads/main'
       permissions:
         contents: write
       uses: sdgirard/ci-workflows/.github/workflows/harbor-build-push.yml@v1
       with:
         harbor-project: <project>
         image: <image>
         version-prefix: <prefix>
       secrets:
         HARBOR_ROBOT_USERNAME: ${{ secrets.HARBOR_ROBOT_USERNAME }}
         HARBOR_ROBOT_TOKEN: ${{ secrets.HARBOR_ROBOT_TOKEN }}
   ```

   Optional inputs: `dockerfile`, `context`, `platforms`, `build-args`, `version-file`,
   `legacy-unprefixed-tags` (only when migrating a repo that already has unprefixed `X.Y.N` tags).
   Remove any ghcr/Docker Hub login, `docker/metadata-action`, `packages: write`, and `v*` tag
   triggers left over from an older pipeline.
4. **Deploy config**: point the Helm chart / manifests at
   `docker.thecameraeye.ca/<project>/<image>` and make sure an `imagePullSecrets` entry exists
   (Harbor is private). Update README/CLAUDE.md mentions of the old registry.

## 3. Outside the repo (Shawn does these; list them, don't attempt them)

- Harbor: project exists; robot account has push on it.
- Repo secrets `HARBOR_ROBOT_USERNAME` / `HARBOR_ROBOT_TOKEN`. Check with `gh secret list`; to set:
  `gh secret set HARBOR_ROBOT_TOKEN`.
- Cluster: pull secret for `docker.thecameraeye.ca` in the target namespace.

## 4. Verify

- `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"` and, if installed,
  `actionlint`.
- Local image build with the args to confirm they reach the artifact:
  `docker buildx build --load -t x:check --build-arg APP_VERSION=0.1.0 --build-arg APP_COMMIT_SHA=abc1234 .`
- Multi-arch locally needs a `docker-container` builder (`docker buildx ls`; Shawn has `multiarch`).
- The real check is the first push to `main`: image tags `latest`, SHA and `X.Y.0` in Harbor, and a
  `<prefix>-X.Y.0` git tag on the repo.

## Gotchas

- A public repo can only call this workflow because `ci-workflows` is public. Keep secrets out of it.
- The caller job must grant `contents: write`; a reusable workflow can't raise permissions above the caller's.
- `secrets: inherit` also works but passes every secret; prefer the explicit two.
- Bump `VERSION` by hand for a minor/major release; never create release tags manually.
