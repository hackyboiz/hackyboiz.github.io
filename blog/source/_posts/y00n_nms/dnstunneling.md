---
title: "[해키피디아] DNS Tunneling"
author: y00n_nms
tags: [y00n_nms, dns]
categories: [Hackypedia]
date: 2022-03-23 14:00:00
cc: true
index_img: /img/hackypedia.png
---

DNS(Domain Name Service)는 간단히 말해서 인터넷의 전화번호부입니다. 대부분의 사용자는 방문하려는 웹사이트의 도메인 이름이나 URL을 입력하지만, 서버는 IP주소를 사용하기 때문에 DNS는 도메인 이름과 IP 주소 간의 변환을 제공합니다. 때문에 DNS 트래픽은 인터넷에서 신뢰받는 트래픽 중 하나로, 방화벽을 모두 통과하도록 허용합니다.

DNS Tunneling은 이를 악용해 DNS 쿼리 내에 악성 페이로드를 숨겨 해커가 피해자의 컴퓨터에 악성코드에 대한 C2 채널을 생성할 수 있는 기술입니다. 인바운드 DNS 트래픽은 악성코드에 명령을 전달할 수 있고, 아웃바운드 트래픽은 민감한 데이터를 유출하거나 해커의 요청에 대한 응답을 제공할 수 있습니다.