# RSB

**Rust crates for creatives who want to build local-first desktop applications they own.**

RSB is an ecosystem of Rust crates for building desktop applications that run on your machine, store their data on your disk, and answer to no one but the person using them. No subscriptions. No telemetry. No cloud lock-in. No quiet decisions made by a vendor about what you're allowed to do with your own work.

The motivating use case is creative work — photography, illustration, design — where the current generation of professional tools has drifted toward subscription rentals, cloud-dependent workflows, and surveillance-by-default. RSB is the foundation for tools that go the other direction.

## What's in the ecosystem

RSB is currently in active early development. The architecture is settled at the macro level; the crates are being built and refined.

The ecosystem is organized into two tiers of reusable crates plus the applications that consume them.

The **desktop tier** provides the building blocks any desktop application would need — shells, the failure system, reporting and telemetry, the worker model, window and surface lifecycle, UI primitives. These crates live in `libs/desktop/`.

The **imaging tier** provides shared photographic primitives — high-precision color-managed pixel pipelines, raw decode abstractions, the node-graph editing engine, the document model. These crates live in `libs/camera/`.

The **applications** are the products that compose those tiers into specific tools. They live in `apps/`. The intermediate applications (a raw image viewer, a device-configuration tool, and others) each force a specific capability into existence and ship as useful artifacts in their own right.

## The opinion

RSB is opinionated. It encodes a specific position on how to build desktop applications in Rust — a layered architecture with strict dependency direction, a particular threading and event-loop model, a specific approach to failure handling and reporting, and a set of conventions for how applications wire themselves together. That position will be documented in detail as the ecosystem matures.

The opinion is not a layer on top of neutral primitives. The primitives themselves encode it. A consumer of RSB's desktop crates is adopting the RSB way of building desktop applications, not just using a collection of unrelated utilities. If that's appealing, the documentation will tell you why. If it isn't, the crates are dual-licensed (MIT OR Apache-2.0) and you can use whatever pieces fit your own approach.

## Repository layout

This repository is a Cargo workspace. Its top level is organized by the purpose of each crate, not by its layer position:

```
rsb/
├── apps/          runnable applications — intermediate apps, demonstrations
├── libs/          library crates — the reusable building blocks
│   ├── camera/    imaging-domain crates
│   └── desktop/   desktop-domain crates
├── examples/      cross-cutting examples demonstrating integration patterns
└── docs/          architecture, design notes, contribution guide
```

Within `libs/`, crates are organized first by domain (`camera/`, `desktop/`) and then by layer position (`base/`, `runtime/`, `substrate/`, and so on). Crate names carry their domain — `rsb-desktop-fail`, `rsb-camera-raw-decode` — because the abstractions are domain-shaped and the names should be honest about that.

## Status

RSB is in early development. The architecture has been worked out in detail; the crates are being built incrementally through a sequence of intermediate applications, each of which forces a specific capability into existence and ships as a useful artifact in its own right. None of the crates are yet published to crates.io. They will be published individually as they stabilize.

## License

All crates in this repository are dual-licensed under either of:

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option. This is the standard Rust ecosystem convention and means you can use RSB's crates in any project, including closed-source and commercial work, under whichever of the two licenses suits your needs.

## Contributing

Contributions are welcome, and contributor guidance lives in [`CONTRIBUTING.md`](CONTRIBUTING.md).

Before opening a substantial PR, please familiarize yourself with the architecture — RSB has strong architectural commitments, and changes that touch the structure are most productive when they engage with the existing rationale.

## Why "RSB"

RSB is the initials of the author. The ecosystem carries the name because the opinion is the author's, not a generic neutral framework. Building tools for creatives is a personal project with a clear point of view; the name reflects that.
