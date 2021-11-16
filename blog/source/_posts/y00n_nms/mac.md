---
title: "[해키피디아] MAC(Message Authentication Code)"
author: y00n_nms
tags: [y00n_nms, mac, reply attack]
categories: [Hackypedia]
date: 2021-11-16 14:00:00
cc: true
index_img: /img/hackypedia.png

---

**MAC(Message Authentication Code)** 알고리즘은 메시지의 **무결성과 인증성**(출처가 분명함)을 보호합니다. 송신자와 수신자가 공유하는 키 K, 공유하고자 하는 메세지 M으로  `T = MAC(K, M)`이라는 값을 생성하여 T를 M의 인증값(Authentication tag) 또는 단순히 MAC이라고 합니다.

Alice와 Bob이 키 K를 공유하고 있을 때, Bob이 Alice에게 메세지 M과 T=MAC(K, M), 즉 인증값(MAC)을 계산하여 보내면 Alice는 받은 메세지로 MAC(K, M)을 계산하여 받은 인증값과 동일한지 확인합니다.

동일하다면 받은 메세지가 중간에 변조되지 않았다는 무결성과, 키 K는 Alice와 Bob만 공유하기 때문에 메세지를 보낸 것이 Bob이라는 인증성을 확인할 수 있습니다.

Alice가 Bob에게 보낸 메세지 M과 인증값 MAC을 만약 중간에 공격자 Trudy가 도청 등으로 얻게된다면, 다시 Bob에게 M과 MAC을 보내 Alice인 척할 수 있습니다. 이를 재전송 공격(reply attack)이라고 하며, 막기 위하여 다음 세 가지 방법을 사용합니다.

1. sequence number: 메세지를 보낼때 마다 1씩 증가하는 메세지 순서 번호를 넣는다.
2. time stamp: 메세지를 보낼 때 현재 시각을 함께 넣는다.
3. nonce: 수신자는 송신자에게 일회용으로 랜덤 값을 주어 해당 논스로 MAC을 계산한다.