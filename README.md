# OwnVocalRemover

A high-performance audio source separation library for .NET that uses ONNX models to separate vocals, instrumentals and individual stems from mixed audio files.
Built on top of [OwnAudioSharp](https://www.nuget.org/packages/OwnAudioSharp) for all audio I/O, decoding and DSP.

I've been looking for a good vocal and music separation code in C# for a very long time that provides decent quality.
Unfortunately, I could only find such code in Python, so I decided to create a pure C# vocal separator that would deliver the quality created by the Python code!

## Features

- **Three separators** in one package:
  - `SimpleAudioSeparationService` – fast vocals / instrumental split (MDX-Net style models)
  - `HTDemucsAudioSeparator` – 4-stem separation (vocals, drums, bass, other)
  - `MultiModelAudioSeparator` – run several models and average their outputs for higher quality
- **On-demand model download**: models are fetched from HuggingFace on first use and cached per user – nothing is embedded in the package
- **GPU acceleration**: CoreML on macOS, CUDA on Windows/Linux, with automatic CPU fallback
- **Streaming, low-memory processing**: large files are handled chunk by chunk
- **Noise reduction**: optional denoising for improved separation quality
- **Auto-configuration**: model parameters (FFT / frequency / time dimensions) are detected from the ONNX metadata
- **Progress tracking**: real-time progress reporting through events
- **Custom models**: any compatible MDX-Net / HTDemucs ONNX model can be loaded from disk

## Installation

```
dotnet add package OwnVocalRemover
```

Target framework: **.NET 10**. The `OwnAudioSharp` and `Microsoft.ML.OnnxRuntime` dependencies are pulled in automatically.

## Dependencies

- `OwnAudioSharp` – audio decoding, DSP (FFT), buffer management and WAV writing
- `Microsoft.ML.OnnxRuntime` – ONNX model inference

## Models

Models are **not** bundled with the package. On first use they are downloaded from HuggingFace and cached in a per-user directory (`VocalRemoverModelManager.DefaultModelsDirectory`, by default `LocalApplicationData/OwnAudio/models`).

<div align="center">
  <a href="https://huggingface.co/ModernMube/HTDemucs_onnx/tree/main">
    <img src="https://img.shields.io/badge/Download_Models-OwnAudioSharp_Vocal_Remove_models-red?style=for-the-badge" alt="Download" width="400">
  </a>
</div>

Available models (`InternalModel` enum):

| Model | Purpose |
|-------|---------|
| `Default` | General purpose vocals / instrumental split, fastest |
| `Best` | Higher quality vocals / instrumental split |
| `Karaoke` | Lead-vocal removal, keeps backing vocals |
| `HTDemucs` | 4-stem separation (vocals / drums / bass / other) |

### Downloading a model

You can let the separator download the model automatically, or manage it explicitly:

```csharp
using OwnVocalRemover;

const InternalModel model = InternalModel.Default;

if (!VocalRemoverModelManager.IsModelAvailable(model))
{
    await VocalRemoverModelManager.DownloadModelAsync(
        model,
        new Progress<ModelDownloadProgress>(p =>
        {
            string pct = p.Percentage >= 0 ? $"{p.Percentage:F1}%" : "?%";
            Console.Write($"\rDownloading: {pct} ({p.BytesDownloaded / 1024 / 1024} MB)");
        }));
}

// Change the storage location (call once at startup, before any separator is initialized)
// VocalRemoverModelManager.DefaultModelsDirectory = "/my/models";

// Ensure several models at once
await VocalRemoverModelManager.EnsureModelsAvailableAsync(
    new[] { InternalModel.Best, InternalModel.Karaoke });
```

## Support My Work

If you find this project helpful, consider buying me a coffee!

<a href="https://www.buymeacoffee.com/ModernMube"
    target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/arial-yellow.png"
    alt="Buy Me A Coffee"
    style="height: 60px !important;width: 217px !important;" >
 </a>

## Quick Start – Vocals / Instrumental

The simplest way to split a track is the `SimpleSeparator` helper, which creates and initializes the service for you:

```csharp
using OwnVocalRemover;
using static OwnVocalRemover.SimpleSeparator;

const InternalModel model = InternalModel.Default;

// Make sure the model is present (downloads on first run)
if (!VocalRemoverModelManager.IsModelAvailable(model))
    await VocalRemoverModelManager.DownloadModelAsync(model);

// Create + initialize the separator
var (service, _, _) = Separator(model, outputDirectory: "output");

if (service != null)
{
    service.ProgressChanged += (s, p) =>
        Console.WriteLine($"{p.Status}: {p.OverallProgress:F1}%");

    var result = service.Separate("song.mp3");

    Console.WriteLine($"Vocals:       {result.VocalsPath}");
    Console.WriteLine($"Instrumental: {result.InstrumentalPath}");

    service.Dispose();
}
```

Or configure it manually:

```csharp
var options = new SimpleSeparationOptions
{
    Model = InternalModel.Best,       // or ModelPath = "models/custom.onnx"
    OutputDirectory = "output",
    EnableGPU = true,
    DisableNoiseReduction = false,
    ChunkSizeSeconds = 15,
    Margin = 44100
};

using var service = new SimpleAudioSeparationService(options);
service.Initialize();
var result = service.Separate("song.wav");
```

### SimpleSeparationOptions

| Property | Default | Description |
|----------|---------|-------------|
| `ModelPath` | `null` | Path to a custom ONNX model (takes priority over `Model`) |
| `Model` | `Best` | Built-in model to use |
| `OutputDirectory` | `separated` | Where result files are written |
| `EnableGPU` | `true` | Use CoreML (macOS) / CUDA (other) with CPU fallback |
| `DisableNoiseReduction` | `false` | Turn off denoising for faster processing |
| `Margin` | `44100` | Overlap margin between chunks (samples) |
| `ChunkSizeSeconds` | `15` | Chunk length in seconds (0 = whole file at once) |
| `NFft` / `DimT` / `DimF` | `6144` / `8` / `2048` | STFT parameters (auto-detected from the model) |

Output files: `{filename}_vocals.wav` and `{filename}_music.wav`.

## HTDemucs – 4-Stem Separation

Splits a track into **vocals, drums, bass and other**.

```csharp
using OwnVocalRemover;

const InternalModel model = InternalModel.HTDemucs;

if (!VocalRemoverModelManager.IsModelAvailable(model))
    await VocalRemoverModelManager.DownloadModelAsync(model);

var options = new HTDemucsSeparationOptions
{
    Model = model,                    // or ModelPath = "models/htdemucs.onnx"
    OutputDirectory = "output",
    ChunkSizeSeconds = 10,
    OverlapFactor = 0.25f,            // 25% overlap between chunks
    EnableGPU = true,
    TargetStems = HTDemucsStem.All    // or e.g. Vocals | Other
};

using var separator = new HTDemucsAudioSeparator(options);

separator.ProgressChanged += (s, p) =>
    Console.Write($"\r{p.Status}: {p.OverallProgress:F1}%");

separator.Initialize();
var result = separator.Separate("song.mp3");

foreach (var stem in result.StemPaths)
    Console.WriteLine($"{stem.Key}: {stem.Value}");
```

Helper factory methods are available via `HTDemucsExtensions`:

```csharp
var s1 = HTDemucsExtensions.CreateAllStems("output");
var s2 = HTDemucsExtensions.CreateVocalsOnly("output");
var s3 = HTDemucsExtensions.CreateStemSelector(HTDemucsStem.Vocals | HTDemucsStem.Other, "output");
var s4 = HTDemucsExtensions.CreateFromFile("models/htdemucs.onnx", "output");
```

### HTDemucsSeparationOptions (key settings)

| Property | Default | Description |
|----------|---------|-------------|
| `Model` / `ModelPath` | `None` / `null` | Built-in `HTDemucs` model or a custom ONNX file |
| `OutputDirectory` | `separated_htdemucs` | Output folder |
| `ChunkSizeSeconds` | `10` | Chunk length (10–30 s recommended) |
| `OverlapFactor` | `0.25` | Overlap between chunks (0.0–0.5) |
| `TargetStems` | `All` | Which stems to extract (`[Flags]`) |
| `EnableGPU` | `true` | GPU acceleration with CPU fallback |
| `MarginSeconds` / `CrossfadeSeconds` | `0.5` / `0.05` | Edge trim and crossfade for clean overlap-add |

Output files: one WAV per stem, e.g. `{filename}_vocals.wav`, `{filename}_drums.wav`, `{filename}_bass.wav`, `{filename}_other.wav`.

## Multi-Model – Averaged Separation

Runs several models on the original audio and averages their vocal and instrumental outputs for reduced artifacts and higher quality.

```csharp
using OwnVocalRemover;

await VocalRemoverModelManager.EnsureModelsAvailableAsync(
    new[] { InternalModel.Best, InternalModel.Karaoke });

// Simple 2-model averaging
var separator = MultiModelExtensions.CreateSimplePipeline(
    model1: InternalModel.Best,
    model2: InternalModel.Karaoke,
    outputDirectory: "output");

separator.ProgressChanged += (s, p) =>
    Console.Write($"\r[{p.CurrentModelIndex}/{p.TotalModels}: {p.CurrentModelName}] {p.OverallProgress:F1}%");

separator.Initialize();
var result = separator.Separate("song.mp3");

Console.WriteLine($"Vocals:       {result.VocalsPath}");
Console.WriteLine($"Instrumental: {result.InstrumentalPath}");
Console.WriteLine($"Models used:  {result.ModelsProcessed}");

separator.Dispose();
```

For full control, configure each model explicitly – including its `OutputType`
(needed when mixing vocal-output and instrumental-output models):

```csharp
var options = new MultiModelSeparationOptions
{
    Models = new List<MultiModelInfo>
    {
        new MultiModelInfo
        {
            Name = "VocalModel",
            ModelPath = "models/Voc_FT.onnx",
            OutputType = ModelOutputType.Vocals
        },
        new MultiModelInfo
        {
            Name = "InstrumentalModel",
            ModelPath = "models/Inst_HQ_3.onnx",
            OutputType = ModelOutputType.Instrumental
        }
    },
    OutputDirectory = "output",
    EnableGPU = true,
    ChunkSizeSeconds = 15,
    Margin = 44100,
    SaveAllIntermediateResults = true   // also writes each model's individual output
};

using var separator = new MultiModelAudioSeparator(options);
separator.Initialize();
var result = separator.Separate("song.mp3");
```

`CreateTriplePipeline(model1, model2, model3, ...)` is also available for 3-model averaging.

## GPU Acceleration

When `EnableGPU = true` (the default) the separators try, in order:

- **macOS**: CoreML (MLProgram, then NeuralNetwork format) → CPU fallback
- **Windows / Linux**: CUDA → CPU fallback

Set `EnableGPU = false` to force CPU processing.

## Supported Audio Formats

- WAV (`.wav`)
- MP3 (`.mp3`)
- FLAC (`.flac`)

Decoding is handled by OwnAudioSharp; audio is processed as 44.1 kHz stereo.

## Custom Models

Any compatible MDX-Net style ONNX model can be used by setting `ModelPath`
instead of `Model`. Model parameters (input shape `[batch, 4, frequency, time]`)
are auto-detected from the ONNX metadata. For the multi-model pipeline, the
output type is auto-detected from the filename (`Voc` → vocals, `Inst` → instrumental)
unless you set `OutputType` explicitly.

## Error Handling

```csharp
try
{
    var result = service.Separate("song.wav");
}
catch (FileNotFoundException ex)
{
    // Missing input file OR a model that has not been downloaded yet
    Console.WriteLine($"Not found: {ex.Message}");
}
catch (InvalidOperationException ex)
{
    // Service used before Initialize(), or no model configured
    Console.WriteLine($"Invalid operation: {ex.Message}");
}
```

## Disposal

All separators implement `IDisposable` and release their ONNX sessions on dispose.
Prefer `using` or call `Dispose()` explicitly:

```csharp
using var separator = new HTDemucsAudioSeparator(options);
separator.Initialize();
// ...
```

## Examples

The repository contains three runnable console examples:

- `Example.Simplified` – vocals / instrumental split
- `Example.HTDemucs` – 4-stem separation
- `Example.Multimodel` – multi-model averaging pipelines
