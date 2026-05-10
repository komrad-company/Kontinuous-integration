# Kontinuous-integration

> *"The pipeline shall not be broken. Those who break it shall be notified — on Slack."*
> — Komrad Engineering Collective, 2026

This repository is the CI infrastructure of the Komrad collective. It provides shared composite actions and reusable workflows applicable to all projects of the ecosystem. One source of truth. One set of rules. The collective does not duplicate.

```
your-repo/.github/workflows/ci.yml
    └── uses: rust-pipeline.yml
             ├── rust/deny       (no traitors in the dependency tree)
             ├── rust/tests      (code that does not pass tests does not exist)
             ├── rust/fmt        (the collective enforces a single formatting style)
             ├── rust/clippy     (warnings are errors. there are no warnings.)
             ├── rust/release    (the binary is published. the tag is law.)
             └── notify/slack    (failure is reported. always.)
```

---

## Reusable Workflows

### `rust-pipeline.yml`

The standard CI pipeline for Rust crates. The collective runs security first. Nothing else proceeds until the dependency tree is cleared.

```yaml
jobs:
  ci:
    uses: komrad-company/Kontinuous-integration/.github/workflows/rust-pipeline.yml@main
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**Job order — not negotiable:**

| Job | Depends on | Condition |
|---|---|---|
| `rust-security` | — | always |
| `test` | `rust-security` | always |
| `lint` | `rust-security` | always |
| `release` | `lint`, `test` | tag push only |
| `notify-slack` | all | on any failure |

---

### `rust-grpc-pipeline.yml`

Same discipline as `rust-pipeline.yml`. Additionally installs the protobuf compiler before the collective begins its work — mandatory for crates built on `tonic` and `prost`.

```yaml
jobs:
  ci:
    uses: komrad-company/Kontinuous-integration/.github/workflows/rust-grpc-pipeline.yml@main
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

### `security-pipeline.yml`

Secrets detection across the full git history via [gitleaks](https://github.com/gitleaks/gitleaks). No credential shall be committed. No exception shall be granted.

```yaml
jobs:
  security:
    uses: komrad-company/Kontinuous-integration/.github/workflows/security-pipeline.yml@main
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Composite Actions

### `rust/deny`

Runs [`cargo-deny`](https://github.com/EmbarkStudios/cargo-deny). Every dependency is evaluated for license compliance, security advisories, and duplication. Unapproved crates are rejected by the collective.

```yaml
- uses: komrad-company/Kontinuous-integration/.github/actions/rust/deny@main
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

## License

AGPL-3.0-or-later — the source remains open, as all things should be.