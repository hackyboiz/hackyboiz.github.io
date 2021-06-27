---
title: "[Translation] Measured Boot와 멀웨어 시그니처: Windows Loader에서 발견된 두 가지 취약점"
author: L0ch
tags: [L0ch, boot security, windows, malware, cve, kernel]
categories: [Translation]
date: 2021-06-27 18:00:00
cc: false
index_img: 2021/06/27/l0ch/measured-boot-vuln/thumbnail.png
---

안녕하세요 L0ch입니다!

오늘은 Windows Kernel 및 Loader에서 발견된 취약점 두 개를 들고 왔는데요, 원래는 하루한줄로 간단하게 정리하려다 접할 기회가 많이 없는 Boot Security에 대한 설명 등 취약점의 이해를 돕기 위한 내용이 잘 정리되어 있더라구요. 개인적으로 재밌게 읽어서 정리도 할 겸 번역글로 남겨봤습니다!

오역과 의역덩어리 번역글이니 지적은 매우 환영이에요..!

> 원문 : [Measured Boot and Malware Signatures: exploring two vulnerabilities found in the Windows loader](https://bi-zone.medium.com/measured-boot-and-malware-signatures-exploring-two-vulnerabilities-found-in-the-windows-loader-5a4fcc3c4b66)

---

# Introduction to Boot Security

Boot Security에는 Verified Boot와 Measured Boot라는 두 가지 주요 개념이 있다.



## Verified Boot

Verified Boot 프로세스는 신뢰할 수 있는 디지털 서명이 되지 않은 구성 요소가 부팅 중에 실행되지 않도록 한다. 이 프로세스는 서명되지 않았거나 제대로 서명되지 않은 부팅 구성 요소 (예 : 부팅 관리자 및 펌웨어 드라이버)가 시스템에서 실행되는 것을 차단하는 기능인 Secure Boot로 구현된다.



## Measured Boot

Measured Boot 프로세스는 부팅 중 실행하기 전 모든 구성 요소를 기록한다. 이는 변조 방지 방식(tamper-proof way)으로 유지되며 신뢰할 수 있는 플랫폼 모듈 (Trusted Platform Module, TPM)을 사용하여 구현된다. TPM은 펌웨어 및 중요 운영 체제 구성 요소의 해시를 저장하는 데 사용되어 악성 프로그램이 해시를 변경하는 것을 방지한다. 

Verified Boot와 Measured Boot는 동시에 사용할 수 있다. 이러한 개념과 Windows에서의 구현에 대한 기술적 세부 사항은 [Reference [1]](https://edk2-docs.gitbook.io/understanding-the-uefi-secure-boot-chain/overview)과 [Reference [2]](https://docs.microsoft.com/en-us/windows/security/information-protection/secure-the-windows-10-boot-process)에서 찾을 수 있다.

위 두 가지 이외에도 다음과 같이 네 가지의 개념이 있다.

- Post-boot verification - 부팅 이후 확인
- Booting from read-only media - 읽기 전용 미디어에서 부팅
- Booting from a read-only volume(image) - 읽기 전용 볼륨(이미지)에서 부팅
- pre-boot verification - 부팅 전 확인



## Post-boot verification

Post-boot verification은 부팅 프로세스가 완료된 이후 또는 후반 단계에서 시작된 프로그램에 의해 수행된다. 해당 프로그램은 이전에 실행된 OS 구성요소와 데이터의 무결성을 확인한다. 이 방식은 더 높은 권한으로 실행 중인 멀웨어의 경우 프로그램의 파일 읽기 요청을 가로채고 반환되는 파일 데이터를 제어함으로써 우회가 가능하나 그럼에도 안티 멀웨어 소프트웨어, 엔드포인트 탐지 및 대응 솔루션과 암호화 구성 요소의 미티게이션으로써 사용할 가치가 있다.



## Booting from read-only media

읽기 전용 미디어에서 부팅하는 것은 BIOS 및 UEFI와 같은 OS 시작 전 환경(pre-OS environment)에서 사용된다. 이러한 개념이 적용된 온라인 뱅킹을 위한 실시간 배포 방식은 이미 수년 전에 [제안되었으며](http://thinkinghard.com/secureinternetbanking/index.html) 현재는 재택근무 시 보안을 위해 제안되기도 한다. 이는 기존의 전통적인 멀웨어에 완벽하게 대응해 대부분의 멀웨어로부터 시스템을 보호할 수 있다.

그러나 악성 프로그램이 OS 이전 환경에 접근할 수 있으면 우회가 가능하다. 이동식 미디어에서 부팅되는 OS를 속여 기존 미디어에 저장된 코드를 실행할 수 있다. 이는 BIOS 또는 UEFI가 부팅 드라이브에서 데이터를 읽을 때 기본 스토리지 드라이버로 전환하는 동안 트리거할 수 있으며 관련된 코드 실행 이슈에 대한 자세한 내용은 [이전 글에서](https://dfir.ru/2018/07/21/a-live-forensic-distribution-executing-malicious-code-from-a-suspect-drive/) 확인할 수 있다.



## Booting from a read-only volume

읽기 전용 볼륨(이미지)에서 부팅하는 것 또한 비슷한 개념이지만 변경 불가능한 OS 파일만 읽기 전용 볼륨에 저장된다. 이 볼륨은 해시 또는 해시 트리를 사용하여 무결성을 검증한다. 해시 또는 해시 트리의 루트 해시는 하드웨어로 보호되는 루트에 의해 서명되고 검증되며 이 접근 방식은 기존의 검증된 부팅 구현과 관련이 있다. 더 자세한 기술 정보는 [여기](https://source.android.com/security/verifiedboot/verified-boot)서 찾을 수 있다.



## Pre-boot verification

Pre-boot verification은 잘 알려지지 않은 기술이며 일반적으로 PCI (Peripheral Component Interconnect) 장치 또는 사용자 지정 UEFI 이미지로 구현된다. 부팅 드라이브에 있는 부트 로더를 시작하기 전에 확인 프로세스는 하드웨어 구성, 노출된 펌웨어 메모리, 실행 파일 및 기타 데이터 (예 : Windows Installation의 레지스트리 키값) 등의 무결성을 확인한다. 

Pre-boot verification은 이전에는 초기화 코드를 포함하는 읽기 전용 메모리 (option ROM)가 있는 PCI 장치로 구현되었다. 초기화 코드는 부트 로더의 첫 번째 명령어 위치에 중단점을 설치해 중단점에 도달하면 PCI 장치의 메모리에서 사용자 지정 OS가 시작되는 프로세스로 이루어져 있다. OS는 연결된 드라이브에서 찾은 파일 시스템을 파싱하고 확인 프로세스를 수행하는데, 이때 해시는 무결성이 확인된 시스템 드라이브 또는 PCI 장치의 메모리에 저장되며 검증에 성공하면 원래 부트 로더가 실행되고 운영체제가 시작된다.

현재는 PCI 장치에서 사용자 지정 UEFI 이미지를 메인보드에 제공하는 것으로 구현되어 있다. 사용자 지정 UEFI 이미지에는 확인 프로세스에 필요한 파서가 포함되어 있으며 USB 부팅을 위해 USB 드라이브를 기반으로 한 구현도 있다. 이전 방식과 비교해 사용자가 부팅 순서를 변경하지 않는다는 가정하에 BIOS / UEFI 환경에 대한 중단점 또는 애드온을 사용하여 부팅 프로세스에 간섭할 필요가 없다.

이 접근 방식은 BIOS 및 UEFI를 포함한 펌웨어의 무결성을 확인하는 기능이 제한되지만, 사용자 지정 UEFI 이미지를 사용하면 설계상 신뢰할 수 있는 영역에 대부분의 펌웨어를 포함하게 된다.

또 다른 단점은 공식 Windows에서 구현된 파일 시스템 및 Windows 레지스트리 파싱 코드를 동일하게 생성하는 것이 거의 불가능하다는 것이다. 일반적으로 Linux 사용자가 NTFS 파일 시스템을 마운트 하면 동일한 파일 시스템을 탐색하는 Windows 사용자와 동일한 디렉터리 레이아웃과 파일 내용이 표시된다. 그러나 Pre-boot verification은 실제로 드라이브에 저장된 원시 데이터는 동일하나 다른 데이터를 반환할 수 있다. 

예를 들어 대부분의 Pre-boot verification 제품은 Windows 레지스트리 트랜잭션 로그 파일을 지원하지 않는다. 즉, Kernel 수준에서 실행되는 멀웨어가 레지스트리 키값을 수정해 Pre-boot verification에서 임의의 값을 가져오도록 할 수 있다. Windows Kernel은 트랜잭션 로그 파일을 지원하므로 이러한 변경 사항은 부팅 도중과 부팅 후에 운영체제에 표시된다. 이는 [이전 글에서](https://dfir.ru/2018/10/07/hiding-data-in-the-registry/) 자세히 설명되어 있다.  

verified 및 measured boot 프로세스는 코드와 데이터를 사용하기 전 검증하기 때문에 앞서 설명한 공격에 대한 영향을 받지 않을 것으로 예상된다. 실행될 파일이 파일 시스템 드라이버에 따라 다른 해시를 생성하는 경우 파일이 verified 및 measured를 거쳐 동일한 드라이버로 실행되기 때문에 보안에 영향을 미치지 않는다. 그러나 이 접근 방법 또한 항상 완벽하지는 않다. 이 글에서는 Windows Loader (`winload.exe` 또는 `winload.efi`)의 measured boot에서 발견된 두 가지 취약점에 중점을 둘 것이며, 별도의 소프트웨어 구성 요소(이후에 설명할 Loader와 kernel)에서 데이터와 코드를 검증하는 과정에서의 이슈에 대해 강조한다.



# Early Launch Anti-Malware

취약점을 살펴보기 전에 Windows Kernel (`ntoskrnl.exe`)에서 제공하는 안티 멀웨어 인터페이스를 살펴본다.

두 가지의 주요 부팅 보안인  verified boot와 measured boot가 활성화되어 있고 하드웨어/펌웨어 취약점을 고려하지 않는 경우 멀웨어의 초기 목표 삽입 지점은 boot-start 드라이버다.

악성 boot-start 드라이버를 방지하기 위해 Microsoft는 [ELAM (Early Launch Anti-Malware) 인터페이스](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/early-launch-antimalware)를 Windows 8에서부터 도입했다. 

ELAM 드라이버는 다른 드라이버보다 먼저 시작해 종속성과 유효성을 검사한다. 각 boot-start 드라이버에 대해 ELAM 드라이버는 다음 값을 반환한다.

- unknown - 알 수 없음
- good - 정상적인 드라이버
- bad - 악성 드라이버
- bad but critical for the boot - 악성 드라이버지만 부팅 프로세스에 빠져서는 안 될 요소

또한 ELAM 드라이버는 boot-start 드라이버에서 수행하는 레지스트리 작업을 기록하는 콜백을 설정하고 런타임 안티 멀웨어의 구성 요소에 저장할 수 있다.

자체 ELAM 드라이버를 사용하는 안티 멀웨어 소프트웨어는 Windows 8.1에서 도입된 [코드 무결성 보호](https://docs.microsoft.com/en-us/windows/win32/services/protecting-anti-malware-services-)가 제공되어 보호된 서비스로 실행할 수 있다. 

부팅 프로세스 중에 둘 이상의 ELAM 드라이버가 활성화될 수 있으며 각 ELAM 드라이버는 지정된 boot-start 드라이버를 독립적으로 확인하며 이 드라이버에 대한 최종 결과값은 다음 순위를 기반으로 한다.

![measured-boot-vuln/Untitled.png](measured-boot-vuln/Untitled.png)

> Table 1. ELAM scores (ranks)

한 ELAM 드라이버가 `good` 을 반환하고 다른 ELAM 드라이버가 `bad`를 반환하면 이 boot-start 드라이버는 `bad`로 결정된다.

위 ELAM 드라이버의 반환 값과 아래 정책을 기반으로 Kernel은 boot-start 드라이버를 허용하거나 거부할 수 있다.

- 모든 드라이버 허용
- `good`, `unknown`, `bad but critical` 드라이버 허용(기본값)
- `good`, `unknown` 드라이버 허용
- `good` 드라이버 허용

ELAM 드라이버는 다음 데이터를 기반으로 드라이버의 스코어를 결정한다.

- 드라이버 파일의 경로
- 해당 서비스 항목에 대한 레지스트리 경로
- 드라이버에 대한 인증서 정보 (게시자, 발급자)
- 이미지 파일 해시값

문제는 [DRTM (Dynamic Root of Trust for Measurement for Measurement)](https://www.microsoft.com/security/blog/2020/09/01/force-firmware-code-to-be-measured-and-attested-by-secure-launch-on-windows-10/)이 적용되면 이전에 boot-start 드라이버를 메모리로 읽어오는 데 사용된 펌웨어를 신뢰할 수 없어 관련 드라이버가 초기화되기 전에는 드라이브에서 아무것도 읽을 수 없기 때문에 파일 내용이 ELAM 드라이버에 노출되지 않는다. 따라서 기존의 멀웨어 시그니처는 적용할 수 없다. 그러나 경로, 해시 및 인증서 등의 정보로 악성 boot-start 드라이버를 차단하는 것은 효과적인 방법으로 볼 수 있다.

따라서 Microsoft는 다음과 같은 새로운 규칙을 정의했다.

- ELAM 드라이버에는 단일 boot-start 드라이버를 확인할 수 있는 제한된 시간이 주어짐
- ELAM 드라이버에는 모든 boot-start 드라이버를 확인할 수 있는 제한된 시간이 주어짐
- ELAM 드라이버에는 코드 및 구성 데이터에 대해 제한된 메모리가 제공됨
- ELAM 드라이버는 ELAM 레지스트리 하이브 (`C:\Windows\System32\config\ELAM`)의 특정 레지스트리 키 아래에 서명을 저장해야 함
- ELAM 드라이버는 서명의 유효성을 검사해야 함.
- ELAM 드라이버는 잘못된 서명을 처리해야 함 (이 경우 모든 boot-start 드라이버를 unknown으로 처리해야 함).
- ELAM 드라이버는 악성 boot-start 드라이버 (또는 다른 정책 위반)가 식별되면 측정된 부팅 상태를 무효화해야 함

ELAM 인터페이스가 처음 등장했을 때는 [효용성에 의문이 제기되었다](https://www.welivesecurity.com/2012/12/27/win32gapz-new-bootkit-technique/). bootkit은 악성코드를 VBR(Volume Boot Recod)에 기록하거나 IPL(Initial Program Loader)로 대체할 수 있으며 둘 다 Windows Kernel 이전에 실행되므로 ELAM 드라이버를 우회할 수 있었다. 그러나 Verified/Measured 부팅의 등장으로 위 기법의 적용이 어려워져 ELAM 인터페이스가 제 기능을 할 수 있게 되었다.



# ELAM and Measured Signatures

Measured 부팅 중 ELAM 시그니처를 읽은 다음 Windows Loader에서 측정한다([출처](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/elam-driver-requirements#malware-signatures)). 이렇게 하면 ELAM 시그니처가 신뢰 체인에 배치되어 증명 중에 누락 혹은 다운그레이드 된 시그니처를 감지할 수 있다.

ELAM 하이브 내의 특정 레지스트리 값에 저장된 위치만 측정되며 이 값을 `Measured`, `Policy` 및 `Config`라고 한다. 여기에는 공급 업체별 데이터를 포함할 수 있고 이름이 다르거나 유형이 `REG_BINARY`가 아닌 레지스트리 값은 측정되지 않는다.

![measured-boot-vuln/Untitled%201.png](measured-boot-vuln/Untitled%201.png)

> Table 2. Trend Micro 및 Microsoft에서 각각 저장한 시그니처 데이터와 측정값(굵게 표시됨)

ELAM 시그니처는 Windows Loader에 의해 측정되지만 ELAM 인터페이스는 Windows Kernel에 의해 제공된다는 점이 중요하다.



# Real-World ELAM Drivers

2021 년 2월에 필자는 유명한 안티 멀웨어 제품의 ELAM 드라이버를 리버싱했다. 25개의 ELAM 드라이버가 있는 총 26 개의 제품 (2 개의 제품이 동일한 ELAM 드라이버를 공유)에서 24 개가 기본 구성으로 설치되어 있었다. 

흥미롭게도 모든 ELAM 드라이버에는 보호된 안티 멀웨어 서비스를 시작하는 데 필요한 인증서 정보가 포함되어 있지만 26개 중 15개 제품에서 boot-start 드라이버에 대해 검사를 수행하지 않고 하드코딩된 결과값을 반환하거나 결과값을 전혀 제공하지 않았다. 

> 이러한 ELAM 드라이버를 placeholder driver라고 한다.

두 개의 ELAM 드라이버는 런타임 안티 멀웨어 구성 요소에 대한 boot-start 드라이버 정보를 기록했지만 ELAM 드라이버에서는 실제 검사가 수행되지 않았으며 결과값은 항상 `unknown`이었고, 한 ELAM 드라이버는 검사를 수행하지 않고 모든 boot-start 드라이버의 결과값을 `good`으로 보고했다.

11개의 제품의 ELAM 드라이버는 boot-start 드라이버에 대해 몇 가지 검사를 수행했다. 그중 하나는 알려진 정상적인 인증서의 하드코딩된 목록을 가지고 있고 ELAM 하이브는 시그니처 데이터를 저장하는 데 사용되지 않았으며 2개는 시그니처 데이터에 ELAM 하이브와 SYSTEM 하이브를 모두 사용했다. 하나는 ELAM 하이브를 사용한 뒤 SYSTEM 하이브를 시그니처 데이터 대체로 사용하고 7개의 제품은 ELAM 하이브에서만 시그니처 데이터를 읽었다.

11 개의 ELAM 드라이버 중 단 2개만 결과값에 따라 증명을 취소할 수 있었으며 다른 드라이버는 해당 루틴을 호출조차 하지 않았다.

이제 제품의 이름을 언급하도록 하겠다.

놀랍게도 Windows Defender에서 제공되는 ELAM 드라이버는 언급된 모든 규칙을 따르지 않았으며 SYSTEM 하이브를 시그니처 데이터의 저장 위치로 사용하고 (기본 위치 인 ELAM 하이브 제외) 악성 boot-start 드라이버가 감지되었을 때 증명을 취소하지 않았다.

Kaspersky 및 ZoneAlarm 제품의 동일한 ELAM 드라이버는 ELAM 하이브와 SYSTEM 하이브 두 위치에서 시그니처 데이터를 읽었으며 증명 또한 취소되지 않았다.

Sophos 제품의 ELAM 드라이버는 모든 boot-start 드라이버를 `good`으로 표시한다. 이는 Sophos에 보안 이슈로 보고되었지만 보안 상의 결함이 없는 의도된 기능으로 간주되었다.

요약하자면, 대부분의 안티 멀웨어 제품은 ELAM 드라이버를 사용하여 악성 boot-start 드라이버를 검색하지 않았다. 대신 ELAM 드라이버를 사용하여 자체적으로 보호된 서비스로 시작하고 핵심 ELAM 기능은 하드 코딩된 단일 결과값을 제공하거나 결과값을 전혀 제공하지 않도록 구현되었다.



# ELAM Hive

ELAM 하이브는 `C:\Windows\System32\config\ELAM` 레지스트리 파일에 저장된다.

[이전 글에](https://github.com/msuhanov/regf/blob/master/Windows%20registry%20file%20format%20specification.md) 해당 포맷에 대한 설명이 있으며 취약성을 이해하는 데 필요한 몇 가지 핵심 사항이 있다.

- 키 노드는 하위 키 목록을 가리킬 수 있다.
    - 키 노드 : 단일 레지스트리 키를 설명하는 데 사용되는 바이너리 구조
    - 하위 키 목록 : 레지스트리 키의 하위 키를 설명하는 키 노드에 대한 오프셋 목록
- 마찬가지로 키 노드는 값 목록을 가리킬 수 있다.
    - 값 목록: 키 값에 대한 오프셋 목록, 각 키 값은 레지스트리 키에 속하는 단일 레지스트리 값을 설명함
- `0xFFFFFFFF`와 같은 오프셋은 "nil"을 표현한다.
- 이러한 오프셋은 절댓값이 아니며 레지스트리 파일의 시작 부분에서 오프셋을 얻으려면 4096 bytes를 추가해야 한다.
- 키 노드와 키 값은 각각 레지스트리 키의 이름과 레지스트리 값의 이름을 저장한다. 이는 확장 ASCII (Latin-1) 문자열 혹은 UTF-16LE 문자열이다. (확장 ASCII 문자열로 저장할 수 있으면 해당 형식으로 압축됨.)
- 하위 키에서 대소문자를 구분하지 않는 바이너리 검색을 사용하려면 하위 키 목록을 대문자 이름(사전 순 정렬)으로 정렬해야 한다.
    - 값 목록은 정렬할 필요 없음
- 키 값은 레지스트리 값의 데이터 유형(예 : REG_BINARY)을 저장하고 값 데이터를 가리킨다.
- 4 bytes 이하의 값 데이터는 키 값에 직접 저장된다.
- 4 bytes 보다 큰 값 데이터는 다른 오프셋에 저장되며 이 오프셋은 키 값에 기록된다.
- 16344 bytes보다 큰 값 데이터는 16344 btyes 이하의 세그먼트에 저장되고 세그먼트에 대한 오프셋은 목록에서 참조된다. 키 값에는 데이터 대신 목록에 대한 오프셋이 저장된 Big Data Record의 오프셋이 저장된다.

레지스트리 파일의 예제는 다음과 같다.

![measured-boot-vuln/Untitled%202.png](measured-boot-vuln/Untitled%202.png)

> Figure 1
선택된 영역 - ELAM 하이브의 root key 노드
녹색 - 하위 키의 개수(2개) 
빨간색 - 하위 키 목록에 대한 오프셋 (0x3328, 절대 오프셋: 0x3328+4096=0x4328)
노란색 - 키 이름 ("ROOT", ASCII 문자열)



![measured-boot-vuln/Untitled%203.png](measured-boot-vuln/Untitled%203.png)

> Figure 2
선택된 영역 - 루트 키에 대한 하위 키 목록
노란색 - 목록의 요소 수 (2개)
빨간색 - 두 개의 하위 키에 대한 오프셋 (0x3188 및 0x0120, 이 유형의 하위 키 목록에서는 4 개)

각 오프셋 뒤에는 이름 해시가 포함되어 있으며 사전 순 정렬로 저장된다. (`0x3188`은 "Trend Micro" 키 노드에 해당하고 `0x0120`은 "Windows Defender" 키 노드에 해당됨)



![measured-boot-vuln/Untitled%204.png](measured-boot-vuln/Untitled%204.png)

> Figure 3
선택된 영역 - "Windows Defender" 키 노드
빨간색 - 값 데이터(1)
노란색 - 목록에 대한 오프셋 (0x3180)



![measured-boot-vuln/Untitled%205.png](measured-boot-vuln/Untitled%205.png)

> Figure 4
선택된 영역 - 값 목록(유효한 항목은 0x0230)



![measured-boot-vuln/Untitled%206.png](measured-boot-vuln/Untitled%206.png)

> Figure 5
선택된 영역 - 키 값 
노란색 - 데이터 크기 (0x215C 또는 8540 bytes)
빨간색 - 데이터 오프셋 (0x1020, 데이터가 4 bytes 이하인 경우 직접 저장) 
파란색 - 값 유형 (3 또는 REG_BINARY)
녹색 - 값 이름 ( "Measured", ASCII 문자열)



![measured-boot-vuln/Untitled%207.png](measured-boot-vuln/Untitled%207.png)

> Figure 6
값 데이터



![measured-boot-vuln/Untitled%208.png](measured-boot-vuln/Untitled%208.png)

> Figure 7
선택된 영역 - 데이터가 16344 bytes보다 크고 하이브 형식 버전이 1.4 이상인 경우의 Big Data Record Structure
빨간색 - 데이터 세그먼트 수 (2), 
노란색 - 세그먼트 목록에 대한 오프셋 (0x3188)



![measured-boot-vuln/Untitled%209.png](measured-boot-vuln/Untitled%209.png)

> Figure 8
선택된 영역 - 값 데이터 세그먼트 목록
빨간색 - 두 개의 오프셋 (0x4020 및 0x 8020)



![measured-boot-vuln/Untitled%2010.png](measured-boot-vuln/Untitled%2010.png)

> Figure 9
선택된 영역 - 세그먼트, 첫 번째 세그먼트에는 16344 bytes가 포함되고 마지막 세그먼트에는 나머지 데이터가 포함됨

유심히 봐야 할 몇 가지 세부 사항

1. 레지스트리 하이브가 로드되면 형식 위반이 있는지 확인한다. 위반이 감지되면 참조를 포함하여 이를 수정하거나 관련 레지스트리를 삭제하려고 시도한다.
2. 모든 하위 키 목록에 있는 요소의 사전 순 정렬이 확인된다. 두 개의 하위 키 (주어진 목록에 있는 현재 키와 이전 키)를 비교한 결과 잘못된 순서로 표시되면 현재 키가 삭제된다.
3. 일반적으로 하이브가 마운트 되면 유저 모드 응용 프로그램이 잠기게 된다. 따라서 운영체제가 부팅을 완료하면 ELAM 하이브는 마운트 해제된 상태로 유지된다.
4. ELAM 하이브는 형식 버전 1.5를 사용하므로 Big Data Record가 활성화된다.
5. 부팅 중에 Windows Loader는 ELAM 하이브를 단일 메모리 청크로 읽는다.



# Measured Boot Vulnerabilities

2020년 말, 2021년 초 관리자 권한으로 실행되는 악성 프로그램이 Measured Booting 프로세스에 영향을 주지 않고 ELAM 시그니처를 손상 및 삭제할 수 있는 두 가지 취약점을 발견하고 보고했다.

특히, Windows Loader는 ELAM 하이브에서 예상되는 레지스트리 값을 측정하는 반면 Windows Kernel은 손상된 ELAM 시그니처를 포함하거나 해당 레지스트리 값이 없는 하이브에서 다른 레지스트리 데이터를 확인해 이러한 Loader와 Kernel의 레지스트리 파싱 코드의 차이점을 악용한다.

![measured-boot-vuln/Untitled%2011.png](measured-boot-vuln/Untitled%2011.png)

> 발견된 취약점의 CVE ID

두 취약점 모두 보상을 받을 수 있었으며 CVE-2021–27094의 경우 업데이트를 배포하는 데 90일 이상이 걸렸고 해당 패치로 인해 데이터 손상 이슈가 발생했다.



## CVE-2021–28447

16344 bytes보다 큰 데이터로 ELAM 값을 측정할 때 Windows Loader는 Big Data Record를 파싱하지 않는다. 따라서 측정 중인 ELAM blob이 16344 bytes보다 크면 취약점을 트리거할 수 있다. 아래는 측정된 ELAM 값에 대한 데이터를 가져오는 데 사용되는 디 컴파일된 함수이다.

![measured-boot-vuln/Untitled%2012.png](measured-boot-vuln/Untitled%2012.png)

> 측정 값 데이터를 가져 오는 데 사용되는 함수

일반적인 경우 값 데이터가 16344 bytes 보다 크지 않으면 `KeyValue->Data` 필드에는 전체 값 데이터가 포함되고 `memmove()` 함수로 해당 필드의 데이터를 변수 `Heap`에 복사한다. 그러나 해당 함수에는 하이브 형식 버전, 값 데이터 크기에 대한 검사 등 Big Data Record를 처리하기 위한 코드가 존재하지 않는다. 따라서 데이터가 16344 bytes보다 크면 `KeyValue->Data`에는 값 데이터 대신 Big Data Record로 채워지지만 `memmove()` 호출은 이를 데이터로 생각해 Big Data Record와 이후 레지스트리 데이터를 로드된 레지스트리 파일에서 `Heap`으로 복사하게 된다. 

이를 악용하면 특정 조건에서 공격자는 측정에 영향을 주지 않고 ELAM Blob을 수정할 수 있다. 예를 들어 값 데이터 세그먼트가 Big Data Record 이전에, 즉 레지스트리 파일의 더 낮은 오프셋에 저장되면 힙 변수에 포함되지 않고 측정되어 실제 값 데이터는 측정되지 않는다. 또는 값 데이터 세그먼트가 Big Data Record 이후에 저장되면 측정되지 않아 측정 중 계산된 해시를 변경하지 않고 이후의 값 데이터를 변경할 수 있다.

ELAM 하이브는 부팅 완료 이후에는 로드되지 않으므로 어떤 방식으로든 (예 : HEX 편집기 등) 변경할 수 있으므로 공격자가 레지스트리 파일을 제한 없이 임의로 수정할 수 있다. 



### Root Cause

이 취약점은 이전에는 Windows Loader에서 이러한 레지스트리 값을 읽을 필요가 없었으므로 Big Data Record 케이스를 지원하지 않는 레거시 코드로 인해 발생했다.



### Fix

Microsoft는 Windows Loader에서 Big Data Record 케이스에 대한 지원을 구현하여 취약점을 수정했다. 해당 취약점은 ELAM blob의 임계값인 16344 bytes에 도달하지 않는 ELAM 드라이버에는 영향을 주지 않는다. 



### Original vulnerability report

```c
# Summary

When an ELAM driver stores a binary larger than 16344 bytes in one of three measured values (called "Measured", "Policy", or "Config") within the ELAM hive ("C:\Windows\System32\config\ELAM"), this binary isn't measured correctly by the Windows loader (winload.exe or winload.efi).
Under specific conditions, a modification made to an ELAM blob won't result in different PCR values, thus not affecting the measured boot (since PCR values are equal to the expected ones).

# Description

## Steps to reproduce

(Screenshots attached.)
1. Mount the ELAM hive using a registry editor.

2. Add a new key under the root of the ELAM hive. Assign a new value to this key (in this report, the value will be called "Measured").

3. Write more than 16344 bytes of data to that value (see: "01-elam-blob.png").

4. Unmount the ELAM hive.

5. Reboot the system.

6. During the boot, the Windows loader measures data starting from the beginning of the CM_BIG_DATA structure as pointed by the CM_KEY_VALUE structure describing the "Measured" value (see: "02-elam-blob-measured.png"). 
   Since the expected data length is larger than the CM_BIG_DATA structure, subsequent bytes of the hive file (actually, from the memory region used to store the hive file loaded) are included into the measurement (instead of actual value data).

7. After the boot, change (using a registry editor) several bytes within the value data, without altering the data size (see: "03-elam-blob-altered.png").

8. Reboot the system.

9. During the boot, the Windows loader will see the same CM_BIG_DATA structure and subsequent bytes as value data (see: "04-elam-blob-altered-measured.png").

## Root cause

The Windows loader doesn't support parsing value data stored using the CM_BIG_DATA structure. This structure is used when the hive format version is 1.4 or newer and value data to be stored is larger than 16344 bytes.
The ELAM hive uses the format version 1.5. Thus, the CM_BIG_DATA structure is supported in the NT kernel, but not in the Windows loader.
The OslGetBinaryValue routine (in the Windows loader) provides back a pointer to cell data containing the CM_BIG_DATA structure instead of parsing this and related structures and then providing a pointer to consolidated data segments.

## Attack scenarios

First, ELAM blobs larger than 16344 bytes aren't measured correctly. This is a serious security issue by itself.
Finally, if an ELAM driver uses existing measured ELAM blobs larger than 16344 bytes, a malicious usermode program could alter (corrupt or downgrade) these blobs without affecting the measured boot.

Such an attack is possible when:
* a list of cells containing value data segments is stored before the CM_BIG_DATA structure, or
* such value data segments are stored before the CM_BIG_DATA structure, or
* a list of cells containing value data segments and such value data segments are all stored after the CM_BIG_DATA structure, but there is a large gap after the CM_BIG_DATA structure 
  (which isn't smaller than the defined value data size, so the hash calculation won't reach the actual value data, or it's smaller than that, but the hash calculation doesn't reach the modified bytes of actual value data).

Under any specific condition defined above, changing offsets to value data segments or changing value data segments respectively won't be noticed during the measurement. 
(Since the hash is calculated over the internals of the hive file, but not over the actual value data.)
Since the ELAM hive isn't loaded after the boot, a malicious usermode program can open it and alter its data in any way possible 
(this is not limited to registry functions exposed by the Advapi32 library, the hive file can be opened and edited in a HEX editor), thus exploiting any pre-existing condition defined above.

## Possible solution

Handle the CM_BIG_DATA structure when parsing a registry value using the Windows loader.
```

취약점의 관련 스크린샷은 아래에 첨부되어 있다.

![measured-boot-vuln/Untitled%2013.png](measured-boot-vuln/Untitled%2013.png)

> 01-elam-blob.png

![measured-boot-vuln/Untitled%2014.png](measured-boot-vuln/Untitled%2014.png)

> 02-elam-blob-measured.png

![measured-boot-vuln/Untitled%2015.png](measured-boot-vuln/Untitled%2015.png)

> 03-elam-blob-altered.png

![measured-boot-vuln/Untitled%2016.png](measured-boot-vuln/Untitled%2016.png)

> 04-elam-blob-altered-measured.png



## CVE-2021–27094

이 취약점은 앞서 소개된 취약점보다 조금 복잡하다.
Windows Loader 또는 Windows Kernel에서는 하이브를 로드할 때 형식 위반이 있는지 확인한다. 이 검사는 Windows Loader에 의해 로드된 하이브에 대해 두 번 수행한 뒤 메모리의 Windows Kernel로 전달된다 (ELAM 하이브 포함).

이 검사 로직은 Windwos Loader와 Kernel이 유사하게 구현되었지만 차이점이 존재한다. 그중 하나가 Windows Loader는 하위 키 목록에 있는 요소의 사전 순 정렬을 확인하지 않는다는 것이다.

ELAM 하이브는 Windows Kernel에 의해 시작된 ELAM 드라이버에서 사용되므로 Windows Loader의 ELAM blob 측정과 ELAM 드라이버 사용 사이의 사전 순 정렬 검사가 삽입된다. Windows Loader에서 측정한 후 ELAM 드라이버에서 사용하기 전 ELAM blob을 포함하는 레지스트리 키를 임의 삭제하기 위해 이 검사 프로세스의 취약점을 이용할 수 있다.

공격자는 하이브를 마운트 한 다음 표준 API를 호출해 루트 키 아래에 있는 ELAM 하이브에 빈(값이 없는) 키를 삽입할 수 있다. 그리고 Windows Kernel이 측정된 ELAM blob이 존재하는 키를 삭제하도록 유도해야 하는데, 이는 공격자가 키 삽입 후 하이브의 마운트를 해제하고 HEX 편집기로 하이브 파일의 키 목록의 순서를 임의로 변경해 사전 순 정렬을 깨는 것으로 달성할 수 있다. 


> 역자: 
하이브 마운트 → 표준 API로 빈 키 생성 → 마운트 해제 → 하이브 파일의 키 순서 변경 
으로 나타낼 수 있습니다.

<br>
위 설명의 예시는 아래 ELAM 하이브의 레이아웃을 살펴보면 이해가 쉽다.

1. Key: Windows Defender
    - Value: Measured
2. Key: zz
    - No values

> 키 순서는 루트 키의 하위 키 목록에있는 키 노드 오프셋 순서를 반영

위는 표준 API 호출을 사용해 "zz" 키를 삽입한 정상적인 레이아웃이다. 공격자는 원하는 다음 레이아웃을 얻기 위해 하위 키 목록을 수정해야 한다.

1. Key: zz
    - No values
2. Key: Windows Defender
    - Value: Measured

> 하위 키 목록에서 두 요소의 순서를 반대로하면 취약점을 트리거할 수 있다.

<br>
이제 사전 순 정렬이 깨졌다. 

```c
Upcase("zz") > Upcase("Windows Defender")
```

Windows Kernel은 사전 순 정렬이 깨졌으므로 ELAM 드라이버에서 사용하기 전에 "Windows Defender"키를 삭제한다. 그러나 Windows Loader는 이러한 손상을 인식하지 못하고, 측정할 값이 없는 "zz"키를 건너뛰어 이미 삭제된 "Windows Defender" 키에 있는 "Measured"값을 측정한다. 

결과적으로 이는 ELAM blob이 ELAM 드라이버가 실행되기 전에 삭제될 수 있음을 보여준다.

### Root cause

이 취약점은 Windows Loader가 하위 키 목록에 있는 요소의 사전 순 정렬을 확인하지 않기 때문에 발생한다.

유니코드 형식의 키 이름이 허용되고, Windows Loader에는 적절한 대소문자 테이블(문자를 대문자로 변환하는 데 사용되는 테이블)이 없어 대소문자를 구분하지 않는 사전 순 정렬을 수행하지 못한다. 따라서 Windows Loader의 해당 기능이 의도적으로 제거된 것이 취약점이 발생한 원인이 되었다.

유니 코드 대문자 테이블은 Windows Kernel에서 사용되기 때문에 사전 순 정렬은 Kernel에서 구현되었다. 또한 Loader에서 완화된 검사로 인해 Kernel 이전에 로드된 하이브에 대한 검사까지 두 번 수행해야 한다.
흥미롭게도, 현재 구현으로는 유니 코드 문자에 대해 리팩토링 할 수 있지만 NLS 테이블을 로드하기 전에 SYSTEM 하이브를 로드해야 한다. 

### Fix

Microsoft는 Windows Loader에 사전 순 정렬 검사를 도입하여 취약점을 패치했다. 유니 코드 대문자 테이블은 Windows Loader에서 사용되지 않으므로 ASCII 문자를 사용하는 키 이름으로만 제한된다. 현재 키 이름에 ASCII 문자 외의 형식을 사용하는 ELAM 드라이버는 없는 것으로 확인된다.

이러한 패치는 SYSTEM 하이브를 검사할 때 심각한 데이터 손상 문제의 원인이 된다. SYSTEM 하이브는 ELAM 하이브와 마찬가지로 초기 부팅 중에 로드되나 ASCII 문자 외 형식의 키를 포함할 수 있다.

Windows Loader의 유니코드 비교 함수는 유니코드 대문자 테이블을 사용하지 않기 때문에 ASCII 외의 문자를 확인할 수 있는 방법이 없다. 아래 코드는 Windows Kernel에서 두 개의 키 이름 (대소 문자 구분 없음)을 비교하는 데 사용된다.

![measured-boot-vuln/Untitled%2017.png](measured-boot-vuln/Untitled%2017.png)

> 유니 코드 문자열을 비교하는 데 사용되는 함수

코드를 보면 알 수 있듯이 "z"("a"+ 0x19) 보다 높은 코드를 가진 문자의 경우 변환이 수행되지 않는다. 예를 들어 해당 코드에서는 "я"의 대문자 버전은 "Я"이 아니라 "я"이 된다.

Windows Kernel에서 구현된 동일한 함수는 유니 코드 대문자 테이블을 사용하므로 "я"의 대문자 버전은 "Я"로 정상적으로 변환이 된다. SYSTEM 하이브 내의 하위 키 목록에 있는 요소는 이에 따라 정렬되나 부팅하는 동안에는 Windows Loader의 함수를 사용하여 확인한다.

따라서 Windows Loader는 올바르게 정렬된 하위 키 목록을 손상된 것으로 간주해 레지스트리 키를 삭제한다.

해당 이슈를 재현하려면 동일한 상위 레지스트리 키 아래의 SYSTEM 하이브에서 이름이 "я1"및 "Я2"인 두 개의 레지스트리 키를 만든 다음 컴퓨터를 재부팅하면 된다.  Windows Kernel에서는 하위 키 목록이 다음과 같이 정렬된다.

1. *я1*
2. *Я2*

Windows Loader에서는 아래와 같이 인식한다. Loader는 이러한 정렬을 손상된 것으로 인식해 부팅 후 "Я2"키가 삭제되는 것을 확인할 수 있다.

```c
Upcase ("я1")> Upcase ("Я2"), Upcase ("я1") = "я1"
```

Microsoft는 취약점을 확인했지만 범위를 벗어났다는 이유로 업데이트하지 않았다.

### Original vulnerability report

```c
# Summary

A malicious usermode program can modify the ELAM hive ("C:\Windows\System32\config\ELAM"), so its blobs (registry values called "Measured", "Policy", and "Config") are correctly measured on the next boot, but the ELAM driver won't see them because registry keys containing these blobs are deleted by the NT kernel (even before the BOOT_DRIVER_CALLBACK_FUNCTION callback is registered). 
This results in proper (expected) PCR values but registry values (the ones previously measured) are absent when the ELAM driver tries to read them. So, the system will boot without proper ELAM signatures and this won't affect the measured boot.

# Description

## Steps to reproduce

(Screenshots attached.)

1. Mount the ELAM hive using a registry editor.

2. Add the "zz" key under the root key of the ELAM hive, don't assign any values to this key (see: "01-regedit.png").

3. Unmount the ELAM hive.

4. Open the ELAM hive file in a HEX editor (you can open it because it's not loaded), locate the subkeys list (subkeys of the root key), move the "zz" key to the first position on that list. (A key that was the first one before the move should now occupy the second position on the list. If there are three subkeys, just exchange the first and last keys on the list.)
The idea is to break the lexicographical order of subkeys. So, "1 2" becomes "2 1" and "1 2 3" becomes "3 2 1" (see: "02-hexeditor-intact.png" and "03-hexeditor-modified.png" for "before" and "after" states of the hive file respectively).

5. Reboot the system.

6. When the Windows loader (winload.exe or winload.efi) reads the ELAM hive, it doesn't check the lexicographical order of subkeys (see: "04-leaf-as-loaded-by-winload.png"). It's okay for the Windows loader if subkeys are stored in a wrong order.

7. When the Windows loader measures the ELAM hive (in the OslpMeasureEarlyLaunchHive routine), it reads subkeys one-by-one and measures their values (called "Measured", "Policy", and "Config").

8. If you manage to break the lexicographical order of subkeys by inserting empty keys and keeping real (non-empty) keys in the same order (relative to each other, not counting the empty keys), then the Windows loader will measure the usual (expected) data. This can be easily demonstrated with one key – "Windows Defender". 
If you insert the "zz" key using a registry editor, it goes to the end of the subkeys list. Like this:

- Windows Defender
- zz

If you move the "zz" key to the top (using a HEX editor), the lexicographical order is broken, but since the "zz" key has no values, it doesn't get measured. And the "Windows Defender" is measured as usual.
9. When the NT kernel starts, it takes hives attached to the loader parameter block and validates them. At this point, the validation routine checks the lexicographical order. If a subkeys list isn't sorted, offending keys are removed from the list (see: "05-leaf-as-loaded-by-kernel.png", "06-leaf-as-loaded-by-kernel.png", and "07-leaf-after-check-by-kernel.png"). 
This means that the "Windows Defender" key from the example above is removed.

10. When the ELAM driver tries to locate its signatures, they are gone – they were removed by the NT kernel because of the hive format violation (see: "08-leaf-as-seen-by-elam.png").

## Possible solutions

Either check the lexicographical order of subkeys in the Windows loader (which requires you to pick the NLS tables first) or measure empty keys together with non-empty ones.
```

취약점의 관련 스크린샷은 아래에 첨부되어 있다.

![measured-boot-vuln/Untitled%2018.png](measured-boot-vuln/Untitled%2018.png)

> 01-regedit.png

![measured-boot-vuln/Untitled%2019.png](measured-boot-vuln/Untitled%2019.png)

> 02-hexeditor-intact.png

![measured-boot-vuln/Untitled%2020.png](measured-boot-vuln/Untitled%2020.png)

> 03-hexeditor-modified.png

![measured-boot-vuln/Untitled%2021.png](measured-boot-vuln/Untitled%2021.png)

> 04-leaf-as-loaded-by-winload.png

![measured-boot-vuln/Untitled%2022.png](measured-boot-vuln/Untitled%2022.png)

> 05-leaf-as-loaded-by-kernel.png

![measured-boot-vuln/Untitled%2023.png](measured-boot-vuln/Untitled%2023.png)

> 06-leaf-as-loaded-by-kernel.png

![measured-boot-vuln/Untitled%2024.png](measured-boot-vuln/Untitled%2024.png)

> 07-leaf-after-check-by-kernel.png

![measured-boot-vuln/Untitled%2025.png](measured-boot-vuln/Untitled%2025.png)

> 08-leaf-as-seen-by-elam.png

---