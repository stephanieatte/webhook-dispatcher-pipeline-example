# Buildkite Webhook Dispatcher Pipeline Example

[![Build status](https://badge.buildkite.com/FIXME.svg?branch=main)](https://buildkite.com/buildkite/FIXME)
[![Add to Buildkite](https://img.shields.io/badge/Add%20to%20Buildkite-14CC80)](https://buildkite.com/new)

Buildkite's [GitHub webhook integration](https://buildkite.com/docs/pipelines/source-control/github) is 1:1, each pipeline that wants to build automatically on push needs its own webhook on the repo, and GitHub caps a repo at 20 webhooks. Teams running many pipelines off one repo — a large monorepo, or just lots of small pipelines pointed at the same repo — hit that ceiling. 

This repository is an example Buildkite](https://buildkite.com/) pipeline that works around it, by fanning a single webhook out to many downstream pipelines using trigger steps.

## How it works

Use **one dispatcher pipeline** as the single point of contact with GitHub, and fan out to as many downstream pipelines as needed via `trigger` steps. Only the dispatcher consumes a webhook slot, so the webhook limit no longer scales with pipeline count:

👉 **See this example in action:** [buildkite/FIXME](https://buildkite.com/buildkite/FIXME/builds/latest)

See the full [Getting Started Guide](https://buildkite.com/docs/guides/getting-started) for step-by-step instructions on how to get this running, or try it yourself:

[![Add to Buildkite](https://buildkite.com/button.svg)](https://buildkite.com/new)

<a href="https://buildkite.com/buildkite/FIXME/builds/latest?branch=main">
  <img width="2400" alt="Screenshot of example pipeline build page" src=".buildkite/screenshot.png" />
</a>

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


<!-- docs:start -->

## How it works

Use **one dispatcher pipeline** as the single point of contact with GitHub, and fan out to as many downstream pipelines, only the dispatcher consumes a webhook slot, so the webhook limit no longer scales with pipeline count:

```
GitHub push/PR
      │
      ▼
┌─────────────────────┐
│ dispatcher pipeline │  ← the ONLY pipeline with a GitHub webhook
└─────────┬───────────┘
          │ diffs the changed paths, decides what's affected
          ▼
   ┌──────────────┬──────────────┐
   ▼              ▼              ▼
service-a      service-b     (…any number more)
(no webhook)   (no webhook)   (no webhook)
```

There are two options to do the diffing and routing that decides which pipelines to trigger, and this repo includes both. 
### Option 1: Using the Monorepo-diff Plugin

You can use the monorepo-diff plugin [.buildkite/examples/pipeline.monorepo-diff.yml](https://github.com/stephanieatte/webhook-dispatcher-pipeline-example/blob/main/.buildkite/pipeline.monorepo-diff.yml) to declare both the diff and the path → pipeline routing in its watch config. 

### Option 2: Custom script + trigger steps

Use [.buildkite/pipeline.yml](https://github.com/stephanieatte/webhook-dispatcher-pipeline-example/blob/main/.buildkite/pipeline.yml), which uploads a small script that diffs the changed paths against a routing config and generates a trigger step for each matching pipeline. It gives more headroom if your routing logic outgrows a straight path match."

## License

See [LICENSE](LICENSE) (MIT)
