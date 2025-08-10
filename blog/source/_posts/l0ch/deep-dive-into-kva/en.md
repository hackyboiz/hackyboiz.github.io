---
title: "[Research] Deep Dive into KVA Shadow's Dynamic Activation Mechanism(En)"
author: L0ch
tags: [L0ch, windows, KVAS, mitigation, bypass, exploit techniques]
categories: [Research]
date: 2025-08-10 21:00:00
cc: false
index_img: 2025/08/10/l0ch/deep-dive-into-kva/en/thumbnail.png
---

Hello, this is L0ch!  It's been a while since my last post.

Last week's topic was [[Research] Bypassing Windows Kernel Mitigations: Part 0 - Deep Dive into KASLR Leaks Restriction](https://hackyboiz.github.io/2025/04/13/l0ch/bypassing-kernel-mitigation-part0/ko/). I mentioned that one of the conditions for the KASLR bypass PoC to work is that KVA Shadow is disabled!

At the time of writing, I had assumed that KVAS was disabled by default in the latest Windows 11, but through feedback, I learned that KVAS is dynamically activated based on CPU model and specific vulnerabilities, rather than Windows version.

# **SpeculationControl PowerShell script**

KVAS is closely related to various speculative execution-based side-channel vulnerabilities, such as Meltdown and Spectre, so you can easily check the relevant MS support information.

https://support.microsoft.com/en-us/topic/kb4074629-understanding-speculationcontrol-powershell-script-output-fd70a80a-a63f-e539-cda5-5be4c9e67c04

The results of testing on my computer with a 12th generation Intel CPU are summarized as follows.

```powershell
# Windwos 11 24H2 - Intel 12th i7-12700

PS C:\WINDOWS\system32> Install-Module -Name SpeculationControl                                                                                                                                                                                 need a NuGet provider to continue.

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
Although the KVA Shadow feature exists in Windows,(KVAShadowWindowsSupportPresent: True) 
it is not necessary.(KVAShadowRequired: False) 
so not activated(KVAShadowWindowsSupportEnabled: False)
```

We can see that KVAS is disabled, which means that although the mitigation feature exists, **the hardware (CPU) is not vulnerable, so the mitigation feature is disabled**.

So where and how does Windows identify vulnerable CPUs and determine whether to enable KVAS?

# Background Knowledge - CPU Identification

Windows identifies the CPU vendor on its own. As defined in the [CPU_VENDORS](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/amd64_x/cpu_vendors.htm) Enum, unknown manufacturers are defined as 0, AMD as 1, and Intel as 2.

```jsx
typedef enum
{
    CPU_UNKNOWN,   // 0
    CPU_AMD,       // 1
    CPU_INTEL,     // 2
    CPU_VIA        // 3
} CPU_VENDORS;
```

CPU information such as vendor, Faily ID, and Model ID is stored in the [KPRCB](https://www.vergiliusproject.com/kernels/x64/windows-11/24h2/_KPRCB) structure. 

These IDs are defined by each CPU vendor. In the case of Intel, the Family ID, Model ID, and generation information for Intel CPU products are defined in the [Intel Developer Manual](https://www.intel.com/content/www/us/en/content-details/782158/). Since it's not feasible to review the entire 5,000-page manual, I used gemini to summarize the information in the table below.

| **MircoArchitecture(gen)** | Processor family | **Family ID (Hex)** | **Model ID (Hex)** |
| --- | --- | --- | --- |
| Raptor Lake (13 gen) | Intel Core | 06H | B7H, BFH |
| Alder Lake (12 gen) | Intel Core | 06H | 97H, 9AH |
| Tiger Lake (11 gen) | Intel Core | 06H | 8CH, 8DH |
| Rocket Lake (11 gen) | Intel Core | 06H | A7H |
| Ice Lake (10 gen) | Intel Core | 06H | 7DH, 7EH |
| Comet Lake (10 gen) | Intel Core | 06H | A5H, A6H |
| Amber Lake Y (8 gen) | Intel Core | 06H | 8EH |
| Whiskey Lake U (8 gen) | Intel Core | 06H | 8EH |
| Coffee Lake (8, 9 gen) | Intel Core | 06H | 9EH |
| Kaby Lake (7 gen) | Intel Core | 06H | 8EH, 9EH |
| Skylake (6 gen) | Intel Core | 06H | 4EH, 5EH, 55H |
| Broadwell (5 gen) | Intel Core | 06H | 3DH, 47H, 4FH |
| Haswell (4 gen) | Intel Core | 06H | 3CH, 45H, 46H, 3FH |
| Ivy Bridge (3 gen) | Intel Core | 06H | 3AH |
| Sandy Bridge (2 gen) | Intel Core | 06H | 2AH |
| Westmere (2010) | Intel Core | 06H | 25H, 2CH, 2FH |
| Nehalem (1 gen) | Intel Core | 06H | 1AH, 1EH, 1FH, 2EH |
| Penryn | Core 2 Duo/Quad | 06H | 17H, 1DH |
| Merom | Core 2 Duo | 06H | 0FH |
| Yonah | Core Duo/Solo | 06H | 0EH |

I also organized and analyzed the ID lists of AMD and other vendors.

# Analysis of KVAS Activation Logic

Analysis Environment: Windows 24H2 64-bit 10.0.26100.1742

Returning to the topic at hand, in order to analyze the mitigation initialization logic, we need to examine the part of the Windows boot process that initializes the kernel. 

The entry point of the Windows kernel image `ntoskrnl.exe` is `KiSystemStartup`, which performs various tasks such as CPU initialization, mitigation-related feature settings, and kernel debugger initialization.

The function that determines whether KVAS is enabled was surprisingly easy to find.

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
    KiEnableKvaShadowing(CurrentPrcb);              //  Decide whether to enable KVA Shadow
    CurrentPCB_Number = CurrentPrcb->Number;
  }
  if ( KiKvaShadow )                             // When the KVA Shadow flag is enabled
  {
    v27 = KiSystemCall32Shadow;                 // instead of KiSystemCall32/KiSystemCall64  
    v28 = KiSystemCall64Shadow;                 // use KiSystemCall32Shadow/KiSystemCall64Shadow
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

In the above call chain, the `KiDetectKvaLeakage` function is the core function that determines whether KVA is enabled.

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

The `KiDetectKvaLeakage` function determines whether KVAS is enabled based on CPU vendor and model information included in KPRCB and information that can be read from MSR. The verification routine can be divided into three main parts.

### Check 1

- Check whether affected by Branch Confusion vulnerability through the `KiIsKvaShadowNeededForBranchConfusion` function
- If the call result of the function is affected by Branch Confusion vulnerability, proceed to the `ENABLE_KVAS_BRANCH_CONFUSION` branch to activate KVAS

### Check 2

- Filter out older CPUs that are not affected by Meltdown and do not require KVAS activation using a bitmask, and disable KVAS.
- Furthermore, since Meltdown is a vulnerability that only affects Intel processor architectures, disable KVAS for other vendors as well.

### Check 3

- First, check whether the CPU supports the MSR (Model Specific Register) function.
    - If MSR is not supported, proceed to the `ENABLE_KVAS` branch to activate KVAS, as it is an older CPU affected by Meltdown.
    - Models that do not support MSR but are not affected are first filtered through a bitmask in the Check 2 routine.
- Read the MSR address `0x10A` using `__readmsr` - [IA32_ARCH_CAPABILITIES](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/cpuid-enumeration-and-architectural-msrs.html)
    - If field 0 (`RDCL_NO`) of the result (`_RAX`) is not enabled, proceed to the `ENABLE_KVAS` branch to enable KVAS
    - RDCL_NO: A bit flag indicating immunity to Rogue Data Cache Load (Meltdown)

If KVAS is determined to be necessary through the above verification process, `KiKvaLeakage` is set to 1.

The main functions of this function can be summarized as identifying Intel processors affected by the Meltdown vulnerability in Check 2 and Check 3 and setting the KVAS activation flag. Check 1 will be explained in more detail below.

## KVAS Activation for Branch Confusion

KVAS is not a mitigation that only applies to Meltdown. You can confirm this with the function `KiIsKvaShadowNeededForBranchConfusion` called in the Check 1 routine.

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

As the function name suggests, this function checks whether the CPU is affected by the Branch Confusion vulnerability and determines whether KVA Shadow activation is necessary.

- If the `0x8000` bit of `v4`, the result of calling the `KiDetectHardwareSpecControlFeatures` function, is 1, the CPU is affected by Branch Confusion.
- If the relevant mitigation is supported, it returns True. - It enters the KVAS activation routine from the `KiDetectKvaLeakage` function.

Let's take a closer look at the `KiDetectHardwareSpecControlFeatures` function.

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

The function contains logic to check whether it is affected by various speculative execution-based side-channel vulnerabilities, but since the code is too long to write in its entirety, we will only look at the code that sets the `0x8000` bit to enable KVAS.

- For a list of speculative execution-related vulnerabilities, refer to [MS Client Guidance](https://support.microsoft.com/en-us/topic/kb4073119-windows-client-guidance-for-it-pros-to-protect-against-silicon-based-microarchitectural-and-speculative-execution-side-channel-vulnerabilities-35820a8a-ae13-1299-88cc-357f104f5b11)

If the return value of the `KiIsBranchConfusionPresent` function is True, the `0x8000` bit is set.

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

The function logic can be analyzed as follows.

1. If CpuVendor is not AMD, return False.
    - Branch Confusion is a vulnerability that affects only AMD CPUs ([CVE-2022-23825](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2022-23825))
2. If the Hypervisor is enabled, the `HviIsAnyHypervisorPresent` function returns True
    - CVE-2022-23825 is a vulnerability that affects virtualized environments, so KVAS is implemented to activate only when the hypervisor is enabled.
3. Returns True only if the Family Number is 25 for Zen 3 / Zen 3+ / Zen 4.
    - It can be seen that other architectures are not affected.

After that, there is a final check before KVAS activation in `KiEnableKvaShadowing`.

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

- `if ( KiKvaLeakage || KiKvaLeakageSimulate )` branch
    - Global variable `KiKvaLeakage` set in the `KiDetectKvaLeakage` function
    - Global variable `KiKvaLeakageSimulate`, which is estimated to be determined by manual settings (for debugging and testing)
- If either of the two global variable values is set, KVAS is activated

# Three-line summary

- The Windows kernel dynamically activates KVAS (and other mitigations) at initialization depending on the CPU.
- There are two main criteria for determining whether to activate KVAS
    - Intel: Rogue Data Cache Load (Meltdown)
    - AMD: Branch Confusion
- The CPU checks whether hardware mitigations provided through MSR are applied, and if MSR functionality is not available, it determines whether to activate KVAS through legacy checks (bitmask, model number information, etc.).

# Outro
![Untitled](en/image1.jpg)
Actually, when side-channel vulnerabilities based on speculative execution were a hot topic, I was busy fulfilling my military service obligations and didn't have a chance to look into it properly. However, while writing this article and the previous one on KASLR bypass, I had the opportunity to examine it in detail. I realized once again that understanding past issues is just as important as keeping up with current trends.  ~~Now, I'm going to try reading Windows Internals again...~~

I’ll bring you another interesting research topic next time~

# Reference
https://support.microsoft.com/en-us/topic/kb4073119-windows-client-guidance-for-it-pros-to-protect-against-silicon-based-microarchitectural-and-speculative-execution-side-channel-vulnerabilities-35820a8a-ae13-1299-88cc-357f104f5b11

https://msrc.microsoft.com/update-guide/vulnerability/CVE-2022-23825

https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/technical-documentation/cpuid-enumeration-and-architectural-msrs.html

https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/ntos/amd64_x/cpu_vendors.htm

https://support.microsoft.com/en-us/topic/kb4074629-understanding-speculationcontrol-powershell-script-output-fd70a80a-a63f-e539-cda5-5be4c9e67c04

https://www.intel.com/content/www/us/en/content-details/782158/intel-64-and-ia-32-architectures-software-developer-s-manual-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html

https://www.vergiliusproject.com/kernels/x64/windows-11/24h2/_KPRCB