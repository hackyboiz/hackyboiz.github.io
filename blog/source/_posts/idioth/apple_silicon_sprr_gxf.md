---
title: "[Translation] 애플 실리콘 하드웨어의 비밀: SPRR과 Guarded Exception Levels (GXF)"
author: idioth
tags: [idioth, apple, m1, sprr, gxf]
categories: [Translation]
date: 2021-05-16 14:00:00
cc: false
index_img: /2021/05/16/idioth/apple_silicon_sprr_gxf/thumbnail.jpg
---

안녕하세요. idioth입니다. ghidra로 악성코드 분석을 해온다는 녀석이 왜 글은 안 올리고 번역글을 쓰고 있냐고요? 사연이 있습니다. 본래 오늘 해당 글이 올라갔어야 했는데, 분석은 끝냈으나 이번 주 제가 아팠어서 어제 출근을 하였으나 계속되는 기침으로 인해 쫓겨났습니다.

여기서 제가 닉값을 해버렸죠. 크롬 원격 데스크톱을 설치해놓고 절전 설정을 안 건드려놔서 접근이 안됩니다.

![](apple_silicon_sprr_gxf/Untitled.png)

어제 설정하고 지하철 타고 집에 오니 이렇게 되있네요. 아

![](apple_silicon_sprr_gxf/Untitled%201.png)

얼른 집 컴퓨터를 업글을 해서 VM을 집 컴으로 돌리던가 해야겠어요... 요즘 같은 시국에 아프면 집에서 공부를 해야 하니까요! 아무튼 ㅜㅜ... ARM을 공부해볼까 하던 와중에 맥북 M1 칩이 등장하였고 그에 관련된 자료들이 눈에 보이면 잡지 읽는 느낌으로 읽곤 합니다. ~~사람 놀릴만한 버그는 없는지... 동생이 m1 에어를 쓰거든요.~~ ARM 아키텍처로 넘어오면서 스마트폰 등에 적용되어 있던 보호 기법들이 얼마나 들어올까?라는 궁금증이 있었는데, 이번 글이 동작 방식에 대해 어느 정도 의문을 해결해줄 거라 생각합니다!

번역에는 의역과 오역이 존재할 수 있습니다. 틀린 부분이나 이해하기 힘든 부분이 있다면 코멘트 달아주시면 감사드립니다.

> 원문 글 : [Apple Silicon Hardware Secrets: SPRR and Guarded Exception Levels (GXF)](https://blog.svenpeter.dev/posts/m1_sprr_gxf/)

# 소개

1년 전에 [siguza](https://twitter.com/s1guza)는 페이지 테이블의 권한을 재정의하고 커널의 일부를 보호하는 커스텀 ARM extension [Apple APRR에 관한 write-up](https://siguza.github.io/APRR/)을 작성했다. 이후 애플이 출시한 M1 칩은 업데이트된 APRR 뿐만 아니라 부팅 직후 베어 메탈 코드를 쉽게 실행할 수 있다. 새 버전에 대한 루머가 있지만 공식적인 자료는 아직 없다.

이 게시글의 첫 부분은 aarch64의 메모리 관리, 페이지 테이블, 유저/커널 모드에 대해 간략하게 소개하며 이전 Apple SoC와 같은 기능인 APRR도 요악한다. 이미 알고 있는 내용이라면 넘어가도 좋다.

위의 내용을 다루고 나서 SPRR과 GXF가 어떻게 작동하는지 다룬다. SPRR과 GXF가 무엇인지만 알고 싶으면 Asahi Linux 위키를 참고하면 된다.

## MMUs, pagetables, and kernels

ARM CPU는 x86의 ring과 같은 [exception levels](https://developer.arm.com/documentation/102412/0100/Privilege-and-Exception-levels)에서 실행된다. EL0은 애플리케이션이 실행되는 userspace이며 EL1은 커널이 실행되고 EL2는 하이퍼바이저가 실행된다. (펌웨어를 위한 EL3는 M1 칩에는 없다.)

[Virtualization Host Extensions](https://developer.arm.com/documentation/102142/latest/Virtualization-Host-Extensions)이 있는 ARM64 CPU는 EL2를 EL1처럼 만들어 커널을 쉽게 실행할 수 있다.

커널의 작업 중 하나는 유저 랜드에서 실행되는 각 애플리케이션에 종속되어 애플리케이션의 주소 공간을 알리는 것이다. 이는 Memory Management Unit이 수행한다. MMU는 가상 주소를 통해 실제 물리 주소를 관리할 수 있다. 이 매핑의 가장 작은 단위를 4 KiB 크기의 페이지라 하며 페이지에는 가장 주소와 실제 주소가 있다. 애플리케이션이나 커널이 `x` 위치의 메모리에 접근하려 하면 MMU는 페이지 테이블에서 페이지를 확인하고 `y` 주소에서 메모리를 반환한다. 커널은 유저 랜드 애플리케이션의 각 프로세스에 대해 다른 페이지 테이블 세트를 생성하여 별도의 주소 공간을 제공한다.

이 매핑 외에도 각 페이지는 특정 액세스 플래그를 인코딩하는 4 비트가 존재한다. 이 플래그는 페이지의 읽기 쓰기 권한이나 유저 랜드 애플리케이션 혹은 커널이 페이지에서 코드를 실행할 수 있는지를 결정한다. ARMv8-A CPU의 각 페이지 테이블 항목에서 4 비트를 찾을 수 있다.

- *UXN*, Unprivileged Execute never: 이 페이지에서 유저 랜드 (EL0) 코드를 실행할 수 없음
- *PXN*, Prvileged excute never: 이 페이지에서 커널 (EL1) 코드를 실행할 수 없음
- *AP0* and *AP1*:  커널과 유저 랜드에 `rw/--`, `rw/rw`, `r-/--`, `r-/r-`  읽기, 쓰기 접근 권한을 인코딩

마지막 액세스 플래그를 결정하는데 추가적인 복잡함(PAN, 계층 제어)이 있지만 이 게시글에서는 다루지 않는다. 유저 랜드와 커널의 권한이 연관이 있다는 점을 주의해야 한다. 유저랜드는 `rw`이지만 커널에는 `r-`로 페이지를 설정하는 것은 불가능하다.

## APRR

각 페이지에는 EL0/1(유저/커널 모드)에 대한 접근 권한(읽기/쓰기/실행)을 제어하는 4개의 플래그가 있다. APRR은 페이지 테이블 엔트리에 4개의 플래그를 저장하는 것 대신에 별도의 테이블에 대한 인덱스로 변경한다.(액세스 권한을 직접 인코딩하는 대신 4비트 index [AP1][AP0]PXN][UXN]으로 병합된다.) 이 별도의 테이블은 페이지의 실제 권한을 인코딩한다. 게다가 일부 레지스터는 userspace의 이러한 권한을 추가로 제한할 수 있다. 이러한 레지스터들은 커널과 유저 랜드에서 분리되어 있고 페이지 권한을 만들 때 유연성을 제공한다.

보통 모든 개별 엔트리를 수정하려면 많은 page walk가 필요하지만, APRR은 이러한 방식으로 페이지 테이블 권한에 대한 간접적인 계층을 도입하여 단일 레지스터 쓰기로 많은 페이지 권한을 효율적으로 수정할 수 있다.

더 자세한 사항은 [siguza의 write-up](https://siguza.github.io/APRR/)에서 확인할 수 있다.

## Just-In-Time 컴파일러

보통 애플리케이션은 고수준 언어에서 기계 코드로 컴파일된 후 배포된다. 코드는 런타임 중에 수정되지 않아 `r-x`로 매핑할 수 있다.

하지만 JIT 컴파일러는 동적으로 기계 코드를 생성하므로 이를 위해 새 코드를 작성한 후 실행할 수 있도록 메모리 영역을 `rwx`로 매핑해야 한다.

애플은 아이폰에서 CPU가 보증된 명령만 실행하길 원했으므로 이런 매핑을 원하지 않았다. 모든 애플리케이션이 `rxw` 매핑을 요청하면 해당 애플리케이션이 원하는 명령을 실행할 수 있으므로 전체적인 보호 동작이 무의미해진다. 일부 애플리케이션만 이러한 매핑 권한이 있어도 `rxw` 매핑에 shellcode를 작성하여 악용의 대상이 될 수 있다.(`rxw` 영역을 찾아 arbitrary write 후 jump gadget을 찾는 것은 어렵겠지만)

하지만 자바스크립트가 있으므로 애플은 JIT 컴파일러가 필요하다.

이 문제는 APRR로 해결할 수 있다. 특정 유저 랜드 애플리케이션(iOS의 사파리, macOS의 모든 애플리케이션)은 `rw-`와 `r-x`를 빠르게 전환할 수 있는 특수 메모리 영역(`[MAP_JIT` 플래그를 통한 `mmap`, `pthread_jit_write_protect_np`](https://developer.apple.com/documentation/apple-silicon/porting-just-in-time-compilers-to-apple-silicon))을 요청할 수 있다. 여기서 APRR 레지스터 내부의 2 비트를 수정하여 JIT 페이지에 `w` 대신 `x`를 제거하여 모든 페이지를 `rw-`에서 `r-x`로 수정한다.(혹은 그 반대로 동작)

## 페이지 보호 계층(Page Protection Layer)

애플은 가능한 한 모든 실행 가능한 페이지에 code signing을 요구한다. iOS에서는 애플에서 이 서명을 가져와야 하고 macOS에서는 로컬로 생성할 수 있는 ad-hoc(임시) 서명이 있다. Code signing은 보통 커널에 의해 시행된다. 커널은 디바이스 드라이버 같은 코드를 많이 가지고 있으므로 attack surface가 굉장히 크다. 모든 드라이버의 모든 버그는 code signing을 우회할 수 있다. 이러한 이슈는 오래전 비디오 게임 콘솔 [마이크로소프트 Xbox 360 하이퍼바이저](https://free60project.github.io/wiki/Hypervisor/)에서 해결되었다. 모든 커널 코드에 exploitable 한 버그가 존재하는지 확인하는 대신에 하이퍼바이저 자체에 버그가 있는지만 확인하면 된다.

비슷하게 애플은 커널 내부에 매우 낮은 오버헤드를 가진 하이퍼바이저를 생성하기 위해 APRR을 사용한다. 첫 번째로 페이지 테이블(중요한 데이터 구조를 가진 메모리)과 권한 코드 섹션 PPL을 커널 자체에 읽기 전용으로 매핑한다. [trampoline](https://ko.wikipedia.org/wiki/%ED%8A%B8%EB%9E%A8%ED%8E%84%EB%A6%B0_(%EC%BB%B4%ED%93%A8%ED%8C%85)) 함수는 APRR을 사용하여 페이지 테이블을 `rw-`로 다시 매핑하고 PPL 코드를 `r-x`로 매핑한 후 점프한다. 이 trampoline 함수는 PPL 코드의 유일한 엔트리 포인트이므로 hypercall 명령어처럼 동작하며 PPL 자체는 오버 헤드가 매우 낮은 하이퍼바이저처럼 동작한다.

상세한 내용은 [Jonathan의 Casa De P(a)P(e)L write up](http://newosxbook.com/articles/CasaDePPL.html)에서 확인할 수 있다.

# SPRR

## Userland JIT

애플 실리콘의 JIT은 `rw-`와 `r-x` 권한을 빠르게 전환하기 위해 특수한 영역을 할당받는다. 이전 SoC에서 APRR을 사용하여 시행되었다.

[Just-in-Time 컴파일러에 대한 애플의 공식 문서](https://developer.apple.com/documentation/apple-silicon/porting-just-in-time-compilers-to-apple-silicon)에서 M1에서 이러한 전환은 여전히 `_pthread_jit_write_protect_np` 함수가 수행함을 알 수 있다. `otool -xv /usr/lib/system/libsystem_pthread.dylib`을 통해 어떠한 동작을 하는지 확인해보자.

```
_pthread_jit_write_protect_np:
[...]
0000000000007fdc        movk    x0, #0xc118
0000000000007fe0        movk    x0, #0xffff, lsl #16
0000000000007fe4        movk    x0, #0xf, lsl #32
0000000000007fe8        movk    x0, #0x0, lsl #48
0000000000007fec        ldr     x0, [x0]                ; Latency: 4
0000000000007ff0        msr     S3_6_C15_C1_5, x0
0000000000007ff4        isb
[...]
```

주소 `0xffffc118`에서 64비트 정수를 로드한 후 `s3_6_c15_c1_5` 시스템 레지스터에 작성한다. `0xffffc110`에서 새로운 시스템 레지스터 값을 로드하는 비슷한 코드가 더 있다. 이 주소들은 모든 유저 랜드 프로세스에 매핑되고 커널이 유저 랜드에 전달하는 많은 변수를 포함한 commpage 영역이다.

오픈 소스 XNU 코드에는 commpage 내부에 이 변수들을 설정하는 코드가 없지만 이전 세대 APRR에 사용된 `cp_aprr_shadow_jit_rw`에 대한 참조 코드가 남아있다.

이 C 프로그램을 덤프 해보자

```c
#include <stdio.h>
#include <stdint.h>

int main(int argc, char *argv[])
{
  uint64_t *sprr = (uint64_t *)0xfffffc110;
  printf("%llx %llx\n", sprr[0], sprr[1]);
}
```

`0x2010000030300000`과 `0x2010000030100000` 값을 통해 JIT 페이지의 `r-x`와 `rw-` 권한을 전환한다. 이는 APRR과 유사하지만 다른 레지스터를 사용하며 다른 magic number를 포함한다.

SPRR에 대한 대략적인 아이디어를 통해 커널을 disassemble 하고 이 레지스터나 근처 레지스터를 사용하는 함수를 찾을 수 있다.

우리가 찾은 레지스터에서 비트를 수정하고 어떻게 작동하는지 확인하자. M1에서 실행되는 일반 유저 랜드 프로그램에서도 가능하다.

모든 비트를 0과 1로 설정

```c
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <string.h>

void write_sprr(uint64_t v)
{
    __asm__ __volatile__("msr S3_6_c15_c1_5, %0\n"
                         "isb sy\n" ::"r"(v)
                         :);
}

uint64_t read_sprr(void)
{
    uint64_t v;
    __asm__ __volatile__("isb sy\n"
                         "mrs %0, S3_6_c15_c1_5\n"
                         : "=r"(v)::"memory");
    return v;
}

int main(int argc, char *argv[])
{
    for (int i = 0; i < 64; ++i) {
        write_sprr(1ULL<<i);
        printf("bit %02d: %016llx\n", i, read_sprr());
    }
```

commpage에서 찾은 두 값이 다른 두 비트를 제외한 모든 비트가 initial value로 잠겼다. 이를 통해 JIT 페이지 권한과 관련이 있음을 알 수 있다. `mmap`을 사용하여 페이지를 매핑할 수 있다. 읽기나 쓰기가 보호된 페이지를 읽거나 쓰면 `SIGBUS`가 발생하고 실행 권한이 없는 페이지로 점프하면 `SIGSEV`가 발생한다. signal handler를 통해 유저 랜드 애플리케이션에서 이러한 signal을 확인할 수 있다.

보호된 페이지에 접근하기 위해 `x0`을 magic constant로 설정한 signal handler를 설정하고 return 되기 전에 프로그램 카운터를 증가시킨다.

```c
void bus_handler(int signo, siginfo_t *info, void *cx_)
{
    ucontext_t *cx = cx_;
    cx->uc_mcontext->__ss.__x[0] = 0xdeadbeef;
    cx->uc_mcontext->__ss.__pc += 4;
}
```

실행 권한이 없는 페이지를 실행하는 것도 비슷한 방법으로 하면 된다. 프로그램 카운터를 링크 레지스터로 설정하고 callee로 돌아가 magic value를 `x0`에 저장한다.

```c
void sev_handler(int signo, siginfo_t *info, void *cx_)
{
    ucontext_t *cx = cx_;
    cx->uc_mcontext->__ss.__x[0] = 0xdeadbeef;
    cx->uc_mcontext->__ss.__pc = cx->uc_mcontext->__ss.__lr;
}
```

이제 `MAP_JIT`으로 페이지를 매핑하고 시스템 레지스터에 모든 4개의 가능한 값을 메모리 읽기 혹은 쓰기를 실행한다.

- SPRR JIT test code

    ```c
    #define _XOPEN_SOURCE
    #include <signal.h>
    #include <stdbool.h>
    #include <stdint.h>
    #include <stdio.h>
    #include <string.h>
    #include <sys/mman.h>
    #include <sys/utsname.h>
    #include <ucontext.h>

    static void sev_handler(int signo, siginfo_t *info, void *cx_)
    {
        (void)signo;
        (void)info;
        ucontext_t *cx = cx_;
        cx->uc_mcontext->__ss.__x[0] = 0xdeadbeef;
        cx->uc_mcontext->__ss.__pc = cx->uc_mcontext->__ss.__lr;
    }

    static void bus_handler(int signo, siginfo_t *info, void *cx_)
    {
        (void)signo;
        (void)info;
        ucontext_t *cx = cx_;
        cx->uc_mcontext->__ss.__x[0] = 0xdeadbeef;
        cx->uc_mcontext->__ss.__pc += 4;
    }

    static void write_sprr_perm(uint64_t v)
    {
        __asm__ __volatile__("msr S3_6_c15_c1_5, %0\n"
                             "isb sy\n" ::"r"(v)
                             :);
    }

    static uint64_t read_sprr_perm(void)
    {
        uint64_t v;
        __asm__ __volatile__("isb sy\n"
                             "mrs %0, S3_6_c15_c1_5\n"
                             : "=r"(v)::"memory");
        return v;
    }

    static bool can_read(void *ptr)
    {
        uint64_t v = 0;

        __asm__ __volatile__("ldr x0, [%0]\n"
                             "mov %0, x0\n"
                             : "=r"(v)
                             : "r"(ptr)
                             : "memory", "x0");

        if (v == 0xdeadbeef)
            return false;
        return true;
    }

    static bool can_write(void *ptr)
    {
        uint64_t v = 0;

        __asm__ __volatile__("str x0, [%0]\n"
                             "mov %0, x0\n"
                             : "=r"(v)
                             : "r"(ptr + 8)
                             : "memory", "x0");

        if (v == 0xdeadbeef)
            return false;
        return true;
    }

    static bool can_exec(void *ptr)
    {
        uint64_t (*fun_ptr)(uint64_t) = ptr;
        uint64_t res = fun_ptr(0);
        if (res == 0xdeadbeef)
            return false;
        return true;
    }

    static void sprr_test(void *ptr, uint64_t v)
    {
        uint64_t a, b;
        a = read_sprr_perm();
        write_sprr_perm(v);
        b = read_sprr_perm();

        printf("%llx: %c%c%c\n", b, can_read(ptr) ? 'r' : '-', can_write(ptr) ? 'w' : '-',
               can_exec(ptr) ? 'x' : '-');
    }

    static uint64_t make_sprr_val(uint8_t nibble)
    {
        uint64_t res = 0;
        for (int i = 0; i < 16; ++i)
            res |= ((uint64_t)nibble) << (4 * i);
        return res;
    }

    int main(int argc, char *argv[])
    {
        (void)argc;
        (void)argv;

        struct sigaction sa;
        sigfillset(&sa.sa_mask);
        sa.sa_sigaction = bus_handler;
        sa.sa_flags = SA_RESTART | SA_SIGINFO;
        sigaction(SIGBUS, &sa, 0);
        sa.sa_sigaction = sev_handler;
        sigaction(SIGSEGV, &sa, 0);

        uint32_t *ptr = mmap(NULL, 0x4000, PROT_READ | PROT_WRITE | PROT_EXEC,
                             MAP_PRIVATE | MAP_ANONYMOUS | MAP_JIT, -1, 0);
        write_sprr_perm(0x3333333333333333);
        ptr[0] = 0xd65f03c0; // ret

        for (int i = 0; i < 4; ++i)
            sprr_test(ptr, make_sprr_val(i));
    }
    ```

다음과 같은 결과 테이블이 주어진다.

| 레지스터 값 | 페이지 권한 |
| ----------- | ----------- |
| 00          | `---`       |
| 01          | `r-x`       |
| 10          | `r--`       |
| 11          | `rw-`       |


APRR의 동작 방식보다 훨씬 단순하다. 두 개의 레지스터를 사용하여 첫 권한을 설정하고 다른 레지스터를 마스킹하는 대신에 4개의 값 중 하나로만 수정할 수 있다. APRR은 userspace에 `rxw` 매핑을 생성할 수 있는 해킹 방법이 존재하였는데, 이를 인코딩하는 방법이 없으므로 불가능하다. 시스템 레지스터의 다른 바이트들은 페이지 테이블 엔트리에 인코딩 된 16개의 다른 사용 권한일 수도 있다. 그러면 시스템 레지스터의 나머지는 알 수 없겠는걸

이제 새로운 하드웨어 기능의 동작 방식에 대해 살펴보자.

애플의 하이퍼바이저 프레임워크를 사용하여 EL1에서 코드를 실행하여 SPRR이 어떻게 동작하는지 확인하고 싶었지만, SPRR과 연관된 레지스터에 접근하면 오류가 발생했다. 대신에 베어 메탈(bare metal)을 통해 EL2에서 코드를 실행하자.

## m1n1

이전 아이폰 해커들은 XNU을 정적 리버싱 하거나 exploit 하기 위해 EL1까지 가는 과정을 실험했다. 하지만 M1은 많은 하드웨어 기능을 공유하며 부트 프로세스에서 서명되지 않은 코드를 실행할 수 있다.

M1에 대한 upstream linux 지원을 목표로 하는 Asahi Linux project의 일부로 m1n1이라 불리는 작은 부트로더/하드웨어 플랫폼이 개발되었다. m1n1은 XNU와 동시에 제어 권한을 갖는다. 아래의 모든 작업은 EL2에서 실행할 수 있는 shellcode로 작성할 수 있지만 m1n1을 사용하여 진행한다.

## Python에서 알 수 없는 시스템 레지스터 발견

m1n1은 shellcode를 컴파일하고 로드하고 데이터 추출을 핸들링하는 등 이러한 모든 세부 사항을 파이썬 쉘에서 직접 하드웨어를 조작할 수 있다. 또 최근 필자의 USB 가젯 코드를 병합하여 m1 MAC과 일반 USB 케이블만으로 테스트할 수 있다.

`proxyclient/shell.py`를 실행하자. 유저 랜드 SPRR 레지스터에 접근하면 예외가 발생한다.

```python
>>> u.mrs((3, 6, 15, 1, 5))
TTY> Exception: SYNC
TTY> Exception taken from EL2h
TTY> Running in EL2
TTY> MPIDR: 0x80000000
TTY> Registers: (@0x8046b3db0)
TTY>   x0-x3: 0000000000000000 0000000000000000 0000000000000000 0000000000000000
TTY>   x4-x7: 0000000810cb8000 0000000000007a69 0000000804630004 0000000804630000
TTY>  x8-x11: 0000000000000000 00000000ffffffc8 00000008046b3eb0 000000000000002c
TTY> x12-x15: 0000000000000003 0000000000000001 0000000000000000 00000008046b3b20
TTY> x16-x19: 00000008045caa80 0000000000000000 0000000000000000 000000080462b000
TTY> x20-x23: 00000008046b3f78 00000008046b3fa0 0000000000000002 00000008046b3f98
TTY> x24-x27: 00000008046b3f70 0000000000000000 0000000000000001 0000000000000001
TTY> x28-x30: 00000008046b3fa0 00000008046b3eb0 00000008045bad90
TTY> PC:       0x810cb8000 (rel: 0xc70c000)
TTY> SP:       0x8046b3eb0
TTY> SPSR_EL1: 0x60000009
TTY> FAR_EL1:  0x0
TTY> ESR_EL1:  0x2000000 (unknown)
TTY> L2C_ERR_STS: 0x11000ffc00000000
TTY> L2C_ERR_ADR: 0x0
TTY> L2C_ERR_INF: 0x0
TTY> SYS_APL_E_LSU_ERR_STS: 0x0
TTY> SYS_APL_E_FED_ERR_STS: 0x0
TTY> SYS_APL_E_MMU_ERR_STS: 0x0
TTY> Recovering from exception (ELR=0x810cb8004)
Traceback (most recent call last):
  File "/opt/homebrew/Cellar/python@3.9/3.9.4/Frameworks/Python.framework/Versions/3.9/lib/python3.9/code.py", line 90, in runcode
    exec(code, self.locals)
  File "<console>", line 1, in <module>
  File "/Users/speter/asahi/git/m1n1/proxyclient/utils.py", line 80, in mrs
    raise ProxyError("Exception occurred")
proxy.ProxyError: Exception occurred
>>>
```

커널은 컨텍스트 전환 중에 이 레지스터를 수정할 수 있어야 한다. 이는 enable bit가 있음을 의미하며 m1n1 repository에 사용 가능한 시스템 레지스터를 찾는 파이썬 툴이 있다. 내부적으로 모든 명령에 대해 mrs 명령을 생성하고 정의되지 않은 레지스터로 이한 예외를 복구한다. 그것을 실행하여 근처의 레지스터를 찾는다.

```python
$ python3 proxyclient/find_all_regs.py | grep s3_6_c15_c1_
s3_6_c15_c1_0 (3, 6, 15, 1, 0) = 0x0
s3_6_c15_c1_2 (3, 6, 15, 1, 2) = 0x0
s3_6_c15_c1_4 (3, 6, 15, 1, 4) = 0x0
```

MMU를 비활성화하고 각 레지스터에 모든 레지스터를 작성하고, 레지스터를 다시 찾아서 새로운 레지스터를 식별한다. 이것은 몇 줄의 파이썬 코드로 수행할 수 있다.

```python
with u.mmu_disabled():
    for reg in [(3, 6, 15, 1, 0), (3, 6, 15, 1, 2)]:
        old_regs = find_regs()
        u.msr(reg, 1)
        new_regs = find_regs()

        diff_regs = new_regs - old_regs

        print(reg)
        for r in sorted(diff_regs):
            print("  %s" % list(r))

    u.msr((3, 6, 15, 1, 2), 0)
    u.msr((3, 6, 15, 1, 0), 0)
```

- 활성화된 시스템 레지스터

    ```python
    (3, 6, 15, 1, 0)
      [3, 4, 15, 5, 1]
      [3, 4, 15, 5, 2]
      [3, 4, 15, 7, 0]
      [3, 4, 15, 7, 1]
      [3, 4, 15, 7, 2]
      [3, 4, 15, 7, 3]
      [3, 4, 15, 7, 4]
      [3, 4, 15, 7, 5]
      [3, 4, 15, 7, 6]
      [3, 4, 15, 7, 7]
      [3, 4, 15, 8, 0]
      [3, 4, 15, 8, 1]
      [3, 4, 15, 8, 2]
      [3, 4, 15, 8, 3]
      [3, 4, 15, 8, 4]
      [3, 4, 15, 8, 5]
      [3, 4, 15, 8, 6]
      [3, 4, 15, 8, 7]
      [3, 6, 15, 1, 3]
      [3, 6, 15, 1, 5]
      [3, 6, 15, 1, 6]
      [3, 6, 15, 1, 7]
      [3, 6, 15, 3, 0]
      [3, 6, 15, 3, 1]
      [3, 6, 15, 3, 2]
      [3, 6, 15, 3, 3]
      [3, 6, 15, 3, 4]
      [3, 6, 15, 3, 5]
      [3, 6, 15, 3, 6]
      [3, 6, 15, 3, 7]
      [3, 6, 15, 4, 0]
      [3, 6, 15, 4, 1]
      [3, 6, 15, 4, 2]
      [3, 6, 15, 4, 3]
      [3, 6, 15, 4, 4]
      [3, 6, 15, 4, 5]
      [3, 6, 15, 4, 6]
      [3, 6, 15, 4, 7]
      [3, 6, 15, 5, 0]
      [3, 6, 15, 5, 1]
      [3, 6, 15, 5, 2]
      [3, 6, 15, 5, 3]
      [3, 6, 15, 5, 4]
      [3, 6, 15, 5, 5]
      [3, 6, 15, 5, 6]
      [3, 6, 15, 5, 7]
      [3, 6, 15, 6, 0]
      [3, 6, 15, 6, 1]
      [3, 6, 15, 6, 2]
      [3, 6, 15, 6, 3]
      [3, 6, 15, 6, 4]
      [3, 6, 15, 6, 5]
      [3, 6, 15, 6, 6]
      [3, 6, 15, 6, 7]
      [3, 6, 15, 14, 3]
      [3, 6, 15, 15, 5]
      [3, 6, 15, 15, 7]
    (3, 6, 15, 1, 2)
      [3, 1, 15, 8, 2]
      [3, 6, 15, 0, 3]
      [3, 6, 15, 8, 1]
      [3, 6, 15, 8, 2]
      [3, 6, 15, 9, 2]
      [3, 6, 15, 9, 3]
      [3, 6, 15, 9, 4]
      [3, 6, 15, 9, 5]
      [3, 6, 15, 9, 6]
      [3, 6, 15, 9, 7]
      [3, 6, 15, 10, 0]
      [3, 6, 15, 10, 1]
      [3, 6, 15, 12, 0]
      [3, 6, 15, 12, 1]
      [3, 6, 15, 15, 2]
      [3, 6, 15, 15, 3]
    ```

`S3_6_C15_C1_0`를 `SPRR_CONFIG_EL1`로 칭하자. 그 레지스터의 비트 1은 SPRR을 활성화하고 모든 비트를 설정한 후 추가적인 변경을 위해 모든 SPRR 레지스터를 잠그는 것으로 보인다. `S3_6_C15_1_2`과 그것을 활성화하는 레지스터는 파트 2에서 중요하다.

`S3_6_C15_C1_5`의 모든 비트를 수정할 수 있다.

```python
>>> p.mmu_shutdown()
TTY> MMU: shutting down...
TTY> MMU: shutdown successful, clearing cache
>>> u.msr((3, 6, 15, 1, 0), 1)
>>> u.mrs((3, 6, 15, 1, 5))
0x0
>>> u.msr((3, 6, 15, 1, 5), 0xffffffffffffffff)
>>> u.mrs((3, 6, 15, 1, 5))
0xffffffffffffffff
>>>
```

이 레지스터가 EL0에 적용되는 것 같지만, 여기서는 EL2에서 실행된다. 새로 활성화된 레지스터 `S3_6_C15_C1_6`은 EL1에 대한 것이며 `S3_6_C15_C1_7`은 EL2에 대한 것이라고 가정할 수 있다. M1은 항상 `HCR_EL2.E2H`로 실행되며 E1 레지스터에 대한 접근을 EL2 counterparts로 리다이렉션 한다. 이를 사용하여 가정을 증명할 수 있다.

```python
>>> u.msr((3, 6, 15, 1, 6), 0xdead0000)
>>> u.mrs((3, 6, 15, 1, 7))
0xdead0000
>>>
```

SPRR을 이제 활성화할 수 있으며 EL2 권한으로 사용되는 것 같은 레지스터도 있다. 이 레지스터의 4 비트를 이해하기 위해 유저 랜드에서 한 실험을 해보자.

## SPRR 리버스 엔지니어링

간단한 페이지 테이블을 설정하기 위한 파이썬 코드를 작성한 후 유저 랜드에서 했던 것과 동일하게 실험할 수 있다. `S3_6_C15_C1_6`의 권한 바이트를 페이지에 매핑하고 해당 페이지에서 읽기, 쓰기, 실행을 해보자.

파이썬에서 이 작업을 완벽하게 수행하는 것은 `r-x` 페이지에서 실행되고 스택을 `rw-` 페이지에 유지하기 위해 m1n1을 변경할 때만 가능하다. 파이썬에서 가능한 많은 설정 작업을 하고 shellcode를 작성하고 다른 코어에서 실행하는 것이 훨씬 쉽다. 하나가 중단되더라도 재부팅이 되기 전에 다른 것이 남아있다.

```python
pagetable = ARMPageTable(heap.memalign, heap.free)
pagetable.map(0x800000000, 0x800000000, 0xc00000000, 0)   # normal memory, we run from here
pagetable.map(0xf800000000, 0x800000000, 0xc00000000, 1)  # probe memory, we'll try to read/write/execute this
# ...
code_page = build_and_write_code(heap, """
    // [...]
                // prepare and enable MMU
                ldr x0, =0x0400ff
                msr MAIR_EL1, x0
                ldr x0, =0x27510b510 // borrowed from m1n1's MMU code
                msr TCR_EL1, x0
                ldr x0, =0x{ttbr:x}
                msr TTBR0_EL1, x0
                mrs x0, SCTLR_EL1
                orr x1, x0, #5
                msr SCTLR_EL1, x1
                isb
    // [...]
""".format(ttbr=pagetable.l0)
# ...
ret = p.smp_call_sync(1, code_page, sprr_val)
# ...
```

signal handler는 이제 exception vector가 된다. 단일 레지스터를 수정하여 failure를 표시한 후 프로그램 카운터를 두 번 이동한 후 return 한다. 첫 번째 명령어는 다시 실행하고 싶지 않은 잘못된 명령이다. 두 번째는 `mov x10, 0x80`으로 접근을 성공했음을 표시하며 예외가 처리될 경우 표시되지 않는다.

```python
_fault_handler:
# store that we failed
mov x10, 0xf1

mrs x12, ELR_GL2  # get the PC that faulted
add x12, x12, 8   # skip two instructions
msr ELR_GL2, x12  # store the updated PC

isb
# eret restores the state from before the exception was taken
eret

_sprr_test:
# ...

# test read access, x1 contains an address to a page for which we modify the SPRR register values
mov x10, 0    # x10 is our success/failure indicator
ldr x1, [x1]  # this instruction will fault if we can't read from [x1]
mov x10, 0x80 # this instruction will be skipped if the previous one faulted
```

모든 16개의 가능한 설정을 얻을 수 있다.

| 레지스터 값 | 페이지 권한 |
| -------------- | ---------------- |
| 0000           | ---              |
| 0001 | r-x |
| 0010 | r-- |
| 0011 | rw- |
| 0100 | --- |
| 0101 | r-x |
| 0110 | r-- |
| 0111 | --- |
| 1000 | --- |
| 1001 | --x |
| 1010 | r-- |
| 1011 | rw- |
| 1100 | --- |
| 1101 | r-x |
| 1110 | r-- |
| 1111 | rw- |

여기에 이상한 점이 존재하는데, 대부분 하위 두 비트가 권한을 지정한다. 그러나 높은 비트가 권한을 변경하는 두 가지 예외가 존재한다. `0111`의 경우 `rw-`가 아닌 `---` 권한을 가지고, `1001`의 경우 `r-x`가 아닌 `r` 권한만을 가진다.

이를 인코딩하는데 두 개의 비트가 더 낭비될 필요가 없다. 이것은 처음에 쓰기, 실행 권한을 엄격하게 시행하는 유저 vs 커널 권한으로 보인다. 하지만 EL0은 다른 레지스터를 사용하는 것을 알고 있다. 그럼 얘는 뭘까?

# Guarded Exception Levels / GXF

위에서 PPR 레지스터에 인코딩 된 약간 이상한 점을 발견했다. normal exception levels과는 다른 [guarded exception levels](https://twitter.com/s1guza/status/1353749746951839748)에 대한 [몇 가지](https://twitter.com/qwertyoruiopz/status/1174787964100075521) [언급](https://twitter.com/s1guza/status/1355929535699681284)이 있다. 이것은 커스텀 명령어 `0x00201420`과 `0x00201400`(`genter`, `gexit`)을 통해 트리거 된다.

disassembler로 XNU를 가져가서, `otool -xv /System/Library/Kernels/kernel.release.t8101`을 사용하여 살펴보자.

```
fffffe00071f80f0        mov     x0, #0x1
fffffe00071f80f4        msr     S3_6_C15_C1_2, x0
fffffe00071f80f8        adrp    x0, 2025 ; 0xfffffe00079e1000
fffffe00071f80fc        add     x0, x0, #0x9d8
fffffe00071f8100        msr     S3_6_C15_C8_2, x0
fffffe00071f8104        adrp    x0, 2025 ; 0xfffffe00079e1000
fffffe00071f8108        add     x0, x0, #0x9dc
fffffe00071f810c        msr     S3_6_C15_C8_1, x0
fffffe00071f8110        isb
fffffe00071f8114        mov     x0, #0x0
fffffe00071f8118        msr     ELR_EL1, x0
fffffe00071f811c        isb
fffffe00071f8120        .long   0x00201420
fffffe00071f8124        ret
```

이전에 찾은 두 번째 활성화 레지스터 `S3_6_C15_C1_2`를 처음 사용한 후 알 수 없는 시스템 레지스터에 두 개의 포인터를 작성하고 `0x00201420`을 실행한다. 첫 번째 포인터는 무한 루프이지만 두 번째 포인터는 이전에 확인한 SPRR 레지스터도 사용하는 것으로 추측되는 함수를 가리킨다.

아마도 `S3_6_C15_C8_1`에 `0x00201420`이 실행되면 프로세스가 점프하는 포인터가 포함되며 `0x00201420`는 EL3에 트랩하는 `smc`이고, `0x00201400`은 EL2로 돌아가는 `eret`인 하이퍼바이저 호출 방식과 유사하다. 새로운 실행 모드에 대한 다른 페이지 테이블이 없는 것이 차이점이다. SPRR 레지스터에 알려지지 않은 2 비트가 GL2의 페이지 권한이면 어떨까?

이전과 같은 접근법을 사용하여 m1n1에서 다시 검증을 해보았다. guarded excution mode에 exception vector를 설정하고 같은 실험을 반복하자.

`VBAR`라 불리는 레지스터를 호출하여 새로운 모드에서 exception vector를 설정할 수 있다. `genter` 이후 XNU가 설정하는 첫 레지스터 중 하나인 `S3_6_C15_C10_2`가 가리키는 코드를 살펴보자.

```
fffffe00079e0000        b       0xfffffe00079e15d0
fffffe00079e0004        nop
fffffe00079e0008        nop
fffffe00079e000c        nop
[...]
fffffe00079e007c        nop
fffffe00079e0080        b       0xfffffe00079e1000
fffffe00079e0084        nop
[...]
fffffe00079e00fc        nop
fffffe00079e0100        b       0xfffffe00079e11f0
fffffe00079e0104        nop
[...]
```

`S3_6_C15_C10_2`가 `VBAR_GL1`임을 의미하는 예외 벡터 테이블로 보인다.

이것은 전체 권한 테이블을 찾는데에 동작한다. SPRR 레지스터의 값이 `0100`, `0110` 혹은 `1111`인 동안에 EL2에서 코드로 점프할 때 크래시가 난다. 이 모든 값들은 EL2에서  실행할 수 없지만 GL2에서는 가능한 페이지를 나타낸다. 이러한 결함은 몇 가지 이유로 인해 다른 주소로 지정되면 어떻게 될까? 세 가지 결함은 XNU에 무한 루프를 가리키는 시스템 레지스터를 사용한다.

- EL2가 GL2에서만 실행 가능한 코드로 점프하는 경우 `S3_6_C15_C8_2`로 이동(`GXF_ABORT_EL2`)
- EL2에서의 다른 중단은 `VBAR_EL2`로 이동
- GL2에서의 다른 중단은 `VBAR_GL2`로 이동

이제 SPRR 레지스터의 모든 비트가 전체 권한 테이블로 이어짐을 알았다.

| 레지스터 값 | EL 페이지 권한 | GL 페이지 권한 |
| ----------- | -------------- | -------------- |
| 0           | `---`          | `---`          |
| 1           | `r-x`      | `---`          |
| 10           | `r--`         | `---`          |
| 11           | `rw-`        | `---`          |
| 100        | `---`          | `r-x`       |
| 101        | `r-x`       | `r-x`    |
| 110      | `r--`       | `r-x`       |
| 111        | `---`          | `r-x`       |
| 1000       | `---`          | `r--`         |
| 1001     | `--x`         | `r--`         |
| 1010       | `r--`         | `r--`         |
| 1011       | `rw-`        | `r--`         |
| 1100       | `---`          | `rw-`        |
| 1101      | `r-x`       | `rw-`        |
| 1110       | `r--`         | `rw-`        |
| 1111       | `rw-`        | `rw-`        |



GL 권한 비트가 EL 권한 비트의 의미를 수정하는 두 가지 특별한 경우를 살펴보자.

- `0111`은 GL에서 실행이 가능하며 EL에서 쓰기가 가능한 페이지를 만들 수 없다. 소프트웨어 오류에 대한 추가적인 하드웨어 계층 보호 기능을 제공한다. EL에서 GL로 실행 중인 코드를 변경할 수 있는 것은 전체적인 lateral level을 무의미하게 만든다.
- `1001`은 페이지가 GL에서만 읽을 수 있을 때 `r-x` EL 권한을 `--x` 권한으로 변경한다. EL의 코드를 숨기거나 exploit에 대한 추가적인 보호 기법으로 사용하는 것으로 추측된다.

## Python을 통한 GL2 증명

이러한 지식을 가지고 m1n1에서 GL2를 실행하는 custom payload를 추가할 수 있다. EL1/EL0에 이미 존재하는 프레임워크를 추가할 필요가 있다. MMU를 비활성화(SPRR을 활성화한 상태에서 rwx 페이지에서 m1n1이 실행되므로)하고 페이로드로 이동한 다음 return 하기 전에 MMU를 다시 활성화하면 된다.

GL2를 탐색하여 `S3_6_C15_C10_3`이 `SPSR_GL2`일 가능성이 있음을 확인할 수 있다.

```
>>> u.mrs((3, 6, 15, 10, 3), call=p.gl_call)
0x60000009
>>> u.mrs(SPSR_EL2)
0x60000009
```

MSR finder 대신에 GL2에서 확인해보자.

```python
gxf_regs = find_regs(call=p.gl_call)

print("GXF")
for r in sorted(gxf_regs - all_regs):
    print("  %s" % list(r))
```

컨텍스트에서만 사용 가능한 새로운 시스템 레지스터들을 발견했다.

```
GXF
  [3, 6, 15, 0, 1]
  [3, 6, 15, 0, 2]
  [3, 6, 15, 1, 1]
  [3, 6, 15, 2, 6]
  [3, 6, 15, 8, 5]
  [3, 6, 15, 8, 7]
  [3, 6, 15, 10, 2]
  [3, 6, 15, 10, 3]
  [3, 6, 15, 10, 4]
  [3, 6, 15, 10, 5]
  [3, 6, 15, 10, 6]
  [3, 6, 15, 10, 7]
  [3, 6, 15, 11, 1]
  [3, 6, 15, 11, 2]
  [3, 6, 15, 11, 3]
  [3, 6, 15, 11, 4]
  [3, 6, 15, 11, 5]
  [3, 6, 15, 11, 6]
  [3, 6, 15, 11, 7]
```

`3, 6, 15, 10`은 GL1 용도이고 `3, 6, 15, 11`은 GL2를 위한 것인지 알아내는 것은 간단하다. EL2에서 SPRR 및 GXF를 활성화한 후 EL1로 낮추어 동일한 작업을 하면 된다. 이번 작업은 새로운 레지스터만을 구한다.

```
	[3, 6, 15, 0, 1]
  [3, 6, 15, 8, 7]
  [3, 6, 15, 10, 1]
  [3, 6, 15, 10, 2]
  [3, 6, 15, 10, 3]
  [3, 6, 15, 10, 4]
  [3, 6, 15, 10, 5]
  [3, 6, 15, 10, 6]
  [3, 6, 15, 10, 7]
```

`3, 6, 15, 10` 그룹은 EL1 레지스터이다. 하지만 이것은 크게 문제가 되지는 않는다. M1은 항상 `HCR_EL2.E2H`에서 실행되며 이는 `_EL1` 레지스터가 `_EL2` 레지스터로 리다이렉션 됨을 의미한다.

이전 XNU 릴리스에서 다음과 같은 이름을 확인할 수 있다.

```
#define KERNEL_MODE_ELR      ELR_GL11
#define KERNEL_MODE_FAR      FAR_GL11
#define KERNEL_MODE_ESR      ESR_GL11
#define KERNEL_MODE_SPSR     SPSR_GL11
#define KERNEL_MODE_ASPSR    ASPSR_GL11
#define KERNEL_MODE_VBAR     VBAR_GL11
#define KERNEL_MODE_TPIDR    TPIDR_GL11
```

왜 이 레지스터들이 GL11 suffix를 가지고 있는지 분명하지 않지만 위에서 찾은 레지스터와 쉽게 매치할 수 있다. ASPSR에는 `gexit`가 guarded execution으로 return 해야 하는지 일반 실행으로 return 해야 하는지 결정하는 비트가 포함되어 있다.

# SPRR & GXF inside XNU

마지막으로 XNU가 새로운 기능을 어떻게 사용하는지 확인하자. Jonathan의 wirte up에 이미 SPRR과 GXF가 어떻게 사용되는지 있으므로 살펴볼 것이 별로 없다. SPRR은 APRR의 역할을 대체할 뿐이며 커널이 페이지 테이블에 쓸 수 없도록 하고 PPL 코드를 실행할 수 없도록 한다.

주요한 차이점은 GXF이다. APRR 레지스터를 수정하는 trampoline 함수를 주의해서 만드는 대신 GXF 엔트리 벡터를 설정한다. 그러면 페이지 테이블 권한은 자동적으로 변경되고 genter는 PPL을 직접 가리킬 수 있다.

XNU가 SPRR를 다음과 같이 초기화한다. 시작 함수는 SPRR이 EL1 SPRR 권한 레지스터를 `0x2020A505F020F0F0`로 초기화할 수 있도록 잠시 활성화한다.

그 후 초기 GL 부트스트랩 코드는 추가적인 변경을 방지하기 위해 lock 하기 전에 EL1 권한을 `0x2020-A506F020F0E0`으로 업데이트한다.

guarded excution mode의 엔트리 포인트는 커널 텍스트 영역에서 함수로 설정되며 `PPLTEXT`의 시작 부분으로 점프한다. PPL 엔트리 함수는 SPRR 권한이 올바르게 설정되었는지 확인하고 Jonathan의 write up에 작성된 대로 동작한다.

마지막으로 XNU에 사용되는 여러 가지 SPRR 페이지 권한을 살펴보자.

| Index | Normal permissions | SPRR Permissions | Usage |
| --------- | --------- | ----- | ----- |
| 0 | EL0 | EL2 |EL0|
| 1 | `--x` | `rw-` |`---`|
| 3 | `---` | `rw-` |`---`|
| 5 | `rw-` | `rwx` |`rw-` `r-x`|
| 7 | `rw-` | `rw-` |`rw-`|
| 8 | `--x` | `r-x` |`---`|
| 10 | `---` | `r-x` |`---`|
| 11 | `---` | `r--` |`---`|
| 13 | `r-x` | `r--` |`r-x`|
| 15 | `r--` | `r--` |`r--`|

GL 권한은 GL이 커널 코드를 실행하지 못하게 하고(entry 10) 유저 데이터에 접근할 수 없게(entry 7) lock 될 수 있다. 

그 외에도 이전 APRR 하드웨어의 깔끔한 상위 버전으로 느껴진다. 이런 변화들은 전체적인 시스템을 오류에 덜 취약하게 만들며(레지스터들을 잠그거나, kernel→PPL 전환이 하드웨어에서 발생하며 커널과 PPL 예외 벡터는 분리되어 있다) 유연하게 만든다. APRR은 권한을 제거하는 데에만 사용되었지만 SPRR은 `rwx` 페이지가 필요하지 않을 때 권한을 다시 매핑할 수 있다.

# tl;dr

애플 실리콘은 공격에 대한 추가적인 보호 기법으로 함께 동작하는 두 가지 비밀스러운 기능을 가지고 있다. GXF는 GL1과 GL2라는 lateral exception level을 도입하여 다른 페이지 권한으로 EL과 상응하는 같은 페이지 테이블을 사용할 수 있다. SPRR은 EL과 GL의 페이지 엔트리에서 권한 비트를 재정의할 수 있다. 애플은 이를 통해 GL의 모든 페이지 테이블 조작 코드를 숨기고 EL이 페이지 테이블을 수정할 수 없도록 하였다. 이는 커널 모드에서 실행되는 코드로부터 페이지 테이블을 보호하여 적은 attack surface와 오버헤드를 가진 하이퍼 바이저가 효과적으로 도입된다. 이 기능들의 대부분은 파이썬과 m1n1을 사용하여 리버스 엔지니어링 할 수 있다.

리눅스를 M1으로 포팅하는 데에는 유용하지 않지만 XNU를 가상화하여 MMIO access를 trace 할 수 있다.