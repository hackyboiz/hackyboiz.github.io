---
title: "[Research] Re:versing으로 시작하는 ghidra 생활 Part 2 - Data, Functions, Scripts"
author: idioth
tags: [idioth, reversing, ghidra, ghidra tutorials]
categories: [Research]
date: 2021-03-07 14:00:00
cc: true
index_img: /2021/03/07/idioth/ghidra_part2/thumbnail.png
---
**다른 파트 보러가기**

[Re:versing으로 시작하는 ghidra 생활 Part 1 - Overview](https://hackyboiz.github.io/2021/02/07/idioth/ghidra_part1/)

Re:versing으로 시작하는 ghidra 생활 Part 2 - Data, Functions, Scripts (Here!)

[Re:versing으로 시작하는 ghidra 생활 Part 3 - tips for IDA User](https://hackyboiz.github.io/2021/04/04/idioth/ghidra_part3)

[Re:versing으로 시작하는 ghidra 생활 Part 4 - Malware Analysis (1)](https://hackyboiz.github.io/2021/05/19/idioth/ghidra_part4/)

[Re:versing으로 시작하는 ghidra 생활 Part 5 - Malware Analysis (2)](https://hackyboiz.github.io/2021/07/11/idioth/ghidra_part5/)

---

![](ghidra_part2/Untitled%201.png)

파트 2로 돌아왔습니다. 요즘 날씨가 많이 따뜻해졌네요. 최근 제가 하던 게임~~(유사 도박)~~에서 이슈가 터져서 공중파에도 출현하고 있습니다. 닉네임답게 주위에서 하는 말을 무시한 채 즐기고 있었습니다만... 뭐 아무튼 오랜 시간 즐겼던 입장이라 마음이 아프네요. ~~혹시 저와 같은 게임을 하시던 분이라면 잃어버린 아크를 찾으러~~

딴 소리는 그만하고 [저번 파트](https://hackyboiz.github.io/2021/02/07/idioth/ghidra_part1/)에서는 ghidra에 대한 전반적인 것에 대해서 알아보았습니다. 이번 파트는 데이터 변환, 데이터 타입 적용, 함수 호출 트리 및 그래프, 스크립트 매니저와 메모리 맵에 대해서 다룰 예정입니다.

이번 시간에 예제로 다룰 바이너리는 총 두 가지입니다. [Part 1](https://hackyboiz.github.io/2021/02/07/idioth/ghidra_part1/)에서 다루었던 바이너리와 [reversing.kr](http://reversing.kr/)에 업로드된 Easy Keygen 문제입니다.

# 데이터 변환

분석을 수행할 때 각자 선호하는 데이터 타입이 있을 수도 있고, 아니면 부정확하게 데이터 타입이 표시되는 경우가 있습니다. 10진수로 보고 싶은데 16진수로 표시되어 있거나 그 반대인 경우가 있죠.

자 일단 ghidra를 통해 crackme.exe 바이너리 파일을 연 후 main 함수로 들어가 봅시다. 좌측 Symbol Tree에서 Functions 폴더를 확장하고 `_main` 함수를 찾아서 클릭해주시면 중앙 화면과 디스 어셈블 창에서 main 함수를 출력해줍니다.

![](ghidra_part2/Untitled%202.png)

![](ghidra_part2/Untitled%203.png)

저번 파트에서 입력 값이 `0x1232b14`여야 하는 점은 알았습니다. 근데 그냥 대충 쓱 훑어보고 `0x1232b14`에서 0x를 뺀 `1232b14`를 적는다면?

![](ghidra_part2/Untitled%204.png)

Fail 문구를 봐버립니다... 계산기를 켜고 입력해서 10진수로 변환하기 귀찮고, python을 켜서 `0x1232b14`를 10진수로 변환하기도 귀찮은데 ghidra에서 바로 변환할 수 있는 방법은 없을까???

![](ghidra_part2/Untitled%205.png)

변환할 수 있는 방법이 없다면 IDA를 사용하지 왜 ghidra를 사용하겠습니까? 당연히 존재합니다!!!

![](ghidra_part2/Untitled%206.png)

커서를 자신이 변환하길 원하는 데이터에 클릭을 합니다. 그러면 Listing 창은 커서가 위치한 곳에 해당하는 어셈블리로 자동으로 이동하게 됩니다.

![](ghidra_part2/Untitled%207.png)

![](ghidra_part2/Untitled%208.png)

그 후 원하는 데이터 값을 우클릭 - Convert - 원하는 타입으로 클릭하면 데이터 타입이 변경됩니다. 값이 정상적으로 바뀌었는지 한 번 확인을 해봅시다!

![](ghidra_part2/Untitled%209.png)

값이 정상적으로 변환된 것을 볼 수 있습니다. :)

# 데이터 타입 적용

ghidra에서 바이트 배열은 조금 독특하게 표시됩니다. 대부분 crackme 문제를 풀기 위해서, 혹은 분석하기 위해서는 이 바이트 배열이 문자열인지 아닌지를 판별해야 합니다. 물론 ghidra에서는 바이트 배열을 문자열로 변환하는 것 같은 데이터 타입 적용이 존재합니다. :)

이번 부분에 대해서 살펴보기 전에 Easy Keygen 바이너리를 ghidra로 열어봅시다!

![](ghidra_part2/Untitled%2010.png)

![](ghidra_part2/Untitled%2011.png)

main 함수로 가기 위해서 entry point에서 `FUN_00401000`을 클릭하여 이동하여 줍니다.

![](ghidra_part2/Untitled%2012.png)

main 함수를 살펴보던 중 `&DAT_0040805c`라는 녀석을 만났습니다. 이 녀석은 무슨 값일까... 한 번 확인해봅시다.

![](ghidra_part2/Untitled%2013.png)

`%s`였네요. 근데 데이터 타입이 정의되어 있지 않습니다. 디 컴파일 창만 본다면 한눈에 어떤 값인지 들어오지가 않습니다. 이 녀석을 한 번 변환시켜 볼까요?

![](ghidra_part2/Untitled%2014.png)

`DAT_0040805c` 우클릭 → Data → string을 누르면?

![](ghidra_part2/Untitled%2015.png)

음... 뭔가 한눈에 확 들어오진 않지만 어떤 값인지는 대충 알 수 있게 변했습니다.

# 함수 호출 트리와 호출 그래프

분석을 하다 보면 이 함수가 어디에서 호출되는지, 이 함수가 어떤 함수를 호출하는지 확인해야 할 때가 있습니다. 함수 호출 트리를 통해 이 작업을 간단하게 수행할 수 있습니다. Easy Keygen 바이너리에서 메인 함수가 어떤 함수에 의해서 호출이 되는지, 메인 함수는 어떤 함수를 호출하는지 함수 호출 트리를 통해서 확인을 해볼까요?

![](ghidra_part2/Untitled%2016.png)

![](ghidra_part2/Untitled%2017.png)

main 함수에 위치한 채로 Window → Function Call Tress를 클릭하면 기본적으로 화면 하단에 함수 호출 트리가 나타나게 됩니다.(저는 이미지가 좌우로 길게 늘어져서 화면 바깥으로 뺐습니다...^^;;)

함수 호출 트리를 보면 `FUN_00401000`은 `entry`에 의해서 호출되었고, `FUN_00401150`, `FUN_004011a2`, `FUN_004011b9`를 호출하는 것을 볼 수 있습니다. 이런 식으로 한눈에 확인할 수 있겠지만 뭔가 그림으로 되어있으면 좀 더 알아보기 편하겠죠? 그렇다면 Window → Function Call Graph를 클릭해줍니다!

![](ghidra_part2/Untitled%2018.png)

좀 더 알아보기 쉽게 어떤 함수에 의해서 호출되고 어떤 함수를 호출하는지 한눈에 확인할 수 있습니다. 물론 이런 창들은 모두 커스터마이징 해서 원하는 위치에 둘 수 있습니다.

![](ghidra_part2/Untitled%2019.png)

이런 식으로도 구성할 수 있죠. 함수를 이동할 때마다 어떤 함수를 호출하는지 확인할 수 있는 겁니다.

![](ghidra_part2/Untitled%2020.png)

# 스크립트 매니저

점점 ghidra 생활이 편리해지고 있습니다. ghidra만 있으면 어셈도 보고 디컴파일도 보고 함수 그래프도 보고 데이터 타입도 내가 원하는 대로 이것저것 수정하면서 모든 걸 다 할 수 있을 듯한 느낌입니다.

근데 아직 뭔가 허전합니다... 문제를 풀다가 암호문이 나와서 스크립트를 짜야할 것 같은데... 난 ghidra만 있으면 뭐든지 다 할 수 있을 것이라 생각했는데 vscode를 켜서 스크립트를 작성하고 cmd를 켜서 python을 실행하여 스크립트를 실행합니다. 기분이 나쁩니다. ghidra를 통해서 뭐든 해내고 싶습니다.

![](ghidra_part2/Untitled%2021.png)

욕심을 부리고 싶습니다. 툴 하나로 다 해버리고 싶습니다. 혼자서 다 때려 부수고 다니는 저기 저 슬라임처럼요. 그렇다면 스크립트 매니저가 여러분을 도와줄 겁니다! ghidra에서는 IDA Python처럼 JAVA와 python 등을 실행할 수 있는 스크립트 매니저가 존재합니다. 백문이 불여일견! 눈으로 직접 확인해봅시다.

![](ghidra_part2/Untitled%2022.png)

![](ghidra_part2/Untitled%2023.png)

Window → Script Manager를 클릭하면 스크립트 매니저 창이 실행되고 굉장히 많은 스크립트들이 나타납니다. 카테고리로 많이 분류가 되어있습니다. 근데 나는 여기 있는 거 말고 내가 직접 짠 python 스크립트를 실행하고 싶습니다... `Hello, ghidra World!`를 출력하고 싶습니다. 그렇다면 한 번 스크립트를 직접 작성해볼까요?

![](ghidra_part2/Untitled%2024.png)

![](ghidra_part2/Untitled%2025.png)

먼저 Bundle Manager를 실행시켜서 스크립트들의 경로를 확인해봅시다. `$USER_HOME/ghidra_scripts`에 내가 작성한 스크립트가 저장될 것 같은데 `file not found`라고 뜹니다. 음... 바탕화면에 스크립트를 저장할 디렉터리를 만들고 거기에 작성한 스크립트를 넣어봅시다.

![](ghidra_part2/Untitled%2026.png)

`USER_HOME`에 ghidra_scripts 폴더를 만들었으니 이제 스크립트를 생성해보도록 합시다.

![](ghidra_part2/Untitled%2027.png)

번들 매니저에서 초록색 + 버튼을 누른 후 바탕화면의 ghidra_scripts 폴더를 지정하면...

![](ghidra_part2/Untitled%2028.png)

추가가 되었습니다. 이제 해당 디렉터리에 Python 스크립트를 작성한 후 저장해봅시다.

![](ghidra_part2/Untitled%2029.png)

![](ghidra_part2/Untitled%2030.png)

![](ghidra_part2/Untitled%2031.png)

양피지 모양을 클릭하고 Python 스크립트를 작성할 것이니 Python을 체크하고 스크립트 이름을 입력해줍니다. 만약 추가한 디렉터리가 여러 개라면 디렉터리 선택 부분에 여러 개의 디렉터리가 뜨겠죠? :) OK 버튼을 클릭해줍시다.

![](ghidra_part2/Untitled%2032.png)

이런 식으로 편집창이 뜨고 원하는 스크립트를 작성만 하면 되는 것 같습니다.

 

![](ghidra_part2/Untitled%2033.png)

코드를 작성하고 저장한 후 실행시켜봅시다!

![](ghidra_part2/Untitled%2034.png)

아래 콘솔 창에 `hello.py`가 실행돼서 정상적으로 문구가 뜨는 것을 보실 수 있습니다.

# 메모리 맵

이제 벌써 마지막이네요... 메모리 맵입니다. 리버싱을 하거나 아니면 포너블을 하다 보면 메모리 블록들이 어떠한 권한을 가지고 있는지 확인해야 할 때가 있습니다. 혹은 Base Address가 변경되었을 때 함수가 어떤 주소를 갖는지 계산을 해야 할 때도 있죠. 그럴 때 역시 다른 도구를 사용하지 않고 ghidra에서 해치울 수 있습니다.

![](ghidra_part2/Untitled%2035.png)

![](ghidra_part2/Untitled%2036.png)

Window - Memory Map을 클릭하여 메모리 맵을 실행 하면 위와 같은 창을 볼 수 있습니다. 어떤 블록이 무슨 권한을 가지고 있는지, 초기화가 되어있는지 등이 모두 표시되죠. 베이스 주소를 변경한 채로 함수 주소들을 확인하고 싶으면 상단의 집 모양을 눌러서 Base Address를 수정해주기만 하면 됩니다.

![](ghidra_part2/Untitled%2037.png)

![](ghidra_part2/Untitled%2038.png)

![](ghidra_part2/Untitled%2039.png)

Base Image Address를 0으로 바꾸어 버렸더니 모든 함수들이 오프셋 주소로만 표시되는 모습을 볼 수 있습니다.

# The End

오늘은 데이터 변환, 데이터 타입 적용, 함수 호출 트리/그래프, 스크립트 매니저, 메모리 맵에 대해서 알아보았습니다. 이제 ghidra에 대해서는 모든 사용법을 획득한 것 같습니다. 무서운 용이라고 생각하고 레이드 할 생각에 들떠 있었는데 생각보다 싱거웠네요.

하지만! 원래 몬스터를 한 번 수렵 했으면 이제 자잘한 팁이나 TA(타임어택)을 하는 것이 강호의 도리! The End라는 문구만 보고 끝났다고 생각하셨다면~~(끝나길 바라셨을 수도 있겠지만)~~ 서운합니다... 다음 파트에서는 IDA Pro를 사용했던 사용자들에 대한 ghidra Tips 모음집을 들고 오도록 하겠습니다. 그럼 다들 좋은 하루 보내시길 바랍니다!



# Reference

https://www.shogunlab.com/blog/2019/12/22/here-be-dragons-ghidra-1.html

https://ghidra.re/courses/GhidraClass/Intermediate/Scripting_withNotes.html#Scripting.html

https://ghidra-sre.org/CheatSheet.html