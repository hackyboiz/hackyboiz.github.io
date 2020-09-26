---
title: "[번역]Engineering Antivirus Evasion Part 1"
author: y2sman
tags: [y2sman, antivirus, evasion, string, obfuscation, libtools, clang, av-bypass]
date: 2020-09-07 19:00:00
cc: true
---



# [번역]Engineering Antivirus Evasion Part 1

[https://blog.scrt.ch/2020/06/19/engineering-antivirus-evasion/](https://blog.scrt.ch/2020/06/19/engineering-antivirus-evasion/)

tl;dr : 이 블로그 게시물은 안티바이러스 소프트웨어에 대한 우리 연구의 몇 가지 관점과 우리가 마주한 모든 AV/EDR을 우회하기 위해 자동적으로 Meterpreter를 리팩터링하는 것에 대해서 설명할 것이다.

모든 기술에 대한 아이디어와 문자열 난독화 패스의 구현은 아래에 자세히 설명되어 있지만, 블로그 게시글을 가능한 짧게 유지하기 위해 API imports hiding/syscalls에 대한 상세한 것은 나중에 작성하기로 결정했다. 소스코드는 [https://github.com/scrt/avcleaner](https://github.com/scrt/avcleaner)에서 확인할 수 있다.

기업이 정보 시스템에 대한 공격을 보호하기 위해 구현할 수 있는 방어 조치 중, 안티바이러스 혹은 EDR 같은 보안 소프트웨어는 종종 필수적인 도구로 등장한다. 과거에는 모든 종류의 멀웨어 탐지 기술을 우회하기가 다소 쉬웠지만, 요즘은 더 많은 노력이 필요하다.

반면에, 취약점을 증명하기 위한 Proof-of-Concept가 안티바이러스에 의해 차단된 경우 취약점의 위험성에 대해 의사 소통하는 것은 매우 어렵다. 항상 이론적으로 탐지를 우회하고 그것을 유지하는 것이 가능하다고 주장할 수 있지만, 사실 그것은 논의를 좀 더 강하게 할 수 있다.

게다가, 시스템의 existing foothold에서만 발견할 수 있는 취약점들이 있다. 예를 들어, pentester가 initial level에 접근할 수 없는 경우 검사 결과는 범위 내에 시스템의 실질적인 보안 범위를 나타내지 않는다.

이러한 관점에서, 안티바이러스 소프트웨어 우회는 필요하다. 문제를 복잡하게 만들기 위해 SCRT에서는 가능할 때마다 공개적으로 사용가능한 오픈 소스 도구를 사용하여 툴을 사용하는데 충분히 숙련된 누구나 우리의 작업을 재현할 수 있고 private하고 비싼 툴에 의존하지 않아도 된다는 것을 강조한다.

## 문제 설명

커뮤니티는 안티 바이러스의 탐지 매커니즘을 '정적'인지 '동적'인지 분류하는 것을 좋아한다. 일반적으로, 만약 탐지가 멀웨어의 실행 전에 트리거된다면, 그것은 정적 탐지의 종류로 간주된다.

그러나, 시그니쳐가 프로세스 생성, in-memory file downloads 같은 이벤트에 대한 반응으로 멀웨어가 실행되는 동안 호출될 수 있다와 같은 정적 메커니즘을 아는 것은 가치가 있다.

어쨌든, 만약 우리가 모든 종류의 보안 소프트웨어에 대하여 좋은 old Meterpreter를 사용하길 원한다면, 우리는 아래의 조건을 충족하는 방향으로 수정을 해야만한다.

- 파일 시스템 스캔 혹은 메모리 스캔 중에 모든 정적 시그니쳐를 우회해라
- 유저 영역 API 후킹 우회와 관련된 "행동 감지"를 우회해라

그러나, Meterpreter는 여러 모듈로 구성되고 전체 코드 베이스의 양은 약 700,000 줄의 코드이다. 게다가, 그것은 지속적으로 업데이트되므로 프로젝트의 private fork를 수행하는 것은 매우 형편 없는 것이 확실하다.

요약하면, 우리는 자동적으로 코드베이스를 변형할 방법이 필요하다.

## Solutions

안티바이러스 소프트웨어 우회한 수 년의 경험 후, 커뮤니티에 공유할 인사이트가 있다면, 멀웨어 탐지는 항상 대부분 문자열, API 훅, 혹은 둘의 조합을 기반으로 함을 알 수 있다.

심지어 Cylance 같은 머신러닝 분류를 구현하는 제품에도 문자열, API 임포트와 후킹가능한 API 호출이 없는 멀웨어는 Sergio Rico의 방어를 뚫은 축구공처럼 탐지를 통과할 것이 확실하다.

Meterpreter는 수천개의 문자열을 가지고 있고, API imports는 어떠한 방법으로도 숨겨지지 않으며 "WriteProcessMemory"와 같은 민감한 API는 유저 영역 API 후킹으로 쉽게 인터셉트할 수 있다. 그래서, 우리는 자동화된 방식 사용하여 2개의 잠재적인 솔루션을 생성할 필요가 있다.

- Source-to-source 코드 리팩터링
- LLVM 컴파일 시에 코드 베이스 난독화

후자가 선호되는 방식이고, 많은 유명한 연구들에서 같은 결론에 도달했다. 주요한 이유는 transformation pass를 한 번 작성해서 소프트웨어의 프로그래밍 언어나 타겟 아키텍처에 대해 독립적으로 재사용할 수 있다는 것이다.

![](img/Untitled.png)

이미지 출처: [http://www.aosabook.org/en/llvm.html](http://www.aosabook.org/en/llvm.html)

그러나, 이것은 Visual Studio 말고 다른 컴파일러로 Meterpreter를 컴파일할 수 있는 능력이 요구된다. 이것을 바꾸기 위해 2018년 12월부터 몇 가지 작업을 퍼블리시했지만 공식 코드 베이스에 적용은 1년이 더 지난 지금도 진행중이다. 

그동안 우리는 첫번째 접근법을 구현하기로 결정했다. 최첨단 소스 코드 리팩터링 검토를 통해 *libTooling*(Clang/LLVM 툴체인의 일부)이 C/C++ 소스의 구문 분석과 수정에 가장 적합한 후보였다.

Note: 코드베이스는 Visual Studio 의존성이 강하기 때문에 Clang은 Metepreter의 많은 부분의 구문 분석에 실패할 것이다. 그러나 타겟 안티바이러스를 절반의 성공으로 우회하는 것은 여전히 가능했다. 그리고 여기서 컴파일 시간 변환에 비해 source-to-source 변환이 얻는 이득이 있을 것이다. 후자는 어떤 에러 없이 전체 프로젝트를 컴파일하는 것을 요구한다. 전자는 수천개의 컴파일 에러에 탄력적이다; 완벽히 좋은 불완전한 추상 구문 트리로 끝이 난다.

![](img/Untitled%201.png)

LLVM passes vs libTooling

## 문자열 난독화

C/C++에서 문자열은 많은 다른 context 안에 위치한다.

*libTooling*은 유쾌한 일이 아니므로 파레토의 법칙을 적용시키고 Meterpreter의 코드베이스에서 가장 의심스러운 문자열 발생을 다루는 것으로 제한했다:

- 함수 인자
- 리스트 이니셜라이저

### 함수 인자

예를 들어 ESET Nod32는 다음 문맥에서 "ntdll" 문자열을 의심스러운 것으로 flag한다.

```cpp
ntdll = LoadLibrary(TEXT("ntdll"))
```

그러나 다음과 같이 코드를 재작성하면 성공적으로 탐지를 우회한다.

```cpp
wchar_t ntdll_str[] = {'n', 't', 'd', 'l', 'l', 0};
ntdll = LoadLibrary(ntdll_str)
```

첫번째 코드는 최종 바이너리에서 *.rdata* 섹션에 *"ntdll"* 문자열이 저장되도록하고 안티바이러스에 의해 쉽게 발견될 수 있다. 두번째 코드는 문자열이 실행 시에 스택에 저장되도록하고 일반적인 경우에는 코드로부터 정적으로 구분할 수 없다. *IDA Pro* 혹은 대체품에서는 종종 문자열을 감지할 수 있지만 바이너리에 대해 더 진보되고 계산 집약적으로 분석을 실행한다.

### 리스트 이니셜라이저

*Meterpreter*의 코드베이스에서 이런 종류의 구성은 c/meterpreter/source/extensions/extapi/extapi.c와 같은 몇몇 파일에서 찾을 수 있습니다.

```cpp
Command customCommands[] =
{
    COMMAND_REQ("extapi_window_enum", request_window_enum),
    COMMAND_REQ("extapi_service_enum", request_service_enum),
    COMMAND_REQ("extapi_service_query", request_service_query),
    COMMAND_REQ("extapi_service_control", request_service_control),
    COMMAND_REQ("extapi_clipboard_get_data", request_clipboard_get_data),
    COMMAND_REQ("extapi_clipboard_set_data", request_clipboard_set_data),
    COMMAND_REQ("extapi_clipboard_monitor_start", request_clipboard_monitor_start),
    COMMAND_REQ("extapi_clipboard_monitor_pause", request_clipboard_monitor_pause),
    COMMAND_REQ("extapi_clipboard_monitor_resume", request_clipboard_monitor_resume),
    COMMAND_REQ("extapi_clipboard_monitor_purge", request_clipboard_monitor_purge),
    COMMAND_REQ("extapi_clipboard_monitor_stop", request_clipboard_monitor_stop),
    COMMAND_REQ("extapi_clipboard_monitor_dump", request_clipboard_monitor_dump),
    COMMAND_REQ("extapi_adsi_domain_query", request_adsi_domain_query),
    COMMAND_REQ("extapi_ntds_parse", ntds_parse),
    COMMAND_REQ("extapi_wmi_query", request_wmi_query),
    COMMAND_REQ("extapi_pageant_send_query", request_pageant_send_query),
    ...
}
```

이러한 문자열들은 *ext_server_espia_x64.dll*의 *.rdata* 섹션에 평문으로 저장되고 *ESET Nod32*에 의해 감지된다.

설상가상으로 이러한 문자열들은 리스트 이니셜라이저에 위치한 매크로의 매개변수이다. 이로인해 극복할 수 없는 까다로운 코너 케이스가 발생한다. 목표는 아래와 같이 자동적으로 위의 코드를 재작성하는 것이다.

```cpp
char hid_extapi_UQOoNXigAPq4[] = {'e','x','t','a','p','i','_','w','i','n','d','o','w','_','e','n','u','m',0};
char hid_extapi_vhFHmZ8u2hfz[] = {'e','x','t','a','p','i','_','s','e','r','v','i','c','e','_','e','n','u','m',0};
char hid_extapi_pW25eeIGBeru[] = {'e','x','t','a','p','i','_','s','e','r','v','i','c','e','_','q','u','e','r','y'
0};
char hid_extapi_S4Ws57MYBjib[] = {'e','x','t','a','p','i','_','s','e','r','v','i','c','e','_','c','o','n','t','r'
'o','l',0};
char hid_extapi_HJ0lD9Dl56A4[] = {'e','x','t','a','p','i','_','c','l','i','p','b','o','a','r','d','_','g','e','t'
'_','d','a','t','a',0};
char hid_extapi_IiEzXils3UsR[] = {'e','x','t','a','p','i','_','c','l','i','p','b','o','a','r','d','_','s','e','t'
'_','d','a','t','a',0};
char hid_extapi_czLOBo0HcqCP[] = {'e','x','t','a','p','i','_','c','l','i','p','b','o','a','r','d','_','m','o','n'
'i','t','o','r','_','s','t','a','r','t',0};
char hid_extapi_WcWbTrsQujiT[] = {'e','x','t','a','p','i','_','c','l','i','p','b','o','a','r','d','_','m','o','n'
'i','t','o','r','_','p','a','u','s','e',0};
char hid_extapi_rPiFTZW4ShwA[] = {'e','x','t','a','p','i','_','c','l','i','p','b','o','a','r','d','_','m','o','n'
'i','t','o','r','_','r','e','s','u','m','e',0};
char hid_extapi_05fAoaZLqOoy[] = {'e','x','t','a','p','i','_','c','l','i','p','b','o','a','r','d','_','m','o','n'
'i','t','o','r','_','p','u','r','g','e',0};
char hid_extapi_cOOyHTPTvZGK[] = {'e','x','t','a','p','i','_','c','l','i','p','b','o','a','r','d','_','m','o','n','i','t','o','r','_','s','t','o','p',0};
char hid_extapi_smtmvW05cI9y[] = {'e','x','t','a','p','i','_','c','l','i','p','b','o','a','r','d','_','m','o','n','i','t','o','r','_','d','u','m','p',0};
char hid_extapi_01kuYCM8z49k[] = {'e','x','t','a','p','i','_','a','d','s','i','_','d','o','m','a','i','n','_','q','u','e','r','y',0};
char hid_extapi_SMK9uFj6nThk[] = {'e','x','t','a','p','i','_','n','t','d','s','_','p','a','r','s','e',0};
char hid_extapi_PHxnGM7M0609[] = {'e','x','t','a','p','i','_','w','m','i','_','q','u','e','r','y',0};
char hid_extapi_J7EGS6FRHwkV[] = {'e','x','t','a','p','i','_','p','a','g','e','a','n','t','_','s','e','n','d','_','q','u','e','r','y',0};

Command customCommands[] =
{

    COMMAND_REQ(hid_extapi_UQOoNXigAPq4, request_window_enum),
    COMMAND_REQ(hid_extapi_vhFHmZ8u2hfz, request_service_enum),
    COMMAND_REQ(hid_extapi_pW25eeIGBeru, request_service_query),
    COMMAND_REQ(hid_extapi_S4Ws57MYBjib, request_service_control),
    COMMAND_REQ(hid_extapi_HJ0lD9Dl56A4, request_clipboard_get_data),
    COMMAND_REQ(hid_extapi_IiEzXils3UsR, request_clipboard_set_data),
    COMMAND_REQ(hid_extapi_czLOBo0HcqCP, request_clipboard_monitor_start),
    COMMAND_REQ(hid_extapi_WcWbTrsQujiT, request_clipboard_monitor_pause),
    COMMAND_REQ(hid_extapi_rPiFTZW4ShwA, request_clipboard_monitor_resume),
    COMMAND_REQ(hid_extapi_05fAoaZLqOoy, request_clipboard_monitor_purge),
    COMMAND_REQ(hid_extapi_cOOyHTPTvZGK, request_clipboard_monitor_stop),
    COMMAND_REQ(hid_extapi_smtmvW05cI9y, request_clipboard_monitor_dump),
    COMMAND_REQ(hid_extapi_01kuYCM8z49k, request_adsi_domain_query),
    COMMAND_REQ(hid_extapi_SMK9uFj6nThk, ntds_parse),
    COMMAND_REQ(hid_extapi_PHxnGM7M0609, request_wmi_query),
    COMMAND_REQ(hid_extapi_J7EGS6FRHwkV, request_pageant_send_query),
    COMMAND_TERMINATOR
};
```

### API Imports 숨기기

외부 라이브러리에 의해  내보낸 함수를 호출하면 링커가 *Import Address Table*(IAT)에 항목을 기록한다. 결과적으로 함수 이름은 바이너리 내에서 평문으로 드러날 것이고 멀웨어의 실행 없이도 정적으로 복구할 수 있다. 물론 다른 것보다 더 의심스러운 함수 이름이 있다. 모든 의심스러운 것들을 숨기고 대부분 정당한 바이너리에 존재하는 것을 유지하는 것이 현명하다.

예를 들어 Metepreter의 *kiwi* 확장에서 아래와 같은 라인을 찾을 수 있다.

```cpp
enumStatus = SamEnumerateUsersInDomain(hDomain, &EnumerationContext, 0, &pEnumBuffer, 100, &CountRetourned);
```

이 함수는 *samlib.dll*에서 내보내지므로 링커는 컴파일된 바이너리에 *"samlib.dll"*과 *"SamEnumberateUsersInDomain"* 문자열이 표시되도록 한다. 

이러한 이슈를 해결하기 위해서 실행 중에 LoadLibrary/GetProcAddress를 사용하여 API를 가져올 수 있다. 물론 이러한 함수는 문자열과 함께 동작하므로 난독화를 잘 해야한다. 그러므로 아래와 같이 위의 코드를 자동적으로 재작성해야한다.

```cpp
typedef NTSTATUS(__stdcall* _SamEnumerateUsersInDomain)(
    SAMPR_HANDLE DomainHandle,
    PDWORD EnumerationContext,
    DWORD UserAccountControl,
    PSAMPR_RID_ENUMERATION* Buffer,
    DWORD PreferedMaximumLength,
    PDWORD CountReturned
);
char hid_SAMLIB_01zmejmkLCHt[] = {'S','A','M','L','I','B','.','D','L','L',0};
char hid_SamEnu_BZxlW5ZBUAAe[] = {'S','a','m','E','n','u','m','e','r','a','t','e','U','s','e','r','s','I','n','D','o','m','a','i','n',0};
HANDLE hhid_SAMLIB_BZUriyLrlgrJ = LoadLibrary(hid_SAMLIB_01zmejmkLCHt);
_SamEnumerateUsersInDomain ffSamEnumerateUsersInDoma =(_SamEnumerateUsersInDomain)GetProcAddress(hhid_SAMLIB_BZUriyLrlgrJ, hid_SamEnu_BZxlW5ZBUAAe);
enumStatus = ffSamEnumerateUsersInDoma(hDomain, &EnumerationContext, 0, &pEnumBuffer, 100, &CountRetourned);
```

### Syscall 재작성

기본적으로 Cylance가 실행중인 컴퓨터에서 Meterpreter의 *migrate* 명령어를 사용하면 안티바이러스 감지이 트리거 된다. Cylance는 유저 영역 후킹으로 프로세스 인젝션을 탐지한다. 탐지를 우회하려면 요즘 유행하는 접근법으로 보이는 훅을 제거하거나 단순히 피할 수 있다. *ntdll*을 읽고 syscall 번호를 복구하고 즉시 호출 가능한 안티바이러스의 유저 영역 훅을 효과적으로 우회할 수 있는 쉘코드를 삽입하는 것이 더 간단하다는 것을 찾았다. 현재까지 디스크로부터 읽는 *NTDLL.DLL*을 의심스러운 것으로 판단하는 블루팀을 찾지 못했다.

## 구현

모든 앞서말한 아이디어들은 *libTooling*을 기반으로 한 소스 코드 리팩터링 툴에서 구현할 수 있다. 이 섹션은 가능한 시간과 libTooling 문서의 부족에 대한 인내 사이의 타협을 통해 우리가 한 방법을 문서화했다. 따라서 개선의 여지가 있고 무언가 눈에 띄는 경우 우리는 그것에 대해 듣고 싶다.

### 추상 구문 트리 101

컴파일러는 일반적으로 여러 구성 요소로 구성되고 일반적인 구성 요소는 *Parser*와 *Lexer*이다. 소스코드가 컴파일러에 들어가면 먼저 원본 소스 코드(프로그래머가 작성한)에서 Parse Tree를 생성하고 노드(컴파일러가 필요로 하는 것)에 의미 정보를 추가한다. 이 과정의 결과를 *Abstract Syntax Tree*(추상 구문 트리)라고 부른다. 아래의 예는 위키피디아의 예시이다:

```cpp
while b ≠ 0
  if a > b
    a := a − b
  else
    b := b − a
return a
```

작은 프로그램의 일반적인 AST는 다음과 같다:

![](antivirus-evasion-pt1/Untitled%202.png)

추상 구문 트리 예시(https://en.wikipedia.org/wiki/Abstract_syntax_tree)

이 데이터 구조는 다른 프로그램의 속성을 이해하는 프로그램을 작성할 때보다 더 정확한 알고리즘이 가능하므로 대규모 코드 리팩터링을 수행하기 위해 좋은 선택인 것 같다.

### Clang의 추상 구문 트리

The Right Way 소스 코드를 변경해야하기 때문에 *Clang의 AST*에 대해 알아야한다. 좋은 소식은 Clang은 예쁜 색깔로 AST 덤프를 명령줄 스위치에 노출한다는 것이다. 나쁜 소식은 토이 프로젝트를 제외한 모든 것에 대하여 올바른 컴파일러 플래그를 설정하는 것이 까다롭다는 것이다.

지금은 현실적이고 간단한 테스트 변환 유닛을 만들어보자:

```cpp
#include <windows.h>

typedef NTSTATUS (NTAPI *f_NtMapViewOfSection)(HANDLE, HANDLE, PVOID *, ULONG, ULONG,
PLARGE_INTEGER, PULONG, ULONG, ULONG, ULONG);

int main(void)
{
    f_NtMapViewOfSection lNtMapViewOfSection;
    HMODULE ntdll;

    if (!(ntdll = LoadLibrary(TEXT("ntdll"))))
    {
        return -1;
    }

    lNtMapViewOfSection = (f_NtMapViewOfSection)GetProcAddress(ntdll, "NtMapViewOfSection");
    lNtMapViewOfSection(0,0,0,0,0,0,0,0,0,0);
    return 0;
}
```

그리고 다음 *.sh* 파일의 스크립트를 삽입해라(어떻게 모든 컴파일러 플래그를 설정했는지 물어보지 마라, 괴로운 기억을 떠오르게 할 것이다.)

```cpp
WIN_INCLUDE="/Users/vladimir/headers/winsdk"
CLANG_PATH="/usr/local/Cellar/llvm/9.0.1"#"/usr/lib/clang/8.0.1/"

clang -cc1 -ast-dump "$1" -D "_WIN64" -D "_UNICODE" -D "UNICODE" -D "_WINSOCK_DEPRECATED_NO_WARNINGS"\
  "-I" "$CLANG_PATH/include" \
  "-I" "$CLANG_PATH" \
  "-I" "$WIN_INCLUDE/Include/msvc-14.15.26726-include"\
  "-I" "$WIN_INCLUDE/Include/10.0.17134.0/ucrt" \
  "-I" "$WIN_INCLUDE/Include/10.0.17134.0/shared" \
  "-I" "$WIN_INCLUDE/Include/10.0.17134.0/um" \
  "-I" "$WIN_INCLUDE/Include/10.0.17134.0/winrt" \
  "-fdeprecated-macro" \
  "-w" \
  "-fdebug-compilation-dir"\
  "-fno-use-cxa-atexit" "-fms-extensions" "-fms-compatibility" \
  "-fms-compatibility-version=19.15.26726" "-std=c++14" "-fdelayed-template-parsing" "-fobjc-runtime=gcc" "-fcxx-exceptions" "-fexceptions" "-fseh-exceptions" "-fdiagnostics-show-option" "-fcolor-diagnostics" "-x" "c++"
```

*WIN_INCLUDE*는 Win32 API와 상호 작용하기 위해 필요한 모든 헤더들이 포함된 폴더를 가리킨다. 이것들은 표준 윈도우 10 설치로부터 그대로 가져왔고 여러분의 두통을 덜어주기 위해 우리는 MinGW의 것을 선택하는 것 대신에 같은 것을 하는 것을 추천한다. 그 후 테스트 C 파일을 인수로 스크립트를 호출해라. 이렇게 하면 18MB 파일이 생성되지만 *"NtMapViewOfSection"* 같은 우리가 선언한 문자열 리터럴 중 하나를 검색해서 AST의 흥미로운 부분으로 이동할 수 있다:

![](antivirus-evasion-pt1/Untitled%203.png)

이제 AST를 시각화하는 방법을 얻었으므로 결과 소스 코드의 구문 오류 도입 없이 결과를 얻기 위해 노드들이 어떻게 업데이트해야 하는 지 이해하기 쉽다. 후속 섹션은 *libTooling*을 사용한 AST 조작에 관련된 구현 세부사항이 포함되어 있다.

### ClangTool 보일러플레이트

놀랍지 않게, 흥미로운 일을 하기 전에 필요한 보일러플레이트 코드가 있으니 다음 코드를 *main.cpp*에 입력한다.

```cpp
#include "clang/AST/ASTConsumer.h"
#include "clang/AST/ASTContext.h"
#include "clang/AST/Decl.h"
#include "clang/AST/Type.h"
#include "clang/ASTMatchers/ASTMatchFinder.h"
#include "clang/ASTMatchers/ASTMatchers.h"
#include "clang/Basic/SourceManager.h"
#include "clang/Frontend/CompilerInstance.h"
#include "clang/Frontend/FrontendAction.h"
#include "clang/Tooling/CommonOptionsParser.h"
#include "clang/Tooling/Tooling.h"
#include "clang/Rewrite/Core/Rewriter.h"

// LLVM includes
#include "llvm/ADT/ArrayRef.h"
#include "llvm/ADT/StringRef.h"
#include "llvm/Support/CommandLine.h"
#include "llvm/Support/raw_ostream.h"

#include "Consumer.h"
#include "MatchHandler.h"

#include <iostream>
#include <memory>
#include <string>
#include <vector>
#include <fstream>
#include <clang/Tooling/Inclusions/IncludeStyle.h>
#include <clang/Tooling/Inclusions/HeaderIncludes.h>
#include <sstream>

namespace ClSetup {
    llvm::cl::OptionCategory ToolCategory("StringEncryptor");
}

namespace StringEncryptor {

    clang::Rewriter ASTRewriter;
    class Action : public clang::ASTFrontendAction {

    public:
        using ASTConsumerPointer = std::unique_ptr<clang::ASTConsumer>;

        ASTConsumerPointer CreateASTConsumer(clang::CompilerInstance &Compiler,
                                             llvm::StringRef Filename) override {

            ASTRewriter.setSourceMgr(Compiler.getSourceManager(), Compiler.getLangOpts());
            std::vector<ASTConsumer*> consumers;

            consumers.push_back(&StringConsumer);
  
            // several passes can be combined together by adding them to `consumers`
            auto TheConsumer = llvm::make_unique<Consumer>();
            TheConsumer->consumers = consumers;
            return TheConsumer;
        }

        bool BeginSourceFileAction(clang::CompilerInstance &Compiler) override {
            llvm::outs() << "Processing file " << '\n';
            return true;
        }

        void EndSourceFileAction() override {

            clang::SourceManager &SM = ASTRewriter.getSourceMgr();

            std::string FileName = SM.getFileEntryForID(SM.getMainFileID())->getName();
            llvm::errs() << "** EndSourceFileAction for: " << FileName << "\n";

            // Now emit the rewritten buffer.
            llvm::errs() << "Here is the edited source file :\n\n";
            std::string TypeS;
            llvm::raw_string_ostream s(TypeS);
            auto FileID = SM.getMainFileID();
            auto ReWriteBuffer = ASTRewriter.getRewriteBufferFor(FileID);

            if(ReWriteBuffer != nullptr)
                ReWriteBuffer->write((s));
            else{
                llvm::errs() << "File was not modified\n";
                return;
            }

            std::string result = s.str();
            std::ofstream fo(FileName);
       
            if(fo.is_open())
                fo << result;
            else
                llvm::errs() << "[!] Error saving result to " << FileName << "\n";
        }
    };
}

auto main(int argc, const char *argv[]) -> int {

    using namespace clang::tooling;
    using namespace ClSetup;

    CommonOptionsParser OptionsParser(argc, argv, ToolCategory);
    ClangTool Tool(OptionsParser.getCompilations(),
                   OptionsParser.getSourcePathList());

    auto Action = newFrontendActionFactory<StringEncryptor::Action>();
    return Tool.run(Action.get());
}
```

이 보일러플레이트 코드는 공식 문서의 예에서 가져온 것이므로 더 이상 설명할 필요가 없다. 사실 언급할만한 가치가 있는 변경은 *CreateASTConsumer* 내부뿐이다. 우리의 마지막 게임은 같은 변환 유닛에서 여러 개의 변환 패스를 실행하는 것이다. *consumers* 컬렉션(본래 라인은 consumer.push_back(&...); )에 아이템을 추가하면 된다.

## 문자열 난독화

이 섹션은 3가지의 단계로 구성된 문자열 난독화 패스의 가장 중요한 구현 세부 사항을 설명한다:

- 소스 코드에 문자열 리터럴을 찾는다.
- 그것들을 변수로 변환한다.
- 적절한 위치(함수 또는 전역 컨텍스트 포함)에 변수 정의/할당을 삽입한다.

### 소스 코드에서 문자열 리터럴 찾기

StringConsumer는 아래와 같이 정의될 수 있다.(StringEncryptor 네임스페이스의 시작부분)

```cpp
class StringEncryptionConsumer : public clang::ASTConsumer {
public:

    void HandleTranslationUnit(clang::ASTContext &Context) override {
        using namespace clang::ast_matchers;
        using namespace StringEncryptor;

        llvm::outs() << "[StringEncryption] Registering ASTMatcher...\n";
        MatchFinder Finder;
        MatchHandler Handler(&ASTRewriter);

        const auto Matcher = stringLiteral().bind("decl");

        Finder.addMatcher(Matcher, &Handler);
        Finder.matchAST(Context);
    }
};

StringEncryptionConsumer StringConsumer = StringEncryptionConsumer();
```

변환 유닛이 주어지면 *Clang*에게 AST 내부의 패턴을 찾으라고 말할 수 있을 뿐아니라 match가 발견될 때마다 호출될 "handler"를 등록할 수 있다. *Clang의 [ASTMatcher](https://clang.llvm.org/docs/LibASTMatchersReference.html)*에 의해 드러난 패턴 매칭은 꽤 강력하지만 여기서는 우리는 오직 문자열 리터럴을 찾는데만 의존하므로 덜 사용된다.

그 후 우리는 *MatchHandler*를 구현함으로써 문제의 핵심을 파악할 수 있고 *[MatchResult](https://clang.llvm.org/doxygen/structclang_1_1ast__matchers_1_1MatchFinder_1_1MatchResult.html#ab06481084c7027e5779a5a427596f66c)* 인스턴스를 얻을 수 있을 것이다. *MatchResult*는 식별된 AST 노드에 대한 참조와 값이 없는 컨텍스트 정보를 포함한다.

클래스 정의를 구현하고 *[clang::ast_matchers::MatchFinder::MatchCallback](https://clang.llvm.org/doxygen/classclang_1_1ast__matchers_1_1MatchFinder_1_1MatchCallback.html)*으로부터 좋은 stuff를 상속받자:

```cpp
#ifndef AVCLEANER_MATCHHANDLER_H
#define AVCLEANER_MATCHHANDLER_H

#include <vector>
#include <string>
#include <memory>
#include "llvm/Support/raw_ostream.h"
#include "llvm/Support/CommandLine.h"
#include "llvm/ADT/StringRef.h"
#include "llvm/ADT/ArrayRef.h"
#include "clang/Rewrite/Core/Rewriter.h"
#include "clang/Tooling/Tooling.h"
#include "clang/Tooling/CommonOptionsParser.h"
#include "clang/Frontend/FrontendAction.h"
#include "clang/Frontend/CompilerInstance.h"
#include "clang/Basic/SourceManager.h"
#include "clang/ASTMatchers/ASTMatchers.h"
#include "clang/ASTMatchers/ASTMatchFinder.h"
#include "clang/AST/Type.h"
#include "clang/AST/Decl.h"
#include "clang/AST/ASTContext.h"
#include "clang/AST/ASTConsumer.h"
#include "MatchHandler.h"

class MatchHandler : public clang::ast_matchers::MatchFinder::MatchCallback {

public:
    using MatchResult = clang::ast_matchers::MatchFinder::MatchResult;

    MatchHandler(clang::Rewriter *rewriter);
    void run(const MatchResult &Result) override; // callback function that runs whenever a Match is found.

};

#endif //AVCLEANER_MATCHHANDLER_H
```

*MatchHandler.cpp*에서 MatchHandler의 생성자와 *run* 콜백 함수를 구현해야할 것이다. 나중에 사용할 *[clang::Rewriter](https://clang.llvm.org/doxygen/classclang_1_1Rewriter.html)*의 인스턴스를 저장하기만 하면 되므로 생성자는 상당히 간단하다.

```cpp
using namespace clang;

MatchHandler::MatchHandler(clang::Rewriter *rewriter) {
    this->ASTRewriter = rewriter;
}
```

*run*의 구현은 아래와 같다:

```cpp
void MatchHandler::run(const MatchResult &Result) {
    const auto *Decl = Result.Nodes.getNodeAs<clang::StringLiteral>("decl");
    clang::SourceManager &SM = ASTRewriter->getSourceMgr();

    // skip strings in included headers
    if (!SM.isInMainFile(Decl->getBeginLoc()))
        return;

    // strings that comprise less than 5 characters are not worth the effort
    if (!Decl->getBytes().str().size() > 4) {
        return;
    }

    climbParentsIgnoreCast(*Decl, clang::ast_type_traits::DynTypedNode(), Result.Context, 0);
}
```

위에서 3개의 요소는 언급할 가치가 있다:

- StringEncryptionConsumer에서 정의된 패턴에 의해 매치된 AST node를 추출한다. 그러기 위해서 패턴이 바인딩된 식별자와 관련된 인수로 문자열을 예상하는 *[getNodeAs](https://clang.llvm.org/doxygen/classclang_1_1ast__matchers_1_1BoundNodes.html)*를 호출할 수 있다.(해당 줄을 봐라 const auto Matcher = stringLiteral().bind("decl"))
- 분석 중인 변환 유닛에서 정의되지 않은 문자열을 스킵한다. 실제로 우리의 passs는 시스템 헤더를 번환 유닛에 복사-붙여넣기할 *Clang*의 전처리기 이후에 개입된다.
- 그러면 우리는 문자열 리터럴을 처리할 준비가 됐다. 그 문자열 리터럴이 어느 컨텍스트에서 발견됐는지 알아야하기 때문에, 우리는 추출된 노드를 사용자 정의 함수(*climbParentsIgnoreCast*보다 더 좋은 이름이 없다)에 동봉된 AST에 대한 참조가 포함된 *[Result.Context](https://clang.llvm.org/doxygen/structclang_1_1ast__matchers_1_1MatchFinder_1_1MatchResult.html#ab06481084c7027e5779a5a427596f66c)*를 통해 전달한다. 목표는 흥미로운 노드를 찾을 때 까지 위쪽의 트리를 방문하는 것이다. 이 경우에서 *[CallExpr](https://clang.llvm.org/doxygen/classclang_1_1CallExpr.html)* 타입의 노드에 우리는 흥미가 있다.

```cpp
bool
MatchHandler::climbParentsIgnoreCast(const StringLiteral &NodeString, clang::ast_type_traits::DynTypedNode node,
                                     clang::ASTContext *const pContext, uint64_t iterations) {

    ASTContext::DynTypedNodeList parents = pContext->getParents(NodeString);

    if (iterations > 0) {
        parents = pContext->getParents(node);
    }

    for (const auto &parent : parents) {

        StringRef ParentNodeKind = parent.getNodeKind().asStringRef();

        if (ParentNodeKind.find("Cast") != std::string::npos) {

            return climbParentsIgnoreCast(NodeString, parent, pContext, ++iterations);
        }

        handleStringInContext(&NodeString, pContext, parent);
    }

    return false;
}
```

간단히 말해서 이 함수는 흥미로운 것("cast"가 아닌 것)을 찾을 때 까지 *StringLiteral* 노드의 부모 노드들을 재귀적으로 탐색한다. handleStringInContext는 straight-forward하다:

```cpp
void MatchHandler::handleStringInContext(const clang::StringLiteral *pLiteral, clang::ASTContext *const pContext,
                                         const clang::ast_type_traits::DynTypedNode node) {

    StringRef ParentNodeKind = node.getNodeKind().asStringRef();

    if (ParentNodeKind.compare("CallExpr") == 0) {
        handleCallExpr(pLiteral, pContext, node);
    } else if (ParentNodeKind.compare("InitListExpr") == 0) {
        handleInitListExpr(pLiteral, pContext, node);
    } else {
        llvm::outs() << "Unhandled context " << ParentNodeKind << " for string " << pLiteral->getBytes() << "\n";
    }
}
```

위에 코드로부터 분명한 것은 오직 두 종류의 노드만 실제로 처리된다. 또한 필요할 때 더 추가하는게 쉬워야한다. 실제로 두 경우 이미 비슷하게 처리되고 있다.

```cpp
void MatchHandler::handleCallExpr(const clang::StringLiteral *pLiteral, clang::ASTContext *const pContext,
                                  const clang::ast_type_traits::DynTypedNode node) {

    const auto *FunctionCall = node.get<clang::CallExpr>();

    if (isBlacklistedFunction(FunctionCall)) {
        return; // exclude printf-like functions when the replacement is not constant anymore (C89 standard...).
    }

    handleExpr(pLiteral, pContext, node);
}

void MatchHandler::handleInitListExpr(const clang::StringLiteral *pLiteral, clang::ASTContext *const pContext,
                                      const clang::ast_type_traits::DynTypedNode node) {

    handleExpr(pLiteral, pContext, node);
}
```

### 문자열 리터럴 replacing

*[CallExpr](https://clang.llvm.org/doxygen/classclang_1_1CallExpr.html)*과 *[InitListExpr](https://clang.llvm.org/doxygen/classclang_1_1InitListExpr.html)*은 비슷하게 처리되므로 우리는 둘 다 사용할 수 있는 공통적인 함수를 정의한다.

```cpp
bool MatchHandler::handleExpr(const clang::StringLiteral *pLiteral, clang::ASTContext *const pContext,
                                  const clang::ast_type_traits::DynTypedNode node) {

    clang::SourceRange LiteralRange = clang::SourceRange(
            ASTRewriter->getSourceMgr().getFileLoc(pLiteral->getBeginLoc()),
            ASTRewriter->getSourceMgr().getFileLoc(pLiteral->getEndLoc())
    );

    if(shouldAbort(pLiteral, pContext, LiteralRange))
        return false;

    std::string Replacement = translateStringToIdentifier(pLiteral->getBytes().str());

    if(!insertVariableDeclaration(pLiteral, pContext, LiteralRange, Replacement))
        return false ;

    Globs::PatchedSourceLocation.push_back(LiteralRange);

    return replaceStringLiteral(pLiteral, pContext, LiteralRange, Replacement);
}
```

- 우리는 변수명을 랜덤적으로 생성한다.
- 가까운 위치에 빈 공간을 찾고 변수 선언을 삽입해라. 이것은 기본적으로 ASTRewriter→InsertText()를 둘러싼 wrapper이다.
- 1 단계에서 생성된 식별자를 가진 문자열을 replace한다.
- collection에 문자열 리터럴 위치를 추가한다. 이것은 *InitListExpr*를 방문할 때 같은 문자열 리터럴이 두 번 나타날 것이기 때문에 유용하다(이유는 알 수 없다).

마지막 단계는 실제로 구현하기 어려운 유일한 단계이므로 첫 번째에 집중하자:

```cpp
bool MatchHandler::replaceStringLiteral(const clang::StringLiteral *pLiteral, clang::ASTContext *const pContext,
                                        clang::SourceRange LiteralRange,
                                        const std::string& Replacement) {

    // handle "TEXT" macro argument, for instance LoadLibrary(TEXT("ntdll"));
    bool isMacro = ASTRewriter->getSourceMgr().isMacroBodyExpansion(pLiteral->getBeginLoc());

    if (isMacro) {
        StringRef OrigText = clang::Lexer::getSourceText(CharSourceRange(pLiteral->getSourceRange(), true),
                                                         pContext->getSourceManager(), pContext->getLangOpts());

        // weird bug with TEXT Macro / other macros...there must be a proper way to do this.
        if (OrigText.find("TEXT") != std::string::npos) {

            ASTRewriter->RemoveText(LiteralRange);
            LiteralRange.setEnd(ASTRewriter->getSourceMgr().getFileLoc(pLiteral->getEndLoc().getLocWithOffset(-1)));
        }
    }

    return ASTRewriter->ReplaceText(LiteralRange, Replacement);
}
```

일반적으로 텍스트 대체는 *[ReplaceText](https://clang.llvm.org/doxygen/classclang_1_1Rewriter.html#afe00ce2338ce67ba76832678f21956ed)* API로 실현되어야 하지만 실제로  매우 많은 버그가 발생했다. 매크로에 관련되면 *Clang*의 API는 매우 복잡해지는 경향이 있다. 예를 들어 isMacroBodyExpansion() 확인을 제거하면 "TEXT"가 인자를 대신하여 대체된다.

예를 들어 LoadLibrary(TEXT("ntdll"))에서 실제 결과는 LoadLibrary(your_variable("ntdll"))로 잘못된 것이다.

이유는 TEXT가 *Clang*의 전처리기에 의해 처리될 때 L"ntdll"을 대체되는 매크로이기 때문이다. 우리의 변환 패스는 전처리기가 그것의 작업을 완료한 후 발생하므로 토큰 *"ntdll"*의 처음과 끝 위치를 쿼리하는 것은 몇 글자 차이로 잘못된 값이 나올 수 있기 때문에 우리에게 유용하지 않다.불행하게도 원본 번환 유닛에 실질적 위치를 쿼리하는 것은 *Clang* API를 이용한 흑마법의 한 종류이고, 작업 솔루션에서 시행착오를 발생했다, 미안.

### 빈 공간 근처에 변수 선언 삽입

지금 우리는 문자열 리터럴을 변수 식별자로 대체할 수 있으므로 변수를 정의하고 원래 문자열의 값으로 할당하는 것이 목표이다. 요약하자면 우리는 어떤것도 덮어쓰지 않고 char your_variable[] = "ntdll"을 포함한 소스코드를 패치하는 것을 원한다.

2개의 시나리오가 존재할 수 있다:

- 문자열 리터럴은 함수 바디 안에 위치한다.
- 문자열 리터럴은 함수 바디 밖에 위치한다.

후자는 문자열 리터럴이 사용되는 표현식의 시작 부분을 찾는 것에만 필요하므로 가장 간단하다.

전자를 위해서 우리는 enclosing 함수를 찾을 필요가 있다. 그러면 Clang은 API를 노출시켜 함수 바디의 시작 위치(첫 bracket 이후)를 쿼리한다. 이것은 변수가 전체 함수에서 볼 수 있고 우리사 삽입한 토큰이 내용을 덮어씌우지 않기 때문에 변수 선언을 삽입하기에 이상적인 공간이다.

어떤 경우든 두 가지 상황은 FunctionDecl 혹은 VarDecl 타입의 노드가 발견될 때 까지 모든 부모 노드를 방문함에 의해 해결한다.

```cpp
MatchHandler::findInjectionSpot(clang::ASTContext *const Context, clang::ast_type_traits::DynTypedNode Parent,
                                const clang::StringLiteral &Literal, bool IsGlobal, uint64_t Iterations) {

    if (Iterations > CLIMB_PARENTS_MAX_ITER)
        throw std::runtime_error("Reached max iterations when trying to find a function declaration");

    ASTContext::DynTypedNodeList parents = Context->getParents(Literal);;

    if (Iterations > 0) {
        parents = Context->getParents(Parent);
    }

    for (const auto &parent : parents) {

        StringRef ParentNodeKind = parent.getNodeKind().asStringRef();

        if (ParentNodeKind.find("FunctionDecl") != std::string::npos) {
            auto FunDecl = parent.get<clang::FunctionDecl>();
            auto *Statement = FunDecl->getBody();
            auto *FirstChild = *Statement->child_begin();
            return {FirstChild->getBeginLoc(), FunDecl->getEndLoc()};

        } else if (ParentNodeKind.find("VarDecl") != std::string::npos) {

            if (IsGlobal) {
                return parent.get<clang::VarDecl>()->getSourceRange();
            }
        }

        return findInjectionSpot(Context, parent, Literal, IsGlobal, ++Iterations);
    }
}
```

## 테스트

```cpp
git clone https://github.com/SCRT/avcleaner
mkdir avcleaner/CMakeBuild && cd avcleaner/CMakeBuild
cmake ..
make
cd ..
bash run_example.sh test/string_simplest.c
```

![](img/Untitled%204.png)

보다시피 이것은 꽤 효과있다. 지금 이 예제는 regex와 훨씬 더 적은 코드 라인으로 해결할 수 있을 정도로 충분히 간단하다.

## 더 나아가서

지금까지 난독화 패스가 "StringEncryptor"로 이름지어졌음에도 문자열은 실제로 암호화되지 않는다. 실제로 문자열을 암호화하려면 얼마나 많은 노력이 필요할까?