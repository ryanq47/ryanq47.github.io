---
title: Honey I Shrunk the Implant
date: 2026-06-05 00:00:01 +0600
categories: [Implant, Backdoor]
tags: [windows]     # TAG names should always be lowercase
---

# Honey I Shrunk the Implant:

## Table of Contents  
  
- [Introduction](#introduction)  
	- [Project Goals](#project-goals)
	- [Source](#1-source)
	- [Linking](#2-linking)
	- [Comparison to Control Group](#comparison-to-control-group)

- [Playing Blue Team](#playing-blue-team)
	- [Detections](#detections)

- [Conclusion](#conclusion--future-improvements)

- [Technical References](#technical-references)
	- [Local Setup](#local-setup) 
	- [Operations & Commands](#operations--commands)  

- [Resources](#resources)

- [Video Demo](#video-demo) 

# Introduction

I recently found myself wondering just how small a functional Windows payload could realistically be, so I decided to spend some time researching and building one from scratch. I went with a familiar pattern of an SMB backdoor, with some heavy inspiration from DoublePulsar.

## Project Goals
Going into this project, I had a few goals in mind:


1. Make a minimum viable concept of the smallest (reasonable) implant. This entails:
	- Writing it in Assembly (with no C runtime)
	- Avoiding Strings
	- Fun linker settings

2. Test Detections: See if the same behaviour in ASM gets flagged the same as in C. 
  
3. LARP as a nation state dev
	- The market blows right now and this is all I can do to not feel like I'm spinning my wheels

4. Revise & update functionality with lessons learned & improvements



## 1. Source
I’ve always heard the best (and smallest) droppers are written directly in assembly, so I took that route for this tool. My background is mostly in x86 ELF, so this was the perfect excuse to finally get hands-on with x64 Windows ASM. I did my best to minimize the instruction set, though I'm sure there are a few source-level optimizations/tricks that I missed. 

The tool's structure is pretty straightforward. I've broken down each core function with clear labels and sub-labels to keep it clean. I also made sure to comment any weird quirks, so walking through the logic should be a breeze.

By default, `nasm.exe` has `-Ox multipass optimization (default)` enabled, and messing with other flags didn't yield any size improvements.  


## 2. Linking
#### Resources:
- [MSVC Link.exe Documentation](https://learn.microsoft.com/en-us/cpp/build/reference/linker-options?view=msvc-170)
- [PE Format Docs](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)

Apart from writing in assembly, a lot of space was saved from messing with a few linker flags. 

### `/MERGE`:

Merge lead to the largest reduction, specifically by merging the `.rdata` section into the `.text` section (`/MERGE:.rdata=.text`)

| Implementation | Source File | Size (without merge flag) | Size (with merge flag) | Size Reduction (bytes) | Size Reduction (%) |
|---|---|---|---|---|---|
| DLL  | `main.dll.asm` | `3072` bytes |`2560` bytes | `512` bytes | **16.67%** |
| EXE | `main.exe.asm`| `3072` bytes |`2560` bytes | `512` bytes | **16.67%** |
| EXE (Debug) | `main.debug.asm`| `4096` bytes |`3584` bytes | `512` bytes | **16.67%** |

#### Why does merging shrink the binary?

Without the merge flag, there are a total of 5 headers in the binary, which of course includes `.rdata`. 

<img width="560" height="299" alt="ghidra_mmap_no_flags" src="https://github.com/user-attachments/assets/fb1f0a8b-c0a2-409b-951c-bd727b161267" />



`.rdata` shows a size of `0x400` (1024 bytes). There's a lot in there, but most importantly, the import table is located here.


Things get interesting with the merge flag enabled, as we again have 5 headers, but `.idata` is present instead of `.rdata`. Even weirder, it's only 512 bytes, and the other sections are still the same size:

<img width="679" height="307" alt="ghidra_mmap_merged_rdata" src="https://github.com/user-attachments/assets/ebad2d4a-c690-4ce0-be72-084f365f42ce" />

This raises a few questions:


#### Why did `.idata` appear?

The short version is that the IAT needs to be writeable* by the loader in order to resolve the functions the program needs. `.text`, by default, is *not* writeable, as it is executable. There's this nifty little feature you may have heard about before, Data Execution Prevention (DEP), meant to prevent execution from the stack. Due to that, having both a writeable, and executable section is a big no-no, so the import data gets its own section, with the appropriate permissions.

> **NOTE: When compiled, the IAT is read only, it's modified to RW at runtime to patch in the function addresses, then moved back to Read*
>
> *See this article for additional details:*
> https://devblogs.microsoft.com/oldnewthing/20221006-07/?p=107257

#### What was stripped from the binary?

As stated above, `.idata` is half the size that `.rdata` was. It appears to just be padding and fluff that was removed. The start of `.text` in the merged binary has some data that was previously in `.rdata`, as expected due to the merge:

<img width="1279" height="457" alt="ghidra_sxs_text_rdata" src="https://github.com/user-attachments/assets/ae82d430-8039-4b6e-8932-33ddacb9eab5" />


With the merged binary, the actual executable code starts right after the  section map. The non-merged binary, `.rdata` continues on with the `IMAGE_IMPORT_DESCRIPTOR`, and then has 395 instructions of padding, which is likely where a lot of space saving is coming from. 

> Padding is at address `0x140002275` ->`0x1400023ff` of the non-merged binary, it's too long for a full screenshot

<img width="1277" height="590" alt="ghidra_sxs_text_rdata_2" src="https://github.com/user-attachments/assets/44aa21de-fed6-4df4-8927-b38278f014cb" />

### `/FILEALIGN:X`:
- [MSVC-170 `/FILEALIGN`](https://learn.microsoft.com/en-us/cpp/build/reference/filealign?view=msvc-170)

The `/FILEALIGN` linker option lets you specify the alignment of sections written to your output file as a multiple of a specified size.

Initially, in addition to the merge flag, this appears lead to a generous file size reduction:

#### `/FILEALIGN:4`:

| Implementation | Source File | Size (without flag) | Size (with flag) | Size Reduction (bytes) | Size Reduction (%) |
|---|---|---|---|---|---|
| DLL  | `main.dll.asm` | `2560` bytes |`1800` bytes | `760` bytes | **-29.69%** |
| EXE | `main.exe.asm`| `2560` bytes |`1800` bytes | `760` bytes | **-29.69%** |
| EXE (Debug) | `main.debug.asm`| `3584` bytes |`2668` bytes | `916` bytes | **-25.56%** |

However, there's a big problem. Windows no longer recognizes this as a valid PE. There's a huge rabbithole to go down here, but in short, Window 10+ requires a minimum `/FILEALIGN` value of 512. (See PE Format Docs). 

In addition to `/FILEALIGN:4`, I tested 16, 32, and 64, all of which lead to 20-30% smaller file size, but also throw the following error upon execution:

```Powershell
Program 'x64_smb_backdoor.exe' failed to run: The specified executable is not a valid application for this OS platform.
```

You *can* go smaller than 512, and that requires setting the `/ALIGN` flag, which adds a significant amount of size to the binary, making this flag not helpful for our size goal.

#### `/ALIGN:16 /FILEALIGN:16` :

| Implementation | Source File | Size (without flag) | Size (with flag) | Size Addition (bytes) | Size Increase (%)  |
|---|---|---|---|---|---|
| DLL  | `main.dll.asm` | `2560` bytes |`4144` bytes | `1584` bytes | **+61.875%** |
| EXE | `main.exe.asm`| `2560` bytes |`4144` bytes | `1584` bytes | **+61.87%** |
| EXE (Debug) | `main.debug.asm`| `3584` bytes |`4768` bytes | `1184` bytes | **+33.03%** |

## Comparison to Control Group
As a sanity check, I did a control group with a C implementation of the implant. This is *not* the smallest you can get a C binary, as there are a lot of tricks to reduce the size of one. These numbers are from a compilation from `cl.exe`, with the default flags.  


| Implementation | Source File | Size | Size Addition (bytes from .ASM) | Size Increase (% from .ASM) |
|---|---|---|---|---|
| C | `main.c` | `89088` bytes  | `86528` bytes | **+3,380%** |

Yikes, big difference. 

# Playing Blue Team

## Detections

### CrowdStrike Sandbox:

| Implementation | Source File | Threat Score | Threat Level |
|---|---|---|---|
| EXE | `main.c` | 50/100  | Malicious |
| EXE | `main.exe.asm` | 50/100  | Malicious |
| DLL | `main.dll.asm` | 35/100  | Suspicious |

CS marks both EXE's as malicious, with the primary indicator being from "Machine Learning", specifically `win/malicious_confidence_90%` - whatever that means. 


#### ASM Exe

The ASM implementation gets away with only 3 "suspicious" indicators:

| Name | Classification | Source | Relevance | MITRE ATT&CK | Details |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sample detected by CS Static Analysis and ML with relatively high confidence | malicious | External System | 10/10 | N/A | CrowdStrike Static Analysis and ML (QuickScan) yielded detection: win/malicious_confidence_90% (W) |
| Imports suspicious APIs | suspicious | Static Parser | 1/10 | Native API (T1106) | DisconnectNamedPipe<br>WriteFile<br>ConnectNamedPipe<br>VirtualAlloc<br>Sleep |
| Monitors specific registry key for changes | suspicious | API Call | 4/10 | Query Registry (T1012) | "unknown.exe" monitors "HKLM\SOFTWARE\Microsoft\Ole" (Filter: 268435457; Subtree: 0)<br><br>"unknown.exe" monitors "\REGISTRY\USER\S-1-5-21-735145574-3570218355-1207367261-1001_Classes\Local Settings\Software\Microsoft" (Filter: 268435457; Subtree: 1) |
| Queries process mitigation policy via NtQueryInformationProcess | suspicious | API Call | 3/10 | System Checks (T1497.001) | "C:\unknown.exe" queried ProcessMitigationPolicy |

#### C Exe

While the C implementation has a whopping 7:


| Name | Classification | Source | Relevance | MITRE ATT&CK | Details |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sample detected by CS Static Analysis and ML with relatively high confidence | malicious | External System | 10/10 | N/A | CrowdStrike Static Analysis and ML (QuickScan) yielded detection: win/malicious_confidence_90% (W) |
| PE file contains unusual section name| suspicious | Static Parser | 10/10 | Software Packing (T1027.002) | "unknown.exe" has a section named ".fptable" |
| Imports suspicious APIs | suspicious | Static Parser | 1/10 | Native API (T1106) | GetCommandLineA<br>CreateFileW<br>GetModuleHandleExW<br>GetStartupInfoW<br>LoadLibraryExW<br>DisconnectNamedPipe<br>FindFirstFileExW<br>VirtualAlloc<br>WriteFile<br>ConnectNamedPipe<br>TerminateProcess<br>IsDebuggerPresent<br>GetCommandLineW<br>GetModuleHandleW<br>FindNextFileW<br>GetProcAddress<br>UnhandledExceptionFilter<br>GetModuleFileNameW<br>VirtualProtect |
| Monitors specific registry key for changes | suspicious | API Call | 4/10 | Query Registry (T1012) | "unknown.exe" monitors "HKLM\SOFTWARE\Microsoft\Ole" (Filter: 268435457; Subtree: 0)<br><br>"unknown.exe" monitors "\REGISTRY\USER\S-1-5-21-735145574-3570218355-1207367261-1001_Classes\Local Settings\Software\Microsoft" (Filter: 268435457; Subtree: 1) |
| Contains ability to retrieve the contents of the STARTUPINFO structure | suspicious | Hybrid Analysis Technology | 5/10 | Create or Modify System Process (T1543) | GetStartupInfoW@KERNEL32.dll at 11965-594-00407A60 |
| Queries process  mitigation policy via NtQueryInformationProcess | suspicious | API Call | 3/10 | System Checks (T1497.001) | "C:\unknown.exe" queried ProcessMitigationPolicy |
| Contains Ability to retrieve command-line string for the current process | suspicious | Hybrid Analysis Technology | 3/10 | Native API (T1106) | GetCommandLineA@KERNEL32.dll at 11965-792-004049D3 |
| Contains ability to retrieve the fully qualified path of module | suspicious | Hybrid Analysis Technology | 5/10 | Native API (T1106) | GetModuleFileNameW@KERNEL32.dll at 11965-561-0040674C |

#### ASM DLL

The DLL got away with a lower score, at 35/100. It had no high indicators, and 3 suspicious:

| Name | Source | Relevance | MITRE ATT&CK | Details |
| :--- | :--- | :--- | :--- | :--- |
| Imports Suspicious APIs | Static Parser | 1/10 | Native API (T1106) | Sleep<br>VirtualAlloc<br>ConnectNamedPipe<br>WriteFile<br>DisconnectNamedPipe |
| Found a  string that may be used as part of an injection method | File/Memory | 4/10 | Extra Window Memory Injection (T1055.011) | "Shell_TrayWnd" (Taskbar window class may be used to inject into explorer with the SetWindowLong method) in Source: 00000000-00004140.00000000.233363.02A10000.00000002.mdmp<br><br>"Progman" (Program manager) in Source: 00000000-00004140.00000000.233363.02A10000.00000002.mdmp |
| Sample  detected by CS Static Analysis and ML with relatively low confidence | External System | 10/10 | N/A | CrowdStrike Static Analysis and ML (QuickScan) yielded detection: win/malicious_confidence_60% (D) |

#### Detections Breakdown:

It appears that using ASM alone *can* help on the detections front. Despite the EXE implementations scoring the same overall, the ASM EXE implementation yielded fewer indicators than the C EXE implementation. At least one of these can be knocked out fairly easily, as doing your own function resolution should eliminate the 'Imports Suspicious APIs' indicator. 

Bypassing static detections is likely more straightforward with the ASM version as well. Reordering and/or substituting just a few instructions is usually enough to break the signature, and you completely eliminate the headache of a C compiler optimizing away the junk code you would otherwise use to achieve the same effect.

# Conclusion & Future Improvements

Final Size Breakdown

| Implementation | Language | Source File | Size | linker flags | Compiler/Assembler flags |
|---|---|---|---|---|---|
| EXE | ASM |`main.exe.asm` | `2560` bytes  | `/ENTRY:_start /SUBSYSTEM:CONSOLE /NODEFAULTLIB /MERGE:.rdata=.text` | `N/A - Default` |
| DLL | ASM |`main.dll.asm` | `2560` bytes  | `/DLL /ENTRY:DllMain /NODEFAULTLIB /MERGE:.rdata=.text` | `N/A - Default` |
| EXE | C |`main.c` | `89088` bytes  | `N/A - Default` | `N/A - Default` |

*(Note: The [Makefile](https://github.com/ryanq47/backdoor-buddies/blob/main/x64_smb_backdoor/Makefile) handles the linker flags used above.)*

To wrap things up:

- **Yes, a basic implant under 4KB is doable.** If I can put this together, you can bet nation-state actors are doing it infinitely better.
- **Detections *can* be marginally better** compared to a similar C implementation.
    - *Is the tradeoff worth the dev time?* That's your call.
- **C feels incredibly high-level** after working exclusively in ASM.

### Future Improvements & Next Steps
While functional, there is plenty of room for improvement in future iterations:

- **PIC (Position Independent Code) Transition:** Currently, the implant relies on standard Windows imports. Future versions could use PEB walking and dynamic function resolution (e.g., techniques like Hell's Gate/Halo's Gate) to be fully PIC. 
- **Security Features:** Adding encryption for the for the SMB pipe comms.
- **Additional Protocol Implementations:** SMB is great for a POC, however there's additional protocols (ICMP, etc) that might be quieter/deserve an implementation.  

None of these findings are shocking revelations, but I had a great time building it. I got to learn a lot more about Windows ASM and stumbled across some weird PE quirks in the process, so it's a win in my book.

Thanks for reading!

---
> *Note: Extra details on how to run this locally, some implementation details, and version info, are included below.*
# Technical References

## Local Setup:
A quick setup guide if you want to run this locally. 
### Development Environment
For future reference and troubleshooting, this project was built and tested using the following toolchain:
* **OS:** Windows 11 (25H2, Build 26200.8457)
* **Editor:** Visual Studio Code
* **Assembler:** NASM v2.16.03
* **NMAKE:** 14.50.35730.0
* **LINK.exe:** 14.50.35730.0
* **Python:** 3.13.4

### 1. Clone Repo:
`git clone https://github.com/ryanq47/backdoor-buddies`

### 2. Setup python: 
```
cd client
pip install .
```

### 3. Compile Implants

1. CD into build directory: `cd x64_smb_backdoor`
	> The makefile needs  to be called from this directory, it has relative pathing
2. Run MAKE  on desired implants:

| Implementation| Source File | Command | Output  File |
|---|---|---|---|
| DLL  | `main.dll.asm` | `nmake dll` | `x64_smb_backdoor.dll`|
| EXE | `main.exe.asm`|`nmake exe`  | `x64_smb_backdoor.exe`|
| EXE (Debug) | `main.debug.asm`|`nmake debug`  | `x64_smb_backdoor.debug.exe` |
| All of the above | N/A |`nmake`  | N/A | 


##  Operations & Commands

This outlines how to communicate with the implant.

### Command Reference

Quick reference for commands. See sections below for additional details.

| command | opcode | success response | opcode | failure response | opcode |
|---|---|---|---|---|---|
| `ping` | `0x00` | Ping Ok | `0x00` | N/A | N/A |
| `run` | `0x30` | Shellcode Executed | `0x30` | N/A | N/A |
| `kill` | `0x20` | *(Terminates)* | N/A | N/A | N/A |

---

## PING Command

Send a small message to the implant asking if it is still alive.

### Command Structure

| Offset | Field | Size | Windows Type | Description |
|---|---|---|---|---|
| `0x00` | `Command` | 1 byte | BYTE | The command to execute |

### Command

| Request  | Response | Translation | 
|---|---|--|
| `0x00` | `0x30` | "Ping Ok" |

---

## KILL Command

Tell the implant to exit.

### Command Structure

| Offset | Field | Size | Windows Type | Description |
|---|---|---|---|---|
| `0x00` | `Command` | 1 byte | BYTE | The command to execute |

### Command

| Request  | Response | Translation | 
|---|---|---|
| `0x20` | None - implant exits | N/A |

---

## EXEC Command

Run shellcode on the implant.

### Command Structure

| Offset | Field | Size | Windows Type | Description |
|---|---|---|---|---|
| `0x00` | `Command` | 1 byte | BYTE | The command to execute |
| `0x01`-`0x04` | `Size` | 4 bytes | DWORD | The length of the upcoming data payload. (Little Endian*) |
| `0x05` | `Exec Method` | 1 byte | BYTE | Which execution method to call when `EXEC`ing the payload (see Execution Modes) |
| `0x06+` | `Shellcode` | Variable | N/A | The remaining payload bytes. Max Size: `1018` bytes.** |

`*` Skipped network byte order here. It's the only multi-byte field supported by this implant, and adding `bswap` logic to revert to little-endian adds one extra instruction. See goal #1.

`**` Pipe buffer is `1024` bytes. `1018` = Pipe Buffer - Exec Method - Size - Command

### Command

| Request | Response |Translation | 
|---|---|---|
| `0x30<size><exec method><shellcode>` | `0x30`  | "Shellcode Ran"| 


### Execution Modes

| `Exec Method` | Bytes | Description | Sacrificial |
|---|---|---|---|
| Call | `0x00` | Executes shellcode via a `call` | Yes |
| Jump | `0x01` | Executes shellcode via a `jmp` | Yes |

> !! Note: Both execution methods are sacrificial and execute directly within the current thread !!
>
> If the shellcode crashes, corrupts the stack, or calls `ExitProcess`, the implant process will terminate as well.
>
> I may add a thread option in future iterations so the thread dies, instead of the implant. In reality, just choose 1 execution method (sacrificial or not) and stick with it to reduce size even further.

#### Examples

##### MSFVenom Calc Shellcode

Runs `calc.exe`

| Field | Value |
|---|---|
| Size | `14010000` |
| Exec Method | `00` |
| Shellcode | `fc4883e...6500` |


```text
301401000000fc4883e4f0e8c0000000415141505251564831d265488b5260488b5218488b5220488b7250480fb74a4a4d31c94831c0ac3c617c022c2041c1c90d4101c1e2ed524151488b52208b423c4801d08b80880000004885c074674801d0508b4818448b40204901d0e35648ffc9418b34884801d64d31c94831c0ac41c1c90d4101c138e075f14c034c24084539d175d858448b40244901d066418b0c48448b401c4901d0418b04884801d0415841585e595a41584159415a4883ec204152ffe05841595a488b12e957ffffff5d48ba0100000000000000488d8d0101000041ba318b6f87ffd5bbaac5e25d41baa695bd9dffd54883c4283c067c0a80fbe07505bb4713726f6a00594189daffd563616c632e65786500
```

# Resources

Code:
- [Github Repo](https://github.com/ryanq47/backdoor-buddies)
- [X64_SMB_BACKDOOR Assembly Source](https://github.com/ryanq47/backdoor-buddies/blob/main/x64_smb_backdoor/main.asm)

Tooling:
 - [Ghidra](https://github.com/nationalsecurityagency/ghidra) 

Documentation & Articles:
 - [MSVC Link.exe Documentation](https://learn.microsoft.com/en-us/cpp/build/reference/linker-options?view=msvc-170)
 - [MSVC /FILEALIGN Documentation](https://learn.microsoft.com/en-us/cpp/build/reference/filealign?view=msvc-170)
 - [PE Format Documentation](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)
 - [The Old New Thing: IAT Modification](https://devblogs.microsoft.com/oldnewthing/20221006-07/?p=107257)
 - [MSFT Docs](https://learn.microsoft.com)

# Video Demo

Note: If this is busted - the direct link is [here](https://github.com/ryanq47/backdoor-buddies/blob/main/x64_smb_backdoor/blog/images/demo.mp4)

<video width="100%" controls>
  <source src="https://github.com/user-attachments/assets/067c57ca-ea38-4429-941f-c2da3844447b" type="video/mp4">
</video>
