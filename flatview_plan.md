# Flatview — Detailed Technical Project Plan

## Architecture direction

The implementation should be organized around four deliberately separate layers:

```text
┌─────────────────────────────────────────────────────────────┐
│ GTK4 / Libadwaita UI                                        │
│  Window · HeaderBar · GraphCanvas · Suggestions · Dialogs   │
└───────────────────────┬─────────────────────────────────────┘
                        │ view models / commands
┌───────────────────────▼─────────────────────────────────────┐
│ Application / domain layer                                  │
│  GraphModel · LayoutModel · SuggestionEngine · AppActions   │
└───────────────────────┬─────────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────────┐
│ Flatpak integration                                         │
│  CliCollector · MetadataReader · HostCommandRunner          │
│  IconResolver · InstallationScope                           │
└─────────────────────────────────────────────────────────────┘
```

The central architectural rule should be:

> **GTK knows about presentation; the domain layer does not know about GTK; Flatpak collection does not know about either.**

That gives Flatview a large testable core. The parser, graph construction, sizing functions, dependency analysis, and cleanup heuristics can all be tested without starting GTK.

There is one especially important constraint to settle up front: Flatview itself will be sandboxed as a Flatpak. A normal `Command::new("flatpak")` is therefore not sufficient for the shipped application. `flatpak-spawn --host` can execute host commands, but requires access to the `org.freedesktop.Flatpak` D-Bus interface. Flatpak also explicitly recommends restricting filesystem access and using portals where possible. [Flatpak](https://docs.flatpak.org/es/latest/flatpak-command-reference.html?utm_source=chatgpt.com)

I therefore recommend a **host-integration abstraction** rather than scattering `Command` invocations throughout the app.

---

# Phase 0 — Project decisions and scaffolding

## Goal

Create the repository, establish the GNOME application identity, toolchain, coding conventions, and build system before implementing functionality.

### 0.1 Decide the application ID

Recommended:

```text
io.github.<your-account>.Flatview
```

or, when the project has a permanent domain:

```text
com.example.Flatview
```

For a future Flathub submission, use a reverse-DNS ID that corresponds to a controlled project identity. Flatpak uses the application ID in the manifest, desktop file, metainfo, D-Bus identity, icon, and other locations. [Flatpak](https://docs.flatpak.org/en/latest/conventions.html?utm_source=chatgpt.com)

### 0.2 Proposed repository structure

```text
flatview/
├── Cargo.toml
├── Cargo.lock
├── meson.build
├── meson_options.txt
├── README.md
├── LICENSE
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── .gitignore
├── .editorconfig
├── rustfmt.toml
├── clippy.toml
│
├── data/
│   ├── io.github.example.Flatview.desktop
│   ├── io.github.example.Flatview.metainfo.xml
│   ├── io.github.example.Flatview.gresource.xml
│   ├── icons/
│   │   └── hicolor/
│   │       ├── scalable/apps/
│   │       └── symbolic/apps/
│   └── ui/
│       ├── window.blp
│       ├── headerbar.blp
│       ├── suggestions-panel.blp
│       ├── app-context-menu.blp
│       ├── uninstall-dialog.blp
│       └── ...
│
├── src/
│   ├── main.rs
│   ├── application.rs
│   ├── window.rs
│   │
│   ├── model/
│   │   ├── mod.rs
│   │   ├── flatpak_item.rs
│   │   ├── app.rs
│   │   ├── dependency.rs
│   │   ├── graph.rs
│   │   └── installation.rs
│   │
│   ├── flatpak/
│   │   ├── mod.rs
│   │   ├── collector.rs
│   │   ├── cli.rs
│   │   ├── metadata.rs
│   │   ├── parser.rs
│   │   ├── host.rs
│   │   └── icons.rs
│   │
│   ├── graph/
│   │   ├── mod.rs
│   │   ├── layout.rs
│   │   ├── physics.rs
│   │   ├── spatial.rs
│   │   ├── sizing.rs
│   │   ├── hit_test.rs
│   │   └── selection.rs
│   │
│   ├── suggestions/
│   │   ├── mod.rs
│   │   └── scoring.rs
│   │
│   ├── actions/
│   │   ├── mod.rs
│   │   ├── applications.rs
│   │   └── uninstall.rs
│   │
│   └── ui/
│       ├── mod.rs
│       ├── graph_canvas.rs
│       ├── graph_view.rs
│       ├── suggestions.rs
│       ├── app_popover.rs
│       └── dialogs.rs
│
├── tests/
│   ├── parser.rs
│   ├── collector.rs
│   ├── graph.rs
│   ├── sizing.rs
│   └── suggestions.rs
│
├── packaging/
│   └── io.github.example.Flatview.yml
│
└── ci/
    └── ...
```

The names above are deliberately somewhat granular. The project will have enough host-integration and rendering complexity that one giant `main.rs` will become difficult to maintain very quickly.

---

## 0.3 Toolchain detection

Because development is on CachyOS, **do not blindly install anything**.

First check:

```bash
command -v rustc
command -v cargo
command -v meson
command -v ninja
command -v blueprint-compiler
command -v flatpak
command -v flatpak-builder
command -v appstreamcli
command -v desktop-file-validate
command -v git
command -v pkg-config
```

Also:

```bash
rustc --version
cargo --version
meson --version
ninja --version
flatpak --version
flatpak-builder --version
```

Then install only what is missing, preferring Arch/CachyOS repositories.

Current Arch package verification shows:

- `blueprint-compiler` is available in **Extra**, currently `0.22.2-1`. [Arch Linux](https://archlinux.org/packages/extra/any/blueprint-compiler/?utm_source=chatgpt.com)
- `flatpak-builder` is available in **Extra**, currently `1.4.11-1`. [Arch Linux](https://archlinux.org/packages/extra/x86_64/flatpak-builder/?utm_source=chatgpt.com)
- `appstream-glib` is available in **Extra**, currently `0.8.4-1`. [Arch Linux](https://archlinux.org/packages/extra/x86_64/appstream-glib/?utm_source=chatgpt.com)

Recommended installation shape, only for missing packages:

```bash
sudo pacman -S rust meson ninja blueprint-compiler flatpak flatpak-builder \
    appstream-glib desktop-file-utils pkgconf
```

`desktop-file-utils` supplies `desktop-file-validate`.

No AUR dependency should be necessary for the core development environment.

---

## 0.4 Rust dependencies

Initial dependency set:

```text
gtk4
libadwaita
gio
glib
cairo-rs
gdk4
pango
petgraph
serde
serde_json
thiserror
anyhow
```

Potentially later:

```text
rayon
```

For tests / development:

```text
proptest or quickcheck
```

Do not unnecessarily introduce async frameworks at this stage.

### Recommendation: no Tokio

Most Flatview operations are subprocesses, filesystem reads, parsing, and a background layout calculation. GLib/GTK already has its own main loop. Tokio would add another concurrency model without solving a fundamental requirement.

Use:

- standard Rust threads,
- GLib main-context callbacks/channels where useful,
- cancellable worker jobs.

---

## 0.5 Meson + Cargo build policy

Use **Meson as the top-level build system**, with Cargo providing Rust dependency resolution.

The build should establish these Meson targets:

```text
flatview
flatview-resources
flatview-blueprints
flatview-desktop
flatview-metainfo
flatview-tests
```

Meson supports Cargo workspace/package integration, including Rust executable targets. [Meson Build](https://mesonbuild.com/Rust-module.html?utm_source=chatgpt.com)

Avoid trying to duplicate the Rust dependency graph in Meson.

---

## 0.6 Blueprint

Blueprint is appropriate here, but keep in mind that its upstream documentation still describes it as experimental, so pinning the build environment matters. [GNOME Pages](https://gnome.pages.gitlab.gnome.org/blueprint-compiler/?utm_source=chatgpt.com)

Blueprint should be compiled as part of the GResource pipeline. Its documented Meson workflow is to compile `.blp` files into GTK XML and make that generated target a dependency of `gnome.compile_resources()`. [GNOME Pages](https://gnome.pages.gitlab.gnome.org/blueprint-compiler/setup.html?utm_source=chatgpt.com)

---

### Done when

A completely empty project can:

```bash
meson setup build
meson compile -C build
```

and produce:

```text
build/src/flatview
```

plus its compiled GResources.

---

# Phase 1 — Minimal GNOME application shell

## Goal

Get a proper GTK4 + Libadwaita application running before Flatpak integration or graph work.

### Tasks

Create:

```text
src/main.rs
src/application.rs
src/window.rs

data/ui/window.blp
data/ui/headerbar.blp
```

Implement:

- `GtkApplication`/`adw::Application`
- application ID
- `AdwApplicationWindow`
- header bar
- empty graph/content area
- initial refresh action
- about dialog
- keyboard accelerator infrastructure

Initialize Libadwaita normally.

Use:

```text
AdwApplication
    ↓
AdwApplicationWindow
    ↓
AdwToolbarView
    ├── HeaderBar
    └── Content
```

`AdwToolbarView` is the recommended contemporary structure for GNOME toolbar/header layouts, including correct header-bar styling. [GNOME Pages](https://gnome.pages.gitlab.gnome.org/libadwaita/doc/1.6/migrating-to-breakpoints.html?utm_source=chatgpt.com)

---

## Adaptive layout choice

Use:

```text
AdwNavigationSplitView
```

rather than manually hiding/showing arbitrary boxes.

Desktop:

```text
┌───────────────┬───────────────────────────────┐
│ Suggestions   │ Graph                         │
│ / filters     │                               │
│               │                               │
└───────────────┴───────────────────────────────┘
```

Narrow:

```text
┌─────────────────────────────────┐
│ Header                          │
├─────────────────────────────────┤
│ Graph                           │
│                                 │
├─────────────────────────────────┤
│ Suggestions / details           │
└─────────────────────────────────┘
```

Use `AdwBreakpoint` to collapse the split view. Libadwaita specifically provides `AdwNavigationSplitView` and `AdwBreakpoint` for this pattern. [GNOME Pages](https://gnome.pages.gitlab.gnome.org/libadwaita/doc/main/adaptive-layouts.html?utm_source=chatgpt.com)

Proposed thresholds:

```text
> 900sp       desktop split
600–900sp     compact split/overlay
≤ 600sp       navigation/collapsed layout
```

These values should be treated as initial values, not immutable design rules.

### Done when

- Application opens normally.
- Header bar follows Adwaita.
- Light/dark system appearance changes automatically.
- Window collapses gracefully at narrow widths.
- No custom colors have been introduced.

---

# Phase 2 — Flatpak inventory/data-collection core

## Goal

Build a complete, GTK-independent snapshot of the user's installed Flatpaks.

This is the most important non-UI subsystem.

---

## 2.1 Data model

Create:

```rust
model/installation.rs
model/app.rs
model/dependency.rs
model/flatpak_item.rs
```

Core structures should conceptually contain:

### `InstallationScope`

```text
User
System
```

Potentially later:

```text
Custom(path)
```

because Flatpak supports multiple installations.

Do not assume only `/var/lib/flatpak` and `~/.local/share/flatpak` will exist forever.

### `FlatpakApp`

Fields:

```text
id
name
branch
arch
version
scope
installation
runtime_id
runtime_branch
installed_size
icon
desktop_file
data_dir
metadata
```

### `FlatpakDependency`

Fields:

```text
id
display_name
kind
branch
arch
installed_size
scope
```

Where `kind` is:

```text
Runtime
Extension
Locale
Debug
Other
```

The first release should display Locale/Debug extensions but may filter them from the primary graph by default.

---

# Phase 3 — Flatpak collection architecture

## Recommended approach: hybrid CLI + metadata

The user brief requests both:

1. `flatpak` CLI interaction
2. direct metadata parsing

Keep both, but assign them different responsibilities.

### CLI is authoritative for:

- installed applications
- installed runtimes
- installation scope
- installed size
- runtime relationship
- active ref/version
- uninstall
- launching another Flatpak
- detecting Warehouse/Flatseal

`flatpak list` already provides application/runtime IDs, branches, runtimes, refs, installations and installed sizes through machine-readable columns. [Arch Manual Pages](https://man.archlinux.org/man/flatpak-list.1.en?utm_source=chatgpt.com)

`flatpak info` additionally exposes runtime, installed size, metadata, extensions, location and installation scope. [Arch Manual Pages](https://man.archlinux.org/man/extra/flatpak/flatpak-info.1.en?utm_source=chatgpt.com)

### Metadata parser is authoritative for:

- low-level dependency/extension details
- metadata values unavailable from `flatpak list`
- cross-checking CLI results
- future diagnostics

Flatpak metadata is INI-formatted, and installed deployments have metadata files under their deployment directories. [Arch Manual Pages](https://man.archlinux.org/man/flatpak-metadata.5.en?utm_source=chatgpt.com)

---

## 3.1 `flatpak/cli.rs`

Create a narrow wrapper:

```text
FlatpakCli
├── list_apps()
├── list_runtimes()
├── info(id, scope)
├── info_metadata(id, scope)
├── uninstall(id, scope)
├── run(id)
└── is_installed(id)
```

**Never** call `Command::new()` directly from UI code.

Every invocation should:

- use an argument array, never shell strings
- capture stdout/stderr separately
- validate exit status
- return structured errors
- support scope explicitly
- enforce a timeout for operations that can hang

---

# Phase 4 — Flatpak sandbox/host boundary

This should be treated as a first-class architectural subsystem rather than a packaging afterthought.

## Recommended `HostCommandRunner`

```text
flatpak/host.rs
```

Expose:

```text
run(command, args)
```

with two implementations conceptually:

```text
NativeHostCommandRunner
FlatpakHostCommandRunner
```

### Development mode

Direct:

```text
flatpak ...
```

### Shipped Flatpak mode

Use:

```text
flatpak-spawn --host flatpak ...
```

`flatpak-spawn --host` is specifically designed for applications inside a sandbox to execute commands on the host, but requires access to the `org.freedesktop.Flatpak` D-Bus interface. [Flatpak](https://docs.flatpak.org/es/latest/flatpak-command-reference.html?utm_source=chatgpt.com)

That lets the application continue using the required CLI architecture without giving the application blanket `--filesystem=host` access.

---

## 4.1 Permissions decision

There are two viable strategies.

### Option A — host filesystem read + host CLI

Grant read-only access to:

```text
/var/lib/flatpak
~/.local/share/flatpak
~/.var/app
```

and use:

```text
flatpak-spawn --host flatpak ...
```

**Advantages**

- direct metadata parsing works exactly as specified
- efficient bulk reads
- straightforward icon resolution
- CLI remains authoritative for mutating operations

**Disadvantages**

- larger sandbox permissions
- Flathub review needs justification

Flathub specifically flags access to `/var/lib/flatpak`, `~/.local/share/flatpak`, and `~/.var/app` for review, rather than treating them as ordinary permissions. [Flathub Documentation](https://docs.flathub.org/docs/for-app-authors/linter?utm_source=chatgpt.com)

### Option B — CLI-only collection

Run everything through host commands:

```text
flatpak list
flatpak info
flatpak info --show-metadata
```

**Advantages**

- much less filesystem exposure
- cleaner sandbox model

**Disadvantages**

- more subprocesses
- more parsing
- less direct access to exported icons/metadata
- potentially slower bulk inspection

### Recommendation

**Option A for v1**, with read-only filesystem access narrowly scoped to the two Flatpak installation directories and `~/.var/app`.

Then revisit CLI-only collection after profiling.

This is also an explicit product/security decision for the project owner.

---

# Phase 5 — Metadata parsing and dependency extraction

## Goal

Produce reliable dependency relationships.

### Files

```text
flatpak/metadata.rs
flatpak/parser.rs
flatpak/collector.rs
```

Use:

```text
glib::KeyFile
```

or an equivalent strict INI parser rather than handwritten line parsing.

The parser should understand at least:

```text
[Application]
runtime=
sdk=
command=
branch=

[Instance]
...

[Context]
...

[Session Bus Policy]
...
```

and extension-related metadata.

The exact parser should not assume every application contains every key.

---

## Dependency normalization

A runtime reference may contain:

```text
org.gnome.Platform/x86_64/50
```

Internally normalize it as:

```text
RuntimeRef {
    id: org.gnome.Platform
    arch: x86_64
    branch: 50
}
```

The **full ref** should remain available because:

```text
org.gnome.Platform/49
org.gnome.Platform/50
```

are different dependency nodes.

Do not collapse them just because their IDs share the same prefix.

---

## Special handling

### `org.gnome.Platform`

Treat it like every other dependency.

Do not hard-code GNOME runtimes as a special-case node.

The same applies to:

```text
org.freedesktop.Platform
org.kde.Platform
org.gnome.Sdk
```

although SDKs normally should not appear in the user's installed-application dependency graph unless actually relevant.

### Locale extensions

Represent them internally, but default:

```text
visible_in_primary_graph = false
```

because a graph with every locale/debug extension will become noisy.

### Debug extensions

Same policy.

---

## Missing dependency policy

A dependency referenced by an application but not found among installed runtime data should produce:

```text
DependencyState::Missing
```

rather than causing the entire application to disappear.

This is important for damaged/incomplete installations.

---

### Done when

Given a known fixture directory, tests can produce:

```text
Apps:
  org.gnome.Calculator
  org.mozilla.firefox

Dependencies:
  org.gnome.Platform/50
  org.freedesktop.Platform/25.08

Edges:
  Calculator → org.gnome.Platform/50
  Firefox    → org.gnome.Platform/50
```

with scope and size information intact.

---

# Phase 6 — Icon discovery

## Goal

Turn app IDs into `gdk::Texture`/`gdk::Paintable` instances usable by the graph renderer.

Flatpak exports application resources, including desktop files and icons, to the host. [Flatpak](https://docs.flatpak.org/en/latest/basic-concepts.html?utm_source=chatgpt.com)

Create:

```text
flatpak/icons.rs
```

with:

```text
resolve_icon(app) -> IconSource
```

Resolution order:

```text
1. exported Flatpak icon
2. desktop-file icon reference
3. application export icon
4. bundled generic fallback
```

Keep icon resolution lazy.

Do **not** decode every icon during initial inventory if there are many applications.

Use an icon cache:

```text
HashMap<AppId, Paintable>
```

with loading performed on demand as nodes enter the visible graph.

Fallback:

```text
application-x-executable-symbolic
```

or an equivalent system symbolic icon.

---

# Phase 7 — Domain graph model

## Goal

Build a graph independent from GTK.

Use `petgraph`.

`petgraph` currently provides `StableGraph`, graph algorithms, GraphMap and optional Rayon support. [Docs.rs](https://docs.rs/crate/petgraph/latest/features?utm_source=chatgpt.com)

## Recommendation

Use:

```text
StableGraph<NodeData, EdgeData, Directed>
```

rather than `GraphMap`.

Why:

- node weights carry substantial application/dependency metadata
- node indexes remain useful for layout
- `StableGraph` handles removals without invalidating every index
- edge direction is semantically useful

---

## Node types

```rust
enum NodeKind {
    App,
    Dependency,
}
```

The node weight should contain:

```text
NodeId
NodeKind
display_name
installed_size
branch
scope
app_id / dependency_ref
fan_in
icon
```

### Edge

```text
AppUsesDependency
```

No need for a complicated edge type initially.

---

## Derived metrics

Calculate:

```text
fan_in
fan_out
connected_component
dependency_size
app_size
```

Do not store redundant mutable versions unless profiling demonstrates a need.

---

# Phase 8 — Explicit node sizing model

## App nodes

Use installed size, but use a logarithmic scale so a 3 GB app does not overwhelm a 100 MB app.

Recommended starting formula:

```text
size_mib = max(installed_size / 1_048_576, 0.1)

radius =
    clamp(
        18,
        44,
        14 + 8 * sqrt(log2(1 + size_mib))
    )
```

Examples approximately:

```text
50 MiB      → ~32 px
100 MiB     → ~34 px
500 MiB     → ~38 px
1 GiB       → ~40 px
5 GiB       → capped near 44 px
```

The important characteristic is not these exact numbers but the floor/ceiling.

---

## Dependency nodes

Their visual prominence should be driven primarily by fan-in:

```text
radius =
    clamp(
        16,
        42,
        13 + 7 * sqrt(fan_in)
    )
```

So:

```text
1 app   → small
4 apps  → medium
9 apps  → larger
25+     → capped
```

Dependency installed size can affect saturation/gradient or a secondary ring rather than its diameter.

That preserves the intended visual semantics:

> **App size = visual size. Dependency fan-in = visual size.**

---

# Phase 9 — Graph layout engine

## Goal

Produce a physics-based graph that remains usable with large installations.

### Options

### Option 1 — hand-written spring/repulsion

```text
spring force
+
node-node repulsion
+
centering force
+
velocity damping
```

**Pros**

- complete control
- no additional layout dependency
- easy to tune around application/dependency semantics

**Cons**

- naïve repulsion is O(n²)
- more code
- numerical stability needs care

### Option 2 — dedicated force-layout crate

Potentially less code.

**Pros**

- existing algorithms
- potentially more mature physics implementation

**Cons**

- library quality/API suitability may not match GTK interaction needs
- another dependency
- difficult to customize semantic clustering

### Option 3 — hierarchical/bipartite layout

Applications on one side, runtimes on the other.

**Pros**

- very legible
- predictable

**Cons**

- fundamentally different from the Obsidian-style visual requested
- reduces exploration feel

### Recommendation

**Option 1**, but implement Barnes-Hut-style spatial partitioning for repulsion.

Target complexity:

```text
O(N log N)
```

rather than naïve:

```text
O(N²)
```

---

## 9.1 Layout phases

Create:

```text
graph/physics.rs
graph/spatial.rs
graph/layout.rs
```

Simulation:

```text
1. initialize positions
2. compute spatial tree
3. calculate repulsion
4. calculate spring forces
5. calculate center/gravity force
6. integrate velocity
7. damp
8. clamp to finite bounds
9. repeat until converged
```

---

## 9.2 Semantic forces

Add a small force encouraging:

```text
Apps       → outer cloud
Dependencies → inner/high-connectivity region
```

but do not make that an explicit fixed ring.

This preserves organic movement while making dependency hubs easier to understand.

---

## 9.3 Worker thread

The layout engine must not run on the GTK thread.

Architecture:

```text
GTK thread
    │
    │ graph snapshot
    ▼
layout worker
    │
    │ LayoutSnapshot
    ▼
GTK thread
```

`LayoutSnapshot` should contain only plain Rust data:

```text
node_id
x
y
vx
vy
radius
```

Do not send GTK objects, textures, widgets, or `GObject`s to the worker.

---

## 9.4 Incremental simulation

Do not completely reset the layout every refresh.

Match nodes by stable ID:

```text
existing node → retain position
new node       → spawn near neighbor / component centroid
removed node   → disappear
```

This makes refreshes visually stable.

---

# Phase 10 — GraphCanvas

## Goal

Create the actual interactive canvas.

Create:

```text
ui/graph_canvas.rs
```

using `GtkDrawingArea`.

GTK4's `GtkDrawingArea` explicitly provides `set_draw_func()` for custom rendering. [https://docs.gtk.org](https://docs.gtk.org/gtk4/class.DrawingArea.html?utm_source=chatgpt.com)

### Initial implementation

Use Cairo for the first renderer because it makes:

- hit testing
- circles
- paths
- text
- icon placement

straightforward.

### Longer-term rendering

Keep rendering behind:

```text
trait GraphRenderer
```

with the possibility of a GSK/Snapshot implementation later.

GTK4's modern rendering model is scene-graph based, and `GtkSnapshot` can generate render nodes; current GTK also supports gradients and textures directly through snapshot operations. [https://docs.gtk.org](https://docs.gtk.org/gtk4/drawing-model.html?utm_source=chatgpt.com)

### Recommendation

**Cairo first; retain a migration path to Snapshot/GSK.**

That gets the product functional sooner while keeping the canvas architecture replaceable.

---

# Phase 11 — Theme-aware graph visuals

Hard-coded colors should be avoided.

GTK4 specifically provides `gtk_widget_get_color()` for custom drawing that follows the current widget foreground style. [https://docs.gtk.org](https://docs.gtk.org/gtk4/method.Widget.get_color.html?utm_source=chatgpt.com)

Create a small:

```text
GraphPalette
```

calculated from the current GTK/Adwaita style.

Base it on:

```text
foreground
background
accent
success
warning
error
```

where practical.

Dependency colors can then be derived from the current accent rather than using fixed neon RGB values.

Example semantic approach:

```text
Dependency family A:
    accent hue

Dependency family B:
    accent hue + small hue shift

Selected node:
    accent + stronger alpha

Inactive edge:
    foreground with low alpha

Selected edge:
    foreground/accent with high alpha
```

This makes the graph adapt to light/dark mode without maintaining separate light/dark palettes.

---

# Phase 12 — Pan, zoom and navigation

## Goal

Match the usability characteristics of Obsidian's graph view.

Implement:

```text
Zoom
 ├── mouse wheel / trackpad
 └── pinch where available

Pan
 ├── middle-button drag
 ├── space + drag
 └── touch drag

Node drag
 └── pointer drag directly on node
```

Maintain:

```text
ViewTransform {
    offset_x
    offset_y
    zoom
}
```

All graph coordinates remain in world space.

Screen transformation:

```text
screen = world * zoom + offset
```

---

## Zoom limits

Start with:

```text
min_zoom = 0.15
max_zoom = 4.0
```

but tune after real-world testing.

Do not let double-click or scroll unexpectedly re-center the whole graph.

---

# Phase 13 — Spatial hit testing

A naïve:

```text
for every node:
    calculate distance
```

is sufficient for 20 nodes but unnecessary for hundreds.

Reuse the spatial tree from the layout engine, or build a lightweight quadtree for the rendered scene.

Create:

```text
graph/hit_test.rs
```

Operations:

```text
node_at(point)
nodes_in_rect(rect)
edges_near(point)
```

Node hit testing should prioritize:

1. app/dependency node
2. selected node
3. connected edge

---

# Phase 14 — Hover highlighting

## Goal

Provide the requested Obsidian-like interaction.

When the cursor is over a node:

```text
hovered node       → full opacity
connected nodes    → full/high opacity
connected edges    → highlighted
unconnected graph  → reduced opacity
```

This must be done purely by rendering state.

Do not mutate the actual graph.

Create:

```text
SelectionState {
    hovered
    selected
    context_target
}
```

---

# Phase 15 — App node context menu

## Goal

A GNOME-style app-node action menu.

Create:

```text
ui/app_popover.rs
actions/applications.rs
```

Actions:

```text
Open in Warehouse
Open in Flatseal
Open data directory
Uninstall…
```

Do not make destructive actions directly execute.

---

## 15.1 Correct application IDs

Verified against Flathub:

```text
Warehouse
io.github.flattool.Warehouse
``` :chatgpt-content-reference{index="20"}


```text
Flatseal
com.github.tchx84.Flatseal
``` :chatgpt-content-reference{index="21"}


The prompt's concern was correct: these must not be swapped.

---

## 15.2 Availability detection

Implement:

```text
is_installed("io.github.flattool.Warehouse")
is_installed("com.github.tchx84.Flatseal")
```

Prefer:

```text
flatpak list --app --columns=application
```

once per refresh, build:

```text
HashSet<AppId>
```

and then do local membership tests.

Do **not** execute:

```text
flatpak list
```

every time a menu opens.

---

## 15.3 Launching external Flatpaks

Do not construct shell commands.

Use:

```text
flatpak run --command=...
```

through the host-command abstraction.

Conceptually:

```text
host.run(
    "flatpak",
    ["run", "io.github.flattool.Warehouse", "--", app_id]
)
```

The exact Warehouse command-line interface should be treated as a separate compatibility integration and verified against the currently installed version before implementing app-specific arguments.

For v1, the safest fallback is:

```text
Open Warehouse
```

without attempting to force Warehouse to select a specific app unless its current CLI/API reliably supports it.

Same for Flatseal.

---

# Phase 16 — Data directory opening

The directory is conceptually:

```text
~/.var/app/<app-id>/
```

The application should **not** interpolate arbitrary shell text.

Construct a real path:

```text
PathBuf
```

validate that the resulting directory exists, then open it.

Preferred route:

```text
GIO / OpenURI portal
```

Fallback:

```text
xdg-open
```

through the host-command boundary.

This avoids building a shell command such as:

```text
sh -c "xdg-open ..."
```

which should never be used.

---

# Phase 17 — Uninstall flow

## Goal

Make uninstall explicit, safe, and reversible where possible.

Interaction:

```text
Click "Uninstall…"
        ↓
AdwAlertDialog
        ↓
"Uninstall <App Name>?"
        ↓
Cancel / Uninstall
```

Show:

```text
App name
Installed size
Application ID
Scope
```

The dialog should clearly distinguish:

```text
Remove application
```

from:

```text
Remove application data
```

For v1, **only uninstall the Flatpak application**.

Do not automatically delete:

```text
~/.var/app/<app-id>
```

unless a separate, explicit later action is introduced.

The underlying CLI operation is:

```text
flatpak uninstall <app-id>
```

Flatpak documents that command as the standard application removal operation. [Flatpak](https://docs.flatpak.org/en/latest/using-flatpak.html?utm_source=chatgpt.com)

After successful removal:

```text
refresh inventory
rebuild graph
refresh suggestions
```

---

# Phase 18 — Cleanup Suggestions Engine

## Goal

Produce useful candidates without implying that Flatview knows an application's intent.

This distinction matters.

A runtime used by one application may still be important. Therefore suggestions must say:

> "Candidate for review"

not:

> "Safe to remove."

---

## Initial heuristic

For each dependency:

```text
fan_in = number of installed apps using dependency
size = installed size
```

Candidate condition:

```text
fan_in <= 3
```

Then calculate:

```text
size_score = normalized(log(size))
usage_score =
    1.0 if fan_in == 1
    0.55 if fan_in == 2
    0.25 if fan_in == 3
    0   otherwise
```

Overall:

```text
score =
    0.7 * size_score +
    0.3 * usage_score
```

This is an internal sorting value, not something displayed to users as a "risk score".

---

## Important semantic rule

A suggestion must never tell the user:

```text
"This runtime is unused."
```

unless Flatpak itself reports it is actually unused.

Instead:

```text
"Used by 1 application"
```

or:

```text
"Used by 2 applications"
```

and:

```text
"Large runtime — review whether it is still needed"
```

This is more accurate.

---

# Phase 19 — Suggestions UI

## Desktop

Recommended:

```text
AdwNavigationSplitView

sidebar:
    Cleanup candidates

content:
    Graph
```

or, when the panel is intentionally lightweight:

```text
Graph
────────────
Suggestions
```

## Narrow/mobile

Suggestions should become a navigable page rather than permanently stealing graph space.

Use `AdwNavigationSplitView` collapsing into navigation pages, as intended by Libadwaita's adaptive design. [GNOME Pages](https://gnome.pages.gitlab.gnome.org/libadwaita/doc/main/adaptive-layouts.html?utm_source=chatgpt.com)

Each suggestion row:

```text
[icon]
org.gnome.Platform 50
1 application
1.2 GiB
Review
```

Activating it:

```text
→ select dependency node
→ zoom graph to node
→ highlight dependent applications
```

---

# Phase 20 — Refresh architecture

## Initial refresh

At application startup:

```text
show loading state
      ↓
collect Flatpak data
      ↓
parse
      ↓
build graph
      ↓
calculate suggestions
      ↓
start layout
      ↓
render
```

Never block the GTK main thread with:

```text
flatpak list
metadata parsing
layout physics
```

---

# Phase 21 — Live change monitoring

There are three possibilities.

### A. Manual refresh only

Very reliable.

### B. Polling

For example:

```text
every 10 seconds while window is active
```

Easy but wasteful.

### C. File monitoring + explicit refresh

Monitor:

```text
/var/lib/flatpak
~/.local/share/flatpak
```

and react to meaningful changes.

### Recommendation

**v1: manual refresh + refresh after Flatview actions.**

Do not build filesystem monitoring until the rest of the application is stable.

For a desktop utility, a Refresh action is perfectly reasonable.

Later add monitored refresh if real users demonstrate a need.

---

# Phase 22 — Rendering performance

## Required optimization stages

Do not prematurely optimize the first graph implementation.

### Stage 1

Normal Cairo drawing.

### Stage 2

Spatial hit-test structure.

### Stage 3

Incremental layout.

### Stage 4

Skip labels at low zoom.

### Stage 5

Skip rendering low-importance nodes at extreme zoom-out.

### Stage 6

Snapshot/GSK renderer if profiling shows Cairo is inadequate.

GTK's scene-graph architecture can cache render nodes and offers GPU-oriented renderers, so GSK is a credible future optimization path. [https://docs.gtk.org](https://docs.gtk.org/gtk4/drawing-model.html?utm_source=chatgpt.com)

---

# Phase 23 — Label rendering strategy

At normal zoom:

```text
Dependency:
    circular node
    "org.gnome.Platform"
    "1.2 GiB"

App:
    icon
    "Firefox"
```

At low zoom:

```text
icons only
```

At very low zoom:

```text
no labels
```

This is critical for large installations.

---

# Phase 24 — Accessibility

A raw canvas is inherently harder for accessibility than ordinary GTK widgets.

Do not treat this as "the graph is inherently inaccessible."

Provide an alternate accessible representation.

Recommended approach:

```text
Graph
  ↓
Accessible List View
```

with:

```text
Applications
Dependencies
```

Each accessible item can expose:

```text
Name
Type
Installed size
Number of dependencies / dependents
```

Keyboard interaction:

```text
Tab → graph
Arrow keys → selected node navigation
Enter → details/action menu
Escape → clear selection
```

At minimum, all toolbar buttons, dialogs, suggestion rows and menu items must have accessible labels.

---

# Phase 25 — Light/dark mode and visual system

Use Adwaita's system style management.

Do not build:

```text
if dark:
   #xxxxxx

else:
   #yyyyyy
```

for normal UI.

Use:

- Adwaita classes
- symbolic icons
- semantic colors
- `GtkWidget` theme colors for custom drawing

GTK specifically documents `gtk_widget_get_color()` as the mechanism for custom-drawn content that needs to match the current style. [https://docs.gtk.org](https://docs.gtk.org/gtk4/method.Widget.get_color.html?utm_source=chatgpt.com)

---

# Phase 26 — Blueprint/GResource integration

Create:

```text
data/io.github.example.Flatview.gresource.xml
```

Bundle:

```text
ui/*.ui
icons/*
```

Blueprint build flow:

```text
.blp
  ↓ blueprint-compiler
GTK .ui
  ↓ gnome.compile_resources
GResource
  ↓
application
```

That matches Blueprint's documented Meson integration pattern. [GNOME Pages](https://gnome.pages.gitlab.gnome.org/blueprint-compiler/setup.html?utm_source=chatgpt.com)

Rust should use:

```text
gio::resources_register_include!
```

or the equivalent generated-resource registration path. The `gio` Rust bindings expose the resource registration macro for compiled GResources. [Docs.rs](https://docs.rs/gio/latest/gio/?utm_source=chatgpt.com)

---

# Phase 27 — Desktop file

Create:

```text
data/io.github.example.Flatview.desktop
```

Requirements:

```text
Type=Application
Name=Flatview
Exec=flatview
Icon=io.github.example.Flatview
Categories=Utility;
StartupNotify=true
```

Do not put command-line flags into `Exec` until they are actually supported.

Validate with:

```bash
desktop-file-validate data/io.github.example.Flatview.desktop
```

---

# Phase 28 — AppStream metadata

Create:

```text
data/io.github.example.Flatview.metainfo.xml
```

Include:

```text
component
id
name
summary
metadata_license
project_license
description
launchable
icon
categories
keywords
screenshots
releases
supports
```

Flathub's current requirements expect the file in:

```text
/app/share/metainfo/
```

with an exact application-ID filename, and require validation. They also require release information and appropriate desktop/mobile metadata. [Flathub Documentation](https://docs.flathub.org/docs/for-app-authors/metainfo-guidelines?utm_source=chatgpt.com)

Because the project explicitly targets narrow/mobile-sized layouts, add appropriate display-length/control metadata once the minimum supported layout size has been established.

---

# Phase 29 — Flatpak manifest

Create:

```text
packaging/io.github.example.Flatview.yml
```

Initial manifest architecture:

```text
app-id: io.github.example.Flatview
runtime: org.gnome.Platform
runtime-version: "50"
sdk: org.gnome.Sdk
command: flatview
```

GNOME Platform 50 is currently available on Flathub for both x86_64 and aarch64, and the July 2026 runtime status listed GNOME 50 among supported GNOME runtimes. [Flathub](https://flathub.org/az/apps/org.gnome.Platform?utm_source=chatgpt.com)

Use GNOME 50 as the starting target unless implementation reveals a dependency requiring another version.

---

## 29.1 Flatpak permissions

Expected baseline:

```text
--share=ipc
--socket=wayland
--socket=fallback-x11
--device=dri
```

Then only add:

```text
--talk-name=org.freedesktop.Flatpak
```

if the host command path is retained.

Filesystem:

```text
--filesystem=/var/lib/flatpak:ro
--filesystem=~/.local/share/flatpak:ro
--filesystem=~/.var/app:ro
```

only if direct metadata/icon/data access proves necessary.

Do not use:

```text
--filesystem=host
```

or:

```text
--filesystem=home
```

as a shortcut.

Flathub's linter explicitly scrutinizes broad home/host/filesystem access. [Flathub Documentation](https://docs.flathub.org/docs/for-app-authors/linter?utm_source=chatgpt.com)

---

# Phase 30 — Local Flatpak build workflow

Once the native build works:

```bash
flatpak-builder \
    --user \
    --install \
    --force-clean \
    --repo=repo \
    builddir \
    packaging/io.github.example.Flatview.yml
```

This is the standard local workflow documented by Flatpak. [Flatpak](https://docs.flatpak.org/en/latest/first-build.html?utm_source=chatgpt.com)

Then:

```bash
flatpak run io.github.example.Flatview
```

Test both:

```bash
flatpak run --user io.github.example.Flatview
```

and, where useful:

```bash
flatpak run --user \
    --devel \
    io.github.example.Flatview
```

Do not consider "works when run from Cargo" sufficient acceptance.

**The actual Flatpak sandbox is the product environment.**

---

# Phase 31 — Flatpak-specific integration tests

Create a test matrix:

| Scenario | Expected |
|---|---|
| no Flatpaks installed | empty state |
| only one app | one app + runtime |
| multiple apps share runtime | runtime fan-in increases |
| user install | appears |
| system install | appears |
| both user/system same ID | scopes remain distinguishable |
| missing metadata | graceful error |
| missing icon | fallback |
| missing runtime | dependency remains visible as missing |
| Warehouse installed | action enabled |
| Warehouse absent | action unavailable |
| Flatseal installed | action enabled |
| Flatseal absent | action unavailable |
| uninstall canceled | nothing changes |
| uninstall succeeded | graph refreshes |
| uninstall failed | error dialog |
| dark theme | readable |
| narrow window | usable |
| very large graph | stays interactive |

---

# Phase 32 — Testing the domain layer

This should be one of the strongest parts of the project.

## `tests/parser.rs`

Fixtures:

```text
metadata-basic.ini
metadata-runtime.ini
metadata-extensions.ini
metadata-malformed.ini
```

Test:

```text
parse valid
reject malformed
ignore irrelevant keys
handle missing keys
```

## `tests/graph.rs`

Test:

```text
graph construction
fan-in calculation
connected components
missing dependencies
duplicate refs
```

## `tests/sizing.rs`

Test mathematically:

```text
small size > minimum
large size < maximum
monotonicity
finite values
```

## `tests/suggestions.rs`

Test:

```text
fan-in 1 → candidate
fan-in 3 → candidate
fan-in 4 → not candidate
large > small when same fan-in
zero/missing size handled
```

---

# Phase 33 — CI

## Recommendation

Use GitHub Actions.

Pipeline:

```text
push / pull request
        ↓
cargo fmt --check
        ↓
cargo clippy --all-targets --all-features
        ↓
cargo test
        ↓
meson setup
meson compile
        ↓
desktop-file-validate
        ↓
appstreamcli validate
        ↓
flatpak-builder
        ↓
flatpak-builder-lint
```

The Flatpak build should actually execute during CI.

Do not make "Flatpak packaging" a separate afterthought.

---

# Phase 34 — Dependency vendoring / offline Flatpak builds

Flathub builds do not have unrestricted network access during the build. Dependencies therefore need to be supplied through the manifest/source structure. [Flathub Documentation](https://docs.flathub.org/docs/for-app-authors/requirements?utm_source=chatgpt.com)

This is especially important for Cargo.

The project should eventually use a reproducible Cargo dependency source strategy compatible with Flatpak builds, rather than relying on:

```text
cargo fetch
```

against the public network during a Flathub build.

This becomes a dedicated packaging milestone once the application compiles.

---

# Phase 35 — `flatpak-external-data-checker`

Evaluate this after the manifest stabilizes.

It is useful primarily when the manifest contains external source artifacts whose URLs/checksums need automated monitoring.

It is **not** a requirement for the basic Rust/Blueprint/Flatview application build.

Priority:

```text
CI linting             → yes
flatpak-builder        → yes
AppStream validation   → yes
desktop validation     → yes
external-data-checker  → later
```

---

# Phase 36 — GNOME Builder vs VS Code

### GNOME Builder

**Pros**

- GNOME-first environment
- Flatpak integration
- excellent platform awareness

**Cons**

- another IDE/workflow
- less aligned with your existing development habits

### VS Code + rust-analyzer + Meson

**Pros**

- already familiar
- excellent Rust tooling
- easy debugging/test workflow

**Cons**

- more manual GNOME/Flatpak integration

### Recommendation

Use **VS Code + rust-analyzer + Meson/Ninja** as the primary development environment.

Keep GNOME Builder as an optional verification environment, not a project dependency.

---

# Phase 37 — Developer tooling

Use:

```text
rustfmt
clippy
rust-analyzer
blueprint-compiler
meson
ninja
flatpak-builder
appstreamcli
desktop-file-validate
```

Developer command set:

```bash
cargo fmt --check
cargo clippy --all-targets --all-features
cargo test
meson compile -C build
```

plus:

```bash
appstreamcli validate --explain \
    data/io.github.example.Flatview.metainfo.xml

desktop-file-validate \
    data/io.github.example.Flatview.desktop
```

Flatpak's own documentation recommends `appstreamcli validate --explain`, while Flathub's current validation goes further with its builder linter. [Flatpak](https://docs.flatpak.org/en/latest/conventions.html?utm_source=chatgpt.com)

---

# Phase 38 — GNOME HIG polish

The final UI pass should explicitly audit:

### Header bar

- title
- refresh action
- menu
- no unnecessary toolbar clutter

### Menus

- standard action ordering
- destructive action separated
- appropriate keyboard accelerators

### Dialogs

Use modern Libadwaita alert/dialog patterns.

### Empty state

Show:

```text
No Flatpak applications found
```

with useful secondary explanation.

### Loading

Show a non-jarring loading state while inventory collection happens.

### Errors

Never expose raw:

```text
Command failed with status 1
```

as the primary user message.

Instead:

```text
Could not refresh Flatpak information
```

with technical details available in an expandable/secondary area.

---

# Phase 39 — Performance acceptance testing

Create synthetic datasets:

```text
10 apps
50 apps
100 apps
250 apps
500 apps
1000 apps
```

with varying dependency densities.

Measure:

```text
inventory collection time
graph construction time
initial layout time
steady-state frame rate
memory consumption
refresh latency
```

Target behavior:

- UI remains responsive while collection/layout runs
- pan/zoom remains interactive while layout converges
- labels disappear before rendering becomes overwhelmingly expensive
- graph does not restart from random positions on every refresh

The exact numerical performance budget should be decided after the first real profiling pass.

---

# Phase 40 — Release candidate acceptance

Before calling Flatview v1 shippable:

### Functional

- apps detected
- runtimes detected
- extensions represented
- sizes displayed
- fan-in correct
- graph edges correct
- pan/zoom works
- dragging works
- hover works
- app menu works
- Warehouse detection works
- Flatseal detection works
- data-directory action works
- uninstall works
- refresh works
- suggestions work

### UX

- desktop layout
- narrow layout
- light theme
- dark theme
- keyboard interaction
- screen-reader-compatible alternative representation
- no destructive operation without confirmation

### Packaging

- Flatpak installs
- Flatpak launches
- no host-only development assumptions
- AppStream valid
- desktop file valid
- Flatpak manifest valid
- Flathub linter clean
- no unnecessary filesystem permissions

### Community

Repository contains:

```text
LICENSE
README.md
CONTRIBUTING.md
CODE_OF_CONDUCT.md
```

The GNOME Code of Conduct should be explicitly adopted and linked. GNOME currently directs participants to `conduct.gnome.org`, with the Code of Conduct applying across GNOME project/community spaces. [GNOME Blogs](https://blogs.gnome.org/conduct/2026/07/08/transparency-report-from-october-2025-to-june-2026/?utm_source=chatgpt.com)

---

# Recommended milestone sequence

The practical implementation order should be:

```text
M0  Repository + toolchain
 ↓
M1  Empty GNOME application
 ↓
M2  Flatpak CLI abstraction
 ↓
M3  Inventory collector
 ↓
M4  Metadata parser
 ↓
M5  Domain graph
 ↓
M6  Static graph renderer
 ↓
M7  Physics layout
 ↓
M8  Pan/zoom/selection
 ↓
M9  App context actions
 ↓
M10 Suggestions
 ↓
M11 Adaptive/mobile UI
 ↓
M12 Accessibility
 ↓
M13 Flatpak sandbox integration
 ↓
M14 CI + packaging
 ↓
M15 Performance/polish
 ↓
M16 Release candidate
```

This order is intentional: **prove the difficult data model before spending a lot of time polishing the graph UI.**

---

# Important open decisions for the project owner

These are the questions I'd resolve before implementation begins.

### 1. Application ID

What exact permanent Flatview application ID should the project use?

Suggested:

```text
io.github.<your-github-username>.Flatview
```

This affects nearly every packaging file.

### 2. Primary dependency visibility

Should locale/debug extensions be:

```text
hidden from the primary graph
```

while still counted internally, or should there be a graph filter that allows displaying them?

I recommend hidden-by-default.

### 3. Scope visualization

If the same runtime exists at both system and user scope, should these be:

```text
two separate nodes
```

or one dependency node with:

```text
system + user badges
```

I recommend **one logical graph node per exact runtime ref + installation scope**, with scope represented visually.

### 4. Host integration

The biggest architectural choice is whether to allow:

```text
--filesystem=/var/lib/flatpak:ro
--filesystem=~/.local/share/flatpak:ro
--filesystem=~/.var/app:ro
```

and use `flatpak-spawn --host` for mutation/launch operations.

I recommend this for v1 because it directly satisfies the hybrid CLI + metadata requirement, but this should be considered an explicit security/product decision because Flathub scrutinizes these permissions. [Flathub Documentation](https://docs.flathub.org/docs/for-app-authors/linter?utm_source=chatgpt.com)

### 5. Runtime removal

Should Flatview ever remove runtimes itself?

For v1, I recommend **no**.

App uninstall is straightforward; runtime removal has considerably more consequential dependency semantics.

Instead, Flatview can direct users toward:

```text
"Used by 1 app"
```

and later potentially provide a dedicated runtime-removal workflow.

### 6. Warehouse/Flatseal deep linking

Should Flatview merely launch:

```text
Warehouse
Flatseal
```

or should it attempt to select the clicked application within those tools?

I recommend launching the tools without deep-linking for v1 unless their current interfaces provide a stable supported mechanism.

### 7. Graph defaults

The main visual design decision is:

```text
organic force graph
```

versus:

```text
more strongly clustered bipartite graph
```

I recommend the organic force graph because it matches the requested Obsidian-like behavior, while adding semantic forces that keep runtimes visually central.

---

# Proposed final architecture

The resulting architecture would look like this:

```text
                         ┌──────────────────────┐
                         │   Adwaita UI         │
                         │                      │
                         │ Window               │
                         │ GraphCanvas          │
                         │ Suggestions          │
                         │ Menus / Dialogs      │
                         └──────────┬───────────┘
                                    │
                              GTK main thread
                                    │
                    ┌───────────────▼───────────────┐
                    │ Application State             │
                    │                               │
                    │ GraphModel                    │
                    │ LayoutSnapshot                │
                    │ SelectionState                │
                    │ SuggestionModel               │
                    └───────────────┬───────────────┘
                                    │
              ┌─────────────────────┼────────────────────┐
              │                     │                    │
              ▼                     ▼                    ▼
       ┌─────────────┐       ┌──────────────┐     ┌─────────────┐
       │ Flatpak     │       │ Layout       │     │ Suggestions │
       │ Collector   │       │ Worker       │     │ Engine      │
       └──────┬──────┘       └──────────────┘     └─────────────┘
              │
      ┌───────┴─────────┐
      │                 │
      ▼                 ▼
┌────────────┐   ┌──────────────┐
│ Flatpak CLI│   │ Metadata     │
│ abstraction│   │ parser       │
└─────┬──────┘   └──────┬───────┘
      │                 │
      └────────┬────────┘
               ▼
     ┌───────────────────┐
     │ Host integration  │
     │                   │
     │ flatpak-spawn     │
     │ filesystem RO     │
     │ portals           │
     └───────────────────┘
```

The most important implementation principle is to **keep the domain/data layer fully usable without GTK**. That gives Flatview a strong testing foundation while allowing the graph renderer, adaptive UI, and Flatpak sandbox integration to evolve independently.

The other major recommendation is to **treat Flatpak sandboxing as part of the product architecture from day one**, rather than first writing a host application and attempting to retrofit it into a Flatpak later. The current Flatpak model explicitly isolates host files/processes and expects host interaction to go through permissions and portals/Flatpak interfaces. [Flatpak](https://docs.flatpak.org/en/latest/basic-concepts.html?utm_source=chatgpt.com)

A sensible first implementation milestone is therefore **Phase 0 → Phase 1 → Phase 2**, ending with a Flatview window that can collect and display a textual inventory of the user's applications/runtimes. The graph should only begin once that inventory is proven correct.
