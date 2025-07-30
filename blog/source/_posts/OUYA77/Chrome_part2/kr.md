---
title: "[Research] Type Confusion 101으로 시작하는 Chrome Exploit ^-^☆Part 2.(KR)"
author: OUYA77
tags: [Type Confusion 101, Chrome, Chromium, OUYA77, Type Confusion, Relative R/W, OOB]
categories: [Research]
date: 2025-07-30 14:20:00
cc: true
index_img: /2025/07/30/OUYA77/Chrome_part2/kr/TypeConfusion101_Part2.png
---

안녕하세요 OUYA77 입니다. 2025년도 벌써 반이나 지났는데, 게다가 7월도 다지나갔네요. 시간 참 빠르네요. 여러분도 그렇게 생각하시나요? ㅎ.ㅎ

이번 파트에서는 크롬 익스플로잇 찍먹정도는 해보고 싶었는데, 아무래도 크롬이라는 프로그램 자체가 복잡하고 알아야 할 background 내용이 좀 있어서 글이 길어졌네요. 이번 파트에서는 기본적인 Type Confusion과 Chrome에서의 Type Confusion을 비교해보고 이 Type Confusion 이 어떻게 발생하고 결국 Memory Corruption이 어떻게 발생하는지 알아보겠습니다.

그럼 본격적으로 들어가기 전에 지난 시간 내용을 간단히 살펴보고 가겠습니다.

## 0. Part 1. Recap

지난 시간에는 Chrome 전체 아키텍쳐를 살펴보며 프론트엔드 리소스가 어떻게 처리되는지 보았어요. 조금 더 자세히 이야기하자면, HTML, CSS, JS 파일이 크롬에서 렌더링되는 과정에 대해 살펴보았는데 이번 연구글에서는 그 중 JS 파일을 처리하는 V8에 대해서 살펴보려고 합니다.

> 안보셨다면 → [[Research] Type Confusion 101으로 시작하는 Chrome Exploit ^-^☆ Part 1.](https://hackyboiz.github.io/2025/07/01/OUYA77/Chrome_part1/kr/)
> 

![image.png](image%201.png)

V8은 빠르고 효율적인 JavaScript 실행을 위해 다양한 컴파일러 계층(Ignition, SparkPlug, Maglev, TurboFan)을 사용하는데, 이 과정에서 코드를 점점 더 최적화하면서 실행 속도를 높입니다. 하지만 이 최적화는 “항상 이 구조일 것이다”라는 가정에 기반하기 때문에, 만약 공격자가 특정 시점에 객체 타입이나 구조를 바꿔버린다면, V8은 잘못된 가정 하에 잘못된 네이티브 코드를 실행하게 됩니다. 이것이 바로 **Type Confusion**이 발생하는 대표적인 시나리오입니다.

이제 V8 내부에서 코드가 어떻게 실행되는지 기본 구조를 이해했으니, 다음 단계에서는 이 엔진 내부에서 객체가 어떻게 표현되고, 최적화되며, 나아가 이 구조가 어떻게 **Type Confusion**과 같은 보안 이슈로 이어질 수 있는지를 알아보겠습니다!

## 1. Type Confusion 101

### 1.1 Introduction

Type Confusion은 말그대로 `Type`이 `Confusion` 되었기에 발생하는 취약점입니다. 그럼 여기서 말하는 타입(Type)이란 무엇일까요? 컴퓨터에서 '타입'이란 데이터를 저장하고 처리하는 방식을 의미합니다. 간단히 말해, 우리가 사용하는 변수의 데이터 종류를 구분짓는 역할을 하죠.

예를 들어:

- **정수**(Integer): 숫자 데이터를 저장
- **문자**(Char): 단일 글자 데이터를 저장
- **배열**(Array): 여러 데이터를 하나로 묶음

이처럼 타입은 프로그래밍 언어에서 데이터를 처리하는 기본 단위로, 정확한 타입이 지정되어야 컴퓨터가 데이터를 올바르게 처리할 수 있습니다. 그런데 만약, 어떤 데이터가 원래 `정수`여야 하는데, 실수로 (혹은 의도적으로) `문자`나 `객체`처럼 다뤄진다면 어떻게 될까요? 이로 인해서 발생하는 side effect를 예측할 수 없고  프로그램은 예상치 못한 방식으로 동작할 수 있습니다.

### 1.2. Type Confusion 유형

**Type Confusion**은 일반적으로 다른 취약점 분류와 마찬가지로, 그 영향에 따라 Logical Bug와 Memory Corruption 두 가지 유형으로 나눌 수 있습니다. 

- **Logical Bug**

**Logical Bug** 유형은 타입에 따라 달라지는 로직 처리가 잘못 동작하면서 발생하는 취약점입니다. 많은 언어에서 연산자나 내장 메서드는 입력 타입에 따라 다른 방식으로 동작합니다. 타입이 혼동되면, 의도한 흐름과 전혀 다른 결과가 나타날 수 있습니다. 아래와 같은 코드가 전체 어플리케이션 코드 중 일부로 있다고 가정해봅시다.

![image.png](image%202.png)

개발자가 XSS 방지를 위해 특정 입력값에 대해 `<` 문자가 포함되어 있는지를 검사하는 필터를 구현했습니다. 이때 검사 대상이 문자열일 것으로 예상하고 필터를 구현했지만, 실제로는 문자열로 이루어진 배열 객체가 전달된다면 어떻게 될까요? 자바스크립트에서는 이 객체를 직접 검사할 경우 `<`이 포함되어 있더라도 비교 결과가 `false`가 될 수 있습니다. 이는 필터 우회를 야기하며, 결과적으로 XSS(Cross-Site Scripting)나 권한 상승과 같은 심각한 논리적 취약점으로 이어질 수 있습니다.

- **Memory Corruption**

**Memory Corruption** 유형은 잘못된 타입 캐스팅으로 인해 메모리 레이아웃이 어긋나면서 발생하는 치명적인 취약점입니다. 특히 C/C++ 같은 언어에서는 객체의 크기나 구조를 잘못 해석한 결과, 인접한 메모리에 의도하지 않은 접근(Out-Of-Bounds Read/Write) 이 가능해지며, 이는 프로그램의 안정성과 보안을 위협하는 주요 원인이 됩니다.

예를 들어, 다음 코드를 살펴봅시다.

```c
typedef struct {
    int value[2];
} SmallStruct;

typedef struct {
    int data[4];
} LargeStruct;

SmallStruct normal = {0x41, 0x42};
print_memory((int*)&normal , sizeof(normal)/sizeof(int)); 

// SmallStruct 배열을 LargeStruct로 캐스팅
LargeStruct* confused = (LargeStruct*)&normal ;
confused->data[2] = 0xdead; // OOB Write
confused->data[3] = 0xbeef;
print_memory((int*)&normal, sizeof(LargeStruct)/sizeof(int)); // OOB Read
```

위 예제에서 `SmallStruct`는 정수 두 개를 담고 있는 구조체입니다. 반면, `LargeStruct`는 정수 네 개로 이루어진 배열을 포함하고 있습니다. 개발자가 `SmallStruct` 배열을 `LargeStruct` 포인터로 **잘못 캐스팅**하면, 프로그램은 `normal`의 크기를 `LargeStruct`의 메모리 레이아웃으로 오인하게 됩니다.

출력 결과를 보면,

![image.png](image%203.png)

`SmallStruct`구조체의 범위를 벗어난 영역이 쓰기로 인해 변형되었음을 확인할 수 있습니다. 즉, `confused->data[2]`와 `data[3]`는 실제 `normal`의 정의된 구조체 크기를 넘어서 접근한 것입니다.

이를 도식화하면 다음과 같습니다.

![image.png](image%204.png)

그림에서는 `SmallStruct` 인스턴스가 총 2개의 정수 공간만을 차지하고 있음에도 불구하고, `LargeStruct` 포인터를 통해 총 4개의 정수 공간이 존재하는 것처럼 접근하며 메모리 경계를 초과한 읽기/쓰기가 발생하는 것을 보여줍니다.

### 1.3 Type Confusion (case: in Chrome)

저희는 크롬에서 발생하는 Type Confusion에 대해 알아볼거기 때문에 Memory Corruption 류의 두 가지 전형적인 사례(C++ / JavaScript)를 비교하며 크롬에서 이 취약점이 어떻게 나타나는지를 더 알아보겠습니다.

- **Type Confusion – C++**

정적 타입 언어인 C++에서는 Type Confusion이 주로 개발자의 실수로 인해 발생하며, 컴파일 타임에 타입이 결정되기 때문에 잘못된 명시적 캐스팅이 주된 원인입니다.

```jsx
// 잘못된 Casting으로 발생한 Type Confusion
struct A { int x; };
struct B { int y; void (*func)(); };

A a = {42};
B* b = (B*)&a; 
b->func();     // 정의되지 않은 함수 포인터를 호출 → Crash 또는 임의 코드 실행
```

위 예제에서 `struct A`는 정수형 멤버 `x`만을 포함하지만, 이를 `struct B`로 캐스팅하면 존재하지도 않는 함수 포인터(`func`)를 접근하려 하게 됩니다. 이는 정의되지 않은 동작을 유발하며, 시스템 크래시가 발생하거나 공격자가 삽입한 코드가 실행될 수도 있습니다.

- **Type Confusion – JavaScript**

반면, JavaScript와 같은 동적 언어에서는 Type Confusion이 개발자가 직접 명시적으로 타입을 바꾸지 않아도 발생할 수 있습니다. 특히 V8과 같은 최신 JS 엔진에서는 JIT(Just-In-Time) 컴파일러가 런타임 성능 최적화를 위해 타입 정보를 추론하는 과정에서 잘못된 가정을 할 수 있고, 이로 인해 Type Confusion이 유발될 수 있습니다(자세한 내용은 뒤에서 다루겠습니다!).

```jsx
function foo(x) {  
    let arr = [1.1]; // 초기에 숫자(float)가 들어간 배열
    if (x) arr[0] = {y: 42}; // 조건에 따라 객체로 바꾸는 분기
    return arr[1]; // OOB 가능성
}

for (let i = 0; i < 1000; i++) foo(false);  // 최적화를 유도

foo(true); // 최적화 이후 객체 삽입 → 타입 구조 변경
```

이 코드는 처음에는 배열이 숫자만 담고 있어서, V8은 "이 배열은 숫자만 담긴다" 고 판단하고, 그에 맞게 내부 구조를 단순하고 빠르게 처리하도록 최적화합니다. 그러나 조건문에 따라 객체를 배열에 삽입하면 내부 구조가 바뀌어야 하는데, 이미 만들어진 최적화된 코드가 이 변화를 고려하지 않으면, 엔진은 여전히 이 배열이 숫자만 들어 있는 줄 알고 접근하게 됩니다. 그 결과, 원래는 숫자라고 가정했던 값이 실제로는 객체이거나 전혀 다른 데이터일 수 있고, 이로 인해 메모리 해석이 왜곡되거나 배열의 경계를 벗어난 접근(OOB)이 일어날 수 있습니다. 이처럼 런타임 중 발생하는 구조 변경과 JIT 컴파일러의 추론 사이의 불일치는 자바스크립트에서 Type Confusion의 주요 원인이 됩니다.

## 2. Type Confusion in Chrome - Root cause Analysis

앞서 1장에서는 Type Confusion의 개념과 일반적인 발생 사례를 살펴보았습니다.

이제부터는 이 개념을 Chrome의 JavaScript 엔진인 V8에 적용해보겠습니다. V8은 JIT 기반의 고성능 엔진으로, Type Confusion 취약점이 빈번히 발생하는 환경입니다. 성능 최적화를 위해 V8은 객체의 타입 정보를 내부적으로 고정된 형태로 유지하려고 하며, 이로 인해 특정 조건에서 타입 추론 오류가 메모리 손상으로 이어질 수 있습니다. 이 장에서는 V8 내부에서 Type Confusion이 어떤 구조적 배경에서 발생하는지, 그리고 Hidden Class와 ElementsKind 같은 핵심 개념이 어떤 역할을 하는지를 살펴보겠습니다.

### 2.1 Chrome에서의 Type Confusion 사례 개요

실제로 CVE-2018-17463**,** CVE-2023-2033, CVE-2023-4069 등 수많은 Chrome 취약점들이 잘못된 타입 추론에 의한 Type Confusion에서 비롯되었습니다. (해키보이즈 하루한줄에도 있어요! → [링크](https://hackyboiz.github.io/2024/01/18/ogu123/cve-2023-2033/)) 

이들 대부분은 JIT 컴파일러가 코드 실행 패턴을 관찰한 뒤 “이 변수는 항상 숫자만 다룬다”, “이 객체는 항상 같은 구조를 가진다” 같은 가정을 바탕으로 최적화 코드를 생성했지만, 이후 예외적인 실행 흐름에서 이 가정이 깨졌을 때 발생한 것입니다.

V8 엔진에서는 다음과 같은 상황에서 Type Confusion 취약점이 자주 발생합니다:

- 객체 구조(Hidden Class)가 바뀌었음에도 기존 가정으로 동작하는 경우
- 배열 내부 타입(ElementsKind)의 변경이 반영되지 않은 경우
- Inline Cache가 남긴 잘못된 타입 힌트를 사용하는 경우

정리하면, V8은 빠른 실행을 위해 “타입은 고정되어 있을 것”이라는 전제 하에 코드를 생성합니다. 그러나 자바스크립트는 매우 동적인 언어이며, 이 전제는 언제든 깨질 수 있습니다. 공격자는 이러한 구조적 특성을 이용해 내부 타입 정보를 교란시켜 Type Confusion을 발생시킬 수 있습니다.

이러한 구조적 가정의 기반이 되는 **Hidden Class (Map)**와 **ElementsKind**라는 두 가지 핵심 최적화 개념에 대해 자세히 살펴보겠습니다.

### 2.2 V8의 최적화 모델 소개(1) : HiddenClass(Map)

Hidden Class는 V8이 JavaScript 객체의 구조를 추적하고 최적화하기 위해 사용하는 내부 메커니즘입니다. 자바스크립트 객체는 기본적으로 속성과 값 쌍을 자유롭게 추가하거나 제거할 수 있어 구조가 동적으로 변합니다. 하지만 V8은 성능을 위해 객체를 딕셔너리처럼 처리하지 않고, 내부적으로 Hidden Class(또는 Map)라는 메타 구조를 통해 객체의 속성 구성과 순서를 추적합니다.

> 속성(Property): JavaScript 객체에서 속성은 단순히 키-값 쌍처럼 보이지만, V8 내부에서는 속성이 추가될 때마다 객체의 메모리 레이아웃이 바뀌는 계기가 됩니다. 이때 속성의 이름뿐 아니라 추가된 순서도 객체 구조를 결정짓는 중요한 요소가 됩니다.
> 
> 
> 전이(Transition): 속성이 새로 추가되면, V8은 현재 객체가 가진 Hidden Class를 기존 구조에서 새로운 구조로 연결(전이)시킵니다. 이때 각 전이는 하나의 속성 추가에 해당하며, Hidden Class 간 연결로 이루어진 트리 구조를 형성하게 됩니다.
> 

**🧩 Hidden Class의 동작 방식**

```jsx
let obj = {};        // [1] 초기 Hidden Class
obj.x = 10;          // [2] → 'x' 속성이 있는 Hidden Class로 전이
obj.y = 20;          // [3] → 'x, y' 순서의 Hidden Class로 전이
```

- [1] 객체가 생성되면, V8은 해당 객체에 초기 Hidden Class를 할당합니다. 이는 속성이 아무것도 없는 상태를 나타냅니다.
- 이후 속성이 하나씩 추가될 때마다, V8은 기존 Hidden Class에서 새로운 Hidden Class로 전이(transition)를 수행합니다.
    - [2] `obj.x = 10`과 같이 속성이 하나 추가되면 `x`만 포함된 새로운 Hidden Class로 전이
    - [3] `obj.y = 20`이 추가되면 `x, y` 순서를 반영한 또 다른 Hidden Class로 다시 전이
        
        ```c
        [Initial Hidden Class]
                |
                | (add property 'x')
                v
           [Hidden Class: x]
                |
                | (add property 'y')
                v
           [Hidden Class: x, y]
        ```
        

이처럼 속성 추가 순서에 따라 Hidden Class가 단계적으로 연결되며, 동일한 구조를 가진 객체들은 같은 Hidden Class를 공유하게 됩니다. 객체의 속성 추가 순서에 따라 Hidden Class는 분기 형태의 트리 구조를 형성하며 확장되고, 이는 내부적으로 각 객체의 정적 구조 정보를 표현하게 됩니다.

이 구조 덕분에 V8은 `"obj.x"`와 같은 속성 접근 시 문자열 키를 검색하는 방식이 아니라, 해당 Hidden Class가 가진 오프셋 정보를 기반으로 `obj[+8]`처럼 고정된 위치에 직접 접근할 수 있습니다.

이 방식은 속성 조회 속도를 크게 향상시키며, 객체 구조가 변하지 않을 것이라는 가정 하에 V8이 고성능의 최적화된 네이티브 코드를 생성할 수 있도록 해줍니다.

### 2.3 V8의 최적화 모델 소개(2) : **ElementsKind**

**ElementsKind**는 V8이 JavaScript 배열의 요소 타입과 밀도를 추적하기 위해 사용하는 내부 메타정보입니다. JavaScript의 배열은 숫자, 문자열, 객체가 자유롭게 섞이거나 중간에 빈 슬롯이 있는 등 매우 동적인 구조를 가집니다. 하지만 V8은 성능 최적화를 위해 이러한 배열을 가능한 한 단순하고 정형화된 방식으로 내부 표현하려고 합니다. 이때 기준이 되는 것이 바로 ElementsKind입니다.

🧩 **ElementsKind의 동작 방식**

- 배열이 생성되면, V8은 해당 배열의 내용에 따라 가장 단순하고 밀집된 형태의 ElementsKind를 부여합니다.
    
    예를 들어:
    
    ```jsx
    [1, 2, 3]           → PackedSmiElements (32비트 정수 배열)
    [1.1, 2.2]          → PackedDoubleElements (64비트 실수 배열)
    [1, {}, "text"]     → PackedElements (일반 객체 배열)
    ```
    
- 이후 배열에 다른 타입의 요소가 삽입되거나, 중간 요소가 삭제되어 밀도가 낮아지면, V8은 더 일반적인 ElementsKind로 전이시킵니다.
    
    ```
    let arr = [1.1, 2.2];   // 초기: PackedDoubleElements
    arr[2] = {};            // → PackedElements로 전이
    ```
    

이러한 접근 방식은 동적으로 변할 수 있는 JavaScript 배열의 특성과는 다소 상반되지만, 실행 중 변화가 발생하기 전까지는 매우 높은 성능을 제공합니다. 결국 ElementsKind는 배열 요소에 대한 추상적인 타입 정보를 기반으로 JIT 최적화의 핵심 기준을 제공하며, 동적인 JavaScript 코드를 정적인 성능 모델로 수렴시키는 데 중요한 역할을 합니다.

### 2.4 최적화 흐름과 Native Code 생성

V8은 JavaScript의 동적 특성을 최대한 정적으로 가정하여 고성능의 Native Code를 생성합니다. 앞서 살펴본 Hidden Class와 ElementsKind는 객체와 배열의 구조를 예측 가능하게 만들어 주고, 이를 바탕으로 V8은 실행 중 최적화된 코드를 점진적으로 생성해 나갑니다. 지난 시간(Part 1.)에 V8 실행 파이프라인 구조에서는 전체적인 개념과 역사를 보았으니 이제는 Native Code 실행과 최적화 흐름 관점에서 이 컴포넌트들을 살펴보겠습니다.

---

**⚙️ Native Code 실행과 최적화 흐름**

![image.png](image%205.png)

V8의 실행 파이프라인은 다음과 같은 단계를 통해 JavaScript 코드를 점차 고속의 네이티브 머신 코드로 변환합니다.

1. **Ignition** 인터프리터가 JavaScript 코드를 바이트코드로 변환하고 실행을 시작합니다. 이 단계에서는 실행 횟수, 타입 정보 등 런타임 **프로파일링 정보가 수집**됩니다.
2. 코드가 "자주 실행되는(hot)" 경로로 판단되면, **Sparkplug** 또는 **Maglev** JIT 컴파일러가 빠른 컴파일을 수행하여 **Native Code를 생성**합니다. 이 코드는 초기 최적화 수준이지만, 속도와 효율이 우선시됩니다.
3. 이후 코드가 충분히 분석되었다고 판단되면, 고급 JIT 컴파일러인 **TurboFan**이 보다 정교한 최적화를 수행합니다. 이때, Hidden Class나 ElementsKind와 같은 내부 구조 정보를 적극 활용하여, **객체 구조가 고정되어 있다고 가정한 고성능 Native Code를 생성**합니다.
    
    > e.g., obj.x 접근 시, V8은 Hidden Class를 통해 "x"의 오프셋이 +8임을 알고 직접 해당 메모리 위치를 참조하는 코드를 생성할 수 있습니다. 마찬가지로, ElementsKind를 통해 배열의 메모리 접근 방식도 정적 최적화됩니다.
    > 

---

TurboFan은 이 과정에서 Inline Caching(IC), Escape Analysis, Loop Unrolling, Function Inlining 등 다양한 고급 최적화 기법을 활용합니다. 이렇게 생성된 Native Code는 매우 높은 실행 성능을 제공하지만, 동시에 객체의 내부 구조와 타입에 대한 **강한 정적 가정**에 의존하게 됩니다.

V8은 이러한 가정을 바탕으로 최적화된 코드를 함수의 entry point에 등록합니다. 그 결과, 이후 함수 호출 시에는 인터프리터 단계를 건너뛰고 곧바로 Native Code가 실행되며, 실행 속도가 획기적으로 향상됩니다.

하지만 이 최적화는 다음과 같은 전제를 전제로 합니다.

> “객체의 구조나 배열의 타입은 런타임 동안 바뀌지 않는다.”
> 

문제는 JavaScript의 특성상 객체나 배열의 구조는 언제든지 동적으로 변경될 수 있다는 점입니다. 이러한 변화가 감지되면, V8은 기존의 Native Code를 폐기(deoptimize)하고 다시 인터프리터로 되돌아가거나, 새롭게 최적화된 코드를 생성하게 됩니다.

그러나 만약 이러한 변화가 제대로 감지되지 않거나, 공격자에 의해 고의적으로 회피된다면, V8은 여전히 이전의 잘못된 타입 정보를 기반으로 최적화 경로를 실행하게 됩니다.

그 결과, Hidden Class나 ElementsKind가 이미 바뀌었음에도 Inline Cache가 이전 정보를 그대로 사용하는 상황이 발생할 수 있으며, 이는 잘못된 필드 오프셋 계산이나 엉뚱한 메모리 참조로 이어집니다. 결국 이러한 맥락이 **Type Confusion 취약점의 직접적인 원인**이 됩니다.

이처럼 V8의 최적화 파이프라인은 JavaScript의 동적인 특성을 정적인 실행 모델로 수렴시키기 위한 핵심 메커니즘입니다. Hidden Class와 ElementsKind는 이 정적 모델의 기반을 이루며, 이를 바탕으로 생성된 Native Code는 매우 높은 성능을 제공합니다.

![image.png](image%206.png)

하지만 동시에, 이 정적 가정이 무너지는 순간 취약점이 발생할 수 있기 때문에, 성능과 보안 사이의 트레이드오프를 관리하는 것이 V8 설계에서 매우 중요한 과제가 됩니다.

### 2.5 Type Confusion의 발생 구조

앞서 설명한 V8의 최적화 메커니즘은 객체 구조와 배열의 타입이 변하지 않을 것이라는 강한 전제에 기반합니다. 하지만 JavaScript는 본질적으로 매우 동적인 언어이며, 이 전제가 실제 실행 흐름에서 깨지는 순간 Type Confusion이 발생할 수 있습니다.

정적 모델의 기반을 이루는 Hidden Class와 ElementsKind에서 각각 어떤 식으로 Type Confusion이 유발되는지 구체적으로 살펴봅시다!

---

**1) Hidden Class 기반 Confusion**

Hidden Class는 객체의 속성 구조를 추적하여 빠른 프로퍼티 접근과 JIT 최적화를 가능하게 해주는 내부 메커니즘입니다. 그러나 객체에 동적으로 속성이 추가되거나, 속성 타입이 바뀌는 경우, Hidden Class는 다른 클래스로 전이되며, 이로 인해 기존의 Native Code는 더 이상 유효하지 않게 됩니다.

```jsx
function Foo() {
  this.x = 1;  // HiddenClass A
  this.y = 2;  // HiddenClass B
}
let a = new Foo();
a.z = 3;       // HiddenClass C (transition)
```

이 경우 `a`는 최종적으로 Hidden Class C를 갖습니다. 그런데 TurboFan이 Hidden Class B까지의 구조를 기반으로 Native Code를 생성했고, 이후에도 해당 코드가 그대로 `a`에 적용된다면, `a.z`를 `a.y`로 잘못 인식하거나 잘못된 오프셋을 참조할 수 있습니다.

**Hidden Class 전이와 Deoptimization**

속성의 **타입 변화** 또한 Hidden Class 전이를 유발합니다. 예를 들어, 다음과 같이 객체 속성의 타입이 숫자에서 객체로 바뀌는 경우를 생각해 봅시다.

```jsx
let obj = { x: 1 };         // x is number → HiddenClass A
obj.x = { nested: true };   // x is now object → HiddenClass B
```

이때 V8은 타입 변경을 감지하고 기존 Native Code를 deoptimize 한 뒤, 다시 바이트코드로 돌아가거나 새로운 코드를 생성합니다. 그러나 구조 변경이 적절히 감지되지 않거나, 공격자가 이를 우회하도록 입력을 설계한 경우, V8은 여전히 기존 Hidden Class 기반의 Native Code를 실행하면서 잘못된 메모리 접근이 발생할 수 있습니다.

**2) ElementsKind 기반 Confusion**

배열의 경우, V8은 요소들의 타입과 밀도에 따라 다양한 **ElementsKind**를 정의합니다. 

예를 들면,

- `PACKED_SMI_ELEMENTS`: 정수만 포함된 밀집 배열
- `PACKED_DOUBLE_ELEMENTS`: 부동소수점만 포함된 배열
- `PACKED_ELEMENTS`: 여러 타입이 섞인 배열
- 위의 각 종류에 대응하는 HOLEY_* (희소 배열) 변형

TurboFan은 배열의 ElementsKind가 변하지 않는다고 가정하고 최적화 코드를 생성합니다. 그러나 배열에 다른 타입의 요소가 삽입되는 순간, ElementsKind는 전환되고, 기존 Native Code는 더 이상 유효하지 않습니다.

```jsx
function confuse(x) {
  let arr = [1.1];     // PackedDoubleElements
  if (x) arr[0] = {};  // object 삽입 → ElementsKind 전환
  return arr[0];
}
```

`x`가 `false`일 경우, `arr`은 double 배열로 유지되며 TurboFan은 이에 맞는 코드를 생성합니다. 그러나 `x`가 `true`이면 `arr[0]`에 객체가 삽입되면서 배열의 내부 표현이 바뀝니다. 이때 여전히 기존 Native Code가 실행되면, 객체를 double로 해석하는 **잘못된 타입 캐스팅**이 발생하게 됩니다.

## 3. From Type Confusion to Memory Corruption

### 3.1 실질적인 Memory Corruption으로의 연결

단순한 Type Confusion은 왜 Memory Corruption으로 확장될까요?

그 핵심은, V8의 JIT 최적화된 네이티브 코드가 객체 구조나 배열의 타입을 신뢰한 채, 메모리에 직접 접근한다는 데 있습니다. V8은 HiddenClass(Map)나 ElementsKind 정보를 바탕으로 객체 또는 배열의 정확한 필드 오프셋을 계산하여 빠르게 접근합니다. 문제는 이 가정이 깨지는 경우입니다. 객체의 실제 구조가 변경되었는데도 컴파일된 코드는 여전히 이전 구조를 기준으로 접근하게 되면, 완전히 엉뚱한 메모리 주소에 접근하거나 데이터를 잘못 해석하게 됩니다. 이는 곧 예기치 않은 위치의 메모리를 조작하게 되는 메모리 커럽션으로 이어질 수 있습니다.

예를 들면 다음과 같은 상황을 생각해볼 수 있습니다.

- **부동소수점 배열을 객체 배열로 오인한 경우**
    
    `13.37`과 같은 부동소수점 값이 저장된 배열을, JIT가 객체 배열로 잘못 판단하고 접근할 경우, 이 숫자 값이 마치 힙 포인터인 것처럼 해석되어 전혀 의도하지 않은 메모리 위치에 접근하게 됩니다. 이는 OOB Read/Write로 이어지며, 공격자가 이를 통해 인접한 객체를 조작하거나 데이터 유출을 유도할 수 있습니다.
    
- **TypedArray 또는 DataView를 이용한 포인터 조작**
    
    자바스크립트에서 TypedArray나 ArrayBuffer를 사용하는 경우, 해당 버퍼가 실제 메모리 영역을 직접 가리키게 됩니다. 만약 공격자가 객체 구조를 제어할 수 있는 상황이라면, 이 backing store 포인터를 조작하여 임의의 메모리 주소를 버퍼의 시작점으로 설정할 수 있습니다. 이 경우, ArrayBuffer를 통한 읽기/쓰기가 곧 전체 주소 공간에 대한 메모리 접근으로 확장됩니다.
    

이는 객체 내부의 필드 오프셋을 잘못된 타입 해석을 통해 조작함으로써, 해당 객체 인근 메모리나 동일 구조체 내의 특정 필드에 제한적으로 접근할 수 있는 능력을 의미합니다.

여기서 한 가지 더 중요한 요소는 바로 가비지 컬렉터(Garbage Collector, GC)입니다. V8은 JS 객체들의 수명을 자동으로 관리하는 GC 시스템을 사용하는데, 이 시스템은 필요에 따라 객체를 힙 내 다른 위치로 이동시키거나 정리합니다. 이 과정에서 객체의 메모리 주소가 바뀌더라도, 기존의 최적화된 코드는 이를 인식하지 못한 채, 이전 구조를 기준으로 접근을 시도할 수 있습니다. 즉, GC와 JIT 사이의 시차 또는 정보 불일치는 또 하나의 Type Confusion 트리거로 작용할 수 있는 것입니다.

결과적으로, Type Confusion은 단순한 논리 오류나 개발자의 실수가 아닌, V8의 JIT 최적화, 객체 모델, GC 시스템이 상호작용하는 복잡한 경계 지점에서 발생할 수 있는 취약점임을 알 수 있습니다. 

Type Confusion이 왜 발생하고 Memory Corruption이 충분히 발생할 수 있음을 알았으니 이 취약점을 이용해서 공격을 어떻게 시작하는지 알아보겠습니다.

### 3.2 Relative Read/Write Primitive

Type Confusion 에서의 기본적인 exploit 흐름은 Type Confusion 으로 인한 OOB R/W 로부터 시작합니다. OOB R/W를 잘 다듬어서 Relative R/W primitive를 얻고 이를 또 잘 이용해서 AAR/W(Arbitary Address Read/Write)를 얻은 후 Code Execution 을 하는 흐름이 Type Confusion 으로 RCE를 하는 공격 흐름입니다 ㅎ.ㅎ 정말 재밌겠죠!! 생각보다 이번 파트가 길어져서 이번 파트에서는 다 다루지 못할거 같지만 이대로 끝내기는 아쉬우니까, 맛보기로 Relative R/W 부분만 간단히 살펴보겠습니다 :)

---

Type Confusion은 단순한 잘못된 타입 해석 이상의 의미를 지닙니다. 공격자의 입장에서 보면, 이는 원래 접근할 수 없던 메모리 영역을 조작할 수 있는 출발점이 됩니다. 그중 대표적인 것이 바로 Relative Read/Write입니다. 이는 메모리의 정해진 오프셋을 기준으로 제한된 범위 내에서 접근이 가능한 primitive입니다.

예를 들어 구조가 유사하지만 마지막 필드의 의미가 다른 두 클래스를 생각해봅시다.

![image.png](image%207.png)

이 두 클래스는 필드 개수와 순서는 유사하지만, 마지막 필드의 의미와 타입은 다릅니다.

`Worker process` 에서 `Class A` 의 사는 곳을 가리키는 동작을 한다고 생각해봅시다.

![image.png](image%208.png)

이제 Worker process 내부에서 어떤 이유로 `Class A` 인스턴스를 `Class B` 타입으로 오인하게 된 상황을 가정해 봅시다. 이 경우, 정상적인 상황에서는 "사는 곳" 필드에 접근해야 하지만, `Class B` 기준으로 해석하면 이 메모리 위치는 "일하는 곳"에 해당하는 필드로 인식됩니다.

![image.png](image%209.png)

그 결과, 실제로는 Worker process는 `Class A` 인스턴스의 필드에 접근하고 있지만, 런타임에서는 이를 `Class B`의 필드로 간주하면서 의도하지 않은 메모리 영역을 읽거나 쓰게 됩니다. 

> 만약에 집으로 배송시켰는데, 회사로 택배가 가면 곤란하겠죠?;;
> 
> 다음은 다른 곤란한 예시입니다.
> 
> ![image.png](image.png)
> 

이처럼 런타임의 잘못된 타입 해석은, **정확히 알 수는 없지만 일정한 오프셋을 기준으로 메모리 접근을 가능하게 하는 Relative R/W Primitive**를 만들어냅니다. 이를 통해 공격자는 다음과 같은 조건을 만족시킬 수 있습니다.

- 자신이 제어 가능한 객체(`Class A`)를 기반으로, 고정된 오프셋(예: +0x10, +0x18 등)을 기준으로,
- 메모리 내 특정 필드 또는 인접 구조체(`Class B`)의 내용을 읽거나 쓸 수 있음

---

이러한 Relative Primitive는 공격 초기 단계에서 매우 중요한 의미를 가집니다:

- 특정 객체의 내부 구조를 **leak**하거나,
- 포인터나 길이 필드 등의 **핵심 정보**를 조작하여,
- 이후 **Absolute R/W Primitive**로의 확장을 위한 발판을 마련하게 됩니다.

다음 파트에서는 이러한 Relative R/W Primitive가 어떻게 절대 주소 기반의 메모리 조작으로 확장되며, 궁극적으로 원격 코드 실행에 이르게 되는지를 실제 익스플로잇 흐름을 통해 살펴보겠습니다.

다음 파트부터는 찐한 Pwnable 냄새가 날거 같은데 기대해주세요;;; 열심히 준비해보겠습니다…!

(다음은 그냥 사심으로 넣은 디지털 풍화를 맞은 장난전화 사진입니다. 이런 느낌을 쓰고 싶었는데 리암니슨과 중국집은 못참긴하죠 🥵)

![image.png](image%2010.png)

## Reference

v **Conference Video**

- https://www.youtube.com/watch?v=RL2po1swXO4

v **Background**

- [https://v8.dev/docs/hidden-classes](https://v8.dev/docs/hidden-classes)
- [https://v8.dev/blog/elements-kinds](https://v8.dev/blog/elements-kinds)
- [https://richardartoul.github.io/jekyll/update/2015/04/26/hidden-classes.html](https://richardartoul.github.io/jekyll/update/2015/04/26/hidden-classes.html)