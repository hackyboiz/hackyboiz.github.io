---
title: "[Research] Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 2 - NT Heap"
author: L0ch
tags: [L0ch, lfh, windows, heap, nt heap, research]
categories: [Research]
date: 2021-02-28 14:00:00
cc: true
index_img: 2021/02/28/l0ch/pwncoolsexy-part2/Untitled.png
---

안녕하세요! L0ch입니다. 오늘은 이전 글에서 예고한 대로 NT heap의 Low Fragmentation Heap에 대해 쓰려고 했지만..  Windows Heap에 대한 전반적인 정리와 같이 쓰는 것이 좋을 것 같아 Windows Heap으로 주제를 확장해봤습니다. 그래서! 오늘은 WIndows가 Heap을 어떻게 관리하는지 하나씩 뜯어보면서 살펴보도록 할게요!

# Windows Heap

윈도우의 Heap 할당 메커니즘은 NT Heap과 Windows 10에서 추가된 Segment Heap 두 가지가 존재합니다. NT Heap은 Windows의 기본 memory allocator이며 Segment Heap은 최근에 도입된 만큼 더 효율적인 memory allocator입니다.

여담으로 Segment Heap은 도입 초기에는 일부 시스템 프로세스, MS 엣지 브라우저, UWP(Universal Windows Platform) 앱 등에서만 사용하고 있었으나 2020년 5월 업데이트 (버전 2004, 빌드 19041)로 일반적인 Win32 어플리케이션도 사용할 수 있게 되었다고 합니다.. 만 아직은 일반적인 어플리케이션에서는 NT Heap을 사용하는 추세죠. 메모리 잡아먹는 하마 크롬도 Segment Heap을 적용한 패치 이후 메모리 사용량은 줄였지만 성능 저하도 같이 나타나서 다시 비활성화했다고 하네요..ㅋㅋ



![](pwncoolsexy-part2/Untitled%201.png)

Segment Heap에 대해서는 다음에 다뤄보도록 하고, 이번 글에서는 NT Heap에 대해 정리해보겠습니다. 

# NT Heap

NT Heap에 대한 Overview부터 짚고 넘어가 볼까요?

![](pwncoolsexy-part2/Untitled%202.png)

위 사진은 일반적인 `malloc()`과 `free()`를 사용해 Heap을 할당할 때의 내부 할당 프로세스입니다. 

NT Heap은 크게 Front-End와 Back-End로 나눠지며 Front-End가 이후에 설명할 Low Fragmentation Heap(LFH)을 의미합니다. LFH가 비활성화되어 있다면 Heap 메모리 할당 시 Front-End를 건너뛰고 바로 Back-End로 넘어갑니다.

# Low Fragmentation Heap

Heap Fragmentation(단편화)은 반복되는 할당과 해제 이후 가용 메모리가 작고 불연속적으로 나뉜 상태를 말합니다. 이러한 단편화가 발생하면 가용 메모리보다 작은 크기에 대해 할당을 요청해도 연속적인 메모리 구간 확보가 되지 않아 메모리 할당 요청이 실패할 수 있습니다.

![](pwncoolsexy-part2/Untitled%203.png)

> 단편화로 인해 가용 메모리가 충분한데도 할당이 실패할 수 있다.

초기 윈도우에서도 이런 단편화 이슈가 존재했으며 MS는 이런 단편화 이슈를 해결하기 위해 LFH를 도입했고 vista부터 시스템에서 자동으로 설정되면서 본격적으로 사용되기 시작했습니다.

LFH는 동일한 크기의 Heap 메모리를 여러 번 할당하다 보면 자동으로 활성화됩니다. 간단한 테스트 코드로 어느 시점에서 LFH가 활성화되는지 볼까요?

```c
#include <Windows.h>
#include <stdio.h>

#define CHUNK_SIZE 0x300

int main() {
    int i;
    LPVOID chunk;
    HANDLE defaultHeap = GetProcessHeap();
    int prev_chunk = 0;
    for (i = 0; i < 30; i++) {
        chunk = HeapAlloc(defaultHeap, 0, CHUNK_SIZE);
        printf("[%d] Chunk 0x%p   ", i, chunk);

        if (i != 0)
            printf("offset : %d\n", (int)chunk - prev_chunk);
        else
            printf("\n");
        prev_chunk = (int)chunk;
    }

    system("PAUSE");
    return 0;
}
```

<p><img src="Untitled 4.png" height="550px" width="500px" ></p>

동일한 크기의 Heap을 연속적으로 할당했을 때의 결과입니다. 

- 인덱스 0과 1을 제외한 인덱스 16까지의 chunk 사이 거리가 일정함
- 인덱스 17부터 chunk 간 거리가 일정하지 않음

메모리 상황에 따라 조금씩의 차이는 있지만 대부분 이러한 특성을 가지고 있습니다.

chunk가 LFH에 의해 관리되고 있는지는 Windbg에서 다음 명령어로 확인할 수 있습니다.

```bash
!hepa -x [Chunk Address]
```

![](pwncoolsexy-part2/Untitled%205.png)

위 사진에서 인덱스 0 ~ 16 사이 일정한 거리로 할당되는 chunk들은 LFH로 관리되지 않고, 인덱스 17부터 무작위의 거리를 갖는 chunk는 LFH로 관리되고 있음을 확인할 수 있습니다. 

여기서 LFH의 특징 한 가지를 알 수 있습니다!

> LFH가 활성화되면 Heap Chunk 간 거리가 무작위로 할당된다.

이는 Windows에서 Exploit을 진행할 때 Heap을 원하는 대로 세팅하는 Heap Grooming을 더 어렵게 만들어 reliable 한 exploit 제작을 어렵게 합니다. 

- 16KB(0x4000) 보다 큰 Heap을 할당할 경우에는 LFH가 관리하지 않습니다.

# NT Heap Analysis

이제 NT Heap을 본격적으로 분석해보겠습니다! 우선 Heap 관리에 사용되는 구조체와 역할에 대해 정리하겠습니다.

## _HEAP

memory allocator의 핵심 구조체로, 전체 Heap을 관리할 때 쓰입니다. 굳이 분류하자면 NT Heap보다 상위의 구조체로 프로세스에 할당된 Heap의 시작에 위치하는 구조체입니다.

동일한 테스트 코드를 실행한 뒤 Windbg에서 직접 확인해볼게요. 

<p><img src="Untitled 6.png" height="560px" width="500px" ></p>

프로세스의 Heap List는 PEB의 `ProcessHeaps` 필드에 위치합니다. `dt _PEB @$peb` 명령어로 모든 PEB 필드 값을 출력할 수 있습니다. 

```text
0:001> dt _PEB @$peb
ntdll!_PEB
   +0x000 InheritedAddressSpace : 0 ''
   +0x001 ReadImageFileExecOptions : 0 ''
   +0x002 BeingDebugged    : 0x1 ''
   +0x003 BitField         : 0x4 ''
   +0x003 ImageUsesLargePages : 0y0
   +0x003 IsProtectedProcess : 0y0
   +0x003 IsImageDynamicallyRelocated : 0y1
   +0x003 SkipPatchingUser32Forwarders : 0y0
   +0x003 IsPackagedProcess : 0y0
   +0x003 IsAppContainer   : 0y0
   +0x003 IsProtectedProcessLight : 0y0
   +0x003 IsLongPathAwareProcess : 0y0
   +0x004 Padding0         : [4]  ""
   +0x008 Mutant           : 0xffffffff`ffffffff Void
   +0x010 ImageBaseAddress : 0x00007ff6`c6c30000 Void
   +0x018 Ldr              : 0x00007ff9`905b94c0 _PEB_LDR_DATA
   +0x020 ProcessParameters : 0x000001cd`c8f922e0 _RTL_USER_PROCESS_PARAMETERS
   +0x028 SubSystemData    : (null) 
   +0x030 ProcessHeap      : 0x000001cd`c8f90000 Void
...
```

![](pwncoolsexy-part2/Untitled%207.png)

너무 많으니 우리가 찾으려는 필드명으로 검색해보죠

```c
?? ((_PEB*) @$peb)->ProcessHeaps // PEB->ProcessHeaps 필드값 출력
```

![](pwncoolsexy-part2/Untitled%208.png)

![](pwncoolsexy-part2/Untitled%209.png)

`ProcessHeaps` 는 `_Heap` 구조체를 가리키는 포인터 배열을 가리킵니다. 포인터 배열에는 Heap 주소들이 나열되어 있네요. 

첫 번째 구조체를 확인해볼게요. `_HEAP` 은  `dt _HEAP [Address]` 로 확인할 수 있습니다.

```text
0:001> dt _HEAP 000001cd`c8f90000
ntdll!_HEAP
   +0x000 Segment          : _HEAP_SEGMENT
   +0x000 Entry            : _HEAP_ENTRY
   +0x010 SegmentSignature : 0xffeeffee
   +0x014 SegmentFlags     : 2
   +0x018 SegmentListEntry : _LIST_ENTRY [ 0x000001cd`c8f90120 - 0x000001cd`c8f90120 ]
...
   +0x07c EncodeFlagMask   : 0x100000
   +0x080 Encoding         : _HEAP_ENTRY
   +0x090 Interceptor      : 0
...
   +0x138 BlocksIndex      : 0x000001cd`c8f902e8 Void
   +0x140 UCRIndex         : (null) 
   +0x148 PseudoTagEntries : (null) 
   +0x150 FreeLists        : _LIST_ENTRY [ 0x000001cd`c8f95e10 - 0x000001cd`c8fab880 ]
...
   +0x198 FrontEndHeap     : 0x000001cd`c8f20000 Void
   +0x1a0 FrontHeapLockCount : 0
   +0x1a2 FrontEndHeapType : 0x2 ''
   +0x1a3 RequestedFrontEndHeapType : 0x2 ''
   +0x1a8 FrontEndHeapUsageData : 0x000001cd`c8f98210  ""
   +0x1b0 FrontEndHeapMaximumIndex : 0x402
   +0x1b2 FrontEndHeapStatusBitmap : [129]  "x"
   +0x238 Counters         : _HEAP_COUNTERS
   +0x2b0 TuningParameters : _HEAP_TUNING_PARAMETERS
```

- 0x7c EncodeFlagMask : Heap chunk header가 인코딩 되었는지를 나타내는 필드. 처음 Heap을 초기화할 때 `0x100000`으로 설정됨
- 0x080 Encoding : Heap header의 변조를 방지하기 위해 XOR 인코딩을 할 때 사용됨
- 0x138 BlocksIndex : `_HEAP_LIST_LOOKUP` 구조체를 가리킴  `_HEAP_LIST_LOOKUP` 구조체는 Back-End에서 Heap chunk들을 관리하기 위해 사용됨
- 0x150 FreeLists : 할당 해제된 Heap Chunk들의 `_HEAP_ENTRY` 구조체를 가리킴
- 0x198 FrontEndHeap : LFH가 활성화된 경우 할당된 `_LFH_HEAP` 구조체를 가리킴
- 0x1a2 FrontEndHeapType : LFH 활성화 여부를 나타내며 `0x0` 이면 비활성화, `0x2`면 활성화된 상태. `HeapQueryInformaton()` 함수를 통해 LFH 활성화 여부를 구별할 때 사용됨
- 0x1a8 FrontEndHeapUsageData : 할당되는 Chunk의 수를 기록하며 일정 수준에 도달하면 Front-End(LFH)를 활성화

`FrontEndHeap`에 유효한 주소 값이 있고 `FrontEndHeapType` 값이 `0x2`인걸로 LFH가 활성화되어 Heap 메모리들을 관리하는 것을 확인할 수 있습니다.

## Back-End Structure

실질적인 Heap의 할당 및 해제를 관리하는 NT Heap의 Back-End 구조체입니다.

### _HEAP_ENTRY

할당된 HEAP Chunk의 header 구조체입니다. 앞서 `_HEAP`의 `Encoding` 필드 설명대로 header의 변조를 방지하기 위해 XOR 인코딩을 하기 때문에 디코딩 과정을 거쳐야 확인할 수 있습니다.

LFH로 관리되는지 아닌지에 따라 XOR 인코딩 방식이 다르며 직접 디코딩해보면서 `_HEAP_ENTRY` 값을 확인해보겠습니다.

![](pwncoolsexy-part2/Untitled%2010.png)

LFH가 활성화되지 않은 인덱스 0의 Chunk header는 `_Heap`의 Encode 필드와 XOR 연산으로 인코딩 됩니다. 이 두 값을 XOR 한 결과로 `chunk+0x10`(16 bytes 단위이므로 0x18을 수정)을 수정하면 아래와 같이 디코딩된 `_HEAP_ENTRY`를 확인할 수 있습니다.

![](pwncoolsexy-part2/Untitled%2011.png)

### FreeLists(_HEAP_ENTRY)

chunk가 해제되면 size에 따라 FreeList에 배치됩니다. 

FreeLists의 구조는 glibc의 unstored bin과 유사하며 해제된 chunk의 `_HEAP_ENTRY` 를 가리킵니다. 

![](pwncoolsexy-part2/Untitled%2012.png)

```text
0:001> dx -r1 (*((ntdll!_LIST_ENTRY *)0x1cdc8f90150))
(*((ntdll!_LIST_ENTRY *)0x1cdc8f90150))                 [Type: _LIST_ENTRY]
    [+0x000] Flink            : 0x1cdc8f95e10 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0x1cdc8fab880 [Type: _LIST_ENTRY *]
0:001> dx -r1 ((ntdll!_LIST_ENTRY *)0x1cdc8fab880)
((ntdll!_LIST_ENTRY *)0x1cdc8fab880)                 : 0x1cdc8fab880 [Type: _LIST_ENTRY *]
    [+0x000] Flink            : 0x1cdc8f90150 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0x1cdc8f94cd0 [Type: _LIST_ENTRY *]
0:001> dx -r1 ((ntdll!_LIST_ENTRY *)0x1cdc8f94cd0)
((ntdll!_LIST_ENTRY *)0x1cdc8f94cd0)                 : 0x1cdc8f94cd0 [Type: _LIST_ENTRY *]
    [+0x000] Flink            : 0x1cdc8fab880 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0x1cdc8f95f40 [Type: _LIST_ENTRY *]
0:001> dx -r1 ((ntdll!_LIST_ENTRY *)0x1cdc8f95f40)
((ntdll!_LIST_ENTRY *)0x1cdc8f95f40)                 : 0x1cdc8f95f40 [Type: _LIST_ENTRY *]
    [+0x000] Flink            : 0x1cdc8f94cd0 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0x1cdc8f951d0 [Type: _LIST_ENTRY *]
0:001> dx -r1 ((ntdll!_LIST_ENTRY *)0x1cdc8f951d0)
((ntdll!_LIST_ENTRY *)0x1cdc8f951d0)                 : 0x1cdc8f951d0 [Type: _LIST_ENTRY *]
    [+0x000] Flink            : 0x1cdc8f95f40 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0x1cdc8fa7b30 [Type: _LIST_ENTRY *]
0:001> dx -r1 ((ntdll!_LIST_ENTRY *)0x1cdc8fa7b30)
((ntdll!_LIST_ENTRY *)0x1cdc8fa7b30)                 : 0x1cdc8fa7b30 [Type: _LIST_ENTRY *]
    [+0x000] Flink            : 0x1cdc8f951d0 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0x1cdc8f93100 [Type: _LIST_ENTRY *]
```

### _HEAP_LIST_LOOKUP

`_HEAP`의 `BlocksIndex`필드에서 참조하는 구조체로 다양한 크기의 해제된 chunk를 관리하는 데 사용되어 Best Fit 정책을 사용하는 NT Heap에서 적절한 chunk를 빠르게 찾을 수 있습니다.

```text
0:001> dt _HEAP_LIST_LOOKUP 0x000001cd`c8f902e8
ntdll!_HEAP_LIST_LOOKUP
   +0x000 ExtendedLookup   : 0x000001cd`c8f9ab70 _HEAP_LIST_LOOKUP
   +0x008 ArraySize        : 0x80
   +0x00c ExtraItem        : 0
   +0x010 ItemCount        : 9
   +0x014 OutOfRangeItems  : 0
   +0x018 BaseIndex        : 0
   +0x020 ListHead         : 0x000001cd`c8f90150 _LIST_ENTRY [ 0x000001cd`c8f95e10 - 0x000001cd`c8fab880 ]
   +0x028 ListsInUseUlong  : 0x000001cd`c8f90320  -> 0x18850
   +0x030 ListHints        : 0x000001cd`c8f90330  -> (null)
```

- 0x000 ExtendedLookup : 다음 `_HEAP_LIST_LOOKUP` 구조체를 가리킴
- 0x008 ArrySize : chunk의 최대 사이즈 >> 4 로, 설정된 실제 최대 사이즈는 0x800
- 0x010 ItemCount : 관리 중인 chunk의 수
- 0x014 OutOfRangeItems : 최대 관리 가능한 사이즈를 넘은 chunk의 수
- 0x018 BaseIndex : `_HEAP_LIST_LOOKUP` 에서 관리되는 chunk의 시작 index, ListHint에서 적절한 크기의 해제된 chunk를 찾을 때도 사용됨. 현재 BaseIndex의 최댓값은 다음 BlocksIndex의 BaseIndex
- 0x020 ListHead : 해제된 chunk list의 Head
- 0x028 ListsInUseUlong : ListHint가 가리키는 chunk들 중 사용 가능한 chunk를 가리킴
- 0x030 ListHints : 적절한 크기의 chunk를 빠르게 찾기 위한 Back-End 할당자의 핵심으로 같은 크기의 해제된 chunk list를 가리키는 주소 배열(각 요소는 0x10 size)

 

## Front-End Structure

Front-End는 Low Fragmentation Heap 과정이며 메모리 할당 및 해제에 직접적으로 관여하지는 않습니다.

### _HEAP_USERDATA_HEADER

UserBlock의 header 역할을 하는 구조체이며 XOR 연산으로 인코딩 되어 있습니다. LFH chunk는  `_HEAP_ENTRY` 의 **`SubSegmentCode` 필드에 인코딩 결괏값을 저장합니다. 

![](pwncoolsexy-part2/Untitled%2013.png)

```c
(SubSegmentCode ^ pLFHKey ^ &_HEAP ^ (&_HEAP_ENTRY >> 4)) << 0xc
```

의 결괏값이 chunk 주소로부터의 `_HEAP_USERDATA_HEADER` 오프셋으로, 아래와 같이 확인할 수 있습니다.

![](pwncoolsexy-part2/Untitled%2014.png)

- 0x000 SubSegment : UserBlock의 SubSegment 주소를 가리킴
- 0x018 EncodedOffsets : chunk header의 무결성 검증에 사용됨
- 0x020 BusyBitmap : chunk가 사용 중인지를 나타냄

### _LFH_HEAP

LFH chunk를 관리하기 위해 사용되는 구조체이며 `_HEAP` 의 `FrontEndHeap` 필드에서 참조합니다.

```text
0:001> dt _LFH_HEAP 0x000001cd`c8f20000
ntdll!_LFH_HEAP
   +0x000 Lock             : _RTL_SRWLOCK
   +0x008 SubSegmentZones  : _LIST_ENTRY [ 0x000001cd`c8fa30d0 - 0x000001cd`c8fa30d0 ]
   +0x018 Heap             : 0x000001cd`c8f90000 Void
   +0x020 NextSegmentInfoArrayAddress : 0x000001cd`c8f21140 Void
   +0x028 FirstUncommittedAddress : 0x000001cd`c8f22000 Void
   +0x030 ReservedAddressLimit : 0x000001cd`c8f3a000 Void
   +0x038 SegmentCreate    : 7
   +0x03c SegmentDelete    : 0
   +0x040 MinimumCacheDepth : 0
   +0x044 CacheShiftThreshold : 0
   +0x048 SizeInCache      : 0
   +0x050 RunInfo          : _HEAP_BUCKET_RUN_INFO
   +0x060 UserBlockCache   : [12] _USER_MEMORY_CACHE_ENTRY
   +0x2a0 MemoryPolicies   : _HEAP_LFH_MEM_POLICIES
   +0x2a4 Buckets          : [129] _HEAP_BUCKET
   +0x4a8 SegmentInfoArrays : [129] (null) 
   +0x8b0 AffinitizedInfoArrays : [129] (null) 
   +0xcb8 SegmentAllocator : (null) 
   +0xcc0 LocalData        : [1] _HEAP_LOCAL_DATA
```

- 0x018 Heap : 현재 _LFH_HEAP 참조하는 _HEAP 구조체의 시작을 가리킴
- 0x2a4 Buckets : 할당 요청된 chunk 크기와 일치하는 메모리 영역 배열
- 0x4a8 SegmentInfoArrays : `_HEAP_LOCAL_SEGMENT_INFO` 구조체 배열을 가리키며 chunk를 크기별로 각각의 SubSegment로 관리하기 위해 사용됨

    ```text
    0:001> dx -r1 (*((ntdll!_HEAP_LOCAL_SEGMENT_INFO * (*)[129])0x1cdc8f204a8))
    (*((ntdll!_HEAP_LOCAL_SEGMENT_INFO * (*)[129])0x1cdc8f204a8))                 [Type: _HEAP_LOCAL_SEGMENT_INFO * [129]]
        [0]              : 0x0 [Type: _HEAP_LOCAL_SEGMENT_INFO *]
        [1]              : 0x0 [Type: _HEAP_LOCAL_SEGMENT_INFO *]
        [2]              : 0x1cdc8f20e40 [Type: _HEAP_LOCAL_SEGMENT_INFO *]
        [3]              : 0x1cdc8f20f00 [Type: _HEAP_LOCAL_SEGMENT_INFO *]
        [4]              : 0x1cdc8f20fc0 [Type: _HEAP_LOCAL_SEGMENT_INFO *]
        [5]              : 0x1cdc8f21080 [Type: _HEAP_LOCAL_SEGMENT_INFO *]
        [6]              : 0x0 [Type: _HEAP_LOCAL_SEGMENT_INFO *]
    ...
    ```

- 0xcc0 LocalData : _HEAP_LOCAL_DATA를 가리키며 LFH의 주소를 검색하는 데 사용됨

### _HEAP_BUCKET

LFH가 chunk를 할당할 때 참조하며 Bucket을 관리하기 위해 사용됩니다. `_LFH_HEAP`의 Buckets 필드에서 참조합니다.

```text
0:001> dt _HEAP_BUCKET 0x1cdc8f202a4
ntdll!_HEAP_BUCKET
   +0x000 BlockUnits       : 1
   +0x002 SizeIndex        : 0 ''
   +0x003 UseAffinity      : 0y0
   +0x003 DebugFlags       : 0y00
   +0x003 Flags            : 0 ''
```

- 0x000 BlockUnits : Bucket이 가리키는 chunk의 크기 >> 4
- 0x002 SizeIndex : 해당 Bucket의 index값. `_LFH_HEAP`의 `SegmentInfoArrays` 배열의 index 값으로도 사용됨

### _HEAP_LOCAL_SEGMENT_INFO

SubSegment를 관리하는 구조체이며 `_LFH_HEAP` 의 `SegmentInfoArrays` 필드에서 참조합니다.

```text
0:001> dx -r1 ((ntdll!_HEAP_LOCAL_SEGMENT_INFO *)0x1cdc8f20e40)
((ntdll!_HEAP_LOCAL_SEGMENT_INFO *)0x1cdc8f20e40)                 : 0x1cdc8f20e40 [Type: _HEAP_LOCAL_SEGMENT_INFO *]
    [+0x000] LocalData        : 0x1cdc8f20cc0 [Type: _HEAP_LOCAL_DATA *]
    [+0x008] ActiveSubsegment : 0x1cdc8fa3170 [Type: _HEAP_SUBSEGMENT *]
    [+0x010] CachedItems      [Type: _HEAP_SUBSEGMENT * [16]]
    [+0x090] SListHeader      [Type: _SLIST_HEADER]
    [+0x0a0] Counters         [Type: _HEAP_BUCKET_COUNTERS]
    [+0x0a8] LastOpSequence   : 0x3 [Type: unsigned long]
    [+0x0ac] BucketIndex      : 0x2 [Type: unsigned short]
    [+0x0ae] LastUsed         : 0x0 [Type: unsigned short]
    [+0x0b0] NoThrashCount    : 0x0 [Type: unsigned short]
```

### _HEAP_SUBSEGMENT

LFH로 할당한 chunk들을 크기별로 관리할 때 사용하는 구조체입니다. `_HEAP_LOCAL_SEGMENT_INFO`의 `ActiveSubsegment` 필드와 `Cacheditems` 필드에서 참조합니다.

```text
0:001> dx -r1 ((ntdll!_HEAP_SUBSEGMENT *)0x1cdc8fa3170)
((ntdll!_HEAP_SUBSEGMENT *)0x1cdc8fa3170)                 : 0x1cdc8fa3170 [Type: _HEAP_SUBSEGMENT *]
    [+0x000] LocalInfo        : 0x1cdc8f20e40 [Type: _HEAP_LOCAL_SEGMENT_INFO *]
    [+0x008] UserBlocks       : 0x1cdc8fa74e0 [Type: _HEAP_USERDATA_HEADER *]
    [+0x010] DelayFreeList    [Type: _SLIST_HEADER]
    [+0x020] AggregateExchg   [Type: _INTERLOCK_SEQ]
    [+0x024] BlockSize        : 0x3 [Type: unsigned short]
    [+0x026] Flags            : 0x0 [Type: unsigned short]
    [+0x028] BlockCount       : 0x13 [Type: unsigned short]
    [+0x02a] SizeIndex        : 0x2 [Type: unsigned char]
    [+0x02b] AffinityIndex    : 0x0 [Type: unsigned char]
    [+0x024] Alignment        [Type: unsigned long [2]]
    [+0x02c] Lock             : 0x7 [Type: unsigned long]
    [+0x030] SFreeListEntry   [Type: _SINGLE_LIST_ENTRY]
0:001> dx -r1 (*((ntdll!_HEAP_SUBSEGMENT * (*)[16])0x1cdc8f20e50))
(*((ntdll!_HEAP_SUBSEGMENT * (*)[16])0x1cdc8f20e50))                 [Type: _HEAP_SUBSEGMENT * [16]]
    [0]              : 0x0 [Type: _HEAP_SUBSEGMENT *]
    [1]              : 0x0 [Type: _HEAP_SUBSEGMENT *]
    [2]              : 0x0 [Type: _HEAP_SUBSEGMENT *]
    [3]              : 0x0 [Type: _HEAP_SUBSEGMENT *]
    [4]              : 0x0 [Type: _HEAP_SUBSEGMENT *]
    [5]              : 0x0 [Type: _HEAP_SUBSEGMENT *]
    [6]              : 0x0 [Type: _HEAP_SUBSEGMENT *]
...
```

- 0x000 LocalInfo : 현재 SubSegment의 `_HEAP_LOCAL_SEGMENT_INFO` 를 가리킴
- 0x008 UserBlock : 현재 SubSegment의 UserBlock(`_HEAP_USERDATA_HEADER`)를 가리킴
- 0x020 AggregateExchg : `_INTERLOCK_SEQ` 구조체를 참조하며 UserBlock의 해제된 chunk 개수를 나타냄. LFH는 이를 사용하여 Userblock에서 할당해야 하는지 결정함
- 0x024 BlockSize : UserBlock의 각 chunk 크기 >> 4
- 0x028 BlockCount : UserBlock에 할당된 cunk의 개수
- 0x02a SizeIndex : UserBlock에 해당하는 SizeIndex(`_HEAP_LOCAL_SEGMENT_INFO`의 `BucketIndex`와 같은 값)

### _INTERLOCK_SEQ

할당, 해제된 chunk의 개수를 구할 때 참조하는 구조체입니다. `_HEAP_SUBSEGMENT`의 `AggregateExchg` 필드에서  참조합니다.

```text
0:001> dx -r1 (*((ntdll!_INTERLOCK_SEQ *)0x1cdc8fa3190))
(*((ntdll!_INTERLOCK_SEQ *)0x1cdc8fa3190))                 [Type: _INTERLOCK_SEQ]
    [+0x000] Depth            : 0x7 [Type: unsigned short]
    [+0x002 (14: 0)] Hint     : 0x2 [Type: unsigned short]
    [+0x002 (15:15)] Lock     : 0x0 [Type: unsigned short]
    [+0x002] Hint16           : 0x2 [Type: unsigned short]
    [+0x000] Exchg            : 131079 [Type: long]
```

- 0x000 Depth : UserBlock에 남아있는 해제 된 청크 개수



네 이번 글은 여기서 끝입니다. 뭔가 마무리가 찝찝한 것 같죠..? 생각도 못한 구조체 개수로 이번에도 역시나 분량조절에 실패해버렸.. ㅎㅎ;  

![](pwncoolsexy-part2/Untitled%2015.jpg)

NT Heap의 자세한 memory allocate process는 다음 글에서 소개할 예정이며 폰쿨섹시 시리즈는 다음 글로 돌아오겠습니다!



> P.S : 아 그리고 이건 몰래 말씀드리는 건데(속닥) 곧 특별한 게시글이 올라올 예정이니(속닥) 기대해주세요!



# Reference

[https://www.slideshare.net/AngelBoy1/windows-10-nt-heap-exploitation-english-version](https://www.slideshare.net/AngelBoy1/windows-10-nt-heap-exploitation-english-version)

[https://null2root.github.io/blog/2020/02/07/LazyFragmentationHeap-WCTF2019-writeup.html](https://null2root.github.io/blog/2020/02/07/LazyFragmentationHeap-WCTF2019-writeup.html)

[https://blog.rapid7.com/2019/06/12/heap-overflow-exploitation-on-windows-10-explained/?fbclid=IwAR0RI5JuJ7gdFsA_Twju0tW2IdwPUFNppmUcyu7dz_wuqeR3Lq3lWUa8q8U](https://blog.rapid7.com/2019/06/12/heap-overflow-exploitation-on-windows-10-explained/?fbclid=IwAR0RI5JuJ7gdFsA_Twju0tW2IdwPUFNppmUcyu7dz_wuqeR3Lq3lWUa8q8U)