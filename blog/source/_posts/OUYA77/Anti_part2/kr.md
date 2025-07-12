---
title: "[Research] Anti-Debugging Part 2(KR)"
author: OUYA77
tags: [Anti-Debugging, Debugging, Malware, OUYA77, Reverse Engineering]
categories: [Research]
date: 2025-03-24 00:45:00
cc: true
index_img: /2025/03/24/OUYA77/Anti_part2/kr/Debug_part2.jpg
---


안녕하세요 OUYA77 입니다. 🙂

지난 시간에는 

- 디버깅: 코드에서 문제가 있는 곳을 찾아서 범위를 좁혀가며 원인을 분석하고 해결하는 과정
- 안티디버깅: 디버거를 탐지하거나 그 동작을 방해하여, 소프트웨어가 역공학 및 분석에 취약하지 않도록 보호하는 기법

을 공부했었죠(혹시 아직 못보셨다면 👇 ㅎㅎ).

*지난 파트 보러가기*

[[Research] Anti-Debugging Part1(KR)](https://hackyboiz.github.io/2024/12/29/OUYA77/Anti_part1/kr/)

안티디버깅 기법에서는 **Static**과 **Dynamic** 두 가지 안티디버깅 기법에 대해 살펴보았습니다. **Static** 기법은 디버거가 프로세스에 연결되었는지 확인하기 위해 시스템 정보나 API를 사용합니다. 예를 들어, Windows에서는 `IsDebuggerPresent()` 함수를 사용하여 디버거의 존재를 확인할 수 있습니다. **Dynamic** 기법은 프로그램 실행 중의 행동 패턴을 기반으로 디버거를 탐지하며, 타이밍 기반 안티디버깅과 같은 방법을 사용합니다.

## 안티디버깅 기법 우회 방법


### Static 안티디버깅 기법 우회

Static 기법은 구현이 간단하지만, 상대적으로 쉽게 우회될 수 있습니다. 이를 우회하는 방법은 다음과 같습니다:

1. **API Hooking**: 디버거가 사용하는 API를 Hooking하여 API 호출을 가로채고, 디버거의 존재를 숨길 수 있습니다. 예를 들어, Windows에서 `IsDebuggerPresent()` 함수를 Hooking하여 항상 디버거가 없는 것으로 응답하게 할 수 있습니다.
2. **PEB 구조체 조작**: PEB의 `BeingDebugged` 필드를 직접 조작하여 디버거의 존재를 숨길 수 있습니다.

### Dynamic 안티디버깅 기법 우회

Dynamic 기법은 실행 중에 환경 변화나 동적 조건을 활용하여 동작하므로, 이를 우회하기 위해서는 프로그램의 런타임 상황을 충분히 이해해야 합니다. 우회 방법은 다음과 같습니다:

1. **타이밍 기반 우회**: 타이밍 기반 안티디버깅은 프로그램의 실행 속도를 측정하여 디버거의 존재를 탐지합니다. 이를 우회하기 위해, IDA와 같은 디버깅 툴을 이용하여 코드를 패치하여 타이밍 측정 코드를 조작할 수 있습니다.
2. **예외 처리 조작**: Dynamic 기법은 예외 발생 시 디버거의 존재를 탐지할 수 있습니다. 이를 우회하기 위해, 예외 처리 루틴을 조작하여 디버거의 존재를 숨길 수 있습니다.

# 실습

자! 그러면 이제 실습을 진행해보겠습니다.

**실습 환경**

- 운영체제: Windows 11
- 바이너리 아키텍처: 32비트
- 디버깅 도구: IDA Freeware 8.4
- IDE: Visual Studio 2022

이 환경에서 32비트 바이너리를 빌드하고 IDA Freeware를 사용하여 안티디버깅이 적용된 바이너리에 대해 디버깅하는 과정을 실습할 것입니다.

## Static 안티디버깅 기법 우회 실습

### 1. 일반 프로그램

“Part 1”에서도 봤듯이 안티디버깅이 적용된 일반 프로그램을 간단하게 만들어보겠습니다.

Visual Studio 2022에서 빈 프로젝트를 만들어주시고 main.cpp를 만든 후 x86으로 빌드해주세요.

![image.png](image.png)

```c
// main.cpp

#include <windows.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    if (IsDebuggerPresent()) {
        printf("Debugger detected!\n");
    }
    else {
        printf("No debugger found.\n");
    }

    system("pause");
    return 0;
}

```

빌드 후 exe가 생성된 폴더(./Debug)에 들어가서 빌드된 바이너리를 실행하면 다음과 같이 `No debugger found.` 가 뜹니다.

![image.png](image%201.png)

이제 IDA Freeware를 사용해서 해당 프로그램을 디버깅해보겠습니다. IDA에 불러와 F9를 눌러 디버깅을 시도하면..!!

![image.png](image%202.png)

위와 같이 `IsDebuggerPresent()` 함수에 의해 디버거가 감지되었다고 문장이 나옵니다.

해당 바이너리는 `IsDebuggerPresent()` 함수를 통해 디버깅을 탐지했습니다. 그럼 만약 이 함수가 제 기능을 못한다면 어떻게 될까요?! 이제 “API Hooking”을 통해 Static 안티디버깅을 우회하는 실습을 해보겠습니다.

### **2. API Hooking 실습 (Bypass Static Anti-Debugging)**

API Hooking 실습은 Microsoft Detours 라이브러리를 통해 진행하겠습니다. 이 라이브러리는 Windows에서 API 함수를 Hooking하여 디버거의 존재를 숨길 수 있도록 도와줍니다. 

먼저 Detours 라이브러리를 설치해봅시다.

1. [https://github.com/microsoft/detours](https://github.com/microsoft/detours) 에서 압축파일을 다운받은 후에 원하는 폴더에 넣은 후 해제해 주세요.
    
    ![image.png](image%203.png)
    
2. Visual Studio 2022에서 터미널을 켭니다.
    
    ![image.png](image%204.png)
    
3. `nmake` 를 통해서 소스코드를 빌드하면 bin, include, lib 라이브러리가 생성됩니다.
    
    ![image.png](image%205.png)
    
    ![image.png](image%206.png)
    

1. Visual Studio에서 `프로젝트>속성` 으로 들어가 구성은 “모든 구성”, 플랫폼은 “Win32”로 변경합니다.
    
    ![image.png](image%207.png)
    
2. Detours 라이브러리를 Visual Studio에 추가해줍니다.
    - **C/C++ 속성에서 추가 포함 디렉터리 설정**
        1. 프로젝트 속성 창에서 "C/C++" 탭을 선택합니다.
        2. "일반" 하위 메뉴에서 "추가 포함 디렉터리" 항목을 찾습니다.
        3. "추가 포함 디렉터리" 필드에서 Detours의 include 경로를 입력합니다:
            
            `C:\Users\OUYA77\Desktop\Detours-main\include`
            
        4. "확인" 버튼을 클릭하여 변경 사항을 저장합니다.
    - **링커 속성에서 추가 라이브러리 디렉터리 설정**
        1. 프로젝트 속성 창에서 "링커" 탭을 선택합니다.
        2. "일반" 하위 메뉴에서 "추가 라이브러리 디렉터리" 항목을 찾습니다.
        3. "추가 라이브러리 디렉터리" 필드에서 Detours의 lib 경로를 입력합니다:
            
            `C:\Users\OUYA77\Desktop\Detours-main\lib`
            
        4. "확인" 버튼을 클릭하여 변경 사항을 저장합니다.

네 이제 라이브러리를 사용할 준비가 되었으면 dllmain.cpp를 선언해주세요.

```c
// dllmain.cpp
#pragma comment(lib, "detours.lib")
```

이제 진짜로 Detours를 사용해 **`IsDebuggerPresent()`** 함수를 Hooking하여 Static Anti-Debugging 을 우회해보겠습니다.

```c
// main.cpp

#include <windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <detours.h>

// IsDebuggerPresent() 함수를 Hooking하기 위한 함수
BOOL WINAPI HookedIsDebuggerPresent() {
    return FALSE; // 항상 디버거가 없는 것으로 응답
}

int main(void) {
    // IsDebuggerPresent() 함수를 Hooking
    HMODULE hKernel32 = GetModuleHandleA("kernel32.dll");
    if (hKernel32) {
        FARPROC pIsDebuggerPresent = GetProcAddress(hKernel32, "IsDebuggerPresent");
        if (pIsDebuggerPresent) {
            // Detours 라이브러리를 사용하여 Hooking
            DetourTransactionBegin();
            DetourUpdateThread(GetCurrentThread());
            DetourAttach(&(PVOID&)pIsDebuggerPresent, HookedIsDebuggerPresent);
            DetourTransactionCommit();
        }
    }
    // 테스트
    if (IsDebuggerPresent()) {
        printf("Debugger detected!\n");
    }
    else {
        printf("No debugger found.\n");
    }
    system("pause");
    return 0;
}

```

위 소스코드를 동일하게 빌드 후 전과 같이 테스트를 진행해보겠습니다.

![image.png](image%208.png)

당연히 디버거를 붙이지 않은 로컬에서 실행하면 `No debugger found.` 가 출력되어요. 그럼 이제 IDA를 통해서 바이너리에 디버거를 붙이면 어떻게 될까요!!

![image.png](image%209.png)

오우야... IDA를 통해 디버깅 중임에도 불구하고 디버거를 못찾았다고 실행되었습니다. 

![image.png](1b18ed6b-05c4-4b20-a412-522bd1c12bb3.png)

이렇게 간단하게 API 후킹을 통해 Static 안티디버깅을 무력화할 수 있습니다.

Static 안티디버깅은 다양한 시스템 정보를 사용하여 구현된 기법이기에 

- `NtQueryInformationProcess()` 와 같이 안티디버깅에 사용되는 **함수를 후킹**하거나
- PEB의 `BeingDebugged` 플래그 등 안티디버깅과 관련되는 정보를 **메모리 상에서 직접적으로** **수정**해서 안티디버깅을 우회할 수 있습니다.
    - 이 친구들은 보통 안티디버깅에 사용되는 함수들을 통해 값이 변조됩니다.

그러면 디버거의 원리를 역이용하는 **Dynamic 안티디버깅**은 어떻게 우회할 수 있을까요? Static 안티디버깅은 한 번 해제하면 이후 프로그램 디버깅에 문제가 없지만, Dynamic 안티디버깅은 실행 중 지속적으로 적용되므로 매번 우회해야 하거나, 보다 근본적인 방법으로 대응해야 합니다.

Dynamic 안티디버깅 기법을 우회하며 차이를 느껴보시죠! 

## Dynamic 안티디버깅 기법 우회 실습

### 1. 일반 프로그램

Dynamic 안티디버깅 실습도 “Part 1”에서 알아봤던 Timing 기반 안티디버깅 기법으로 진행해보겠습니다. 다음의 코드를 x86으로 빌드해주세요.

```c
// main.cpp
#include <windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h> 

// 키 복호화 함수
void decryptKey(char* key) {
    for (int i = 0; i < strlen(key); i++) {
        key[i] = key[i]-1; // 간단한 Ceaser 암호화
    }
}

// 중요한 데이터를 처리하는 함수
void processImportantData(char* key) {
    printf("Processing important data with key: %s\n", key);
    // 간단한 메시지를 출력합니다.
    printf("Important data processed successfully!\n");
}

int main(void) {
    char key[] = "nztfdsfulfz";

    // [1] 타이밍 기반 안티디버깅: 함수 실행 시간을 측정하여 비정상적으로 긴 시간 소요 시 디버거가 붙었다고 판단
    uint64_t start_time = GetTickCount64();
    decryptKey(key);
    uint64_t end_time = GetTickCount64();

    uint64_t elapsed_time = end_time - start_time;

    // [2] 1초 이상 걸리면 디버거가 붙었다고 판단
    if (elapsed_time > 1000) {
        printf("Debugger detected!\n");
        return 1; // 코드 종료
    }

    // [3] 키를 이용한 중요한 로직 실행: 복호화된 키와 비교하여 정상 동작 여부 판단
    if (strcmp(key, "mysecretkey") == 0) {
        processImportantData(key);
    }
    else {
        printf("Invalid key!\n");
    }

    system("pause");
    return 0;
}

```

이 코드는 타이밍 기반 안티 디버깅을 사용하여 중요한 로직, 즉 키 복호화 로직을 보호합니다([1]).  만약 공격자가 키 복호화 로직을 분석하려고 시도하면, 실행 시간의 차이가 발생하여 ([2]) 프로그램이 종료됩니다. 반대로, 정상적인 실행 환경에서는 ([3]) 로직이 문제없이 수행됩니다.

그럼 빌드하고 실행해봅시다!

![image.png](image%2010.png)

정상적으로 실행이 잘됩니다. 여기서는 간단하게 `Important data processed successfully!` 가 출력이 되지만 이후 키를 사용한 중요한 로직이 실행되는 시나리오라고 생각하시면 되겠습니다.

네 이제 정상적으로 실행했으니까 IDA를 통해서 해당 프로그램에 대해 디버깅을 시도해봅시다.

![image.png](image%2011.png)

아무래도 저희가 궁금한건 키 복호화 로직이니 복호화하는 함수에 Break Point(단축키: F2)를 걸어서 디버깅(F9)을 해봅시다. Step Into(F7)로 해당 함수 내부에 들어가서 로직을 본 후 어떻게 복호화되는지 파악 후 프로그램을 계속 실행시켜보면, 다음과 같이 디버거가 탐지되었다고 뜨면서 프로그램이 종료됩니다.

![image.png](2f9db531-4021-4c69-9805-b40c8cb60ee0.png)

비록 지금은 단순한 예제일지라도, 실행할 때마다 값이 변경된다면 디버깅을 통해 값을 추출하더라도 안티 디버깅 기법으로 인해 프로그램이 강제 종료되어 그 값을 활용할 수 없게 됩니다. 안티 디버깅은 사용자 기술 숙련도에 따라 보호 수준이 크게 달라지므로, 고도의 기술을 적용하면 분석을 더욱 어렵게 만들 수 있습니다.

### 2. 바이너리 패치 (Bypass Dynamic Anti-Debugging)

위 코드의 흐름상 어떤 부분을 패치하면 안티 디버깅을 우회할 수 있을까요? 

다음과 같이 2개의 경우를 생각해볼 수 있겠죠?

> **① 타이밍 체크를 무력화**
> 
> 1. `GetTickCount64()` 함수 호출 부분을 찾아서 패치
>     1. NOP으로 해당 명령을 무력화하거나 특정한 상수값을 반환하는 함수로 패치 등
> 2. `if (elapsed_time > 1000)` 부분을 `if (elapsed_time < 0)` 같은 **불가능한 조건**으로 변경
>     - 또는 `jmp`를 이용해 프로그램이 항상 정상적으로 진행되도록 수정
> 
> **② 프로그램 종료 방지**
> 
> `printf("Debugger detected!\n"); return 1;` 부분을 찾아서 `return 1;`을 **NOP(0x90)로 패치**
> 

컴파일러에 따라 분기문이 실제와 달라질 수 있으니까 바이너리상 어떻게 구현되었는지 IDA를 통해 한번 봅시다.

![image.png](image%2012.png)

저는 여기서 16번째 줄을 패치해보겠습니다. 어셈블리상으로는 0x00271E8E에 있는 코드인데요. 여기서 CF와 ZF가 0이면, 즉 `elapsed_time`이 1초보다 짧아야 하는데 저는 jbe를 ja로 패치해서 1초보다 길때만 키를 보는 로직으로 만들어보겠습니다. 16번째 줄을 누른 후 어셈블리를 확인해보면 2개의 jump 명령어가 있습니다. 첫번째 명령어는 NOP으로 패치하고 두번째 명령어는 런타임에 ja로 패치해보겠습니다.

![image.png](image%2013.png)

네 그러면 이제 아까처럼 키 복호화 함수에 Break Point를 걸어서 디버깅 시간을 1초를 넘긴 후 테스트해보겠습니다.

F9를 눌러 디버깅을 시작하면

![image.png](image%2014.png)

처음에 설정했던 Break Point를 통해서 `decryptKey` 함수를 분석할 수 있게 됩니다. 함수 분석을 끝낸 후 계속 디버깅을 이어가면 2번째 BP에 도달하게 됩니다.

![image.png](image%2015.png)

2번째 BP를 만나면 “Edit>Patch Program>Assemble“ 을 통해 jbe를 ja로 바꿔주면 아래와 같이 정상적으로 안티 디버깅을 우회함을 확인할 수 있습니다 ㅎㅎ

![image.png](image%2016.png)

이외에도 이번 실습과 같은 상황에서는 런타임에서 `CF`, `ZF` 플래그를 수정한다거나 GetTickCount() 함수를 후킹해서 상수값만을 반환하게 한다는 등으로도 충분히 디버거 검사 로직을 우회할 수 있습니다.

# 마무리

![Debug_part2.jpg](Debug_part2.jpg)

오늘 실습한 기법은 안티디버깅 우회를 위한 대표적인 예시들이었습니다. 이러한 기법은 "함정카드"처럼 보안 솔루션이 특정 동작을 막으려 할 때 이를 우회하는 방법으로 사용됩니다.

> Debugger를 막는 Anti-Debugging 기법을 우회하는걸 막는 Anti-Anti-Debugging ~~기법을 막는 Anti-Anti-Anti-Debugging 을 막는 Anti-Anti-Anti-Anti tititi 프레질프레질~~
> 
> 
> ![~~안티티티티티프래질프래질~~](Anti-Fragile.gif)
> 
> ~~안티티티티티프래질프래질~~
> 

## 안티 디버깅의 한계

안티디버깅은 코드 분석을 어렵게 만들어 역공학과 취약점 분석을 방해하는 핵심 기술입니다. 이를 통해 소프트웨어의 무단 복제, 크랙, 악용을 방지할 수 있으며, 특히 금융·게임·보안 제품 등 민감한 데이터를 다루는 분야에서 필수적으로 활용됩니다. (CTF에서도 종종 출제됩니다!) 다양한 환경에 맞춰 적용할 수 있는 유연성과 함께, 공격자가 디버깅을 시도할 때 이를 탐지하고 대응할 수 있다는 점에서 높은 실효성을 가집니다.

그러나 저희가 오늘 한 실습에서 보셨듯이 코드패치나 API 후킹 등에 취약합니다. 이는 해당 부분을 알기 때문에 패치가 가능한건데요. 

![image.png](image%2017.png)

> ???: 그럼 해당 부분을 모르게 하면 되지 않냐! / 000: 어떻게!!
> 

![9o6emu.gif](9o6emu.gif)

난독화를 통해 안티디버깅 로직을 알 수 없게 만듭니다. 안티디버깅 로직을 알 수 없으면 어떤 안티디버깅 기법이 적용된지 몰라서 우회방법을 알기가 더더욱 힘든데요. 이러한 Anti-Reversing 기술은 정적인 기법인 난독화와 동적인 기법인 안티디버깅이 결합되어서 쓰이면 분석가가 분석하기에 상당히 어렵고 복잡한 프로그램이 완성됩니다. (나중에 기회가 된다면 블로그에서 난독화도 다뤄보겠습니다!)

## 의의

악성코드 분석 관점에서 리버싱과 안티 리버싱은 마치 창과 방패의 싸움과 같습니다. 악성코드 분석가는 리버싱 기술을 통해 악성코드의 작동 방식을 파악하고, 악성코드 개발자는 안티 리버싱 기술을 통해 분석을 방해하여 악성코드의 비밀을 숨기려 합니다.

선량한 소프트웨어 개발자 관점에서도 리버싱과 안티 리버싱은 중요한 의미를 갖습니다. 리버싱 기술은 소프트웨어의 취약점을 찾아 보안을 강화하는 데 사용될 수 있으며, 안티 리버싱 기술은 지적 재산권을 보호하고 불법 복제를 방지하는 데 활용될 수 있습니다.

결국 리버싱과 안티 리버싱, 디버깅과 안티 디버깅은 서로의 기술을 발전시키며, 지속적인 대결을 이어가고 있습니다. 이런 관점에서 영화에서 나오는 끊임없는 영웅과 악당의 싸움처럼 느껴집니다. 악성코드 개발자와 분석가, 소프트웨어 개발자와 공격자 사이의 끝나지 않는 싸움이죠. 중요한 것은 기술 자체의 선악이 아니라, 기술을 사용하는 사람의 윤리적 책임입니다. 리버싱과 안티 리버싱 기술을 올바르게 활용하여 안전하고 신뢰할 수 있는 디지털 세상을 만들어가는 것이 우리의 과제이지 않을까요?!

![image.png](image%2018.jpg)

긴 글 읽어주셔서 감사드리고 저는 더 좋은 연구글로 돌아오겠습니다 🙂

### Reference

리버싱 핵심 원리: 악성 코드 분석가의 리버싱 이야기(저자 이승원)

Detoures - [https://secmem.tistory.com/480](https://secmem.tistory.com/480)

[https://www.apriorit.com/dev-blog/367-anti-reverse-engineering-protection-techniques-to-use-before-releasing-software](https://www.apriorit.com/dev-blog/367-anti-reverse-engineering-protection-techniques-to-use-before-releasing-software)

[https://www.bitdefender.com/en-us/blog/businessinsights/the-differences-between-static-malware-analysis-and-dynamic-malware-analysis](https://www.bitdefender.com/en-us/blog/businessinsights/the-differences-between-static-malware-analysis-and-dynamic-malware-analysis)
