---
title: "[Research] Re:versing으로 시작하는 ghidra 생활 Part 3 - A few tips for IDA User"
author: idioth
tags: [idioth, reversing, ghidra, ghidra tutorials]
categories: [Research]
date: 2021-04-04 14:00:00
cc: true
index_img: /2021/04/04/idioth/ghidra_part3/thumbnail.jpg
---

**다른 파트 보러가기**

[Re:versing으로 시작하는 ghidra 생활 Part 1 - Overview](https://hackyboiz.github.io/2021/02/07/idioth/ghidra_part1/)

[Re:versing으로 시작하는 ghidra 생활 Part 2 - Data, Functions, Scripts](https://hackyboiz.github.io/2021/03/07/idioth/ghidra_part2/)

Re:versing으로 시작하는 ghidra 생활 Part 3 - tips for IDA User (Here!)

[Re:versing으로 시작하는 ghidra 생활 Part 4 - Malware Analysis (1)](https://hackyboiz.github.io/2021/05/19/idioth/ghidra_part4/)

[Re:versing으로 시작하는 ghidra 생활 Part 5 - Malware Analysis (2)](https://hackyboiz.github.io/2021/07/11/idioth/ghidra_part5/)

---

안녕하세요! 오랜만에 찾아뵙습니다. 갑자기 날씨가 엄청나게 더워졌습니다. 봄 옷을 꺼낼 겨를도 없이 갑자기 여름이 찾아온 것만 같은 느낌입니다.

![](ghidra_part3/Untitled.png)

일교차는 또 엄청 커서 밤을 생각하고 옷을 입으면 낮에는 불타는 삶을 살고 있습니다. 특히 BMW가 너무 덥습니다. Bus와 Metro는 사람들의 열기로 후끈후끈하고... 이 날씨에 Walk를 하면 마스크에도 땀이 차는...

![](ghidra_part3/Untitled%201.png)

날씨도 덥고 피곤하니 시원한 아메리카노 한 잔 마시면서 오늘의 주제인 IDA 사용자를 위한 ghidra 팁! 을 시작해보겠습니다!

# API call parameter 자동 코멘트

![](ghidra_part3/Untitled%202.png)

IDA의 경우 위와 같이 Windows API가 호출될 경우, 각 파라미터의 이름을 코멘트로 달아줍니다.

![](ghidra_part3/Untitled%203.png)

하지만 analysis를 할 때 기본 설정으로 진행을 한다면 ghidra는 코멘트를 달아주지 않습니다. 물론 `memset`이라 적힌 부분에 있는 코멘트를 보면 어떠한 파라미터 이름이 있는지 알 수 있고 그거에 따라서 매치를 하면 되긴 합니다만... 한 번 더 생각을 해야 하니 좀 불편하죠.

![](ghidra_part3/Untitled%204.png)

analysis option에서 WindowsPE x86 Propagate External Parameters 옵션을 체크한 후 Analyze를 진행하면..?

![](ghidra_part3/Untitled%205.png)

어떤 함수에서 사용되는 파라미터인지, 파라미터의 자료형과 이름까지 모두 코멘트로 작성됩니다!

혹시.. 옵션을 설정하는 걸 까먹고 Analyze를 진행을 하셨다면 굳이 껐다 킬 필요가 없습니다.

![](ghidra_part3/Untitled%206.png)

Analysis - One Shot - WindowsPE x86 Propagate External Parameters를 통해서 진행을 할 수 있습니다.

![](ghidra_part3/Untitled%207.png)

# flag constant 설정

악성코드 분석 등을 하다 보면 `CreateFileA()` 등의 API를 사용하는 경우가 많습니다. 그리고 이러한 API들은 flag 값에 `0x00000004`이 `FILE_SHARE_DELETE`라던지의 constant를 갖는 경우가 많이 있죠.

IDA에서는 해당하는 값을 우클릭하여 'Use standard symbolic constant'를 사용하여 값을 변경할 수 있습니다.

![](ghidra_part3/Untitled%208.png)

`CreateProcessA`의 `0x80000000` 플래그 값은 `CREATE_NO_WINDOW`의 값을 가지고 있습니다. 따라서 이를 선택하여 설정하면

![](ghidra_part3/Untitled%209.png)

이런 식으로 표시가 됩니다.

![](ghidra_part3/Untitled%2010.png)

ghidra에서 확인한 똑같은 부분입니다. `0x80000000` 부분을 똑같이 `CREATE_NO_WINODW`로 표시할 수 있는 방법은 똑같습니다. 해당 부분을 우클릭하고 'Set Equation(단축키 E)'를 누릅니다.

![](ghidra_part3/Untitled%2011.png)

![](ghidra_part3/Untitled%2012.png)

해당 창을 확인하시면 여러 가지 값들이 뜹니다.

![](ghidra_part3/Untitled%2013.png)

IDA에서 뜨는 창과 비교해 봤을 때, 큰 차이가 없죠? 이제 저기서 `CREATE_NO_WINDOW`를 선택하시면 됩니다.

![](ghidra_part3/Untitled%2014.png)

![](ghidra_part3/Untitled%2015.png)

`0x80000000`이 모두 `CREATE_NO_WINDOW`로 바뀐 것을 확인할 수 있습니다!

# 분량 조절 실패!

예상치 못하게 분량 조절을 실패해버렸습니다! unreachable code 등의 부분은 기본 설정으로 되어 있고... (분명 예전에는 기본 설정 아니었던 것 같은데 업데이트하면서 바뀌었나...? ㅠㅠ) IDAPython이나 그래프 뷰 등은 이전 파트에서 다루었죠.

사실 이 부분에서 WinAPI에 대한 것을 다룬 이유는 다음 파트를 위해서입니다... 급하게 마무리하는 것이 아닙니다. Part 4에서는 악성코드를 ghidra로 분석해보려 합니다!

와! 악성코드를 분석하려면 API 파라미터에 어떤 게 들어가는지 알아야 하고, flag 값도 hex로 나오는 것보다 constant 이름을 알려주면 너무나도 편리하잖아? 맞죠? 그렇죠? .... 다음에는 풍족한 분량의 글을 가져오도록 하겠습니다. 하하

![](ghidra_part3/Untitled%2016.png)

머리를 박으며 다음 파트를 준비해오도록 하겠습니다! 감사합니다!