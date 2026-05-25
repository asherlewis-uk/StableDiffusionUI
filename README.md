# StableDiffusionUI

A small SwiftUI macOS app that wraps Apple's Core ML Stable Diffusion pipeline to generate images from text prompts on Apple Silicon.

This README aims to be honest about what the project is and what it is not. It is a personal/hobby project, not a polished product.

## What it actually does

- Provides a single-window macOS app with two screens:
  - **Text 2 Image**: enter a prompt, adjust steps and guidance scale, optionally enter a seed, and generate an image.
  - **History**: a list of previously generated images, persisted with Core Data. Selecting one shows its prompt/seed/steps/guidance and the saved image (read-only).
- Runs inference via the [`StableDiffusion`](https://github.com/Wanaldino/ml-stable-diffusion) Swift package (a fork of Apple's `ml-stable-diffusion`), pinned in `Package.resolved` on the `develop` branch.
- Saves generated images as JPEGs to `~/Pictures/Stable Diffusion Results/<uuid>.jpeg` and stores metadata (prompt, seed, steps, guidance scale, file URL) in a Core Data store.

That is the entire feature set. There is no model downloader, no model picker, no settings screen, no image-to-image, no inpainting, no upscaling, no batch export, and no sharing UI.

## Requirements

- macOS 13.0 or later (the deployment target set in the Xcode project).
- Xcode 14+ to build.
- Apple Silicon strongly recommended — the pipeline is configured with `computeUnits = .cpuAndNeuralEngine`.
- A set of Core ML Stable Diffusion model resources (the `.mlmodelc` bundles produced by Apple's `python_coreml_stable_diffusion` converter, plus the tokenizer files). These are **not** included in this repository and the app does not download them for you.

## How model loading works (important)

The pipeline is initialized with:

```swift
let url = Bundle.main.resourceURL!
pipeline = try? StableDiffusionPipeline(resourcesAt: url, configuration: configuration, disableSafety: true)
```

This means the app expects the Core ML resources to be present **inside the built app bundle's `Resources` directory**. To actually generate images you must add the converted model files to the Xcode project (as bundle resources) before building. If the resources aren't found, `pipeline` is silently `nil` and the Generate button will appear to do nothing.

There is no in-app error message for a missing or failed pipeline — failures are swallowed by `try?`.

## Known limitations and rough edges

These are real characteristics of the current code, not aspirational TODOs:

- **The safety checker is disabled** (`disableSafety: true` in `ContentView.swift`). The app will not filter NSFW or otherwise objectionable outputs.
- **The "Image count" slider is disabled in the UI** (`.disabled(true)`). Only one image per generation is actually produced, regardless of the slider value.
- **No error handling surfaced to the user.** Failures during pipeline init, image generation, file writes, and Core Data saves are silently ignored via `try?`.
- **Force-unwraps and `fatalError`** exist on the hot path (e.g. `Bundle.main.resourceURL!`, and a `fatalError()` in the progress handler if a frame is missing). The app can crash on unexpected pipeline state.
- **Directory existence check is buggy**: the output directory is checked with `FileManager.default.fileExists(atPath: directoryURL.absoluteString)`, which uses a `file://` URL string instead of a filesystem path. The `createDirectory` call still runs with `withIntermediateDirectories: true`, so this usually works in practice but the check itself does nothing useful.
- **History entries are read-only** but the editor UI is reused with `.disabled(true)`; there is no delete action.
- **Tests are stubs.** `StableDiffusionUITests` and the UI tests are the default Xcode-generated templates with no meaningful assertions.
- **Sandbox/entitlements** are whatever is in `StableDiffusionUI.entitlements`; writing to `~/Pictures/Stable Diffusion Results` assumes the app has access to that location.
- The package dependency points at a personal fork on a moving `develop` branch, so reproducibility depends on `Package.resolved`.

## Building

1. Open `StableDiffusionUI.xcodeproj` in Xcode.
2. Let Swift Package Manager resolve `ml-stable-diffusion` and `swift-argument-parser`.
3. Add your converted Core ML Stable Diffusion resources to the app target so they are copied into the app bundle's `Resources` directory.
4. Select a development team for code signing if needed and run on macOS 13+.

## Project layout

```
StableDiffusionUI/
├── StableDiffusionUIApp.swift   # @main entry point, wires up Core Data
├── ContentView.swift            # Root navigation + pipeline init + history list
├── Text2Image.swift             # Generation screen (prompt, sliders, generate)
├── ListItem.swift               # Row view for the history list
├── Data/
│   ├── Persistence.swift        # Core Data stack
│   ├── ImageDAO.swift           # Core Data entity helpers
│   └── StableDiffusionUI.xcdatamodeld
└── Assets.xcassets, Preview Content/, *.entitlements
```

## License

MIT — see [`LICENSE`](LICENSE). Original copyright © 2022 Wanaldino Antimonio.

## Credits

- Apple's [`ml-stable-diffusion`](https://github.com/apple/ml-stable-diffusion) (used via the [`Wanaldino/ml-stable-diffusion`](https://github.com/Wanaldino/ml-stable-diffusion) fork).
- Stable Diffusion model weights are the property of their respective authors and are governed by their own licenses; this repository does not ship any model weights.
