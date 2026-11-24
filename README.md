# flutter-rs

[![Crates.io][crates-badge]][crates-url]
[![flutter version][flutter-badge]][flutter-url]
[![MIT licensed][mit-badge]][mit-url]

Build Flutter desktop apps in Dart and Rust.

![flutter-app-template][flutter-app-template]

## Status

This workspace targets the modern Flutter embedder API and is developed against
Flutter **3.50 – 3.51.x**: 3.50.0 stable (engine `371f7720…`) and
3.51.0-1.0.pre (engine `f586770e…`, content hash `2f315d36…`). The vendored
`flutter-engine-sys/flutter-engine.h` is Flutter 3.50's `flutter_embedder.h`
verbatim; the multi-view APIs it exposes require the Flutter 3.50 framework.
`examples/multiple_windows` runs Flutter's official multi-window reference app
on the GLFW backend.

## Multi-window on the official embedder API

Multi-window support is not a private protocol: flutter-rs implements the same
C embedder API that Flutter's official C++ embedders (GTK, Windows, macOS) use.
One `FlutterEngine` drives any number of views:

- `FlutterEngineAddView`/`FlutterEngineRemoveView` create and destroy views at
  runtime; the first window still owns the implicit view `0`, so `runApp` apps
  work unchanged;
- each view receives its own `FlutterWindowMetricsEvent`, pointer/scroll events
  and `FlutterViewFocusEvent`, and the framework's
  `view_focus_change_request_callback` is honoured;
- a `FlutterCompositor` supplies one backing store per view
  (`create_backing_store`/`collect_backing_store`) and presents it through
  `present_view_callback`; the renderer is OpenGL on Linux/Windows, Metal on
  macOS (Impeller by default, Skia with `--enable-impeller=false`), with a
  software fallback;
- `engine_id` carries a pointer to the windowing function table, mirroring the
  native engine pointer official embedders expose to Dart.

The framework's own Dart windowing APIs — `WindowController`,
`DialogWindowController`, `TooltipWindowController`, `PopupWindowController`,
`SatelliteWindowController` — run unmodified on top: no Flutter SDK patches, no
exported C symbols, and no private channels. `examples/multiple_windows` is the
SDK's reference app with only a `main_glfw.dart` wrapper and a Rust runner.

Verified configurations:

| Platform | Renderer | Status |
| --- | --- | --- |
| Linux (X11, XWayland) | OpenGL compositor | verified |
| macOS (Apple Silicon) | Metal + Impeller, Skia, software | verified |
| Windows 11 VM | OpenGL (Mesa llvmpipe) | exercised, see CHANGELOG |

Native Wayland cannot position top-level windows; run under XWayland for
tooltips, popups and satellites.

The newest release on crates.io is **0.3.5 (2019)**. Everything here is the
unreleased 0.5.0 work; see [CHANGELOG.md](CHANGELOG.md).

`cargo-flutter`, which older copies of these instructions (and the app
template's own README) tell you to install, has not been released since
January 2020 and no longer compiles. There is no replacement CLI: you drive
`flutter` and `cargo` yourself, as described below.

## Crates

| Crate | What it is |
| --- | --- |
| [`flutter-engine-sys`](flutter-engine-sys) | Raw bindgen bindings to the embedder C API. |
| [`flutter-engine`](flutter-engine) | Safe engine lifecycle, platform-message channels, codecs, task runner, textures. |
| [`flutter-plugins`](flutter-plugins) | The `flutter/*` system channels: text input, keyboard, clipboard, cursor, locales, settings, … |
| [`flutter-glfw`](flutter-glfw) | The windowing backend: one engine, many GLFW windows/views, compositor, event loop. The only maintained one. |
| `flutter-glfw/dart` | `flutter_glfw_windowing`, the Dart `WindowingOwner` that lets the framework's windowing API drive flutter-glfw. |
| [`flutter-tools`](flutter-tools) | Finds the Flutter SDK and downloads the matching engine library. |

`flutter-winit` is still in the tree but is unmaintained, does not compile, and
is `exclude`d from the workspace.

## Requirements

- [Rust](https://www.rust-lang.org/tools/install) 1.88 or newer for
  flutter-engine-sys / flutter-engine / flutter-plugins / flutter-tools;
  flutter-glfw declares 1.95, because glfw 0.63 uses `cfg_select!`.
- A [Flutter SDK](https://flutter.dev/) on `PATH` (or `FLUTTER_ROOT` set),
  3.50 – 3.51.x. The windowing APIs used by `examples/multiple_windows` require
  Flutter 3.50+; they are behind a runtime feature flag, which the example's
  entrypoint enables itself (see its README).
- On Linux, the system packages listed under
  [Building and testing](CONTRIBUTING.md#building-and-testing).

## Flutter engine library

The embedder links against `libflutter_engine` (`.so` / `.dylib` /
`flutter_engine.dll`), which must match your Flutter SDK's engine revision.
Nothing in this repository ships or builds it for you.

### JIT (debug) — downloaded

Google publishes a JIT build of the generic embedder for each engine revision.
`flutter-tools` fetches and caches it:

```rust
use flutter_tools::{Build, Engine, Flutter};

let flutter = Flutter::auto_detect()?;
let engine = Engine::new(
    flutter.engine_version()?,
    "x86_64-unknown-linux-gnu".to_owned(),
    Build::Debug,
)
.with_realm(flutter.engine_realm());

engine.download()?;
println!("{}", engine.library_path()?.display());
```

Or by hand, which is what CI does:

```bash
hash="$(cat "$FLUTTER_ROOT/bin/cache/engine.stamp" 2>/dev/null \
  || cat "$FLUTTER_ROOT/bin/internal/engine.version")"
curl -fL -o embedder.zip \
  "https://storage.googleapis.com/flutter_infra_release/flutter/$hash/linux-x64/linux-x64-embedder.zip"
unzip embedder.zip -d ~/.flutter-engine   # libflutter_engine.so lands here
```

Official builds no longer link the ICU data in, so you must also pass
`icudtl.dat` (`Flutter::icudtl_path`, or
`$FLUTTER_ROOT/bin/cache/artifacts/engine/*/icudtl.dat`) to
`FlutterEngineBuilder::with_icu_data_path` /
`FlutterDesktop::create_window_with_icu`.

### AOT (release / profile) — built from source

AOT engines are *not* published for the generic embedder. Build one matched to
your SDK:

```bash
scripts/build-engine.sh --mode release --out /path/to/your/app/.engine-release
```

The script detects the SDK version, checks out `flutter/flutter` at the matching
commit, runs `gclient sync` + `gn` + `ninja`, and copies the library out. Run
`scripts/build-engine.sh --help` for the options. Expect hours and tens of GB on
the first run.

### Telling the toolchain where it is

- **Link time:** set `FLUTTER_ENGINE_LIB_DIR=<dir>` (flutter-engine-sys's build
  script turns it into a `-L`), or pass `-L <dir>` yourself via `RUSTFLAGS`.
  The two are interchangeable and compose.
- **Run time:** put the same directory on `LD_LIBRARY_PATH` (Linux),
  `DYLD_LIBRARY_PATH` (macOS) or `PATH` (Windows).

## Writing an app

The Dart half is an ordinary Flutter project; build its assets with
`flutter build bundle`. The Rust half creates one engine and any number of
windows:

```rust
use flutter_glfw::{AppArgs, WindowArgs, WindowMode};
use std::path::PathBuf;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // The thread that calls `init` becomes the engine's platform thread.
    let mut desktop = flutter_glfw::init()?;

    let mut app_args = AppArgs::new(PathBuf::from("build/flutter_assets"));
    app_args.icu_data_path = Some(PathBuf::from("icudtl.dat")); // from the SDK
    let app = desktop.create_app(app_args)?;

    // The first window owns the implicit view (0), so `runApp` renders into it.
    let _window = app.create_window(&WindowArgs {
        width: 1280,
        height: 720,
        title: "My app",
        mode: WindowMode::Windowed,
    })?;

    app.run(None, None)?; // owns this thread until the app exits
    app.shutdown();
    Ok(())
}
```

Channels and plugins are registered on the app, not a window:
`app.register_channel(...)`, `app.add_plugin(...)`, `app.create_texture()`.
Windows created *after* `run` get new engine views; `app.create_window` is safe
to call from platform-thread callbacks.

glfw-rs 0.63 leaks windows created with `create_shared`; this repository vendors
a one-line fixed copy in `vendor/glfw` and applies it with
`[patch.crates-io]`. A downstream app must add the same patch (or a fixed
glfw-rs) or its visible windows will never be destroyed:

```toml
[patch.crates-io]
glfw = { path = "path/to/flutter-rs/vendor/glfw" }
```

### Multi-window

`examples/multiple_windows` runs Flutter's official reference app. It adds
`lib/main_glfw.dart`, which enables the framework's experimental windowing flag
at runtime and installs `GlfwWindowingOwner` from `flutter-glfw/dart`. The Dart
owner talks to the Rust backend through a function-pointer table whose address
travels as `PlatformDispatcher.engineId`, so no C symbols have to be exported.

```bash
cd examples/multiple_windows
./run.sh
# macOS (the engine has no OpenGL backend there):
FLUTTER_ROOT=/path/to/flutter-3.50 ./run_macos.sh
```

The example creates its windows from Dart; the Rust runner only starts the
engine. Under the hood this is the embedder C API described above: the Dart
side drives the framework's own windowing interfaces, and the backend adds one
engine view per native window through `AddView`/`RemoveView`. The compositor
keeps one render target per view and presents each into its own GLFW window.
Window management is best-effort on native Wayland (top-level positioning is
unavailable); run a Wayland session through XWayland
(`FLUTTER_GLFW_PLATFORM=x11`) for tooltips, popups and satellites.

Linux and macOS are verified end-to-end (see the matrix above); Windows shares
the same backend and ABI and is exercised in a Win11 VM. Windows needs a native
build (or a mingw toolchain with the dialog feature disabled), and child windows
get a Win32 owner (`GWLP_HWNDPARENT`, the equivalent of AppKit's
`addChildWindow:ordered:` and X11's `WM_TRANSIENT_FOR`) so tooltips, popups,
dialogs and satellites stay directly above their parent. macOS was built and
run on Apple Silicon: its engine has no OpenGL backend. The default there is
Metal with Impeller: the engine renders into `MTLTexture` backing stores that the compositor blits into each window's
`CAMetalLayer`. `FLUTTER_GLFW_RENDERER=software` forces the CPU renderer, whose
pixels the compositor uploads through the window's GL context, and passing
`--enable-impeller=false` (through `AppArgs.engine_args`) keeps Skia on the
Metal path. The
generic macOS embedder is a universal `FlutterEmbedder.framework`; a downstream
binary still needs `DYLD_FRAMEWORK_PATH` pointing at the directory that holds
it (the rpath emitted by `flutter-engine-sys` only covers that crate's own
targets).

Parented dialogs are transient/modal on X11 (`WM_TRANSIENT_FOR` +
`_NET_WM_STATE_MODAL`), and the backend also drops parent input while any modal
child exists, so the parent stays blocked on WMs without modality support.
Satellites keep an absolute offset from their parent and are floating, so fast
parent moves stay tight and the satellite stays above the parent.

[flutter-app-template](https://github.com/flutter-rs/flutter-app-template) is a
worked example of the project layout, but ignore its `cargo flutter` commands.

Hot reload and distribution packaging (appimage / snap / nsis) came from
`cargo-flutter` and are **currently unsupported**.

## Building this repository

```bash
cargo build --workspace
cargo test --workspace     # needs an engine library, see above
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full check list that CI enforces.

## Contribution

To contribute to flutter-rs, please see [CONTRIBUTING](CONTRIBUTING.md).

## ChangeLog

[CHANGELOG](CHANGELOG.md).

## License

MIT — see [LICENSE](LICENSE).

[flutter-badge]: https://img.shields.io/badge/flutter-v3.50%20--%20v3.51-blueviolet.svg
[flutter-url]: https://flutter.dev/
[crates-badge]: https://img.shields.io/crates/v/flutter-engine.svg
[crates-url]: https://crates.io/crates/flutter-engine
[mit-badge]: https://img.shields.io/badge/license-MIT-blue.svg
[mit-url]: LICENSE
[flutter-app-template]: https://user-images.githubusercontent.com/741807/72476798-5a99e280-37ee-11ea-9e08-b0175ae21ad6.png
