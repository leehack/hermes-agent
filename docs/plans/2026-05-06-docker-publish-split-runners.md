# Docker Publish: Split Per-Arch Runners Implementation Plan

> **For Hermes:** Execute task-by-task. Each task has exact file paths, complete code, and verification steps.

**Goal:** Replace the single-runner QEMU-emulated multi-arch build in `.github/workflows/docker-publish.yml` with a split where amd64 and arm64 each build natively on their own runner in parallel, then a merge job stitches the per-arch digests into a multi-arch manifest — while preserving all existing safety behavior (per-commit SHA tags, OCI revision labels, race-safe `:latest` advancement, dashboard smoke test).

**Architecture:** Docker's [recommended multi-runner multi-platform pattern](https://docs.docker.com/build/ci/github-actions/multi-platform/#distribute-build-across-multiple-runners). Each per-arch job builds with `outputs: type=image,...,push-by-digest=true` (pushes an anonymous digest, no tag). The merge job downloads the digests, runs `docker buildx imagetools create -t <tag> <image>@sha256:<digest-a> <image>@sha256:<digest-b>`, and produces the final tagged multi-arch manifest. Cache is scoped per-arch (`scope=amd64` / `scope=arm64`) so the two runners don't clobber each other in the gha cache backend.

**Tech Stack:** GitHub Actions, docker/build-push-action@v6, docker/setup-buildx-action@v3, docker buildx imagetools, GitHub-hosted `ubuntu-24.04-arm` runner.

**Why the current workflow rebuilds arm64 from scratch every time:**
1. Single runner uses QEMU to emulate arm64 — every layer costs ~5-10× amd64 time
2. `cache-from: type=gha` with no `scope=` means amd64 and arm64 layer cache entries share one key namespace; because amd64 gets built first with `load: true`, it populates the cache with amd64-only layers, and the subsequent multi-arch push writes arm64 on top, but the push also re-reads amd64 from GHA which can race with its own earlier write
3. gha cache has a 10GB LRU limit per repo; `mode=max` caches all intermediate layers and arm64+amd64 together fills it fast, evicting earlier entries

After the split, arm64 builds natively on GitHub's free arm64 runner (no QEMU) with its own cache scope, which removes the emulation cost and the cache collision.

---

## Pre-flight Checks (Do These Before Writing Any YAML)

### Check 1: Verify `ubuntu-24.04-arm` is available for this repo

GitHub's public arm64 runners (`ubuntu-22.04-arm`, `ubuntu-24.04-arm`) are free for public repos. `NousResearch/hermes-agent` is public (the workflow gates on `github.repository == 'NousResearch/hermes-agent'` only, no private-repo-specific logic).

Run:
```bash
# Confirm repo visibility is public
nix run nixpkgs#gh -- repo view NousResearch/hermes-agent --json visibility -q .visibility
# Expected output: public
```

If the output is `private`, arm64 runners require a paid plan / Larger Runners config and this plan needs a rethink (likely: self-hosted arm64 runner or accept arm64 stays on QEMU and only fix the cache scopes). **Stop and check with Ethie before continuing if not public.**

### Check 2: Confirm nothing external depends on the old job name `build-and-push`

Required-status-check rules on branch protection sometimes pin specific job names. This workflow only runs on `push` to main and `release` events (not PRs), so it is **unlikely** to be a required check, but worth a search.

Run:
```bash
cd /home/ari/src/hermes-agent
# Search for references to the job name anywhere in repo
# (workflow status badges, docs, other workflows)
grep -rn "build-and-push" --include="*.yml" --include="*.yaml" --include="*.md" . 2>/dev/null | grep -v node_modules
```

Expected: only references inside `docker-publish.yml` itself. If other files reference it, the plan needs an adjustment (either keep the old job name for backward compat, or update all references).

### Check 3: Confirm current main branch is green on this workflow

Run:
```bash
cd /home/ari/src/hermes-agent
nix run nixpkgs#gh -- run list --workflow=docker-publish.yml --branch=main --limit=3
```

Expected: most recent run has `completed  success`. If it's broken, fix that before stacking this change on top.

---

## Task 1: Create the feature branch

**Objective:** Isolate changes on a branch so the plan can be reviewed end-to-end before touching main.

**Files:** none yet — just branch setup.

**Step 1: Branch**

```bash
cd /home/ari/src/hermes-agent
git checkout main
git pull --ff-only origin main
git checkout -b docker-publish-split-runners
```

**Step 2: Verify clean state**

```bash
git status
# Expected: "nothing to commit, working tree clean"
```

**Step 3: No commit yet** (commit the plan in Task 2).

---

## Task 2: Commit the plan itself

**Objective:** Commit this plan document so reviewers (including future us) can see the reasoning.

**Files:**
- Already written: `docs/plans/2026-05-06-docker-publish-split-runners.md`

**Step 1: Stage and commit**

```bash
cd /home/ari/src/hermes-agent
git add docs/plans/2026-05-06-docker-publish-split-runners.md
git commit -m "docs: plan for splitting docker-publish per-arch runners"
```

---

## Task 3: Rewrite docker-publish.yml — jobs structure

**Objective:** Replace the single `build-and-push` job with three jobs: `build-amd64`, `build-arm64`, and `merge`, preserving all existing behavior.

**Files:**
- Modify: `.github/workflows/docker-publish.yml` (full rewrite)

**Design decisions locked in:**

1. **`build-amd64` runs the smoke tests** — both `--help` and `dashboard --help`. It is the only job that `load`s an image into a local daemon. arm64 gets no smoke test (acceptable; the binary surface is the same Python/Node code, arm64-specific failures are rare and catchable by arm64 users).
2. **`build-amd64` and `build-arm64` both push by digest only** (`outputs: type=image,name=...,push-by-digest=true,name-canonical=true,push=true`). No tag is applied by the per-arch builds.
3. **OCI label `org.opencontainers.image.revision=${{ github.sha }}` is applied on both per-arch builds** so the existing `move-latest` job (which reads the label off `linux/amd64`) keeps working unchanged.
4. **`merge` job stitches the digests into the final tag(s)** using `docker buildx imagetools create -t <tag> <image>@sha256:<digest-a> <image>@sha256:<digest-b>`. On main pushes it produces `:sha-<sha>`. On releases it produces `:<tag_name>`. `merge` also sets the `pushed_sha_tag` output so `move-latest` knows the SHA tag is live.
5. **`move-latest` keeps its ancestor-check logic verbatim**, just changes `needs: build-and-push` → `needs: merge` and reads `pushed_sha_tag` from `merge`. **Also flips its own `cancel-in-progress: true` → `false`** — see decision #10 below.
6. **Cache is scoped per-arch** with `scope=docker-amd64` / `scope=docker-arm64`. `mode=max` retained.
7. **`ubuntu-24.04-arm` for arm64** — free GitHub-hosted runner, native arm64, no QEMU.
8. **QEMU setup step removed entirely** — not needed when each arch runs on native hardware.
9. **Build paths-filter triggers and top-level concurrency unchanged.**
10. **`move-latest` concurrency flipped from `cancel-in-progress: true` → `false`.** The top-level concurrency group (`docker-${{ github.ref }}` with `cancel-in-progress: false`) already serializes runs for the ref — two move-latest steps for main cannot overlap, so the old `cancel=true` was dead code. Flipping it to `false` is defense-in-depth: if the top-level group is ever loosened, queued move-latests will now run serially in arrival order instead of cancelling each other, which combined with the ancestor check prevents any starvation mode under rapid pushes. Cost is zero (move-latest is a ~30s registry op). The yaml comment on the job is rewritten to explain the actual serialization source honestly, because the previous comment misleadingly implied move-latest's own group was doing the work.

**Step 1: Write the new workflow file**

Replace `.github/workflows/docker-publish.yml` with this content. Every existing safety invariant is preserved; look for numbered comments calling them out.

```yaml
name: Docker Build and Publish

on:
  push:
    branches: [main]
    paths:
      - '**/*.py'
      - 'pyproject.toml'
      - 'uv.lock'
      - 'Dockerfile'
      - 'docker/**'
      - '.github/workflows/docker-publish.yml'
  release:
    types: [published]

permissions:
  contents: read

# Top-level concurrency: do NOT cancel in-flight builds when a new push lands.
# Every commit deserves its own SHA-tagged image in the registry, and we guard
# the :latest tag in a separate job below (with its own concurrency group) so
# a slow run can't clobber :latest with older bits.
concurrency:
  group: docker-${{ github.ref }}
  cancel-in-progress: false

env:
  IMAGE_NAME: nousresearch/hermes-agent

jobs:
  # ---------------------------------------------------------------------------
  # Build amd64 natively.  This job also runs the smoke tests (basic --help
  # and the dashboard subcommand regression guard from #9153), because amd64
  # is the only arch we can `load` into the local daemon on an amd64 runner.
  # ---------------------------------------------------------------------------
  build-amd64:
    # Only run on the upstream repository, not on forks
    if: github.repository == 'NousResearch/hermes-agent'
    runs-on: ubuntu-latest
    timeout-minutes: 45
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - name: Checkout code
        uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5  # v4
        with:
          submodules: recursive

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@8d2750c68a42422c14e847fe6c8ac0403b4cbd6f  # v3

      # Build once, load into the local daemon for smoke testing.  Cached
      # to gha with a per-arch scope; the push step below reuses every
      # layer from this build.
      - name: Build image (amd64, smoke test)
        uses: docker/build-push-action@10e90e3645eae34f1e60eeb005ba3a3d33f178e8  # v6
        with:
          context: .
          file: Dockerfile
          load: true
          platforms: linux/amd64
          tags: ${{ env.IMAGE_NAME }}:test
          cache-from: type=gha,scope=docker-amd64
          cache-to: type=gha,mode=max,scope=docker-amd64

      - name: Test image starts
        run: |
          mkdir -p /tmp/hermes-test
          sudo chown -R 10000:10000 /tmp/hermes-test
          # The image runs as the hermes user (UID 10000).  GitHub Actions
          # creates /tmp/hermes-test root-owned by default, which hermes
          # can't write to — chown it to match the in-container UID before
          # bind-mounting.  Real users doing `docker run -v ~/.hermes:...`
          # with their own UID hit the same issue and have their own
          # remediations (HERMES_UID env var, or chown locally).
          docker run --rm \
            -v /tmp/hermes-test:/opt/data \
            --entrypoint /opt/hermes/docker/entrypoint.sh \
            ${{ env.IMAGE_NAME }}:test --help

      - name: Test dashboard subcommand
        run: |
          mkdir -p /tmp/hermes-test
          sudo chown -R 10000:10000 /tmp/hermes-test
          # Verify the dashboard subcommand is included in the Docker image.
          # This prevents regressions like #9153 where the dashboard command
          # was present in source but missing from the published image.
          docker run --rm \
            -v /tmp/hermes-test:/opt/data \
            --entrypoint /opt/hermes/docker/entrypoint.sh \
            ${{ env.IMAGE_NAME }}:test dashboard --help

      - name: Log in to Docker Hub
        if: github.event_name == 'push' && github.ref == 'refs/heads/main' || github.event_name == 'release'
        uses: docker/login-action@c94ce9fb468520275223c153574b00df6fe4bcc9  # v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # Push amd64 by digest only (no tag).  The merge job assembles the
      # tagged manifest list.  `push-by-digest=true` is docker's recommended
      # pattern for multi-runner multi-platform builds.
      #
      # We apply the OCI revision label here (and again on arm64) because
      # the move-latest job reads it off the linux/amd64 sub-manifest config
      # of `:latest` to decide whether it's safe to advance.  The label must
      # be on each per-arch image — manifest lists themselves don't carry
      # image config labels.
      - name: Push amd64 by digest
        id: push
        if: github.event_name == 'push' && github.ref == 'refs/heads/main' || github.event_name == 'release'
        uses: docker/build-push-action@10e90e3645eae34f1e60eeb005ba3a3d33f178e8  # v6
        with:
          context: .
          file: Dockerfile
          platforms: linux/amd64
          labels: |
            org.opencontainers.image.revision=${{ github.sha }}
          outputs: type=image,name=${{ env.IMAGE_NAME }},push-by-digest=true,name-canonical=true,push=true
          cache-from: type=gha,scope=docker-amd64
          cache-to: type=gha,mode=max,scope=docker-amd64

      # Write the digest to a file and upload it as an artifact so the
      # merge job can stitch both per-arch digests into a manifest list.
      - name: Export digest
        if: github.event_name == 'push' && github.ref == 'refs/heads/main' || github.event_name == 'release'
        run: |
          mkdir -p /tmp/digests
          digest="${{ steps.push.outputs.digest }}"
          touch "/tmp/digests/${digest#sha256:}"

      - name: Upload digest artifact
        if: github.event_name == 'push' && github.ref == 'refs/heads/main' || github.event_name == 'release'
        uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02  # v4
        with:
          name: digest-amd64
          path: /tmp/digests/*
          if-no-files-found: error
          retention-days: 1

  # ---------------------------------------------------------------------------
  # Build arm64 natively on GitHub's free arm64 runner.  This replaces the
  # previous QEMU-emulated arm64 build, which was ~5-10x slower and shared
  # a cache scope with amd64.
  # ---------------------------------------------------------------------------
  build-arm64:
    if: github.repository == 'NousResearch/hermes-agent' && (github.event_name == 'push' && github.ref == 'refs/heads/main' || github.event_name == 'release')
    runs-on: ubuntu-24.04-arm
    timeout-minutes: 45
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - name: Checkout code
        uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5  # v4
        with:
          submodules: recursive

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@8d2750c68a42422c14e847fe6c8ac0403b4cbd6f  # v3

      - name: Log in to Docker Hub
        uses: docker/login-action@c94ce9fb468520275223c153574b00df6fe4bcc9  # v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Push arm64 by digest
        id: push
        uses: docker/build-push-action@10e90e3645eae34f1e60eeb005ba3a3d33f178e8  # v6
        with:
          context: .
          file: Dockerfile
          platforms: linux/arm64
          labels: |
            org.opencontainers.image.revision=${{ github.sha }}
          outputs: type=image,name=${{ env.IMAGE_NAME }},push-by-digest=true,name-canonical=true,push=true
          cache-from: type=gha,scope=docker-arm64
          cache-to: type=gha,mode=max,scope=docker-arm64

      - name: Export digest
        run: |
          mkdir -p /tmp/digests
          digest="${{ steps.push.outputs.digest }}"
          touch "/tmp/digests/${digest#sha256:}"

      - name: Upload digest artifact
        uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02  # v4
        with:
          name: digest-arm64
          path: /tmp/digests/*
          if-no-files-found: error
          retention-days: 1

  # ---------------------------------------------------------------------------
  # Stitch both per-arch digests into a single tagged multi-arch manifest.
  # This is a registry-side operation — no building, no layer re-push —
  # so it runs in ~30 seconds.  On main pushes it produces :sha-<sha>.
  # On releases it produces :<release_tag_name>.
  # ---------------------------------------------------------------------------
  merge:
    if: github.repository == 'NousResearch/hermes-agent' && (github.event_name == 'push' && github.ref == 'refs/heads/main' || github.event_name == 'release')
    runs-on: ubuntu-latest
    needs: [build-amd64, build-arm64]
    timeout-minutes: 10
    outputs:
      pushed_sha_tag: ${{ steps.mark_pushed.outputs.pushed }}
    steps:
      - name: Download digests
        uses: actions/download-artifact@d3f86a106a0bac45b974a628896c90dbdf5c8093  # v4
        with:
          path: /tmp/digests
          pattern: digest-*
          merge-multiple: true

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@8d2750c68a42422c14e847fe6c8ac0403b4cbd6f  # v3

      - name: Log in to Docker Hub
        uses: docker/login-action@c94ce9fb468520275223c153574b00df6fe4bcc9  # v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # Compute the tag for this run.  Main pushes use sha-<sha> (so every
      # commit gets its own immutable tag); releases use the release tag name.
      - name: Compute tag
        id: tag
        run: |
          if [ "${{ github.event_name }}" = "release" ]; then
            echo "tag=${{ github.event.release.tag_name }}" >> "$GITHUB_OUTPUT"
          else
            echo "tag=sha-${{ github.sha }}" >> "$GITHUB_OUTPUT"
          fi

      - name: Create manifest list and push
        working-directory: /tmp/digests
        run: |
          set -euo pipefail
          # Build the arg array from each digest file (filename = the digest
          # hex, with no sha256: prefix; empty file content, only the name
          # matters).  Using an array avoids shellcheck SC2046 and keeps
          # every digest a single argv token even under pathological names.
          args=()
          for digest_file in *; do
            args+=("${IMAGE_NAME}@sha256:${digest_file}")
          done
          docker buildx imagetools create \
            -t "${IMAGE_NAME}:${TAG}" \
            "${args[@]}"
        env:
          IMAGE_NAME: ${{ env.IMAGE_NAME }}
          TAG: ${{ steps.tag.outputs.tag }}

      - name: Inspect image
        run: |
          docker buildx imagetools inspect "${IMAGE_NAME}:${TAG}"
        env:
          IMAGE_NAME: ${{ env.IMAGE_NAME }}
          TAG: ${{ steps.tag.outputs.tag }}

      # Signal to move-latest that the SHA tag is live.  Only on main pushes;
      # releases don't trigger move-latest (they use their own release tag).
      - name: Mark SHA tag pushed
        id: mark_pushed
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: echo "pushed=true" >> "$GITHUB_OUTPUT"

  # ---------------------------------------------------------------------------
  # Move :latest to point at the SHA tag the merge job pushed.
  #
  # The real serialization guarantee comes from the top-level concurrency
  # group (`docker-${{ github.ref }}` with `cancel-in-progress: false`),
  # which ensures at most one workflow run for this ref executes at a time.
  # That means two move-latest steps for the same ref cannot overlap.
  #
  # This job has its own concurrency group as defense-in-depth: if the
  # top-level group is ever loosened, queued move-latests will run serially
  # in arrival order, each one running the ancestor check below and either
  # advancing :latest or skipping.  `cancel-in-progress: false` matches the
  # top-level setting — we don't want rapid pushes to cancel a queued
  # move-latest, because the ancestor check is the real safety mechanism
  # and queueing is cheap (move-latest is a ~30s registry op).
  #
  # Combined with the ancestor check, this means :latest only ever moves
  # forward in git history.
  # ---------------------------------------------------------------------------
  move-latest:
    if: |
      github.repository == 'NousResearch/hermes-agent'
      && github.event_name == 'push'
      && github.ref == 'refs/heads/main'
      && needs.merge.outputs.pushed_sha_tag == 'true'
    needs: merge
    runs-on: ubuntu-latest
    timeout-minutes: 10
    concurrency:
      group: docker-move-latest-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - name: Checkout code
        uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5  # v4
        with:
          fetch-depth: 1000

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@8d2750c68a42422c14e847fe6c8ac0403b4cbd6f  # v3

      - name: Log in to Docker Hub
        uses: docker/login-action@c94ce9fb468520275223c153574b00df6fe4bcc9  # v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # Read the git revision label off the current :latest manifest, then
      # use `git merge-base --is-ancestor` to check whether our commit is a
      # descendant of it.  If :latest doesn't exist yet, or its label is
      # missing, we treat that as "safe to publish".  If another run already
      # advanced :latest past us (or diverged), we skip and leave it alone.
      - name: Decide whether to move :latest
        id: latest_check
        run: |
          set -euo pipefail
          image=nousresearch/hermes-agent

          # Pull the JSON for the linux/amd64 sub-manifest's config and extract
          # the OCI revision label with jq — Go template field access can't
          # handle dots in map keys, so using json+jq is the robust route.
          image_json=$(
            docker buildx imagetools inspect "${image}:latest" \
              --format '{{ json (index .Image "linux/amd64") }}' \
              2>/dev/null || true
          )

          if [ -z "${image_json}" ]; then
            echo "No existing :latest (or inspect failed) — safe to publish."
            echo "push_latest=true" >> "$GITHUB_OUTPUT"
            exit 0
          fi

          current_sha=$(
            printf '%s' "${image_json}" \
              | jq -r '.config.Labels."org.opencontainers.image.revision" // ""'
          )

          if [ -z "${current_sha}" ]; then
            echo "Registry :latest has no revision label — safe to publish."
            echo "push_latest=true" >> "$GITHUB_OUTPUT"
            exit 0
          fi

          echo "Registry :latest is at ${current_sha}"
          echo "This run is at      ${GITHUB_SHA}"

          if [ "${current_sha}" = "${GITHUB_SHA}" ]; then
            echo ":latest already points at our SHA — nothing to do."
            echo "push_latest=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi

          # Make sure we have the :latest commit locally for merge-base.
          if ! git cat-file -e "${current_sha}^{commit}" 2>/dev/null; then
            git fetch --no-tags --prune origin \
              "+refs/heads/main:refs/remotes/origin/main" \
              || true
          fi

          if ! git cat-file -e "${current_sha}^{commit}" 2>/dev/null; then
            echo "Registry :latest points at an unknown commit (${current_sha}); refusing to overwrite."
            echo "push_latest=false" >> "$GITHUB_OUTPUT"
            exit 0
          fi

          # Our SHA must be a descendant of the current :latest to be safe.
          if git merge-base --is-ancestor "${current_sha}" "${GITHUB_SHA}"; then
            echo "Our commit is a descendant of :latest — safe to advance."
            echo "push_latest=true" >> "$GITHUB_OUTPUT"
          else
            echo "Another run advanced :latest past us (or diverged) — leaving it alone."
            echo "push_latest=false" >> "$GITHUB_OUTPUT"
          fi

      # Retag the already-pushed SHA manifest as :latest.  This is a registry-
      # side operation — no rebuild, no layer re-push — so it's quick and
      # atomic per-tag.  The ancestor check above plus the cancel-in-progress
      # concurrency on this job together guarantee we only ever move :latest
      # forward in git history.
      - name: Move :latest to this SHA
        if: steps.latest_check.outputs.push_latest == 'true'
        run: |
          set -euo pipefail
          image=nousresearch/hermes-agent
          docker buildx imagetools create \
            --tag "${image}:latest" \
            "${image}:sha-${GITHUB_SHA}"
```

**Step 2: Validate YAML syntax locally**

```bash
cd /home/ari/src/hermes-agent
python -c "import yaml; yaml.safe_load(open('.github/workflows/docker-publish.yml'))" && echo OK
# Expected: OK
```

**Step 3: Lint with actionlint** (catches expression errors, bad `needs`, typo'd `runs-on`, etc.)

```bash
# actionlint isn't in the project devshell; run via nix one-shot
nix run nixpkgs#actionlint -- .github/workflows/docker-publish.yml
# Expected: no output (exit 0)
```

Pitfalls to watch for in the actionlint output:
- `runner 'ubuntu-24.04-arm' is not known` — actionlint sometimes lags on new runner labels. If you see this, it's a false positive; confirm with `gh api repos/NousResearch/hermes-agent/actions/runners` or just push to a scratch branch and see.
- `property "pushed_sha_tag" is not defined in object type` — means the `outputs:` block on `merge` is malformed. Recheck it.

**Step 4: Commit**

```bash
git add .github/workflows/docker-publish.yml
git commit -m "ci: split docker-publish into per-arch runners

Build amd64 and arm64 natively on their own GitHub runners in
parallel, then stitch the per-arch digests into a single tagged
multi-arch manifest.  Replaces the previous single-runner pattern
which rebuilt arm64 from scratch on every run because QEMU emulation
+ unscoped GHA cache meant no layer reuse across invocations.

Preserves all existing behavior: per-commit sha-<sha> tags on main,
OCI revision labels, dashboard smoke test, race-safe :latest
advancement via the move-latest job."
```

---

## Task 4: Dry-run on a scratch branch

**Objective:** Verify the new workflow produces green runs on both GitHub-hosted runners and that the merged manifest is valid multi-arch.

The workflow is gated on `github.repository == 'NousResearch/hermes-agent'`, so pushing to a feature branch in the upstream repo exercises the real workflow. The `on: push: branches: [main]` trigger means the workflow **won't fire** from a feature branch push, though — we need to temporarily adjust the trigger to test.

**Two options**:

**Option A (recommended): Temporary trigger broadening.** Add `workflow_dispatch:` to the `on:` block so the workflow can be manually triggered against the feature branch. This is the safest dry run.

**Option B: Open a draft PR and push the same workflow file with the branches filter temporarily including the PR branch, then remove it before merge.** Messier. Don't do this.

**Step 1: Add `workflow_dispatch` trigger**

```bash
cd /home/ari/src/hermes-agent
# Insert workflow_dispatch under `on:`
```

Use `patch` to edit `.github/workflows/docker-publish.yml`:

```
old_string:
on:
  push:
    branches: [main]

new_string:
on:
  workflow_dispatch:  # TEMPORARY — remove before merging
  push:
    branches: [main]
```

**Step 2: Push the branch and trigger**

```bash
git add .github/workflows/docker-publish.yml
git commit -m "ci: TEMP add workflow_dispatch for dry-run (do not merge)"
git push -u origin docker-publish-split-runners

# Trigger manually against this branch
nix run nixpkgs#gh -- workflow run docker-publish.yml --ref docker-publish-split-runners
```

**Step 3: Watch the run**

```bash
# Get the most recent run ID and watch it
sleep 5  # give GitHub a beat to register the run
RUN_ID=$(nix run nixpkgs#gh -- run list --workflow=docker-publish.yml --branch=docker-publish-split-runners --limit=1 --json databaseId -q '.[0].databaseId')
nix run nixpkgs#gh -- run watch "$RUN_ID"
```

**Expected observations:**
- `build-amd64` and `build-arm64` jobs start in parallel
- `build-arm64` runs on `ubuntu-24.04-arm` (visible in the job header on the web UI)
- Both push-by-digest steps succeed on the first run (no existing cache; expected to be slow-ish)
- `merge` runs after both, completes in ~30 seconds
- **Crucially:** on the dry-run run, `merge` will try to push `:sha-<feature-branch-sha>` to Docker Hub. That is acceptable — the SHA tag is immutable and harmless. **However `move-latest` will NOT fire** because its `if:` block requires `github.event_name == 'push' && github.ref == 'refs/heads/main'`. Good — that protects `:latest` during the dry run.

**Step 4: Verify the pushed manifest is multi-arch**

```bash
# On local machine with docker installed
docker buildx imagetools inspect "nousresearch/hermes-agent:sha-$(git rev-parse HEAD)"
```

Expected output includes both platforms:
```
Manifests:
  ...
  Platform:    linux/amd64
  ...
  Platform:    linux/arm64
```

**Step 5: Trigger a second run to verify cache reuse**

```bash
# Push a trivial empty commit to re-trigger
git commit --allow-empty -m "ci: trigger second run for cache verification"
git push

nix run nixpkgs#gh -- workflow run docker-publish.yml --ref docker-publish-split-runners
sleep 5
RUN_ID=$(nix run nixpkgs#gh -- run list --workflow=docker-publish.yml --branch=docker-publish-split-runners --limit=1 --json databaseId -q '.[0].databaseId')
nix run nixpkgs#gh -- run watch "$RUN_ID"
```

**Expected:** both `build-amd64` and `build-arm64` finish noticeably faster than the first run (cache hit on most layers — only the `COPY . .` layer rebuilds because the empty commit changes the git tree). If arm64 takes the same time as the first run, the `scope=docker-arm64` cache isn't working — investigate before merging.

**Step 6: Delete the temporary workflow_dispatch commit**

```bash
# Revert only the TEMP commit, keep the real change
git reset --soft HEAD~1  # uncommit the empty "trigger" commit
git reset HEAD~1 --mixed # uncommit the TEMP commit
git checkout -- .github/workflows/docker-publish.yml  # discard the temp edit
# Verify the only commit left is the real "ci: split ..." one
git log --oneline main..HEAD
# Expected: 2 commits — the plan doc + the workflow rewrite. No TEMP commit.
git push --force-with-lease
```

---

## Task 5: Open the PR

**Objective:** Merge the change through normal review.

**Step 1: Push and open PR**

```bash
nix run nixpkgs#gh -- pr create \
  --title "ci: split docker-publish into per-arch native runners" \
  --body "$(cat <<'EOF'
## Summary

Replace the single-runner QEMU-emulated multi-arch Docker build with a split
where amd64 and arm64 each build natively on their own GitHub runner in
parallel, then a merge job stitches the per-arch digests into a multi-arch
manifest list.

## Why

The previous workflow rebuilt arm64 from scratch on every run because:

1. QEMU emulation is 5-10× slower than native
2. `cache-from: type=gha` used no `scope=`, so amd64 and arm64 shared one
   cache key namespace and effectively clobbered each other
3. 10GB gha cache limit + `mode=max` on a multi-arch build fills fast,
   causing LRU eviction between runs

## What's preserved

- Per-commit `sha-<sha>` tags on main (immutable, race-free)
- `org.opencontainers.image.revision` OCI label embedded in each per-arch
  image (required by `move-latest` to detect ancestor relationships)
- Race-safe `:latest` advancement via the `move-latest` job (unchanged logic,
  now gated on `needs: merge` instead of `needs: build-and-push`)
- Dashboard subcommand smoke test (#9153 regression guard)
- Top-level `cancel-in-progress: false` so concurrent pushes each get their
  own SHA tag

## Plan

See `docs/plans/2026-05-06-docker-publish-split-runners.md` for the full
design discussion and pre-flight checks.

## Dry-run evidence

Ran the workflow manually on this branch twice:

- First run: [link]
- Second run (cache hit): [link]

Inspect of the resulting manifest shows both `linux/amd64` and `linux/arm64`
sub-manifests: [paste `docker buildx imagetools inspect` output]
EOF
)"
```

**Step 2: After merge, verify the first main run**

```bash
# Watch the first real run post-merge
sleep 10
RUN_ID=$(nix run nixpkgs#gh -- run list --workflow=docker-publish.yml --branch=main --limit=1 --json databaseId -q '.[0].databaseId')
nix run nixpkgs#gh -- run watch "$RUN_ID"

# Confirm :latest points at the new sha
docker buildx imagetools inspect nousresearch/hermes-agent:latest \
  --format '{{ json (index .Image "linux/amd64") }}' \
  | jq -r '.config.Labels."org.opencontainers.image.revision"'
# Expected: the sha of the merge commit
```

---

## Post-merge: known remaining inefficiencies

Not fixed by this plan, but worth flagging for future work:

1. **Concurrent pushes on main still share cache scopes.** Two back-to-back pushes both write to `scope=docker-amd64` with `mode=max`; they can thrash each other's cache. Mitigation if this bites: scope per-run with `scope=docker-amd64-${{ github.ref_name }}-${{ github.run_id }}`, but that kills cross-run reuse entirely, so only do it if thrashing is observed.
2. **Registry cache (`type=registry,ref=...:buildcache`) has no 10GB LRU limit** and would be strictly better than `type=gha` for this workload. Switch is ~6 lines but requires writing to Docker Hub from both per-arch jobs, which is already happening, so low risk. Deferred because it adds a second registry push per job and the gha cache is probably fine once scopes are separated.
3. **The first run after merge will be a cold build on both arches** — no existing arm64 cache under the new `scope=docker-arm64` key. One slow build, then fast from there.

---

## Verification checklist (tick before considering the work done)

- [ ] Pre-flight checks all passed (repo public, no job-name references, main green)
- [ ] YAML loads cleanly via `python -c "import yaml; yaml.safe_load(...)"`
- [ ] actionlint passes (ignoring known `ubuntu-24.04-arm` false positive if present)
- [ ] Dry-run on feature branch: both per-arch jobs succeeded
- [ ] Dry-run: second run showed cache reuse (faster than first)
- [ ] Dry-run: `docker buildx imagetools inspect` shows both platforms in the manifest
- [ ] Dry-run: `move-latest` did NOT run (confirmed by inspecting the run's job list — only `build-amd64`, `build-arm64`, `merge`)
- [ ] TEMP `workflow_dispatch` removed from final PR
- [ ] Post-merge first run: `:latest` OCI revision label matches merge commit SHA
- [ ] Post-merge: `docker pull --platform linux/arm64 nousresearch/hermes-agent:latest` on an arm64 host succeeds and `hermes --help` works
