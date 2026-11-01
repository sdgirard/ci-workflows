# ci-workflows

Shared GitHub Actions workflows for my projects.

## `harbor-build-push.yml`

Reusable release job: version → multi-arch image → Harbor → git tag.

1. **Version**: `VERSION` in the calling repo holds `Major.Minor` (e.g. `0.1`), bumped by hand.
   Patch is the latest `<version-prefix>-<Major.Minor>.N` git tag + 1 (or `0`). Bumping `VERSION`
   resets the patch.
2. **Build and push**: `linux/amd64,linux/arm64` to
   `<harbor-registry>/<harbor-project>/<image>`, tagged `latest`, the commit SHA and the version.
   `APP_VERSION`, `APP_COMMIT_SHA` and `APP_BUILD_DATE` are passed as build args.
3. **Git tag**: `<version-prefix>-X.Y.Z`, pushed only after the image was pushed.

Runs are serialized per repo + prefix, so two quick pushes can't claim the same version.

### Usage

```yaml
jobs:
  test:
    # ... repo-specific build/test ...

  build-and-push:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: write
    uses: sdgirard/ci-workflows/.github/workflows/harbor-build-push.yml@v1
    with:
      harbor-project: my-app
      image: my-app
      version-prefix: my-app
    secrets:
      HARBOR_ROBOT_USERNAME: ${{ secrets.HARBOR_ROBOT_USERNAME }}
      HARBOR_ROBOT_TOKEN: ${{ secrets.HARBOR_ROBOT_TOKEN }}
```

| Input | Default | |
|---|---|---|
| `image` | required | Image name in the Harbor project |
| `harbor-project` | required | Harbor project |
| `version-prefix` | required | Git tag prefix, one per independently versioned image |
| `harbor-registry` | `docker.thecameraeye.ca` | |
| `dockerfile` | `Dockerfile` | |
| `context` | `.` | |
| `platforms` | `linux/amd64,linux/arm64` | |
| `version-file` | `VERSION` | |
| `build-args` | `''` | Extra `KEY=value` lines |
| `legacy-unprefixed-tags` | `false` | Continue from unprefixed `X.Y.N` tags when no prefixed tag exists |

Outputs: `version`, `git_tag`.

### Dockerfile side

Declare the args right before `dotnet publish` (not at the top of the stage, or every new commit
SHA busts the restore cache):

```dockerfile
ARG APP_VERSION=0.0.0-dev
ARG APP_COMMIT_SHA=unknown
ARG APP_BUILD_DATE=unknown
RUN dotnet publish ... \
    -p:Version=${APP_VERSION} \
    -p:InformationalVersion="${APP_VERSION}+${APP_COMMIT_SHA}.${APP_BUILD_DATE}"
```

### Releasing changes to this workflow

Callers pin `@v1`. After a compatible change, move the tag:

```bash
git tag -f v1 && git push -f origin v1
```

Breaking input changes get `v2`.
