# Kontinuous-integration

> *"The pipeline shall not be broken. Those who break it shall be notified — on Slack."*
> — Komrad Engineering Collective, 2026

This repository is the CI infrastructure of the Komrad collective. It provides shared composite actions and reusable workflows applicable to all projects of the ecosystem. One source of truth. One set of rules. The collective does not duplicate.

```
your-repo/.github/workflows/ci.yml
    ├── uses: rust-security-pipeline.yml
    │            ├── rust/deny       (no traitors in the dependency tree)
    │            └── rust/geiger     (unsafe code is counted, not ignored)
    ├── uses: rust-pipeline.yml
    │            ├── rust/tests      (code that does not pass tests does not exist)
    │            ├── rust/coverage   (untested code is accounted for. ignorance is not.)
    │            ├── rust/fmt        (the collective enforces a single formatting style)
    │            ├── rust/clippy     (warnings are errors. there are no warnings.)
    │            └── rust/release    (the binary is published. the tag is law.)
    ├── uses: security-pipeline.yml
    │            └── gitleaks        (no credential shall be committed)
    ├── uses: container-pipeline.yml
    │            └── container/buildah  (the image is built. the tag is law.)
    ├── uses: container-cleanup-pipeline.yml
    │            └── GHCR API           (stale sha-* images are purged. the registry stays clean.)
    └── uses: notify-pipeline.yml
                 └── notify/slack    (failure is reported. always.)
```

---

## Reusable Workflows

### `rust-pipeline.yml`

The standard CI pipeline for Rust crates. Tests and linting run in parallel. Release is gated on both.

```yaml
jobs:
  ci:
    uses: komrad-company/Kontinuous-integration/.github/workflows/rust-pipeline.yml@main
```

**Job order — not negotiable:**

| Job | Depends on | Condition |
|---|---|---|
| `test` | — | always |
| `lint` | — | always |
| `coverage` | `test` | always |
| `release` | `lint`, `test` | tag push only |

---

### `rust-security-pipeline.yml`

Security audit for Rust crates. Runs `cargo-deny` and `cargo-geiger` in parallel — the collective tolerates neither unlicensed dependencies nor unacknowledged unsafe code.

```yaml
jobs:
  ci:
    uses: komrad-company/Kontinuous-integration/.github/workflows/rust-security-pipeline.yml@main
```

**Jobs:**

| Job | Tool | What it checks |
|---|---|---|
| `rust-deny` | `cargo-deny` | license compliance, advisories, duplication |
| `rust-geiger` | `cargo-geiger` | unsafe code across the full dependency graph |

---

### `rust-grpc-pipeline.yml`

Same discipline as `rust-pipeline.yml`. Additionally installs the protobuf compiler before the collective begins its work — mandatory for crates built on `tonic` and `prost`.

```yaml
jobs:
  ci:
    uses: komrad-company/Kontinuous-integration/.github/workflows/rust-grpc-pipeline.yml@main
```

---

### `security-pipeline.yml`

Secrets detection across the full git history via [gitleaks](https://github.com/gitleaks/gitleaks). No credential shall be committed. No exception shall be granted.

```yaml
jobs:
  security:
    uses: komrad-company/Kontinuous-integration/.github/workflows/security-pipeline.yml@main
```

---

### `container-pipeline.yml`

Builds and publishes OCI images via Buildah. Tag strategy is derived from the calling context — the collective does not guess what you intend to ship.

| Trigger | Tags produced | Push |
|---|---|---|
| Pull request | — | no |
| Push to `develop` | `sha-<short-sha>` | yes |
| Tag `v*-rc.*` | `X.Y.Z-rc.N` | yes |
| Tag `v*` | `X.Y.Z`, `X.Y`, `latest` | yes |

**Job order:**

| Job | Depends on | Condition |
|---|---|---|
| `container` | — | always |

| Input | Required | Default | Description |
|---|---|---|---|
| `image` | yes | — | Image name without registry or tag (e.g. `korelator`) |
| `context` | no | `.` | Build context directory |
| `containerfile` | no | `./Containerfile` | Path to the Containerfile |
| `platforms` | no | `linux/amd64` | Comma-separated target platforms |

The registry is fixed to `ghcr.io/komrad-company`. The caller must grant `packages: write`.

```yaml
jobs:
  container:
    permissions:
      contents: read
      packages: write
    uses: komrad-company/Kontinuous-integration/.github/workflows/container-pipeline.yml@main
    with:
      image: korelator
```

---

### `container-cleanup-pipeline.yml`

Purges `sha-*` tagged images from GHCR older than a configurable retention window. Every development image not promoted to a release candidate within that window is considered expired. Run on a weekly schedule.

| Input | Required | Default | Description |
|---|---|---|---|
| `image` | yes | — | Image name to clean up (e.g. `korelator`) |
| `days` | no | `30` | Delete `sha-*` images older than this many days |

| Secret | Required | Description |
|---|---|---|
| `KOMRAD_GITHUB_TOKEN` | yes | GitHub PAT with `read:packages` and `delete:packages` scopes |

```yaml
on:
  schedule:
    - cron: '0 3 * * 0'

jobs:
  cleanup:
    uses: komrad-company/Kontinuous-integration/.github/workflows/container-cleanup-pipeline.yml@main
    with:
      image: korelator
      days: 30
    secrets:
      KOMRAD_GITHUB_TOKEN: ${{ secrets.KOMRAD_GITHUB_TOKEN }}
```

---

### `notify-pipeline.yml`

Transmits the pipeline status to Slack. Fires on any non-success outcome — failure or cancellation. Success is expected; it warrants no dispatch.

```yaml
jobs:
  notify:
    uses: komrad-company/Kontinuous-integration/.github/workflows/notify-pipeline.yml@main
    with:
      status: ${{ needs.ci.result }}
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

| Input | Required | Description |
|---|---|---|
| `status` | yes | Pipeline status to report (`success`, `failure`, `cancelled`) |

---

## Composite Actions

### `rust/coverage`

Generates an HTML coverage report via [`cargo-tarpaulin`](https://github.com/xd009642/tarpaulin). The report is uploaded as a GitHub Actions artifact named `coverage-report` and retained for 30 days. Coverage does not block the pipeline — it informs it.

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/rust/coverage@main
```

---

### `rust/deny`

Runs [`cargo-deny`](https://github.com/EmbarkStudios/cargo-deny). Every dependency is evaluated for license compliance, security advisories, and duplication. Unapproved crates are rejected by the collective.

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/rust/deny@main
```

---

### `rust/geiger`

Runs [`cargo-geiger`](https://github.com/rust-secure-code/cargo-geiger). Counts unsafe code blocks across the full dependency tree. The collective does not pretend unsafe code does not exist — it counts it.

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/rust/geiger@main
```

---

### `rust/tests`

Runs `cargo test`. Code that does not pass tests does not ship.

| Input | Default | Description |
|---|---|---|
| `release` | `false` | Run tests with `--release` |

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/rust/tests@main
  with:
    release: false
```

---

### `rust/fmt`

Enforces a single formatting style via `cargo fmt --check`. Deviation from the standard is not a matter of taste — it is an error.

| Input | Default | Description |
|---|---|---|
| `continue-on-error` | `false` | Allow formatting errors without failing the job |

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/rust/fmt@main
  with:
    continue-on-error: false
```

---

### `rust/clippy`

Runs `cargo clippy -- -D warnings`. Warnings are promoted to errors. The collective does not tolerate ambiguity.

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/rust/clippy@main
```

---

### `rust/release`

Builds the crate in release mode, collects all executable binaries, and publishes a GitHub Release for the triggering tag. Tags containing `-rc` are marked as pre-releases — not yet ratified by the collective.

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/rust/release@main
```

---

### `proto/setup`

Installs `protobuf-compiler` via `apt`. Required before building any crate that calls upon `tonic` or `prost-build`. The compiler must be present before the collective can speak in gRPC.

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/proto/setup@main
```

---

### `container/buildah`

Builds an OCI image with [Buildah](https://buildah.io) and optionally pushes it to a registry. The collective ships in containers, never in chaos. OCI-format by default; multi-platform via `platforms`. Source, revision and ref labels are stamped automatically.

| Input | Required | Default | Description |
|---|---|---|---|
| `image` | yes | — | Image name without registry or tag (e.g. `korelator`) |
| `tags` | no | `latest` | Space-separated list of tags |
| `containerfile` | no | `./Containerfile` | Path to the Containerfile or Dockerfile |
| `context` | no | `.` | Build context directory |
| `build-args` | no | — | Newline-separated `KEY=value` build arguments |
| `platforms` | no | `linux/amd64` | Comma-separated target platforms |
| `labels` | no | — | Newline-separated OCI labels (added to the default source/revision/ref triplet) |
| `oci` | no | `true` | Produce an OCI-format image instead of Docker v2 |
| `push` | no | `false` | Push the image after build |
| `registry` | no | — | Registry to push to (required when `push=true`) |
| `username` | no | — | Registry username (required when `push=true`) |
| `password` | no | — | Registry password or token (required when `push=true`) |

| Output | Description |
|---|---|
| `image` | Image name produced by Buildah |
| `tags` | Tags applied to the image |
| `image-with-tag` | Fully-qualified reference of the first tag |
| `digest` | Image digest reported by the registry after push |
| `registry-path` | Registry path of the pushed image |

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/container/buildah@main
  with:
    image: korelator
    tags: ${{ github.ref_name }} latest
    containerfile: ./Containerfile
    platforms: linux/amd64,linux/arm64
    push: true
    registry: ghcr.io/komrad-company
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

Buildah must be available on the runner. The `komrad-runners` self-hosted runners provision it.

---

### `notify/slack`

Transmits an urgent dispatch to the Slack channel of the collective when the pipeline fails. If the webhook is absent, silence is maintained — but the failure is not forgotten.

| Input | Required | Default | Description |
|---|---|---|---|
| `webhook-url` | yes | — | Slack incoming webhook URL |
| `status` | yes | `failure` | Status to report (`success`, `failure`, `cancelled`) |

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/notify/slack@main
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
    status: failure
```

---

## Requirements

- **Self-hosted runner** — all jobs target the `komrad-runners` label. The collective runs on its own infrastructure.
- **`SLACK_WEBHOOK_URL` secret** — optional. If absent, Slack notifications are silently skipped. The pipeline does not fail for lack of a webhook.
- **`KOMRAD_GITHUB_TOKEN` secret** — required by `container-cleanup-pipeline.yml`. Must be a GitHub PAT with `read:packages` and `delete:packages` scopes. `GITHUB_TOKEN` cannot delete org-level packages and is not a substitute.

## License

AGPL-3.0-or-later — the source remains open, as all things should be.
