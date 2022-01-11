---
title: "[Research] Hip하게 Heap 정복하기 Part 1 - Segment Heap(1)"
author: L0ch
tags: [L0ch, research, windows, segment heap, heap, windows 10]
categories: [Research]
date: 2021-09-19 18:00:00
cc: true
index_img: 2021/09/19/l0ch/segment-heap-part1/thumnail.png
---

# 시리즈 바로가기
Hip하게 Heap 정복하기 Part 1 - Segment Heap(1)  ← Now!
[Hip하게 Heap 정복하기 Part 2 - Segment Heap(2)](https://hackyboiz.github.io/2021/10/10/l0ch/segment-heap-part2/)
[Hip하게 Heap 정복하기 Part 3 - HITCON 2020 Michael's Storage ](https://hackyboiz.github.io/2021/11/21/l0ch/segment-heap-part3/)
[Hip하게 Heap 정복하기 Part 4 - HITCON 2020 Michael's Storage(2)](https://hackyboiz.github.io/2021/12/19/l0ch/segment-heap-part4/)
[Hip하게 Heap 정복하기 Part 5 - HITCON 2020 Michael's Storage(3)](https://hackyboiz.github.io/2022/01/09/l0ch/segment-heap-part5/)



안녕하세요! 드라이버 익스플로잇 이후로 오랜만에 연구글로 돌아온 L0ch입니다. 

사실 저번주에 올라와야 했었던 글이지만.. 코로나 백신을 맞고 이틀이 삭제돼서 미루게 됐습니다...

![Untitled](segment-heap-part1/Untitled.png)

> 팔이.. 팔이 움직이질 않아..!

모두 코로나 조심하세요ㅜㅜ

이번 시리즈에서는 Windows의 Segment Heap에 대해 알아보고, HITCON 2020에 출제된 Segment Heap Challenge 문제를 풀어보려고 합니다! 저번에 NT Heap 공부하면서 문제푸느라 고생했던거 벌써 까먹고 또또 고생을 사서하네요.. 이번에도 어떻게든 되겠죠!

# 개요 - Segment Heap

Segment Heap은 Windows 10부터 도입된 새로운 memory allocator입니다. Windows의 NT Kernel에서 사용된 기존 memory allocator인 NT Heap에 대해서는 이전 시리즈 [Pwn하고 Cool하고 Sexy한 Windows 탐방기](https://hackyboiz.github.io/2021/01/31/l0ch/pwncoolsexy-part1/) 파트 2부터 설명되어 있으며 Segment Heap에 대한 언급도 살짝 나오니 NT Heap과의 차이점이 궁금하시다면 같이 보면 도움이 됩니다!

Segment Heap의 아키텍처는 아래와 같습니다.

![Untitled](segment-heap-part1/Untitled%201.png)

- 128KB 보다 작거나 같은 크기이며 LFH에 의해 처리되지 않는 경우 가변 크기(variable size, VS) 할당자가 사용됩니다.
- 128KB보다 크고 508KB보다 작은 할당 요청은 백엔드에서 직첩 처리됩니다.
- 508KB보다 큰 Large Block 할당 요청은 기본적인 64KB 할당 단위를 사용해 VirtualAlloc을 직접 호출해 처리됩니다.
- 16,368 bytes보다 작거나 같은 경우 LFH allocator가 사용됩니다.
    - VS와 LFH 모두 Heap SubSegment를 생성하기 위해 백엔드를 사용

NT Heap과의 차이는 다음과 같습니다.

- Segment Heap은 메타데이터가 더 작은 메모리를 사용해 모바일 장치에 적합합니다.
- 두 Heap 모두 LFH 할당을 지원하지만 내부 구현은 완전히 다르며 Segment Heap이 메모리 사용량과 성능면에서 더 효율적입니다.
- Segment Heap은 유저 제공 메모리 맵 파일과는 함께 사용할 수 없고, 대신 NT Heap이 생성됩니다.
- Segment Heap의 메타데이터는 실제 데이터와 구분되어 특정 블록에 대한 메타데이터를 찾기 어렵게 만들어 비교적 안전합니다.

Segment Heap은 또한 메모리 커럽션 취약점에 의한 코드 실행에 대비해 여러 보안 매커니즘을 추가했습니다.

- Segment Heap은 세그먼트와 서브세그먼트를 추적하기 위해 Linked List를, 레드블랙(RB) 트리로 백엔드와 VS 할당을 추적합니다. NT Heap과 유사한 방식으로 노드 삽입 및 삭제 작업 전에 검사를 하지만, 손상된 노드가 감지되면`RTLFailFast` 를 호출하는 빠른 실패 메커니즘을 사용해 프로세스를 즉시 종료합니다.
- Segment Heap의 일부는 콜백을 허용하나 함수포인터를 내부 난수화된 힙의 키와 컨텍스트 주소로 XOR 인코딩해 콜백을 overwrite하기 어렵도록 합니다.
- LFH와 VS 서브세그먼트, Large Block이 할당되면 가드 페이지가 추가됩니다. 가드 페이지는 인접한 데이터 손상을 탐지할 수 있습니다.

이렇게 보면 Segment Heap이 NT Heap보다 더욱 발전된 Heap이라고 볼 수 있을 것 같습니다. 

![Untitled](segment-heap-part1/Untitled%202.png)

하지만 여전히 레거시가 문제죠... ㅋㅋㅋ 사진처럼 기존 NT Heap에 최적화된 애플리케이션에 영향을 줄 수 있어 도입된지 꽤 되었지만 Windows의 기본 힙 할당자는 여전히 NT Heap입니다. 하지만 시간이 지나면서 점점 Segment Heap을 많이 사용하게 될거고, 기본 힙 할당자가 될 날도 오겠죠?  

현재 UWP(Universal Windows Platform) 앱은 기본적으로 Segment Heap을 사용합니다. UWP뿐만 아니라 `csrss.exe`, `lsass.exe`, `runtimebroker.exe`, `services.exe`, `smss.exe`, `svchost.exe` 등과 같은 특정 시스템 프로세스에서도 사용합니다.

특정 실행파일에 대해 Segment Heap을 활성화/비활성화 하기 위해서는 아래 레지스트리 설정을 하면 됩니다.

```c

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\(executable)
FrontEndHeapDebugOptions = (DWORD)
Bit 2 (0x04): Disable Segment Heap
Bit 3 (0x08): Enable Segment Heap
```

모든 실행 파일에 대해 전역적으로 Segment Heap을 활성화하려면 아래 레지스터 설정을 하면 됩니다.

```c
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Segment Heap
Enabled = (DWORD)
0 : Disable Segment Heap
(Not 0): Enable Segment Heap
```

프로세스에서 Segment Heap이 활성화된 경우에도 NT Heap에서 관리해야 하는 힙이 있어 모든 힙을 Segment Heap으로 관리하는 것은 아닙니다. Windows 10에서 UWP 앱으로 바뀐 계산기를 디버깅하며 Segment Heap에 대해 살펴보겠습니다.

![Untitled](segment-heap-part1/Untitled%203.png)

> 컴퓨터 하는 사람 특) 계산기 키면 기본값 프로그래머임

## Heap Creation

Segment Heap이 활성화된 경우 `HeapCreate()`에 의해 생성된 힙은 힙을 확장할 수 없음을 의미하는 `dwMaximumSize`가 0이 아니면 Segment Heap에 의해 관리됩니다.

`RtlCreateHeap()` API가 힙을 생성하는 데 직접 사용되는 경우 Segment Heap이 생성된 힙을 관리하려면 다음 조건을 만족해야 합니다.

- `RtlCreateHeap()`에 전달되는 `HEAP_GROWABLE`이 설정되어 있어야 합니다(확장 가능한 힙)
- `RtlCreateHeap()`에 전달된 `HeapBase` 인수는 NULL로, 힙 메모리가 미리 할당되지 않은 상태여야 합니다.
- 매개변수 인수가 `RtlCreateHeap()`에 전달되면 `SegmentReserve`, `SegmentCommit`, `VirtualMemoryThreshold` 및 `CommitRoutine`과 같은 매개변수 필드를 0 또는 NULL로 설정해야 합니다.
- `RtlCreateHeap()`에 전달된 Lock 인수는 NULL이어야 합니다.

계산기 프로세스에 디버거를 attach하고 `!heap` 명령어를 실행하면 아래와 같은 결과가 나옵니다.

```c
0:022> !heap
        Heap Address      NT/Segment Heap

         23486a90000         Segment Heap
         23486a20000              NT Heap
         23486be0000         Segment Heap
         234891d0000         Segment Heap
         234899d0000         Segment Heap
         23489e10000              NT Heap
         23489e30000              NT Heap
         23489fe0000              NT Heap
         23489800000              NT Heap
```

첫 번째 Heap은 프로세스 기본 힙으로, Segment Heap으로 생성된 것을 볼 수있습니다. 두 번째 힙은 유저 정의 메모리 블록을 사용하며 이 기능은 Segment Heap에서는 지원되지 않아 NT Heap으로 생성된 것을 볼 수 있죠. UWP 앱에서도 NT Heap에서 관리하는 힙이 존재한다는 것을 알 수 있네요. 첫 번째 기본 힙 정보를 더 보도록 하겠습니다.

## _SEGMENT_HEAP

Segment Heap이 활성화되면 `HeapCreate()` 또는 `RtlCreateHeap()`에 의해 반환된 Heap 주소와 핸들은 NT Heap의 `_HEAP` 구조체 대신 아래처럼 `_SEGMENT_HEAP` 구조체를 가리킵니다. 

```c
0:022> dt ntdll!_SEGMENT_HEAP 23486a90000
   +0x000 EnvHandle        : RTL_HP_ENV_HANDLE
   +0x010 Signature        : 0xddeeddee
   +0x014 GlobalFlags      : 0
   +0x018 Interceptor      : 0
   +0x01c ProcessHeapListIndex : 1
   +0x01e AllocatedFromMetadata : 0y0
   +0x020 CommitLimitData  : _RTL_HEAP_MEMORY_LIMIT_DATA
   +0x020 ReservedMustBeZero1 : 0
   +0x028 UserContext      : (null) 
   +0x030 ReservedMustBeZero2 : 0
   +0x038 Spare            : (null) 
   +0x040 LargeMetadataLock : 0
   +0x048 LargeAllocMetadata : _RTL_RB_TREE
   +0x058 LargeReservedPages : 0
   +0x060 LargeCommittedPages : 0
   +0x068 StackTraceInitVar : _RTL_RUN_ONCE
   +0x080 MemStats         : _HEAP_RUNTIME_MEMORY_STATS
   +0x0d8 GlobalLockCount  : 0
   +0x0dc GlobalLockOwner  : 0
   +0x0e0 ContextExtendLock : 0
   +0x0e8 AllocatedBase    : 0x00000234`86a95300  ""
   +0x0f0 UncommittedBase  : 0x00000234`86a96000  "--- memory read error at address 0x00000234`86a96000 ---"
   +0x0f8 ReservedLimit    : 0x00000234`86aa9000  "--- memory read error at address 0x00000234`86aa9000 ---"
   +0x100 SegContexts      : [2] _HEAP_SEG_CONTEXT
   +0x280 VsContext        : _HEAP_VS_CONTEXT
   +0x340 LfhContext       : _HEAP_LFH_CONTEXT
```

- `Signature` : 두 유형의 Heap을 구분하는 데 사용됨. `0xDDEEDDEE`가 Segment Heap의 Signature

> NT Heap의 signature는 `0xFFEEFFEE`

- `LargeAllocMetadata` - Large Block 메타데이터의 레드블랙(RB) 트리
- `LargeReservedPages` - 모든 Large Block 할당을 위해 예약된 페이지 수
- `LargeCommittedPages` - 모든 Large Block 할당에 대해 커밋된 페이지 수
- `VsContext`, `LfhContext` : 각각 VS와 LFH 할당자 정보를 가지고 있음

Segment Heap의 개요는 이 정도로 정리하고 다음 파트부터는 Segment Heap이 메모리를 할당하는 방식에 대해 좀 더 자세히 알아보겠습니다. 문제까지 풀려면 갈 길이 머네요..! 

