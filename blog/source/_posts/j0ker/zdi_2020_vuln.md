---
title: "[Translation] ZDI에서 선정한 2020년 제일 중요한 취약점 Top 5"
author: j0ker
tags: [j0ker, cve,  zdi]
categories: [Translation]
date: 2021-03-21 14:00:00
cc: true
index_img: /2021/03/21/j0ker/zdi_2020_vuln/Untitled.png
---



안녕하세요. 믿거 조 씨 집안 28대손 Joker(조-커)입니다. 첫 글은 부담스럽지 않게 번역글로 준비해봤습니다. 앞으로 재미있는 연구 많이 해서 좋은 글 올려드리도록 하겠습니다 (__ __) 참고로 번역에는 의역이 많을 예정입니다. 이런 걸로 태클 ㄴㄴ

![](zdi_2020_vuln/Untitled.png)

두두등장

---

연말이 되면서, 우리는 2020년에 받은 취약점들을 뒤돌아 보면서 최고의 취약점들을 뽑아보면 어떨까라는 생각을 하게 되었다. 생각은 이렇게 했지만, 작년에(글은 작년에 나온 거라 올해라고 되어 있지만 읽기 편하도록 현재 시점에서 작년이라 하겠다) 우리가 공개한 Advisor들의 수가 거의 최고에 다다랐기 때문에 1400개가 넘는 버그들 중에서 최고의 버그 5개를 골라내는 것은 생각보다 어려운 일이었다. 그리고 마침내 우리는 2020년에 제출된 버그들 중 최고의 버그 Top 5를 꼽아보았다! 짜잔!

# CVE-2020-0688/ZDI-20-258
**Microsoft Exchange Server Exchange Control Panel Fixed Cryptographic Key Remote Code Execution Vulnerability**

이 취약점은 Microsoft Exchange Server의 Exchange Admin Center 웹 인터페이스에서 발견된 매우 크리티컬한 취약점으로, 인증된 Exchange 사용자는 이 취약점을 통해 서버의 SYSTEM 권한을 얻을 수 있다. 이 웹 인터페이스는 "Admin" 인터페이스라고 불리기는 하지만, Exchange 서버의 mailbox에 대한 인증서를 가지고 있는 유저라면 모두 접근이 가능하며 Outlook Web Access와 같이 네트워크에 노출되어 있다. 이 취약점은 Exchange Admin Center ASP.NET에 설치되어 있는 암호화 키("machine keys")와 연관이 있다. Exchange에서는 설치할 때마다 이 키들을 랜덤하게 생성하여 해당 키가 유니크하고 안전하도록 해야 한다. 하지만, 실제로는 이 키들을 install media에서 바로 복사해오는데, 이로 인해 해커가 다른 제품의 installation을 참고하여 이 키들을 알 수 있다. 해커는 이 키들을 이용해 서버에 조작된 메시지를 보내고, 해당 메시지가 deserialize 되면서 arbitrary code execution을 할 수 있다. Exchange는 보통 엔터프라이즈의 중심에 있기 때문에 Exchange Server의 취약점은 매우 중요하고 해커들도 매우 탐낸다. 만약 당신의 조직에서 아직 이 취약점을 패치하지 않았다면, 최대한 빨리 패치를 진행하길 바란다. 좀 더 디테일한 내용들은 이 [링크](https://www.zerodayinitiative.com/blog/2020/2/24/cve-2020-0688-remote-code-execution-on-microsoft-exchange-server-through-fixed-cryptographic-keys)에서 확인할 수 있다.

# CVE-2020-3992/ZDI-20-1377
**VMware ESXi SLP Use-After-Free Remote Code Execution Vulnerability**

이 버그는 ZDI 취약점 연구원 Lucas Leong이 발견한 버그이다. ESXi는 VMWare 사에서 개발한 엔터프라이즈급 하이퍼바이저이다. Service Location Protocol(SLP)는 ESXi에 기본적으로 활성화되어 있는 프로토콜 중 하나이다. SDP는 클라이언트가 네트워크 서비스를 탐색할 수 있도록 지원하는 프로토콜이다. SLP에서 가장 인기 있는 기능이 OpenSLP인데, Lucas는 ESXi에서 자체 제작한 버전을 사용한다는 것을 발견하였다. 거기다가 이 자체 제작 버전에서 크리티컬한 취약점 두 개가 발견되었다.

[첫 번째 취약점](https://www.zerodayinitiative.com/advisories/ZDI-20-1377/)은 Use-After-Free(UAF) 취약점으로 `SLPDPorcessMessage()`에서 SLPMessage 오브젝트가 free 되었음에도 불구하고 SLPDatabase 구조체에서 계속해서 해당 메모리를 레퍼런스 하고 있어 발생한 취약점이다. 이 취약점은 WAN 환경에 있는 해커에 의해 활용될 수 있다.  

취약점이 발견된 이후, VMware는 이 취약점을 패치하였지만 제대로 처리하지 못하였고 ZDI-CAN-12190으로 패치에 대한 bypass 취약점이 제보되었다. 

또 한 가지 주목할만한 점은 이 취약점을 이용해서 원격으로 공격할 수 있기도 하지만, 제한된 환경에서 실행되는 프로세스에서의 sandbox escape도 가능하다는 것이다. 이 취약점을 통해, EXSi 같이 많이 연구된 제품들에서도 아직 알지 못했던(잘 살펴보지 않았던) attack surface들이 있다는 것을 알 수 있었다.

# CVE-2020-9850/ZDI-20-672
**Apple Safari in Operator JIT Type Confusion Remote Code Execution Vulnerability**

이 버그는 Georgia Tech Systems Software & Security Lab팀이 [Pwn2Own에 참여하면서](https://taesoo.kim/pubs/2020/jin:pwn2own2020-safari-slides.pdf) 제보한 버그이다. 이 버그는 DFG 단에서 발생하는 Webkit type confusion으로 시작되는 버그 체인의 한 부분으로 작년에 발견된 이 [버그](https://www.zerodayinitiative.com/blog/2019/11/25/diving-deep-into-a-pwn2own-winning-webkit-bug)와 유사하다. OpenGL's CVM(Core Virtual Machine)의 힙 오버플로우 버그와 체이닝 하면 사파리에서 ".app" symlink를 실행할 수 있다. 그런 다음 레이스 컨디션을 통해 cfprefsd와 kextload에서 first-time app protection 우회, root 접근, 권한 상승이 차례대로 이뤄지며 모든 것들은 피해자가 웹페이지를 방문하고 난 후 백그라운드에서 이루어진다. 

한 웹 페이지에 접속했는데 10초 후에 악성코드가 당신의 컴퓨터에서 실행된다고 생각해보라. 소름이 돋을 정도이다. 결과적으로 그들은 Pwn2Own에서 성공적으로 데모를 마쳐 $70,000를 획득하였다. 

# CVE-2020-7460/ZDI-20-949
**FreeBSD Kernel sendmsg System Call Time-Of-Check Time-Of-Use Privilege Escalation Vulnerability**

이 취약점은 m00nbsd라는 닉네임을 사용하는 한 연구원이 ZDI에 제보하였다. 해커는 FreeBSD의 32-bit `sendmsg()` 시스템 콜에 존재하는 Time-Of-Check, Time-Of-Use(TOCTOU) 취약점을 이용하여 낮은 권한에서 kernel-level code execution을 할 수 있다. 이 취약점은 시스템 콜에 존재하는 double-fetch 버그로 유저 랜드에 있는 MsgLen 변수에 첫 access가 있고 난 후, 두 번째 access가 발생하기 전에 더 큰 값으로 바꾸면 오버플로우를 트리거할 수 있다. 해커는 루프에서 정상적인 인자로 `sendmsg()`를 호출하는 스레드를 하나와 MsgLen을 큰 값으로 조작하는 스레드 하나를 실행하여 레이스 컨디션을 발생시켜 오버플로우를 트리거할 수 있다. 이 버그의 난이도는 낮지만 비교적 오랫동안 살아남았다. 우리는 작년 9월에 이 버그에 대해 포스팅을 하였으니 관심이 있다면 [여기](https://www.zerodayinitiative.com/blog/2020/9/1/cve-2020-7460-freebsd-kernel-privilege-escalation)에서 자세한 내용을 확인할 수 있다.

# [CVE-2020-17057](https://hackyboiz.github.io/2021/01/01/l0ch/2021-01-01/)/ZDI-20-1371
**Microsoft Windows DirectComposition Uninitialized Pointer Privilege Escalation Vulnerability**

이 취약점은 Windows의 DirectComposition이라는 커널 모드의 그래픽 구성요소에서 발견된 취약점이다. `win32kbase!DirectComposition::CInteractionTrackerMarshaler::SetBufferProperty` 함수는 유저 모드에서 전달된 데이터를 기준으로 `DirectComposition::CInteractionTrackerMarshaler` 타입의 객체를 채운다. 만약 함수를 실행하는 과정에서 비정상적인 데이터를 발견하면, 에러를 핸들링하는 분기로 들어가게 되는데, 여기에서 함수에서 생성한 객체에 있는 리소스들을 release 한다. Release 과정에서 초기화되지 않은 포인터를 release 하게 된다. 이를 통해 해커는 커널 모드에서 실행 흐름을 제어할 수 있고 결과적으로 SYSTEM 권한을 획득할 수 있다.