---
title: "[해키피디아] LFI(Local File Inclusion)"
author: y00n_nms
tags: [y00n_nms, LFI]
categories: [Hackypedia]
date: 2021-02-16 14:00:00
cc: true
index_img: /img/hackypedia.png
---
File Inclusion 취약점은 파라미터 값을 정확하게 검사하지 않을 때에 해커가 피해자 서버에서 파일을 읽고 때때로 실행할 수 있도록 하거나 서버에 악성 스크립트 또는 웹쉘을 만들고 웹 사이트 변조에 사용할 수 있습니다.
그 중 LFI(Local File Inclusion)은 **공격 대상 서버에 위치한 로컬 파일**을 노출하거나 실행할 수 있습니다. 즉, include() 사용 시 입력에 대해 필터링하지 않아 include 문에서 로컬 파일을 사용할 수 있습니다.
단순한 정보 공개를 포함해 원격 코드 실행, 또는 XSS(Cross Site Scripting), Directory Traversal 등의 공격으로 이어질 수 있습니다.

```c
$file = $_GET['file'];
include('directory/' . $file);
```
최악의 시나리오의 예로, 위의 코드에서 GET으로 특정 파라미터를 이용하여 include가 수행될 때 해커는 `http://example.com/?file=../../uploads/attack.php`과 같은 요청을 하여 웹쉘을 업로드할 수 있습니다. 하지만 항상 악성 파일을 애플리케이션에 업로드할 수 있는 것은 아닙니다. 저장하더라도 LFI 취약점이 존재하는 동일한 서버에 애플리케이션이 파일을 저장한다는 보장은 없습니다.
파일를 업로드하고 실행할 수 있는 기능이 없더라도 LFI 취약점은 위험할 수 있습니다. 해커는 여전히 다음과 같이 LFI 취약점을 이용하여 Directory Traversal(Path Traversal) 공격을 수행할 수 있습니다.
```c
<http://example.com/?file=../../../../etc/passwd>
```
위의 예에서 해커는 서버의 사용자 목록이 포함된 /etc/passwd 파일의 내용을 가져올 수 있습니다. 마찬가지로 해커는 Directory Traversal 취약점을 이용하여 로그 파일(예: Apache access.log 또는 error.log), 소스 코드 및 기타 민감한 정보에 액세스할 수 있습니다. 이후에 이 정보를 사용하여 공격을 진행할 수 있습니다.