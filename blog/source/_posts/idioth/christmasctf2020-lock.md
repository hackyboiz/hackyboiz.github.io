---
title: "[Write-Up] Christmas CTF 2020 - lock"
author: idioth
tags: [idioth, ctf, christmas ctf 2020, reversing, arm, crackme]
categories: [Write-Up]
date: 2020-12-29 21:00:00
cc: true
index_img: /2020/12/29/idioth/christmasctf2020-lock/image0.png
---

# 출제 의도

baby_crackme를 하나 간단하게 내고 싶었는데 arm으로 내면 어떨까? 싶어서 낸 문제입니다.



# Solution

dump로 파일이 하나 주어지며 vscode나 메모장 등으로 열면 아래와 같은 dump 코드가 나옵니다.

![](christmasctf2020-lock/image0.png)

aarch64에서 동작하는 바이너리이며 Input 값을 통해 어떠한 연산을 수행하는 것으로 볼 수 있습니다.

```
0000000000000c50 <main>:
 c50:   a9be7bfd    stp x29, x30, [sp, #-32]!
 c54:   910003fd    mov x29, sp
 c58:   d2800021    mov x1, #0x1                    // #1
 c5c:   d2800260    mov x0, #0x13                   // #19
 c60:   97fffee0    bl  7e0 <calloc@plt>
 c64:   f9000be0    str x0, [sp, #16]
 c68:   97ffff94    bl  ab8 <sub_ab8>
 c6c:   97ffff51    bl  9b0 <sub_9b0>
 c70:   f9000fe0    str x0, [sp, #24]
 c74:   97ffff3e    bl  96c <sub_96c>
 c78:   f9400be1    ldr x1, [sp, #16]
 c7c:   90000000    adrp    x0, 0 <_init-0x760>
 c80:   9138a000    add x0, x0, #0xe28              // #0xe28 '%s'
 c84:   97fffeeb    bl  830 <__isoc99_scanf@plt>
 c88:   f9400fe1    ldr x1, [sp, #24]
 c8c:   f9400be0    ldr x0, [sp, #16]
 c90:   97ffffa8    bl  b30 <sub_b30>
 c94:   f9400be0    ldr x0, [sp, #16]
 c98:   97fffee2    bl  820 <free@plt>
 c9c:   52800000    mov w0, #0x0                    // #0
 ca0:   a8c27bfd    ldp x29, x30, [sp], #32
 ca4:   d65f03c0    ret
```

main 함수를 보면 `sp + #16`에 calloc(0x13,1)을 해주고 `sub_ab8`함수가 호출된 후 나온 값을 인자로 `sub_9b0` 함수가 실행된 후 `sp + #24`에 저장합니다. 그 후 `sub_96c` 함수를 호출하고 값을 받아서`sp+#16`에 넣어주고  `sub_9b0(sub_ab8())` 한 값과 input 값을 인자로 `sub_b30` 함수를 호출하고 프로그램이 종료됩니다. 해당 함수를 c 코드로 간단하게 나타내면 아래와 같습니다.

```c
// main
int main()
{
    char *var1 = calloc(0x13, 1);
    int var2 = sub_9b0(sub_ab8());

    sub_96c();
    scanf("%s", var1);

    sub_b30(var1, var2);
    free(var1);

    return 0;
}
```

`sub_ab8`과 `sub_9b0` 함수를 살펴봅시다.

```
value0  DCB 0x96, 0x19, 0x7, 0x11, 0x99, 0x19, 0x2, 0x11
value1  DCB 0xF7, 0x7B, 0x64, 0x75, 0xFC, 0x7F, 0x65, 0x79, 0xFF,
0x73, 0x6C, 0x7D, 0xF4, 0x77, 0x6D, 0x61, 0xE7, 0x6B, 0x74, 0x65,
0xEC, 0x6F, 0x75, 0x69, 0xEF, 0x63, 0x46, 0x53, 0xDA, 0x5D, 0x47,
0x57, 0xD1, 0x51, 0x4E, 0x5B, 0xD2, 0x55, 0x4F, 0x5F, 0xD9, 0x49, 0x56,
0x43, 0xCA, 0x4D, 0x57, 0x47, 0xC1, 0x41, 0x5E, 0x4B, 0xA9, 0x28, 0x30,
0x22, 0xA2, 0x2C, 0x31, 0x26, 0xA1, 0x20

0000000000000ab8 <sub_ab8>:
 ab8:   a9be7bfd    stp x29, x30, [sp, #-32]!
 abc:   910003fd    mov x29, sp
 ac0:   52800103    mov w3, #0x8                    // #8
 ac4:   90000000    adrp    x0, value0@page
 ac8:   9136e002    add x2, x0, value0@pageoff
 acc:   528007c1    mov w1, #0x3e                   // #62
 ad0:   90000000    adrp    x0, value1@page
 ad4:   91372000    add x0, x0, value1@pageoff
 ad8:   97ffffc9    bl  9fc <sub_9fc>
 adc:   f9000fe0    str x0, [sp, #24]
 ae0:   f9400fe0    ldr x0, [sp, #24]
 ae4:   a8c27bfd    ldp x29, x30, [sp], #32
 ae8:   d65f03c0    ret
```

aarch64의 calling convention은 `x0`, `x1`, `x2`, `x3` 이므로 인자에 value1, 0x3e, value0, 0x8을 넣어 `sub_9fc`를 호출합니다. 호출하고 난 후 연산된 문자열을 return 해줍니다.

```
00000000000009b0 <sub_9b0>:
 9b0:   a9bd7bfd    stp x29, x30, [sp, #-48]!
 9b4:   910003fd    mov x29, sp
 9b8:   f9000fe0    str x0, [sp, #24]
 9bc:   f9400fe0    ldr x0, [sp, #24]
 9c0:   97ffff78    bl  7a0 <strlen@plt>
 9c4:   91000400    add x0, x0, #0x1
 9c8:   97ffff7e    bl  7c0 <malloc@plt>
 9cc:   f90017e0    str x0, [sp, #40]
 9d0:   f94017e0    ldr x0, [sp, #40]
 9d4:   f100001f    cmp x0, #0x0
 9d8:   54000061    b.ne    9e4 <sub_9b0+0x34>  // b.any
 9dc:   d2800000    mov x0, #0x0                // #0
 9e0:   14000005    b   9f4 <sub_9b0+0x44>
 9e4:   f9400fe1    ldr x1, [sp, #24]
 9e8:   f94017e0    ldr x0, [sp, #40]
 9ec:   97ffff95    bl  840 <strcpy@plt>
 9f0:   f94017e0    ldr x0, [sp, #40]
 9f4:   a8c37bfd    ldp x29, x30, [sp], #48
 9f8:   d65f03c0    ret
```

`sub_ab8`에서 반환한 문자열을 받아서 길이를 계산한 후 `malloc` 해줍니다. 할당한 주소가 0이라면 0을 반환해주고, 아니면 `strcpy`를 통해 `sub_ab8`에서 온 문자열을 복사하여 return 해줍니다.

이제 `sub_9fc` 함수를 분석을 해봅시다.

```
00000000000009fc <sub_9fc>:
 9fc:   a9bc7bfd    stp x29, x30, [sp, #-64]!
 a00:   910003fd    mov x29, sp
 a04:   f90017e0    str x0, [sp, #40]
 a08:   b90027e1    str w1, [sp, #36]
 a0c:   f9000fe2    str x2, [sp, #24]
 a10:   b90023e3    str w3, [sp, #32]
 a14:   b94027e0    ldr w0, [sp, #36]
 a18:   11000400    add w0, w0, #0x1
 a1c:   93407c00    sxtw    x0, w0
 a20:   97ffff68    bl  7c0 <malloc@plt>
 a24:   f9001fe0    str x0, [sp, #56]
 a28:   b98027e0    ldrsw   x0, [sp, #36]
 a2c:   f9401fe1    ldr x1, [sp, #56]
 a30:   8b000020    add x0, x1, x0
 a34:   3900001f    strb    wzr, [x0]
 a38:   b90037ff    str wzr, [sp, #52]
 a3c:   14000018    b   a9c <sub_9fc+0xa0>
 a40:   b98037e0    ldrsw   x0, [sp, #52]
 a44:   f94017e1    ldr x1, [sp, #40]
 a48:   8b000020    add x0, x1, x0
 a4c:   39400002    ldrb    w2, [x0]
 a50:   b94037e0    ldr w0, [sp, #52]
 a54:   b94023e1    ldr w1, [sp, #32]
 a58:   1ac10c03    sdiv    w3, w0, w1
 a5c:   b94023e1    ldr w1, [sp, #32]
 a60:   1b017c61    mul w1, w3, w1
 a64:   4b010000    sub w0, w0, w1
 a68:   93407c00    sxtw    x0, w0
 a6c:   f9400fe1    ldr x1, [sp, #24]
 a70:   8b000020    add x0, x1, x0
 a74:   39400001    ldrb    w1, [x0]
 a78:   b98037e0    ldrsw   x0, [sp, #52]
 a7c:   f9401fe3    ldr x3, [sp, #56]
 a80:   8b000060    add x0, x3, x0
 a84:   4a010041    eor w1, w2, w1
 a88:   12001c21    and w1, w1, #0xff
 a8c:   39000001    strb    w1, [x0]
 a90:   b94037e0    ldr w0, [sp, #52]
 a94:   11000400    add w0, w0, #0x1
 a98:   b90037e0    str w0, [sp, #52]
 a9c:   b94037e1    ldr w1, [sp, #52]
 aa0:   b94027e0    ldr w0, [sp, #36]
 aa4:   6b00003f    cmp w1, w0
 aa8:   54fffccb    b.lt    a40 <sub_9fc+0x44>  // b.tstop
 aac:   f9401fe0    ldr x0, [sp, #56]
 ab0:   a8c47bfd    ldp x29, x30, [sp], #64
 ab4:   d65f03c0    ret
```

인자로 받은 값을 차례대로 스택에 저장합니다. `w1`과 `w3`은 데이터의 길이로 추정되므로 `x0`, `x2`에 들어온 값을 보면 `[sp + #40] = \xFC\x7F\x65\x79\xFF\x73\x6C\x7D\xF4\x77\x6D\x61\xE7\x6B\x74\x65\xEC\x6F\x75\x69\xEF\x63\x46\x53\xDA\x5D\x47\x57\xD1\x51\x4E\x5B\xD2\x55\x4F\x5F\xD9\x49\x56\x43\xCA\x4D\x57\x47\xC1\x41\x5E\x4B\xA9\x28\x30\x22\xA2\x2C\x31\x26\xA1\x20`, `[sp + #24] = \x96\x19\x7\x11\x99\x19\x2\x11` 이 됩니다. 스택에 값을 저장한 후 첫 번째 인자의 길이+1 만큼 `malloc`을 해준 후 `sp+#56`에 저장합니다. 그 후 첫 번째 인자의 길이를 가져와서 `malloc`한 주소의 마지막 index에 0x0을 넣어주고 `sp+#52`에 0을 저장해준 후 `a9c`로 이동합니다.

`a9c`에서는 0을 `w1`에 넣고 첫 번째 문자열의 길이를 `w0`에 불러온 후 두 개를 비교하여 `a40`으로 이동합니다. `w1=w0`이 되면 `flag`가 `0`으로 세팅되고 `b.lt` 연산이 수행되지 않으므로 첫 번째 문자열의 길이만큼 반복하는 구간임을 알 수 있습니다. `a40`에서 `[sp+#40 + sp+#52]`의 값을 `w2`에 넣고 `w3`에는 `sp+#52 - sp+#32 * (sp+#52 / sp+#32)` 연산을 통해 `sp+#52 % sp+#32` 를 수행한 후 해당 주소의 값을 가져와서 두 문자열을 `xor` 연산합니다.

`sub_9fc` 를 c로 간단하게 나타내면 아래와 같습니다.

```c
// sub_9fc
char* sub_9fc(char* param1, int param2, char* param3, int param4)
{
    char* var1 = malloc(param2 + 1);
    var1[param2] = '\0';

    for(int i = 0; i < param2; i++)
    {
        var1[0] = (param1[i] ^ param3[i % param4]) & 0xff;
    }
    return var1;
}
```

해당 문자열들이 `sub_9fc`를 거쳐 어떠한 값이 나오는지 구하는 python 스크립트는 아래와 같습니다.

```python
string1 = [
    0xF7, 0x7B, 0x64, 0x75, 0xFC, 0x7F, 0x65, 0x79, 0xFF,
    0x73, 0x6C, 0x7D, 0xF4, 0x77, 0x6D, 0x61, 0xE7, 0x6B, 0x74, 0x65,
    0xEC, 0x6F, 0x75, 0x69, 0xEF, 0x63, 0x46, 0x53, 0xDA, 0x5D, 0x47,
    0x57, 0xD1, 0x51, 0x4E, 0x5B, 0xD2, 0x55, 0x4F, 0x5F, 0xD9, 0x49, 0x56,
    0x43, 0xCA, 0x4D, 0x57, 0x47, 0xC1, 0x41, 0x5E, 0x4B, 0xA9, 0x28, 0x30,
    0x22, 0xA2, 0x2C, 0x31, 0x26, 0xA1, 0x20
]

string2 = [0x96, 0x19, 0x7, 0x11, 0x99, 0x19, 0x2, 0x11]
result = ""

for i in range(len(string1)):
    result += chr(string1[i] ^ string2[i % len(string2)])

print(result)

'''
C:\Users\idioth\Desktop>lock.py
abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789
'''
```

`sub_9fc`까지 마쳤으니 다시 main으로 돌아가죠!

```c
// main
int main()
{
    char *var1 = calloc(0x13, 1);
    int var2 = sub_9b0(sub_ab8());

    sub_96c();
    scanf("%s", var1);

    sub_b30(var1, var2);
    free(var1);

    return 0;
}
```

이제 입력 값을 받아서 `sub_b30`하는 것만 남았네요. `sub_b30` 함수를 확인해봅시다.

```
0000000000000b30 <sub_b30>:
 b30:   a9be7bfd    stp x29, x30, [sp, #-32]!
 b34:   910003fd    mov x29, sp
 b38:   f9000fe0    str x0, [sp, #24]
 b3c:   f9000be1    str x1, [sp, #16]
 b40:   f9400fe0    ldr x0, [sp, #24]
 b44:   91001800    add x0, x0, #0x6
 b48:   39400001    ldrb    w1, [x0]
 b4c:   f9400be0    ldr x0, [sp, #16]
 b50:   91003800    add x0, x0, #0xe
 b54:   39400000    ldrb    w0, [x0]
 b58:   6b00003f    cmp w1, w0
 b5c:   54000761    b.ne    c48 <sub_b30+0x118>  // b.any
 b60:   f9400fe0    ldr x0, [sp, #24]
 b64:   91001000    add x0, x0, #0x4
 b68:   39400001    ldrb    w1, [x0]
 b6c:   f9400be0    ldr x0, [sp, #16]
 b70:   91000c00    add x0, x0, #0x3
 b74:   39400000    ldrb    w0, [x0]
 b78:   6b00003f    cmp w1, w0
 b7c:   54000661    b.ne    c48 <sub_b30+0x118>  // b.any
 b80:   f9400fe0    ldr x0, [sp, #24]
 b84:   91000c00    add x0, x0, #0x3
 b88:   39400001    ldrb    w1, [x0]
 b8c:   f9400be0    ldr x0, [sp, #16]
 b90:   91002800    add x0, x0, #0xa
 b94:   39400000    ldrb    w0, [x0]
 b98:   6b00003f    cmp w1, w0
 b9c:   54000561    b.ne    c48 <sub_b30+0x118>  // b.any
 ba0:   f9400fe0    ldr x0, [sp, #24]
 ba4:   91001c00    add x0, x0, #0x7
 ba8:   39400001    ldrb    w1, [x0]
 bac:   f9400be0    ldr x0, [sp, #16]
 bb0:   91004400    add x0, x0, #0x11
 bb4:   39400000    ldrb    w0, [x0]
 bb8:   6b00003f    cmp w1, w0
 bbc:   54000461    b.ne    c48 <sub_b30+0x118>  // b.any
 bc0:   f9400fe0    ldr x0, [sp, #24]
 bc4:   91000400    add x0, x0, #0x1
 bc8:   39400001    ldrb    w1, [x0]
 bcc:   f9400be0    ldr x0, [sp, #16]
 bd0:   91003800    add x0, x0, #0xe
 bd4:   39400000    ldrb    w0, [x0]
 bd8:   6b00003f    cmp w1, w0
 bdc:   54000361    b.ne    c48 <sub_b30+0x118>  // b.any
 be0:   f9400fe0    ldr x0, [sp, #24]
 be4:   91001400    add x0, x0, #0x5
 be8:   39400001    ldrb    w1, [x0]
 bec:   f9400be0    ldr x0, [sp, #16]
 bf0:   9100d000    add x0, x0, #0x34
 bf4:   39400000    ldrb    w0, [x0]
 bf8:   6b00003f    cmp w1, w0
 bfc:   54000261    b.ne    c48 <sub_b30+0x118>  // b.any
 c00:   f9400fe0    ldr x0, [sp, #24]
 c04:   39400001    ldrb    w1, [x0]
 c08:   f9400be0    ldr x0, [sp, #16]
 c0c:   9100d400    add x0, x0, #0x35
 c10:   39400000    ldrb    w0, [x0]
 c14:   6b00003f    cmp w1, w0
 c18:   54000181    b.ne    c48 <sub_b30+0x118>  // b.any
 c1c:   f9400fe0    ldr x0, [sp, #24]
 c20:   91000800    add x0, x0, #0x2
 c24:   39400001    ldrb    w1, [x0]
 c28:   f9400be0    ldr x0, [sp, #16]
 c2c:   91000800    add x0, x0, #0x2
 c30:   39400000    ldrb    w0, [x0]
 c34:   6b00003f    cmp w1, w0
 c38:   54000081    b.ne    c48 <sub_b30+0x118>  // b.any
 c3c:   f9400fe0    ldr x0, [sp, #24]
 c40:   97ffffab    bl  aec <sub_aec>
 c44:   d503201f    nop
 c48:   a8c27bfd    ldp x29, x30, [sp], #32
 c4c:   d65f03c0    ret
```

위에서 복호화한 `[a-zA-Z0-9]` 값과 입력 값을 통해 하나씩 비교한 후 조건이 모두 맞으면 `x0`에  input 값을 넣고 `sub_aec`를 호출합니다. 값을 비교하는 부분을 간단하게 정리하면 아래와 같습니다.

- var1[0x6] == var2[0xe]
- var1[0x4] == var2[0x3]
- var1[0x3] == var2[0xa]
- var1[0x7] == var2[0x11]
- var1[0x1] == var2[0xe]
- var1[0x5] == var2[0x34]
- var1[0x0] == var2[0x35]
- var1[0x2] == var2[0x2]

순서에 맞춰서 배열하면 input 값을 구할 수 있습니다.

```python
user_input = ""
user_input += result[0x35] + result[0xe] + result[0x2] + result[0xa] + result[0x3] +
result[0x34] + result[0xe] + result[0x11]

print(user_input)

'''
C:\Users\idioth\Desktop>lock.py
abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789
1ockd0or
'''
```

조건이 맞는 입력 값을 인자로 `sub_aec` 값을 호출하니 해당 함수를 확인해보면 아래와 같습니다.

```
value2  DCB 0x69, 0x22, 0x22, 0x38, 0x1F, 0x43, 0x5B, 0x1C, 0x45, 0xE, 0x3C,
0x8, 0x5, 0x5E, 0x30, 0x17, 0x5F, 0x1B, 0x6, 0x19, 0x3B, 0x44, 0x7, 0x17,
0x6E, 0x7, 0x53, 0x1E, 0x17, 0x55, 0x12

0000000000000aec <sub_aec>:
 aec:   a9bd7bfd    stp x29, x30, [sp, #-48]!
 af0:   910003fd    mov x29, sp
 af4:   f9000fe0    str x0, [sp, #24]
 af8:   f9400fe0    ldr x0, [sp, #24]
 afc:   97ffff29    bl  7a0 <strlen@plt>
 b00:   2a0003e3    mov w3, w0
 b04:   f9400fe2    ldr x2, [sp, #24]
 b08:   528003e1    mov w1, #0x1f                   // #31
 b0c:   90000000    adrp    x0, value2@page
 b10:   91382000    add x0, x0, value2@pageoff
 b14:   97ffffba    bl  9fc <sub_9fc>
 b18:   f90017e0    str x0, [sp, #40]
 b1c:   f94017e0    ldr x0, [sp, #40]
 b20:   97ffff3c    bl  810 <puts@plt>
 b24:   d503201f    nop
 b28:   a8c37bfd    ldp x29, x30, [sp], #48
 b2c:   d65f03c0    ret
```

입력 값과 value2를 인자로 `sub_9fc`를 수행하는 것을 확인할 수 있습니다. `sub_9fc`의 경우 아까 python script를 짜 놓았기 때문에 그냥 값을 입력하여 연산하면 됩니다.

```python
string3 = [
    0x69, 0x22, 0x22, 0x38, 0x1F, 0x43, 0x5B, 0x1C, 0x45, 0xE, 0x3C,
0x8, 0x5, 0x5E, 0x30, 0x17, 0x5F, 0x1B, 0x6, 0x19, 0x3B, 0x44, 0x7, 0x17,
0x6E, 0x7, 0x53, 0x1E, 0x17, 0x55, 0x12
]
flag = ""
for i in range(len(string3)):
    flag += chr(string3[i] ^ ord(user_input[i % len(user_input)]))

print(flag)

'''
C:\Users\idioth\Desktop>lock.py
abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789
1ockd0or
XMAS{s4nta_can_enter_the_h0use}
'''
```

FLAG : XMAS{s4nta_can_enter_the_h0use}