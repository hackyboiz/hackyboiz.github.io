---
title: "[Research] Re:versing으로 시작하는 ghidra 생활 Part 1 - Overview"
author: idioth
tags: [idioth, reversing, ghidra, crackme]
categories: [Research]
date: 2021-02-07 14:00:00
cc: true
index_img: /2021/02/07/idioth/ghidra_part1/thumbnail.png
---
**다른 파트 보러가기**

Re:versing으로 시작하는 ghidra 생활 Part 1 - Overview (Here!)

[Re:versing으로 시작하는 ghidra 생활 Part 2 - Data, Functions, Scripts](https://hackyboiz.github.io/2021/03/07/idioth/ghidra_part2/)

[Re:versing으로 시작하는 ghidra 생활 Part 3 - tips for IDA User](https://hackyboiz.github.io/2021/04/04/ghidra_part3)

---

안녕하세요! idioth입니다. 제가 좋아하는 애니메이션의 이름을 빌려 시리즈의 제목을 정했습니다 하하하! 제 기억이 맞다면 저희 블로그의 첫 research 글은 Ghidra의 플러그인인 AngryGhidra였죠?

사실 저는 ghidra가 처음 공개되었을 때 관심이 많았습니다. 그때는 국가의 요원(?)으로 복무 중이었는데, NSA에서 IDA와 같은 리버스 엔지니어링을 위한 툴을 발표한다! 했을 때 신기했었죠.

![](ghidra_part1/Untitled.png)

깔아보고 사용해보면서 느낀 점은 "아.. 이거 엄청 불편하구나"였습니다. 아무래도 IDA에 익숙해져 있었으니 당연한 수순...

그 후로 잊고 살다가 소집해제를 하고 여러 글들을 보는데 reverse engineering with ghidra, malware analysis with ghidra 등 해외에서는 ghidra를 많이 쓰는 것 같더라고요? 하지만 국내 자료 중에는 ghidra에 대한 자료가 많지 않아서... 제가 공부하거나 찾아본 자료를 토대로 시리즈 물을 진행하려고 합니다.

글은 offical docs와 여러 해외 게시글들의 tip들을 종합하여 진행될 예정입니다. 그러면 이제 시작해보도록 하죠!

# Ghidra란 무엇인가?

![](ghidra_part1/Untitled%201.png)

> 자기 꼬리를 먹으려 하는 용..??

ghidra는 NSA(미국 국가 안보국)에서 만든 소프트웨어 리버스 엔지니어링을 위한 프레임워크입니다. [WikiLeaks의 Vault 7](https://wikileaks.org/ciav7p1/index.html)에 의해서 처음 정체(?)가 공개되었고, [2019년 RSA 컨퍼런스](https://www.rsaconference.com/industry-topics/presentation/come-get-your-free-nsa-reverse-engineering-tool)에서 발표되었습니다.

여러 플랫폼에서 동작하게 하기 위해서 JAVA로 개발되었고, 다음과 같은 5개의 메인 파트를 가집니다.

- Programs
- Plugins
- Tools
- Project Manager
- Server

하나하나가 어떤 동작을 하는지 살펴볼까요?

## Programs

ghidra에서 모든 데이터는 ghidra의 커스텀 데이터베이스를 통해 저장됩니다. Symbols, Bytes, Reference, Comments 등 사용자가 추가한 모든 데이터 또한 Ghidra 프로그램 데이터베이스에 저장이 되죠.

## Plugins

ghidra는 플러그인들의 라이브러리라고 볼 수 있습니다. 각각의 플러그인들은 특정한 기능을 제공하죠. 모든 플러그인들은 서로 소통하며 ghidra 재시작 없이 플러그인의 추가/제거가 가능합니다. 사용자가 원하는 대로 커스텀할 수 있고 사용자가 직접 플러그인을 작성할 수도 있죠.

## Tools

Tools는 플러그인과 플러그인 설정의 모임이라고 생각하면 편합니다. 모든 tools는 커스터마이징이 가능하고 플러그인을 추가해서 원하는 tools를 만들 수도 있습니다.

## Project Manager

이름에서부터 알 수 있듯이, 특정 프로그램 그룹에 대해서 프로젝트, tools, data를 관리합니다. IDA에서는 바이너리만 열면 되는 것과 다르게 ghidra는 프로젝트를 생성하고, 거기에 바이너리를 추가해야 하는 과정이 필요하죠.

## Ghidra Server

ghidra는 협업을 위한 Shared-Project를 제공합니다. 이 공유 프로젝트 데이터들은 ghidra 서버를 통해서 공유할 수 있죠. 파일 버전을 지정할 수도 있고, check out, check in, version history 등의 기능을 제공합니다.

# Ghidra vs IDA Pro

ghidra는 분석 환경을 커스터마이징 할 수 있고 팀 활동을 지원하는 것 등 다양한 장점이 있는데 가장 큰 장점은 **무료**라는 점이죠. IDA Pro의 가격과 비교를 해본다면 엄청난 장점...!

![](ghidra_part1/Untitled%202.png)

개인적인 사용을 하거나 공부 용도로 200만 원가량을 지불하기에는... 크흠

Ghidra가 공개되고 나서부터 사람들은 IDA Pro랑 ghidra의 차이점, 뭐가 더 나은가에 대한 토론이 많이 이루어졌습니다. 어떤 게 더 나은지에 대한 생각은 사람마다 다르므로 차이점에 대해서 한 번 다뤄보도록 할까요?

1. ghidra는 decompiler를 포함하여 모든 것이 무료다!
    - IDA가 무료 버전을 제공하지만 Hex-rays 기능을 제공하지 않는 것을 생각하면 ghidra의 큰 장점 중 하나로 볼 수 있죠.
2. IDA에는 debugger가 있지만 ghidra에는 없다!
    - IDA에는 debugger를 붙여서 디버깅을 가능하지만 ghidra는 디버깅에 대한 기능을 제공하지 않습니다.
3. ghidra는 한 번에 여러 바이너리를 프로젝트에 추가할 수 있다.
    - 애플리케이션과 라이브러리 간의 코드 추적을 쉽게 할 수 있습니다.
4. ghidra는 데이터 흐름 분석을 통해 레지스터나 변수의 데이터가 어디에서 참조되는지 알 수 있다.
5. ghidra보다 IDA가 더 오래 사용되어서, IDA 플러그인이 좀 더 많다.

대략적으로 살펴본 차이점은 이 정도가 있고, Hex-rays와 Ghidra Decompiler의 디컴파일 방식에 대해서도 차이가 존재합니다.

## Hex-rays vs Ghidra Decompiler

### Hex-rays

- [Microcode](https://ko.wikipedia.org/wiki/%EB%A7%88%EC%9D%B4%ED%81%AC%EB%A1%9C%EC%BD%94%EB%93%9C) 기반
- 일부 아키텍처만 지원
- 변수, 데이터, 함수에 대한 xrefer 제공
- 변수 매핑 가능
- 디컴파일러에서 변수 표현을 변경할 수 있음(decimal, hex, char 등)

### Ghidra Decompiler

- [P-code](https://ko.wikipedia.org/wiki/P-%EC%BD%94%EB%93%9C) 기반
- 모든 아키텍처 지원
- xref 제공하지 않음
- 변수 매핑 불가능
- 디컴파일러에서 변수 표현 변경 불가능

IDA Pro와의 차이점에 대해서도 간략하게 알아보았으니, 이제 ghidra를 만나러 가봅시다!

# Ghidra를 설치해보자!

먼저 ghidra를 사용하기 위해서는 설치를 해야겠죠? ghidra는 다음과 같은 플랫폼을 지원합니다.

- Windows 7 or 10 (64-bit)
- Linux (64-bit, CentOS 7 is preferred)
- macOS (OS X) 10.8.3+ (Mountain Lion or later)

32비트 OS에 대한 지원은 일절 제공하지 않으니 참고해주세요!

그리고 위에서 ghidra는 JAVA로 개발되었다고 말씀드렸죠? 따라서 jdk 또한 필요합니다. ghidra의 원활한 사용을 위해서 [jdk 11](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)을 설치해주시면 됩니다!

ghidra 바이너리 설치는 [ghidra 홈페이지](https://ghidra-sre.org/)에서 Download Ghidra 버튼을 눌러 간단하게 설치할 수 있습니다. 별도의 setup 없이 압축을 해제한 후 플랫폼에 따라 `ghidraRun.sh`이나 `ghidraRun.bat`를 클릭해주면 실행이 가능합니다.

![](ghidra_part1/Untitled%203.png)

> ~~뭘 눌러야 될지 모르겠으면 그냥 둘 다 눌러보자~~

Windows의 경우 `ghidraRun.bat`을 통해서 실행하면 되고, Linux나 MacOS는 `ghidraRun.sh`을 통해 실행하면 됩니다.

# 간단한 Crackme를 통한 Ghidra 탐색전

제가 Windows 머신을 사용하므로 Windows 기준으로 진행됩니다. Linux와 MacOS에서도 실행 방식의 차이만 존재할 뿐, 차이가 없으니 걱정 안 하셔도 됩니다! 이 게시글에서 사용한 예제 파일은 [여기](https://crackmes.one/crackme/6004daf233c5d42c3d016717)에서 다운로드하실 수 있습니다.(압축 파일의 비밀번호는 `crackmes.one`입니다.)

먼저 `ghidraRun.bat`이나 `ghidraRun.sh`을 통해 ghidra를 실행하면 다음과 같은 프로젝트 창이 뜹니다.

![](ghidra_part1/Untitled%204.png)

프로젝트 창에서는 프로젝트를 생성하고 바이너리를 로드, 구성할 수 있습니다. 새 프로젝트를 만든 후 바이너리를 드래그 앤 드롭을 하여 추가를 할 수 있습니다.

![](ghidra_part1/Untitled%205.png)

File - New Project (Ctrl+N)을 통해 새로운 프로젝트를 생성하고 위에서 다운로드한 예제 파일을 드래그 앤 드롭하거나 Import File을 통해 추가하여 줍니다.

![](ghidra_part1/Untitled%206.png)

바이너리 import가 완료되면, Import Summary 창을 통해 간략한 정보를 제공합니다. 해당 정보에서 컴파일러, Compiler ID 등등을 통해 대략적인 바이너리에 대한 정보를 얻을 수 있습니다.(악성코드 분석을 할 때 좋겠네요.)

이제 프로젝트 창에서 crackme.exe를 더블클릭하면 드래곤(!) 애니메이션이 뜨고 분석을 진행하겠냐고 묻는 창이 뜹니다. 당연히 Yes를 클릭해줍니다!

![](ghidra_part1/Untitled%207.png)

![](ghidra_part1/Untitled%208.png)

Yes를 클릭하면 ghidra에서 가능한 분석 옵션을 보여줍니다. 개별적인 세팅은 무시하고 기본 값을 그대로 사용하셔도 큰 문제는 없습니다.

![](ghidra_part1/Untitled%209.png)

분석이 완료되고 난 후 메인 화면은 위와 같습니다. 이제 메인 화면에 각 섹션이 어떠한 기능을 하는지 살펴봅시다!

## Program & Symbol Trees, Data Type Manager

### Program Trees

Program Trees를 활용하면 원하는 섹션으로 트리를 나눌 수 있습니다.

![](ghidra_part1/Untitled%2010.png)

![](ghidra_part1/Untitled%2011.png)

> Dominance를 통해 구성된 코드 섹션 예시

### Symbol Tree

![](ghidra_part1/Untitled%2012.png)

Symbol Tree의 경우 Import 된 DLL, Exports 된 함수 등등의 정보를 제공합니다. 특정 함수가 어디에서 참조되는지 확인할 수도 있습니다.

![](ghidra_part1/Untitled%2013.png)

![](ghidra_part1/Untitled%2014.png)

### Data Type Manager

![](ghidra_part1/Untitled%2015.png)

Data Type Manager에서는 기본적으로 제공되거나, 바이너리에서 제공하는 데이터 타입을 확인할 수 있습니다. 데이터 타입을 우클릭하고 "Find Uses of"(`Ctrl + Shift + F`)를 클릭하여 해당 바이너리 내에서 데이터 유형이 사용되는 곳을 확인할 수 있습니다.

![](ghidra_part1/Untitled%2016.png)

![](ghidra_part1/Untitled%2017.png)

## Disassembly View & Function Graph

![](ghidra_part1/Untitled%2018.png)

중앙에 큰 화면은 디스 어셈블 된 코드를 확인할 수 있는 창입니다. 여러 필드가 존재하고, 어떤 부분에 무엇이 표시될지 등 모두 사용자가 커스터마이징 할 수 있습니다.

상단 표시줄에 존재하는 "Edit the Listing fileds"를 클릭하고 "Instruction/Data"를 클릭하면 수정이 가능합니다. 크기 조정, 위치 이동, 비활성화, 새로운 기능 추가 등 다양하게 커스터마이징 할 수 있습니다.

![](ghidra_part1/Untitled%2019.png)

IDA에서 함수 이름을 수정할 수 있는 것처럼 ghidra에서는 `L` 단축키를 이용하여 함수의 이름을 수정할 수 있습니다.

![](ghidra_part1/Untitled%2020.png)

또한 함수의 이동 없이 해당 함수가 어떠한 코드로 되어있는지 확인하고 싶으면 함수 이름에 마우스 커서를 올려놓고 기다리면 해당 내용을 확인할 수 있습니다.

![](ghidra_part1/Untitled%2021.png)

원하는 부분 우클릭-Comments- 원하는 Comment를 통해 주석도 추가가 가능합니다. `;` 단축키를 사용하여 원하는 부분에 바로 주석을 달 수도 있습니다.

![](ghidra_part1/Untitled%2022.png)

> `;`를 눌렀을 때 나오는 창. 해당 창에 주석을 입력하고 'OK'를 누르면 된다.

IDA Pro를 사용할 때 가장 편했던 기능 중 하나가 그래프 모드였죠? ghidra에서는 디스 어셈블 창에서 바로 그래프 뷰를 확인할 순 없지만 "Windows" - "Function Graph"를 통해 그래프 뷰를 사용할 수 있습니다.

![](ghidra_part1/Untitled%2023.png)

![](ghidra_part1/Untitled%2024.png)

> ghidra에서 제공하는 function graph. IDA Pro와 큰 차이가 나진 않는다.

![](ghidra_part1/Untitled%2025.png)

Function Graph 또한 디스 어셈블 창과 마찬가지로 표시되는 내용을 커스터마이징 할 수 있습니다. 우측 상단의 "Edit Code Block Fileds"를 클릭한 후 "Instruction/Data"에서 아무 공간이나 우클릭-Add Field로 원하는 코드 블록을 추가할 수 있습니다.

## Decompiler

이제 IDA Pro의 hex-rays 기능에 해당하는 디컴파일러 창입니다. 기본적으로 디스 어셈블러 창 우측에 위치하며, 크기를 조정할 수 있고 위치 조정 또한 가능합니다.

![](ghidra_part1/Untitled%2026.png)

함수 내부를 클릭하면 자동으로 함수 내용이 디컴파일 돼서 우측 디컴파일 창에 표시됩니다. 위의 사진처럼 Function Graph를 끌어다 넣어서 디스 어셈블 창이 아닌 Function Graph 창이 좌측에 표시되게 커스터마이징도 할 수 있죠!

![](ghidra_part1/Untitled%2027.png)

위에서 설명드린 것과 마찬가지로 디컴파일러 창에서도 변수나 함수의 이름을 수정할 수 있고 주석을 달 수도 있습니다. 디컴파일러 창에서 수정한 내용은 디스 어셈블 창에도 적용이 되고, 디스 어셈블 창에서 수정한 내용도 디컴파일러 창에 적용이 됩니다.

## Crackme를 풀어보자!

이제 ghidra의 대략적인 기능도 확인을 해보았으니 Crackme를 풀어볼 수 있을 것 같습니다. 일단 파일을 실행해서 어떠한 녀석인지 확인을 해봅시다.

![](ghidra_part1/Untitled%2028.png)

맞는 Input 값을 찾아야 하는 문제군요.(~~Crackme니까... 당연하겠지~~) ghidra를 확인해서 맞는 input이 무엇인지 찾아보도록 합시다!

ghidra를 실행한 후 Project 생성 후 바이너리를 추가하고, 분석까지 완료한 화면에서 main 화면을 찾아봅시다. String 검색을 통해 찾을 수도 있지만(그게 더 간단하지만), Symbol Tree를 통해 찾아봅시다. 좌측 Symbol Tree에서 Function 옆 + 버튼을 눌러 폴더를 열어줍니다.

![](ghidra_part1/Untitled%2029.png)

`___main`이 존재하는 것을 확인할 수 있습니다. 밑으로 더 스크롤해서 다른 함수를 확인해보면

![](ghidra_part1/Untitled%2030.png)

`_main`과 `_mainCRTStartup`이 존재하는 걸 확인할 수 있습니다. Entry Point인 `_mainCRTStartup`을 통해 어떤 게 처음 시작되는 main 함수인지 확인해봅시다.

![](ghidra_part1/Untitled%2031.png)

`_mainCRTStartup`의 내용을 확인해보니 `___mingw_CRTStartup` 함수를 호출합니다. 해당 함수도 탐색을 해볼까요? 더블클릭을 하면 해당 함수로 이동할 수 있습니다.

![](ghidra_part1/Untitled%2032.png)

`___main` 함수와 `_main` 함수를 차례대로 실행을 합니다. 위에 함수부터 들어가 봅시다.

![](ghidra_part1/Untitled%2033.png)

![](ghidra_part1/Untitled%2034.png)

음... 실행했을 때 나오는 문자열을 출력하는 부분이 없는 걸 보아 `_main`이 진짜 메인 함수인 것 같네요. 뒤로 가거나 앞으로 가는 것은 좌측 상단의 버튼을 사용하거나 `ALT + 좌우 방향키`를 사용하시면 됩니다.

![](ghidra_part1/Untitled%2035.png)

`_main`으로 오니 위에서 확인했던 문자열도 출력되고, 입력 값과 비교하는 부분도 있네요. 아무래도 난이도 1짜리 간단한 문제이다 보니 바로 풀려버립니다ㅋㅋㅋ. 뭐 ghidra를 어떻게 사용하는지에 대해서 설명하려고 가져온 문제니까! `0x1232b14 = 19082004`이므로 입력을 해주면...

![](ghidra_part1/Untitled%2036.png)

정답임을 알 수 있습니다.

# 마치며...

오늘은 간단하게 ghidra란 무엇인지, IDA랑 비교했을 때 어떠한 차이점이 있는지, ghidra는 어떠한 기능들을 가지고 있는지에 대해 Easy Crackme를 풀어보며 간단히 살펴보았습니다.

Part 2에서는 오늘 간단하게 짚고 넘어간 Data, Function, Scripts에 대해서 좀 더 다뤄볼 예정입니다. 국내에는 아직 IDA Pro 사용자가 많지만 ghidra를 사용해보고 싶으신 분들이 계시다거나, IDA Pro를 접하기 힘들어서 ghidra를 써보고 싶은데 잘 모르겠다 하시는 분들에게 큰 도움이 됐으면 하며...

잘못된 설명이나, 부족한 부분이 있다면 언제든 댓글로 남겨주시면 반영하고 수정하도록 하겠습니다. 그럼 다들 좋은 밤 보내세요~! 저는 이제 자러 가 보도록 하겠습니다!

![](ghidra_part1/Untitled%2037.png)

~~밤에 깨어있고 낮에 자는 뱀파이어 같은 모순적인 삶...~~

# Reference

[https://dannyquist.github.io/gootkit-reversing-ghidra/](https://dannyquist.github.io/gootkit-reversing-ghidra/)

[https://www.shogunlab.com/blog/2019/04/12/here-be-dragons-ghidra-0.html](https://www.shogunlab.com/blog/2019/04/12/here-be-dragons-ghidra-0.html)

[https://ghidra.re/online-courses/](https://ghidra.re/online-courses/)