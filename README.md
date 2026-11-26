# reusable-workflows

Shared GitHub Actions workflows for the gv-energy .NET repositories. Callers reference them as
`gv-energy/reusable-workflows/.github/workflows/<file>.yml@main`.

| Workflow | Used by | What it does |
|---|---|---|
| `build-test-publish.yml` | services (new pipeline) | Compiles once, runs the tests on that build, publishes every component and bakes the images from the published output |
| `test.yml` + `publish.yml` | services (old pipeline) | `test.yml` builds and tests; `publish.yml` builds each component again inside an SDK container |
| `test-web.yml` | backoffice, caps-online, shop-app | Backend tests plus the ClientApp unit tests (and optionally Playwright e2e) |
| `publish-library.yml` | core, contract and messaging repos | Packs and pushes NuGet packages with the GitVersion version |

## .NET SDK versions

Every workflow writes a `global.json` that pins the SDK to the highest version it installed,
unless the repository already has one. This keeps a repository off the SDK that the runner
image preinstalls (e.g. SDK 11), which has changed compiled output before.

The defaults do **not** include .NET 10, so existing callers keep the exact toolchain they
had. A repository opts in by passing `DOTNET_VERSIONS`, a newline-separated list for
`actions/setup-dotnet`:

| Workflow | Input | Default |
|---|---|---|
| `build-test-publish.yml` | `DOTNET_VERSIONS` | `9.x` |
| `test.yml` | `DOTNET_VERSIONS` (replaces `USE_DOTNET_7` and the fixed 6/7/8/9 steps) | empty: legacy steps |
| `test-web.yml` | `DOTNET_VERSIONS` (replaces `USE_DOTNET_7` and the fixed 6/7/8/9 steps) | empty: legacy steps |
| `publish-library.yml` | `DOTNET_VERSIONS` | `8.x` and `9.x` |

The highest version in the list is the SDK that builds; the lower ones only provide their
runtimes, which tests need for every framework they target.

- **Project on `net10.0`:** `DOTNET_VERSIONS: "10.x"`.
- **Project that targets `net9.0;net10.0`** (GV.Core): list both, so the tests run on both runtimes.

  ```yaml
  jobs:
    test:
      uses: gv-energy/reusable-workflows/.github/workflows/test.yml@main
      with:
        DOTNET_VERSIONS: |
          9.x
          10.x
      secrets: inherit
    publish:
      needs: test
      uses: gv-energy/reusable-workflows/.github/workflows/publish-library.yml@main
      with:
        NUGET_SOURCE: https://nuget.pkg.github.com/gv-energy/index.json
        DOTNET_VERSIONS: |
          9.x
          10.x
      secrets: inherit
  ```

- **Contract and messaging packages** stay on `net9.0`. A `net9.0` package is consumable by
  `net9.0` and `net10.0` services alike, so they need no change to support .NET 10.

The old `publish.yml` builds inside the SDK image named in the repository's own Dockerfile, so
for a service still on the old pipeline, moving to .NET 10 also means changing the `sdk` and
`aspnet` images in that Dockerfile. Prefer moving to `build-test-publish.yml` at the same time.

## Moving a service to `build-test-publish.yml`

`ev` and `filesync` already use it; copy from them. The old pipeline compiles the solution once
for the tests and once more per component inside Docker. The new one compiles once.

1. **`.github/workflows/ci-cd.yml`** — replace the `test` and `publish-*` jobs with one job.
   Keep the existing `on:` triggers.

   ```yaml
   jobs:
     build-test-publish:
       uses: gv-energy/reusable-workflows/.github/workflows/build-test-publish.yml@main
       with:
         COMPONENTS: Api Worker Migrator   # the components the service ships
         DOTNET_VERSIONS: "9.x"            # or "10.x"
       secrets: inherit
   ```

   `SERVICE_NAME` and the ACR secrets come from the repository secrets, as for `publish.yml`.
   `CONTEXT` defaults to `./src`; every component must be at
   `<CONTEXT>/<SERVICE_NAME>.<Component>/<SERVICE_NAME>.<Component>.csproj`.

2. **`src/Dockerfile`** — runtime image only. It copies the published output; it no longer
   restores or builds. Use `aspnet:<version>-alpine` matching the target framework
   (`runtime` is enough for a worker without ASP.NET, but one image for all components is
   simpler). See `filesync/src/Dockerfile`:

   ```dockerfile
   FROM mcr.microsoft.com/dotnet/aspnet:9.0-alpine AS base
   WORKDIR /app
   EXPOSE 8080
   ENV ASPNETCORE_URLS=http://*:8080
   ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false
   RUN apk add --no-cache icu-libs tzdata
   RUN adduser --disabled-password --home /app --gecos '' dotnetuser && chown -R dotnetuser /app
   USER dotnetuser
   WORKDIR /app

   FROM base AS final
   ARG SERVICE_NAME
   ARG COMPONENT
   COPY out/${COMPONENT}/ .
   ENV PROJECT_DLL=$SERVICE_NAME.$COMPONENT".dll"
   RUN test -f "$PROJECT_DLL"
   ENTRYPOINT ["sh", "-c", "dotnet $PROJECT_DLL"]
   ```

   Carry over anything the old runtime stage set up (extra `apk` packages, env vars, the
   non-root user), and keep the `ENTRYPOINT` the deployment expects.

3. **`src/docker-bake.hcl`** — one target per component, generated from `COMPONENTS`. Copy
   `filesync/src/docker-bake.hcl` unchanged.

4. **`src/.dockerignore`** — exclude `**/bin/` and `**/obj/` but **not** `out/`.

5. **Do not** add `-r <rid>`, `PublishReadyToRun`, `PublishSingleFile` or AOT to the
   components. The images run the runner-built output on Alpine, which only works for a
   framework-dependent, RID-agnostic publish.

6. **Required status checks** — the job names change, so the `develop` ruleset's required
   checks `test / testing` and `publish-* / publish` never report again and every PR to
   `develop` stays blocked. A repository admin must replace them with
   `build-test-publish / build` in the ruleset (Settings → Rules → the develop ruleset),
   right after the PR merges. Check `main`'s ruleset too if it requires checks.

7. **Verify** on the PR run: the tests ran, and every component image was pushed with a tag of
   the same shape as before (`<branch>-<timestamp>-<sha>`). After the merge, check the
   `develop` run and that ArgoCD picks the new images up on test.
