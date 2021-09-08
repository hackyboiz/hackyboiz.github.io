---
title: "[Daily-Life] 부산까지 가서 발표한 썰 푼다, 시원포럼 발표 후기"
author: L0ch
tags: [L0ch, Fabu1ous, security one, daily , gookbab, review, ]
categories: [Daily-Life]
date: 2021-09-08 22:00:00
cc: true
index_img: 2021/09/08/l0ch/securityone_forum_review/Untitled 2.png
---



안녕하세요! L0ch입니다. 일상 글로는 오랜만에 돌아왔네요. → 맨날 연구에 찌들어 살았다는 뜻..

얼마 전 Fabu1ous와 함께 부산 벡스코에서 열리는 2021 사이버보안 컨퍼런스에서 발표를 하고 왔는데요, 간단한 후기를 남겨보려고 합니다 ㅎㅎ

# 갑자기 발표를 하라구요?

어느 평화로운 주말, 슬랙에 어떤 사진 하나가 올라오는데..

![Screen Shot 2021-09-07 at 17.13.31.png](securityone_forum_review/Screen_Shot_2021-09-07_at_17.13.31.png)

Security One에서 2021 사이버보안 컨퍼런스 때 진행할 시원포럼의 발표자를 모집한다는 내용의 캡처였습니다.

하지만 전 주말에 매우 바쁘기 때문에 슬랙을 보지 않았죠. 뭐하느라 바빴냐구요?

.

.

![Untitled](securityone_forum_review/Untitled.png)

> 아크라시아 여행하느라 바빴습니다!

당당하게 월요일에 출근해서 신청 마감일을 보니까 마감일까지 당장 3일이 남아서 호다닥 주제를 정하고 발표자료부터 만들기로 했습니다. 다행히 예전에 윈도우 패치를 디핑분석한 연구글인

[[Research] Windows Patch Diffing 맛보기 Part 1](https://hackyboiz.github.io/2020/11/15/l0ch/windows-patch-diffing-part1/) 

[[Research] Windows Patch Diffing 맛보기 Part 2](https://hackyboiz.github.io/2020/11/29/l0ch/windows-patch-diffing-part2/) 

이 두 개를 쓰면서 나중에 발표주제로 해도 좋을것 같다~ 라는 이야기를 한 적이 있어 주제는 금방 정했습니다.

중간에 내용도 조금 추가할 겸 Windows 프린터 스풀러 API 취약점인 CVE-2020-0986과 취약점 패치를 bypass하는 CVE-2021-1648을 patch diffing으로 직접 분석하는 내용도 넣으면 조금 여유롭게 준비하겠구나 생각했어요. 

![Screen Shot 2021-09-07 at 20.30.34.png](securityone_forum_review/Screen_Shot_2021-09-07_at_20.30.34.png)

어..여유롭게...?

![Screen Shot 2021-09-07 at 20.45.42.png](securityone_forum_review/Screen_Shot_2021-09-07_at_20.45.42.png)

43페이지.. 어라?

![Screen Shot 2021-09-07 at 19.35.45.png](securityone_forum_review/Screen_Shot_2021-09-07_at_19.35.45.png)

> 결국 여유롭게 3일 밤샘했습니다.

# 가자 부산으로!

~~벼락치기로~~ 열심히 준비한 덕에 다행히 발표자 선정 결과에 저와 Fabu1ous 둘 다 발표자로 선정이 되었더라구요! 얼른 김포공항에서 김해공항으로 가는 항공편을 예매했습니다. 발표가 1시부터였으니 적어도 9시 비행기는 타야 시간을 맞출 수 있었죠. 김포공항에서 15분 거리에 살긴 하지만 그래도 7시에 일어나는거 너무 힘듬 ㅠㅠ 

> 저 새x.. 아니 저 형이?  (Fabu1ous, 공항까지 한시간 반 거리)

아무튼! 김해공항으로 가는 비행기를 타고 부산으로 출발했습니다! (Photo by Fabu1ous)

![Untitled](securityone_forum_review/Untitled%201.png)

한시간 정도를 날아 김해공항에 도착했어요. 전역여행 이후로 두 번째로 오는 부산!.. 을 느낄 새도 없이 바로 벡스코로 향했습니다. ㅠㅠ 

![Screen Shot 2021-09-07 at 16.59.38.png](securityone_forum_review/Screen_Shot_2021-09-07_at_16.59.38.png)

> 나 사진 진짜 못찍네..? ~~PEXCO 아닙니다~~

벡스코에 도착하니 점심시간이라 배도 고팠고 시간도 좀 남아서 밥부터 먹었어요.

![Screen Shot 2021-09-07 at 20.54.22.png](securityone_forum_review/Screen_Shot_2021-09-07_at_20.54.22.png)

> 부산가면 뭐다? 돼지국밥이다! 여기 존맛이에요 ㄹㅇ 꼭가십쇼

![Screen Shot 2021-09-07 at 16.49.09.png](securityone_forum_review/Screen_Shot_2021-09-07_at_16.49.09.png)

> 나 이거 먹으려고 부산왔네

다대기 대신 쌈장을 넣어 주던데 부산 특색인가요? 아니면 이 가게만의 레시피인가요? 아무튼 엄청 맛있었습니다.

![Screen Shot 2021-09-07 at 16.48.47.png](securityone_forum_review/Screen_Shot_2021-09-07_at_16.48.47.png)

> 도착하자마자 화장실 급하다고 사라진 Fabu1ous 기다리면서

확실히 코로나라서 벡스코에서 하는 행사가 모두 비대면으로 진행되더라구요. 북적북적해야 재밌는데 아쉬웠습니다ㅠㅠ 다음에 또 발표할 기회가 있다면 그때는 상황이 좀 더 나아졌으면 좋겠네요.

준비기간이 조금 짧아서 아쉬운 점도 있었지만 다행히 발표는 큰 실수 없이 무사히 마칠 수 있었습니다. 다른 발표자분들이 발표하신 다른 좋은 주제들도 직접 볼 수 있어 좋은 경험이 되었네요. 

이번 포럼을 준비해주신 스탶분들, 그리고 저희 팀에 관심을 가져주신 2021 사이버 보안 컨퍼런스 참여자분들 모두 감사합니다!

![Untitled](securityone_forum_review/Untitled%202.png)

안녕!