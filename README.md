# FlowHarness Vibe Check

`flowharness-ai/vibe-check-action` runs the FlowHarness Vibe Check in a pull request, writes the
full report to the step summary, and posts or updates a same-repository PR comment when permitted.

## Usage

```yaml
# consumer-side workflow — the whole integration story
name: FlowHarness Vibe Check
on:
  pull_request:
    paths: ["CLAUDE.md", ".claude/**", ".cursor/**", ".cursorrules", "AGENTS.md",
            ".continue/**", ".opencode/**", "playbook/**", ".flowharness/**",
            "**/crew*.py", "**/agents.y*ml", "**/tasks.y*ml"]
permissions:
  contents: read
  pull-requests: write        # sticky comment only
jobs:
  vibe-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with: { fetch-depth: 0 }          # diffscope needs the merge base
      - uses: flowharness-ai/vibe-check-action@v1
        with:
          executor: replay                 # 'live' requires provider_key
          upload: ${{ secrets.FLOWHARNESS_TOKEN != '' }}
        env:
          FLOWHARNESS_TOKEN: ${{ secrets.FLOWHARNESS_TOKEN }}
```

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `base` | no | `origin/main` | git ref compared with HEAD; checkout needs full history |
| `executor` | no | `replay` | offline replay; `live` is not wired in 0.1.1 |
| `suite` | no | `.flowharness/suite.lock.json` | committed suite lockfile |
| `gate-policy` | no | `.flowharness/gate-policy.toml` | committed gate policy |
| `upload` | no | `false` | opt-in signed run upload when exactly `true` |

## Forks and tokens

Fork PRs do not attempt a sticky comment. The report body remains in the step summary and the
action emits one fixed `::notice`. The same-repository sticky-comment path is not claimed as
live-proven here; B7 owns that proof.

`github.token` is internally mapped to `GH_TOKEN` only for same-repository sticky-comment posting.
`FLOWHARNESS_TOKEN` is a separate optional consumer secret, scoped to the action step, and is used
only to sign and authenticate upload when `upload: true`. It is not an action input, is not
required for local gating or comments, and must not be printed or solicited.

## Source and license

The canonical source for this action is
[`flowharness/.github/actions/vibe-check/action.yml`](https://github.com/suleimanmahmoud/flowharness/blob/main/.github/actions/vibe-check/action.yml);
this repository is a one-way public export of that source.

Licensed under the [Apache License 2.0](LICENSE).
