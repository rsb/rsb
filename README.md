# RSB

**I'm building a software factory to keep me connected to the code.**

I am stepping away from manually writing and into creating the architecture which allows me to directly influence the code generation by defining its rules, checks, heuristics and patterns that make up the code.

>Most of the repos are currently private but will be made public as the system gets created


The factory is a feedback loop around coding agents. I write the rules and the measurements; agents write the code inside a sandbox they cannot escape and with no credentials to anything; a custodian performs every effect on their behalf; CI and reviewers produce evidence; I merge on that evidence instead of reading every diff. When the evidence lies, I add a rule or a template, regenerate the same app from the same tickets, and measure whether it got better. Nothing is adopted on faith, including my own design choices.

The other half of the work is the record: every load-bearing decision is public, argued, dated, and held to. The code is what the record produced.

## The shape of it

<img src="loop.svg" alt="The factory loop: rules and templates feed a spec that becomes vertical-slice tickets; an agent works a ticket in a sandbox with no credentials; a custodian performs every effect; a gate runs tests, checks and behavioural scenarios; a scorecard with merge danger is produced; I merge on evidence; drift, escapes and variance are measured and fed back into the rules" width="680">

Rules are few and mostly prohibitions. Deterministic checks block; judgement calls accrue as debt in a ledger and get paid down on their own timeline. Every profile has a reference app that is regenerated to test the rules, never the other way round.

## Where it lives

<img src="components-by-folder-public.svg" alt="The rsb folder tree: factory/ (gauge, agent-registry, agent-box, agent-host, rsb-skills), projects/ (services, desktop, lib), ref/ (reference apps), research/ (the record); outside the tree the store, the always-on desktop, GitHub and the agents" width="680">

One folder per repo role. The factory never hosts an agent attempt; agents only ever work in `projects/` and `ref/`. The loop runs on an always-on desktop under systemd and is driven from the issue tracker, so work can be started from anywhere.

## The factory

- [gauge](https://github.com/rsb/gauge) — the engine: a stateless reconciler that derives each run from git and GitHub, supervised by systemd, Rust. Holds the driver that alone touches GitHub.
- `agent-box` — the sandbox an attempt runs in. Default-deny egress, no secrets, read-only everything the agent doesn't own.
- `agent-registry` — rules, heuristics, checks, patterns and profiles, as TOML, versioned.
- `rsb-skills` — the spec, ticket and hand-back skills the agents and I both use.
- [rsb.sh](https://github.com/rsb/rsb.sh) — the engineering record and, eventually, the published rules and data so you can build your own.

The private pieces open as they stabilize; the rules and the data are the point of publishing, not the code.

## What it builds

- **Lumen** — a free, professional-grade raw photo editor for the Linux desktop. Local files, no subscriptions, no telemetry, no cloud. Rust. ([libraw](https://github.com/rsb/libraw) bindings and [errors-rs](https://github.com/rsb/errors-rs) are its first open crates.)
- **Rollout** — feature-flag management on Kubernetes. The reference application for the k8s profile.
- **Daybook** — double-entry ledger accounting for small creative businesses. The public example of a complete serverless system, with a live demo when it ships.
- **Reference apps** — one per stack profile, deliberately small, rebuilt from fixed tickets every time a rule changes.

Three families of software, each with its own profile: serverless TypeScript on Cloudflare, rust and typescript for Kubernetes services, and Rust desktop applications.
