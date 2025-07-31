---
title: "[Research] Starting Chrome Exploitation with Type Confusion 101 ^-^☆ Part 2.(EN)"
author: OUYA77
tags: [Type Confusion 101, Chrome, Chromium, OUYA77, Type Confusion, Relative R/W, OOB]
categories: [Research]
date: 2025-07-30 14:20:00
cc: true
index_img: /2025/07/30/OUYA77/Chrome_part2/kr/TypeConfusion101_Part2.png
---

Hello, this is OUYA77. 2025 is already more than halfway over, and July is almost gone too. Time flies, doesn't it? ㅎ.ㅎ

In this part, I really wanted to take a stab at Chrome exploit development, but because Chrome itself is a complex program with quite a lot of background knowledge required, this post ended up being a bit long. In this section, we'll compare basic **Type Confusion** with Type Confusion in Chrome, and explore how this Type Confusion occurs and ultimately leads to **Memory Corruption**.

Before we dive in, let's briefly recap what we covered last time.

## 0. Part 1. Recap

Last time, we looked at the overall architecture of Chrome and how frontend resources are processed. To be more precise, we examined the process of HTML, CSS, and JS files being rendered in Chrome. In this research post, we'll focus on **V8**, which handles JS files

> If you haven't seen it yet → [[Research] Starting Chrome Exploitation with Type Confusion 101 ^-^☆ Part 1.](https://hackyboiz.github.io/2025/07/01/OUYA77/Chrome_part1/en/)
> 

![image.png](image%201.png)

V8 uses various compiler tiers (Ignition, SparkPlug, Maglev, TurboFan) for fast and efficient JavaScript execution, gradually optimizing code and increasing execution speed. However, this optimization is based on the assumption that "this structure will always be the same." If an attacker changes the object type or structure at a specific point, V8 will execute incorrect native code based on a false assumption. **This is a typical scenario where Type Confusion occurs.**

Now that we understand the basic structure of how code executes within V8, in the next step, we'll delve into how objects are represented and optimized inside this engine, and furthermore, how this structure can lead to security issues like Type Confusion!

## 1. Type Confusion 101

### 1.1 Introduction

**Type Confusion** is, as the name suggests, a vulnerability that arises when a **`Type`** is **`Confused`**. So, what exactly do we mean by 'type' here? In computing, a 'type' refers to the method by which data is stored and processed. Simply put, it serves to distinguish the data kind of the variables we use.

For example:

- **Integer:** Stores numerical data
- **Char:** Stores single character data
- **Array:** Groups multiple data items together

As seen above, types are the basic units for processing data in programming languages, and for a computer to handle data correctly, the exact type must be specified. But what happens if some data, which should originally be an integer, is mistakenly (or intentionally) treated like a character or an object? The side effects of this cannot be predicted, and the program may behave in unexpected ways.

### 1.2 Types of “Type Confusion”

Type Confusion, like other vulnerability classifications, can generally be divided into two types based on their impact: **Logical Bug** and **Memory Corruption**.

- **Logical Bug**

The **Logical Bug** type of Type Confusion occurs when logic processing, which varies according to type, operates incorrectly. In many languages, operators or built-in methods behave differently depending on the input type. If types are confused, the result can be entirely different from the intended flow. Let's assume the following code is part of a larger application.

![image.png](image%202.png)

A developer has implemented a filter to prevent XSS by checking if the `<` character is included in certain input values. The filter was implemented expecting the input to be a string, but what if an array object consisting of strings is actually passed? In JavaScript, if this object is directly checked, the comparison result might be `false` even if it contains `<`. This can lead to **filter bypasses** and, as a result, serious logical vulnerabilities such as **XSS (Cross-Site Scripting)** or privilege escalation.

- **Memory Corruption**

The **Memory Corruption** type is a critical vulnerability that occurs when a **memory layout is misaligned due to incorrect type casting**. Especially in languages like C/C++, misinterpreting the size or structure of an object can lead to unintended access to adjacent memory (**Out-Of-Bounds Read/Write**). This is a major cause threatening program stability and security.

For example, let's look at the following code.

```c
typedef struct {
    int value[2];
} SmallStruct;

typedef struct {
    int data[4];
} LargeStruct;

SmallStruct normal = {0x41, 0x42};
print_memory((int*)&normal , sizeof(normal)/sizeof(int)); 

// Cast SmallStruct array to LargeStruct
LargeStruct* confused = (LargeStruct*)&normal ;
confused->data[2] = 0xdead; // OOB Write
confused->data[3] = 0xbeef;
print_memory((int*)&normal, sizeof(LargeStruct)/sizeof(int)); // OOB Read
```

In the example above, `SmallStruct` is a structure holding two integers. In contrast, `LargeStruct` contains an array of four integers. If a developer incorrectly casts a `SmallStruct` instance to a `LargeStruct` pointer, the program will misinterpret the size of `normal` based on the memory layout of `LargeStruct`.

Looking at the output, 

![image.png](image%203.png)

you can confirm that the area outside the bounds of the `SmallStruct` structure has been modified by the write operation. That is, `confused->data[2]` and `data[3]` accessed memory beyond the defined structure size of `normal`.

This can be visualized as follows.

![image.png](image%204.png)

The diagram illustrates how, even though the `SmallStruct` instance occupies only two integer spaces, it is accessed through a `LargeStruct` pointer as if four integer spaces exist, leading to out-of-bounds read/write operations.

### 1.3 Type Confusion (case: in Chrome)

Since we're focusing on Type Confusion in Chrome, we'll further explore how this vulnerability manifests in Chrome by comparing two typical cases of **Memory Corruption** type confusion: C++ and JavaScript.

- **Type Confusion – C++**

In statically-typed languages like C++, Type Confusion primarily occurs due to developer errors. Since types are determined at compile-time, incorrect explicit casting is the main cause.

```jsx
// Type Confusion caused by incorrect Casting
struct A { int x; };
struct B { int y; void (*func)(); };

A a = {42};
B* b = (B*)&a; 
// Attempts to call an undefined function pointer → Crash or arbitrary code execution
b->func();     
```

In the example above, `struct A` only contains an integer member `x`. However, if it's cast to `struct B`, it will attempt to access a non-existent function pointer (`func`). This leads to **undefined behavior**, which can result in a system crash or even the execution of attacker-supplied code.

- **Type Confusion – JavaScript**

On the other hand, in dynamic languages like JavaScript, Type Confusion can occur even without the developer explicitly changing types. Especially in modern JS engines like V8, the **JIT (Just-In-Time) compiler** can make incorrect assumptions during the type inference process for runtime performance optimization, leading to Type Confusion (The details will be covered later!).

```jsx
function foo(x) {  
    let arr = [1.1]; // Initially an array containing a number (float)
    if (x) arr[0] = {y: 42}; // Conditional branch to change to an object
    return arr[1]; // Potential OOB
}

for (let i = 0; i < 1000; i++) foo(false);  // Induce optimization

foo(true); // After optimization, insert an object → change in type structure
```

Initially, this code's array contains only numbers, leading V8 to assume, "this array only holds numbers." Accordingly, it optimizes its internal structure for simple and fast processing. However, if an object is inserted into the array based on the conditional statement, the internal structure should change. If the already generated optimized code doesn't account for this change, the engine might still access the array assuming it only contains numbers. As a result, a value assumed to be a number might actually be an object or entirely different data, leading to **memory misinterpretation** or **Out-Of-Bounds access** to the array. This kind of mismatch between runtime structural changes and the JIT compiler's inference is a primary cause of Type Confusion in JavaScript.

## 2. Type Confusion in Chrome - Root cause Analysis

In Chapter 1, we explored the concept of Type Confusion and its common occurrences.
Now, let's apply this concept to **V8**, Chrome's JavaScript engine. V8 is a high-performance, JIT-based engine, which makes it an environment where Type Confusion vulnerabilities frequently occur. For performance optimization, V8 internally tries to maintain object type information in a fixed form. This can lead to type inference errors resulting in memory corruption under specific conditions. In this chapter, we will examine the structural background of how Type Confusion occurs within V8 and the role of key concepts like **Hidden Class** and **ElementsKind**.

### 2.1 Overview of Type Confusion Cases in Chrome

Indeed, numerous Chrome vulnerabilities, such as **CVE-2018-17463**, **CVE-2023-2033**, and **CVE-2023-4069**, have originated from Type Confusion due to incorrect type inference. (You can also find them in HackyBoys 1day-1line! → [Link](https://hackyboiz.github.io/2024/01/18/ogu123/cve-2023-2033/))

Most of these occur when the JIT compiler observes code execution patterns and generates optimized code based on assumptions like "this variable always handles numbers" or "this object always has the same structure." The vulnerability then arises when these assumptions are broken by an exceptional execution flow later on.

In the V8 engine, Type Confusion vulnerabilities frequently occur in the following situations:

- When the object's structure **(Hidden Class)** has changed, but operations continue based on previous assumptions.
- When changes to the **array's internal type (ElementsKind)** are not reflected.
- When incorrect type hints left by the **Inline Cache** are used.

In summary, V8 generates code assuming "types will be fixed" for fast execution. However, JavaScript is a highly dynamic language, and this assumption can be broken at any time. Attackers can exploit these structural characteristics to disrupt internal type information and cause Type Confusion.

We will now delve into two core optimization concepts that form the basis of these structural assumptions: **Hidden Class (Map)** and **ElementsKind**.

### 2.2 Introduction to V8's Optimization Model (1): HiddenClass (Map)

A **Hidden Class** is an internal mechanism V8 uses to track and optimize the structure of JavaScript objects. JavaScript objects are inherently dynamic, allowing properties and value pairs to be freely added or removed, which means their structure can change on the fly. However, for performance, V8 doesn't treat objects like dictionaries. Instead, it internally tracks the property configuration and order of objects using a meta-structure called a Hidden Class (or Map).

> **Property:** While properties in JavaScript objects simply appear as key-value pairs, internally in V8, each time a property is added, it triggers a change in the object's memory layout. At this point, not only the property's name but also the **order in which it's added** becomes a crucial factor in determining the object's structure.
> 
> 
> **Transition:** When a new property is added, V8 connects (transitions) the object's current Hidden Class from its existing structure to a new one. Each such transition corresponds to the addition of one property, forming a **tree structure** composed of connections between Hidden Classes.
> 

**🧩 How Hidden Class Works**

```jsx
let obj = {};        // [1] Initial Hidden Class
obj.x = 10;          // [2] → Transitions to a Hidden Class with property 'x'
obj.y = 20;          // [3] → Transitions to a Hidden Class with properties 'x, y' in that order
```

- [1] When an object is created, V8 assigns an **initial Hidden Class** to it. This represents a state where the object has no properties
- Subsequently, each time a property is added, V8 performs a **transition** from the existing Hidden Class to a new one
    - [2] if `obj.x = 10` is added, it transitions to a new Hidden Class containing only `x` .
    - [3] when `obj.y = 20` is added, it transitions again to another Hidden Class that reflects the order `x, y`.
        
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
        

As shown, Hidden Classes are sequentially linked according to the property addition order. Objects with identical structures share the same Hidden Class. The Hidden Class expands, forming a branching tree structure based on the property addition order of an object, internally representing the static structural information of each object.

Due to this structure, when accessing properties like `obj.x`, V8 doesn't search for a string key. Instead, it directly accesses a fixed memory location, such as `obj[+8]`, based on the offset information held by the corresponding Hidden Class. This method significantly **speeds up property lookup** and enables V8 to generate highly optimized native code, under the assumption that the object's structure will not change.

### 2.3 Introduction to V8's Optimization Model (2): ElementsKind

**ElementsKind** is an internal meta-information V8 uses to track the **element type and density of JavaScript arrays**. JavaScript arrays have a very dynamic structure; they can freely mix numbers, strings, and objects, or contain sparse (empty) slots. However, for performance optimization, V8 tries to represent these arrays internally in the simplest and most standardized way possible. ElementsKind is the criterion for this internal representation.

🧩 **How ElementsKind Works**

- When an array is created, V8 assigns the simplest and most compact **ElementsKind** based on the array's content.
    
    For example:
    
    ```jsx
    [1, 2, 3]           → PackedSmiElements (32bits integer array)
    [1.1, 2.2]          → PackedDoubleElements (64bits floating-point array)
    [1, {}, "text"]     → PackedElements (general object array)
    ```
    
- Subsequently, if elements of a different type are inserted into the array, or if intermediate elements are deleted reducing its density, V8 transitions the array to a more general ElementsKind.
    
    ```
    let arr = [1.1, 2.2];   // Initial: PackedDoubleElements
    arr[2] = {};            // → Transitions to PackedElements
    ```
    

This approach somewhat contrasts with the dynamic nature of JavaScript arrays but provides very high performance until a change occurs during execution. Ultimately, ElementsKind provides a core criterion for JIT optimization based on abstract type information about array elements, playing a crucial role in converging dynamic JavaScript code into a static performance model.

### 2.4 Optimization Flow and Native Code Generation

V8 generates high-performance **Native Code** by making the most static assumptions possible about JavaScript's dynamic nature. As discussed earlier, **Hidden Class** and **ElementsKind** make object and array structures predictable, allowing V8 to progressively generate optimized code during execution. In the previous section (Part 1.), we covered the overall concept and history of the V8 execution pipeline structure. Now, let's look at these components from the perspective of native code execution and optimization flow.

---

**⚙️ Native Code Execution and Optimization Flow**

![image.png](image%205.png)

V8's execution pipeline transforms JavaScript code into increasingly fast native machine code through the following stages.

1. The **Ignition interpreter** converts JavaScript code into bytecode and begins execution. In this stage, runtime profiling information, such as execution counts and type information, is collected.
2. If code is determined to be a "hot" execution path, the **Sparkplug** or **Maglev JIT compilers** perform rapid compilation to generate Native Code. This code represents an initial level of optimization, prioritizing speed and efficiency.
3. Subsequently, once the code is deemed sufficiently analyzed, **TurboFan**, the advanced JIT compiler, performs more sophisticated optimizations. At this point, it actively utilizes internal structural information like **Hidden Class** and **ElementsKind** to generate high-performance Native Code, assuming that object structures remain fixed.
    
    > e.g., When accessing `obj.x`, V8 can use the Hidden Class to know that the offset of "x" is `+8` and generate code that directly references that memory location. Similarly, array memory access is statically optimized via ElementsKind.
    > 

---

During this process, TurboFan employs various advanced optimization techniques, including Inline Caching (IC), Escape Analysis, Loop Unrolling, and Function Inlining. The Native Code generated this way offers very high execution performance but concurrently relies on strong static assumptions about the object's internal structure and type.

V8 registers this optimized code at the function's entry point based on these assumptions. As a result, subsequent function calls bypass the interpreter stage and execute the Native Code directly, dramatically improving execution speed.

However, this optimization is premised on the following assumption:

> "The structure of objects or types of arrays will not change during runtime."
> 

The problem is that, due to the dynamic nature of JavaScript, the structure of objects or arrays can change at any time. If such a change is detected, V8 will **deoptimize** the existing Native Code, revert to the interpreter, or generate newly optimized code.

However, if these changes are not properly detected, or if they are intentionally bypassed by an attacker, V8 will continue to execute the optimized path based on the previous, incorrect type information. As a result, a situation can arise where the **Inline Cache** still uses old information even though the **Hidden Class** or **ElementsKind** has already changed. This leads to incorrect field offset calculations or erroneous memory references. Ultimately, this context is the direct cause of **Type Confusion** vulnerabilities.

As such, V8's optimization pipeline is a core mechanism for converging JavaScript's dynamic characteristics into a static execution model. Hidden Class and ElementsKind form the foundation of this static model, and the Native Code generated based on them provides very high performance. 

![image.png](image%206.png)

However, simultaneously, a vulnerability can arise the moment this static assumption is broken. Therefore, managing the **trade-off between performance and security** becomes a very critical challenge in V8's design.

### 2.5 The Structure of Type Confusion Occurrence

As previously explained, V8's optimization mechanisms are based on a strong premise that object structures and array types will not change. However, JavaScript is fundamentally a very dynamic language, and Type Confusion can occur the moment this premise is broken in the actual execution flow.

Let's specifically examine how Type Confusion can be triggered within **Hidden Class** and **ElementsKind**, which form the basis of V8's static model.

---

**1) Hidden Class-based Confusion**

A Hidden Class is an internal mechanism that tracks an object's property structure, enabling fast property access and JIT optimization. However, when properties are dynamically added to an object or property types change, the Hidden Class transitions to a different class, rendering the existing Native Code invalid.

```jsx
function Foo() {
  this.x = 1;  // HiddenClass A
  this.y = 2;  // HiddenClass B
}
let a = new Foo();
a.z = 3;       // HiddenClass C (transition)
```

In this case, object `a` ultimately gets Hidden Class C. But if TurboFan generated Native Code based on the structure up to Hidden Class B, and that code is still applied to `a`, it might incorrectly perceive `a.z` as `a.y` or reference an incorrect offset.

**Hidden Class Transition and Deoptimization**

Changes in property types also trigger Hidden Class transitions. Consider, for example, a case where an object property's type changes from a number to an object.

```jsx
let obj = { x: 1 };         // x is number → HiddenClass A
obj.x = { nested: true };   // x is now object → HiddenClass B
```

In this scenario, V8 detects the type change and deoptimizes the existing Native Code, either reverting to bytecode or generating new code. However, if the structural change is not properly detected, or if an attacker designs the input to bypass this detection, V8 may still execute the existing Hidden Class-based Native Code, leading to incorrect memory access.

**2) ElementsKind-based Confusion**

For arrays, V8 defines various **ElementsKind** types based on the elements' types and density. 

For example:

- `PACKED_SMI_ELEMENTS`: Dense array containing only integers.
- `PACKED_DOUBLE_ELEMENTS`: Array containing only floating-point numbers.
- `PACKED_ELEMENTS`: Mixed-type array.
- `HOLEY_*` (sparse array) variants corresponding to each of the above.

TurboFan generates optimized code assuming that an array's ElementsKind will not change. However, the moment an element of a different type is inserted into the array, the ElementsKind transitions, and the existing Native Code becomes invalid.

```jsx
function confuse(x) {
  let arr = [1.1];     // PackedDoubleElements
  if (x) arr[0] = {};  // insert object → ElementsKind transition
  return arr[0];
}
```

If `x` is `false`, `arr` remains a double array, and TurboFan generates corresponding code. However, if `x` is `true`, an object is inserted into `arr[0]`, changing the array's internal representation. If the existing Native Code is still executed at this point, an incorrect type casting occurs, interpreting an object as a double.

## 3. From Type Confusion to Memory Corruption

### 3.1 Connecting to Practical Memory Corruption

Why does simple Type Confusion escalate to **Memory Corruption**?

The core reason is that V8's JIT-optimized Native Code directly accesses memory, relying on its trust in object structures or array types. V8 rapidly accesses objects or arrays by calculating precise field offsets based on **HiddenClass (Map)** or **ElementsKind** information. The problem arises when these assumptions are broken. If an object's actual structure changes but the compiled code still accesses it based on the old structure, it will access completely incorrect memory addresses or misinterpret data. This can directly lead to **memory corruption**, manipulating memory at unexpected locations.

Consider the following scenarios.

- **Misinterpreting a Floating-Point Array as an Object Array**
    
    If an array storing floating-point values like `13.37` is mistakenly treated and accessed by the JIT as an object array, this numerical value could be interpreted as a heap pointer, leading to access of an entirely unintended memory location. This results in **OOB Read/Write**, which an attacker can leverage to manipulate adjacent objects or induce data leakage.
    
- **Pointer Manipulation using TypedArray or DataView**
    
    In JavaScript, when using `TypedArray` or `ArrayBuffer`, the buffer directly points to an actual memory region. If an attacker can control the object's structure, they might manipulate this **backing store pointer** to set an arbitrary memory address as the buffer's starting point. In this case, read/write operations through the `ArrayBuffer` expand to memory access across the entire address space. 
    

This implies the ability to access memory near the object or specific fields within the same structure in a limited way, by manipulating the internal field offsets of the object through incorrect type interpretation.

One more crucial factor here is the **Garbage Collector (GC)**. V8 uses a GC system that automatically manages the lifespan of JS objects. This system moves or reclaims objects within the heap as needed. Even if an object's memory address changes during this process, the existing optimized code might not recognize it and could attempt to access it based on its old structure. In other words, a timing difference or information mismatch between the GC and JIT can act as another **Type Confusion trigger**.

Consequently, we can see that Type Confusion is not merely a logical error or a developer's mistake, but rather a vulnerability that can arise at the complex boundary where V8's JIT optimization, object model, and GC system interact.

Now that we understand why Type Confusion occurs and how it can lead to Memory Corruption, let's explore how to initiate an attack using this vulnerability.

### 3.2 Relative Read/Write Primitive

The basic exploit flow for Type Confusion starts with an OOB Read/Write stemming from the Type Confusion itself. The general attack flow to achieve **RCE via Type Confusion** involves carefully refining the OOB R/W to obtain a **Relative R/W** primitive, then skillfully leveraging this to gain **AAR/W** (Arbitrary Address Read/Write), and finally achieving Code Execution. Sounds exciting, right?! This part is turning out to be longer than expected, so I probably won't cover everything. But it'd be a shame to end here, so let's briefly touch on the Relative R/W portion as a sneak peek :)

---

Type Confusion means more than just a simple misinterpretation of types. From an attacker's perspective, it becomes a **starting point for manipulating memory regions that were originally inaccessible**. A prime example of this is the **Relative Read/Write**. This is a primitive that allows limited access to memory based on a fixed offset from a known location.

For instance, consider two classes with similar structures but where the meaning of the last field differs.

![image.png](image%207.png)

> Class A
- name
- age
- residence
>
>Class B
- name
- age
- workspace
> 

These two classes have a similar number and order of fields, but the meaning and type of the last field are different. Let's imagine that in a Worker process, an operation points to the "residence" field of a `Class A` instance.

![image.png](image%208.png)

Now, let's assume a situation within the Worker process where a `Class A` instance is mistakenly perceived as a `Class B` type for some reason. In this case, under normal circumstances, the "residence" field should be accessed. However, when interpreted based on `Class B`, this memory location is recognized as the field corresponding to "workplace.”

![image.png](image%209.png)

As a result, even though the Worker process is actually accessing a field of a `Class A` instance, the runtime considers it a field of `Class B`, leading to unintended memory reads or writes.

> It'd be quite embarrassed if a delivery meant for your home ended up at your workplace, wouldn't it?
> 
> Here's another embarrassed example.
> 
> ![image.png](image.png)
> 

As shown, the runtime's incorrect type interpretation creates a **Relative R/W Primitive**, enabling memory access based on a fixed offset, even if the exact address isn't known. Through this, an attacker can satisfy the following conditions:

- Based on an object they control (`Class A`), at a fixed offset (e.g., `+0x10`, `+0x18`, etc.),
- They can read or write the contents of a specific field or an adjacent structure (`Class B`) within memory.

---

Such a **Relative Primitive** holds significant meaning in the early stages of an attack:

- It allows for leaking the internal structure of a specific object, or
- It enables the manipulation of key information like pointers or length fields,
- Thereby paving the way for expansion into an Absolute R/W Primitive.

In the next part, we'll explore through an actual exploit flow how this Relative R/W Primitive expands into absolute address-based memory manipulation, ultimately leading to remote code execution.

From the next part, you can expect a strong "pwnable" scent; please look forward to it! I'll do my best to prepare it...!

(The following is just a humorous, digitally distorted prank call photo I added because I wanted this kind of vibe. But Liam Neeson and a Chinese restaurant are better, right? 🥵)

![image.png](image%2010.png)

(subtitle: Robbers at home lol)

## Reference

v **Conference Video**

- https://www.youtube.com/watch?v=RL2po1swXO4

v **Background**

- [https://v8.dev/docs/hidden-classes](https://v8.dev/docs/hidden-classes)
- [https://v8.dev/blog/elements-kinds](https://v8.dev/blog/elements-kinds)
- [https://richardartoul.github.io/jekyll/update/2015/04/26/hidden-classes.html](https://richardartoul.github.io/jekyll/update/2015/04/26/hidden-classes.html)