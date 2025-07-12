---
title: "[Research] A spoonful of javascript for windbg"
author: pwndorei
tags: [pwndorei, windbg, TTD, Debugging]
categories: [Research]
date: 2025-06-01 23:00:00
cc: true
index_img: /2025/06/01/pwndorei/windbg_js_en/image%206.png
---

# Introduction

Hello, this is **pwndorei**… Since I started working at a company around last September, I’ve stopped writing research posts for a while, but now I can’t avoid it anymore so I’m back 🤦🏻‍♂️. However, aside from work, I don’t really have anything I’m researching, so no topic came to mind that was worth writing about.  
So I decided to simply present a JavaScript script that’s handy when analyzing with WinDbg. All right, let’s go!

# Windbg + Script ⇒ ?

Using a script for debugging? The goal is one and the same—to debug lazily.

![image.png](windbg_js_en/image.png)

> ~~It’s absolutely *not* about working hard in advance so you can relax later~~

Of course, you *can* carefully `dq`, `dd`, etc. every single time to check what data lives in a crazy-looking data structure. And with TTD, even if you accidentally step over an important call, you don’t have to relaunch the process, so you can just smash **F10** mindlessly while analyzing.  
But! The larger the data gets, the less time it takes to write a script instead of doing it by hand, and if you rely on TTD too much you’ll turn into a fool.  
So today, with a simple example, we’ll see how to use scripts effectively—and as a bonus I’ll also show you how to push TTD to the limit.

# Windbg Script Basic

In WinDbg you can create a new script or load an existing one from the **Scripting** menu, like below. Besides JavaScript there’s something called **NatVis**, but since we’re going to use JavaScript we’ll just ignore that.

![image.png](windbg_js_en/image%201.png)

Next, in **New Script** you choose between **Extension** and **Imperative**; it’s merely a difference in the default template.

For **Imperative**, the following boilerplate code is generated:

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

You can run it by pressing **Execute** in **Actions** or with commands such as `.scriptrun`.

![image.png](windbg_js_en/image%202.png)

For **Extension**, it’s used literally to add extension functionality, and this template appears:

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

Extensions added through a script can be invoked in WinDbg with a command like `!extname`.

We’ll see what functions we can implement through examples later, but first a few more script-related commands:

- `.scriptlist`  
  Shows all currently loaded scripts.
  
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
  Commands to load or unload a script.  
  When loading, you can give the full path or just the name; WinDbg searches any directories in the `PATH` environment variable.  
  `.scriptload` only calls the script’s `initializeScript` function.
- `.scriptrun`  
  Unlike `.scriptload`, it also calls the `invokeScript` function.

There are two commands—`.scriptload` and `.scriptrun`—that execute scripts. If the script is **imperative**, you need to run `invokeScript`, so you use **run**; if it’s an **extension**, you use **load**, right?

![image.png](windbg_js_en/image%203.png)

## Hello World

We’re not learning a *new* language, but because we can’t use `console.log` the way we do in the browser, let’s start by printing “Hello World” to the console.

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

As you can see, you use `host.diagnostics.debugLog` instead of `console.log`. If you run it, it prints normally.

![image.png](windbg_js_en/image%204.png)

Since I have no intention of typing that code every time I need to log, let’s make an alias and use that instead 🤣

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

We’re not going to write scripts just to print “Hello World”, right? `host.memory` has the following functions, which let you read and write data in memory:

- `readMemoryValues`
- `readWideString`
- `readString`
- `writeMemoryValues`

For today, knowing this much is plenty!

# Practice

Now let’s debug a sample program and see in what situations a script is useful.

## 1. Dump Data Structure

First up is dumping the data structure we mentioned earlier. ChatGPT made the sample program like this:

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

It’s simple code that stores strings in a binary tree in order. It prints the root node’s address and waits for console input. We can attach the debugger and write a script that traverses the tree to print the data.

![image.png](windbg_js_en/image%205.png)

A `Node` is a 0x18-byte structure with three pointer fields, so I defined `getNode` like this:

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

Reading memory is done with `host.memory.readMemoryValues`. As you can guess from the parameters, it reads 3 elements (second parameter) of 8 bytes each (third parameter, `elementSize`) starting at `addr`. The return value is an array of integers containing the data, and I use it to build a node object.

Now that we can use the `Node` structure stored in memory, we need a traversal function. Below is a simple in-order traversal: given a node returned by `getNode`, it reads the string at `word` and prints it.

```jsx
function traverse(node){
    if(node["left"] != 0) traverse(getNode(node["left"]))
    log(host.memory.readString(node["word"]) + "\n")
    if(node["right"] != 0) traverse(getNode(node["right"]))
}
```

To read a string, use `host.memory.readString` like above. If it were a wide string, you’d use `readWideString` instead. The full script is:

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

If you want to go beyond simple printing and, say, find the address where specific data is located, just add a condition and print it out:

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

This script checks whether the string “melon” is stored, and if so prints its address. Result:

```
Found at 25dd2c21280
0:004> da poi(25dd2c21280)
0000025d`d2c19360  "melon"
```

## 2. TTD

We’ve used an **imperative** script, so now it’s the **extension**’s turn. For the extension example, here’s a simple script I always use with TTD:

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

See the main difference from an **imperative** script? In `initializeScript` I create aliases with `host.functionAlias`. An extension registers JavaScript functions as aliases so they can be used directly at the command line.

I shorten the frequently used `@$cursession.TTD.Calls` and `@$cursession.TTD.Memory` to `@$calls` and `@$memory` to save my fingers 😎

```jsx
function __calls(...args){
	return host.currentSession.TTD.Calls(...args)
}

function __memory(...args){
	return host.currentSession.TTD.Memory(...args)
}
```

So where do we use this?

### TTD Calls & Memory

[The Time-Traveling Hacker’s Guide](https://hackyboiz.github.io/2021/03/14/fabu1ous/ttd-3/#Filtering-in-a-query) by Fabu1ous briefly explains Calls and Memory, but only skims over what you can do. Let’s see how to use them through an example.

First, load the extension with the script below:

```
0:038> .scriptload utils.js
JavaScript script successfully loaded from 'D:\windbg_Ext\utils.js'
```

Then you can access Calls and Memory like this:

```
0:000> !calls "kernelbase!CreateFileW"
@$calls("kernelbase!CreateFileW")                
    [0x0]           
    [0x1]           
    ...
0:000> dx @$calls("kernelbase!CreateFileW")[0]
@$calls("kernelbase!CreateFileW")[0]                
    EventType        : 0x0
    ThreadId         : 0xe5c
    UniqueThreadId   : 0x2
    TimeStart        : 6F4F:1742 [Time Travel]
    TimeEnd          : 6F57:7E3 [Time Travel]
    Function         : kernelbase!CreateFileW
    FunctionAddress  : 0x7ffbf27f05f0
    ReturnAddress    : 0x7ffbf2835846
    ReturnValue      : 0x31c
    Parameters      
    SystemTimeStart  : Fri May 30 2025 09:13:24.782
    SystemTimeEnd    : Fri May 30 2025 09:13:24.782
```

`!calls` does not let you directly index the returned array, so I more often use `dx @$calls`.

Now let’s take a TTD recording of the example below and see what we can do.

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

> ~~It’s no secret that ChatGPT wrote this too~~

The code creates 10 instances, calls a virtual function 100 times, chooses random values and random instances, then waits.

Though extreme, I often face similar situations while analyzing.  
My main purposes for using Calls and Memory are:

1. Only function calls made by a specific instance  
2. Filtering by return value or argument  
3. Filtering calls or memory accesses before/after a given TTD position

First, check how many times the function was called:

```
0:000> dx @$calls("TTDCalls!ConcreteProcessor::process").Count()
@$calls("TTDCalls!ConcreteProcessor::process").Count() : 0x64
```

That’s 100 times. To see only calls made by the 5th instance, we first need that instance’s address. Because it’s created with `std::make_unique` (inlined), we can’t directly list constructor calls.

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

But we can list executions of the `operator new` instruction and pick the 5th one:

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

By looking at the `TimeStart` and `TimeEnd` fields, we can see when this memory access event started and when it ended. We can jump to that point in time by pressing `[Time Travel]` next to the position, or by calling the `SeekTo` method of `TimeStart` or `TimeEnd` directly.

Now we can determine the address of the fifth instance from the return value of `operator new` and use `Where` to see only the `process` calls of that instance, as shown below, or we can use Calls and access the `ReturnValue` field to find out without going to the position.

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

If you only want to see calls made by a specific instance, you can use the first argument (`this`), which is the address of the instance, right? In this example, we have the pdb as well as the source code, so we can also go directly to the name of the parameter instead of the index in `Parameters`.

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

Need only the arguments? Pipe to `Select(...)` and display with `-g`.

Of course, you can use other conditions in the `Where`, or you can implement the TTD position comparison as an extension and add it as a condition in the `Where`


Putting it all together, you can produce queries like:

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

The above command determines the position of the first and last `process` calls of the fifth `ConcreteProcessor` instance and lists only the `process` calls of other instances that were called between them.

# Wrapping up

After finishing, it feels like this post talks more about WinDbg’s TTD data objects (`Calls`, `Memory`) than about scripting itself 😅. It wouldn’t have been bad to cover how to write and use scripts based on the [MS documentation](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/debugger/javascript-debugger-scripting), but I didn’t feel like reading it all;; I also learned by roughly studying **0vercl0k**’s [examples](https://github.com/0vercl0k/windbg-scripts) and using only very simple scripts, yet the difference between *with* and *without* scripts was clear. I already thought TTD alone was revolutionary, and every time I learn a new feature I wonder how I ever lived without it.

![image.png](windbg_js_en/image%206.png)

Anyway, thanks for reading this post, and I hope my teammates give it a try!
