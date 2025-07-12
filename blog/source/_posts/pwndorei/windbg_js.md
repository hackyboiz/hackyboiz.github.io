---
title: "[Research] windbg를 위한 javascript 한 스푼"
author: pwndorei
tags: [pwndorei, windbg, TTD, Debugging]
categories: [Research]
date: 2025-06-01 23:00:00
cc: true
index_img: /2025/06/01/pwndorei/windbg_js/image%206.png
---

# Introduction

안녕하세요, pwndorei입니다… 작년 9월 정도부터 회사에 다니기 시작하면서 그동안 연구글을 쭉 쉬었는데 이젠 더이상 피할 수 없게 되어 돌아왔습니다 🤦🏻‍♂️ 근데 회사일 말고는 연구하는게 없어서 글로 쓸만한 주제가 딱히 떠오르질 않더라고요. 그래서 그냥 windbg로 분석할때 쓰기 좋은 js 스크립트를 선보이기로 했습니다. 그럼 가보시죠!

# Windbg + Script ⇒ ?

디버깅을 하는데 있어서 스크립트를 사용한다? 목적은 단 하나, 게으르게 디버깅하기 위함입니다.

![image.png](windbg_js/image.png)

> ~~절대 미리 고생해서 나중에 편하자는게 아닙니다~~

엄청 복잡하게 생겨먹은 자료구조를 하나하나 정성스럽게 dq, dd 하면서 어떤 데이터가 있는지 확인하는 것도 물론 가능합니다. TTD를 쓰면 중요한 call을 실수로 step over해버려도 프로세스를 새로 실행할 필요가 없으니 무지성으로 F10만 눌러가면서 분석해도 됩니다. 하지만! 데이터가 커지면 커질수록 손수 작업하는 것보다 스크립트를 짜는 편이 시간이 더 적게 걸리고 TTD도 너무 의존하다가는 바보가 되어버립니다. 그래서 오늘은 간단한 예시로 스크립트를 어떻게 쓰면 좋을지 알아보고 덤으로 TTD를 극한까지 활용하는 방법도 알려드리겠습니다.

# Windbg Script Basic

일단 windbg에서는 아래와 같이 Scripting 메뉴에서 새로운 스크립트를 생성하거나 기존 스크립트를 불러올 수 있습니다. JavaScript말고 NatVis라는 것도 있긴 한데 우리가 쓸건 JavaScript이니까 그냥 무시합시다.

![image.png](windbg_js/image%201.png)

그 다음으로 New Script에서 새로 만들 스크립트의 형식을 Extension, Imperative 중에 고를 수 있는데 이건 그냥 기본 템플릿 차이입니다.

Imperative의 경우에는 아래와 같은 기본 코드가 생성됩니다.

```jsx
"use strict";

function initializeScript()
{
    return [new host.apiVersionSupport(1, 9)];
}

function invokeScript()
{
    //
    // Insert your script content here.  This method will be called whenever the script is
    // invoked from a client.
    //
    // See the following for more details:
    //
    //     https://aka.ms/JsDbgExt
    //
}
```

Actions에서 직접 Execute를 누르거나 `.scriptrun` 등의 커맨드로 스크립트를 실행할 수 있습니다.

![image.png](windbg_js/image%202.png)

Extension의 경우에는 이름처럼 확장 기능을 추가하기 위해 사용되고 아래와 같은 기본 코드가 생성됩니다.

```jsx
"use strict";

function initializeScript()
{
    //
    // Return an array of registration objects to modify the object model of the debugger
    // See the following for more details:
    //
    //     https://aka.ms/JsDbgExt
    //
    return [new host.apiVersionSupport(1, 9)];
}
```

스크립트를 통해 추가된 extension은 windbg 커맨드에서 `!extname`과 같은 형식으로 호출해서 사용하는 것이 가능합니다.

어떤 기능을 어떻게 구현할 수 있는지는 뒤에서 예제와 함께 알아보도록 하고 그 전에 스크립트 관련 커맨드 몇가지만 마저 알아보자면 아래와 같습니다

- `.scriptlist`
    - 현재 로드된 스크립트를 전부 보여줍니다.
    
    ```
    0:038> .scriptlist
    Command Loaded Scripts:
    		...
        JavaScript script from 'D:\windbg_Ext\utils.js'
        JavaScript script from 'D:\windbg_Ext\codecov.js'
        JavaScript script from 'C:\Program Files\WindowsApps\Microsoft.WinDbg_1.2504.15001.0_x64__8wekyb3d8bbwe\amd64\winext\ApiExtension\CodeFlow.js'
    Other Clients' Scripts:
        Script named 'untitled0'
        Script named 'untitled1'
    ```
    
- `.scriptload` & `.scriptunload`
    - 스크립트를 load, unload하는 커맨드입니다.
    - 로드되는 스크립트는 전체 경로를 지정하거나 이름을 지정하면 PATH 환경변수로 설정된 경로들에서 탐색합니다.
    - `.scriptload`로는 스크립트의 `initializeScript` 함수만 호출됩니다.
- `.scriptrun`
    - `.scriptload`와 달리 `invokeScript` 함수까지도 호출됩니다.

스크립트를 실행하는 커맨드가 `.scriptload`, `.scriptrun` 두개가 있는데 실행할 스크립트가 imperative면 `invokeScript`까지 실행해야 하니까 run, extension이면 load를 쓰면 되겠죠?

![image.png](windbg_js/image%203.png)

## Hello World

새로운 언어…를 배우는건 아니지만 지금껏 브라우저에서 사용해온 JavaScript처럼 `console.log` 같은건 사용할 수 없으니 일단 “Hello World”부터 콘솔에 출력해봅시다.

```jsx
"use strict";

function initializeScript()
{
    return [new host.apiVersionSupport(1, 9)];
}

function invokeScript()
{
    host.diagnostics.debugLog("Hello World!\n")
}
```

위처럼 `console.log` 대신에 `host.diagnostics.debugLog` 를 사용하면 되는데요? 실행해보면 평범하게 잘 출력됩니다.

![image.png](windbg_js/image%204.png)

로그를 출력할때마다 일일이 저런 코드를 칠 자신은 없으니 아래처럼 alias를 만들어주고 대신 이걸 사용합시다 🤣

```jsx
"use strict";

const log = x => host.diagnostics.debugLog(x)

function initializeScript()
{
    return [new host.apiVersionSupport(1, 9)];
}

function invokeScript()
{
    log("Hello World!\n")
}
```

## Read & Write Memory

고작 콘솔에 “Hello World”나 출력하려고 스크립트를 쓰진 않겠죠? `host.memory`에는 아래와 같은 함수들이 존재하고 이를 사용하면 메모리에서 데이터를 읽고 쓰는 것이 가능합니다.

- readMemoryValues
- readWideString
- readString
- writeMemoryValues

오늘 해볼 건 이정도만 알아도 충분합니다!

# Practice

그럼 이제 예제 프로그램을 디버깅하면서 스크립트가 어떤 상황에 사용될 수 있을지 사용해봅시다.

## 1. Dump Data Structure

첫 번째로 해볼 것은 아까 예시를 들었던 자료구조 덤프하기입니다. 예제 프로그램은 아래와 같이 ChatGPT가 만들어줬습니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct Node {
    char* word;
    struct Node* left;
    struct Node* right;
} Node;

Node* createNode(const char* word) {
    Node* newNode = (Node*)malloc(sizeof(Node));
    if (!newNode) return NULL;

    newNode->word = _strdup(word);
    newNode->left = NULL;
    newNode->right = NULL;
    return newNode;
}

Node* insert(Node* root, const char* word) {
    if (root == NULL) {
        return createNode(word);
    }

    if (strcmp(word, root->word) < 0) {
        root->left = insert(root->left, word);
    } else if (strcmp(word, root->word) > 0) {
        root->right = insert(root->right, word);
    }
    return root;
}

void inorderTraversal(Node* root) {
    if (root != NULL) {
        inorderTraversal(root->left);
        printf("%s\n", root->word);
        inorderTraversal(root->right);
    }
}

void freeTree(Node* root) {
    if (root != NULL) {
        freeTree(root->left);
        freeTree(root->right);
        free(root->word);
        free(root);
    }
}

int main() {
    const char* words[10] = {
        "banana", "apple", "grape", "orange", "kiwi",
        "peach", "lemon", "mango", "cherry", "melon"
    };

    Node* root = NULL;

    for (int i = 0; i < 10; i++) {
        root = insert(root, words[i]);
    }

    printf("Address of root node: %p\n", (void*)root);

    printf("Press Enter to exit...");
    getchar();

    freeTree(root);
    return 0;
}

```

이진 트리에 문자열을 순서로 저장하는 간단한 코드인데요? 루트 노드의 주소를 출력하고 콘솔 입력을 기다립니다. 디버거를 attach하고 트리를 순회하는 스크립트를 짜면 데이터를 출력할 수 있겠네요.

![image.png](windbg_js/image%205.png)

일단 `Node`가 3개의 포인터 필드를 가지는 총 길이 0x18 바이트의 구조체니까 아래와 같이 getNode 함수를 정의했습니다.

```jsx
function getNode(addr){
    var node = host.memory.readMemoryValues(addr, 3, 8)
    return {
        "word": node[0],
        "left": node[1],
        "right": node[2]
    }
}
```

메모리에서 데이터를 읽어오는 동작은 `host.memory.readMemoryValues`를 통해 이루어지는데요? 보면 느낌적으로 아시겠지만 노드가 위치한 주소인 `addr`에서 8바이트씩(세 번째 파라미터; `elementSize`) 총 3개(두 번째 파라미터; `numElements`)의 element를 읽습니다. 반환되는 것은 읽은 데이터가 정수로 저장된 배열이고 이를 사용해서 노드 오브젝트를 만들었습니다. 

이제 메모리에 저장되어 있는 `Node` 구조체를 스크립트에서 사용할 수 있게 되었으니 다음으로는 트리를 순회하는 함수를 구현해야 합니다. 아래는 간단히 `getNode`로 반환된 노드를 파라미터로 중위 순회를 하면서 word필드에 저장된 주소에서 문자열을 읽어 출력하는 `traverse`함수입니다.

```jsx
function traverse(node){
    if(node["left"] != 0) traverse(getNode(node["left"]))
    log(host.memory.readString(node["word"]) + "\n")
    if(node["right"] != 0) traverse(getNode(node["right"]))
}
```

문자열을 읽을 때는 위처럼 `host.memory.readString`을 사용하면 되는데요? 문자열이 WideString이라면 `readWideString`을 대신 사용하면 됩니다. 전체 코드는 아래와 같습니다.

```jsx
"use strict";

const log = x => host.diagnostics.debugLog(x)

function initializeScript()
{
    return [new host.apiVersionSupport(1, 9)];
}

function getNode(addr){
    var node = host.memory.readMemoryValues(addr, 3, 8)
    return {
        "word": node[0],
        "left": node[1],
        "right": node[2]
    }
}

function traverse(node){
    if(node["left"] != 0) traverse(getNode(node["left"]))
    log(host.memory.readString(node["word"]) + "\n")
    if(node["right"] != 0) traverse(getNode(node["right"]))
}

function invokeScript()
{
    traverse(getNode(0x25DD2C211C0))
}
```

만약에 단순 출력을 넘어 특정 데이터가 위치한 주소를 찾고 싶다면? 그냥 데이터 비교 등 조건만 추가하고 출력해주면 됩니다.

```jsx
"use strict";

const log = x => host.diagnostics.debugLog(x)
const dat = "melon"

function initializeScript()
{
    return [new host.apiVersionSupport(1, 9)];
}

function getNode(addr){
    var node = host.memory.readMemoryValues(addr, 3, 8)
    return {
        "word": node[0],
        "left": node[1],
        "right": node[2]
    }
}

function traverse(addr){
    var node = getNode(addr)
    if(node["left"] != 0) traverse(node["left"])
    if(dat.localeCompare(host.memory.readString(node["word"])) == 0){
        log("Found at " + addr.toString(16) + "\n")
    }
    if(node["right"] != 0) traverse(node["right"])
}

function invokeScript()
{
    traverse(0x25DD2C211C0)
}

```

위 스크립트에서는 “melon”이란 문자열이 저장되어 있는지 검사하고 맞으면 주소를 출력하는 동작을 하고 있고 그 결과 아래와 같이 “melon”이란 문자열이 저장된 주소를 찾을 수 있었습니다.

```
Found at 25dd2c21280
0:004> da poi(25dd2c21280)
0000025d`d2c19360  "melon"
```

## 2. TTD

imperative 스크립트를 사용해봤으니 이번에는 extension의 차례입니다. extension 스크립트의 예제로는 제가 TTD를 사용할 때 무조건 쓰는 간단한 스크립트를 보시죠.

```jsx
'use strict';

const log = host.diagnostics.debugLog;
const logln = p => host.diagnostics.debugLog(p + '\n');
const hex = p => p.toString(16);

function __calls(...args){
	return host.currentSession.TTD.Calls(...args)
}

function __memory(...args){
	return host.currentSession.TTD.Memory(...args)
}

function __memoryforpositionrange(...args)
{
	return host.currentSession.TTD.MemoryForPositionRange(...args);
}

function ttdParsePosition(pos){
	var sequence = 0
	var steps = 0
	if(typeof(pos) == "string"){
		sequence = parseInt(pos.split(":")[0], 16)
		steps = parseInt(pos.split(":")[1], 16)
	}
	else{
		sequence = pos.Sequence
		steps = pos.Steps
	}

	return {
		Sequence: sequence,
		Steps: steps
	}
}

function __ttd_position_compare(pos1, pos2){
	/*
	alias: poscmp
	parameters:
		pos1 & pos2: Position object or string(Sequence:Steps in hexadicimal)
	usage:
		> !poscmp(pos1, pos2)
		> dx @$poscmp(pos1, pos2)
	example:
		> dx @$calls("function").Where(x => @$poscmp("17C83A:1777", x.TimeStart) < 0).Where(x => @$poscmp("17C861:1B3A", x.TimeEnd) > 0)
		=> get all `function` calls between 17C83A:1777 and 17C861:1B3A
	*/

	pos1 = ttdParsePosition(pos1)
	pos2 = ttdParsePosition(pos2)

	if(pos1.Sequence != pos2.Sequence){
		return pos1.Sequence - pos2.Sequence
	}

	return pos1.Steps - pos2.Steps
	
}

function __d(...args){
	return host.memory.readMemoryValues(...args)
}

function initializeScript() {
    return [
        new host.apiVersionSupport(1, 2),
        new host.functionAlias(
            __calls,
            'calls'
        ),
		new host.functionAlias(
			__memory,
			'memory'
		),
		new host.functionAlias(
			__memoryforpositionrange,
			'memoryrange'
		),
		new host.functionAlias(
			__ttd_position_compare,
			'poscmp'
		),
		new host.functionAlias(
			__d,
			'd'
		)
    ];
}
```

`imperative` 스크립트와 크게 다른 부분이 있죠? 바로 `initializeScript` 함수에 `host.functionAlias`로 함수의 alias를 생성한다는 점입니다. extension 스크립트는 이런 식으로 javascript로 구현한 기능을 alias로 등록해서 커맨드에서 사용할 수 있습니다.

저는 일단 아래와 같이 TTD에서 자주 사용하는 `@$cursession.TTD.Calls` 와 `@$cursession.TTD.Memory`의 alias를 만들어서 손가락의 피로를 줄이는 편입니다 😎

```jsx
function __calls(...args){
	return host.currentSession.TTD.Calls(...args)
}

function __memory(...args){
	return host.currentSession.TTD.Memory(...args)
}
```

이러면 각각 `@$calls`, `@$memory` 로 줄여서 쓸 수 있습니다. 그래서 이걸 어디에 쓰냐고요? 

### TTD Calls & Memory

예전에 Fabu1ous님이 쓴 [시간을 여행하는 해커를 위한 안내서](https://hackyboiz.github.io/2021/03/14/fabu1ous/ttd-3/#Filtering-in-a-query) 시리즈에 살짝 Calls와 Memory의 용도가 설명되어 있는데요? 특정 함수 호출이나 메모리 접근을 검색해서 볼 수 있다는 정도만 짚고 넘어가서 TTD의 위대함이 덜 전달되었던 것 같습니다. 아무튼 이거도 예제를 쓰면서 어떻게 활용할 수 있는지 알아보시죠

일단 extension 스크립트를 아래와 같은 스크립트를 써서 로드부터 합시다

```
0:038> .scriptload utils.js
JavaScript script successfully loaded from 'D:\windbg_Ext\utils.js'
```

그럼 이제 Calls와 Memory를 사용할 때 아래처럼 접근하는 것이 가능해집니다.

```
https://hackyboiz.github.io/2021/03/14/fabu1ous/ttd-3/#Filtering-in-a-query
```

`!calls`를 쓰면 커맨드에서 리턴된 배열에 바로 접근할 수가 없어서 저는  `dx @$calls`를 더 자주 활용하는 편입니다.

그럼 이제 아래의 예제를 녹화한 TTD를 가지고 어떻게 활용할 수 있을지 살펴보죠!

```c
#include <iostream>
#include <vector>
#include <memory>
#include <random>
#include <cstdlib>
#include <ctime>

// Base class
class BaseProcessor {
public:
    virtual void process(int value) = 0;
    virtual ~BaseProcessor() = default;
};

// Derived class with actual behavior
class ConcreteProcessor : public BaseProcessor {
public:
    void process(int value) override {
        // Dummy operation to prevent optimization
        volatile int sink = value;
        (void)sink; // Prevent unused variable warning
    }
};

int main() {
    std::srand(static_cast<unsigned int>(std::time(nullptr)));
    std::mt19937 rng(std::random_device{}());
    std::uniform_int_distribution<int> distIndex(0, 9);
    std::uniform_int_distribution<int> distValue(0, 1000);

    // Create at least 10 instances
    std::vector<std::unique_ptr<BaseProcessor>> processors;
    for (int i = 0; i < 10; ++i) {
        processors.push_back(std::make_unique<ConcreteProcessor>());
    }

    // Call virtual function 100 times with random instance and value
    for (int i = 0; i < 100; ++i) {
        int idx = distIndex(rng);
        int value = distValue(rng);
        processors[idx]->process(value);
    }

    // Wait for user input
    std::cout << "All calls completed. Press Enter to exit...";
    std::cin.get();

    return 0;
}
```

> ~~이거도 ChatGPT가 짠 코드인건 안 비밀~~
> 

 Calls와 Memory의 활용법을 설명하기 위해 아래와 같은 동작을 하도록 만들어졌습니다.

- 클래스의 인스턴스를 10개 생성해서 사용
- 클래스의 가상함수(`process`)를 총 100번 호출
- 가상함수 호출에 사용할 인자에 랜덤한 값 사용
- 가상함수 호출에 사용할 인스턴스를 랜덤하게 선택

극단적인 예시이긴 하지만 분석을 하다 보면 의외로 이와 비슷한 상황에 처하게 되더라고요. 

예를 들어 제가 Calls와 Memory를 사용하는 주된 목적은 아래의 세 가지 정도가 있습니다.

1. 특정 인스턴스에 의한 함수 호출만
2. 반환값이나 인자로 사용된 값을 필터링
3. TTD Position 비교를 통해 특정 시점 이전/이후의 호출, 메모리 접근만 필터링

그럼 바로 Calls를 사용해봅시다. 일단 아래와 같이 `Count`를 통해 함수가 몇 번 호출된지 볼 수 있는데요?

```
0:000> dx @$calls("TTDCalls!ConcreteProcessor::process").Count()
@$calls("TTDCalls!ConcreteProcessor::process").Count() : 0x64
```

0x64(100)번의 호출 중에서 다섯 번째로 생성된 `ConcreteProcessor` 인스턴스에 의한 호출만 보려면 어떻게 해야할까요? 일단 다섯 번째로 생성된 인스턴스의 주소부터 확인해야겠죠? 원래같으면 클래스의 생성자의 호출을 리스팅하고 그 중 다섯 번째 호출의 반환값을 확인했겠지만 위 예제에서는 `std::make_unique` 를 통해 생성하고 있으니 이걸 이용해봅시다.

```
0:000> u TTDCalls!main+b0
TTDCalls!std::make_unique [C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.41.34120\include\memory @ 3597] [inlined in TTDCalls!main+0xb0 [D:\Src\WindbgExamplePrograms\TTDCalls\main.cpp @ 34]]:
00007ff7`84d91260 b908000000      mov     ecx,8
00007ff7`84d91265 e8c6080000      call    TTDCalls!operator new (00007ff7`84d91b30)
00007ff7`84d9126a 488930          mov     qword ptr [rax],rsi
00007ff7`84d9126d 4889442420      mov     qword ptr [rsp+20h],rax
00007ff7`84d91272 488b542438      mov     rdx,qword ptr [rsp+38h]
00007ff7`84d91277 483b542440      cmp     rdx,qword ptr [rsp+40h]
00007ff7`84d9127c 740e            je      TTDCalls!main+0xdc (00007ff7`84d9128c)
00007ff7`84d9127e 488bcf          mov     rcx,rdi
```

근데 `std::make_unique`가 인라인 함수로 처리되어 있어서 Calls를 사용할 수가 없는데요? 그렇다고 `TTDCalls!operator new`를 사용하면 관련 없는 다른 호출도 보여질 가능성이 있습니다. 이런 경우에는 `call TTDCalls!operator new` 인스트럭션의 주소에 실행(”e”)으로 접근하는 시점을 아래와 같이 Memory를 사용해서 리스팅할 수 있습니다.

```
0:000> dx @$memory(0x7ff784d91265, 0x7ff784d91265+1, "e")
@$memory(0x7ff784d91265, 0x7ff784d91265+1, "e")                
    [0x0]           
    [0x1]           
    [0x2]           
    [0x3]           
    [0x4]           
    [0x5]           
    [0x6]           
    [0x7]           
    [0x8]           
    [0x9]      
```

그런데 Memory를 사용하게 되면 Calls와 달리 해당 시점으로 이동해야 사용된 파라미터나 반환값을 확인할 수 있는데요? 지금같은 상황에서는 `00007ff784d91265`라는 주소에서 호출된 `operator new`는 `ReturnAddress`가 `00007ff784d9126a`일테니 이를 활용하는 것도 방법입니다.

```
0:000> dx @$calls("TTDCalls!operator new").Where(x => x.ReturnAddress == 0x7ff784d9126a)
@$calls("TTDCalls!operator new").Where(x => x.ReturnAddress == 0x7ff784d9126a)                
    [0x0]           
    [0x2]           
    [0x4]           
    [0x6]           
    [0x8]           
    [0xa]           
    [0xb]           
    [0xd]           
    [0xe]           
    [0xf]   
```

지금은 둘중 어떤 방법을 사용하더라도 소요되는 시간은 비슷할테니 Memory의 결과를 사용하도록 하죠. 다섯 번째 호출을 확인해보면 아래와 같습니다

```
0:000> dx @$memory(0x7ff784d91265, 0x7ff784d91265+1, "e")[4]
@$memory(0x7ff784d91265, 0x7ff784d91265+1, "e")[4]                
    EventType        : 0x1
    ThreadId         : 0x567c
    UniqueThreadId   : 0x2
    TimeStart        : 4D8:24A2 [Time Travel]
    TimeEnd          : 4D8:24A2 [Time Travel]
    AccessType       : Execute
    IP               : 0x7ff784d91265
    Address          : 0x7ff784d91265
    Size             : 0x5
    Value            : 0x8c6e8
    SystemTimeStart  : 2025년 5월 31일 토요일 12:24:28.628
    SystemTimeEnd    : 2025년 5월 31일 토요일 12:24:28.628
0:000> dx @$memory(0x7ff784d91265, 0x7ff784d91265+1, "e")[4].TimeStart.SeekTo()
(43f8.567c): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 4D8:24A2
```

`TimeStart`, `TimeEnd` 필드를 보면 이 메모리 접근 이벤트가 언제 시작해서 언제 끝나는지 확인할 수 있습니다. position 옆에 위치한 `[Time Travel]`을 눌러서 해당 시점으로 이동할 수도 있고 `TimeStart`나 `TimeEnd`의 `SeekTo` 메소드를 바로 호출해서 이동하는 것도 가능합니다.

이제 `operator new`의 반환 값을 통해 다섯 번째 인스턴스의 주소를 확인하고 아래와 같이 `Where`를 사용해서 해당 인스턴스의 `process` 호출만 확인하는 것이 가능합니다. 혹은 Calls를 사용하고 `ReturnValue` 필드에 접근해서 해당 position으로 이동하지 않고 알아내는 것도 가능합니다.

```
0:000> dx @$memory(0x7ff784d91265, 0x7ff784d91265+1, "e")[4].TimeStart.SeekTo()
(43f8.567c): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 4D8:24A2
0:000> p
Time Travel Position: 4D8:25EA
TTDCalls!main+0xba:
00007ff7`84d9126a 488930          mov     qword ptr [rax],rsi ds:000001e0`00059750=6562575c32336d65
0:000> r rax
rax=000001e000059750

0:000> dx @$calls("TTDCalls!operator new").Where(x => x.ReturnAddress == 0x7ff784d9126a)[0x8].ReturnValue
@$calls("TTDCalls!operator new").Where(x => x.ReturnAddress == 0x7ff784d9126a)[0x8].ReturnValue                 : 0x1e000059750 [Type: void *]
```

 특정 인스턴스에 의한 호출만 보려면 첫 번째 인자(`this`)가 인스턴스의 주소인 것을 이용하면 되겠죠? 이 예제의 경우에는 pdb는 물론 소스코드까지 있어서 `Parameters`에서 인덱스 대신 파라미터의 이름으로 바로 접근할 수도 있습니다.

```
0:000> dx @$calls("TTDCalls!ConcreteProcessor::process").Where(x => x.Parameters[0] == 0x1e000059750)
@$calls("TTDCalls!ConcreteProcessor::process").Where(x => x.Parameters[0] == 0x1e000059750)                
    [0x6]           
    [0xf]           
    [0x21]          
    [0x2a]          
    [0x2c]          
    [0x2f]          
    [0x32]          
    [0x39]          
    [0x3f]          
    [0x43]          
    [0x57]          
    [0x5e]        
```

사용된 인자만 확인해보려면 아래와 같이 Select를 사용해서 Parameters 필드만 남기고 `-g` 옵션으로 그리드 뷰로 출력하면 됩니다.

```jsx
0:000> dx -g @$calls("TTDCalls!ConcreteProcessor::process").Where(x => x.Parameters.this == 0x1e000059750).Select(x => x.Parameters)
=====================================================================
=           = value  = (+) this         = (+) [0x0]        = [0x1]  =
=====================================================================
= [0x6]     - 397    - 0x1e000059750    - 0x1e000059750    - 397    =
= [0xf]     - 187    - 0x1e000059750    - 0x1e000059750    - 187    =
= [0x21]    - 290    - 0x1e000059750    - 0x1e000059750    - 290    =
= [0x2a]    - 983    - 0x1e000059750    - 0x1e000059750    - 983    =
= [0x2c]    - 209    - 0x1e000059750    - 0x1e000059750    - 209    =
= [0x2f]    - 447    - 0x1e000059750    - 0x1e000059750    - 447    =
= [0x32]    - 123    - 0x1e000059750    - 0x1e000059750    - 123    =
= [0x39]    - 229    - 0x1e000059750    - 0x1e000059750    - 229    =
= [0x3f]    - 248    - 0x1e000059750    - 0x1e000059750    - 248    =
= [0x43]    - 18     - 0x1e000059750    - 0x1e000059750    - 18     =
= [0x57]    - 754    - 0x1e000059750    - 0x1e000059750    - 754    =
= [0x5e]    - 319    - 0x1e000059750    - 0x1e000059750    - 319    =
=====================================================================
```

당연히 `Where`에 다른 조건들을 더 사용할 수도 있고 아래와 같이 TTD position을 비교하는 기능을 extension으로 구현해두고 `Where` 안에서 조건으로 추가하는 것도 가능합니다.

```jsx
function ttdParsePosition(pos){
	var sequence = 0
	var steps = 0
	if(typeof(pos) == "string"){
		sequence = parseInt(pos.split(":")[0], 16)
		steps = parseInt(pos.split(":")[1], 16)
	}
	else{
		sequence = pos.Sequence
		steps = pos.Steps
	}

	return {
		Sequence: sequence,
		Steps: steps
	}
}

function __ttd_position_compare(pos1, pos2){
	/*
	alias: poscmp
	parameters:
		pos1 & pos2: Position object or string(Sequence:Steps in hexadicimal)
	usage:
		> !poscmp(pos1, pos2)
		> dx @$poscmp(pos1, pos2)
	example:
		> dx @$calls("function").Where(x => @$poscmp("17C83A:1777", x.TimeStart) < 0).Where(x => @$poscmp("17C861:1B3A", x.TimeEnd) > 0)
		=> get all `function` calls between 17C83A:1777 and 17C861:1B3A
	*/

	pos1 = ttdParsePosition(pos1)
	pos2 = ttdParsePosition(pos2)

	if(pos1.Sequence != pos2.Sequence){
		return pos1.Sequence - pos2.Sequence
	}

	return pos1.Steps - pos2.Steps
	
}

```

이걸 모두 종합해보면? 아래와 같은 괴물이 탄생합니다…

```
0:000> dx -r1 @$calls("TTDCalls!ConcreteProcessor::process").Where(x => x.Parameters.this == 0x1e000059750)[6]
@$calls("TTDCalls!ConcreteProcessor::process").Where(x => x.Parameters.this == 0x1e000059750)[6]                
    EventType        : 0x0
    ThreadId         : 0x567c
    UniqueThreadId   : 0x2
    TimeStart        : 4DA:10F0 [Time Travel]
    TimeEnd          : 4DA:10F3 [Time Travel]
    Function         : TTDCalls!ConcreteProcessor::process
    FunctionAddress  : 0x7ff784d911a0
    ReturnAddress    : 0x7ff784d9134e
    Parameters      
    SystemTimeStart  : 2025년 5월 31일 토요일 12:24:28.633
    SystemTimeEnd    : 2025년 5월 31일 토요일 12:24:28.633
0:000> dx -r1 @$calls("TTDCalls!ConcreteProcessor::process").Where(x => x.Parameters.this == 0x1e000059750)[94]
@$calls("TTDCalls!ConcreteProcessor::process").Where(x => x.Parameters.this == 0x1e000059750)[94]                
    EventType        : 0x0
    ThreadId         : 0x567c
    UniqueThreadId   : 0x2
    TimeStart        : 4DB:820 [Time Travel]
    TimeEnd          : 4DB:823 [Time Travel]
    Function         : TTDCalls!ConcreteProcessor::process
    FunctionAddress  : 0x7ff784d911a0
    ReturnAddress    : 0x7ff784d9134e
    Parameters      
    SystemTimeStart  : 2025년 5월 31일 토요일 12:24:28.633
    SystemTimeEnd    : 2025년 5월 31일 토요일 12:24:28.633
0:000> dx @$calls("TTDCalls!ConcreteProcessor::process").Where(x => x.Parameters.this != 0x1e000059750 && @$poscmp(x.TimeStart, "4DA:10F0") > 0 && @$poscmp("4DB:823", x.TimeEnd) < 0)
@$calls("TTDCalls!ConcreteProcessor::process").Where(x => x.Parameters.this != 0x1e000059750 && @$poscmp(x.TimeStart, "4DA:10F0") > 0 && @$poscmp("4DB:823", x.TimeEnd) < 0)                
    [0x5f]          
    [0x60]          
    [0x61]          
    [0x62]          
    [0x63]          

```

위 커맨드는 다섯 번째 `ConcreteProcessor` 인스턴스의 첫 번째와 마지막 `process`호출의 position을 확인하고 이 사이에 호출된 다른 인스턴스의 `process`호출만 리스팅합니다.

# 마치며

다 쓰고 나니 뭔가 스크립팅보다도 windbg의 TTD 데이터 오브젝트(Calls, Memory)에 대한 내용이 더 많은것 같네요 😅 [MS 문서](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/debugger/javascript-debugger-scripting)에서 설명하는 내용 위주로 스크립트를 어떻게 작성해서 써야하는지를 다루는 것도 나쁘진 않았겠지만 저걸 읽는게 별로 내키지 않더군요;; 저도 0vercl0k님의 [예제들](https://github.com/0vercl0k/windbg-scripts)로 적당히 공부하고 아주 간단한 스크립트만 사용했는데도 스크립트를 썼을때와 안 썼을때의 차이는 분명했습니다. 특히 TTD만으로도 이미 혁신이라고 생각했는데 기능을 하나씩 알게 될때마다 이거 안쓰고 어떻게 살았나 싶었어요;;

![image.png](windbg_js/image%206.png)

아무튼 이번 글도 읽어주셔서 감사하고 이 기회에 저희 팀원들도 꼭 좀 써봤으면 좋겠네요.