# FlowHarness Vibe Check

FlowHarness Vibe Check evaluates an agent-context change by replaying committed cases. The local
replay gate is deterministic and offline. The GitHub Action adds a step-summary report and, for a
same-repository pull request, creates or updates one sticky comment.

Available today: FlowHarness Scan and FlowHarness Vibe Check.

## Prerequisites and local setup

Install [uv](https://docs.astral.sh/uv/getting-started/installation/) and run these developer
commands from a Git repository with its agent-context files tracked. First initialize FlowHarness
configuration without generating another workflow or hook:

```console
uvx --no-config --no-sources --from flowharness==0.1.2 \
  flowharness init --dir . --hook none --workflow none
```

Then generate starter replay cases, the suite lockfile, and the starter gate policy:

```console
uvx --no-config --no-sources --from flowharness-ci-runner==0.1.2 \
  flowharness-ci seed
```

`flowharness-ci seed` is a developer command, not a CI step. Review and commit the generated
`.flowharness/cases/`, `.flowharness/suite.lock.json`, `.flowharness/gate-policy.toml`, and
FlowHarness configuration before enabling the Action. Replay requires those committed files, their
case fixtures, and full Git history so the runner can resolve the merge base.

To exercise the released local runner before opening a pull request:

```console
uvx --no-config --no-sources --from flowharness-ci-runner==0.1.2 \
  flowharness-ci vibe-check --base origin/main --executor replay
```

An absent gate policy defaults safely to `NEEDS_HUMAN`; an unconfigured repository never silently
auto-passes.

## Add the Vibe Check Action

Copy this least-privilege workflow to `.github/workflows/flowharness-vibe-check.yml`:

```yaml
name: FlowHarness Vibe Check
"on":
  pull_request:
permissions:
  contents: read
  pull-requests: write
jobs:
  vibe-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: flowharness-ai/vibe-check-action@v1
        with:
          base: origin/main
          executor: replay
          suite: .flowharness/suite.lock.json
          gate-policy: .flowharness/gate-policy.toml
          upload: ${{ secrets.FLOWHARNESS_TOKEN != '' }}
        env:
          FLOWHARNESS_TOKEN: ${{ secrets.FLOWHARNESS_TOKEN }}
          FLOWHARNESS_API_URL: ${{ vars.FLOWHARNESS_API_URL }}
```

`fetch-depth: 0` is required for the merge base. The Action appends the report to the GitHub step
summary on every pull request. On a same-repository pull request it also creates or updates one
marker-keyed sticky comment, identified by `<!-- flowharness-vibe-check -->`; re-pushes update that
comment instead of creating duplicates. A comment-post failure is non-fatal—the local replay
verdict still determines the job result.

The workflow intentionally grants only `contents: read` and `pull-requests: write`. It uses the
`pull_request` trigger and must not be changed to `pull_request_target` to regain write access or
secrets while evaluating untrusted pull-request content.

## Forks, tokens, and optional upload

Fork pull requests do not receive repository secrets or the write token needed for a comment. The
Action skips the sticky comment, emits a fixed notice, keeps the complete report in the step
summary, and propagates the local replay verdict. It does not elevate privileges to compensate.

`github.token` is used only for the same-repository sticky comment. `FLOWHARNESS_TOKEN` is a
separate optional FlowHarness platform signing key, and `FLOWHARNESS_API_URL` is the instance's
HTTPS base URL; the client appends `/v1/evaluation/runs`. Upload is telemetry, never judgment: its
success, failure, or absence does not change the local replay verdict.

The example enables upload only when `FLOWHARNESS_TOKEN` is present. Keep both
`FLOWHARNESS_TOKEN` and `FLOWHARNESS_API_URL` on the invoking Action step exactly as shown—never
in `with`, workflow-level `env`, or job-level `env`. Step scope limits exposure, although child
processes of a composite Action inherit that step environment. With an absent token, upload stays
disabled; this is the normal fork behavior.

The fixed acquisition URL is [flowharness.ai/go/vibe-check](https://flowharness.ai/go/vibe-check).
It is privacy-safe: do not add repository names, owners, branches, paths, finding text, report
bodies or hashes, tokens, secrets, or checkout-controlled query values to that URL.

## Version matrix and supply chain

| Surface | Released version | Embedded Python artifact |
| --- | --- | --- |
| Local CLI | 0.1.2 | flowharness 0.1.2 |
| Vibe Check Action | v1.0.1 | flowharness-ci-runner 0.1.1 |

Vibe Check Action v1.0.1 embeds runner 0.1.1. The local released setup above uses
FlowHarness/runner 0.1.2; it is a separately released local toolchain and does not change the
Action's embedded runtime. Major Action tags make the example easy to adopt. Organizations that
require immutable supply-chain inputs should replace `@v1` with the relevant immutable release
commit SHA and SHA-pin every third-party Action according to their policy.

## Troubleshooting and public guides

- **The Action cannot determine a base.** Confirm `fetch-depth: 0`, committed replay fixtures, the
  suite lockfile, and the gate policy.
- **There is no pull-request comment.** Confirm the pull request originates in the same repository
  and that the workflow keeps `pull-requests: write`; fork pull requests intentionally receive the
  summary and fixed notice instead.
- **Upload is skipped.** An empty `FLOWHARNESS_TOKEN` disables upload; a token also requires a
  valid HTTPS `FLOWHARNESS_API_URL`.
- **Versions differ.** The Action's embedded runner is 0.1.1, while the local released setup is
  0.1.2; use the version matrix above to choose the surface you are debugging.

For setup, read the [public Vibe Check guide](https://github.com/flowharness-ai/flowharness/blob/main/docs/vibe-check.md).
For permissions, tokens, and data handling, read the [privacy guide](https://github.com/flowharness-ai/flowharness/blob/main/docs/privacy-permissions-and-tokens.md).
Browse the [Vibe Check Action source](https://github.com/flowharness-ai/vibe-check-action).

## From one repository to organizational governance

FlowHarness is open-core. Its local tools and GitHub Actions are free and Apache-2.0. The
organization-wide governance and evaluation control plane is a commercial product available as
managed SaaS, on-premises, and air-gapped enterprise deployments.

The free tools remain useful without an account. Teams that need shared policy across repositories,
human sign-off, signed agent-context distribution, fleet convergence, or centralized evaluation
evidence can discuss a design-partner deployment at [flowharness.ai](https://flowharness.ai/).

Licensed under the [Apache License 2.0](LICENSE).
