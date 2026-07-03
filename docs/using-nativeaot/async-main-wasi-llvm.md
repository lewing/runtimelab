# Async `Main` on wasi-wasm with NativeAOT-LLVM

This document describes the async-`Main`-on-WASI change carried on this branch, and how
to build and consume the NativeAOT-LLVM toolchain packages that contain it (including for
cross-platform testing, e.g. from [nesm](https://github.com/Blazor-Playground/nesm)).

## Why

On `wasi-wasm`, `AsyncHelpers.NonBrowser.cs` routes the compiler-generated async entry point
through a blocking `Task.GetAwaiter().GetResult()`. WASI is single-threaded
(`!IsMultithreadingSupported`), so that blocking wait throws
`PlatformNotSupportedException` at runtime. As a result, `async Task Main` /
`async Task<int> Main` fail on wasi-wasm NativeAOT-LLVM.

This branch adds a WASI-specific `AsyncHelpers` partial that routes the entry point through
the WASI event loop instead, mirroring the pattern from
[dotnet/runtime#130051](https://github.com/dotnet/runtime/pull/130051) (the CoreCLR
wasi-wasm interpreter standup).

## What changed

Two files, entirely in the managed framework (codegen-agnostic):

- `src/libraries/System.Private.CoreLib/src/System/Runtime/CompilerServices/AsyncHelpers.Wasi.cs`
  (new) — routes `HandleAsyncEntryPoint(Task)` / `HandleAsyncEntryPoint(Task<int>)` through
  `WasiEventLoop.PollWasiEventLoopUntilResolvedVoid` / `PollWasiEventLoopUntilResolved`.
- `src/libraries/System.Private.CoreLib/src/System.Private.CoreLib.Shared.projitems` — compiles
  `AsyncHelpers.Wasi.cs` when `TargetsWasi == true`, and excludes `AsyncHelpers.NonBrowser.cs`
  on WASI.

`WasiEventLoop.PollWasiEventLoopUntilResolved{,Void}` already exist on this branch, so no
runtime/native change is required. The fix ends up in the managed
`System.Private.CoreLib.dll` that ILC links.

## Building the toolchain packages

The change ships inside the NativeAOT-LLVM packages. Building them requires the LLVM
codegen host (ILC + the LLVM jit) and the wasi framework.

### Prerequisites

- This branch checked out.
- A WASI SDK. The build downloads a matching one automatically if `WASI_SDK_PATH` points to a
  version it doesn't expect, but you can also point at your own:
  `export WASI_SDK_PATH=/path/to/wasi-sdk` (must contain `share/wasi-sysroot` and a `VERSION`
  file). See `eng/AcquireWasiSdk.targets` for the expected version.
- LLVM built from source for the host jit (see next step).

### 1. Build LLVM (host jit dependency)

The LLVM codegen jit links a small subset of LLVM. Build LLVM `18.1.3` with only the
WebAssembly target and the `LLVMCore` + `LLVMBitWriter` libraries. See
`eng/pipelines/runtimelab/install-llvm.ps1` for the canonical script; the equivalent manual
steps are:

```bash
git clone --depth 1 --branch llvmorg-18.1.3 https://github.com/llvm/llvm-project
cd llvm-project
cmake -G Ninja -S llvm -B build-release \
  -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_ENABLE_TERMINFO=0 \
  -DLLVM_TARGETS_TO_BUILD=WebAssembly -DCMAKE_BUILD_TYPE=Release
cmake --build build-release --parallel --target LLVMCore LLVMBitWriter
export LLVM_CMAKE_CONFIG_RELEASE="$PWD/build-release/lib/cmake/llvm"
```

On Windows, run `install-llvm.ps1` directly (it sets `LLVM_CMAKE_CONFIG_RELEASE` for you).
This build only needs to be done once per machine (it is independent of the runtime).

### 2. Build the wasi framework

```bash
./build.sh clr.aot+libs -a wasm -os wasi -c Release      # build.cmd on Windows
```

### 3. Build the host jit + ILC (host architecture)

```bash
./build.sh clr.llvmjit+clr.aot -c Release                # no -a/-os: builds for the host
```

### 4. Produce the packages

```bash
./build.sh nativeaot.packages -a wasm -os wasi -c Release   # runtime.wasi-wasm.* (framework)
./build.sh nativeaot.packages -c Release                    # runtime.<hostRID>.* (host ILC/jit)
```

Output lands in `artifacts/packages/Release/Shipping/`:

| Package | Contents | Per-platform? |
|---------|----------|---------------|
| `Microsoft.DotNet.ILCompiler.LLVM` | Build targets, RID graph (architecture-independent) | No — shared |
| `runtime.wasi-wasm.Microsoft.DotNet.ILCompiler.LLVM` | wasi framework incl. the patched `System.Private.CoreLib.dll` | No — shared |
| `runtime.<hostRID>.Microsoft.DotNet.ILCompiler.LLVM` | Native ILC + LLVM jit for the build host | **Yes** — one per host |

The default package version is `10.0.0-dev`.

### Cross-platform testing

For xplat testing you only need to rebuild the **host** package on each platform where you
run the compiler; the `runtime.wasi-wasm.*` framework package and the architecture-independent
`Microsoft.DotNet.ILCompiler.LLVM` package are host-independent and can be shared.

1. On each host (`linux-x64`, `win-x64`, `osx-x64`, `osx-arm64`), run steps 1, 3 and the
   second command of step 4 to produce that host's `runtime.<hostRID>.Microsoft.DotNet.ILCompiler.LLVM`.
2. Build the shared `runtime.wasi-wasm.*` and `Microsoft.DotNet.ILCompiler.LLVM` packages once
   (steps 2 and the first command of step 4).
3. Publish all packages to a feed the consuming project restores from.

## Consuming the packages (e.g. from nesm)

Add the package feed and reference both the architecture-independent package and the host
runtime package. The SDK resolves the correct `runtime.<targetRID>.*` (i.e. `wasi-wasm`) via
the RID graph:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.DotNet.ILCompiler.LLVM" Version="[10.0.0-dev]" />
  <PackageReference Include="runtime.$(NETCoreSdkPortableRuntimeIdentifier).Microsoft.DotNet.ILCompiler.LLVM" Version="[10.0.0-dev]" />
</ItemGroup>
```

Use an **exact** version pin. `10.0.0-dev` sorts *below* published `-rc`/`-preview` versions,
so a floating `10.0.0-*` would prefer a public feed build without this fix. Alternatively,
re-stamp the packages to a published version string before publishing them to your feed.

Then publish for wasi-wasm:

```bash
export WASI_SDK_PATH=/path/to/wasi-sdk
dotnet publish -r wasi-wasm -c Release
```

### Required consumer settings

- **Target `net11.0`.** The async routing is only emitted by the C# compiler if
  `AsyncHelpers.HandleAsyncEntryPoint` is visible in the targeting pack. The `net10.0` GA
  targeting pack does not include it, so a `net10.0` project silently falls back to the
  blocking path and still throws `PlatformNotSupportedException`. The `net11.0` targeting pack
  includes it.
- Standard NativeAOT-LLVM project settings apply: `SelfContained=true`, `PublishTrimmed=true`,
  `MSBuildEnableWorkloadResolver=false`, `UseAppHost=false`. Do **not** set `PublishAot=true`
  (NativeAOT-LLVM is not integrated into the SDK's PublishAot path for wasm).

### WASI SDK version

The linker (`wasm-ld`, invoked via `clang` from the WASI SDK) is sensitive to version:

- **25.x** — links (a version-mismatch warning is expected).
- **29.x** — the expected version.
- **33.x** — fails at link time with
  `--global-base cannot be less than stack size when --stack-first is used`. Do not use.

## Verifying

Run the produced module under any WASI runtime, e.g. `wasmtime app.wasm`. An `async Task Main`
that `await`s should suspend and resume:

```
before await
resumed after await
```

Note: on wasi-wasm the process exit code collapses non-zero returns to `1` (a known WASI
limitation, unrelated to this change). Assert on stdout, not the exit code.
