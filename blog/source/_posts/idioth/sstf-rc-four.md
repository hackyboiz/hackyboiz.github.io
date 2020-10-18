---
title: "[Write-Up] SSTF RC_four"
author: idioth
tags: [idioth, samsung, ctf, sstf, rc4, crypto, decrypt]
categories: [Write-Up]
date: 2020-10-18 15:00:00
cc: true
index_img: /2020/10/18/idioth/sstf-rc-four/image.png
---



# [Wirte-Up] SSTF RC_four

## Intro

![](sstf-rc-four/image.png)

압축을 풀면 challenge.py와 output.txt 파일을 볼 수 있습니다.

![](sstf-rc-four/image1.png)

output.txt 파일에는 암호화된 것으로 추측되는 문자열이 2줄 존재합니다.



## challenge.py 분석

```python
from Crypto.Cipher import ARC4
from secret import key, flag
from binascii import hexlify

#RC4 encrypt function with "key" variable.
def encrypt(data):
	#check the key is long enough
	assert(len(key) > 128)

	#make RC4 instance
	cipher = ARC4.new(key)

	#We don't use the first 1024 bytes from the key stream.
	#Actually this is not important for this challenge. Just ignore.
	cipher.encrypt("0"*1024)

	#encrypt given data, and return it.
	return cipher.encrypt(data)

msg = "RC4 is a Stream Cipher, which is very simple and fast."

print (hexlify(encrypt(msg)).decode())
print (hexlify(encrypt(flag)).decode())
```

challenge 파일을 해보면 arc4를 사용하여 암호화를 진행한 것을 알 수 있습니다.

output.txt의 첫번째 줄은 msg를 암호화한 부분이고 두번째 줄은 flag임을 알 수 있습니다.

[rc4 알고리즘](https://en.wikipedia.org/wiki/RC4)은 스트림 암호로 key 값을 사용하여 셔플링을 통해 키 스트림 바이트를 생성한 후 해당 키 스트림과 xor 연산을 통해 암호화를 진행합니다.

key 값을 사용하여 생성된 key stream과 문자열을 xor 연산하여 최종 암호문이 나오는 것을 활용하면 key 값을 알지 못해도 flag 암호문을 복호화할 수 있습니다.

key stream ^ 문자열 = 암호문이므로 암호문 ^ 문자열을 수행하면 key stream을 알 수 있고 해당 key stream과 flag 암호문을 xor 연산을 수행하면 flag 값을 얻을 수 있습니다.



## Decrypt Code

```python
text = "RC4 is a Stream Cipher, which is very simple and fast."
result = [0x63, 0x4c, 0x33, 0x23, 0xbd, 0x82, 0x58, 0x1d, 0x9e, 0x5b, 0xbf, 0xaa, 0xeb, 0x17, 0x21, 0x2e, 0xeb, 0xfc, 0x97, 0x5b, 0x29, 0xe3, 0xf4, 0x45, 0x2e, 0xef, 0xc0, 0x8c, 0x09, 0x06, 0x33, 0x08, 0xa3, 0x52, 0x57, 0xf1, 0x83, 0x1d, 0x9e, 0xb8, 0x0a, 0x58, 0x3b, 0x8e, 0x28, 0xc6, 0xe4, 0xd2, 0x02, 0x8d, 0xf5, 0xd5, 0x3d, 0xf8]
stream = [0 for i in range(len(text))]
dec_flag = [0x62, 0x4c, 0x53, 0x45, 0xaf, 0xb3, 0x49, 0x4c, 0xdd, 0x63, 0x94, 0xbb, 0xbf, 0x06, 0x04, 0x3d, 0xda, 0xca, 0xd3, 0x5d, 0x28, 0xce, 0xed, 0x11, 0x2b, 0xb4, 0xc8, 0x82, 0x3e, 0x45, 0x33, 0x2b, 0xeb, 0x41, 0x60, 0xdc, 0xa8, 0x62, 0xd8, 0xa8, 0x0a, 0x45, 0x64, 0x9f, 0x7a, 0x96, 0xe9, 0xcb]
flag = ""

for i in range(len(text)):
    stream[i] = str(int(ord(text[i]) ^ result[i]))

for i in range(len(dec_flag)):
    flag += str(chr(int(stream[i]) ^ dec_flag[i]))

print(flag)
```

![](sstf-rc-four/image2.png)