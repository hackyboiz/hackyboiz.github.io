---
title: "[Research] Hip하게 Heap 정복하기 Part 2 - Segment Heap(2)"
author: L0ch
tags: [L0ch, research, windows, segment heap, heap, windows 10]
categories: [Research]
date: 2021-10-10 18:00:00
cc: true
index_img: 2021/10/10/l0ch/segment-heap-part2/thumbnail.jpg
---

# 시리즈 바로가기
[Hip하게 Heap 정복하기 Part 1 - Segment Heap(1)](https://hackyboiz.github.io/2021/09/19/l0ch/segment-heap-part1/)
Hip하게 Heap 정복하기 Part 2 - Segment Heap(2) ← Now! 
[Hip하게 Heap 정복하기 Part 3 - HITCON 2020 Michael's Storage ](https://hackyboiz.github.io/2021/11/21/l0ch/segment-heap-part3/)
[Hip하게 Heap 정복하기 Part 4 - HITCON 2020 Michael's Storage(2)](https://hackyboiz.github.io/2021/12/19/l0ch/segment-heap-part4/)
[Hip하게 Heap 정복하기 Part 5 - HITCON 2020 Michael's Storage(3)](https://hackyboiz.github.io/2022/01/09/l0ch/segment-heap-part5/)



안녕하세요, 백신 2차 맞고 온 L0ch입니다...ㅠㅠㅠ 

![Untitled](segment-heap-part2/Untitled.png)

저번 Part 1올릴 때도 똑같은 이슈로 일주일 밀렸던 것 같은데, 타이밍이 또또 겹치게 됐네요.. ~~살려줘~~

아무튼 3일 전에 백신을 맞고 첫날 이튿날까지 거의 24시간을 내리 자다가 왼쪽 팔 부여잡으며 쓴 글입니다 ㅎㅎ; 

![Untitled](segment-heap-part2/Untitled%201.png)

> 1차 아프면 2차는 안아프다며!!!! 날 속였어!!!
> 

그나마 백신 맞기 전에 거의 다 작성해놔서 펑크는 안 냈네요 휴~!  (또 펑크 냈으면 앞으로 한 달 동안 제 글만 올라갔을 뻔했지 뭐예요.. → 죽음을 의미)

이전 파트에서 힙 생성에 대해 알아봤으니 바로 블록 할당부터 시작해보도록 하죠!



# RtlpHpAllocateHeap()

`HeapAlloc()` 또는 `RtlAllocateHeap()`을 통해 Heap Block을 할당할 때 Segment Heap에 의해 관리되는 경우 할당 요청은 `RtlpHpAllocateHeap()` 으로 라우팅 되며 `RtlpHpAllocateHeap()` 의 함수 원형은 다음과 같습니다.

```c
PVOID RtlpHpAllocateHeap(_SEGMENT_HEAP* HeapBase, SIZE_T UserSize, ULONG Flags, USHORT Unknown)
```

함수는 AllocSize를 기반으로 적절한 Segment Heap의 할당 기능을 호출합니다. (기본적으로 AllocSize는 함수의 매개변수로 전달되는 `UserSize`이며 `UserSize`가 0인 경우 1이 됩니다.)

크기에 따른 할당 메커니즘은 [Segment Heap 개요](https://hackyboiz.github.io/2021/09/19/l0ch/segment-heap-part1/#%EA%B0%9C%EC%9A%94-Segment-Heap) 에서 그림과 같이 간단하게 설명했었죠? 다시 나열해봅시다.

- Backend
- Variable Size Allocation(VS)
- Low Fragmentation Heap(LFH)
- Large Blocks Allocation

이 중 이번 파트에서는 Backend와 VS를, 다음 파트에서는 LFH와 Large Block에 대해 알아볼 건데요, 백엔드부터 바로 넘어가 봅시다!



# Backend Allocation

백엔드는 131,073 (0x20001, 128kb) 부터 520,192 (0x7F000, 508kb) 까지의 할당 요청을 처리합니다.

백엔드 블록에는 헤더가 없고  페이지 크기 단위가 존재합니다. 또한 VS/LFH의 subsegment는 백엔드 블록의 특수 타입이기 때문에 VS와 LFH 블록이 할당될 때에도 백엔드가 사용됩니다.



## Segment Structure

백엔드는 `NtAllocateVirtualMemory()` 를 통해 할당된 가상 메모리의 1MB(0x100000) 블록인 Segment 구조체에서 관리되며 이렇게 생성된 Segment는  `_SEGMENT_HEAP` → `SegContexts` → `SegmentListHead` 필드에서 참조합니다.

![Untitled](segment-heap-part2/Untitled%202.png)

 

세그먼트의 첫 0x2000 bytes 가 세그먼트 헤더가 되고 나머지는 백엔드 블록 할당에 사용되며 0x2000 bytes를 제외한 나머지는 예약 상태로 존재하다가 필요에 따라 사용됩니다.

세그먼트 헤더에는 세그먼트의 각 페이지 상태를 설명하는 256 bytes 크기의 `_HEAP_PAGE_RANGE_DESCRIPTOR` 구조체 배열이 존재합니다. 세그먼트의 데이터 부분이 오프셋 0x2000에서 시작하기 때문에 첫 번째 `_HEAP_PAGE_RANGE_DESCRIPTOR` 위치에는 `_HEAP_PAGE_SEGMENT` 구조체가 대신 저장되고 두 번째 `_HEAP_PAGE_RANGE_DESCRIPTOR`는 사용되지 않는다고 하네요..? 이 부분에 대해서는 명확한 이유를 찾을 수는 없었지만 뇌피셜로 `_HEAP_PAGE_SEGMENT`의 크기가 20 bytes를 넘기 때문에 정렬을 위해 남겨둔 공간이지 싶습니다.

![Untitled](segment-heap-part2/Untitled%203.png)

그림으로 보면 Segment의 처음 부분은 ` _HEAP_PAGE_SEGMENT`와 unused로 채워져 있고 0x40부터 `_HEAP_PAGE_RANGE_DESCRIPTOR` 배열로 사용되는 것을 볼 수 있네요. 이후 0x2000 이후 공간이 실제 페이지로 사용됩니다.

`_HEAP_PAGE_SEGMENT` 와 `_HEAP_PAGE_RANGE_DESCRIPTOR` 을 좀 더 살펴보겠습니다!



## _HEAP_PAGE_SEGMENT

`_HEAP_PAGE_SEGMENT` 구조체의 필드는 다음과 같습니다.

```c
0:021> dt ntdll!_HEAP_PAGE_SEGMENT
   +0x000 ListEntry        : _LIST_ENTRY
   +0x010 Signature        : Uint8B
   +0x018 SegmentCommitState : Ptr64 _HEAP_SEGMENT_MGR_COMMIT_STATE
   +0x020 UnusedWatermark  : UChar
   +0x000 DescArray        : [256] _HEAP_PAGE_RANGE_DESCRIPTOR
```

- `ListEntry` : 각 세그먼트는 `SegmentListHead`의 Linked List 노드
- `Signature` : 주소가 세그먼트의 일부인지 확인하는 데 사용됨
    - `(세그먼트 주소 >> 0x14) ^ RtlpHeapKey ^ HeapBase(_SEGMENT_HEAP) ^ 0xA2E64EAD2E64EAD` 로 계산됨



## _HEAP_PAGE_RANGE_DESCRIPTOR

 `_HEAP_PAGE_RANGE_DESCRIPTOR` 는 각 페이지 상태를 나타내는 필드들이 존재합니다. 백엔드 블록은 여러 페이지를 사용할 수 있어 백엔드 블록의 첫 번째 페이지의 `_HEAP_PAGE_RANGE_DESCRIPTOR`에도 이를 알릴 수 있는 추가적인 필드가 필요합니다.

```c
0:021> dt ntdll!_HEAP_PAGE_RANGE_DESCRIPTOR
   +0x000 TreeNode         : _RTL_BALANCED_NODE
   +0x000 TreeSignature    : Uint4B
   +0x004 UnusedBytes      : Uint4B
   +0x008 ExtraPresent     : Pos 0, 1 Bit
   +0x008 Spare0           : Pos 1, 15 Bits
   +0x018 RangeFlags       : UChar
   +0x019 CommittedPageCount : UChar
   +0x01a Spare            : Uint2B
   +0x01c Key              : _HEAP_DESCRIPTOR_KEY
			+0x000 Key              : Uint4B
      +0x000 EncodedCommittedPageCount : Pos 0, 16 Bits
      +0x000 LargePageCost    : Pos 16, 8 Bits
      +0x000 UnitCount        : Pos 24, 8 Bits
   +0x01c Align            : [3] UChar
   +0x01f Offset       : UChar
   +0x01f Size         : UChar
```

- `TreeNode` : free 된 백엔드 블록의 첫 번째 페이지임을 나타냄
    - `FreePageRanges`의 노드
- `UnusedBytes` : 첫 번째 페이지인 경우 `UserSize`와 할당된 블록 크기의 사이즈 차이, 즉 alignment로 인해 할당되었지만 사용되지 않는 크기를 나타냄
- `RangeFlags` : 백엔드 블록의 타입과 페이지 state를 나타내는 비트 필드
    - `0x01`: `PAGE_RANGE_FLAGS_LFH_SUBSEGMENT` 첫 번째 페이지인 경우 LFH 하위 세그먼트임을 나타냄
    - `0x02`: `PAGE_RANGE_FLAGS_COMMITED` 페이지가 커밋됨
    - `0x04`: `PAGE_RANGE_FLAGS_ALLOCATED` 페이지가 할당되거나 사용 중임
    - `0x08`: `PAGE_RANGE_FLAGS_FIRST` 첫 번째 페이지임을 표시
    - `0x20`: `PAGE_RANGE_FLAGS_VS_SUBSEGMENT` 첫 번째 페이지인 경우 VS 하위 세그먼트임을 나타냄
- `Key` : free 된 백엔드 블록의 첫 번째 페이지에만 있으며 backend free tree에 free 된 블록을 삽입할 때 사용함
    - `Key` - backend free tree에 사용되는 WORD 크기의 키. 상위 bytes는 `PageCount` 필드/하위 bytes는 `EncodedCommitCount` 필드
    - `EncodedCommitCount` - 백엔드 블록의 커밋된 페이지 수에 대한 Bitwise NOT
    - `UnitCount` : 백엔드 블록의 페이지 수
- `Offset` : 첫 번째 페이지가 아닌 경우 첫 번째  `_HEAP_PAGE_RANGE_DESCRIPTOR` 에서부터의 오프셋
- `Size` : 첫 번째 페이지인 경우 `Key.PageCount` 와 동일한 값

아래 그림은 128kb(0x20100) 백엔드 할당했을 때의 Segment Heap의 모습입니다.

![Untitled](segment-heap-part2/Untitled%204.png)

`0x00`에는 `_HEAP_PAGE_SEGMENT`가, `0x01`은 unused고 `0x02`의 `_HEAP_PAGE_RANGE_DESCRIPTOR`의  `RagneFlags`에 First 비트가 설정되어 있습니다. 

Page의 단위가 0x1000이므로 0x20100 할당을 위해서는 #0x02부터 #0x22까지 21개의 페이지를 할당해줘야 하겠네요. 커밋된 페이지 전체 크기는 0x21000이므로 할당 요청을 받은 0x20100을 빼면 `UnusedBytes : 0xF00`이 나옵니다. 

첫 번째 `_HEAP_PAGE_RANGE_DESCRIPTOR` 외의 `RanageFlags`에는 오프셋 정보가 있는 것을 확인할 수 있습니다.



# VARIABLE SIZE ALLOCATION

가변 크기(Variable Size, VS) 할당은 크기가 1~131,072(0x20000) bytes인 할당에 사용됩니다. VS 블록은 16 byte 단위를 가지며 각각 시작 부분에 블록 헤더가 있습니다. 

## VS Subsegments

VS 할당 과정에서는 VS subsegment를 생성하기 위해 백엔드에 의존합니다. VS subsegment는 첫 번째 `_HEAP_PAGE_RANGE_DESCRIPTOR` 의 `RangeFlags`에 `PAGE_RANGE_FLAGS_VS_SUBSEGMENT(0x20)` 비트가 설정된 특수한 타입의 백엔드 블록입니다.

![Untitled](segment-heap-part2/Untitled%205.png)

그림을 보면 VS subsegment는 백엔드 블록 자리를 차지하는 것을 볼 수 있습니다. VS 할당을 위해서는 백엔드 할당이 먼저 필요하다는 것을 확인할 수 있네요.



## _HEAP_VS_CONTEXT

`_HEAP_VS_CONTEXT` 구조체는 `_SEGMENT_HEAP`의 `VsContext` 필드에서 참조하며 사용 가능한 VS 블록, VS subsegment 및 VS 할당 state와 관련된 기타 정보에 대한 필드가 존재합니다. 

```c
0:017> dt ntdll!_HEAP_VS_CONTEXT
   +0x000 Lock             : Uint8B
   +0x008 LockType         : _RTLP_HP_LOCK_TYPE
   +0x010 FreeChunkTree    : _RTL_RB_TREE
   +0x020 SubsegmentList   : _LIST_ENTRY
   +0x030 TotalCommittedUnits : Uint8B
   +0x038 FreeCommittedUnits : Uint8B
   +0x040 DelayFreeContext : _HEAP_VS_DELAY_FREE_CONTEXT
   +0x080 BackendCtx       : Ptr64 Void
   +0x088 Callbacks        : _HEAP_SUBALLOCATOR_CALLBACKS
   +0x0b0 Config           : _RTL_HP_VS_CONFIG
   +0x0b4 Flags            : Uint4B
```

- `FreeChunkTree` : free 된 VS 블록의 RB tree
- `SubsegmentList` : 모든 VS subsegement의 linked list
- `BackendCt` : 상위 `_SEGMENT_HEAP` 에 대한 포인터
- `Callbacks` : VS subsegment 관리에 사용되는 인코딩된 콜백



## _HEAP_VS_SUBSEGMENT

VS subsegment는 VS 블록이 할당되는 위치입니다. VS subsegment는 `RtlpHpVsSubsegmentCreate()`를 통해 할당 및 초기화되며  `_HEAP_VS_SUBSEGMENT` 헤더를 갖습니다.

```c
0:014> dt ntdll!_HEAP_VS_SUBSEGMENT
   +0x000 ListEntry        : _LIST_ENTRY
   +0x010 CommitBitmap     : Uint8B
   +0x018 CommitLock       : Uint8B
   +0x020 Size             : Uint2B
   +0x022 Signature        : Pos 0, 15 Bits
   +0x022 FullCommit       : Pos 15, 1 Bit
```

- `ListEntry` : 각 VS subsegment는 VS subsegment Linked List의 노드입니다.
- `CommitBitmap` : VS subsegment 페이지의 커밋 비트맵
- `Size` : 16bytes 블록의 VS 하위 세그먼트 크기
- `Signature` : VS subsegemnt가 손상되었는지 확인하는 데 사용(`Size  ^ 0xABED`)



## _HEAP_VS_CHUNK_HEADER

사용 중인 VS 블록에는 16 bytes 크기의 헤더가 있습니다.

```c
0:014> dt ntdll!_HEAP_VS_CHUNK_HEADER -r
   +0x000 Sizes            : _HEAP_VS_CHUNK_HEADER_SIZE
      +0x000 MemoryCost       : Pos 0, 16 Bits
      +0x000 UnsafeSize       : Pos 16, 16 Bits
      +0x004 UnsafePrevSize   : Pos 0, 16 Bits
      +0x004 Allocated        : Pos 16, 8 Bits
      +0x000 KeyUShort        : Uint2B
      +0x000 KeyULong         : Uint4B
      +0x000 HeaderBits       : Uint8B
   +0x008 EncodedSegmentPageOffset : Pos 0, 8 Bits
   +0x008 UnusedBytes      : Pos 8, 1 Bit
   +0x008 SkipDuringWalk   : Pos 9, 1 Bit
   +0x008 Spare            : Pos 10, 22 Bits
   +0x008 AllocatedChunkBits : Uint4B
```

- `Size` : 크기 및 상태 정보를 캡슐화하는 인코딩 된 QWORD 크기의 하위 구조체
- `EncodedSegmentPageOffset` : 페이지의 VS subsegment 시작부터 블록까지의 오프셋(인코딩 됨)
- `UnusedBytes` :  사용되지 않은 크기 플래그

## _HEAP_VS_CHUNK_FREE_HEADER

free 된 VS 블록에는 32 bytes (0x20) 헤더가 있으며 처음 8 bytes는 앞서 본`_HEAP_VS_CHUNK_HEADER` 구조체와 동일합니다. 오프셋 0x08에서 시작하는 노드 필드는 `VsContext.FreeChunkTree` 에서 free 된 VS 블록 노드 역할을 합니다.

```c
0:014> dt ntdll!_HEAP_VS_CHUNK_FREE_HEADER -r
   +0x000 Header           : _HEAP_VS_CHUNK_HEADER
      +0x000 Sizes            : _HEAP_VS_CHUNK_HEADER_SIZE
         +0x000 MemoryCost       : Pos 0, 16 Bits
         +0x000 UnsafeSize       : Pos 16, 16 Bits
         +0x004 UnsafePrevSize   : Pos 0, 16 Bits
         +0x004 Allocated        : Pos 16, 8 Bits
         +0x000 KeyUShort        : Uint2B
         +0x000 KeyULong         : Uint4B
         +0x000 HeaderBits       : Uint8B
      +0x008 EncodedSegmentPageOffset : Pos 0, 8 Bits
      +0x008 UnusedBytes      : Pos 8, 1 Bit
      +0x008 SkipDuringWalk   : Pos 9, 1 Bit
      +0x008 Spare            : Pos 10, 22 Bits
      +0x008 AllocatedChunkBits : Uint4B
   +0x000 OverlapsHeader   : Uint8B
   +0x008 Node             : _RTL_BALANCED_NODE
...
```

정리할게 너무 많아서 벌써부터 어질어질하네요.. 아 백신 때문인가? 그래도 문제를 풀기 위해서는 꼭 알고 넘어가야 하는 부분이라 열심히 정리했습니다 ㅠㅠ

다음 파트에서는 LFH와 Large Block까지 살펴보고 여유가 된다면 CTF에 출제된 Segment Heap 문제를 슬쩍 스포 해보도록 할게요!

# Reference

[https://www.blackhat.com/docs/us-16/materials/us-16-Yason-Windows-10-Segment-Heap-Internals-wp.pdf](https://www.blackhat.com/docs/us-16/materials/us-16-Yason-Windows-10-Segment-Heap-Internals-wp.pdf)

[https://www.blackhat.com/docs/us-16/materials/us-16-Yason-Windows-10-Segment-Heap-Internals.pdf](https://www.blackhat.com/docs/us-16/materials/us-16-Yason-Windows-10-Segment-Heap-Internals.pdf)

[https://speakerdeck.com/scwuaptx/windows-kernel-heap-segment-heap-in-windows-kernel-part-1?slide=207](https://speakerdeck.com/scwuaptx/windows-kernel-heap-segment-heap-in-windows-kernel-part-1?slide=207)

