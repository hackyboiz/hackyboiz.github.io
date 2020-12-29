---
title: "[Write-Up] Christmas CTF 2020 - angrforge"
author: idioth
tags: [idioth, ctf, christmas ctf 2020, reversing, angr]
categories: [Write-Up]
date: 2020-12-29 21:00:00
cc: true
index_img: /2020/12/29/idioth/christmasctf2020-angrforge/thumbnail.png
---

# 출제 의도

angr를 통해 입력 값을 뽑아내는 것이 의도인 문제였습니다. 이 문제는 여러 번의 수정을 거쳤습니다. 원래 이 문제도 arm 환경에서 angr를 돌리는 문제로 낼 예정이었는데 arm에서 제대로 동작하지 않아서 arm은 포기. 그리고 원래 처음에는 c++로 냈었는데 검수 후 수정을 했더니 simulation manager를 돌려도 값이 제대로 나오지 않아서 c로 옮기는 과정에서 연산 몇 개를 뺐습니다..ㅠ

# Solution

```
idioth@ubuntu:~/Desktop$ file angrforge
angrforge: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2,
BuildID[sha1]=1162ed229a12133d07de26301dad1ada34a9c3ff, for GNU/Linux 3.2.0, stripped
```

64bit ELF 파일이며, stripped 되어있습니다.

```c
undefined8 FUN_00103be1(void)

{
  int iVar1;
  long in_FS_OFFSET;
  undefined8 local_58;
  undefined8 local_50;
  undefined8 local_48;
  undefined8 local_40;
  undefined8 local_38;
  undefined8 local_30;
  undefined8 local_28;
  undefined local_20;
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  local_58 = 0;
  local_50 = 0;
  local_48 = 0;
  local_40 = 0;
  local_38 = 0;
  local_30 = 0;
  local_28 = 0;
  local_20 = 0;
  puts("General Angerforge, the Dark Iron responsible for stealing my computer.");
  puts("But I\'m just a programmer.. so Call me my best warrior friend.");
  puts("If you call my friend, I will give you a good reward.");
  fgets((char *)&local_58,0x39,stdin);
  FUN_00103a48(&local_58);
  iVar1 = FUN_001039f7(&local_58);
  if (iVar1 == 1) {
    puts("OMG, Thank you for your good works :)");
  }
  else {
    FUN_001039d9();
  }
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```

ghidra를 통해 디컴파일을 한 후 main을 확인해보면, `0x39(57)`만큼의 입력 값을 받아서 `FUN_00103a48`을 호출하고, `FUN_001039f7`을 통해 문자열을 체크하는 것을 확인할 수 있습니다.

```c
void FUN_00103a48(char *param_1)

{
  size_t sVar1;
  
  sVar1 = strlen(param_1);
  if (sVar1 < (ulong)(long)(int)((DAT_00106014 | DAT_00106010) + 5)) {
    FUN_001033e9(param_1);
    FUN_00103481(param_1);
    FUN_00103519(param_1);
    FUN_001035b1(param_1);
    FUN_00103649(param_1);
    FUN_001036e1(param_1);
    FUN_00103779(param_1);
    FUN_00103811(param_1);
    FUN_001038a9(param_1);
    FUN_00103941(param_1);
  }
  else {
    FUN_001011c9(param_1);
    FUN_00101396(param_1);
    FUN_00101563(param_1);
    FUN_00101730(param_1);
    FUN_001018fd(param_1);
    FUN_00101abf(param_1);
    FUN_00101bbf(param_1);
    FUN_00101cce(param_1);
    FUN_00101dce(param_1);
    FUN_00101eee(param_1);
    FUN_001022cb(param_1);
    FUN_00102589(param_1);
    FUN_00102847(param_1);
    FUN_00102b05(param_1);
    FUN_00102dc3(param_1);
    FUN_00102ef9(param_1);
    FUN_00103035(param_1);
    FUN_00103171(param_1);
    FUN_001032ad(param_1);
  }
  return;
}
```

`FUN_00103a48` 함수는 입력 값을 받아서, 길이에 따라서 여러 가지 다른 sub 함수를 수행합니다. 각 sub 함수의 연산은 서로 다른 바이트에 영향을 미치지 않고 함수가 다른 함수를 호출하는 로직도 있어서 상당히 복잡하게 얽혀있습니다. c++에는 곱 연산 같은 것도 넣었는데 c로 급하게 옮기면서 보니 바이트가 증발하더군요..ㅠ ~~시간 부족으로 인한 역 연산 가능 로직~~

```c
undefined8 FUN_001039f7(long param_1)
{
  int local_c;
  
  local_c = 0;
  while( true ) {
    if (0x37 < local_c) {
      return 1;
    }
    if (*(char *)(param_1 + local_c) != (&DAT_00104080)[local_c]) break;
    local_c = local_c + 1;
  }
  return 0;
}
```

`FUN_001039f7`에서는 0x38만큼 `DAT_00104080`과 값을 비교하여 맞으면 `1`, 아닐 시 `0`을 리턴해줍니다.

`\n`을 제외한 문자열의 길이는 56이고, `stdin`을 통해 입력 값이 들어가므로 입력 값 56과 `stdin`을 처리하는 state를 구성하여 simulation manager를 돌리면 값을 구할 수 있습니다.

```python
import angr
import claripy

p = angr.Project('./angrforge')

flag_chars = [claripy.BVS('flag_%d' % i, 8) for i in range(56)]
flag = claripy.Concat(*flag_chars + [claripy.BVV(b'\n')])

st = p.factory.full_init_state(
    stdin = flag,
    add_options = angr.options.unicorn,
)

for i in flag_chars:
    st.solver.add(i != 0)
    st.solver.add(i != 10)

sm = p.factory.simulation_manager(st)
sm.run()

for i in sm.deadended:
    if b'OMG' in i.posix.dumps(1):
        print(i.posix.dumps(0))
```

```
idioth@ubuntu:~/Desktop$ python3 solve.py
WARNING | 2020-12-28 20:50:44,086 | cle.loader | The main binary is a position-independent executable.
It is being loaded with a base address of 0x400000.
WARNING | 2020-12-28 20:50:45,089 | angr.simos.simos | stdin is constrained to 57 bytes (has_end=True).
If you are only providing the first 57 bytes instead of the entire stdin,
please use stdin=SimFileStream(name='stdin', content=your_first_n_bytes, has_end=False).
b'XMAS{h3_1s_b1o0d3lf_d3athkni9ht_wh0_will_kill_4ngrf0rge}\n'
```

Flag : XMAS{h3_1s_b1o0d3lf_d3athkni9ht_wh0_will_kill_4ngrf0rge}