---
title: "[하루한줄] GfxDownloadWrapper.exe LOLBIN을 이용한 Assembly Loading"
author: idioth
tags: [idioth, lolbin, lolbas, living of the land]
categories: [1day1line]
date: 2020-10-22 22:00:00
cc: true
index_img: /img/1day1line.png
---

## URL 

[Exploring an Assembly Loading Technique and Detection Mechanism for the GfxDownloadWrapper.exe LOLBIN](https://bohops.com/2020/10/21/exploring-an-assembly-loading-technique-and-detection-mechanism-for-the-gfxdownloadwrapper-exe-lolbin/)



## Target

Windows



## Explain

GfxDownloadWrapper.exe는 인텔 비디오 카드 드라이버 소프트웨어로 인텔 그래픽 컨트롤 패널 및 게임 그래픽 설정을 지원하는 .NET 응용 프로그램입니다. 해당 파일은 "Microsoft Windows Third Party Component CA 2012"로 인증되어 있습니다. GfxDownloadWrapper.exe는 임의 다운로드를 방지하기 위한 루틴이 존재하지만 낮은 버전에서는 그러한 검증이 없습니다.

`main()` 진입점에서 어셈블리를 로드하는 부분을 발견할 수 있으며 인자를 조작함으로써 영향을 미칠 수 있습니다.

- args[0]: "run"의 문자열 값
- args[1]: 어셈블리 DLL 페이로드 경로
- args[2]: 필수 어셈블리 method 숫자 문자열 값
  - 0: `ApplyRecommendedSettings`, 1: `RestoreRecommendedSettings`, 2: `CacheCleanup`
- args[3]: `AppData` 상대 경로에 있는 게임 식별자 값

```powershell
GfxDownloadWrapper.exe "run" "payload.dll" "MethodNumber" ";AppData\\Local\\Intel\\Games\\임의 값"
```
따라서 인자를 맞춰준 후 DLL 경로를 악성 dll로 지정하면 악성 어셈블리를 로드한 후 성공적으로 실행하게 됩니다.

