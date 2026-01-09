# UAssetAPI.Lightweight - API Reference

A comprehensive guide to using the lightweight UAssetAPI for JSON/UAsset conversion and batch processing.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Core Classes](#core-classes)
  - [UAsset](#uasset)
  - [Usmap](#usmap)
- [JSON Serialization](#json-serialization)
  - [UAsset to JSON](#uasset-to-json)
  - [JSON to UAsset](#json-to-uasset)
- [Batch Processing](#batch-processing)
  - [BatchProcessor](#batchprocessor)
  - [BatchProcessOptions](#batchprocessoptions)
  - [BatchProcessResult](#batchprocessresult)
- [Engine Versions](#engine-versions)
- [Complete Examples](#complete-examples)

---

## Quick Start

```csharp
using UAssetAPI;
using UAssetAPI.UnrealTypes;
using UAssetAPI.Unversioned;

// Load mappings for unversioned assets
var mappings = new Usmap("path/to/game.usmap");

// Read asset
var asset = new UAsset("path/to/asset.uasset", EngineVersion.VER_UE5_3, mappings);

// Convert to JSON
string json = asset.SerializeJson(isFormatted: true);
File.WriteAllText("output.json", json);

// Convert back to UAsset
var rebuilt = UAsset.DeserializeJson(json);
rebuilt.Write("output.uasset");
```

---

## Core Classes

### UAsset

The main class for reading, writing, and manipulating Unreal Engine assets.

#### Constructors

```csharp
// From file path with engine version and optional mappings
UAsset(string path, EngineVersion engineVersion, Usmap mappings = null, 
       CustomSerializationFlags flags = CustomSerializationFlags.None)

// From file path, controlling whether to load .uexp
UAsset(string path, bool loadUexp, EngineVersion engineVersion, Usmap mappings = null,
       CustomSerializationFlags flags = CustomSerializationFlags.None)

// From BinaryReader
UAsset(AssetBinaryReader reader, EngineVersion engineVersion, Usmap mappings = null,
       bool useSeparateBulkDataFiles = false, CustomSerializationFlags flags = CustomSerializationFlags.None)

// Empty constructor (for manual initialization)
UAsset(EngineVersion engineVersion = EngineVersion.UNKNOWN, Usmap mappings = null,
       CustomSerializationFlags flags = CustomSerializationFlags.None)
```

#### Key Properties

| Property | Type | Description |
|----------|------|-------------|
| `FilePath` | `string` | Path to the asset file on disk |
| `Exports` | `List<Export>` | List of exported objects in the asset |
| `Imports` | `List<Import>` | List of imported references |
| `Mappings` | `Usmap` | Usmap mappings for unversioned properties |
| `HasUnversionedProperties` | `bool` | Whether the asset uses unversioned properties |
| `UseSeparateBulkDataFiles` | `bool` | Whether .uexp/.ubulk files are used |

#### Key Methods

```csharp
// Write asset to file
void Write(string outputPath)

// Write to MemoryStream
MemoryStream WriteData()

// Get name map entries
List<string> GetNameMapIndexList()

// Verify binary equality after round-trip
bool VerifyBinaryEquality()
```

---

### Usmap

Handles loading .usmap mapping files for unversioned game assets.

#### Constructor

```csharp
// Load from file path
Usmap(string path)

// Load from stream
Usmap(Stream stream)
```

#### Key Properties

| Property | Type | Description |
|----------|------|-------------|
| `Schemas` | `Dictionary<string, UsmapSchema>` | All loaded struct/class schemas |
| `Enums` | `Dictionary<string, UsmapEnum>` | All loaded enum definitions |
| `NameMap` | `List<string>` | Name map from the usmap |

---

## JSON Serialization

### UAsset to JSON

Convert a UAsset to a JSON string.

```csharp
// Basic usage - formatted JSON
string json = asset.SerializeJson(isFormatted: true);

// Compact JSON (smaller file size)
string json = asset.SerializeJson(isFormatted: false);

// With Newtonsoft.Json formatting control
string json = asset.SerializeJson(Formatting.Indented);
```

#### Serialize Specific Objects

```csharp
// Serialize any object using the asset's JSON settings
string json = asset.SerializeJsonObject(someObject, isFormatted: true);
```

### JSON to UAsset

Convert a JSON string back to a UAsset.

```csharp
// From string
UAsset asset = UAsset.DeserializeJson(jsonString);

// From stream
using var stream = File.OpenRead("asset.json");
UAsset asset = UAsset.DeserializeJson(stream);
```

**Important:** After deserializing, set the mappings before writing if the asset uses unversioned properties:

```csharp
var asset = UAsset.DeserializeJson(json);
asset.Mappings = mappings;  // Required for unversioned assets
asset.Write("output.uasset");
```

#### Deserialize Specific Objects

```csharp
// Deserialize any object from JSON
MyType obj = asset.DeserializeJsonObject<MyType>(jsonString);
```

---

## Batch Processing

### BatchProcessor

Static class for parallel processing of multiple files.

#### UAsset to JSON (Batch)

```csharp
// Convert specific files
string[] files = { "asset1.uasset", "asset2.uasset", "asset3.uasset" };
BatchProcessResult[] results = BatchProcessor.UAssetToJson(
    inputPaths: files,
    outputDirectory: @"C:\Output",  // null = same directory as input
    options: options
);

// Async version
BatchProcessResult[] results = await BatchProcessor.UAssetToJsonAsync(files, outputDir, options);
```

#### JSON to UAsset (Batch)

```csharp
// Convert specific files
string[] jsonFiles = { "asset1.json", "asset2.json" };
BatchProcessResult[] results = BatchProcessor.JsonToUAsset(
    inputPaths: jsonFiles,
    outputDirectory: @"C:\Output",
    options: options
);

// Async version
BatchProcessResult[] results = await BatchProcessor.JsonToUAssetAsync(jsonFiles, outputDir, options);
```

#### Directory Conversion

```csharp
// Convert all .uasset files in a directory to JSON
BatchProcessResult[] results = BatchProcessor.ConvertDirectoryToJson(
    inputDirectory: @"C:\Game\Content",
    outputDirectory: @"C:\Output\Json",
    recursive: true,
    options: options
);

// Convert all .json files back to .uasset
BatchProcessResult[] results = BatchProcessor.ConvertDirectoryToUAsset(
    inputDirectory: @"C:\Output\Json",
    outputDirectory: @"C:\Output\Assets",
    recursive: true,
    options: options
);
```

#### File Discovery

```csharp
// Find all .uasset files
string[] uassetFiles = BatchProcessor.FindUAssetFiles(
    directory: @"C:\Game\Content",
    recursive: true
);

// Find all .json files
string[] jsonFiles = BatchProcessor.FindJsonFiles(
    directory: @"C:\Output",
    recursive: true
);
```

---

### BatchProcessOptions

Configuration for batch processing operations.

```csharp
var options = new BatchProcessOptions
{
    // Parallelism (default: Environment.ProcessorCount)
    MaxDegreeOfParallelism = 8,
    
    // Engine version for parsing
    EngineVersion = EngineVersion.VER_UE5_3,
    
    // Usmap mappings (required for unversioned assets)
    Mappings = new Usmap("game.usmap"),
    
    // Custom serialization flags
    CustomSerializationFlags = CustomSerializationFlags.None,
    
    // JSON formatting
    FormatJson = true,
    
    // Overwrite existing files
    OverwriteExisting = true,
    
    // Progress callback
    ProgressCallback = (result, current, total) =>
    {
        string status = result.Success ? "OK" : "FAIL";
        Console.WriteLine($"[{current}/{total}] {status}: {result.InputPath}");
    },
    
    // Cancellation support
    CancellationToken = cancellationTokenSource.Token
};
```

---

### BatchProcessResult

Result information for each processed file.

| Property | Type | Description |
|----------|------|-------------|
| `InputPath` | `string` | Path to the input file |
| `OutputPath` | `string` | Path to the output file |
| `Success` | `bool` | Whether processing succeeded |
| `ErrorMessage` | `string` | Error message if failed |
| `Exception` | `Exception` | Full exception if failed |
| `ProcessingTime` | `TimeSpan` | Time taken to process |

#### Checking Results

```csharp
var results = BatchProcessor.UAssetToJson(files, outputDir, options);

int succeeded = results.Count(r => r.Success);
int failed = results.Count(r => !r.Success);

Console.WriteLine($"Processed: {succeeded} succeeded, {failed} failed");

// Get failed files
var failures = results.Where(r => !r.Success);
foreach (var fail in failures)
{
    Console.WriteLine($"Failed: {fail.InputPath} - {fail.ErrorMessage}");
}
```

---

## Engine Versions

Common engine versions for the `EngineVersion` enum:

| Version | Description |
|---------|-------------|
| `VER_UE4_0` - `VER_UE4_27` | Unreal Engine 4.x versions |
| `VER_UE5_0` | Unreal Engine 5.0 |
| `VER_UE5_1` | Unreal Engine 5.1 |
| `VER_UE5_2` | Unreal Engine 5.2 |
| `VER_UE5_3` | Unreal Engine 5.3 |
| `VER_UE5_4` | Unreal Engine 5.4 |
| `VER_UE5_5` | Unreal Engine 5.5 |
| `UNKNOWN` | Auto-detect (for versioned assets) |

---

## Complete Examples

### Example 1: Single File Conversion

```csharp
using UAssetAPI;
using UAssetAPI.UnrealTypes;
using UAssetAPI.Unversioned;

// Setup
var mappings = new Usmap("Marvel.usmap");

// Read and convert to JSON
var asset = new UAsset("weapon.uasset", EngineVersion.VER_UE5_3, mappings);
string json = asset.SerializeJson(true);
File.WriteAllText("weapon.json", json);

// Modify JSON externally, then convert back
string modifiedJson = File.ReadAllText("weapon_modified.json");
var modifiedAsset = UAsset.DeserializeJson(modifiedJson);
modifiedAsset.Mappings = mappings;
modifiedAsset.Write("weapon_modified.uasset");
```

### Example 2: Batch Convert Entire Game

```csharp
using UAssetAPI;
using UAssetAPI.UnrealTypes;
using UAssetAPI.Unversioned;

var mappings = new Usmap("game.usmap");

var options = new BatchProcessOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount,
    EngineVersion = EngineVersion.VER_UE5_3,
    Mappings = mappings,
    FormatJson = true,
    ProgressCallback = (result, current, total) =>
    {
        double percent = (double)current / total * 100;
        Console.Write($"\r[{percent:F1}%] Processing... {current}/{total}");
    }
};

Console.WriteLine("Converting all assets to JSON...");
var results = BatchProcessor.ConvertDirectoryToJson(
    @"C:\Game\Content",
    @"C:\Output\Json",
    recursive: true,
    options
);

Console.WriteLine($"\nDone! {results.Count(r => r.Success)} succeeded, {results.Count(r => !r.Success)} failed");
```

### Example 3: With Cancellation Support

```csharp
using var cts = new CancellationTokenSource();

var options = new BatchProcessOptions
{
    EngineVersion = EngineVersion.VER_UE5_3,
    Mappings = mappings,
    CancellationToken = cts.Token
};

// Start async conversion
var task = BatchProcessor.UAssetToJsonAsync(files, outputDir, options);

// Cancel after 30 seconds
cts.CancelAfter(TimeSpan.FromSeconds(30));

try
{
    var results = await task;
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operation was cancelled");
}
```

### Example 4: Error Handling

```csharp
try
{
    var asset = new UAsset("asset.uasset", EngineVersion.VER_UE5_3, mappings);
    string json = asset.SerializeJson(true);
}
catch (FormatException ex)
{
    Console.WriteLine($"Asset parsing error: {ex.Message}");
}
catch (FileNotFoundException ex)
{
    Console.WriteLine($"File not found: {ex.Message}");
}
catch (Exception ex)
{
    Console.WriteLine($"Unexpected error: {ex.Message}");
}
```

---

## Notes

- **Usmap Required**: For unversioned game assets (most modern UE5 games), you must provide a .usmap file
- **Engine Version**: Match the engine version to your game for best results
- **Binary Equality**: Rebuilt assets may have minor hash differences but are functionally identical
- **Kismet Bytecode**: Blueprint bytecode is preserved as raw bytes (not parsed in this lightweight build)
