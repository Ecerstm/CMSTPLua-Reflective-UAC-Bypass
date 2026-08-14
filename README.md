# COM BypassUAC - Reflective DLL Injection

This project is a tool designed to bypass Windows UAC (User Account Control) using COM (Component Object Model) interfaces, combined with the Reflective DLL Injection technique.

## Overview

This project is a reflective DLL built to bypass UAC protection on Windows systems. It consists of two main components:

1. **Reflective DLL Loader** – The mechanism that loads and executes the DLL in memory
2. **UAC Bypass** – Uses COM interfaces to launch programs with elevated privileges

## Features

* **Reflective DLL Injection**: Loads a DLL entirely from memory without touching disk
* **UAC Bypass**: Uses the COM interface `ICMLuaUtil` to bypass UAC
* **Multi-Architecture Support**: x86 (32-bit), x64 (64-bit), and ARM
* **Stealth**: Leaves no artifacts on disk
* **Minimal Dependencies**: Uses only Windows API functions

## How It Works

### 1. Reflective DLL Injection

Reflective DLL Injection is a technique for loading a DLL directly into memory without using the standard Windows APIs (`LoadLibrary`, `GetProcAddress`).
The loader in this project works through the following steps:

1. **PE Header Search**: Locates the PE (Portable Executable) header by scanning backwards in memory
2. **Find Kernel32.dll**: Obtains a reference through the PEB (Process Environment Block)
3. **Resolve API Functions**: Finds required APIs using hash-based lookup:

   * `LoadLibraryA`
   * `GetProcAddress`
   * `VirtualAlloc`
   * `NtFlushInstructionCache`
4. **Memory Allocation**: Allocates memory for the DLL
5. **PE Loading**: Copies PE headers and sections into the new memory region
6. **Import Table Processing**: Loads required DLLs and resolves imports
7. **Relocation Processing**: Fixes base address differences
8. **DllMain Call**: Calls the DLL's entry point

### 2. UAC Bypass Mechanism

The UAC bypass is performed by using the Windows internal `ICMLuaUtil` COM interface:

1. **Create COM Moniker**: Generates a moniker in the format
   `Elevation:Administrator!new:{CLSID}`
2. **Create COM Object**: Calls `CoGetObject` to create an elevated COM instance
3. **ShellExec Call**: Uses `ICMLuaUtil::ShellExec` to run the target program with elevated privileges

### Execution Flow

```
[DLL Injection] → [ReflectiveLoader] → [DllMain] → [CMLuaUtilBypassUAC] → [Elevated Program]
```

## Requirements

### For Building

* **Visual Studio 2022**
* **Windows SDK**
* **C/C++ Compiler**

### For Running

* **Windows Vista or later** (systems that support UAC)
* **A non-administrator user account** (recommended for testing UAC bypass)

## Building

### Building with Visual Studio

1. Clone or download the project

2. Open `reflective_dll.sln` in Visual Studio

3. Build the project:

   * `Build` → `Build Solution` (Ctrl+Shift+B)

## Usage

### Basic Usage

This DLL must be injected into a target process using reflective injection. When injected:

1. `ReflectiveLoader` is invoked
2. The DLL loads itself into memory
3. `DllMain` is called with `DLL_PROCESS_ATTACH`
4. The target program path is read from the `lpReserved` parameter
5. `CMLuaUtilBypassUAC` performs the UAC bypass

### Example Usage Scenario

```c
// Example: Loader injecting the DLL

// 1. Load DLL into memory
LPVOID pDllBuffer = LoadDllFromFile("comBypassUac.dll");

// 2. Find the ReflectiveLoader function
REFLECTIVELOADER pReflectiveLoader = GetReflectiveLoader(pDllBuffer);

// 3. Prepare the target program path
char* targetProgram = "C:\\Windows\\System32\\cmd.exe";

// 4. Call ReflectiveLoader (passing the program path via lpReserved)
pReflectiveLoader(targetProgram);
```

### Specifying the Target Program

The DLL receives the target program path via the `lpReserved` parameter.
This is the program that will be executed with elevated privileges after the UAC bypass.

### File Descriptions

#### `src/ReflectiveDll.c`

* Main DLL entry point (`DllMain`)
* `CMLuaUtilBypassUAC`: The UAC bypass function
* `CoCreateInstanceAsAdmin`: Creates elevated COM instances
* `ICMLuaUtil` interface definitions

#### `src/ReflectiveLoader.c`

* `ReflectiveLoader`: Main function that loads the DLL from memory
* PE header parsing
* Import table resolution
* Relocation handling
* Hash-based API resolution

#### `bypassuac.h`

* COM interface definitions for `ICMLuaUtil`
* CLSID and IID declarations
* Function prototypes

## Technical Details

### COM Interface: ICMLuaUtil

This project relies on the Windows internal `ICMLuaUtil` interface:

* **CLSID**: `{3E5FC7F9-9A51-4367-9063-A120244FBEC7}` (CMSTPLUA)
* **IID**: `{6EDD6D74-C007-4E75-B76A-E5740995E24C}` (ICMLuaUtil)

### Hash-Based API Lookup

The reflective loader resolves API functions using precomputed hash values instead of string names, hiding sensitive strings in the binary:

```c
#define KERNEL32DLL_HASH            0x6A4ABC5B
#define LOADLIBRARYA_HASH           0xEC0E4E8E
#define GETPROCADDRESS_HASH         0x7C0DFCAA
#define VIRTUALALLOC_HASH           0x91AFCA54
```

### PE Relocation

When the DLL is loaded at a different base address, all relative addresses must be corrected.
The loader processes the PE relocation table to apply these adjustments.

## License

This project is licensed under the MIT License. For more information, see the [LICENSE file](LICENSE).