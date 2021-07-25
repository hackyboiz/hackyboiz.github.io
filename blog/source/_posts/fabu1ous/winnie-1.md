---
title: "[Research] "
author: Fabu1ous
tags: [Fabu1ous, winnie, fuzzing, harness, bug bounty, windows]
categories: [Research]
date: 2021-07-25 14:00:00
cc: true
index_img: /2021/07/25/fabu1ous/winnie-1/thumb.png
---



# 머릿말

안녕하세요. Fabu1ous 입니다. 제가 가장 최근에 올린 연구글 시리즈 "WinAFL로 마구 퍼징 하기"에서 의미 있는 퍼징 속도가 나오는 Harness를 작성하기 위해 고통받는 모습을 보여드렸죠.

![](/winnie-1/1.png)

시리즈 막바지엔 이게 정녕 "자동" 취약점 탐색이 맞냐면서 징징 거리기까지 했습니다. 그만큼 잘 작동하는 Harness를 제작하기 위해선 큰 노력이 필요한데 이 사실을 불편하게 느낀 게 저만은 아니더라고요. 최근에 Georgia Tech과 PennState Univ에서 Winnie라는 새로운 솔루션을 공개했습니다. Harness를 자동 생성한다는 점과 많은 퍼저들이 빠른 속도를 내기 위해 채택한 Persistent mode(프로세스를 종료하지 않는 퍼징 방식)와는 다른 방식을 사용하는 것이 흥미로웠습니다. 그래서, 새로운 시리즈로 찾아왔습니다.



# Windows 환경의 Fuzzing 문제점

![](/winnie-1/2.png)

운영 체제의 종류는 정말 다양합니다. Windows, Mac, Linux, Android, 붉은별... 그 중 Windows가 약 73%라는 압도적인 시장 점유율을 갖고 있습니다. 그만큼 Windows 환경에서의 Product security가 중요하다고 할 수 있습니다. 하지만 Windows 환경에서의 Fuzzing은 몇 가지 장애물이 있습니다.

- GUI

  실제로 퍼징 하게 될 코어 루틴 앞 뒤로 User interaction을 담당하는 GUI가 있기 때문에 User interaction을 재현하는 등의 작업이 필요하고 프로그램을 종료시킬 시점을 판단하는 별도의 동작이 필요합니다. User interaction을 재현하려는 시도가 있지만 속도 저하 문제로 큰 성과는 없는 것 같습니다.

- Lack of fast cloning

  Windows는 `fork()`와 같은 메커니즘이 미흡합니다. Linux는  `fork()`를 통해 현재 프로세스와 동일한 또 다른 프로세스를 빠르게 복제해 퍼징을 이어나가지만 Windows는 프로세스를 다시 startup 합니다. 따라서 같은 Persistent mode fuzzing이라고 부르지만 Unix-like와 Windows 사이에 차이가 발생합니다.

- Closed-source

  현존하는 퍼징 기술들이 Unix-like 운영체제 중심으로 발전해왔고 Windows에 적용하는데 한계가 있는 가장 큰 이유가 아닐까 싶습니다. Windows는 Closed-source ecosystem의 특성상 내부 context를 알아내는데 어려움이 있습니다. 또한 2006년, 코드 커버리지에 기반한 퍼징 방법론이 제시된 이후 퍼징의 중심 개념으로 자리 잡게 되었는데 Windows 환경에서는 DynamoRIO나 IntelPT 기술을 접목해야 하는 등의 차이가 있습니다.

  

# 기존 해결법

![](/winnie-1/3.png)

- Harness 작성

  프로그램의 코어 루틴만을 추출해 Harness를 작성하는 것으로 GUI를 생략할 수 있습니다. 퍼징 하는 전체 프로그램 크기를 줄여 속도 면에서도 이득을 볼 수 있죠. 하지만 저번 시리즈에서 제가 WinAFL을 사용하기 위해 Harness를 작성하면서 정말 골머리를 앓았죠. Harness를 작성하기 위해 필요한 정보 수집부터 쉬운 일이 아니며 테스트와 수정도 여러 번 했습니다. 심지어 제가 작성한 Harness는 최대한 날로 먹을 수 있는 수준으로 간단하게 작성한 겁니다. 신속하고 정확한 Harness를 만들기 위해선 대단한 노력이 필요합니다.

- persistent fuzzing

  매 iteration 마다 프로그램을 startup 하는 방식은 속도 저하가 심합니다. Loop를 통해 프로세스 초기 상태로 되돌리는 persistent mode fuzzing을 통해 이를 해결하고 있죠. 하지만 persistent mode도 완벽한 것은 아닙니다. 각 iteration 마다 조금씩 프로그램의 global state가 오염되고, 따라서 일정 iteration이 지나면 프로그램을 재시작해야 합니다. 위에서 언급한대로 Unix-like는 `fork()`를 통해 프로세스를 복제하지만 Windows는 다시 처음부터 startup 해야 하는 차이를 갖습니다. `fork()`의 경우 각 child process 마다 독자적인 address space를 갖기 때문에 crash나 hang을 검사하는데 좀 더 안정적입니다.

  

# 그래서 Winnie는

![](/winnie-1/4.png)

- Semi-automated harness generation

  User interaction과 termination 문제를 해결하기 위해 작성하던 Harness를 알아서 만들어 줍니다.

- Windows version of `fork()` mechanism

  프로세스 생성 과정을 리버싱 해 `fork()`를 Windows로 포팅했습니다.

- Hybrid analysis and fullspeed fuzzing

  구현된 `fork()`를 사용하고 basic block 단위 커버리지 기록을 합니다.

  

# Semi-automated harness generator

![](/winnie-1/5.png)

<s>반자동 소총 아니구요. 반자동 하네스 생성기 입니다.</s>

Winnie는 크게 Harness generator와 Fuzzer로 구성되어 있습니다. 그중 첫 번째인 Harness generator에 대해 알아보겠습니다.

- Trace collector

  Run trace를 캡처하는 Trace collector입니다. 메인 프로그램과 라이브러리, 라이브러리와 API 간의 교류를 모니터링하고 덤핑 합니다. API call, Argument Value, Retrun Value를 수집합니다.

- Target extractor

  타겟 함수 후보를 선정하는 Target extractor입니다. Dynamic Launch trace를 수집하고 그로부터 control flow gragh를 재구성 합니다. `CreateFile` 또는 `ReadFile`과 같이 중요한 API에 도달하는 함수를 퍼징 후보로 선정합니다.

- Harness Builder

  선정된 타겟 함수 후보를 바탕으로 Harness를 생성하는 Harness builder입니다. 퍼징 후보의 위치를 정한 후 valid input과 invalid input을 사용해 run trace 수집하고 differential analysis를 통해 분석합니다. Valid input을 사용한 run trace 끼리 비교하면 ASLR 때문에 달라진 포인터를 분석할 수 있습니다. 그리고 Valid input을 사용한 run trace와 invalid input을 사용한 run trace를 비교해 코어 로직을 추출합니다. Harness 자동 생성과 평가 과정은 논문의 중심 내용으로 제가 설명하기엔 벅차더라고요... ㅎㅎ. 일단 간단히 알아보고 직접 사용해보는 게 목적이라 이 글에선 다루지 않겠습니다.

  

# Fuzzer

논문에선 코드 커버리지 수집을 위한 Fullspeed fuzzing 기능을 소개하고 있습니다. Basic block 단위로 s/w break point를 설정하고 input이 새로운 basic block에 도달하면 break point 인터럽트를 통해 커버리지를 기록합니다.

![](/winnie-1/6.png)

책을 보면서 컴파일러 공부를 잠시 했었는데 여기서 굉장히 의아했습니다.

위키피디아에 따르면 basic block의 정의가 다음과 같음을 생각해 보면 엄청난 속도 저하를 예상하실 수도 있는데, (Control flow gragh를 그리는 기본 단위 정도로 생각하시면 됩니다.)

> a straight-line code sequence with no branches in except to the entry and no branches out except at the exit

한번 인터럽트가 발생한 break point를 비활성화하는 방식으로 같은 커버리지를 기록하는 일을 없애고 새로운 basic block에 도달할 때만 오버헤드가 발생하게 됩니다.

또한 `fork()`를 사용해 프로세스를 복제하기 때문에 break point를 삽입과 비활성화 작업을 새로운 프로세스에 적용할 필요가 없어집니다.

![](/winnie-1/7.png)

Winnie야 너는 `fork()`가 있구나!

Winnie는 현존하는 다른 솔루션보다 약 310% 정도 코드 커버리지가 높다고 합니다.



# 마치며

이번 글에선 Winnie의 전체적인 아이디어를 훑어봤습니다. 지금까지 퍼저를 설치하고 사용만 해봤지 논문까지 읽으면서 깊게 알아본 건 처음이네요. 물론 이 정도도 겉 핡기이긴 하지만 ㅎㅎ. 기본적인 코드 커버리지 개념과 컴파일러 이론 복습도 하는 계기가 됐습니다.

감사하게도 Winnie는 github에 오픈소스 되어있습니다. 다음 파트에선 환경 설정을 해보고 직접 사용해 봐야겠죠? Winnie가 Harness를 알아서 만들어 준다고는 했는데 털 끝 하나 정도는 조금 손봐야 한다고 그러네요.