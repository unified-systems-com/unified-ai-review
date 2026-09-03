# Unified AI Review

Two-stage, fork-covering AI code review for GitHub pull requests — machinery only, **bring your
own prompts**.

Most AI review products make you choose: hand a vendor's GitHub App write access to your code, or
lose coverage of contributor (fork) PRs because GitHub withholds repository secrets from fork
runs. Unified AI Review refuses both: it runs entirely in **your** CI under permissions **you**
author, reviews **every** PR including forks, and no vendor holds any key to your org.

## How it works

```
  PR opened (fork or not)
       │
       ▼
  STAGE 1 — capture  (pull_request, UNPRIVILEGED)
  No secrets, no write permissions. Computes the PR diff
  and uploads it as an artifact. Data, never code.
       │
       ▼
  STAGE 2 — review  (workflow_run, PRIVILEGED, base-repo context)
  Holds the vendor API keys. NEVER checks out or executes PR
  content — the diff is consumed as text. Resolves PR identity
  from its own event context, never from the artifact. Runs
  deterministic screens (binary/image additions, invisible
  Unicode, hidden comments, opaque blobs) plus one job per
  model seat, then posts a single advisory comment.
```

The invariant: **the privileged stage treats the PR as a document, never as a program.** A prompt
injection in a PR can, at worst, produce a wrong comment — no seat holds any write path to code.
Seats fail loud: a reviewer that produces no verdict is a red job plus an explicit absence marker,
never a silent skip. Injection indicators fail the run red and raise an out-of-band alert issue.

## Consuming it

Two thin shims in your repo. Pin everything by full commit SHA — the machinery, and your prompts.

`.github/workflows/ai-review-capture.yml`:

```yaml
name: AI review capture
on:
  pull_request:
    types: [opened, synchronize, reopened]
concurrency:
  group: ai-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true
permissions: {}
jobs:
  capture:
    permissions:
      contents: read
    uses: unified-systems-com/unified-ai-review/.github/workflows/capture.yml@<MACHINERY_SHA>
```

`.github/workflows/ai-review.yml`:

```yaml
name: AI review
on:
  workflow_run:
    workflows: ["AI review capture"]
    types: [completed]
concurrency:
  group: ai-review-run-${{ github.event.workflow_run.head_sha }}
  cancel-in-progress: true
permissions: {}
jobs:
  review:
    if: github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'pull_request'
    permissions:
      contents: read
      pull-requests: write
      issues: write
    uses: unified-systems-com/unified-ai-review/.github/workflows/review.yml@<MACHINERY_SHA>
    with:
      prompts-repo: your-org/your-prompts-repo
      prompts-ref: <PROMPTS_SHA>
      prompt-pack: security
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      XAI_API_KEY: ${{ secrets.XAI_API_KEY }}
```

Your prompts repo follows one contract: `packs/<name>/prompt.md`. The reference packs live in
[unified-ai-review-prompts](https://github.com/unified-systems-com/unified-ai-review-prompts).
Because the shim (base branch) pins the prompts SHA, a PR under review can never edit the
instructions being applied to it — and a prompt change in your repo is a reviewable pin bump.

### Inputs (`review.yml`)

| Input | Default | Meaning |
| --- | --- | --- |
| `prompts-repo` | `unified-systems-com/unified-ai-review-prompts` | Repo holding prompt packs |
| `prompts-ref` | *(required)* | FULL commit SHA of the prompts repo |
| `prompt-pack` | `security` | Pack under `packs/` to apply |
| `openai-model` | `gpt-5.5` | Model id for the OpenAI seat |
| `xai-model` | `grok-4.6` | Model id for the xAI seat |
| `diff-artifact` | `unified-ai-review-diff` | Artifact name from capture |

Secrets: `OPENAI_API_KEY`, `XAI_API_KEY` — mint them restricted (inference only) in dedicated,
hard-spend-capped vendor projects.

## Consuming these reviews

A posted review only helps if someone reads it — including the suppressed findings. The
reference implementation of the watch-and-triage pattern is TAP's
[`scripts/pr-review-triage`](https://github.com/unified-systems-com/tap/blob/main/scripts/pr-review-triage)
(link, not copy: the canonical version lives in TAP, where its push-workflow spec governs it —
a vendored copy here would be a drift surface nothing polices).

What it gives an adopter:

- **One-shot mode** — the authoritative read: every review on the PR, with suppressed findings
  (`<details>` blocks) surfaced for conscious triage rather than silent scroll-past.
- **`--watch` mode** — one line per event, built to sit under a monitor:
  `WATCHING` / `REVIEW` / `COMMENT` / `MERGESTATE` / `CHECKFAIL` / `CHECKRECOVERED` /
  `FETCHFAIL` / `TERMINAL`. Per-check `CHECKFAIL` fires the moment an individual check (a seat
  run, a scanner, a CI lane) goes red — findings are workable before the full gate resolves.
  Signatures are edit-aware (position-weighted body digest for reviews, `updatedAt` for
  comments); consecutive fetch failures are reported every tenth one (`FETCHFAIL`) while it
  keeps retrying, so a watcher never mistakes an auth or rate-limit outage for a quiet PR.
- **Coverage contract, honestly stated**: each event is detected within one comment page; the
  one-shot read is the authoritative record.

The only thing an adopter tunes is the bot-name regex.

## Design provenance

Built for and specified by [TAP](https://github.com/unified-systems-com/tap) —
`specs/spec-cicd-ai-review.md` there is the governing spec (reviewer least privilege, untrusted
PR content, fail-loud seats, the verdict ledger). Reviews are **advisory**: no bot approval is
ever load-bearing for a merge.

## License

Apache-2.0. This repository deliberately contains no prompt content — prompts live in their own
repo under their own licenses.
