---
title: "[Research] 어느날 IP CAM이 내게 주어진다면 (ko)"
author: jihun son
tags: [Bug Bounty, IP CAMERA, IoT, hunjison]
categories: [Research]
date: 2025-02-16 20:00:00
cc: true
index_img: /2025/02/16/hunjison/IPCAM/ko/image%202.png
---

---

안녕하세요, hunjison입니다.

저는 해킹 공부를 하면서 항상 아쉬웠던 점이 있었는데요. 바로 CVE가 하나도 없다는 것입니다.

그러던 어느 날,,, `OUYA77` 에게 흥미로운 이야기를 하나 듣게 됩니다.

두….둥..!! 바로 KISA에서 IP CAM 1.5 배 이벤트를 진행한다는 것이었는데요.

이걸 보고 부터는 모든 IP CAM이 돈으로 보이기 시작했습니다. 물론 잘하면 CVE도 발급 받을 수 있겠다 생각했죠.

![image.png](image.png)

# IP CAM target search

`OUYA77`과 함께 바로 IP CAM 제품군 리스트를 추리기 시작했습니다. 가정용 IP CAM은 2만원에서 10만원 정도의 가격대였고, 기업용 IP CAM은 20만원대부터 100만원이 넘는 것까지 다양했어요.

저희는 가난했고 돈은 많이 벌고 싶었기 때문에, 가정용 IP CAM을 여러 개 사기로 했습니다!

그 중에서도 외부 인터페이스 위주로 Attack vector가 많은 제품을 선정하기로 했습니다. 저희가 파악한 인터페이스는 아래와 같아요.

- Wifi (AP)
- **App (iOS/Android → IP CAM)**
- **Windows program**
- External cloud storage
- Ethernet
- SDCard
- USB

최근 IP CAM 들은 위 언급한 인터페이스 중 대부분을 가지고 있기 때문에, 살짝 비슷한 느낌이 있었는데요. 추가적으로 펌웨어가 온라인에서 획득 가능한지를 고려했습니다.

최종적으로 4개의 작고 귀여운 IP CAM들을 구매하게 되었습니다. 그 중에서도 2개 IP CAM의 분석 후기를 소개할게요.

# Target A

## #1 - Initial exploration

Target A의 사용자 interface에는 App (iOS / Android)와 Windows EXE가 있었습니다. 아래는 초기 분석을 수행한 내용입니다.

- App
    - 회원가입/로그인이 필수이고, App → 서버로 데이터를 전송하는 것으로 보임
    - 기기 등록 시 App의 QR code를 기기에 인식시켜야 하며, 여기에는 Wi-Fi 접속 정보가 포함됨
    - 기기 등록 완료 후 영상 시청이 가능함
- Windows 프로그램
    - 회원가입/로그인할 경우, EXE → 서버로 데이터를 전송하는 것으로 보임
    - 로그인 이후 LAN 네트워크에 있는 기기를 자동으로 탐색함. 기기 등록은 기기의 시리얼 번호를 기준으로 이루어지는 것으로 확인
    - 로그인하지 않을 경우 LAN 네트워크에서 기기를 탐색하고 영상을 시청할 수 있는 기능이 있음
- 인터넷에 공개된 블로그 게시글에서 펌웨어 다운로드 방법 알아냄
    - 공식 홈페이지에서 `/device/{device-model}/{device-model}` 형태의 URL 이용해서 다운로드 가능
    - `binwalk` 이용해 분석해보니 `squashfs` 파일시스템으로 확인해 파일 추출함

## #2 - Windows EXE analysis

Windows 프로그램 분석을 통해 아래와 같은 것들을 분석하려 했습니다.

- 로그인 버튼을 누르면 서버와 어떤 데이터를 주고 받는지?
- 영상 시청 버튼을 누르면 서버 / IP CAM과 어떤 프로토콜을 통해 인증하는지?

우선 Wireshark를 이용해 네트워크 분석을 시도했어요.

- TLS 암호화 때문에 정상적으로 인식 가능한 패킷이 없었습니다.
- 브라우저에서 `SSLKEYLOGFILE`를 추출해 Wireshark에서 TLS 패킷을 복호화하듯이, 프로그램의 TLS 로직을 동적 디버깅해 Key 파일을 추출하려 시도했습니다.
- 생각보다 TLS 구조 파악해서 리버싱하는 것이 어려워 다른 방법을 찾아보기로 했습니다.

다행히 이전에 Frida 프레임워크를 사용해본 경험이 있었고, Frida를 이용해 SSL 관련 암복호화 함수들을 Hooking 시도했습니다.

- Windows EXE에서 Import하는 SSL 관련 함수들을 식별했습니다.
- 함수의 argument 값 또는 return buffer를 정적 분석 결과를 참고해 적절히 출력했습니다. 오른쪽 사진을 보면, HTTP 200이 보이네요!

![image.png](image%201.png)

Frida Hooking 방법을 통해 Windows EXE 동작 방식을 어느 정도 파악할 수 있었습니다.

### #2.1 - Watching video after user login (WAN network)

Windows 프로그램과 IP CAM이 같은 네트워크에 있지 않은 WAN 네트워크의 경우에는, 아래 그림과 같이 서버를 통해 서로 통신해요.

- TLS 복호화를 해제하고 나면, 서버에 사용자 패스워드를 전달해요.
- 인증에 성공하면 서버가 영상 데이터를 보내줍니다.

![image.png](image%202.png)

그러나 IP CAM과 server가 주고 받는 TLS 1.2로 암호화된 데이터는 복호화가 불가했어요. TLS 1.2 프로토콜의 경우 Cipher suite가 RSA인 경우에만 몇몇 취약점들이 알려져 있다고 하는데, 분석 대상 IP CAM의 Cipher suite는 `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`로 MITM 공격이 불가했습니다.

### [See also] Packet sniffing with Wi-Fi dongle

IP CAM과 서버가 주고 받는 패킷을 캡쳐하기 위해서는 Wi-Fi dongle을 이용했어요. Dongle은 일반적으로는 Wi-Fi 기능이 없는 기기에서 Wi-Fi를 사용하기 위해 사용하는데요. 우리는 반대로 PC에 연결된 이더넷을 Wi-Fi로 뿌려주기 위한 목적으로 dongle을 사용했습니다. 

- 아래 그림처럼 dongle이 공유기처럼 동작하게 됩니다.
- Wireshark에서 dongle 네트워크 인터페이스 선택해 캡쳐하면, IP CAM으로 전달되는 모든 패킷이 캡쳐 가능해집니다.

![image.png](image%203.png)

### #2.2 - Watching video after user login (LAN network)

흥미로운 로직은 LAN 네트워크에서 있었습니다. Windows EXE와 IP CAM이 LAN encrypt key를 이용해 데이터를 암호화해 주고 받는 것을 확인했을 때, 저는 당연히 고정키를 이용할 것이라고 생각했습니다. 제가 Target A를 너무 얕봤던 것이죠,,,

아래 그림에서 설명하고 있는 것처럼 사용자 로그인 시 서버가 JWT Token을 발급하고, 이를 이용해 LAN encrypt key를 전달합니다. Server와 IP CAM 간에도 유사한 방식으로 LAN encrypt key가 전달될 것으로 추측됩니다.

![image.png](image%204.png)

### #2.3 - Watching video with no user login (LAN network)

Target A에서는 로그인 없이 LAN 네트워크에서 기기를 스캔해 영상을 시청할 수 있는 기능이 있었는데요. 영상 시청을 위해서는 사용자 비밀번호가 필요했습니다.

- Windows EXE에서는 한 번 성공한 패스워드를 캐싱해서 다음부터는 별도로 패스워드를 물어보지 않기도 하는데, 저는 그걸 잊어버려 제가 취약점을 찾았다고 착각했었죠. 보고서 작성 직전까지 가서야 무언가 잘못되었음을 깨달았습니다..

![image.png](image%205.png)

### #2.4 - Watching video with no user login (RTSP protocol)

Target A는 RTSP 프로토콜도 지원했는데요. RTSP(Real Time Streaming Protocol)은 네트워크를 통해 오디오, 비디오, 데이터 등의 멀티미디어 데이터를 스트리밍 하기 위해 사용하는 프로토콜입니다. 인터넷 블로그 게시글을 통해 Target A의 RTSP URL을 식별할 수 있었습니다. 

- 과거 버전의 Target A에서는 **6자리의 device code를 RTSP 패스워드로 이용**했으나, 최신 버전에서는 기기 등록 시 패스워드를 설정하는 방식으로 패치되었습니다.

![image.png](image%206.png)

## #3 - API server

Frida로 Windows EXE를 hooking하다가 API 서버의 주소를 발견했어요. Windows 프로그램과 동일 기능을 지원하지만, 무려 Internet explorer에서만 접근 가능한 !!! 아주 허름한 웹사이트였죠. Windows 프로그램과 동일하게 회원 가입, 기기 등록, 패스워드 변경 등 기능을 지원하는 것으로 보였습니다.

몇몇 가능성을 테스트해보려고 고민하던 중, 버그바운티의 범위를 벗어난다는 것을 깨닫고 분석을 멈췄습니다…

## #4 - Firmware analysis

분석 초기에 펌웨어를 획득했다고 언급했었는데요. 아래와 같은 부분들을 중점적으로 분석했으나 취약점을 찾을 수 없었습니다.

- Command Injection: 시스템 명령어를 인자로 받는 프로세스가 있어 해당 부분 중점적으로 분석했으나 실패
- BOF / DOS attack: 사용자 request 처리 과정에서 length 검증을 올바르게 하는지 살펴봤는데, 별도로 찾지 못했음

## #5 - Result

Target A에서는 Windows EXE 중심으로 시나리오별 로직을 주로 분석했는데요. 아쉽게도 취약점을 찾을 수가 없었습니다. Target A를 분석하면서 느낀 점은, 이미 많은 해커들이 해당 제품을 거쳐간 걸로 보인다는 점입니다.

- #2에서 분석한 4가지 시나리오에 대해 안전한 암호화 방법을 적용했습니다. 처음부터 이렇게 설계했을 거라고는 절대 생각하지 않습니다..
- BOF 관련 length 검증이 비정상적으로 철저했습니다.

![image.png](image%207.png)

 

---

# Target B

## #1 - Initial exploration

 Target B의 사용자 interface에는 App (iOS / Android)와 Windows EXE가 있었습니다. Windows EXE의 네트워크 분석을 수행하다 IP CAM의 IP 주소에 Web Interface를 추가로 식별할 수 있었습니다.

- App
    - 제품 사용설명서에 초기 설정과 관련된 부분이 꽤나 취약해 보입니다.
    - App을 통해 기기를 등록할 때에는, IP CAM의 Wi-Fi에 모바일 기기가 접속한 후에 모바일 기기에서 가지고 있던 Wi-Fi 접속 정보를 넘겨주면 초기 등록이 완료됩니다.
    - 기기 등록 완료 후 영상 시청이 가능합니다.
    - 영상 시청 이외에도 녹화, 클라우드 업로드, 카메라 상하좌우 이동, 버저음 출력 등 다양한 기능이 존재합니다.
- Windows EXE
    - 로컬 네트워크에 있는 기기를 탐색하고 등록할 수 있습니다.
    - 많은 기능이 없습니다.
- Web Interface
    - IP CAM의 IP 주소에 열려있는 CGI 기반 웹사이트입니다. 웹사이트 접속을 위해 사용자 패스워드로 인증이 필요합니다. UI가 허접해 취약점이 많아 보입니다!!
    - App에서 제공하는 대부분의 기능을 제공합니다.
    - **설정 파일 백업 / 로드, 펌웨어 업데이트 기능**을 제공합니다.

## #2 - Web Interface

### #2.1 - System configuration backup / load

Web interface에는 시스템 설정 파일을 백업(IP CAM → PC 다운로드)하고, 로드(PC → IP CAM)할 수 있는 기능이 있었는데요. 바이너리 파일의 구조가 이랬습니다.

- Header (0x200)
    - MD5 hash 존재
- Body
    - TAR 구조

BODY 필드의 데이터가 TAR 구조임을 식별하고 추출해보니, 아래 사진처럼 파일의 경로와 원본 데이터를 얻을 수 있었습니다. 만약 구조를 수정해 `/etc/shadow` 경로에 내가 원하는 데이터를 넣어주면 시스템에서 overwrite가 가능하다고 생각했죠.

![image.png](image%208.png)

문제는 MD5 hash였습니다. 길이가 MD5임이 분명한데 값이 전혀 맞지 않았습니다. 그러던 중,, 2019년 한 게시글에서 MD5 값을 맞춰 동일 IP CAM의 루트 패스워드를 변경한 사례를 알게 됩니다.

이런 이런,,,, 이미 누군가 취약점을 제보해 패치된 것이 틀림 없습니다. 😂 이후 펌웨어 추출 단계에서 더 설명하겠지만 MD5 hash 필드의 계산법은 해커들의 노력에 따라 계속해서 업데이트된 것으로 보입니다.

### #2.2 - Firmware update

제조사의 홈페이지에서 펌웨어를 다운로드 받을 수 있었는데요. 분명 모두가 아는 ZIP 구조인데도 정상적으로 압축 해제가 되지 않아 살펴보니 구조에 일부 customization된 부분이 존재했습니다. 

ZIP 구조에 맞춰 따라가면서 규칙을 파악해 바이너리를 일부 수정해주었습니다. 그 결과 펌웨어를 추출하는 데에 성공했습니다 (일부 에러가 발생해 모든 파일 추출은 힘들었죠)

![[https://blog.forensicresearch.kr/3](https://blog.forensicresearch.kr/3)](image%209.png)

[https://blog.forensicresearch.kr/3](https://blog.forensicresearch.kr/3)

그러던 중,, 제조사 홈페이지에서 현재부터 과거까지 펌웨어 모음을 다운로드할 수 있는 방법을 발견했습니다. 저희는 아래와 같은 방법으로 분석했습니다.

- ZIP 구조 파악
    - ZIP 구조에 맞춰 스크립트 변경해가며 펌웨어 추출
- 메인 바이너리 리버싱
    - System config 파일(firmware와 구조 같음) 생성 로직 리버싱해 변경 사항 추적

![image.png](b6c855a3-b38e-402f-bffd-ea5f99fa942b.png)

분석 결과 펌웨어 버전이 올라가면서 바뀐 점들을 알아낼 수 있었습니다. 변경 과정에서 얼마나 많은 해커가 참여했을지 느낄 수 있었습니다.. 그저 부럽네요…

- ZIP 파일 구조 변경
    - 초기에는 파일 시그니처를 단순 변경해 알아볼 수 없게 만들었으나, 중간에는 파일의 일부분을 부분 암호화했고, 현재는 파일 전체가 암호화된 것으로 보임
- MD5 hash 생성 로직
    - 초기에는 `MD5(BODY)` 형태였으나, `MD5(BODY + "secret string")` 형태로 바뀌고, 현재는 어떻게 생성하는지 알 수 없음
- 펌웨어 파일 업로드 위치
    - 2017년부터 2021년까지 펌웨어는 홈페이지에 업로드 되었으나, 그 이후에는 App과 연동되는 백엔드 어딘가로 업로드됨

## #3 - App

Android 루팅 폰을 가지고 있지 않았던 관계로, App 분석은 네트워크 수준에서만 진행했어요. 귀찮아서 안 한게 맞긴 합니다..^^ iPhone을 가지고 있었기 때문에 패킷 캡쳐를 아래와 같이 수행했습니다.

- 구글에 검색해보니 맥북과 iPhone을 연결하면 iPhone 패킷 캡쳐가 가능했어요

![[https://lxxyeon.tistory.com/157](https://lxxyeon.tistory.com/157)](image%2010.png)

[https://lxxyeon.tistory.com/157](https://lxxyeon.tistory.com/157)

iPhone 에서 수집한 패킷을 분석해 파악한 IP CAM과의 통신 과정은 이랬어요.

- IP CAM의 전원이 켜지면 server와 주기적으로 통신해요. IP CAM은 자신의 IP 와 접속 가능한 포트 정보를 server에 보냅니다. 포트 정보가 1분마다 바뀌어요.
- 사용자가 App을 켜면 server에 communication request를 보냅니다. 이를 받은 서버는 가지고 있는 IP / port 정보로 IP CAM에 communicaiton request를 보내죠. 여기에는 사용자의 IP / port 정보가 포함돼요.
    - IP CAM은 정보를 받아 사용자에게 연결 요청을 보냅니다.

![image.png](image%2011.png)

Target B에 대해 블로그에 공개된 공격 사례를 보면, 그들은 아래와 같은 순서로 공격을 수행했습니다.

- 플래시 메모리를 칩오프하고, 펌웨어를 덤프함
- 바이너리를 분석하고 BOF 취약점을 발견함
- RCE를 통해 쉘을 획득함

![image.png](image%2012.png)

만약 우리가 처음부터 다시 IP CAM을 분석한다면 그들과 같이 플래시 메모리에서 펌웨어를 추출하는 작업부터 수행했을 것입니다. 이번 프로젝트를 통해 펌웨어 확보하고 동적 디버깅이 가능한 환경을 가지는 것이 무엇보다 중요함을 알게되었습니다.

- 정말로, 정말로 이게 제일 중요해요..

## #4 - Windows EXE

Windows EXE는 로직은 단순했지만, 여기에서의 흥미로운 점은 App과 Windows EXE가 사용하는 프로토콜이 전혀 달랐다는 점입니다. IP CAM이 지원하는 프로토콜이 너무나 다양해 잘 찾기만 한다면 꽤나 많은 취약점이 나올 수 있을 듯도 합니다.

![image.png](image%2013.png)

---

# 마무리

저는 약 3달(24.11 ~ 25.1)에 걸쳐 이 글에 소개한 IP CAM 2개를 포함해 약 4개의 IP CAM을 분석했습니다. 처음 경험하는 IoT 기기 분석이라 미흡한 점이 많았습니다. 실제로 Windows EXE 분석과 네트워크 프로토콜 분석은 많이 해보지 않은 것이라 꽤나 어색했습니다.

그럼에도 제가 프로젝트를 통해 배운 점은 아래와 같습니다.

- 취약점이 많이 나오는 타겟은 따로 있다. 해커들이 많이 거쳐간 타겟은 보안 수준이 매우 높다.
- IP CAM에서 너무 기본적인 취약점은 어느정도 패치가 된 것으로 보인다.
- IoT 기기 분석이 생각보다 기초 지식을 많이 요구한다. Windows EXE 정적 분석, 동적 분석을 포함해 네트워크 프로토콜 분석, 소켓 프로그래밍, 웹해킹, 시스템 해킹, 하드웨어적 접근 등 알아야 할 게 많다.
- 펌웨어 추출과 동적 디버깅 환경 구축이 가장 우선시 되어야 한다.

![image.png](image%2014.png)

긴 글 읽어주셔서 감사합니다.