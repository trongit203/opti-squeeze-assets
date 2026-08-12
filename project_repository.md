# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

OptiSqueeze is a single-module Android app (`:app`) that compresses, converts, and edits images and PDFs on-device and saves the results to the public `Downloads/OptiSqueeze` folder. Package `com.apollo.optisqueeze`, `minSdk 24`, `compileSdk`/`targetSdk 36`. UI is Jetpack Compose; DI is Hilt; revenue is AdMob native + interstitial ads; monitoring is Firebase Analytics + Crashlytics + Performance.

## Build & test

```bash
./gradlew assembleDebug          # build the debug APK
./gradlew installDebug           # build + install to a connected device/emulator
./gradlew testDebugUnitTest      # JVM unit tests (app/src/test)
./gradlew connectedDebugAndroidTest   # instrumented tests (needs a device/emulator)
./gradlew lint                   # Android lint

# Run a single unit test
./gradlew testDebugUnitTest --tests "com.apollo.optisqueeze.ExampleUnitTest"
```

There is no app-specific CI, lint baseline, or custom Gradle task — these are the stock AGP tasks.

## Toolchain constraints (do not bump blindly)

- **AGP is pinned to 8.13** (`gradle/libs.versions.toml`). Hilt's Gradle plugin breaks on AGP 9 — do not upgrade AGP past 8.x without resolving that first.
- Java/Kotlin target is **11**. Kotlin 2.2.x with KSP (Hilt uses KSP, not kapt).
- All dependency versions live in `gradle/libs.versions.toml` (version catalog). Add/upgrade deps there, not inline in `build.gradle.kts`.
- `release` build type has `isMinifyEnabled = false` — ProGuard rules exist but are not exercised.
- `androidx.exifinterface` is currently declared **twice** in the catalog under two different aliases/versions (`androidx-exifinterface` → 1.3.6, `exifinterface` → 1.4.2), and `app/build.gradle.kts` applies both. Consolidate to one alias/version rather than adding a third when touching this dependency.

## Architecture

Clean Architecture with MVVM, three layers under `app/src/main/java/com/apollo/optisqueeze/`:

- **domain/** — pure Kotlin. `model/` data classes + enums, `repository/` interfaces, `usecase/UseCases.kt` (one small `@Inject`-able use case class per operation, invoked via `operator fun invoke`). No Android dependencies here.
- **data/** — implements the domain repository interfaces (`data/repository/*Impl`). Splits into `compressor/` (the actual byte-crunching for every tool, not just compression), `local/` (SharedPreferences-backed stores), and `source/` (file metadata resolution, Downloads persistence, original-file replacement).
- **presentation/** — Compose UI + `@HiltViewModel` ViewModels. `home/` holds the Tool Hub and every tool screen; `compression/`, `files/`, `settings/` are the other features. `OptiSqueezeApp.kt` is the root: a single Scaffold with a 3-tab bottom bar (Home/Files/Settings) **plus** a `CompressionPhase` overlay (IDLE → COMPRESSING → RESULTS) that takes over the whole screen during any tool run (compression or otherwise — the overlay is shared by all tools).

DI wiring: `OptiSqueezeApplication` is `@HiltAndroidApp`, `MainActivity` is `@AndroidEntryPoint`. `di/RepositoryModule.kt` `@Binds` **four** repository impls as `@Singleton` (Compression, History, Settings, Analytics). Everything else is constructor-injected (compressors, stores, sources take `@ApplicationContext` directly).

### Tool Hub: one ViewModel drives every tool, not just compression

`presentation/home/Tool.kt` defines `ToolSet` (`IMAGE`/`PDF`) and a `Tool` enum with ~21 entries — Compress, Merge, Split, Extract, Lock/Unlock, Rotate, Rearrange, Image↔PDF, Resize, Crop, Convert (HEIC/PNG⇄JPG/WEBP/generic), Strip metadata, Rename, Batch rename. `Tool.forSet(set)` feeds the `ToolHubScreen` grid; `Tool.compressFor(set)` is the entry point launcher shortcuts and share-sheet arrivals route to.

There is **no per-tool ViewModel**. `OptiSqueezeApp.kt` holds `activeTool: Tool?` and dispatches to one of ~17 screen composables in `presentation/home/` (`MergeScreen`, `SplitScreen`, `SecureScreen`, `RearrangeScreen`, `ExtractScreen`, `RotateScreen`, `ImageToPdfScreen`, `PdfToImageScreen`, `ResizeScreen`, `RotateImageScreen`, `StripMetadataScreen`, `RenameScreen`, `BatchRenameScreen`, `CropScreen`, `ConvertScreen`, `CompressScreen`, `ToolHubScreen`) — every one of them takes the same shared `CompressionUiState` plus tool-specific callback lambdas bound to methods on the single `CompressionViewModel`. A `LaunchedEffect(activeTool)` in `OptiSqueezeApp.kt` re-derives the session's mode on every tool switch by calling `setImageOutputFormat`/`setMergeMode`/`setSplitMode`/`setSecurityMode`/`setTransformKind`/`setImageEditKind` on the ViewModel — these all get set together from one place, so exactly one mode (or none, in ordinary compression) is ever active at a time. `CompressionUiState` carries all of these mode flags plus the original compression fields; treat it as shared session state for whichever tool is currently active, not compression-specific state.

When adding a new tool: add a `Tool` entry, a domain model for its parameters if needed, a `data/compressor/` class that does the work, a use case wrapping the repository method, a `CompressionRepository`/`Impl` method, a `start*`/`run*` method on `CompressionViewModel`, a screen in `presentation/home/`, and a branch in `OptiSqueezeApp.kt`'s tool dispatch + `LaunchedEffect(activeTool)` mode setters. This is more files than it sounds like, but each one is small and mirrors an existing tool of the same shape (1→1, 1→N, N→1, or N→N — see the doc comments on `CompressionViewModel`'s `start*` methods for which existing tool to copy).

### Compression flow (the original tool, still the default)

1. User picks files → `CompressionViewModel.addFiles(uris)` → `ResolveFilesUseCase` → `CompressionRepository.resolve` turns content URIs into `SourceFile`s (name/size/kind/detail).
2. `startCompression()` launches a coroutine that processes selected files **sequentially**, calling `CompressFileUseCase` per file. The repository returns a `Flow<CompressionUpdate>` (Progress/Success/Failure) running on `Dispatchers.Default`; the ViewModel maps each update into per-item `CompressionItem` state. Other tools follow the same Progress/Success/Failure streaming shape (`SplitUpdate` for the 1→N tools: Split, PDF-to-image), just fanned into 1, N, or a single synthetic item depending on the tool's shape.
3. On completion it persists `HistoryRecord`s and flips phase to RESULTS. `saveAll()` writes compressed cache files to shared storage via `SaveCompressedFileUseCase` → `DownloadsSaver` (MediaStore on API 29+, legacy file path below that). When `keepOriginals = false`, `ReplaceOriginalFileUseCase` → `OriginalFileReplacer` overwrites the source URI in place; on failure it falls back to `DownloadsSaver` automatically. Any tool whose output can't safely replace a same-format original in place (merge, split, convert, and generally anything where `CompressionUiState.isTransform` is true) forces `keepOriginals = true` for that save.

All compressors/tool classes write to **cache files** (`CompressedFileStorage.newCacheFile`); nothing touches public storage until the user taps Save. `CompressionRepositoryImpl` wraps every tool method uniformly: `catch (CancellationException) { throw it }` → `catch (OutOfMemoryError)` → `catch (Throwable)`, plus tool-specific catches where meaningful (`IllegalArgumentException` for bad page-range input on Split/Extract, `InvalidPasswordException` for Lock/Unlock). The `data/compressor/` classes themselves mostly don't catch — they rely on the repository's wrapping — except where a class needs to react to a specific failure internally (PDF image-to-PDF, PDF security).

### Quality presets

`CompressionQuality` (MAXIMUM / BALANCED / LOSSLESS) is the single knob. Its enum fields (`jpegQuality`, `maxImageDimension`, `pdfRenderDpi`, `lossless`) are the parameters the compressors read. PDF-specific JPEG qualities are tuned separately inside `PdfCompressor`. Default is BALANCED.

`CompressionSettings` holds the full per-session configuration: `quality`, `keepOriginals` (default `true`), `keepTextSelectable` (default `true`), plus the per-tool parameter fields tools write into before invoking their `start*`/`run*` method — `imageOutputFormat` (convert), `rotationDegrees`, `resize` (`ResizeSpec`: `LongEdge` or `Exact`), `cropRect` (`NormalizedRect`), `stripMetadata`. The ViewModel seeds session settings from `SettingsLocalStore` on startup but does not write back automatically — per-session changes are scoped to the session. Entering a non-format image-edit tool resets these param fields to defaults so a stale angle/resize/crop never leaks from a previous tool.

### PdfCompressor — read the class doc before editing

`PdfCompressor` has **two independent strategies**, selected by `CompressionSettings.keepTextSelectable`:

**1. Text-preserving (default, `keepTextSelectable = true`)** — uses PDFBox Android to parse the document structure and recompress only the *embedded image XObjects*. Text, fonts, and vectors are left intact so the output stays selectable, searchable, and accessible. Additional passes strip non-rendering metadata (`/Info`, XMP, `/Thumb`, `/PieceInfo`), deduplicate font file streams and ToUnicode CMaps that many generators embed once per page, and re-deflate FlateDecode streams at `Deflater.BEST_COMPRESSION` (level 9). Identical images are deduped by MD5 content hash so they are re-encoded only once. The reason this strategy exists: `android.graphics.pdf.PdfDocument` stores pages as Flate-compressed raw pixels, so using it would produce output as large as or larger than the input — JPEG re-encoding of embedded images is the only lever that actually shrinks document-heavy PDFs without destroying text.

**2. Flatten / smallest (`keepTextSelectable = false`)** — rasterises each page with `PdfRenderer` at a DPI determined by the quality preset (96/150/200 DPI) and re-embeds it as one full-page **DCTDecode (JPEG) XObject** in a hand-written PDF (`PdfWriter` builds objects/xref by hand). Produces the absolute smallest file; trade-off is that text is no longer selectable.

Both strategies share the same safety net: if the rebuilt PDF is not actually smaller than the original, the original bytes are copied back so we never return a larger file. The same pattern (cache-file output, never-larger guarantee) applies to `ImageCompressor` as well. **This never-larger guarantee is specific to Compress** — the other `data/compressor/` tool classes (`PdfMerger`, `PdfSplitter`, `PdfSecurity`, `PdfRearrange`, `PdfExtract`, `PdfRotate`, `PdfImageToPdf`, `PdfToImage`, `FilePassthrough`) don't have (and shouldn't have) an equivalent, since shrinking isn't their job — their correctness bar is "never report success on a zero-byte or partially-written file," which they enforce with a `length() > 0` check before returning, plus (as of the most recent hardening pass) cleaning up any partial/empty cache file on the failure path too.

PNG-with-alpha input converting to a JPEG output (Convert tool, or ordinary compression choosing JPEG) is flattened onto an opaque white background first — JPEG has no alpha channel, and compositing must happen before `Bitmap.compress`, not be skipped.

### Persistence

No database. History and settings are JSON/primitives in **SharedPreferences** (`HistoryLocalStore` caps at 500 records newest-first; `SettingsLocalStore`). Stores expose `StateFlow` so repositories/use cases observe reactively.

### Analytics

`AnalyticsRepository` (domain interface) / `AnalyticsRepositoryImpl` (data layer, wraps `FirebaseAnalytics`) is injected directly into `CompressionViewModel` and `AppAnalyticsViewModel`. Key events include `compression_started`/`compression_completed` (with `saved_percent` and `all_already_optimal`), `files_added`, `retry_stronger`, `files_saved`, `share_completed`, plus a `*_started`/`*_completed` pair per tool (`merge_started`, `split_completed`, `pdf_secure_started`, `rearrange_completed`, etc. — follow the existing naming when adding a tool). Screen views are deduplicated and logged by `AppAnalyticsViewModel.logScreenView()`, keyed off both `selectedTab` and `activeTool` so each tool screen gets its own screen-view name. Firebase Crashlytics and Firebase Performance are included in the dependency graph but have no manual instrumentation beyond automatic traces.

### Ads & in-app review

`NativeAdLoader` wraps AdMob's `AdLoader`; all failures are swallowed to a `null` ad so the UI falls back to a placeholder and never crashes offline. Two interstitial slots (`InterstitialAdSlot.SAVE_ALL`, `SHARE_COMPLETED`) are preloaded only during the RESULTS phase and shown via `InterstitialAdController`/`rememberInterstitialAd` in `OptiSqueezeApp.kt` before running the triggering action. The in-app review prompt (`ReviewManagerFactory`) is triggered after `shareCompleted()` — the user completing a share is the strongest satisfaction signal; the share-completion signal itself comes from a broadcast fired when the user picks a target in the system share sheet (`ResultsScreen.kt`), not from an activity result, since most share targets never call `setResult()`. **The AdMob app ID in `AndroidManifest.xml` and the ad unit IDs are Google's public sample/test IDs — replace before any real release.**

### Share intent & app shortcuts

`MainActivity.kt` has `ACTION_SEND`/`ACTION_SEND_MULTIPLE` intent filters (for `application/pdf` and `image/*`) and two launcher shortcuts (`compress_pdf`, `compress_image`, defined in `shortcuts.xml`). Both paths ultimately call `compressionVm.addFiles(uris)` and set a `PickerRequest(toolSet)`; `OptiSqueezeApp.kt` converts that into `activeTool = Tool.compressFor(toolSet)`, landing the user on the ordinary `CompressScreen` for that file type — shortcuts and shared files always go through Compress, never directly into one of the other tools.
