---
title: "[하루한줄] Discord Desktop app RCE"
author: L0ch
tags: [L0ch, context isolation, javascript, xss, cve, rce, discord, electron]
categories: [1day1line]
date: 2020-10-20 18:00:00
cc: true
index_img: /img/1day1line.png
---

## URL
[Discord Desktop app RCE](https://mksben.l0.cm/2020/10/discord-desktop-rce.html)



## Target
Discord Desktop app



## Explain
node.js 기반 오픈소스 프레임워크인 Electron으로 개발된 Discord에서 원격 코드 실행 취약점이 발견되었습니다. 

Electron의 `BrowserWindow API` 옵션 중 코드 실행 컨텍스트를 분리하는 `contextIsolation` 옵션이 discord에서는 비활성화 돼있어 injection된 JS가 내부 페이지 코드 실행에 영향을 줄 수 있습니다. 이를 이용하면 iframe embed XSS 를 통해 JavaScript 내장 메소드를 재정의할 수 있으며 이때 Navigation restriction을 우회하면 iframe을 escape해 최상위 컨텍스트에서 임의 코드를 실행할 수 있습니다.
