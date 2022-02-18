---
title: "[해키피디아] RFI(Remote File Include)"
author: poosic
tags: [poosic, rfi, lfi, include]
categories: [Hackypedia]
date: 2022-02-18 14:00:00
cc: true
index_img: /img/hackypedia.png
---

RFI(Remote File Inclusion)는 이전 해키피디아의 [LFI](https://hackyboiz.github.io/2022/02/16/y00n_nms/LFI/)와 같이 File Inclusion 취약점의 한 종류입니다.  LFI가 공격 대상 서버에 위치한 로컬 파일을 노출하거나 실행할 수 있는 취약점이었다면 RFI는 악성 스크립트를 공격 대상 서버에게 전달해 해당 스크립트가 취약한 페이지를 통해 실행할 수 있는 취약점 입니다. 즉, 특정 작업을 처리하는 페이지에 **원격지(Remote) 파일을 포함시킬 수 있는** 취약점입니다.

```php
$file = $_GET["file"];
include($file."_Web.php");
```

예시 코드처럼 `GET`으로 특정 파라미터를 통해 `include()` 구문이 사용될 때 해커는 악성 스크립트를 게시판이나 블로그와 같이 다운로드가 가능한 원격지에 업로드한 후 다운로드 링크를 다음과 같이 삽입해 공격 대상서버에 삽입이 가능합니다.

