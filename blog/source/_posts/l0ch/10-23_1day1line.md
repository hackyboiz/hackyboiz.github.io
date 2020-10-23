---
title: "[하루한줄] Bypassing NTFS permissions to read any files as unprivileged user"
author: L0ch
tags: [L0ch, cve, arbitrary file read, windows, ntfs]
categories: [1day1line]
date: 2020-10-23 18:00:00
cc: true
index_img: /img/1day1line.png
---

## URL
[POC using Windows API calls](https://github.com/ioncodes/CVE-2020-16938)



## Target
Windows 10 2004/Server 2004



## Explain
권한이 없는 일반 사용자가 로컬 드라이브의 전체 데이터를 열람할 수 있는 취약점의 POC가 공개되었습니다. 
해당 취약점은 최근 업데이트로 파티션, 볼륨 장치에 대한 권한이 변경되어 ``\\.\PhysicalDrive0\``경로로 장치에 접근하면 권한을 우회해 arbitrary file read가 가능합니다. 

![image1](10-23_1day1line/image1.png)

NTFS parser인 7zip으로도 arbitrary file read 가 가능합니다.

## reference

https://twitter.com/jonasLyk/status/1316104870987010048