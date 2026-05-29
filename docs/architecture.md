# The RSB Ecosystem — Architecture, Principles, and Roadmap

*A planning document and knowledge base for the RSB ecosystem of local-first desktop applications for creatives, and its
target application, The Lab.*

---

## 0. How to read this document

This is both a planning artifact and the seed of a public knowledge base (the articles will be drawn from it). It is
organized so that every architectural conclusion is paired with the reasoning that forced it, because in this project
the *justifications* are load-bearing: several decisions are correct only because they are over-determined — justified
independently on product, correctness, and performance grounds at once. Where a decision was reached by discovering that
a thing we already had was a special case of something more general, that is noted explicitly, because that pattern
recurs and is worth recognizing.

Two recurring principles run underneath nearly every decision here and are worth stating up front, because they function
as design compasses:

**The generality-first principle.** Build the general thing first, then derive the constrained thing from it as a
projection — never the reverse. A constrained surface depends on the general system it reduces; building it first means
inventing it in a vacuum.

**The honest-seam principle (user agency through honesty).** The system stays trustworthy by being explicit about the
boundary of what any given surface or operation can express, and never pretends a thing is editable, reversible, or
preserved when it is not. Destruction, flattening, and loss of fidelity are legitimate — but only as deliberate, named
acts the user knowingly chooses, never as silent side effects.

---

## 1. What the RSB ecosystem is

RSB is a system of Rust crates for building **local-first desktop applications for creatives**. Its mission is to give
creative professionals tools they own outright — no subscriptions, no surveillance, no cloud dependency — built in Rust
for reliability, security, and performance.

The ecosystem is large, so it is being designed as a layered system of components rather than a monolith, and it is
being built through a sequence of smaller, genuinely useful applications that each force a specific capability into
existence before the target application needs it at full scale.

### The target application: The Lab (rsb-lab)

The Lab is a from-scratch raw processor and image post-processing editor intended to genuinely replace the combination
of tools a serious photographer uses today (e.g. DXO for raw development plus Affinity Photo for grading and local
work). It is too large to build directly, so it functions as the north star that pulls the architecture into existence.
The intermediate applications are how we avoid designing The Lab's hardest subsystems in a vacuum.

---

## 2. The three-tier layering

The most important structural discovery of the architecture work is that the ecosystem is **three tiers, not two**. The
original mental model was a generic desktop framework underneath specific apps. Working through what must be *shared
between multiple photo apps* revealed a distinct middle tier of imaging-domain primitives that are neither
generic-desktop nor app-specific.

### Desktop domain

Used by *any* desktop app in the ecosystem, imaging or not.

- **Owns:** Shells, the Area/Surface/Region workspace system, UI components and widget toolkit, the failure system,
  reporting, telemetry/observability, worker/threading model, window & surface lifecycle
- **Examples:** shell core, shell topologies, widget/panel system, workers, fail, report, substrate

### Imaging domain (shared photographic concerns)

Used by every photo app in the ecosystem.

- **Owns:** The high-precision pixel pipeline, color management, raw decode abstraction, the node-graph editing engine,
  the `.rsb` / document model
- **Examples:** precision pixel type, color/ICC, raw-decode port, node engine, document model

### Applications

Used by end users.

- **Owns:** The specific product: which surfaces are present, which node vocabulary, the workflow
- **Examples:** The Lab, raw viewer, raw-developer, post/composite app, device-config app

### Why the imaging tier is distinct from the desktop tier

A non-photo application using RSB's desktop crates (a text editor, a 3D tool, a device-configuration utility) would
never touch a color-managed float pixel pipeline, raw decode, or a photographic node graph — yet *every* photo
application must share exactly those things. That is the precise definition of a distinct tier: shared by one family of
apps, irrelevant to the rest, and sitting above the desktop tier but below any specific product.

This also resolves a long-standing tension in the existing prototype's dependency graph. The current crates are prefixed
`lab-` (`lab-image`, `lab-raw-decoder`, `lab-libraw`, etc.) because they were born inside The Lab before it was
understood that they are *ecosystem imaging primitives*, not Lab-internal code. The `lab-` prefix on those crates is a
**naming fossil**; on generalization they are re-homed into the imaging tier and re-prefixed (`rsb-camera-…`). The
much-discussed `canvas → image` coupling is not an awkward dependency to apologize for — it is a real tier boundary: the
**canvas is desktop-tier** (must not know it is compositing a photograph), and the **image pipeline is imaging-tier**
(color-managed float, photographic). The seam between them is where the desktop tier ends and the imaging tier begins.

### The desktop / app separation rule (testable form)

The desktop tier owns what is generic to *any* desktop application in the ecosystem; the app owns what makes it *that*
application. The rule must be precise enough that a stranger can sort any given component into the correct tier without
ambiguity. Applied to the shell: the shell *core* (event loop, surface lifecycle, threading, scheduling) and the shell
*topologies* (e.g. the Blender-style tiling workspace) are desktop-tier; *which editors fill the regions* is the app.
Applied to rendering: the generic 2D/UI rendering and the canvas surface are desktop-tier; the color-managed pixel
pipeline that produces what the canvas displays is imaging-tier.

> **Note on the name "Easel".** Earlier drafts of this architecture used "Easel" as the name for the desktop-tier
> opinion — the layer model, the dependency rules, the trait-and-registration pattern, the pump model, the wiring
> conventions. That name has been retired in favor of treating the opinion as **RSB's own position on how to build
> local-first desktop applications in Rust**, rather than as a layer on top of neutral primitives. The opinion is the
> ecosystem's, not a separate brand. RSB's desktop crates encode the opinion; The Lab embodies it as the worked example;
> the documentation articulates it. There is no "Easel" — only RSB and its position.

---

## 3. The substrate and the rendering line

The substrate is built directly on low-level dependencies. Using an existing GUI framework (egui, iced, Slint, etc.) is
explicitly **not an option**; the ecosystem owns its stack from the windowing and GPU layers up.

### The fixed foundation

- **winit** — window creation, event loop, input. Not practically replaceable; it is the Rust windowing standard.
- **wgpu** — the GPU abstraction (Metal/Vulkan/DX12). This *replaces* the prototype's `softbuffer` CPU path. The
  prototype uses softbuffer today; the real system is wgpu.
- **raw-window-handle** — the trait bridge between windowing and rendering; comes with the territory.

### Where the rendering line sits

Between raw wgpu and a finished pixel there is a stack of problems (GPU abstraction → 2D scene/vector rendering →
layout/widget model). The "rendering line" is the height at which the first dependency sits. The decision is **not one
line for the whole app**, because the application has two distinct rendering worlds:

1. **The image canvas** — the photograph being edited. This is color-managed, precision-critical, performance-critical,
   GPU-resident, and unlike anything a general 2D library optimizes for. It is written as **custom wgpu** regardless of
   every other choice. This is the most important rendering code in the system.

2. **The UI chrome** — panels, sliders, menus, histograms, the node graph. Here the genuine choice lives: own the 2D
   renderer outright, or take a 2D renderer (e.g. a wgpu-based vector/scene library) and own only the component/layout
   architecture on top of it.

The shell's threading model *confirms* the canvas-is-custom-wgpu conclusion rather than complicating it: the shell needs
fine control over producing rendered image results off-thread and presenting them on-thread, which a generic UI renderer
is not built to give.

### The shell is its own layer

The shell is neither a component nor a renderer — it is the **runtime the components live inside**. The renderer answers
"how do I turn this description into pixels"; the components answer "what do I look like and what do I do when poked";
the **shell answers "when does any of that happen, and on which thread."** It owns the event/frame loop, the sync/async
boundary, the work-draining and scheduling strategy, the window/surface lifecycle, and how components are mounted and
told to render.

Because the ecosystem has *many* shells, the shell tier splits in two:

- **Shell core** — generic runtime every shell needs (event loop, surface lifecycle, sync/async boundary,
  draining/scheduling, frame pacing). Identical regardless of the app's shape.
- **Shell topology** — the *kind* of shell. The Blender-style topology owns the tiling/docking model, region splitting,
  the active-editor concept, and workspace persistence. Both core and topologies are desktop-tier; only the population
  of regions is the app.

---

## 4. The product thesis — no boundary between raw and post

The Lab's central bet is that the boundary between "raw processing" and "post-processing" is **an accident of
application history, not a real creative boundary**. Today the boundary exists because DXO and Affinity are different
programs, forcing a lossy TIFF handoff that severs the link to the raw. The Lab eliminates the handoff: a single
continuous non-destructive edit, with different *kinds* of work (raw adjustment, local adjustment, grading, export) all
recorded in one master against one untouched raw source.

### The real boundary that does exist: develop vs. composite

While the raw/post boundary is fake, the **develop / composite** boundary is real — it corresponds to an actual change
in the nature of the data, not a change in application:

- **Develop stage** — everything is an *interpretation* of a single coherent capture. All image content is derived from
  the untouched raw by evaluating rules.
- **Composite stage** — the image stops being an interpretation of a capture and becomes an *assembly* of content from
  different origins (pasted pixels, painted artwork, text, graphics).

This boundary is **over-determined** — it is forced independently by three arguments, which is why it is bedrock rather
than an arbitrary cut:

1. **Product** — it is the genuine seam the competitors hide inside a manual file handoff; The Lab makes it visible and
   keeps it live.
2. **Re-derivability** — confining stored image content to composite is what keeps the develop graph fully re-derivable
   when the raw development changes underneath.
3. **Resolution independence** — because develop stores only rules and weights (never a fixed pixel grid), the entire
   develop stage is resolution-independent, which is what makes proxy editing (edit on a downscaled proxy, render
   full-res on export) work, which is what makes large files feel fast.

### The precise invariant

The develop-stage invariant is **not** "no stored pixels" — that would be contradicted the moment a brush paints a mask.
The correct, load-bearing statement is:

> **In the develop stage, all image *content* is derived from the raw. The only things a node may store are *rules*
> (numbers) and *weights* (masks) — never image content.**

A mask stores *where* and *how strongly* an operation applies, never *what color* a pixel is. This is what keeps the
whole develop graph re-derivable: change the white balance at the bottom of the stack and every mask still means the
same "where," every parametric op still means the same "how much," and the image recomputes from the new foundation with
nothing stale.

### The live link across the boundary

The composite stage consumes the develop stage's output as its **live base layer**. Crossing into composite does *not*
sever the link to develop (this is exactly what Affinity cannot do — leaving its Develop persona freezes the image into
a pixel layer). In The Lab, the develop graph stays alive underneath the composite, and the composite re-renders over it
whenever develop changes. **This live link is the single most important thing the UI must make feel effortless** — "go
back, change the exposure underneath, watch the composite update on top without redoing anything" is the entire payoff
of refusing the handoff.

---

## 5. The editing engine — the node graph is the truth

### Why a graph is forced

The duality is *structurally* a graph whether or not it is called one. "Non-destructive operations, reorderable, against
an untouched source, with masks that modulate where things apply, the output of develop feeding the base of composite"
is a description of a directed graph of image operations. The only real question was ever "explicit graph the user can
manipulate, or implicit graph hidden behind fixed sliders." Lightroom has a graph too — frozen into a fixed order and
hidden — which is *why* it cannot express arbitrary local work in arbitrary order. The Lab's duality requires a
*flexible* graph, and a flexible graph is hard to hide. **The graph is the document; everything the user touches is a
lens onto it.**

### The brush is a mask producer

The brush engine, positioned as first-class and tablet-friendly, is fundamentally a **mask producer**. A brush stroke
does not (by default) set pixels to a color; it produces a per-pixel *weight* describing where and how strongly an
operation applies. This single capability feeds dodge/burn, local adjustments, and the clone/heal operations alike.
Separately, for genuine foreign content, the brush can also act as a **pixel producer** writing into a raster node. One
brush, two roles; the role is determined by which kind of node it feeds, never by a global "destructive mode" switch.

### Two meanings of "non-destructive" (the cloning insight)

Cloning exposes that "non-destructive" names two different things:

- **Layer non-destruction** (Photoshop/Affinity lineage) — paint the clone onto a separate layer; the *original* is
  protected but the *cloned pixels are baked* (sampled once, frozen, do not follow later changes to the development
  underneath). "Non-destructive" here really means "additive."
- **Parametric/graph non-destruction** (Lightroom/darktable/Lab lineage) — store the *instruction*, not the pixels. A
  clone is "within this mask, sample from this source offset and composite," computed at evaluation time. If the
  development underneath changes, the cloned region **re-samples and updates**. Nothing is baked.

The Lab is therefore *more* non-destructive at cloning than Affinity, not merely as much — and this is forced by the
thesis, not chosen. Healing is the same shape with a heavier evaluation function (gradient-domain / Poisson solve over
the mask): a mask, a source reference, an evaluation function. **Clone and heal are reference nodes and live in the
develop stage** (they reference live upstream content, store only mask + offset, and are resolution-independent).

### What genuinely needs stored pixels

Only content with **no source anywhere upstream** — pasted foreign photographs, hand-painted original artwork,
generative fill. These must be stored as actual pixel data because nothing can regenerate them. This is *not* a
destructive mode; it is a **raster node** within the same non-destructive graph (still reorderable, maskable,
toggleable). Plus an explicit, opt-in, one-way **bake/flatten** command for when the user deliberately chooses to freeze
a region (for performance or finality) — never silent.

### The node taxonomy (derived, not guessed)

| Node flavor       | Carries                       | Stage     | Example                                             |
| ----------------- | ----------------------------- | --------- | --------------------------------------------------- |
| Global parametric | numbers (rules)               | develop   | exposure, white balance                             |
| Masked parametric | mask (weight) + parametric op | develop   | local exposure via brushed mask                     |
| Reference         | mask + source transform       | develop   | clone, heal                                         |
| Raster            | stored pixels (content)       | composite | pasted foreign image, painted artwork, baked result |

Every flavor composes in the same graph, evaluates against the live raw, and tiles onto wgpu compute the same way (each
node is a compute dispatch; the graph is a dependency DAG evaluated tile-wise so large files need not fit in VRAM at
once).

### Distinct socket types per stage (the boundary made structural)

Develop-stage nodes and composite-stage nodes have **different socket types**, and this is the strongest available
expression of the develop/composite boundary. Because develop data and composite data are *genuinely different kinds of
thing* — a resolution-independent, re-derivable interpretation of one capture versus a resolved, possibly-assembled
raster — they should have different types, and the socket types are simply the type system telling the truth about that
difference. A develop node's output cannot connect to a composite node's input, not because a runtime check fires, but
because the connection is **unrepresentable**. The wrong wiring has no expressible form.

This upgrades the honest-seam principle from a *runtime* behavior the system must remember to perform (warn on a broken
transfer, refuse an illegal bake) into a *structural* property of what is even constructible. The stage-compatibility
dimension of the node taxonomy above is therefore enforced automatically: a raster node cannot be dropped into a develop
graph because its socket type is a composite type, not because of a special rule about raster nodes. **Prefer making the
wrong thing unrepresentable over making it merely detectable.**

The type boundary is one-directional: data flows **develop → composite** through exactly one conversion point (where the
develop graph's output is resolved into the composite stage's live base layer). There is no composite → develop path,
and the type system forbids it absolutely. This matches reality rather than fighting it — raw-developing pixels that
never came from a raw is incoherent — so the rigidity forbids only genuinely incoherent things, which is the good kind
of rigidity. *(This is a hard commitment: once socket types enforce one-way flow, reversing the decision is expensive.)*

### The transfer unit: node / sub-graph (not document)

In the single-document model (§8), the unit that moves between windows and instances — via copy/paste or drag — is **not
a document but a node or sub-graph** (a "layer," in photographer-facing language). This rides on the **same node-group
boundary machinery** that powers panel projection (§7): a group's interface is the "plug" that lets a sub-graph detach
from its source graph, survive a process boundary, and graft into a destination. One structure, two uses (projection and
transfer), so cross-instance layer transfer comes nearly for free once node groups exist.

Portability follows from node flavor, and most of it is now handled by the type system:

- **Stage compatibility** is enforced *structurally* by distinct socket types — a develop node cannot graft into a
  composite slot or vice versa, with no runtime check needed.
- **Global parametric** nodes carry only numbers and graft anywhere their socket type fits.
- **Masked parametric** nodes carry self-contained weight data and travel cleanly.
- **Reference** nodes (clone/heal) are the **one residual runtime-honesty case**: two nodes can share a socket type yet
  the destination may lack the *source content* the reference points at. When a dragged reference node's origin cannot
  be satisfied in the destination, the system must say so plainly (honest-seam principle) rather than silently producing
  a broken or misinterpreted result.
- **Raster** nodes carry their pixels with them but are composite-type, so the type system already confines them to the
  composite stage.

---

## 6. Multi-source operations — deliberately deferred

HDR merge, panorama stitching, and focus stacking are **out of scope until the core single-source editor is proven.**
They are edge cases relative to the primary problem (prove we can edit photos first). Designing them now would be
designing in a vacuum — their internal structure is the defining question of the *focused apps* that will handle them,
discovered when those apps are built.

What must be true *today* is small and durable: it must be **possible to go from multi-source to single-source**, and
the **contract** must specify what happens to the data at that transition — what crosses, what is left behind, and how
the user is shown the operation is happening. Not *how* the collapse is computed (that belongs to the future app's
interior), only *that* it is an expressible, well-defined, never-implicit transition. We design the **socket, not the
appliance**: a stable seam any future multi-source operation conforms to. The single-source baseline is the lingua
franca of the ecosystem; multi-source documents terminate into it through an explicit, user-understood transition.

> Note: "never flatten" was refined to **"never *implicitly* flatten."** Flattening is legitimate; the user always
> retains the option to export and transform. What matters is that the user understands the operation they are
> performing. This is the honest-seam principle again.

---

## 7. The dual-surface interface

Interface is a **view**, not the model. The graph is the truth; there are two surfaces onto it, each honest and complete
for its audience, neither the "real" one.

### Node view (technical artist)

A real, visible, navigable graph editor (Blender-style). Technical artists wire nodes, group them, and build things the
panel vocabulary has no word for. First-class, not a debug view. Fully adopted, visually and otherwise.

### Panel view (photographer)

A constrained, familiar, Affinity/Lightroom-style surface — canvas, toolbars, adjustment panels (and a layer-stack-like
presentation). The photographer never has to see a node. The panels are **not a separate system**: they are a
*projection* of the graph. The slider and the node parameter are one value with two faces; they cannot drift because
there is nothing to drift. A layer stack is simply a graph with a constrained, mostly-linear topology that humans find
legible — so "traditional panels" and "the node graph" are not alternatives; the panel view is *what the graph looks
like when projected for a photographer.*

### How Blender actually does this (verified)

Blender's mechanism, confirmed against current documentation and developer notes, is the **node group**: a cluster of
nodes bundled into a named unit with an authored *interface* (the exposed, organized inputs). The **group is the binding
unit**, and the group's interface is what projects into a UI. Exposed inputs organize into **nested panels with
headers**, and a geometry node group used as a modifier surfaces its exposed inputs as an actual **panel of editable
controls** in the Properties region. This is production-proven and is exactly the "technical artists author guided
experiences that others consume as panels" model.

Blender's layout is **author-controlled** (the author groups, nests, names, and orders), but with **system-imposed
structural rules** (e.g. panels always render after sockets, at the bottom). Blender's own developers documented that
the friction lives precisely at the *seam between author freedom and system constraint* — enforcing the consistency
rules during authoring is "annoying" and worsens as more rules are added. Blender also needs a scripting **escape
hatch** (custom Python `draw()` panels) for cases its built-in exposure can't handle.

### Where The Lab diverges from Blender — and why it can

Blender's nodes are **general-purpose**, so it *cannot* predict what a good panel looks like; it must hand authors broad
layout control, which is why adapter nodes proliferate (e.g. inserting a Combine-XYZ node purely to expose a clean
"Width" slider) and why the author/system seam hurts.

The Lab's domain is **narrow**: photographic adjustment is a large but *finite, well-understood* vocabulary. That
narrowness is an asset Blender lacks. It lets The Lab:

- Ship a **curated set of high-level, photographically-meaningful nodes** whose parameters already map cleanly onto good
  panels — so in the common case **no adapter nodes and no interface hand-shaping are needed** (the exposure node *is*
  the exposure panel, because both ends were designed together).
- Choose **system-controlled panel layout** rather than author-controlled. The author chooses only *which* parameters to
  expose; the system decides arrangement. This buys guaranteed cross-document consistency and **designs out the exact
  author/system seam where Blender's friction lives.** The author's freedom moves entirely to the node-graph side.

### The escape hatch and graceful degradation

Adopt Blender's *structure* (groups as binding unit, authored interface, nested panels) but reject its *primitivism*
(curated photographic nodes, system layout). For the rare frontier case — a novel node outside the curated vocabulary,
with no panel form — degrade **honestly and gracefully**:

- Every node has a **common envelope** (enable, opacity, mask, order in stack) that the panel can *always* render,
  because those properties exist on every node regardless of its interior. The photographer can reorder, mask, toggle,
  and dial the strength of any node as a respected block in their stack.
- Only the node's **interior parameters**, when beyond the panel vocabulary, are covered by a **placeholder** that
  *educates* the user and offers escalating routes to the full feature set: split the window (Blender-style), open a new
  window, or switch to a node-capable workspace.

This keeps the placeholder a **safety net for the frontier**, not a substitute for doing the projection work. Designing
each node *together with* its panel face keeps the no-panel case genuinely rare.

---

## 8. Workspaces and windows

The two surfaces are not two codebases — they are two **workspace configurations** of the one Area/Surface/Region shell
(directly analogous to Blender's Layout / Shading / Compositing workspaces). The node-graph editor is a Surface type;
the canvas is a Surface type; the histogram and adjustment panels are Regions. A "workspace" is a recipe of which
Surfaces and Regions are present and where. The photographer's workspace and the technical artist's workspace draw from
one shared pantry of components.

The technical-artist workspace has a larger **spatial appetite** (node graph + canvas + histogram), so the shell must
support **one document spanning multiple windows** (canvas on one monitor, node graph on another — both live views of
the same graph). This is distinct from the multi-*document* question below: it is multiple *windows onto one document*.

### Single-document model (settled — moving away from tabs)

The editor uses a **single-document** model (Blender-style) rather than tabbed multi-document. This is **settled**, and
it is over-determined — forced independently by two unrelated arguments, which is the signal that it is a real decision
rather than a preference:

1. **UI-structural (forcing).** A node-graph editor is spatial and wants to *consume the window* — it spreads across the
   screen, often a whole monitor, alongside the canvas and histogram. Tabs are designed to multiplex many documents
   through one frame and swap the whole document on switch. The two fight over the same screen real estate and the same
   mental model: tabs want to hide and swap what the node graph wants to *be*. A node-first editor therefore cannot
   comfortably have tabs for the graph. (This is the same reason Blender is single-document — its editors are spatial
   and want the window.)
2. **Resource/residency.** One document means one graph, one pipeline, one resident working set — no eviction strategy,
   no "which tab is hot," because the multi-pipeline GPU-memory problem (§9) never arises.

The UI argument says single-document is *necessary*; the residency argument says it is *cheaper*. Same destination, two
roads.

Working on two images together is served by **multiple instances / windows** (leaning on OS window management). The
transfer unit between instances is the **node or sub-graph** (§5) — copy/paste or drag a "layer" between windows —
riding on the node-group boundary machinery.

The one honest residual: this covers "move this edit to that image" well, but the **batch / sync-across-a-shoot** case
(apply one edit to fifty frames at once) is the workflow single-document genuinely strains. Layer drag-and-drop does not
fully replace it. This is the thing to watch as real editing is built (it first bites around intermediate app #3), but
it does not block the foundation.

---

## 9. Multi-image residency — what the competitors taught us

Research into how the established editors handle multiple open images yielded a clear convergence and a direct
implication for The Lab.

- The three raw-centric tools (**Lightroom Classic, Lightroom CC, Capture One**) and **DXO** all use a **single active
  editing pipeline** re-bound to whichever image is selected via a filmstrip/browser/catalog — *not* tabs. Switching
  images re-renders from the raw; only cached previews/proxies and small edit-state are kept for the others.
- Only **Affinity Photo** uses genuine document tabs — because it is a pixel/composite editor, not a raw flow — and even
  it keeps all documents resident and leans on OS paging rather than evicting GPU resources.
- The editing pipeline runs on the **GPU** essentially everywhere (DX12/Metal for Lightroom; OpenCL/Metal for Capture
  One; Metal/OpenCL-on-D3D12 for Affinity; DXO is CPU-dominant except for its AI stages). All work in **linear float**
  internally.
- Perceived speed comes from the **proxy strategy**: paint a cached low-cost preview instantly, then replace it with the
  full pipeline output once the GPU catches up. None keep multiple full pipelines pre-warmed.

**Implication for The Lab.** Because a Lab edit is a small `.rsb` instruction graph evaluated against an untouched raw,
a Lab tab/document is architecturally a Lightroom-style *single pipeline* but with Affinity-grade editing *depth*. The
state an inactive document must retain is the **edit graph plus a small preview texture — kilobytes, not gigabytes** of
intermediate float rasters. This makes inactive documents nearly free and is a genuine advantage over Affinity's
resident-raster model. The single-document direction (§8) takes this further by avoiding multi-residency entirely.

### Pixel pipeline precision (the foundation)

The prototype currently collapses LibRaw's 16-bit linear output to an **8-bit sRGB** `Pixel` — a viewport shortcut, not
an editing foundation. Exposure recovery, highlight reconstruction, and wide-gamut work all need the headroom that is
discarded at decode time, and once discarded it cannot be recovered. Moving to a **high-precision working pixel**
(16-bit float minimum; 32-bit float where stacked operations accumulate rounding) with proper **color management**
(camera input profiles, a wide-gamut working space, output profiles, soft-proofing) is the **precondition** for the
editing engine, not one feature among many. Recommended working formats on wgpu: `Rgba16Float` for intermediates,
`Rgba32Float` where precision genuinely matters; tile every operation by workgroup; convert to display space at exactly
one point (the last step). This precision decision and the canvas→image boundary decision are the **same decision** —
the canvas takes a presentation-ready buffer; the imaging layer owns the float pipeline that produces it.

---

## 10. The intermediate-app strategy

Each intermediate app is a **shippable artifact that forces one slice of the architecture into existence and hardens it
in isolation**, before The Lab needs everything working together. This prevents "framework astronomy" — building the
desktop and imaging tiers in the abstract and discovering at integration time that the abstractions were wrong. Every
intermediate app is a falsification test for a slice of the system, *and* delivers standalone value.

### The build-order principle, applied

From the generality-first principle: **build the technical-artist node surface before the photographer's panel
surface.** The panel is a projection of the graph; it cannot be built before the nodes it reflects exist. Building the
node surface first means every later panel is a reduction of a *proven* system, and it is what concretely *reveals* the
finite photographic vocabulary that makes system-controlled panel layout affordable. The two decisions support each
other: technical-first sequencing makes system-controlled panels cheap.

### Derived ladder of intermediate apps

The develop/composite boundary that is the product seam, the technical/crate boundary, *and* the natural seam in the app
ladder all coincide — strong evidence it is a real joint. The raw-only app is the develop stage in isolation; the
post-only app is the composite stage in isolation; The Lab is the two integrated with the live link, so The Lab becomes
an *integration* rather than an *invention*.

**1. Raw image viewer** — Imaging (foundational)

- **Forces into existence:** Substrate (winit + wgpu), the custom-wgpu canvas, raw decode port, **display-end color
  management**, shell core, progressive decode pipeline
- **Standalone value:** A fast, correct, local raw viewer

**2. Device-config app** (tablet + TourBox, Linux) — **Desktop-tier generality (parallel)**

- **Forces into existence:** Proves the desktop tier can build a **non-imaging** app: UI components/widget toolkit,
  config persistence, settings-style workspace — touching *none* of the imaging tier
- **Standalone value:** A genuinely underserved Linux tool the author needs

**3. Minimal raw developer** — Imaging

- **Forces into existence:** The **high-precision float pipeline**, the **node engine** with a tiny real vocabulary
  (exposure, WB, a curve), and the **first panel projections** of those nodes
- **Standalone value:** A simple, real raw developer

**4. Node-graph editor app** — Imaging

- **Forces into existence:** The **node-graph Surface** (technical-artist workspace), node grouping, the
  authored-interface → panel projection contract, multi-window-single-document
- **Standalone value:** A node-based image tool

**5. Raw-only (develop) app** — Imaging

- **Forces into existence:** Hardens the **entire develop stage** in isolation — single root, purely parametric,
  resolution-independent; the mask system; reference nodes (clone/heal)
- **Standalone value:** A focused, fast raw developer

**6. Post-only (composite) app** — Imaging

- **Forces into existence:** Hardens the **composite stage** — raster nodes, foreign content, the
  brush-as-pixel-producer role, layer compositing over a (here static) base
- **Standalone value:** A focused compositor/finisher

**7. The Lab** — Target

- **Forces into existence:** The **live link across the develop/composite seam** — the one genuinely new thing left to
  invent once 5 and 6 are proven
- **Standalone value:** The full editor

> The ordering of 3–6 is a sketch, not a contract; the point is the dependency direction (general before constrained;
> develop and composite proven separately before integration). Multi-source apps (panorama, HDR, focus-stack) come
> *after* The Lab is proven and conform to the §6 transition contract.

### The desktop-tier generality canary

App #2 (device config) plays a structural role beyond its personal value: it is the **one app that uses the desktop tier
while touching no imaging primitives**, which keeps the desktop/imaging boundary honest. Everything else in the ladder
is imaging; without a non-imaging app, the desktop tier could quietly absorb photo-specific concerns and no one would
notice until a second, different kind of app tried to use it. The device-config app is the canary that keeps the desktop
tier actually domain-clean. It can be built early precisely because it has no dependency on the imaging tier.

---

## 11. Repositories, workspace, and publishing

### Two repositories

| Repository          | Contents                                                                                              | License                                                                                                                                     |
| ------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `rsb` (ecosystem)   | Desktop-tier crates, imaging-tier crates, foundational primitives, all intermediate apps as exemplars | **MIT OR Apache-2.0** (standard Rust dual-license; both `LICENSE-MIT` and `LICENSE-APACHE` at the repo root)                                |
| `rsb-lab` (product) | The Lab; consumes ecosystem crates as published artifacts                                             | **Undecided** — deliberately deferred. May be open-source, source-available, or commercial; the decision is held open as a strategic option |

The ecosystem follows the Rust ecosystem convention so that any Rust consumer — including commercial closed-source
projects and people building their own desktop frameworks on the same primitives — can adopt it freely. The Lab is a
downstream consumer of the ecosystem regardless of which license is eventually chosen for it.

### Web properties

The websites (`rsb.sh`, `rsb.ink`) use a **triple license**: `LICENSE-MIT`, `LICENSE-APACHE`, and `LICENSE-CC` (Creative
Commons for content). Code on the sites is dual-licensed under the same terms as the ecosystem; content is governed by
the CC license.

### Contributor License Agreement (CLA) — with relicensing clause

External contributions to ecosystem repos are accepted under a **CLA modeled on the Apache Software Foundation's
individual CLA**. Critically, the CLA includes a **relicensing clause**: contributors grant the project the right to
release their contributions under different licenses in the future. This is a *deliberate strategic decision*, not
deferral: it keeps the door open for The Lab (or any future component) to ship under a different license without being
foreclosed by early contributor grants to the framework crates. Without this clause, every external contributor would
hold copyright on their patches under the original terms only, making any future relicensing effectively impossible.

### Ecosystem workspace and publishing

The `rsb` repository is a **single Cargo workspace** containing every ecosystem crate — primitives, desktop tier,
imaging tier, and intermediate apps — in one place. One clone, one `Cargo.lock`, one `target/`, atomic cross-crate
refactors, shared lints and CI. This is the contributor experience.

Library crates are **published individually to crates.io** with their own versions when ready for outside users.
Intermediate apps and internal helpers are unpublished (`publish = false`). The "monorepo vs. polyrepo" question
dissolves by separating *where code lives* (one repo) from *how code ships* (per-crate). External consumers see focused
libraries on crates.io regardless of source layout.

### Workspace directory layout

The workspace root has **three top-level directories**, each named by the *purpose* of the crates it contains — not by
their mechanical shape:

```text
rsb/
├── Cargo.toml          (workspace manifest: members = ["apps/*", "libs/*", "examples/*"])
├── LICENSE-MIT
├── LICENSE-APACHE
├── apps/               crates whose purpose is to be a runnable artifact
├── libs/               crates whose purpose is to be depended on as a library
└── examples/           crates whose purpose is to demonstrate behavior
```

`apps/`, `libs/`, and `examples/` sit as **peers**, because they describe peer purposes, not variations on a theme.
Naming by *purpose* rather than by *artifact shape* is honest: an example can technically be a library or a binary, but
what defines it is its demonstrative intent — and that's the property that determines everything downstream (publishing,
lifecycle, allowed dependencies, what it signals to contributors).

**Discipline rules** (mechanically enforceable in CI via an `xtask` walker):

- Crates in `libs/` have only `src/lib.rs` and no `[[bin]]` targets.
- Crates in `apps/` produce a binary (may have `src/lib.rs` for internal organization, but the primary artifact is the
  binary).
- Crates in `examples/` may be either lib or bin in shape — purpose, not shape, determines membership.
- **Nothing in `libs/` or `apps/` may depend on anything in `examples/`.** This is what keeps examples genuinely
  disposable. When an example becomes load-bearing it has *graduated* and should move into `apps/` or `libs/` as
  appropriate. Graduation is the natural and healthy direction; silent load-bearing while still living in `examples/` is
  the failure mode this rule prevents.

Applying *unrepresentable beats detectable* to file layout: enforce the rules mechanically rather than by memory.

**Two flavors of examples coexist by scope:**

- Small, single-library demonstrations live as **per-crate `examples/` subdirectories** inside the library they
  exemplify (Cargo's native pattern, auto-discovered).
- Cross-cutting examples that span multiple crates, or are large enough to warrant their own dependencies and identity,
  live as **workspace members under the top-level `examples/`** directory.

The two are separated by scope, not by importance — a per-crate example is tightly bound to one library; a
workspace-level example demonstrates an integration pattern across several.

**The `libs/` interior is organized by domain directories, not by crate hierarchy.** Inside `libs/`, top-level
subdirectories group crates by *domain* — the kind of program the crates inside help build:

```text
libs/
├── camera/                  imaging-domain crates
│   ├── raw-decode/
│   ├── raw-demosaic/
│   └── ...
├── desktop/                 desktop-program crates — building blocks any desktop opinion can compose
│   ├── base/
│   │   ├── fail/
│   │   └── report/
│   ├── runtime/
│   │   ├── canvas/
│   │   ├── shell/
│   │   └── workers/
│   ├── substrate/
│   │   ├── substrate-winit/
│   │   └── ...
│   └── ...
└── base/                    (optional, currently empty) — reserved for truly runtime-model-free utilities if any earn it
```

`desktop/` is named by *domain*, not by opinion. It describes what kind of programs the crates inside help build
(desktop ones), not whose particular opinion about how to build them. The crates encode RSB's opinion; the directory
just names the domain they serve. The same naming discipline applies to `camera/`: a kind-of-thing, not a brand.

**Crate granularity is serde-style — many small, focused crates, optimized for consumer composition.** Each leaf
directory under `libs/<domain>/<layer>/` is a separate crate with its own `Cargo.toml`, its own version, its own
publication. The trade — more `Cargo.toml`s and more version streams in exchange for fine-grained downstream composition
and modular design discipline — is accepted deliberately.

**Crate names carry the domain qualifier.** Crates are named `rsb-<domain>-<crate>` when their abstractions are
domain-shaped (e.g., `rsb-desktop-fail`, `rsb-desktop-report`, `rsb-desktop-shell`, `rsb-camera-raw-decode`). The bare
`rsb-<crate>` form is reserved for truly runtime-model-free crates at top-level `base/` if any earn that placement. The
directory path describes *where the crate lives* (which is for contributors reading the source tree); the crate name
describes *what the crate is shaped for* (which is for downstream consumers who only see the published name). Both
happen to encode the same domain word when applicable — that's honest consistency, not redundancy.

The layer position (`base`, `runtime`, `substrate`, …) is *not* part of the crate name. It is a contributor-facing fact
about how the desktop crates compose internally, not a consumer-facing fitness claim. Layer enters a name only when
needed to disambiguate sibling crates that share a domain and a concept but live in different layers — a forced
disambiguation, not a default.

**Directory hierarchy and crate granularity are decoupled.** Directory depth follows architectural meaning (it teaches
the reader where things belong); crate granularity follows consumer reuse (it serves the downstream composition story).
Cargo doesn't care how deeply nested crates are — the workspace manifest uses a recursive glob (`members =
["libs/**/*"]` with appropriate filtering, or explicit listing) to pick them all up. This decoupling means the directory
tree can be as expressive as it needs to be for human comprehension without imposing any Cargo cost.

**Domain placement follows abstraction shape, not concept category.** Two crates that look like "the same thing" by
concept (failure handling, logging, persistence) may have abstractions tuned to different runtime models, and merging
them into one generic crate is the false economy. Example: `desktop/base/fail/` (published as `rsb-desktop-fail`) is the
*desktop's* failure system, shaped by interactive-shell concerns (frame-error tolerance, the relationship between errors
and the report stream, failure categories that only matter when a UI is responding). A server framework would want a
differently-shaped failure system — same concept, different abstraction — and would live in a different domain (e.g.,
`server/base/fail/` published as `rsb-server-fail`, if RSB ever grows that side). The concept is shared; the abstraction
is domain-tuned. The top-level `libs/base/` (when populated) is reserved only for utilities whose abstractions presume
*no* runtime model at all — pure data, math primitives, generic algorithms, string handling. Most "foundational" things
turn out, on inspection, to carry runtime-model assumptions and belong in a domain directory rather than at top-level
base.

**The opinion is RSB's, not a separate brand.** RSB encodes a specific position on how to build local-first desktop
applications in Rust — the layer model, the dependency rules, the trait-and-registration pattern, the pump model, the
wiring conventions. That position is *not* a layer on top of neutral primitives; it lives in the crates themselves (the
recovery semantics in `rsb-desktop-fail`'s `Kind` enum, the closed `Domain` enum in `rsb-desktop-report`, the shell
concepts in `rsb-desktop-runtime`, etc.) and is *embodied* by Lab as the worked example any future developer can point
at. The articulated description lives in documentation (the layer-model specification, the conventions guide). There is
no `rsb-desktop` umbrella crate, no separate framework name, and no neutral "primitives that any opinion could use"
layer underneath — the opinion is integral to what RSB *is*.

### Versioning

Effort scales with *number of published crates*, not workspace size. Unpublished crates have no version semantics.
Published crates follow standard per-crate semver, coordinated by tooling (`cargo-release`, `release-plz`, or `cargo
workspaces`). Adopt one from the first publication rather than retrofit.

### Blast radius is contained by the tiering

The dependency tiering (§2) doubles as blast-radius containment. Nothing in a lower tier depends on anything above, so a
change's blast radius has a *ceiling* set by the changed crate's tier — lowest at the apps, highest at the foundational
primitives (which change rarely). Monorepo CI catches cross-crate breakage synchronously in the same PR; polyrepo would
discover it asynchronously weeks later. The effective blast radius is *smaller* in the monorepo on the dimension that
matters most.

### Cross-repo development workflow (Lab ↔ ecosystem)

When The Lab needs an ecosystem change, the normal flow is: change the ecosystem crate, publish a new version, bump The
Lab's dependency. For active iteration, `[patch.crates-io]` in `rsb-lab/Cargo.toml` temporarily resolves a
published-crate dependency to a local path — letting the two-repo arrangement feel like one workspace during development
while remaining clean publicly. Documented Cargo feature, designed for exactly this case.

### Long-term failure mode (acknowledged, not addressed)

Monorepos can rot at the scale of dozens of contributors and hundreds of thousands of lines. Does not apply to current
or near-term RSB. The right move *if* it ever begins to arrive is to split deliberately at that point, not to have built
differently from the start.

---

## 12. Open decisions (explicitly deferred)

These are known-open and intentionally not yet resolved:

1. **Rendering line for UI chrome** — own the 2D renderer outright, or adopt a wgpu-based 2D/vector renderer and own
   only the component/layout layer. (Canvas is custom wgpu regardless.) Depends partly on how much "own the whole stack"
   is a goal vs. a cost, and tolerance for a young dependency in the foundation.
2. **Multi-source transition contract** — whether crossing the multi→single seam guarantees *round-trip provenance* (a
   door back to the live sources) or only a *clean forward step* (a faithful baseline with no obligation to remember its
   making). Deferred with the multi-source apps (§6).
3. **`.rsbp` vs. per-app formats** — single unified document format vs. per-app formats joined by a non-flattening
   import contract. Both serialize the *same* document model; this is a packaging decision derived *after* the document
   model is firm, and can be deferred.
4. **Crate granularity** — e.g. shell core and Blender-style topology as two crates or one crate with a topology module;
   feeds the roadmap's crate graph.
5. **Batch / sync-across-a-shoot** — *not* an architecture fork but a watch-item: the one workflow the (now settled)
   single-document model genuinely strains (§8). Layer drag-and-drop between instances does not fully cover it. First
   bites around intermediate app #3; revisit then rather than now.
6. **LibRaw binding placement / licensing (kept as a separate repo, not imported)** — the LibRaw bindings
   (`rsb-camera-libraw`, `rsb-camera-libraw-ffi`) are deliberately kept in a **separate repository** rather than
   imported into this monorepo. The vendored LibRaw 0.22.1 is dual-licensed **LGPL-2.1 / CDDL-1.0** (and bundles
   further BSD-3-Clause / MIT third-party code) and is *statically linked* by the ffi crate, so pulling it into the
   `MIT OR Apache-2.0` workspace would make the entire tree inherit those copyleft obligations. Deferred: whether to
   consume it as a feature-gated optional path/git dependency, or to expose LibRaw only through the future
   `rsb-camera-raw-decode` port while the binding crate stays external. Until decided, both crates remain
   `publish = false` and live outside the monorepo.

> **Closed since first draft:** the single-document vs. tabs question is **settled** (§8) — single-document,
> over-determined by the node-graph UI's spatial appetite and by residency economics.

---

## 13. Principles index (for the articles)

The recurring ideas, collected for reuse in the public writing:

- **Interface is a view, not the model.** The graph is the truth; panels and node-editor are projections.
- **Generality first.** Build the general system, derive the constrained surface from it.
- **Honest seams.** No implicit flattening, no silent baking, no pretending a surface can edit what it cannot.
  Destruction is a named, chosen act.
- **Unrepresentable beats detectable.** The strongest form of an honest seam is structural: prefer making the wrong
  thing *impossible to express* (distinct develop/composite socket types, so an illegal wire has no form) over making it
  *detectable at runtime* (a check that fires, a warning shown). Push honesty up into the type system wherever it can
  reach. Reserve runtime honesty for the genuinely irreducible cases (e.g. a transferred reference node whose origin
  content is absent).
- **Find the more general structure.** The single-root document is the n=1 case of multi-root; the layer stack is a
  constrained-topology graph; "purely parametric" is really "rules and weights, never content." Repeatedly, the right
  move was not a special case but the general structure of which the thing at hand is a special case.
- **Over-determination signals bedrock.** When one joint is justified independently on product, correctness, and
  performance grounds (the develop/composite boundary), it is real.
- **Narrow domain is an asset.** It is precisely what lets The Lab choose curated nodes and system-controlled layout
  where Blender's generality cannot.
- **Make the hard problem not arise.** The most elegant wins (single-document residency) avoid the hard problem rather
  than solving it well.
- **Domain placement follows abstraction shape, not concept category.** Two crates that look like "the same thing" by
  concept (failure handling, logging, persistence) may have abstractions tuned to different runtime models. Merging them
  into one generic crate is the false economy. Let concept-cousins live in their own domain directories; reserve the
  top-level domain-free space for abstractions that presume no runtime model at all.
- **Decouple architectural communication from build artifacts.** Directory hierarchy and crate granularity serve
  different audiences with different mechanisms — directory depth teaches the reader the architecture; crate granularity
  serves downstream composition. The two don't have to use the same grouping. Use directories as freely as architectural
  meaning demands; let crates be the unit consumers actually depend on.
