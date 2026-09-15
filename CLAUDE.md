# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Windows DLL that extends TradeStation's EasyLanguage with custom native functions (an "EasyLanguage Extension"). TradeStation loads the built DLL and calls the exported `__stdcall` functions from EasyLanguage code.

## Build

Visual Studio 2026 (v18) solution (`EasyLanguageEncryption.sln`), C++ toolset `v145` (Visual Studio 2022 ships `v143` and would need the project retargeted), VCProjectVersion `16.0`. Four project configurations: `Debug|Win32`, `Release|Win32`, `Debug|x64`, `Release|x64`. The **solution** names the 32-bit platform `x86` (mapped to project `Win32`), so msbuild on the `.sln` needs `x86` — `/p:Platform=Win32` fails with `MSB4126`.

```bash
# From a Developer Command Prompt (or VS bash with msbuild on PATH)
msbuild EasyLanguageEncryption.sln /p:Configuration=Release /p:Platform=x64
msbuild EasyLanguageEncryption.sln /p:Configuration=Debug   /p:Platform=x86
```

Output DLL lands in `x64/<Config>/` or `<Config>/` for Win32. No test suite, no lint, no package manager.

**TradeStation dependency:** the Win32 configs add `C:\Program Files (x86)\TradeStation 10.0\Program` to `ReferencePath` so the compiler can resolve `#import "tskit.dll"`; x64 configs do not set it. **But the `#import` currently sits above `#include "pch.h"` in `Encryption.cpp`, and with precompiled headers (`/Yu`) MSVC skips every line before the PCH include** — no `tskit.tlh` is generated, and `IEasyLanguageObject` is only the forward declaration `extern class IEasyLanguageObject;`. As a result both Win32 and x64 currently build without resolving `tskit.dll`. Before using any SDK interface members, move the `#import` below `#include "pch.h"`; from then on `tskit.dll` must be resolvable, so add the reference path to the x64 configs as well.

## Architecture

- `Encryption.cpp` — the only file with extension logic. Two pieces coexist:
  - A self-contained Windows CryptoAPI (`advapi32`) file-encryption routine (`MyEncryptFile`) using `CALG_RC4` + MD5 password hash. Not currently wired to an export.
  - EasyLanguage-callable exports: `IsExtensionReady`, `AESEncrypt`, `AESDecrypt` (the latter two are stubs returning `NULL`).
- `EasyLanguageEncryption.def` — **must list every symbol** you want TradeStation to see. Currently only `IsExtensionReady` is exported. Adding a new EL-callable function requires adding its name here and rebuilding.
- `dllmain.cpp` — standard stub, no per-attach logic.
- `pch.h` / `pch.cpp` / `framework.h` — precompiled header infrastructure. All `.cpp` files use `pch.h` as PCH (set to `Create` on `pch.cpp`, `Use` elsewhere); `framework.h` pulls in `<windows.h>` with `WIN32_LEAN_AND_MEAN`.
- `json.h` / `json.cpp` — bundled copy of the json-parser C library (third-party, BSD-style license in the header). Not currently referenced from `Encryption.cpp`. `3rd-party/` contains an additional copy of jsoncpp and json-parser headers that is **not** in the vcxproj — only the top-level `json.cpp`/`json.h` are compiled.

### EasyLanguage export ABI

Functions called from EasyLanguage follow this signature shape (see `Encryption.cpp`):

```cpp
double __stdcall IsExtensionReady(IEasyLanguageObject* pELObject);
LPSTR  __stdcall AESEncrypt(IEasyLanguageObject* pELObject, LPSTR plaintext);
```

- `__stdcall` calling convention is mandatory.
- First parameter is always `IEasyLanguageObject*` (meant to come from `tskit.dll`; currently only forward-declared — see Build).
- Return/parameter types map to EasyLanguage types (`double` for numeric, `LPSTR` for string).
- Add the unadorned function name to `EasyLanguageEncryption.def` under `EXPORTS`.

### Character set quirk

x64 configs use `CharacterSet=Unicode`; Win32 configs use `CharacterSet=NotSet` (i.e., MBCS/ANSI). The `TEXT()` / `LPTSTR` macros in `Encryption.cpp` therefore resolve differently per platform — be careful when passing strings across the EL boundary, which is ANSI (`LPSTR`).
