---
title: "[Research] 공대오빠가 알려주는 Windows Driver Part 1 - Setting Up Kernel Debugging"
author: L0ch
tags: [L0ch, research, windows, windows driver, debugging, kernel debugging, kernel, third-party driver exploitation]
categories: [Research]
date: 2021-05-30 18:00:00
cc: true
index_img: 2021/05/30/l0ch/windows-driver/Untitled.png
---

# 시리즈 바로가기

공대오빠가 알려주는 Windows Driver Part 1 - Setting Up Kernel Debugging                            ← Now!

[공대오빠가 알려주는 Windows Driver Part 2 - CVE-2020-12928: AMD Ryzen Master 분석(1)](https://hackyboiz.github.io/2021/07/14/l0ch/windows-driver-part2/)

[공대오빠가 알려주는 Windows Driver Part 3 - CVE-2020-12928: AMD Ryzen Master 분석(2)](http://localhost:4000/2021/07/28/l0ch/windows-driver-part3/)

[공대오빠가 알려주는 Windows Driver Part 4 - CVE-2020-12928: AMD Ryzen Master 분석(3)](http://hackyboiz.github.io/2021/08/11/l0ch/windows-driver-part4/)

------



# 서론

안녕하세요! L0ch입니다. 지난번 pwn cool sexy 시리즈에 이어 새로운 시리즈로 돌아왔습니다!  그 사이에 기말고사 기간이 된 건 덤.. 

![windows-driver/Untitled%201.png](windows-driver/Untitled%201.png)

> 과거의 나야.. 너 기말도 망했어

![windows-driver/Untitled%202.png](windows-driver/Untitled%202.png)

교수님 종강좀요ㅠㅠㅠ

이번 시리즈에서는 하루한줄을 보다 보면 꽤 자주 나오는 Windows third-party driver의 권한상승 취약점을 다룰 예정입니다! 

![windows-driver/Untitled%203.png](windows-driver/Untitled%203.png)

> 이 드라이버 아님ㅎㅎ;

예시를 하나 들어보자면.. 최근 이슈가 된 Dell의 BIOS 업데이트에 사용되는 드라이버에서 나온 [[하루한줄] CVE-2021-21551: 수억 대의 Dell PC에 영향을 주는 권한 상승 취약점 ](https://hackyboiz.github.io/2021/05/07/l0ch/2021-05-07/) 정도가 있겠네요. 대충 심각한 취약점인건 알겠는데 IOCTL은 뭐고... `EPROCESS`? 토큰을 어떻게 덮어 쓴다는 건지 또 권한상승은 어떻게 한다는 건지.. 정말 모르는 용어, 개념 투성이네요. 공부할게 너무 많지만 ㅠㅠㅠ 하나씩 공부하다보면 언젠가는 드라이버 취약점을 찾을수도 있겠죠?



오늘은 시리즈 첫 글이니 가볍게 디버깅 환경 세팅으로 시작해서 앞으로 간단한 exploit 예제를 다루고, one-day 분석하는 것을 끝으로 마무리하려고 합니다. 이번 시리즈도 길어질 것 같은 예감이 드는 건.. 기분 탓인가..?

# 준비

우선 디버깅 세팅부터 해야 합니다. 커널 디버깅은 유저 모드 애플리케이션과 달리 로컬에서 디버깅할 수 없습니다. 완전히 불가능한 것은 아니지만 제약사항이 많죠. 그래서! 보통은 원격지 PC를 하나 구성해 디버거를 세팅합니다.

이전에는 named pipe를 사용해 가상 시리얼 포트로 커널 디버깅을 했지만 시리얼 포트 속도의 한계로 KDNET 네트워크 디버깅을 사용하며 필요한 준비는 아래와 같습니다.



**1. 호스트 Windows와 VMware or Hyper-V의 게스트 Windows**

Windbg를 사용해 커널을 분석할 것이기 때문에 분석대상이 되는 디버기 Windows와 분석을 진행할 Windows가 각각 필요합니다. 호스트가 Windows 환경이면 게스트 Windows VM 하나만 올리면 됩니다. 

![windows-driver/tempsnip.png](windows-driver/tempsnip.png)

> ?

 

네 저는 맥을 사용하기 때문에.....  Windows VM 두개를 올려서 진행했습니다. 

네? 어디서 뭐가 돌아가는 소리 안나냐구요?

.

.

![windows-driver/Untitled%204.png](windows-driver/Untitled%204.png)

> 이륙한드아아ㅏㅏ !!!

맥이 잘 버텨주고 있으니 다행이네요 ㅎㅎ 아님말고 
글에서는 편의상 호스트 OS가 Windows라고 가정하고 분석할 디버기 Windows를 게스트라고 부르겠습니다! 

**2. 블루스크린을 친구처럼 생각하는 강인한 멘탈(?)**

사실 취약점 찾는 입장에서 블루스크린은 곧 크래시기 때문에 블루스크린을 본다면 개이득이죠ㅋㅋ  

![windows-driver/Untitled%205.png](windows-driver/Untitled%205.png)

![windows-driver/Untitled%206.png](windows-driver/Untitled%206.png)

> 편안-

멘탈까지 준비가 모두 되었다? 바로 디버깅 세팅으로 고고고



# Setting Up Debugging

먼저 게스트와 호스트 Windows에 [Windbg](https://docs.microsoft.com/ko-kr/windows-hardware/drivers/debugger/debugger-download-tools)를 설치합니다. 호스트 Windows에는 Windbg Preview를 설치해 사용해도 되지만 Windbg Preview는 kdnet이 없기 때문에 게스트 Windows에는 Windbg를 SDK로 설치해야 합니다!

![windows-driver/tempsnip%201.png](windows-driver/tempsnip%201.png)

Windbg 설치를 끝냈으면 게스트 VM 설정→Network Adapter에서 Network connection을 Bridged로 설정해 호스트와 통신할 수 있게 설정한 뒤 통신이 잘 되는지 테스트를 해봅니다.

![windows-driver/Untitled%207.png](windows-driver/Untitled%207.png)

핑 테스트 할 때는 호스트와 게스트 모두 방화벽에서 ICMP 인바운드 허용하는 것 잊지 마세요!

![windows-driver/Untitled%208.png](windows-driver/Untitled%208.png)

정상적으로 통신이 되는 걸 확인했으면 호스트의 host name을 확인합니다.

![windows-driver/Untitled%209.png](windows-driver/Untitled%209.png)

게스트에서 cmd를 관리자 권한으로 실행해 다음과 같이 `kdnet.exe`을 실행합니다. 이때 포트는 50000 ~ 50039 범위 안에서 줍니다.

```
> kdnet.exe <host name> 50001
```

![windows-driver/Untitled%2010.png](windows-driver/Untitled%2010.png)

해당 머신을 디버깅하기 위해 커맨드를 호스트 머신에서 실행하라고 하네요! 빨간 박스의 커맨드를 복사해놓습니다.

이제 호스트의 Windbg Preview를 실행하고 Start debugging → Attach to Kernel → Net 에서 포트와 커맨드를 입력합니다.

![windows-driver/Untitled%2011.png](windows-driver/Untitled%2011.png)

OK를 눌러 Windbg Preview가 대기 상태가 되면 게스트를 `shutdown -r -t 0` 으로 재부팅해줍니다. 

재부팅하면..!

![windows-driver/Untitled%2012.png](windows-driver/Untitled%2012.png)

잘 붙은 것 같죠? 디버기 머신인 게스트 Windows도 확인해볼게요.

![windows-driver/Untitled%2013.png](windows-driver/Untitled%2013.png)

디버거가 붙은 상태에서 부팅되면 우측 하단에 테스트 모드와 빌드 버전 정보가 표시되는 것을 확인할 수 있습니다. 

다음 파트부터는 본격적으로 Windows의 third-party driver와 취약점, one-day 분석까지 알아보도록 하겠습니다 ㅎㅎ