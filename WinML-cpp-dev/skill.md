---
name: winml-cpp-dev
description: C++ development using Microsoft.Windows.AI.MachineLearning package (Windows ML / ONNX Runtime) for running AI inference on Windows devices with NPU/GPU/CPU execution providers. Covers EP registration, session creation, model compilation, tensor I/O, and image pre/post-processing pipelines.
topics: [windows-ml, cpp-onnx-inference, winrt, ryzenai-npu]
---

# Windows ML C++ Development (Microsoft.Windows.AI.MachineLearning)

## Quick Start Checklist

1. Target **Windows App SDK** project with WinRT support (`/W` or `/winrt` flag in MSVC).
2. Include headers: `<winml/onnxruntime_cxx_api.h>` and WinRT Machine Learning namespace.
3. Register installed Execution Providers via `ExecutionProviderCatalog::EnsureAndRegisterCertifiedAsync()`.
4. Configure `SessionOptions` with EP selection policy (`SetEpSelectionPolicy()`).
5. Compile ONNX model to optimized representation (one-time cost, cache result).
6. Create `Ort::Session`, prepare input tensor, run inference, decode output.

---

## Prerequisites

- Visual Studio 2022 or later with C++ workload and Windows App SDK extension installed
- Windows 11 build 22621+ (Windows ML new runtime) **or** Windows 10/11 with Windows App SDK for back-level targets
- ONNX Runtime C++ API is shipped as part of `Microsoft.Windows.AI.MachineLearning` NuGet package — do NOT install a separate onnxruntime NuGet

### Key Source Files and Includes

```cpp
#include <winml/onnxruntime_cxx_api.h>        // Ort::Env, Ort::Session, etc.
#include <winrt/base.h>
#include <winrt/Microsoft.Windows.AI.MachineLearning.h>

using namespace winrt::Microsoft::Windows::AI::MachineLearning;
```

For image loading/preprocessing (WinRT):
```cpp
#include <winrt/Windows.Foundation.h>
#include <winrt/Windows.Foundation.Collections.h>
#include <winrt/Windows.Graphics.Imaging.h>
#include <winrt/Windows.Media.h>
#include <winrt/Windows.Storage.h>
#include <winrt/Windows.Storage.Streams.h>
```

For C (non-WinRT) EP catalog enumeration (alternate path):
```cpp
#include <WinMLEpCatalog.h>                   // WinMLEpCatalogCreate, etc.
```

---

## Core Concepts

### Execution Providers (EPs)

An Execution Provider is a hardware-backed inference backend:
- **CPUExecutionProvider** — always available by default
- **DmlExecutionProvider** — GPU (DirectML)
- Device-specific EPs downloaded from Microsoft Store on demand: QNNExecutionProvider (Qualcomm NPU), VitisAI / MIGraphX (AMD RYZEN AI NPU), etc.

EPs have `ReadyState` lifecycle:

| State       | Meaning                                           | Action                                  |
|------------|---------------------------------------------------|-----------------------------------------|
| NotPresent  | EP not installed on device                        | Call `EnsureReadyAsync()` to download   |
| NotReady    | Installed but not in app's dependency graph       | Call `EnsureReadyAsync()` to bind       |
| Ready       | Available and bound to this app                   | Call `TryRegister()` or auto-register   |

### EP Selection Policies

Set on `SessionOptions` — tells the runtime how to rank available EPs:

| Policy Value                                         | Intent                        |
|------------------------------------------------------|-------------------------------|
| `OrtExecutionProviderDevicePolicy_PREFER_NPU`        | Prefer NPU over GPU/CPU       |
| `OrtExecutionProviderDevicePolicy_PREFER_CPU`        | Force CPU                     |
| `OrtExecutionProviderDevicePolicy_MIN_OVERALL_POWER` | Optimize for power efficiency |
| `OrtExecutionProviderDevicePolicy_MAX_PERF`          | Optimize for raw performance  |
| `OrtExecutionProviderDevicePolicy_DEFAULT`           | OS default                    |

---

## Step-by-Step Implementation Guide

### 1. Create ONNX Runtime Environment and Register EPs (WinRT path)

```cpp
auto env = Ort::Env();
std::cout << "ONNX Version: " << Ort::GetVersionString() << std::endl;

// Get the default catalog of Windows ML execution providers
auto catalog = ExecutionProviderCatalog::GetDefault();

// Register all certified EPs already present on this machine (downloads missing ones)
catalog.EnsureAndRegisterCertifiedAsync().get();

// Optionally enumerate what got registered
for (const auto& provider : catalog.FindAllProviders())
{
    std::wcout << L"Provider: " << provider.Name().c_str()
               << L", ReadyState: " << static_cast<int>(provider.ReadyState()) << std::endl;
}

// Verify ONNX Runtime sees them now
for (const auto& device : env.GetEpDevices())
{
    std::cout << "ONNX EP: " << device.EpName() << std::endl;
}
```

### 2. Register a Specific EP Individually

```cpp
auto catalog = ExecutionProviderCatalog::GetDefault();
for (const auto& provider : catalog.FindAllProviders())
{
    if (provider.Name().compare(L"VitisAIExecutionProvider") == 0)
    {
        // Download and add to app dependency graph if needed
        auto result = provider.EnsureReadyAsync().get();
        if (result.Status() == ExecutionProviderReadyResultState::Success)
        {
            bool registered = provider.TryRegister();
            std::cout << "VitisAI EP registered: " << registered << std::endl;
        }
    }
}
```

### 3. Register All Certified EPs via C API (non-WinRT fallback)

When not using WinRT wrappers, enumerate with `WinMLEpCatalog` and register each by library path to `Ort::Env`:

```cpp
#include <WinMLEpCatalog.h>
#include <onnxruntime_cxx_api.h>
#include <filesystem>

struct RegisterContext { Ort::Env* env; };

BOOL CALLBACK RegisterInstalledCallback(WinMLEpHandle ep, const WinMLEpInfo* info, void* context)
{
    auto* ctx = static_cast<RegisterContext*>(context);
    if (!info || !info->name) return TRUE;
    if (info->certification != WinMLEpCertification_Certified) return TRUE;
    if (FAILED(WinMLEpEnsureReady(ep))) return TRUE;

    size_t pathSize = 0;
    if (FAILED(WinMLEpGetLibraryPathSize(ep, &pathSize)) || pathSize == 0) return TRUE;

    std::string libraryPathUtf8(pathSize, '\0');
    if (FAILED(WinMLEpGetLibraryPath(ep, info->name, sizeof(info->name),
                                     libraryPathUtf8.data(), nullptr))) return TRUE;

    std::filesystem::path libPath(libraryPathUtf8);
    ctx->env->RegisterExecutionProviderLibrary(info->name, libPath.wstring());
    return TRUE;
}

// Usage:
Ort::Env env(ORT_LOGGING_LEVEL_ERROR, "MyApp");
WinMLEpCatalogHandle catalog = nullptr;
HRESULT hr = WinMLEpCatalogCreate(&catalog);
if (SUCCEEDED(hr)) {
    RegisterContext ctx{&env};
    WinMLEpCatalogEnumProviders(catalog, RegisterInstalledCallback, &ctx);
}
```

### 4. Configure Session Options

```cpp
Ort::SessionOptions sessionOptions;
sessionOptions.SetLogSeverityLevel(1);  // 0=Verbose, 1=Info, 2=Warning, 3=Error

// Prefer NPU-first execution (falls back to GPU then CPU)
sessionOptions.SetEpSelectionPolicy(OrtExecutionProviderDevicePolicy_PREFER_NPU);

// Thread spinning is DISABLED by default in Windows ML for battery life.
// Only enable if responsiveness matters and you've tested thermal impact:
// sessionOptions.AddConfigEntry("session.intra_op.allow_spinning", "1");
// sessionOptions.AddConfigEntry("session.inter_op.allow_spinning", "1");
```

### 5. Optional Compile ONNX Model (One-Time, Cache Result)

New API from ONNX Runtime 1.22+ (`OrtCompileApi`), WinRT C++ wrapper:

```cpp
bool isCompiledModelAvailable = std::filesystem::exists(compiledModelPath);

if (!isCompiledModelAvailable)
{
    Ort::ModelCompilationOptions compile_options(env, sessionOptions);
    compile_options.SetInputModelPath(modelPath.c_str());
    compile_options.SetOutputModelPath(compiledModelPath.c_str());

    std::cout << "Compiling model (may take several minutes)...";
    Ort::Status status = Ort::CompileModel(env, compile_options);
    if (!status.IsOK()) {
        std::cerr << "\nCompilation failed: " << status.GetErrorMessage() << std::endl;
        // Fall back to uncompiled model — still works, just slower first run
    } else {
        isCompiledModelAvailable = true;
    }
}

std::filesystem::path modelPathToUse = isCompiledModelAvailable ? compiledModelPath : modelPath;
```

**Important**: Compilation can take minutes. Run it on a background thread to keep the UI responsive. Store compiled models in `ApplicationData.Current.LocalFolder` for reuse. Updates to EPs or runtime may require recompilation.

### 6. Create Inference Session and Inspect Model Signature

```cpp
Ort::Session session(env, modelPathToUse.c_str(), sessionOptions);

// Get input info
auto inputTypeInfo = session.GetInputTypeInfo(0).GetTensorTypeAndShapeInfo();
auto inputElementType = inputTypeInfo.GetElementType();  // e.g., ONNX_TENSOR_ELEMENT_DATA_TYPE_FLOAT
auto inputShape = inputTypeInfo.GetShape();              // e.g., {1, 3, 224, 224}

// Get output info
auto outputTypeInfo = session.GetOutputTypeInfo(0).GetTensorTypeAndShapeInfo();
auto outputElementType = outputTypeInfo.GetElementType();
auto outputShape = outputTypeInfo.GetShape();            
```

### 7. Prepare Input Tensor from Image Data

Complete image preprocessing pipeline using WinRT (no OpenCV dependency needed):

```cpp
#include <winrt/Windows.Storage.h>
#include <winrt/Windows.Graphics.Imaging.h>
#include <windows.ai.machinelearning.densetensor.h>

using namespace winrt::Windows::Storage;
using namespace winrt::Windows::Graphics::Imaging;
using namespace winrt::Windows::Storage::Streams;

// Async image loader
IAsyncOperation<SoftwareBitmap> LoadImageFileAsync(winrt::hstring filePath)
{
    auto file = co_await StorageFile::GetFileFromPathAsync(filePath);
    auto stream = co_await file.OpenAsync(FileAccessMode::Read);
    auto decoder = co_await BitmapDecoder::CreateAsync(stream);
    auto bitmap = co_await decoder.GetSoftwareBitmapAsync();
    co_return bitmap;
}

// Convert SoftwareBitmap -> float tensor with normalization
std::vector<float> BitmapToTensor(SoftwareBitmap const& bitmap,
                                    int targetW, int targetH,
                                    std::array<float,3> mean, std::array<float,3> stddev)
{
    // 1. Convert to BGRA8
    auto bgra = SoftwareBitmap::Convert(bitmap, BitmapPixelFormat::Bgra8, BitmapAlphaMode::Ignore);

    // 2. Resize to target dimensions using BitmapEncoder (highest quality interpolation)
    InMemoryRandomAccessStream stream;
    auto encoder = co_await BitmapEncoder::CreateAsync(BitmapEncoder::BmpEncoderId(), stream).get();
    encoder.SetSoftwareBitmap(bgra);
    encoder.BitmapTransform().ScaledWidth(targetW);
    encoder.BitmapTransform().ScaledHeight(targetH);
    encoder.BitmapTransform().InterpolationMode(BitmapInterpolationMode::Fant);
    encoder.FlushAsync().get();

    stream.Seek(0);
    auto decoder = BitmapDecoder::CreateAsync(stream).get();
    auto resized = decoder.GetSoftwareBitmapAsync(BitmapPixelFormat::Bgra8, BitmapAlphaMode::Ignore).get();

    // 3. Lock pixels and read into interleaved CHW float tensor
    auto buf = resized.LockBuffer(BitmapBufferAccessMode::Read);
    auto ref = buf.CreateReference();
    impl::com_ptr<IMemoryBufferByteAccess> spByteAccess;
    ref.as(&spByteAccess);

    byte* pixelData = nullptr;
    uint32_t capacity = 0;
    spByteAccess->GetBuffer(&pixelData, &capacity);

    const int64_t channels = 3;
    std::vector<float> tensor(channels * targetH * targetW);

    for (int y = 0; y < targetH; ++y) {
        for (int x = 0; x < targetW; ++x) {
            int idx = (y * targetW + x) * 4;  // BGRA stride of 4
            float r = pixelData[idx + 2] / 255.0f;
            float g = pixelData[idx + 1] / 255.0f;
            float b = pixelData[idx]     / 255.0f;

            tensor[0 * targetH * targetW + y * targetW + x] = (r - mean[0]) / stddev[0];
            tensor[1 * targetH * targetW + y * targetW + x] = (g - mean[1]) / stddev[1];
            tensor[2 * targetH * targetW + y * targetW + x] = (b - mean[2]) / stddev[2];
        }
    }
    return tensor;
}

// For ImageNet models (ResNet, EfficientNet): normalize with these values:
// mean  = {0.485f, 0.456f, 0.406f}
// stddev= {0.229f, 0.224f, 0.225f}
```

### 8. Create OrtValue Tensor and Run Inference

```cpp
auto inputShape = std::array<int64_t, 4>{1, 3, 224, 224};  // NCHW
auto memoryInfo = Ort::MemoryInfo::CreateCpu(OrtArenaAllocator, OrtMemTypeDefault);

// Handle both FLOAT and FLOAT16 model inputs:
OnnxTensorElementDataType inputType = session.GetInputTypeInfo(0)
                                        .GetTensorTypeAndShapeInfo()
                                        .GetElementType();

std::vector<float> tensorData = BitmapToTensor(bitmap, 224, 224,
    {0.485f, 0.456f, 0.406f}, {0.229f, 0.224f, 0.225f});

std::vector<uint8_t> rawBytes;
if (inputType == ONNX_TENSOR_ELEMENT_DATA_TYPE_FLOAT16) {
    // Float32->Float16 conversion needed for quantized models:
    auto fp16data = ConvertFp32ToFp16(tensorData);
    rawBytes.assign(reinterpret_cast<uint8_t*>(fp16data.data()),
                    reinterpret_cast<uint8_t*>(fp16data.data()) + fp16data.size() * sizeof(uint16_t));
} else {
    rawBytes.assign(reinterpret_cast<uint8_t*>(tensorData.data()),
                    reinterpret_cast<uint8_t*>(tensorData.data()) + tensorData.size() * sizeof(float));
}

// Create input OrtValue from raw bytes:
Ort::AllocatorWithDefaultOptions allocator;
auto inputName = session.GetInputNameAllocated(0, allocator).get();
auto outputName = session.GetOutputNameAllocated(0, allocator).get();

std::vector<const char*> inputNames  = {inputName};
std::vector<const char*> outputNames = {outputName};

OrtValue* ortValuePtr = nullptr;
ORT_THROW_ON_ORTAPIERROR(CREATE_TENSOR_WITH_DATA_AS_ORTVALUE(
    memoryInfo, rawBytes.data(), rawBytes.size(),
    inputShape.data(), inputShape.size(), inputType, &ortValuePtr) -> Ort::Env);

Ort::Value inputTensor{ortValuePtr};

// Run inference:
std::vector<Ort::Value> outputs;
outputs = session.Run(Ort::RunOptions{}, inputNames.data(), &inputTensor, 1, outputNames.data(), 1);
```

Using `Windows.AI.MachineLearning.DenseTensor` (WinRT path) — this wraps OrtValue internally:

```cpp
#include <winrt/Windows.AI.MachineLearning.h>
using namespace winrt::Windows::AI::MachineLearning;

// Create a tensor from unmanaged data (bind existing memory):
auto tensor = MachineLearningTensorFunctionBinding::CreateInputTensor(
    LearningModelBindingDestinationKind::Cpu,  // or Gpu/Npu
    OnnxTensorDataType::Float,
    winrt::single_threaded_vector<int64_t>({1, 3, 224, 224}),
    out_tensor_data->data(),          // void* pointer to float array
    sizeof(float) * tensorData.size() // byte length
);

// Add binding:
binding.Bind(inputName.c_str(), tensor);
```

### 9. Extract and Decode Output

```cpp
auto outputTensor = outputs[0];
size_t elementCount = outputTensor.GetTensorTypeAndShapeInfo().GetElementCount();

std::vector<float> results;
if (inputElementType == ONNX_TENSOR_ELEMENT_DATA_TYPE_FLOAT16) {
    auto fp16ptr = outputTensor.GetTensorData<uint16_t>();
    std::vector<uint16_t> fp16buf(fp16ptr, fp16ptr + elementCount);
    results = ConvertFp16ToFp32(fp16buf);
} else {
    auto ptr = outputTensor.GetTensorData<float>();
    results.assign(ptr, ptr + elementCount);
}

// Softmax for classification models:
std::vector<float> probs = ApplySoftmax(results);

// Show top-K predictions with labels:
ShowTopK(labels, probs, 5);
```

---

## Complete Minimal Example

Full standalone `main.cpp` template (wmain entry point):

```cpp
#include <winml/onnxruntime_cxx_api.h>
#include <winrt/base.h>
#include <winrt/Microsoft.Windows.AI.MachineLearning.h>

using namespace winrt::Microsoft::Windows::AI::MachineLearning;

int wmain(int argc, wchar_t* argv[]) noexcept
try
{
    // --- 1. Environment + EP Registration ---
    auto env = Ort::Env();
    auto catalog = ExecutionProviderCatalog::GetDefault();
    catalog.EnsureAndRegisterCertifiedAsync().get();  // download & register all certified EPs

    for (const auto& device : env.GetEpDevices())
        std::cout << "EP available: " << device.EpName() << std::endl;

    // --- 2. Session Options with NPU-first policy ---
    Ort::SessionOptions sessionOpts;
    sessionOpts.SetLogSeverityLevel(1);
    sessionOpts.SetEpSelectionPolicy(OrtExecutionProviderDevicePolicy_PREFER_NPU);

    // --- 3. Compile model (one-time) or use cached .ctx file ---
    std::filesystem::path modelPath{ L"path/to/model.onnx" };   // source ONNX
    std::filesystem::path compiledPath{ L"path/to/model_ctx.onnx" };

    if (!std::filesystem::exists(compiledPath))
    {
        Ort::ModelCompilationOptions compileOpts(env, sessionOpts);
        compileOpts.SetInputModelPath(modelPath.c_str());
        compileOpts.SetOutputModelPath(compiledPath.c_str());
        std::cout << "Compiling model...";
        auto status = Ort::CompileModel(env, compileOpts);
        if (!status.IsOK())
            throw std::runtime_error{std::string("Compilation failed: ") + status.GetErrorMessage()};
    }

    // --- 4. Create Inference Session ---
    Ort::Session session(env, compiledPath.c_str(), sessionOpts);
    auto inputTypeInfo = session.GetInputTypeInfo(0).GetTensorTypeAndShapeInfo();
    auto outputTypeInfo = session.GetOutputTypeInfo(0).GetTensorTypeAndShapeInfo();

    // -- TODO: load image data into float tensor (see preprocessing section above) --
    // std::vector<float> tensorData;       // shape [1, 3, H, W]
    // std::array<int64_t, 4> inputShape{1, 3, H, W};

    // -- Create OrtValue from data and run: --
    // auto memoryInfo = Ort::MemoryInfo::CreateCpu(OrtArenaAllocator, OrtMemTypeDefault);
    // Ort::Value inputTensor = Ort::Value::CreateTensor<float>(memoryInfo, tensorData.data(),
    //     tensorData.size(), inputShape.data(), 4);

    // auto allocator = Ort::AllocatorWithDefaultOptions{};
    // std::vector<const char*> inp{session.GetInputNameAllocated(0, allocator).get()};
    // std::vector<const char*> out{session.GetOutputNameAllocated(0, allocator).get()};

    // std::vector<Ort::Value> outputs = session.Run({}, inp.data(), &inputTensor, 1, out.data(), 1);

    return 0;
}
catch (const std::exception& ex)
{
    std::cerr << "Fatal: " << ex.what() << std::endl;
    return 1;
}
```

---

## Build Configuration (Visual Studio Property Pages / CMake)

### MSBuild / .vcxproj Key Settings

- **Windows SDK**: Windows App SDK target version (10.0.22621 or later)
- **C++ Language Standard**: Latest (`/std:c++` latest); project requires WinRT support
- **Additional Include Directories**: inherited automatically from `Microsoft.Windows.AI.MachineLearning` NuGet package
- **Platform Toolset**: Visual Studio 17 (MSVC v143+)

**Compile Flags via NuGet Package:**
```xml
<ItemGroup>
    <ClCompile PrecompiledHeaderFile="$(IntDir)$(TargetName).pch">
        <AdditionalOptions>/winrt %(AdditionalOptions)</AdditionalOptions>
    </ClCompile>
</ItemGroup>
<PropertyGroup>
    <WindowsSDKDesktopARM64Support>true</WindowsSDKDesktopARM64Support>
    <AutoGenerateBindingRedirects>true</AutoGenerateBindingRedirects>
    <AppxPackage>false</AppxPackage>              <!-- desktop app, not packaged MSIX -->
    <AppxDefaultResourceId Condition="'$(Platform)'=='X86'">en-US</AppxDefaultResourceId>
</PropertyGroup>
```

### CMake (for non-VS IDE like VSCode)

```cmake
cmake_minimum_required(VERSION 3.25)
project(WinMLInference LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /winrt")

# ONNX Runtime C++ API is shipped with the Windows ML NuGet, so you reference:
find_package(onnxruntime CONFIG REQUIRED)
target_link_libraries(MyApp PRIVATE onnxruntime)

# WinRT projection headers (provided by Windows App SDK workload):
# No extra find_package needed — /winrt generates them from Windows Metadata.
```

### Restoring NuGet References in CLI Build

```powershell
cd YourProjectDirectory
nuget.exe restore .\YourProject.sln
msbuild .\YourProject.sln /p:Configuration=Release /m
```

---

## Float16 (FP32 ↔ FP16) Utility Functions

Many RyzenAI-optimized models expect `FLOAT16` input. If the model element type is `ONNX_TENSOR_ELEMENT_DATA_TYPE_FLOAT16`, convert before creating tensor:

```cpp
// FP32 -> FP16 (simple IEEE 754 half precision, no rounding control)
uint16_t Float32ToFloat16(float f) {
    uint32_t i; memcpy(&i, &f, sizeof(f));
    uint32_t sign = (i >> 16) & 0x8000;
    int exp = static_cast<int>(((i >> 23) & 0xFF)) - 127 + 15;
    uint32_t mantissa = i & 0x007FFFFF;
    if (exp <= 0)     return static_cast<uint16_t>(sign);
    if (exp >= 31)    return sign | 0x7C00;  // INF
    return sign | (exp << 10) | (mantissa >> 13);
}

std::vector<uint16_t> ConvertFp32ToFp16(std::vector<float> const& fp32data) {
    std::vector<uint16_t> out(fp32data.size());
    for (size_t i = 0; i < fp32data.size(); ++i) out[i] = Float32ToFloat16(fp32data[i]);
    return out;
}

// FP16 -> FP32 reconstruction
float Float16ToFloat32(uint16_t h) {
    uint32_t sign = (h >> 15) & 1;
    uint32_t exp = ((h >> 10) & 0x1F);
    uint32_t mantissa = h & 0x3FF;
    if (!exp) return 0.f;                        // denorm/zero -> approximate as zero
    if (exp == 31) { /* INF or NaN */ }
    exp += (127 - 15);
    uint32_t r = (sign << 31) | ((int)(exp) << 23) | (mantissa << 13);
    float f; memcpy(&f, &r, sizeof(f)); return f;
}

std::vector<float> ConvertFp16ToFp32(std::vector<uint16_t> const& fp16data) {
    std::vector<float> out(fp16data.size());
    for (size_t i = 0; i < fp16data.size(); ++i) out[i] = Float16ToFloat32(fp16data[i]);
    return out;
}
```

---

## Softmax and Top-K Utility

For classification models that produce raw logits:

```cpp
#include <algorithm>
#include <limits>

std::vector<float> ApplySoftmax(std::vector<float> const& logits) {
    float maxVal = *std::max_element(logits.begin(), logits.end());
    std::vector<float> exps(logits.size());
    float sum = 0.f;
    for (size_t i = 0; i < logits.size(); ++i) {
        exps[i] = std::expf(logits[i] - maxVal);
        sum += exps[i];
    }
    for (float& v : exps) v /= sum;
    return exps;
}

void ShowTopK(const std::vector<std::string>& labels,
              const std::vector<float>& probabilities, int k = 5) {
    std::priority_queue<
        std::pair<float, size_t>,
        std::vector<std::pair<float, size_t>>> pq;
    for (size_t i = 0; i < probabilities.size(); ++i) pq.emplace(probabilities[i], i);

    k = std::min(k, static_cast<int>(pq.size()));
    while (!pq.empty()) {
        auto [prob, idx] = pq.top(); pq.pop();
        std::cout << "  [" << idx << "] " << labels[idx]
                  << ": " << (prob * 100.f) << "%\n";
        if (--k == 0) break;
    }
}
```

---

## Common Normalization Constants by Model Family

| Model                 | Mean                      | StdDev                     | Input Shape     |
|----------------------|---------------------------|----------------------------|-----------------|
| ResNet (all variants)| [0.485, 0.456, 0.406]    | [0.229, 0.224, 0.225]     | N x 3 x H x W   |
| EfficientNet         | [0.5, 0.5, 0.5]          | [0.5, 0.5, 0.5]           | N x 3 x H x W   |
| YOLO detection       | [0, 0, 0]                | [255, 255, 255] (div only)| N x C x H x W   |
| Segment Anything     | Varies — check model card|                            | Varies          |

---

## Useful Diagnostic Commands

**Check what EPs ONNX Runtime discovered after registration:**
```cpp
for (const auto& d : env.GetEpDevices())
    std::cout << "EP: " << d.EpName() << "\n";
```

Expected output on an AMD Ryzen AI device with VitisAI NPU installed:
```
CPUExecutionProvider
DmlExecutionProvider
VitisAIExecutionProvider
MIGraphXExecutionProvider   // may appear depending on EP version
```

**Check Windows ML catalog ready states:**
```cpp
for (auto& p : ExecutionProviderCatalog::GetDefault().FindAllProviders()) {
    std::wcout << L"EP: " << p.Name() << L"\n";
    switch(p.ReadyState()) {
        case NotPresent:  std::cout << " status = Not present\n"; break;
        case NotReady:    std::cout << " status = Installed, not ready (call EnsureReadyAsync)\n"; break;
        case Ready:       std::cout << " status = Ready for inference\n";     break;
        default:          std::cout << " status = Unknown (" << static_cast<int>(p.ReadyState()) << ")\n"; break;
    }
}
```

---

## Project Structure (Reference Layout Based on AMD RyzenAI Demo)

```
CppResnetBuildDemo/
  CppResnetBuildDemo.vcxproj       # MSVC project file with WinRT + NuGet refs
  CppResnetBuildDemo.vcxproj.filters
  CppResnetBuildDemo.cpp           # wmain() — session creation, EP reg, inference loop
  ResnetModelHelper.hpp            # declarations: LoadImageFileAsync, BitmapToTensor, etc.
  ResnetModelHelper.cpp            # image loading (WinRT), normalization, FP16 convert
  ResNet50Labels.txt               # 1000 ImageNet label strings, one per line
  model/
    resnet50.onnx                  # source ONNX model
    resnet50_ctx.onnx             # compiled cached output (generated at first run)
  images/
    dog.jpg                        # test image
```

---

## Critical Gotchas and Troubleshooting

### "EnsureAndRegisterCertifiedAsync" Hangs or Never Returns on First Run
The runtime may download EP DLL files from the Microsoft Store on first invocation. Ensure:
1. Device has internet connectivity at install time
2. User account has permissions to write to `%ProgramData%\Windows ML` (or per-user app cache location)

### Compilation Takes 2-5 Minutes and UI Freezes
Run `Ort::CompileModel()` off the main thread:
```cpp
std::async(std::launch::async, [&](){
    auto status = Ort::CompileModel(env, compileOpts);
    // notify main thread via std::promise or event when done
});
```

### Model Loads but Returns Garbage Results
Verify:
- Input tensor shape (N,C,H,W) exactly matches model expectation
- Normalization uses correct mean/stddev for the specific model architecture
- If the model expects FP16, you must convert from FP32 before `CreateTensorWithDataAsOrtValue`

### NPU Not Being Used Despite PREFER_NPU Policy
Debug by checking:
- EP is actually registered (call `env.GetEpDevices()` after registration)
- Device supports NPU and the correct EP DLL is available on disk
- Model operations are all compatible with that EP — unsupported ops silently fall back to CPU

### Windows App SDK Not Found / WinRT Headers Missing
Visual Studio requires the **Windows App SDK** workload installed. In VS Installer:
> Individual components → "Windows App SDK (Production) Version X.Y Z" and "C++/WinRT"
