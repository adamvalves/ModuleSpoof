# ModuleSpoof — DLL Memory Location Redirection & Honeypot Framework

[![ModuleSpoof CI](https://github.com/adamvalves/ModuleSpoof/actions/workflows/ci.yml/badge.svg)](https://github.com/adamvalves/ModuleSpoof/actions/workflows/ci.yml)
![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Language: C++20](https://img.shields.io/badge/Language-C%2B%2B20-orange.svg)
![Platform: Windows%20x64](https://img.shields.io/badge/Platform-Windows%20x64-0078D6.svg)

`ModuleSpoof` is a proof-of-concept C++ library designed to spoof the in-memory location of a target dynamic link library (`DLL`). By intercepting internal Win32 and Native API calls (`GetModuleHandleW`, `GetModuleHandleExW`, `GetProcAddress`, `GetProcAddressForCaller`, `LoadLibraryExW`), internal execution is transparently redirected to a manually mapped copy of the module residing at a secondary memory region.

Simultaneously, the original PEB-registered module location acts as an active **honeypot** to detect unauthorized memory inspection or unbacked access attempts by external tools.

---

## 📐 Architecture & Execution Flow

```
+--------------------------------------------------------------------------------+
|                               Process Memory                                   |
|                                                                                |
|  +------------------------------+             +-----------------------------+  |
|  |     PEB -> LDR Registered    |             |   Manually Mapped Module    |  |
|  |   "protected.dll" (Fake)     |             |      (Real Execution)       |  |
|  |  +------------------------+  |             |  +------------------------+ |  |
|  |  | Honeypot Memory Guard  |  |             |  | Valid Code & Export    | |  |
|  |  | (PAGE_NOACCESS trap)   |  |             |  | Table                  | |  |
|  |  +------------------------+  |             |  +------------------------+ |  |
|  +------------------------------+             +-----------------------------+  |
|                 ^                                            ^                 |
|                 | External Scans                             | Internal Calls  |
|                 +------------------ [ API Traps ] -----------+                 |
+--------------------------------------------------------------------------------+
```

### 1. Interception & Redirection Engine
When target module resolution APIs are invoked within the process:
- `GetModuleHandleW` / `GetModuleHandleExW`: Redirected to return the base address of the manually mapped payload (`remapped_location`).
- `GetProcAddress` / `GetProcAddressForCaller`: Intercepted to resolve export entrypoints dynamically from the manually mapped image headers.
- `LoadLibraryExW`: Trapped to prevent secondary loads of the original disk binary.

### 2. Honeypot Trap Memory Guard
The PEB-registered module base address remains visible to standard module enumeration APIs (`PEB->LDR`). However, memory pages at this address are trapped (e.g. marked `PAGE_NOACCESS` or hooked). Any external reader or scanner attempting to inspect or dump the module base triggers an exception catch/detection routine (`honeypot.cpp`).

---

## ⚙️ Technical Details

1. **`LdrRegisterDllNotification` Hooking**: Listens for module loading events. Upon detecting `protected.dll` load, it intercepts the entrypoint and manually maps the DLL payload to an independent virtual memory region using `manual_map`.
2. **Win32 API Detours**: Uses a lightweight inline hooking engine (`z_hook`) targeting `kernelbase.dll` exports:
   - `GetModuleHandleW`
   - `GetModuleHandleExW`
   - `GetProcAddress`
   - `GetProcAddressForCaller`
   - `LoadLibraryExW`
   - `FreeLibrary`

---

## 🛠 Project Structure

```
ModuleSpoof/
├── ModuleSpoof.sln           # Visual Studio 2022 Solution
├── ModuleSpoof/
│   ├── main.cpp              # Entrypoint & initialization logic
│   ├── spoof_module.cpp      # API interception, Ldr notification & redirection logic
│   ├── spoof_module.hpp      # Redirection function prototypes & header declarations
│   ├── manual_map.cpp        # Manual PE image mapping loader implementation
│   ├── manual_map.hpp        # PE loader headers & relocation structures
│   ├── honeypot.cpp          # Memory page guard trap & access detection
│   ├── honeypot.hpp          # Honeypot prototypes
│   ├── z_hook.cpp            # Inline trampoline hooking library
│   ├── z_hook.hpp            # Hook definitions & gateway allocation
│   └── util.hpp              # Internal logging & helper macros
└── Protected/
    ├── main.cpp              # Sample target DLL containing test exports
    └── Protected.vcxproj     # Test target project
```

---

## 🚀 Building & Usage

### Prerequisites
- **Visual Studio 2022** (C++ Desktop Development workload)
- **Windows SDK 10.0**

### Build Steps
1. Open `ModuleSpoof.sln` in Visual Studio.
2. Select Configuration (`Release` | `x64`).
3. Build Solution (`Ctrl + Shift + B`).

---

## 🔬 Proof of Concept Demonstration

The included `Protected.dll` contains basic test exports (`export_test`). When injected or loaded alongside `ModuleSpoof`:
- Calls to `GetModuleHandleA("Protected.dll")` return the redirected, manually mapped base.
- Calls to `GetProcAddress` execute valid functions from the remapped memory space.
- Access attempts to the PEB-registered base address trigger the honeypot guard.

---

## 📜 References & Acknowledgments

- **Self-Remapping Code**: [changeofpace/Self-Remapping-Code](https://github.com/changeofpace/Self-Remapping-Code)

---

## 📄 License

Distributed under the MIT License. See `LICENSE` for details.
