---
title: "[해키피디아] IP주소&MAC 주소"
author: poosic
tags: [poosic, ip, mac]
categories: [Hackypedia]
date: 2022-01-05 14:00:00
cc: true
index_img: /img/hackypedia.png

---

## Daemon

Daemon은 사용자가 직접 제어하지 않고 백그라운드에서 여러 작업을 하는 프로그램을 말합니다. 즉, Daemon은 백그라운드에서 시스템과 관련된 활동을 하는 프로세스들을 말합니다. Window에서는 서비스라고 부르는 것들이 Daemon의 역할을 한다고 할 수 있습니다. 그렇다면 Daemon은 어떻게 만들어지고 수많은 프로세스중 Daemon을 어떻게 구분할 수 있을까요?

일반적으로 PPID가 1인 프로세스 트리에서 init의 자식 프로세스를 Daemon이라고 합니다. 이러한 특징을 가지는 이유는 Daemon이 만들어지는 과정과 관련이 있습니다.

![Untitled.jpg](daemon/image1.png)

Daemon은 일반적으로 fork off and die 라는 방식을 통해 만들어집니다. 여기서 fork란 여기서 말하는 fork란 프로세스가 자기 자신을 복제하는 동작을 말하는데요. fork off and die는 ‘복제하고 죽다.’라고 해석할 수 있습니다. 뜻처럼 프로세스를 복제한 뒤 원래 프로세스는 삭제되고 복제된 프로세스는 init 프로세스를 부모로 삼는 방식을 말합니다.

### 작동방식

Daemon은 백그라운드에서 작동되고 사용자가 직접 제어하지 않습니다. 또한, 평소 CPU를 할당받지 않고 메모리에 상주하다가 호출이 되는 경우에만 작동합니다. 이렇게 독립적으로 실행되어 항상 Daemon이 메모리에 상주하는 실행방식을 Stand alone 방식이라고 합니다. 때문에 부팅 시 자동으로 실행되는 Daemon들은 Stand alone 방식의 Daemon이라고 할 수 있습니다.

하지만 사용되지 않는 Daemon이 계속해서 메모리에 상주해 있다면 자원 낭비이기 때문에 Super Daemon이라는 것이 등장하게 됩니다. Super Daemon은 여러 가지 Daemon을 제어하며 필요에 따라 메모리에 적재할 일반 Daemon들을 관리해 자원 낭비를 줄입니다. 이렇게 Super Daemon에 의해 관리되는 방식을 Xinted(eXinted Interent Service Daemon)방식이라고 부릅니다. Super Daemon에 의해 필요한 경우에만 메모리에 적재되어 사용되기 때문에 자원에 대한 부담은 줄어들지만, 응답속도는 이미 메모리에 적재된 Daemon을 사용하는 Stand alone 방식에 비해 느립니다.

1. fork 호출 →자식프로세스를 생성

2. 부모 프로세스 종료
3. setsid 호출→ 새로운 세션 생성,

4. 현재프로세스(자식)의 PID가 세션의 제어권을 소유하도록 변경.
5. chdir 호출→ 루트 디렉터리를 현재  작업 디렉터리로 변경.