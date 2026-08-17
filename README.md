# Buildkite Webhook Dispatcher Pipeline Example

[![Build status](https://badge.buildkite.com/FIXME.svg?branch=main)](https://buildkite.com/buildkite/FIXME)
[![Add to Buildkite](https://img.shields.io/badge/Add%20to%20Buildkite-14CC80)](https://buildkite.com/new)

This repository is an example [Buildkite](https://buildkite.com/) pipeline for fanning a single GitHub webhook out to many downstream pipelines using `trigger` steps — the recommended workaround for GitHub's 20-webhook-per-repo limit.

👉 **See this example in action:** [buildkite/FIXME](https://buildkite.com/buildkite/FIXME/builds/latest)

See the full [Getting Started Guide](https://buildkite.com/docs/guides/getting-started) for step-by-step instructions on how to get this running, or try it yourself:

[![Add to Buildkite](https://buildkite.com/button.svg)](https://buildkite.com/new)

<a href="https://buildkite.com/buildkite/FIXME/builds/latest?branch=main">
  <img width="2400" alt="Screenshot of example pipeline build page" src=".buildkite/screenshot.png" />
</a>

<!-- docs:start -->
## The problem

Buildkite's [GitHub webhook integration](https://buildkite.com/docs/pipelines/source-control/github) is 1:1 — each pipeline that wants to build automatically on push needs its own webhook on the repo, and GitHub caps a repo at **20 webhooks**. Teams running many pipelines off one repo — a large monorepo, or just lots of small pipelines pointed at the same repo — hit that ceiling.

## How it works

Use **one dispatcher pipeline** as the single point of contact with GitHub, and fan out to as many downstream pipelines as needed via `trigger` steps:

```
GitHub push/PR
      │
      ▼
┌─────────────────────┐
│  dispatcher pipeline │  ← the ONLY pipeline with a GitHub webhook
└─────────┬────────────┘
          │ inspects diff / branch, decides what's affected
          ▼
   ┌──────────────┬──────────────┐
   ▼              ▼              ▼
service-a      service-b     (…any number more)
(no webhook)   (no webhook)   (no webhook)
```

Only the dispatcher consumes a webhook slot. Downstream pipelines are ordinary Buildkite pipelines — they're just started via `trigger` instead of GitHub, so the webhook limit no longer scales with pipeline count.

This repo includes **three dispatcher variants**, so you can pick whichever fits your routing logic best. Only run one of the three against an actual GitHub webhook — they're alternatives, not complementary layers.

| Variant | File | Routes on | Best fit |
|---|---|---|---|
| Custom script (default — this repo's public pipeline) | `.buildkite/pipeline.yml` + `.buildkite/scripts/generate-trigger-steps.py` | path prefixes, via `.buildkite/routes.yml` | Routing logic more complex than path matching (PR labels, manifest files, custom rules) |
| [`monorepo-diff` plugin](https://github.com/buildkite-plugins/monorepo-diff-buildkite-plugin) | `.buildkite/examples/pipeline.monorepo-diff.yml` | path prefixes, via plugin `watch` config | Straightforward path-based routing, no script to maintain |
| Native `if` conditionals | `.buildkite/examples/pipeline.conditional-trigger.yml` | branch, tag, commit message — **not** file diff | Routing by branch/environment/tag rather than by changed files |

### Variant 1: the custom script (default)

`.buildkite/pipeline.yml` runs one step: it executes `.buildkite/scripts/generate-trigger-steps.py` and pipes the output straight into `buildkite-agent pipeline upload`. This is the standard [dynamic pipelines](https://buildkite.com/docs/pipelines/configure/dynamic-pipelines) technique — the script's stdout *is* the next set of steps.

The script:

1. Figures out which files changed (diffing against the PR base branch for PR builds, or the previous commit for pushes — falling back to "treat the whole tree as changed" if neither is available).
2. Loads `.buildkite/routes.yml` and checks each route's path prefixes against the changed files.
3. Also checks `full_rebuild_branches` (e.g. `main`) — on those branches, every routed pipeline triggers regardless of diff.
4. Emits a `trigger` step for every matched pipeline (or an informational no-op step if nothing matched), forwarding the branch, commit, message, and a bit of env context about the triggering build.

### Variant 2: the `monorepo-diff` plugin

`.buildkite/examples/pipeline.monorepo-diff.yml` uses the official [`monorepo-diff-buildkite-plugin`](https://github.com/buildkite-plugins/monorepo-diff-buildkite-plugin) instead of a custom script. The plugin diffs the changed files itself and triggers a downstream pipeline for each `watch` entry whose `path` matches something in the diff — the plugin-config equivalent of `routes.yml`. It also supports `if` conditions per watched path, so you can layer a branch/tag check on top of a path match without leaving the plugin config.

Its default `diff` command (`git diff --name-only HEAD~1`) compares against the previous commit, which is fine for simple pushes but usually wrong for PR builds — the file has a comment showing how to point it at the PR base branch instead.

### Variant 3: native `if` conditionals

`.buildkite/examples/pipeline.conditional-trigger.yml` uses Buildkite's built-in [conditional expressions](https://buildkite.com/docs/pipelines/configure/conditionals) (the `if` attribute) directly on `trigger` steps — no diffing, no plugin. Each step's `if` checks things like `build.branch`, `build.tag`, or `build.message`, and Buildkite decides at pipeline-upload time whether to create that step at all. This is the simplest option when your routing question is "what kind of build is this" (a hotfix branch, a tagged release, a commit flagged `[skip smoke]`) rather than "which files changed".

## Repository layout

```
.
├── .buildkite/
│   ├── pipeline.yml                        # dispatcher variant 1: custom script (this repo's public pipeline)
│   ├── routes.yml                          # declarative path → pipeline routing table (edit this)
│   ├── scripts/
│   │   └── generate-trigger-steps.py       # reads routes.yml + git diff, emits `trigger` steps
│   └── examples/
│       ├── pipeline.monorepo-diff.yml       # dispatcher variant 2: monorepo-diff plugin
│       └── pipeline.conditional-trigger.yml # dispatcher variant 3: native `if` conditionals
├── services/
│   ├── service-a/.buildkite/pipeline.yml   # downstream pipeline #1 (no webhook)
│   └── service-b/.buildkite/pipeline.yml   # downstream pipeline #2 (no webhook)
└── libs/
    └── shared-proto/                       # a shared dependency, used to show routing on
                                             # a path that isn't a single service's own directory
```

`services/service-a/.buildkite/pipeline.yml` and `services/service-b/.buildkite/pipeline.yml` are ordinary pipelines with nothing dispatcher-specific about them. In the Buildkite dashboard each is its own pipeline entity with its GitHub webhook **left disabled**, and its "Steps" configuration pointing at its own `pipeline.yml` path in this repo. They only ever run when the dispatcher triggers them.



## License

See [LICENSE](LICENSE) (MIT)
