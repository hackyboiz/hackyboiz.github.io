---
title: "[Research] Anti-Debugging Part 3(KR)"
author: OUYA77
tags: [Anti-Debugging, Debugging, Malware, OLLVM, OUYA77, Obfuscation, Reverse Engineering]
categories: [Research]
date: 2025-04-28 22:15:00
cc: true
index_img: /2025/04/28/OUYA77/Anti_part3/kr/image.png
---

안녕하세요 OUYA77 입니다. 🙂

나중에 기회가 된다면 블로그에서 난독화도 다뤄보겠다 했는데 마침 기회가 되어서(?) 이렇게 Part 3로 돌아왔습니다. 

Part 1에서는 디버깅과 안티 디버깅에 대해서 살펴보았고 Part 2에서는 "함정카드"처럼 안티 디버깅이 특정 동작을 막으려 할 때 이를 우회하는 방법인 안티 디버깅 우회에 대해서 알아보았습니다.

(혹시 아직 못보셨다면 👇 ㅎㅎ).

> *지난 파트 보러가기*
[[Research] Anti-Debugging Part1(KR)](https://hackyboiz.github.io/2024/12/29/OUYA77/Anti_part1/kr/)
[[Research] Anti-Debugging Part2(KR)](https://hackyboiz.github.io/2025/03/24/OUYA77/Anti_part2/kr/)
> 

긴 호흡으로 이야기가 전개된 만큼 까먹으셨을 수도 있을거라 생각이 드는데요. 혹시 그러셨을까봐 제가 간단한 카툰(?)을 준비해보았습니다. ~~만드는데 사심은 절대 섞지 않았어요. 만들다가 프레임을 짜는게 어려워서 전문가님의 도움을 받은 것도 비밀이랍니다.~~

## Anti-Dubbugging Part 요약

![image.png](image%201.png)

![image.png](image%202.png)

![image.png](image%203.png)

위 카툰에서도 보셨다시피 디버깅을 막기 위해 안티 디버깅이 나왔고 그걸 또 우회할 수 있다고 Part 1, 2에 걸쳐서 살펴보았습니다. 안티 디버깅의 한계는 **결국 프로그램이 실행되기 위해서는 메모리에 올라가야 하다 보니**, 그리고 **분석의 대상이 되는 바이너리 자체가 최종적으로 사용자에게 주어지다 보니**, 사용자가 안티 디버깅의 로직을 파악해서 패치하는 등으로 우회가 가능하였습니다. 그래서 고든 램지 선생님이 "이 재료(안티 디버깅 로직)가 뭔지 모르게 숨겨야 해!" 라고 불쑥 나타나셔서 난독화하라고 하신 거죠! 🔥👨‍🍳

> In Part 2…
> 
> 
> ![image.png](image%204.png)
> 
> > ???: 그럼 해당 부분을 모르게 하면 되지 않냐! / 000: 어떻게!!
> > 

난독화가 적용된 안티디버깅이 걸린 프로그램은 어떤 부분이 안티디버깅 로직인지 알 수 없고 그렇기에 분석하기가 더욱 까다롭습니다. 그래서 이번 Part 3에서는 이 **난독화**라는 흥미로운 녀석을 집중적으로 파헤쳐 보겠습니다!

## 1. 난독화(Obfuscation)

### 1.1 소개

- **난독화(Obfuscation)**란 말 그대로 프로그램의 코드를 읽기 어렵게 만드는 기술입니다. Wikipedia에서는 다음과 같이 정의합니다:

> "소프트웨어 난독화란, 소프트웨어의 소스코드 또는 머신코드를 난독화하여 사람 또는 분석 도구가 이해하거나 분석하기 힘들게 만드는 일이다."
> 

이는 단순한 코드 가독성 저하를 넘어서, **리버스 엔지니어링, 디버깅, 정적 분석 등을 방해**하고, 지적 재산 보호, 보안 강화, 악성 코드 분석 회피 등 다양한 목적에 사용됩니다.

### 1.2 목적

난독화는 프로그램의 동작을 변경하지 않으면서도, 다음과 같은 분석 방지 목적을 달성합니다:

- **가독성 저하:** 코드의 논리 구조와 의미를 숨겨 분석자의 이해를 방해
- **구조적 복잡성 증가:** 불필요한 연산, 비선형적 흐름, 난수 기반 조건 등 삽입
- **패턴 탐지 회피:** 시그니처 기반 탐지 도구의 오탐 또는 탐지 회피 유도

그래서 분석가가 “아~ 곤란하네 ;;” 이렇게 생각이 들었다면 난독화의 목적을 어느정도 이뤘다고 봐도 무방합니다. 

### 1.3 난독화의 분류

난독화는 크게 두 가지 범주로 나눌 수 있습니다:

1. **소스코드 난독화 (Source Code Obfuscation)**

- **적용 대상**: 사람이 작성한 고수준 언어 (Java, JavaScript, Python 등)
- **적용 시점**: 컴파일 이전
- **목적**: 내부 보안 확보, 외주/고객용 코드 일부 공개 시 플로우 보호

소스코드 난독화는 종종 내부 또는 고객에게 소스 일부를 공개해야 할 경우, 전체 로직이나 알고리즘이 유출되지 않도록 사용됩니다. 코드를 읽는 사람은 일부 수정은 가능하지만, 전체 구조나 알고리즘을 파악하기 어렵게 됩니다. 예시 기법으로는 “식별자 이름 변경”, “코드 블록 재배열”, “제어 흐름 난독화”, “문자열 인코딩”을 들 수 있습니다.

2. **바이너리 난독화 (Binary Obfuscation)**

- **적용 대상**: 컴파일된 실행파일 (C, C++, Go 등)
- **적용 시점**: 컴파일 시 혹은 이후 변형
- **목적**: 제품 보호, 알고리즘 유출 방지, 크랙/리버싱 방지

바이너리 난독화는 **실제 배포되는 제품 보호의 핵심**입니다. 상용 소프트웨어는 대부분 바이너리 형태로 배포되며, 이 안에는 중요한 알고리즘, 라이선스 로직, 인증 체계 등이 포함됩니다. 이를 보호하지 않으면, 공격자나 고객이 이를 리버스 엔지니어링을 하거나 크랙하는 것이 가능해집니다.

> 여기서는 “제품 보호”라고 했는데 악성코드도 제품이라면 제품이긴 하죠.. 대부분의 악성코드에도 이 바이너리 난독화가 들어갑니다.
> 

바이너리 난독화는 분석 도구(IDA, Ghidra 등)의 정적 분석을 방해하고 디버깅 흐름을 교란하며, 심지어는 실행 시간에만 코드가 해독되는 형태로 구현되기도 합니다. 예시 기법으로는 “제어 흐름 변형”, “문자열 암호화”, “가짜 코드 삽입”, “네임 맹글링”, “간접 점프 및 호출” 등이 있습니다.

### 1.4 바이너리 난독화

이번 파트에서는 특별히 **바이너리 난독화**에 집중하려 합니다. 왜 바이너리 난독화가 중요한가? 물으신다면, 이유는 간단합니다.  우리가 쓰는 프로그램 대부분은 결국 **바이너리** 형태로 제공되는데, 여기엔 개발사의 핵심 노하우, 알고리즘, 라이선스 정보 등이 담겨있기 때문입니다. 이를 보호하지 않으면 크래킹, 불법 재배포, 심지어 악성코드 악용까지 이어져 단순한 기술적 공격을 넘어 심각한 비즈니스 피해를 야기할 수 있습니다. 그렇기 때문에 필요한게 Anti-Reversing 기법이고 우리는 그에 대해서 Part 1부터 계속 봐오고 있습니다. 

안티 디버깅이 동적 분석, 즉 실행 중인 프로그램의 내부를 들여다보려는 시도를 적극적으로 감지하고 방해한다면, 난독화는 **정적 분석**을 방해하는 데 초점을 맞춥니다. 이 둘이 결합될 경우, 분석자 입장에서는 그야말로 무적과 같은 방어 체계를 마주하게 되는 셈이죠.

![image.png](image.png)

> ~~(절대 뚫리지 않는 방패도 뚫는 창이 있긴 하죠…)~~
> 

## 2.바이너리 난독화의 세 가지 축

바이너리 난독화는 일반적으로 다음 세 가지 카테고리로 분류할 수 있습니다. 

(일부분은 간단하게 짚고 넘어가겠습니다.)

### 2.1 구조적 난독화 (Structural Obfuscation)

> 코드의 제어 흐름 구조 자체를 뒤틀어 분석이 어려워지게 만드는 난독화
> 

이 범주는 주로 **정적 분석 도구가 함수나 블록 단위로 프로그램을 분석할 때**, 구조를 파악하기 어렵도록 만듭니다.

- **제어 흐름 평탄화 (Control Flow Flattening)**
    
    ![ref. https://gosecure.github.io/presentations/2020-05-15-advanced-binary-analysis/](image%205.png)
    
    ref. https://gosecure.github.io/presentations/2020-05-15-advanced-binary-analysis/
    
    - 원래는 `if`, `while`, `switch` 등으로 표현되던 논리 흐름을, 모두 하나의 `dispatcher` 루프 안에 넣고, **상태 변수**를 기반으로 다음 실행할 블록을 결정하는 방식으로 바꿉니다.
        - 모든 흐름이 루프 내부의 상태 전이로 표현되어 정적 분석 도구의 흐름 트리 분해가 무력화됩니다.
        - 따라서 흐름이 비직관적으로 변경되어, 실제 실행 경로를 추적하기 어려워집니다.
    - 이 부분은 뒤에서 Obfuscator-LLVM(OLLVM) 예제를 통해 살펴보도록 하겠습니다!
- **가짜 코드 삽입 (Junk / Dead Code Insertion)**
    - 프로그램의 실제 동작에는 아무런 영향이 없는 의미 없는 코드(Junk Code)나, 실제로는 절대 실행되지 않는 코드(Dead Code)를 삽입하는 기법입니다.
    - 대표적인 예로는 다음과 같습니다:
        - 의미 없는 연산 (`x = x;`, `add eax, 0;`)
        - 실행되지 않는 분기 (`if (false) { ... }`)
- **간접 분기 / 호출 (Indirect Branching)**
    - `jmp eax`, `call [ecx+0x10]`처럼 주소 계산 후 점프합니다.
    - 이로써 코드 흐름을 추적할 수 없게 만듭니다(API 추론 불가).
- **Opaque Predicate**
    - 조건식이 항상 true/false인 분기 삽입합니다. (예: `if ((x*x) >= 0)`)
    - 이 코드는 동적으로는 무해하나 정적으로는 흐름 분석을 방해합니다.
- **주의할 점:**
    - 컴파일 시 최적화 옵션(`O2`, `O3`)을 주면 구조가 최적화 되면서 의미 없는 코드들이 제거될 수 있습니다.
    - 따라서 난독화 시에는 **최적화 옵션을 비활성화**하거나, **런타임 변수를 사용해 제거를 방지하는 기법**을 함께 적용해야 합니다.
- **효과:**
    - 분석자는 불필요한 루프나 분기 속에 숨어 있는 실제 코드를 찾아야 해서 심리적으로 지치고 허탈함을 느끼게 됩니다. 다음과 같이 왼쪽 사진처럼 생긴 프로그램의 흐름에서 오른쪽 아래의 더미코드를 간단하게 몇 부분만 넣어도 오른쪽과 같이 프로그램의 흐름이 더 복잡해짐을 볼 수 있습니다.
        
        ![image.png](image%206.png)
        

🔍 *구조적 난독화는 주로 분석 도구의 CFG 분석과 흐름 파악을 무력화합니다.*

### 2.2 데이터 난독화 (Data Obfuscation)

> 데이터 자체를 숨기거나 다르게 표현하여 의미를 유추하기 어렵게 만드는 난독화
> 

이 범주는 주로 문자열, 상수, API 명 등 **분석 시 유용한 단서**를 제거하거나 숨기기 위한 기법입니다.

- **문자열 암호화 (String Encryption)**
    - 중요한 문자열을 런타임 복호화 방식으로 전환합니다.
    - 정적 분석 시 `"CreateProcess"` 같은 문자열이 보이지 않습니다.
- **상수 인코딩 (Constant Encoding)**
    - 숫자 상수나 플래그 값을 간접 계산으로 표현합니다. (예: `1337` → `1000 + 337`)
    - 이로써 즉각적으로 상수의 의미를 파악할 수 없게 되고, 특히 디컴파일된 코드가 산술식으로 복잡하게 표현되면서 프로그램의 목적이나 상태를 유추하는 데 걸리는 시간을 대폭 증가시킵니다.
- **API 호출 난독화 / Import Hiding**
    - API 이름을 해시나 암호화로 숨기고, 런타임에 동적 로딩(`LoadLibrary`, `GetProcAddress`)합니다.
    - IAT(Import Address Table)가 무력화되어 자동 분석 불가합니다.

🔍 *데이터 난독화는 분석자나 도구가 의미 있는 정보(식별자, 문자열, API 등)를 추출하는 것을 방해합니다.*

### 2.3 알고리즘 난독화 (Algorithm Obfuscation)

> 실제 기능이나 로직을 복잡하게 만들어 기능을 파악하기 어렵게 만드는 난독화
> 

이는 프로그램이 수행하는 **행위 자체**를 난해하게 하여, 코드를 뜯어봐도 "무엇을 하는지"를 알기 어렵게 만듭니다.

- **네임 맹글링 (Name Mangling) 및 심볼 제거**
    - 디버깅 심볼을 제거하거나, 함수/변수 이름을 무작위 문자열로 대체합니다.
    - 이로써 전체적인 기능 맥락을 읽기 어렵게 합니다.
    - 다음은 Hack The Drone 2024(Final) 에 나온 문제 중 패킷을 인코딩해서 비행을 위한 시동 패킷을 만드는 문제의 일부분입니다. 해당 문제에서는 수도 랜덤 생성에서 사용되는 LFSR Galois 알고리즘을 이용했는데요.
    
    ![image.png](image%207.png)
    
    - 위 사진을 보시면 함수 이름을 통해 어떤 알고리즘이 적용되었는지랑 각각의 파라미터들이 어떻게 처리가 되는지 쉽게 파악할 수 있습니다.
    
    ![image.png](image%208.png)
    
    - 그러나 네임 맹글링이 적용되면 위와 같이 함수명과 파라미터 변수명이 무작위 문자열로 대체되어 전체적으로 어떤 알고리즘인지 파악하는데 시간이 보다 소요됩니다.
        - 요즘에는 그런데 GPT 같은 LLM에 넣어주면 너무 잘 알고리즘을 파악해주긴 하더라구요 ㅎ.ㅎ…
- **Self-Modifying Code**
    - 실행 중 자신의 코드를 메모리에서 수정합니다.
    - 따라서 정적 분석으로는 완전한 코드 구조를 볼 수 없습니다.
- **가상 머신 기반 난독화 (Virtualization Obfuscation)**
    - 자체 가상 ISA를 만들어 원래 코드를 바이트코드로 변환합니다.
    - 그렇기에 가상머신 해석 루틴 없이 실제 로직을 이해할 수 없습니다.

🔍 *알고리즘 난독화는 전체 프로그램이 **무엇을 수행하는지**조차도 파악하기 어렵게 만듭니다.*

## 3. Obfuscator-LLVM 실습

**OLLVM(Obfuscator-LLVM)** 은, 오픈소스 컴파일러 프레임워크인 **LLVM**을 기반으로 만들어진 **난독화 기능 확장판**입니다. 원래 LLVM은 "클린한" 코드를 생성하는 데 초점을 맞춘 컴파일러 프레임워크입니다. 그런데 연구진들이 LLVM의 구조가 워낙 유연하다는 점에 착안해서, **코드를 빡세게 난독화하는 기능**을 추가한 버전을 만들었죠. 그게 바로 OLLVM입니다.

OLLVM은 C/C++ 코드를 컴파일할 때 **자동으로 난독화 패스(pass)** 를 거치면서 코드를 변형합니다. 덕분에 개발자는 소스 코드를 따로 복잡하게 수정할 필요 없이, 그냥 컴파일만 다르게 해도 강력한 난독화 바이너리를 만들 수 있습니다.

### 3.1 OLLVM용 Clang 빌드 방법

[https://github.com/obfuscator-llvm/obfuscator/wiki/Installation](https://github.com/obfuscator-llvm/obfuscator/wiki/Installation)

```bash
$ git clone -b llvm-4.0 https://github.com/obfuscator-llvm/obfuscator.git
$ mkdir build && cd build
$ cmake -DCMAKE_BUILD_TYPE=Release ../obfuscator/
$ make -j7
```

저 같은 경우에는 에러가 좀 나서 소스코드를 수정해줬어요. 혹시 에러가 나신다면 로그 참고하시고 트러블슈팅해보세요.

obfuscator/tools/clang/lib/CodeGen/CGOpenMPRuntime.cpp:6275, 6321, 6400

`error:lambda parameter ‘CGF’ previously declared as a capture` 

앞에 있는 `&CGF` 제거

```c
auto &&BeginThenGen = [&D, &CGF, Device, &Info, &CodeGen, &NoPrivAction](...)
↓
auto &&BeginThenGen = [&D, Device, &Info, &CodeGen, &NoPrivAction](...)
```

obfuscator/include/llvm/ExecutionEngine/Orc/OrcRemoteTargetClient.h:696

`error: cannot convert`

`char` → `uint8_t` 로 변환

```c
Expected<std::vector<char>> readMem(...) { ... }
↓
Expected<std::vector<uint8_t>> readMem(...) { ... }

```

![image.png](image%209.png)

빌드를 잘했으면 위와 같이 OLLVM 용 clang이 잘 나타납니다!

### 3.2 OLLVM 사용법

Obfuscator-LLVM을 사용하는 가장 간단한 방법은, Clang을 통해 LLVM 백엔드(-mllvm)에 특정 플래그를 전달하는 것입니다. 현재 사용할 수 있는 주요 플래그는 다음과 같습니다:

[https://github.com/obfuscator-llvm/obfuscator/wiki/Features](https://github.com/obfuscator-llvm/obfuscator/wiki/Features)

- `fla`: 제어 흐름 평탄화(Control Flow Flattening)
- `sub`: 명령어 치환(Instruction Substitution)
- `bcf`: 가짜 제어 흐름(Bogus Control Flow)

또, 특정 함수에만 난독화를 적용하고 싶다면 [함수 어노테이션](https://github.com/obfuscator-llvm/obfuscator/wiki/Functions-annotations) 방법도 참고할 수 있습니다.

### 3.3 실습

- 예제 프로그램

```c
// flatten_example.c
#include <stdio.h>

int secret_function(int x) {
    if (x > 0) {
        return x * 2;
    } else {
        return x - 2;
    }
}

int main() {
    int value = 5;
    int result = secret_function(value);
    printf("Result: %d\n", result);
    return 0;
}
```

위 코드를 저장해 아래와 같이 빌드해줍니다.

```bash
$ clang flatten_example.c -o flatten_example_norm
$ path_to_the/build/bin/clang -mllvm -fla flatten_example.c -o flatten_example_obf
```

네 이제 IDA를 통해서 빌드된 바이너리를 분석해봅시다. 왼쪽이 `flatten_example_norm` 이고 오른쪽이 `flatten_example_obf` 입니다.

![image.png](image%2010.png)

디스어셈블된 `secret_function` 코드

![image.png](image%2011.png)

`secret_function` 의 CFG

아주 간단한 프로그램인데 벌써부터 극명한 차이가 보이죠?

그리고 너무나 당연한 이야기지만 실행결과는 같습니다.

![image.png](image%2012.png)

## 4. 마무리

### 한계

물론 난독화에도 명확한 한계는 존재합니다.

- **성능 저하:** 난독화 과정에서 삽입되는 불필요한 로직이나 복잡한 분기는 프로그램 성능을 저하시킬 수 있습니다.
- **도구 발전의 역습:** 정적 분석기, 디옵퓨스케이터(Deobfuscator) 등 분석 도구들도 끊임없이 발전하면서, 일부 난독화는 비교적 쉽게 무력화될 수 있습니다.
- **법적·윤리적 문제:** 상용 소프트웨어에 과도한 난독화를 적용할 경우, 사용자의 신뢰를 저하시킬 수 있으며, 때로는 법적 분쟁의 원인이 될 수도 있습니다.

### 의의

지금까지 살펴본 것처럼, 난독화 기법은 그 작동 방식에 따라 크게 **구조적(Structural)**, **데이터(Data)**, **알고리즘(Algorithm)** 측면으로 구분해볼 수 있습니다. 다만, 실제 소프트웨어 분석 환경에서는 이러한 기본적인 분류를 넘어, 이 세 가지 요소가 복합적으로 적용되기도 하고 더욱이, 기존 난독화 기법의 변형은 물론, 새로운 아이디어를 기반으로 한 다양한 변종 기법들이 꾸준히 등장하며 분석의 복잡성을 더하고 있습니다.

난독화가 빡세게 걸려있다고 소문난 상용 드론 DJI 에서의 난독화 기법에 대해서 일부분만 간단하게 살펴보겠습니다. 

- **구조적 난독화**: 8개의 DEX 파일을 128KB 청크 단위로 분할하고, 각각을 **AES-256**으로 암호화
- **데이터 난독화**: `com.dji.industry.pilot` 문자열과 하드코드된 상수를 조합하여 **XOR 기반 키**를 생성
- **알고리즘 난독화**: JNI 호출 시 **3단계 트램펄린 함수**를 거쳐 실제 네이티브 함수 주소로 리다이렉션

이처럼 현대의 난독화 기술은 단일 기법에 머무르지 않고 다층적이고 모듈화된 보호 체계를 갖추고 있으며,  매년 새로운 변형 기법을 도입하여 분석 환경을 더욱 복잡하게 만듭니다. 나아가, 인공지능 기반의 가짜 코드 패턴 생성이나 양자 암호 기술을 활용한 키 교환 등 차세대 기술을 시험적으로 도입하는 사례도 등장하며 난독화 기술도 전체적인 기술의 발전에 따라 발전하고 있습니다.

Part 1,2,3을 진행하며 계속한 이야기지만, 

> "모든 보호 계층은 시간을 벌어주는 장치일 뿐, 궁극적인 안전을 보장하지 못함"
> 
> - Quarkslab 리서치 팀

난독화와 안티 디버깅 기술은 결국 **코드를 지키기 위한 싸움**입니다. 리버서와 개발자의 대결 구도 속에서 기술은 끊임없이 진화하고, 이 과정은 보안 기술 전반의 발전으로 이어집니다. 특히 난독화는 합법적인 소프트웨어 보호뿐 아니라, 악성코드의 탐지 회피와 분석 지연에도 적극적으로 활용되고 있습니다. **공격과 방어 양쪽 모두에서 필수적인 기술**로 자리 잡은 셈이죠.

난독화는 공격을 지연시키고 방어의 시간을 벌어주는 수단이지만, 완벽한 방패는 존재하지 않습니다. 분석자는 끈질기게 흐름을 추적하고 도구를 개선하며, 개발자는 이에 대응해 새로운 난독화 기법을 찾아내 방어를 강화합니다.

**결국 이 싸움은 끝날 수 없습니다.**

기술이 발전하는 만큼, 공격과 방어의 방법도 끊임없이 진화하기 때문입니다.

우리는 이 기술을 단순한 장벽이 아니라, **더 나은 보안 환경을 만들어가기 위한 진화의 일부**로 바라봐야 할 것입니다. 창과 방패의 대결 속에서도, 우리의 목표는 변함없습니다.

![image.png](image%2013.png)

> “게임은 계속된다. 그리고 우리는, 각자의 이유로, 이 끝나지 않는 게임에 뛰어든다.”
> 

### Reference

리버싱 핵심 원리: 악성 코드 분석가의 리버싱 이야기(저자 이승원)

[https://m.blog.naver.com/choijo2/60196192627](https://m.blog.naver.com/choijo2/60196192627)

[http://en.wikipedia.org/wiki/Obfuscation_(software)](http://en.wikipedia.org/wiki/Obfuscation_(software))

[https://blog.quarkslab.com/dji-the-art-of-obfuscation.html](https://blog.quarkslab.com/dji-the-art-of-obfuscation.html)

[https://gosecure.github.io/presentations/2020-05-15-advanced-binary-analysis/](https://gosecure.github.io/presentations/2020-05-15-advanced-binary-analysis/)

[https://github.com/obfuscator-llvm/obfuscator](https://github.com/obfuscator-llvm/obfuscator)