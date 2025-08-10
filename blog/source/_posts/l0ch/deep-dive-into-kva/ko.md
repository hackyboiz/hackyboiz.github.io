---
title: "[Research] Deep Dive into KVA Shadow's Dynamic Activation Mechanism(Ko)"
author: L0ch
tags: [L0ch, windows, KVAS, mitigation, bypass, exploit techniques]
categories: [Research]
date: 2025-08-10 21:00:00
cc: false
index_img: 2025/08/10/l0ch/deep-dive-into-kva/ko/thumbnail.png
---

안녕하세요, L0ch입니다!  지난 번 글 이후로 오랜만에 돌아왔습니다

지난 주제는 [[Research] Bypassing Windows Kernel Mitigations: Part0 - Deep Dive into KASLR Leaks Restriction](https://hackyboiz.github.io/2025/04/13/l0ch/bypassing-kernel-mitigation-part0/ko/) 였었는데요, KASLR 우회 PoC의 동작 조건 중 하나가 KVA Shadow 비활성화다! 라는 이야기를 했었습니다.

작성 당시에는 최신 Windows 11에서는 KVAS가 기본적으로 비활성화되어 있다고 추측했었는데, 피드백을 통해 KVAS는 Windows 버전 기준이 아닌 CPU 모델과 특정 취약점에 영향을 받는지 확인하고 동적으로 활성화된다는 것을 알았습니다.

# **SpeculationControl PowerShell script**

KVAS는 Meltdown, Spectre로 대표되는 다양한 추측 실행 기반 사이드채널 취약점과 밀접한 관련이 있는 만큼, 관련 MS의 지원 정보를 쉽게 확인할 수 있습니다.

https://support.microsoft.com/en-us/topic/kb4074629-understanding-speculationcontrol-powershell-script-output-fd70a80a-a63f-e539-cda5-5be4c9e67c04

제 컴퓨터인 12세대 Intel CPU로 테스트한 결과를 정리하면 다음과 같습니다.
```powershell
# Windwos 11 24H2 - Intel 12th i7-12700

PS C:\WINDOWS\system32> Install-Module -Name SpeculationControl                                                                                                                                                                                 계속하려면 NuGet 공급자가 필요합니다.

PS C:\WINDOWS\system32> Get-SpeculationControlSettings

BTIHardwarePresent                  : True
BTIWindowsSupportPresent            : True
BTIWindowsSupportEnabled            : True
BTIDisabledBySystemPolicy           : False
BTIDisabledByNoHardwareSupport      : False
BTIKernelRetpolineEnabled           : False
BTIKernelImportOptimizationEnabled  : True
RdclHardwareProtectedReported       : True
RdclHardwareProtected               : True
KVAShadowRequired                   : False
KVAShadowWindowsSupportPresent      : True
KVAShadowWindowsSupportEnabled      : False
KVAShadowPcidEnabled                : False
SSBDWindowsSupportPresent           : True
SSBDHardwareVulnerable              : True
SSBDHardwarePresent                 : True
SSBDWindowsSupportEnabledSystemWide : False
L1TFHardwareVulnerable              : False
L1TFWindowsSupportPresent           : True
L1TFWindowsSupportEnabled           : False
L1TFInvalidPteBit                   : 0
L1DFlushSupported                   : True
HvL1tfStatusAvailable               : True
HvL1tfProcessorNotAffected          : True
MDSWindowsSupportPresent            : True
MDSHardwareVulnerable               : False
MDSWindowsSupportEnabled            : False
FBClearWindowsSupportPresent        : True
SBDRSSDPHardwareVulnerable          : False
FBSDPHardwareVulnerable             : False
PSDPHardwareVulnerable              : False
FBClearWindowsSupportEnabled        : False

```


| Hardware requires kernel VA shadowing | Maps to KVAShadowRequired. This line tells you whether your system requires kernel VA shadowing to mitigate a vulnerability. |
| --- | --- |
| Windows OS support for rogue data cache load mitigation is present | Maps to KVAShadowWindowsSupportPresent. This line tells you whether Windows operating system support for the kernel VA shadow feature is present. |
| Windows OS support for kernel VA shadow is present | Maps to KVAShadowWindowsSupportPresent. This line tells you whether Windows operating system support for the kernel VA shadow feature is present. If it is True, the January 2018 update is installed on the device, and kernel VA shadow is supported. If it is False, the January 2018 update is not installed, and kernel VA shadow support does not exist. |
| Windows OS support for rogue data cache load mitigation is enabled | Maps to KVAShadowWindowsSupportEnabled. This line tells you whether the kernel VA shadow feature is enabled. If it is True, the hardware is believed to be vulnerable to CVE-2017-5754, Windows operating system support is present, and the feature is enabled. |
| Windows OS support for kernel VA shadow is enabled | Maps to KVAShadowWindowsSupportEnabled. This line tells you whether the kernel VA shadow feature is enabled. If it is True, Windows operating system support is present, and the feature is enabled. The Kernel VA shadow feature is currently enabled by default on client versions of Windows and is disabled by default on versions of Windows Server. If it is False, either Windows operating system support is not present, or the feature is not enabled. |
```cpp
Windows에 KVA Shadow 기능은 존재하나(KVAShadowWindowsSupportPresent: True) 필요하지 않아서(KVAShadowRequired: False) 활성화되지는 않은 상태 (KVAShadowWindowsSupportEnabled: False)
```
KVAS가 비활성화된 걸 알 수 있는데, 이는 다시 말하면 미티게이션 기능이 존재하지만 **하드웨어(CPU)가 취약하지 않아서 미티게이션 기능이 비활성화되었다**고 말할 수 있겠네요. 

그렇다면 Windows는 어디서, 어떻게 취약한 CPU를 식별하고 KVAS 활성화 여부를 판단할까요?

# Background Knowledge - CPU Identification

Windows는 자체적으로 CPU의 벤더를 식별합니다. [CPU_VENDORS](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/amd64_x/cpu_vendors.htm) Enum에 정의되어 있듯이, 알 수 없는 제조사는 0, AMD는 1, Intel은 2로 정의되어 있죠.

```jsx
typedef enum
{
    CPU_UNKNOWN,   // 0
    CPU_AMD,       // 1
    CPU_INTEL,     // 2
    CPU_VIA        // 3
} CPU_VENDORS;
```

벤더뿐만 아니라 Faily ID, Model ID 와 같은 CPU 정보들은 [KPRCB](https://www.vergiliusproject.com/kernels/x64/windows-11/24h2/_KPRCB) 구조체에 저장됩니다. 

이러한 ID들은 각 CPU 벤더사가 정의합니다. Intel의 경우 Intel CPU 제품의 Family ID, Model ID, 세대 정보를 [Intel Developer Manual](https://www.intel.com/content/www/us/en/content-details/782158/intel-64-and-ia-32-architectures-software-developer-s-manual-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html) 에서 확인할 수 있는데요, 5000장이 넘는 매뉴얼을 전부 볼 수는 없으니.. gemini 시켜서 아래 표로 정리했습니다. 

| **마이크로아키텍처 (세대)** | **프로세서 제품군** | **Family ID (Hex)** | **Model ID (Hex)** |
| --- | --- | --- | --- |
| Raptor Lake (13세대) | Intel Core | 06H | B7H, BFH |
| Alder Lake (12세대) | Intel Core | 06H | 97H, 9AH |
| Tiger Lake (11세대) | Intel Core | 06H | 8CH, 8DH |
| Rocket Lake (11세대) | Intel Core | 06H | A7H |
| Ice Lake (10세대) | Intel Core | 06H | 7DH, 7EH |
| Comet Lake (10세대) | Intel Core | 06H | A5H, A6H |
| Amber Lake Y (8세대) | Intel Core | 06H | 8EH |
| Whiskey Lake U (8세대) | Intel Core | 06H | 8EH |
| Coffee Lake (8, 9세대) | Intel Core | 06H | 9EH |
| Kaby Lake (7세대) | Intel Core | 06H | 8EH, 9EH |
| Skylake (6세대) | Intel Core | 06H | 4EH, 5EH, 55H |
| Broadwell (5세대) | Intel Core | 06H | 3DH, 47H, 4FH |
| Haswell (4세대) | Intel Core | 06H | 3CH, 45H, 46H, 3FH |
| Ivy Bridge (3세대) | Intel Core | 06H | 3AH |
| Sandy Bridge (2세대) | Intel Core | 06H | 2AH |
| Westmere (2010) | Intel Core | 06H | 25H, 2CH, 2FH |
| Nehalem (1세대) | Intel Core | 06H | 1AH, 1EH, 1FH, 2EH |
| Penryn | Core 2 Duo/Quad | 06H | 17H, 1DH |
| Merom | Core 2 Duo | 06H | 0FH |
| Yonah | Core Duo/Solo | 06H | 0EH |

AMD나 다른 벤더들의 ID 리스트도 모두 정리하고 분석을 진행했습니다.

# Analysis of KVAS Activation Logic

분석 환경: Windows 24H2 64bit 10.0.26100.1742 

다시 돌아와서, 미티게이션 초기화 로직을 분석하기 위해서는 Windows 부팅 시 커널을 초기화하는 부분까지 들어가야 합니다.  

Windows 커널 이미지 `ntoskrnl.exe`의 Entry Point는 `KiSystemStartup` 으로, CPU 초기화부터 미티게이션 관련된 Feature 세팅 수행, 커널 디버거 초기화 등 다양한 역할을 수행합니다.

KVAS 활성화 여부를 결정하는 함수는 의외로 금방 찾을 수 있었습니다.

- KiSystemStartup → KiInitializeBootStructures → KiSetFeatureBits → KiDetectKvaLeakage

```cpp
NTSTATUS __stdcall __noreturn KiSystemStartup(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
//...
  KeLoaderBlock_0 = (__int64)DriverObject;
  if ( !*((_DWORD *)DriverObject->MajorFunction[3] + 9) )
    KasanInitSystem(DriverObject, 0i64);
  if ( !*(_DWORD *)(*(_QWORD *)(KeLoaderBlock_0 + 136) + 36i64) )
    KdInitSystem(0xFFFFFFFFi64, KeLoaderBlock_0);
  v2 = *(unsigned int **)(KeLoaderBlock_0 + 136);

//...

  KiInitializeBootStructures(KeLoaderBlock_0);  // Initialize Boot Structures
  if ( !*MK_FP(43, *MK_FP(43, KeLoaderBlock_0 + 136) + 36i64) )
    KdInitSystem(0i64, KeLoaderBlock_0);
//...

}

__int64 __fastcall KiInitializeBootStructures(__int64 a1)
{
  KPCR *Pcr; // r14
  _KPROCESS **v2; // rbx
  struct _KPRCB *CurrentPrcb; // rdi
//...

  if ( !KeGetPcr()->Prcb.Number )
    KiInitializeNxSupportDiscard(v19, v18, v20);
  HalInitializeProcessor(Number, a1, v20);
  KiSetFeatureBits(CurrentPrcb);                // Set Prcb Feature Bits
  CurrentPCB_Number = CurrentPrcb->Number;
  v27 = KiSystemCall32;
  v28 = KiSystemCall64;
  if ( !CurrentPCB_Number )
  {
    KiEnableKvaShadowing(CurrentPrcb);              // KVA Shadow 활성화 여부를 결정
    CurrentPCB_Number = CurrentPrcb->Number;
  }
  if ( KiKvaShadow )                             // KVA Shadow 플래그가 활성화되면
  {
    v27 = KiSystemCall32Shadow;                 // KiSystemCall32/KiSystemCall64 대신 
    v28 = KiSystemCall64Shadow;                 // KiSystemCall32Shadow/KiSystemCall64Shadow 사용
  }
  
//...
}

__int64 __fastcall KiSetFeatureBits(_KPRCB *CurrentPRCB)
{
  char CpuType; // bl
  unsigned int CpuModel; // ecx
  unsigned __int8 CpuVendor; // dl
  unsigned int ProcessorSignature; // eax
  unsigned __int8 CpuStepping; // cl
  unsigned __int8 v57; // al
  unsigned __int64 v58; // rcx
//...

  CpuType = CurrentPRCB->CpuType_Family;        // CpuType = Processor Family
  CpuModel = CurrentPRCB->CpuModel;
  CpuVendor = CurrentPRCB->CpuVendor;
  v123 = (CpuVendor - 1) <= 1u;
  if ( CurrentPRCB->Number )
  {
    ProcessorSignature = KiGetProcessorSignature(0i64, 0i64, 0i64, 0i64);
    KiSetProcessorSignature(CurrentPRCB, ProcessorSignature);
    goto LABEL_40;
  }

//... 
  KiDetectKvaLeakage(CurrentPRCB);      // Detect - KVA needs to be activated
  _m_prefetchw(CurrentPRCB);
  if ( CurrentPRCB->CpuVendor == 1 )
  {
    v52 |= 0x100000u;
    HIDWORD(v134) = v52;
  }
//...
}
 
```

## KVAS Activation for Meltdown

위 호출 체인에서 `KiDetectKvaLeakage` 함수가 바로 KVA 활성화 여부를 결정하는 코어 함수입니다.

```cpp
int __fastcall KiDetectKvaLeakage(_KPRCB *a1)
{
  __int64 p_CpuVendor; // rsi
  unsigned __int64 _RAX; // rax
  __int64 v4; // rcx
  __int64 _RAX; // rax
  __int64 _RAX; // rax
  __int64 _RDX; // rdx
  unsigned __int64 v13; // rdx
  bool v14; // zf
  int *v15; // rdx
  int v16; // ecx
  __int64 v17; // rbx
  __int64 _RAX; // rax
  __int64 _RAX; // rax
  __int64 _RAX; // rax
  unsigned int Number; // edx
  int v29[6]; // [rsp+30h] [rbp-20h] BYREF

  v29[0] = 0;
  p_CpuVendor = &a1->CpuVendor;
  
  // Check 1
  LODWORD(_RAX) = KiIsKvaShadowNeededForBranchConfusion(a1);
  if ( _RAX )
    goto ENABLE_KVAS_BRANCH_CONFUSION;
  LODWORD(_RAX) = *p_CpuVendor;
  
  // Check 2
  // Intel - Check with bitmask
  if ( *p_CpuVendor == 2 )
  {
    _RAX = a1->CpuModel;
    if ( a1->CpuType == 6 && _RAX <= 0x36u )
    {
      v4 = 0x6000C010000000i64;
      if ( _bittest64(&v4, _RAX) )
        return _RAX;
    }
  }
  // Others - disable KVAS
  else if ( _RAX != 3 || a1->CpuType == 6 && a1->CpuModel == 13)
  {
    return _RAX;
  }
	
	// Check 3
	// IA32_ARCH_CAPABILITIES support query
  _RAX = 0i64;
  __asm { cpuid }
  if ( _RAX < 7 )
    goto ENABLE_KVAS;
  _RAX = 7i64;
  __asm { cpuid }
  if ( (_RDX & 0x20000000) == 0 )
    goto ENABLE_KVAS;

	// Check 3
	// msr address 0x10A - IA32_ARCH_CAPABILITIES
  _RAX = __readmsr(0x10Au);
  //IA32_ARCH_CAPABILITIES bit 0 : RDCL_NO
  if ( (_RAX & 1) == 0 )
    goto ENABLE_KVAS;
 
  KiMicrocodeTrackerEnabled = 1;
  LODWORD(_RAX) = 3670016;
  LOBYTE(v13) = (KeFeatureBits2 & 0x28) == 8;
  if ( (KeFeatureBits2 & 0x380000) != 3670016 )
  {
    LODWORD(_RAX) = KiIsFbClearSupported(KeFeatureBits2 & 0x380000, v13);
    LOBYTE(v13) = _RAX | v13;
  }
  if ( v13 )
  {
ENABLE_KVAS:
    if ( a1->Number && !KiKvaLeakage )
      KeBugCheckEx(0x5Du, 0x4B56414Cui64, 0i64, 0i64, 0i64);
ENABLE_KVAS_BRANCH_CONFUSION:
    v14 = *p_CpuVendor == 2;
    **KiKvaLeakage = 1;**
//... for enable KVA
  }
  return _RAX;
}
```

`KiDetectKvaLeakage` 함수는 KPRCB에 포함된 CPU 벤더 및 모델 정보와, MSR을 읽어 확인할 수 있는 정보를 통해 KVAS 활성화 여부를 결정합니다. 확인하는 루틴은 크게 세 가지로 나눌 수 있습니다.

### Check 1

- `KiIsKvaShadowNeededForBranchConfusion`함수를 통해 Branch Confusion 취약점에 영향받는지 확인
- 해당 함수의 호출 결과 Branch Confusion 취약점 영향을 받는 경우 KVAS를 활성화하는 `ENABLE_KVAS_BRANCH_CONFUSION` 분기로 진행

### Check 2

- Meltdown에 영향받지 않아 KVAS 활성화가 필요 없는 구형 CPU를 bitmask를 통해 필터링하고 KVAS를 비활성화
- 또한, Meltdown은 Intel 프로세서 아키텍쳐에만 영향을 주는 취약점이므로 때문에 그 외 벤더에 대해서도 KVAS를 비활성화

### Check 3

- 먼저 CPU가 MSR(Model Specific Register) 기능을 지원하는지 확인
    - MSR을 지원하지 않으면 Meltdown에 영향을 받는 구형 CPU이므로 KVAS 활성화하는 `ENABLE_KVAS` 분기로 진행
    - MSR을 지원하지 않지만 영향을 받지 않는 모델은 Check 2 루틴에서 bitmask를 통해 먼저 필터링
- `__readmsr`을 통해 MSR 주소 `0x10A` 읽기 - [IA32_ARCH_CAPABILITIES](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/cpuid-enumeration-and-architectural-msrs.html)
    - 결과(`_RAX`)의 필드 0 (`RDCL_NO`) 가 활성화되지 않았으면 KVAS 활성화하는 `ENABLE_KVAS`분기로 진행
    - RDCL_NO: Rogue Data Cache Load(Meltdown) 에 취약하지 않음을 나타내는 비트 플래그

위 확인 과정을 통해 KVAS가 필요하다고 판단되면 `KiKvaLeakage`를 1로 설정합니다. 

해당 함수의 주요 기능은 Check 2와 Chekc3의 Meltdown 취약점에 영향받는 Intel 프로세서를 식별하고 KVAS 활성화 플래그를 설정하는 것으로 요약할 수 있겠네요. Check 1은 아래에서 더 자세하게 설명하겠습니다.

## KVAS Activation for Branch Confusion

KVAS는 Meltdown만 적용되는 미티게이션은 아닙니다. 그걸 확인할 수 있는 함수가 Check 1 루틴에서 호출하는 `KiIsKvaShadowNeededForBranchConfusion` 인데요.

```powershell
__int64 __fastcall KiIsKvaShadowNeededForBranchConfusion(__int64 a1)
{
  unsigned int v2; // ebx
  __int128 v4; // [rsp+20h] [rbp-28h] BYREF
  __int64 v5; // [rsp+30h] [rbp-18h]

  v5 = 0i64;
  v4 = 0i64;
  KiDetectHardwareSpecControlFeatures(a1, 0i64, (__int64)&v4, 0i64);
  if ( (v4 & 0x8000) == 0 )
    return 0i64;
  v2 = 0;
  if ( !(unsigned int)KiIsBranchConfusionMitigationDesired(a1, &v4) )
    return 0i64;
  LOBYTE(v2) = (unsigned int)KiIsBranchConfusionMitigationSupported(a1, &v4) != 0;
  return v2;
}
```

함수명에서도 알 수 있듯, Branch Confusion 취약점의 영향을 받는 CPU인지 확인하고 KVA Shadow 활성화가 필요한지 결정하는 함수입니다.

- `KiDetectHardwareSpecControlFeatures` 함수 호출 결과인 `v4` 의 `0x8000` 비트가 1이면 Branch Confusion의 영향을 받음
- 이후 관련 Mitigation이 지원되면 True 반환 - `KiDetectKvaLeakage` 함수에서 KVAS 활성화 루틴으로 진입함

`KiDetectHardwareSpecControlFeatures` 함수에서 좀 더 자세히 살펴보도록 하죠

```cpp
char *__fastcall KiDetectHardwareSpecControlFeatures(_KPRCB *PRCB, __int64 _zero, __int64 result2, char *__zero)
{
  int CpuModel_1; // r14d
  unsigned __int8 CpuVendor; // al
  char CpuType_Family; // r12
  bool IsAnyHypervisorPresent; // r9
  char CpuVendor_2; // bl
  unsigned int ProcessorFlags; // ecx
  unsigned __int8 CpuModel; // cl
  unsigned __int8 CpuStepping; // al
//...
  char *result; // rax
  unsigned __int8 CpuVendor_1; // [rsp+20h] [rbp-60h]
  
  
  CpuVendor = PRCB->CpuVendor;
  CpuType_Family = PRCB->CpuType_Family;
  LOBYTE(CpuModel_1) = PRCB->CpuModel;
//...
  CpuVendor_1 = CpuVendor;
  v48 = 1;
  
  // Hypervisor check
  if ( HviIsHypervisorMicrosoftCompatible() )
  {
    HviGetEnlightenmentInformation(&v54);
    v53 = 0i64;
    HviGetHypervisorFeatures(&v53);
    if ( (v53 & 0x100000000000i64) == 0 || (v54 & 0x1000) != 0 )
    {
      IsAnyHypervisorPresent = 1;
    }
    else
    {
      IsAnyHypervisorPresent = 0;
      v48 = 0;
    }
  }
  else
  {
    IsAnyHypervisorPresent = HviIsAnyHypervisorPresent();
    v48 = IsAnyHypervisorPresent;
  }
  v11 = result1;

  if ( KiIsBranchConfusionPresent(PRCB) )
  {
    v11 |= 0x8000ui64;
    *&result1 = v11;
  }
//...
  *result2 = result1;
  *(result2 + 16) = 4i64;
  result = __zero;
  if ( __zero )
    *__zero = v8;
  return result;
}
```

해당 함수에는 다양한 추측 실행 기반 side-channel 취약점에 영향을 받는지 확인하는 로직이 있지만, 코드를 다 쓰기에는 너무 기니까 KVAS 활성화를 결정하는 `0x8000` 비트를 설정하는 코드만 보겠습니다.

- 추측 실행 관련 취약점 목록은 [MS 클라이언드 가이던스](https://support.microsoft.com/en-us/topic/kb4073119-windows-client-guidance-for-it-pros-to-protect-against-silicon-based-microarchitectural-and-speculative-execution-side-channel-vulnerabilities-35820a8a-ae13-1299-88cc-357f104f5b11) 참고

`KiIsBranchConfusionPresent` 함수 반환값이 True면 `0x8000` 비트를 설정하네요.

`KiIsBranchConfusionPresent`

```jsx
__int64 __fastcall KiIsBranchConfusionPresent(_KPRCB *a1)
{
  bool IsAnyHypervisorPresent; // al
  unsigned int v3; // edx

  if ( a1->CpuVendor != 1 || (KeFeatureBits2 & 0x1000000) != 0 )
    return 0i64;
  IsAnyHypervisorPresent = HviIsAnyHypervisorPresent();
  v3 = 0;
  if ( IsAnyHypervisorPresent )
    return 1i64;
  LOBYTE(v3) = a1->CpuType_Family != 25;
  return v3;
}
```

함수 로직을 분석하면 다음과 같습니다.

1. CpuVendor가 AMD가 아니면 False 반환
    - Branch Confusion은 AMD CPU에만 영향을 주는 취약점([CVE-2022-23825](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2022-23825))
2. `HviIsAnyHypervisorPresent`함수를 통해 Hypervisor가 활성화되어 있으면 True 반환
    - CVE-2022-23825는 가상화 환경에 영향을 주는 취약점이므로 Hyperviosr가 활성화된 경우에만 KVAS를 활성화하도록 구현됨
3. Family Number가 Zen 3 / Zen 3+ / Zen 4 인 25 인 경우에만 True 반환
    - 그 외 아키텍처는 영향받지 않음을 알 수 있음

이후 `KiEnableKvaShadowing` 에서 KVAS 활성화 전 마지막 검사가 존재합니다.

```cpp
__int64 __fastcall KiEnableKvaShadowing(_KPRCB *a1)
{
  __int64 v2; // rdx
  __int64 v3; // rcx
  char v4; // cl
  unsigned __int64 v5; // rax
  __int64 v6; // rdx
  __int64 v7; // r11
  unsigned __int8 v8; // cf
  unsigned __int64 v9; // rax
  unsigned __int64 v10; // rax
  __int64 result; // rax
  unsigned __int16 v12; // cx

  if ( KiIsKvaShadowDisabled() )
  {
    KiIsKvaShadowConfigDisabled = 1;
  }
  else
  {
    if ( (KeFeatureBits2 & 0x18000) == 0x8000 )
      *(_QWORD *)(v3 + 11520) = 3i64;
    v4 = KiKernelCetEnabled;
    if ( !(_BYTE)KiKernelCetEnabled && (unsigned __int8)KiIsKvaLeakSimulated() )
      KiKvaLeakageSimulate = 1;
    
    // Enable KVAS
    if ( KiKvaLeakage || KiKvaLeakageSimulate )
    {
      if ( v4 )
        KeBugCheckEx(0x5Du, 0x4B766120ui64, 0x4B434554ui64, 0i64, 0i64);
      v5 = __readcr3();
      a1->KernelDirectoryTableBase = v5;
      *(_QWORD *)(v2 + 4216) = *(_QWORD *)(v2 + 4100);
      KiInitializeDescriptorIst(a1);
      *(_QWORD *)(v7 + 4100) = v7 + 16896;
      if ( a1->Number )
      {
        result = KiShadowProcessorAllocation(a1, v7);
        if ( !(_DWORD)result )
          return result;
        v12 = *(_WORD *)(KeGetPrcb(0i64) + 44714);
        a1->ShadowFlags |= 2u;
        a1->VerwSelector = v12;
      }
      else
      {
        LOBYTE(v6) = 1;
        KiInitializeIdt(v7, v6);
        KeGetCurrentThread()->ApcState.Process->AddressPolicy = 1;
        byte_140FCE0E0 = 1;
        _InterlockedOr(dword_140FCE57C, 0x4000u);
        KiSetAddressPolicy(1i64);
        v8 = _bittest64((const signed __int64 *)&a1->FeatureBits, 0x2Au);
        a1->VerwSelector = 24;
        if ( v8 )
        {
          v9 = __readcr4();
          __writecr4(v9 & 0xFFFFFFFFFFFDFF7Fui64 | 0x20000);
          v10 = __readcr3();
          __writecr3(v10 | 2);
          KiFlushPcid |= 1u;
        }
        if ( (a1->FeatureBits & 0x240000000000i64) == 0x240000000000i64 )
          KiFlushPcid |= 2u;
        HvlRescindEnlightenments();
        KiKvaShadow = 1;
        KiKvaShadowMode = 2 - (KiFlushPcid != 0);
      }
      if ( KiFlushPcid )
        _interlockedbittestandset64((volatile signed __int32 *)&a1->KernelDirectoryTableBase, 0x3Fui64);
    }
  }
  return 1i64;
}
```

- `if ( KiKvaLeakage || KiKvaLeakageSimulate )`  분기
    - `KiDetectKvaLeakage` 함수에서 설정한 전역변수`KiKvaLeakage`
    - 수동 설정값(디버깅 및 테스트용)을 통해 결정되는 것으로 추정되는 전역변수 `KiKvaLeakageSimulate`
- 두 전역변수 값 중 하나라도 설정되어 있으면 KVAS를 활성화

# 세줄요약

- Windows Kernel은 초기화 시 CPU에 따라 KVAS(및 다른 미티게이션)을 동적으로 활성화함.
- KVAS 활성화 여부를 결정하는 기준은 크게 두 가지
    - Intel: Rogue Data Cache Load(Meltdown)
    - AMD: Branch Confusion
- CPU가 MSR을 통해 제공하는 하드웨어 미티게이션 적용 여부를 확인하고, MSR 기능이 없으면 레거시 체크 (bitmask, 모델넘버 정보 등)를 통해 KVAS 활성화 여부를 결정함.

# 마무리하며
![Untitled](ko/image1.jpg)

사실 예전에 한창 추측 실행 기반의 사이드채널 취약점이 이슈될 때 국방의 의무를 수행하느라 제대로 파보질 못했는데요, 저번 KASLR bypass와 이번 글을 작성하면서 자세히 파볼 기회가 된 것 같습니다. 과거의 이슈를 제대로 파악하는 게 현재의 트렌드를 따라가는 것 만큼이나 중요하다는 것을 다시 한 번 느끼면서,  ~~이제 Windows Internals 완독을 다시 시도하는 걸로..~~

다음에도 재밌는 연구 주제 들고 오도록 하겠습니다~

# Reference
https://support.microsoft.com/en-us/topic/kb4073119-windows-client-guidance-for-it-pros-to-protect-against-silicon-based-microarchitectural-and-speculative-execution-side-channel-vulnerabilities-35820a8a-ae13-1299-88cc-357f104f5b11

https://msrc.microsoft.com/update-guide/vulnerability/CVE-2022-23825

https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/cpuid-enumeration-and-architectural-msrs.html

https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/amd64_x/cpu_vendors.htm

https://support.microsoft.com/en-us/topic/kb4074629-understanding-speculationcontrol-powershell-script-output-fd70a80a-a63f-e539-cda5-5be4c9e67c04

https://www.intel.com/content/www/us/en/content-details/782158/intel-64-and-ia-32-architectures-software-developer-s-manual-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html

https://www.vergiliusproject.com/kernels/x64/windows-11/24h2/_KPRCB