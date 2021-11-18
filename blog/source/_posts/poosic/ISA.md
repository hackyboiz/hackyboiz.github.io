---
title: "[해키피디아] ISA(Instruction Set Architecture)"
author: poosic
tags: [poosic, isa, instruction set]
categories: [Hackypedia]
date: 2021-11-19 14:00:00
cc: true
index_img: /img/hackypedia.png
---

## ISA

ISA(Instruction Set Architecture)는 CPU가 기능을 이해하고 실행할 수 있는 0과 1로 이루어진 기계어 명령의 집합을 의미합니다. 메모리 구조, 예외 처리 등과 함께 컴퓨터 구조의 일부로써 컴퓨터의 작동에 필요한 사칙연산, 논리연산, 데이터 전송, 실행제어, 인터럽트 등 다양한 동작의 명령어들을 포함하고 있습니다. 대표적인 ISA의 예시로는 IA-32, IA-64, AMD64, MIPS 등이 존재합니다.

## ISA가 다양한 이유

ISA가 다양한 이유는 CPU가 다양하기 때문인데요. CPU마다 구조가 다르므로 어떤 동작에 대해서 처리하는 방식도 달라질 수 있습니다. 그래서 각 CPU마다 그에 맞는 ISA가 존재해야 합니다. 그렇다면 CPU는 왜 다양한 걸까요?

CPU는 다양한 환경에서 다양한 형태로 필요로 해서 여러 가지 CPU가 존재합니다. 예를 들어 데스크탑에는 쿨링 팬을 장착할 공간이 있어서 발열이 심하더라도 성능이 좋은 CPU를 필요로 하고, 스마트폰과 같은 기기에는 쿨링 팬을 장착할 공간이 없어서 성능이 다소 떨어지더라도 발열이 심하지 않은 CPU를 요구하겠죠. 그래서 다양한 CPU 들이 존재하고 그에 맞는 다양한 ISA 들이 필요로 하게 된 것이죠.

