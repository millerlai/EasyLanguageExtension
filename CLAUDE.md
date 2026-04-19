# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Windows DLL that extends TradeStation's EasyLanguage with custom native functions (an "EasyLanguage Extension"). TradeStation loads the built DLL and calls the exported `__stdcall` functions from EasyLanguage code.

## Build

Visual Studio 2022 solution (`EasyLanguageEncryption.sln`), C++ toolset `v145`, VCProjectVersion `16.0`. Four configurations: `Debug|Win32`, `Release|Win32`, `Debug|x64`, `Release|x64`.

```bash
# From a Developer Command Prompt (or VS bash with msbuild on PATH)
msbuild EasyLanguageEncryption.sln /p:Configuration=Release /p:Platform=x64
msbuild EasyLanguageEncryption.sln /p:Configuration=Debug   /p:Platform=Win32
```

Output DLL lands in `x64/<Config>/` or `<Config>/` for Win32. No test suite, no lint, no package manager.

**TradeStation dependency (Win32 only):** the Win32 configs add `C:\Program Files (x86)\TradeStation 10.0\Program` to `ReferencePath` so the compiler can resolve `#import "tskit.dll"` in `Encryption.cpp`. Win32 builds require TradeStation 10.0 installed at that path; x64 configs do not set this and will fail to resolve `tskit.dll` unless the reference path is added. The `IEasyLanguageObject` type comes from `tskit.dll`.

## Architecture

- `Encryption.cpp` — the only file with extension logic. Two pieces coexist:
  - A self-contained Windows CryptoAPI (`advapi32`) file-encryption routine (`MyEncryptFile`) using `CALG_RC4` + MD5 password hash. Not currently wired to an export.
  - EasyLanguage-callable exports: `IsExtensionReady`, `AESEncrypt`, `AESDecrypt` (the latter two are stubs returning `NULL`).
- `EasyLanguageEncryption.def` — **must list every symbol** you want TradeStation to see. Currently only `IsExtensionReady` is exported. Adding a new EL-callable function requires adding its name here and rebuilding. `.def.bak` is a stale version with the old name `IsEncryptionReady`.
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
- First parameter is always `IEasyLanguageObject*` (from `tskit.dll`).
- Return/parameter types map to EasyLanguage types (`double` for numeric, `LPSTR` for string).
- Add the unadorned function name to `EasyLanguageEncryption.def` under `EXPORTS`.

### Character set quirk

x64 configs use `CharacterSet=Unicode`; Win32 configs use `CharacterSet=NotSet` (i.e., MBCS/ANSI). The `TEXT()` / `LPTSTR` macros in `Encryption.cpp` therefore resolve differently per platform — be careful when passing strings across the EL boundary, which is ANSI (`LPSTR`).
