---
title: "[하루한줄] CVE-2020-6449: Exploiting a textbook use-after-free in Chrome"
author: L0ch
tags: [L0ch, cve, use after free, rce, chrome, chromium]
categories: [1day1line]
date: 2020-10-30 18:00:00
cc: true
index_img: /img/1day1line.png
---

## URL
[Exploiting a textbook use-after-free in Chrome](https://securitylab.github.com/research/CVE-2020-6449-exploit-chrome-uaf)



## Target
Chrome version: master branch build 79956ba, asan build 80.3987.132 Operating System: Ubuntu 18.04



## Explain
CVE-2020-6449는 Chrome에서 사용하는 blink 엔진의 `WebAudio` 모듈에서 발생하는 Use-After-Free 취약점 입니다. 

취약점은 `DeferredTaskHandler::BreakConnections` 에서 발생합니다. 일반적으로 `active_source_handlers_`는 `finished_source_handlers_` 의 원시 포인터를 활성 상태로 유지하는 역할을 하며 사용이 완료된 이후에는 할당된 `active_source_handlers_` 와 `finished_source_handlers_` 가 같이 free되어야 합니다.

그러나 컨텍스트를 삭제하여 `BaseAudioContext::Uninitialize` 가 실행된 이후 `DeferredTaskHandler::ClearHandlersToBeDeleted` 를 호출하면 `active_source_handlers_` 만 free되고 `finished_source_handlers_` 에 free된 포인터가 남게 됩니다. 이후 `DeferredTaskHandler::BreakConnections` 를 호출해 UAF 취약점을 트리거할 수 있습니다.



## Reference
[https://securitylab.github.com/research/garbage-collection-uaf-chrome_gc](https://securitylab.github.com/research/garbage-collection-uaf-chrome_gc)

[https://securitylab.github.com/advisories/GHSL-2020-040-chrome](https://securitylab.github.com/advisories/GHSL-2020-040-chrome)

