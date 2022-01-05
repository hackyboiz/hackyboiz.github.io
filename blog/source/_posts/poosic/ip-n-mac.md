---
title: "[해키피디아] IP주소&MAC 주소"
author: poosic
tags: [poosic, ip, mac]
categories: [Hackypedia]
date: 2022-01-05 14:00:00
cc: true
index_img: /img/hackypedia.png

---

## IP 주소

IP 주소란 IP 통신에 필요한 고유 주소를 말합니다. IPv4(Internet Protocol version 4)와 IPv6 (Internet Protocol version 6),이 두 가지 체계가 있으며 우리가 흔히 127.0.0.1 처럼 4개의 필드에 0~255의 10진수가 들어가 있는 IP 주소는 IPv4의 형식입니다.

## MAC주소

MAC 주소란 무선 LAN 카드, 무선 LAN 기능 내장 기기에 부여되는 고유 식별 번호입니다. 즉, IP주소가 네트워크의 주소라면 MAC 주소는 하드웨어의 주소(=하드웨어 식별자)라고 할 수 있습니다.

### 사용 이유

MAC주소는 간단히 말하자면 통신과정에서 정확한 대상을 특정짓기 위해 사용합니다. 반대로 말하자면 사용자가 필요한 정보만을 통신하기 위해 사용하는 것입니다.

![Untitled.jpg](ip-n-mac/image1.png)

예시로 택배를 보낼 때를 생각해보겠습니다.택배 수신자가 아파트에 사는 경우 송신자 우린 상세 주소를 작성해서 보냅니다. 만약 "서울특별시 중랑구 오동동 여기로 300 우리아파트 107동 504호" 주소에 택배를 보낸다고 하면 "서울특별시~ 여기로 300"은 IP주소에, "107동 504호"는 MAC 주소에 비유할 수 있습니다. 이런 MAC 주소와 IP주소에 상관없이 모든 정보를 받아들이면, 정보가 가지 말아야 할 곳에 전달될수도 있습니다. 실제로 Monitor Mode라는 것이 존재합니다 기본적으로 무선 랜카드나 어댑터는 Managed Mode라는 자신에게 오는 패킷만을 수신하는 모드로 설정되어 있습니다만 Monitor Mode는 주변에 뿌려지고 있는 모든 패킷을 수신하는 모드입니다. 그렇기 때문에 본래는 무선 트래픽 분석 등 WLAN 유지/관리를 위한 목적으로 만들어졌지만 [스니핑](https://stibee.com/api/v1.0/emails/share/uXvtk4FQZNF4WMDqKBGeYjiJRTp_CA==)에 악용이 충분히 가능하게 됩니다. 