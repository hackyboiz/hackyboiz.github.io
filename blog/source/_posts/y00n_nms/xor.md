---
title: "[해키피디아] XOR(Exclusive OR), shift 연산"
author: y00n_nms
tags: [y00n_nms, xor, otp]
categories: [Hackypedia]
date: 2021-11-03 14:00:00
cc: true
index_img: /img/hackypedia.png
---

**XOR(Exclusice OR, 배타적 논리합)연산**을 벤다이어그램으로 나타내면 다음과 같으며, XOR, xor, ^, $\oplus$ 로 표시됩니다. 비트간의 배타적 논리합 연산은 두 개의 비트 중 하나만 1일때 1의 결과가 나오며, 이는 두 수를 더해 2로 나누는 나머지와 같습니다. ($mod\space 2$)

![image](xor/image.png)

같은 비트로 연산하면 0이 되는 점($A\oplus A = 0$)을 이용해 $A \oplus (B\oplus B) = A$로 사용될 수 있으며, 이러한 특징으로 One Time Pad에서 암호화와 복호화에 사용됩니다.

$$P = C \oplus K = (P \oplus K) \oplus K = P \oplus (K \oplus K) = P$$



**Shift 연산**은 비트를 오른쪽(`>>`), 또는 왼쪽(`<<`)으로 옮기는 연산을 수행합니다.

`변수` `<<` `이동할 비트 수`의 형식으로 예를 들어 `<< 1`은 왼쪽으로 1개씩 모두 옮기며 남은 비트에는 0을 채웁니다.

![image1](xor/image1.png)