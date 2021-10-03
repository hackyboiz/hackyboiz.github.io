---
title: "[Research] 퍼징 교양 수업 fuzz 101 - part2"
author: Fabu1ous
tags: [Fabu1ous, fuzzer, fuzzing, servey]
categories: [Research]
date: 2021-10-03 14:00:00
cc: true
index_img: /2021/10/03/fabu1ous/fuzz-2/1.gif
---



# 머릿말

![](fuzz-2/1.gif)

안녕하세요. Fabu1ous입니다.

정말 오랜만에 Part 2를 들고 돌아왔습니다. 저번 Part 1에서 퍼징 알고리즘의 가장 보편적인 모델에 대한 설명을 했습니다. 퍼징 알고리즘은 Preprocess와 Iteration으로 구성되어있으며 Iteration은 Schedule, InputGen, InputEval, ConfUpdate, Continue 총 5개 단계로 이루어져 있습니다. 이 중에서 Preprocess와 Schedule 단계를 알아봅시다.

Part 1을 안 보셨거나 너무 오래전에 읽어서 내용이 가물가물하신 분들은 Part 1을 쓱 훑어보고 오시길 바랍니다. Part 1에서 다양한 용어를 정의하기 때문에 위 GIF 이미지 같은 상황이 발생할 수 있습니다.



# 3. Preprocess

```
Preprocess(C) -> C
```

퍼저의 첫 iteration 수행 전에 초기 fuzz configuration 집합을 수정하기도 합니다. 이를 Preprocess, 전처리 단계라고 부르며 주로 아래와 같은 작업을 수행합니다.

1. PUT에 대한 instrumentation 적용
2. 쓸모없는 configuration 제거
3. seed trimming
4. driver app 생성

Preprocess는 InputGen에 대한 준비 단계의 의미도 지니고 있습니다.



## 3.1 Instrumentation

![](fuzz-2/2.png)

PUT 코드 사이사이에 instrumentation 코드를 삽입해 각 input에 대한 PUT의 실행 흐름이 어디까지 도달(code coverage)하는지 확인하는 것을 Instrumenation이라 부릅니다.

black-box, gray-box, white-box 등 퍼징 테스트의 색은 PUT의 내부 정보를 얼마나 많이 수집하는가에 따라 달라지는데, gray-box와 white-box는 PUT에 instrumentaion을 적용해 execution feedback을 수집하고 그에 맞춰 더 정교한 퍼징을 수행합니다.

PUT 내부에 대한 정보를 얻는 방법은 processor trace, system call usage 등 다양하지만 그중 instrumentation이 가장 쓸모 있는 데이터를 수집하는 경우가 많다고 합니다.

Program instrumentation은 static(정적)과 dynamic(동적) 방식으로 나뉩니다. Static 방식은 PUT 실행 전 Preprocess 단계에서 적용되고, dynamic 방식은 실제로 PUT가 실행되는 InputEval 단계에서 적용됩니다.

- Static Instrumentaion

  주로 컴파일 도중 소스코드나 intermediate code에 적용함

  런타임 전에 수행되니 당연히 런타임 오버해드가 크지 않음

  PUT이 라이브러리를 사용한다면 각각 라이브러리에 같은 instrumentaion 적용 필요(귀찮다!)

  소스 레벨 말고 바이너리 레벨에서 static instrumentation도 가능함

- Dynamic instrumentaion

  오버해드가 크지만 instrumentation을 PUT가 사용하는 DLL에 그때그때 적용할 수 있다는 것이 장점

  DynInst, DynamoRIO, Valgrid, QEMU 등 동적 instrumentation tool을 사용

![](fuzz-2/3.png)

꼭 static과 dynamic 방식 중 한 가지 instrumentaion만 사용해야 하는 것은 아닙니다. 예를 들면 AFL은 전용 컴파일러로 소스코드에 static instrumentation을 적용하고 QEMU로 바이너리 레벨 dynamic instrumentation도 적용합니다. 이는 직접 컴파일하지 못하는 external library에도 instrumentation을 적용해 좀 더 완벽한 코드 커버리지를 측정할 수 있게 해 줍니다.

### 3.1.1 Execution Feedback

![](fuzz-2/4.png)

gray-box 퍼저는 주로 execution feedback을 input으로 받아 test case를 발전시킵니다. 그 예로 AFL과 AFL로부터 파생된 퍼저들은 PUT의 branch instruction에 intstrumentation을 걸어 branch coverage 계산합니다. Branch coverage는 bit vector의 형태로 수집하기 때문에 path collision이 발생할 수 있어 이를 해결하기 위한 노력들이 존재합니다.

- CollAFL : path-sensitive hash function으로 이를 해결
- LibFuzzer, Syzkaller : node coverage를 execution feedback으로 사용
- Honggfuzz : execution feedback을 유저가 고를 수 있게 함



### 3.1.2 In-Memory Fuzzing

프로그램의 크기가 크면 오버해드도 크겠죠? 매번 프로세스를 re-spawning하지 않고 PUT의 일부만 퍼징 해 오버해드 최소화하는 것이 좋습니다.

특히 GUI 프로그램들 같은 경우 initializing이 정말 오래 걸리는데, 그래서 initialization 작업이 끝난 시점의 메모리 스냅샷을 찍어놓고 복구하는 방법을 사용합니다. 이런 기술을 in-memory fuzzing이라 부르고 클라이언트와 서버 간의 상호작용이 많은 네트워크 애플리케이션을 퍼징 할 때 자주 사용되는 방식입니다.

GRR은 input byte를 로드하기 전에 스냅샷을 찍어 매 iteration마다 불필요한 초기 실행 코드를 실행하지 않고 건너뜁니다.

AFL은 fork server를 사용해 몇 가지 스타트업 코드를 건너뛰는 방식을 선택했습니다. In-memory fuzzing과 비슷 하지만 스냅샷을 찍는 대신 iteration 마다 새로운 프로세스를 fork 합니다.

PUT의 상태를 원상복구 하지 않으면서 특정 함수에 대한 in-memory fuzzing을 수행하는 방법도 있습니다. In-memory API fuzzing이라고 하는데 AFL의 persistence mode가 이에 해당합니다. PUT의 상태를 원상 복구하지 않고 특정 함수를 루프를 통해 반복 실행합니다. Winnie 연구글에서 짧게 언급했듯이 이런 persistence mode는 메모리의 global state를 서서히 오염시켜 false positive 결과가 나올 수 있다는 문제가 있습니다. 그래서 in-memory API fuzzing을 할 땐 entry-point가 될 함수를 지정하는데 드는 노력이 상당합니다.



### 3.1.3 Thread Scheduling

Race condition 버그들은 non-deterministic behavior(비결정론적 행동: 같은 인풋을 받고도 다른 행동을 보임)에 의존하는 경우가 많아 트리거 하기 까다롭습니다. 하지만 instrumentation을 이용해 쓰래드 처리 순서를 정하면(Thread scheduling) 특정 비결정론적 행동을 트리거할 수 있습니다.



## 3.2 Seed selection

![](fuzz-2/5.png)

퍼저는 fuzz configuration 집합을 받아 fuzzing algorithm의 행동을 제어합니다. 문제는 fuzz configuration의 몇몇 파라미터들 value domain(정의역)이 너무 방대합니다. 예를 들면 mutation-based fuzzer의 seed가 있습니다. mp3파일을 시드로 퍼징 한다고 가정해보면 "정상적인 mp3파일"의 범위는 무한합니다. 세상에 존재하는 모든 노래와 앞으로 나올 노래 모두 정상적인 mp3파일의 범주안에 들기 때문에 이 중에서 어떤 시드를 사용해야 하는 걸까요?

초기 seed pool의 크기 축소 문제를 seed selection problem이라 합니다.

이 문제를 해결하는 다양한 접근법이 있지만 가장 흔히 사용되는 방법은 minset(최대 coverage metric을 갖는 최소 seed 집합)을 계산하는 것입니다. 적은 실행 시간 비용으로 많은 프로그램 로직을 테스트해야 버그 발견율을 증가시킬 수 있겠죠?

seed selection은 Preprocess 뿐만 아니라 ConfUPDATE 단계에서도 사용합니다.

다양한 coverage metic이 존재하는데, 먼저 AFL은 각 branch의 logarithmic counter(지수 카운터)를 이용한 branch coverage 기반 퍼저입니다. branch counter의 크기 정도(상용로그의 척도: 근사값의 크기 등급)가 달라야 다른 branch count로 취급하도록 구현되어있습니다.

HonggFuzz는 실행된 인스트럭션의 개수, 실행된 branch 개수, 실행된 unique basic block 개수를 사용해 coverage를 계산합니다.



## 3.3 Seed Trimming

크기가 작은 seed일수록 메모리 소모량이 적고, 높은 처리량(throughput)을 갖습니다. 그래서 몇몇 퍼저들은 퍼징 전에 seed의 크기를 줄이는 seed trimming 작업을 수행합니다. Seed trimming은 Preprocess나 ConfUPDATE 과정에서 수행됩니다.

AFL의 경우 seed가 동일한 coverage를 갖는 범위에서 서서히 seed의 일부를 삭제합니다.

Linux system call handler를 퍼징 할 때는 static 분석을 통해 알아낸 call dependency가 달라지지 않는 선에서 seed의 크기를 줄이기도 합니다.



## 3.4 Preparing a Driver Application

PUT을 직접 퍼징 하는 것이 어려운 경우가 많습니다. 예를 들면 PUT이 라이브러리일 때, 라이브러리 속 함수를 호출해주는 중간다리 역할의 프로그램이 필요합니다. 이를 driver 프로그램이라 부르는데, 흔히 driver라고 말할 때 떠올리는 device driver와는 다른 개념입니다. fuzz campaign 시작 전에 한 번만 만들면 되긴 하지만 상당히 번거로운 작업입니다. 제가 작성한 WinAFL 연구글에서 얼마나 귀찮은 작업인지 확인하 실 수 있습니다.

비슷하게 커널을 퍼징 하기 위해 유저 랜드 애플리케이션을 사용한다거나 IoT 장비를 퍼징 하기 위해 스마트폰 애플리케이션을 사용하는 등이 driver application의 예시입니다.

Harness 또한 Driver의 개념 안에 속해있다고 생각하시면 되겠습니다.



# 4. Scheduling

```
Schedule(C, t_elapsed, t_limit) -> conf
```

스케쥴링은 다음 iteration에 적용할 fuzz configuration을 정하는 것을 의미합니다. 당연히 퍼저의 종류에 따라 configuration의 내용이 달라집니다. 하나의 configuration만을 사용해서 스케쥴링 단계가 없는 퍼저도 있지만 BFF나 AFLFast처럼 잘 짜여진 스케쥴링으로 성공한 사례도 있기 때문에 주목해 볼 필요가 있습니다.

white-box의 경우 symbolic executor 등 복잡한 내용이 많아 생략하고 black-box와 grey-box 퍼저의 Scheduling 알고리즘만을 다룹니다.



## 4.1 Fuzz Configuration Scheduling (FCS) Problem

![](fuzz-2/6.png)

스케쥴링의 목적은 현재 이용 가능한 정보들을 통해 최선의 결과를 낼 수 있는 configuration을 정하는 것입니다. 예를 들면 unique 버그를 가장 많이 찾거나, 가장 넓은 코드 커버리지를 가질 것으로 예상되는 configuration을 선택하는 것입니다.

따라서 모든 스케쥴링 알고리즘은 Exploration(탐색)과 Exploitation(활용) 사이에서 갈등합니다.(Exploration vs Exploitation trade off)

- exploration : 미래를 위해 최대한 많은 정보를 수집하는 방향으로 갈 것이냐
- exploitation : 실직적인 성과를 내는 방향으로 갈 것이냐

그리고 퍼징에선 이런 문제를 Fuzz configuration Scheduling(FCS) 문제라고 부릅니다.



## 4.2 Black-box FCS Algorithms

블랙박스의 경우 FCS 알고리즘이 사용할 수 있는 정보는 이전 configuration의 퍼징 결과밖에 없습니다. 예를 들면 크래쉬와 버그의 수, 소모된 시간 등이 있겠죠. 이런 정보가 어떻게 사용될 수 있는지 처음 연구되고 적용된 게 BFF입니다. 해당 연구가 적용되고 BFF의 성능은 (실행당 버그수)가 85% 증가했고 이를 통해 잘 짜여진 FCS 알고리즘의 중요성이 주목받습니다.

곧바로 후속 연구들이 나왔는데, 아래와 같은 아이이어로 FCS 알고리즘을 개선했습니다.

1. 블랙박스 뮤테이션 퍼징의 수학적 모델을 기존에 사용하던 Bernoulli trial(베르누이 실행)에서 WCCP/UW(Weighted Coupon Collector's Problem with Unknown Weights)으로 바꿈

   ![](fuzz-2/7.png)

   - 베르누이 시행

     - 동전 던지기를 생각하면 됨
     - 성공, 실패 둘 중 하나의 결과만 존재
     - 각 configuration은 고정된 성공률이 있다고 가정

   - 쿠폰 수집 문제

     - 포켓몬 빵을 사면 255마리의 포켓몬중 하나의 스티커를 얻음.
     - 255마리 포켓몬 스티커를 모두 수집하기 위해 몇 개의 빵을 먹어야 하는가? 와 같은 질문에서 시작된 문제
     - 퍼징 하면서 지속적으로 성공률의 최대 범위(upper-bound)가 변함

     

2. 알고리즘이 Multi-Armed Banbit(MAB) 문제를 조사(investigate)함

   ![](fuzz-2/8.png)

   팔이 여러 개 달린 강도 문제(MAB)는 decison science에서 exploration vs exploitation 문제를 설명하는 방식입니다. 여러 개의 슬롯머신을 제한된 시간 동안만 플레이할 수 있고, 한 번에 한개의 슬롯머신만을 작동시킬 수 있으며(그럼 굳이 팔이 여러 개 달려있을 필요가 있나?) 각 슬롯머신마다 보상이 다릅니다.

   이때 강도가 취할 수 있는 선택은 다음과 같습니다.

   - 보상을 알고 있는 슬롯머신을 당긴다.
   - 다른 슬롯머신의 보상을 확인한다.

   강도가 제한된 시간 동안 최대 수익을 내기 위해 어떤 순서로 어떤 규칙으로 슬롯머신의 레버를 당겨야 하는가?

   

3. 다른 조건이 모두 동일할 때 퍼징 속도가 빠른 configuration이 unique bug를 더 많이 수집하고, 미래의 성공률에 대한 upper-bound를 줄이는데 좋다는 사실을 제시함



4. 한 번의 configuration selection 마다 고정된 iteration 횟수를 실행하는 대신 제한된 시간 동안 가능한 만큼 iteration을 실행하도록 BFF를 바꿔 느린 configuration에 시간 낭비를 방지함

위 연구를 통해 기존 BFF에 비해 동일한 시간 동안 약 1.5배 많은 unique bug를 찾을 수 있게 됨



## 4.3 Grey-box FCS Algorithms

Code coverage 등 Grey-box의 FCS 알고리즘은 black-box에 비해 사용할 정보의 선택 폭이 넓습니다. 진화 알고리즘(Evolution Algorithm)을 사용하는 AFL이 grey-box FCS 알고리즘의 선두주자입니다. 각 configuration마다 `fitness`라는 값을 갖고, 진화 알고리즘이 적절한 configuartion을 generic transformation(mutation, recombination)해서 자식을 만듭니다.

이렇게 만들어진 자식들 중 부모보다 fit 한 놈이 있을 거다라는 가정하에 진행됩니다.

진화 알고리즘의 개념으로 FCS를 이해하기 위해선 아래 질문들을 답해야 합니다.

1. configuration이 fit 한 기준이 뭔가
2. configuration을 어떻게 선택하나
3. 선택한 configuration으로 무얼 하나

high-level에서 예상해보면 빠르고 작은 input을 fit 하다고 생각할 수 있습니다. AFL은 configuration들을 원형 queue로 관리하며 다음으로 fit 한 configuration을 가져와서 고정된 횟수만큼 실행합니다.

빠른 configuration을 선호한다는 것은 black-box FCS와 동일한 것을 알 수 있습니다.

AFLFast는 아래 세 가지 아이디어를 통해 만들어졌습니다.

1. 선호하는 input (Favroite input)
   - 특정 control flow edge를 수행하는 configuration 중 AFLFast는 선택을 가장 적게 받은 configuration을 선호해 exploration 증가
   - 선택받은 횟수가 같은 configuration들이 있다면 가장 적게 실행된 path를 갖는 configuration 선호해 희귀 path의 탐색에 대한 exploration 증가
2. round-robin 선택 방식을 버리고 fit configuration의 우선순위를 사용해 다음 configuration을 선택
3. power schedule에 의해 정해진 횟수만큼 선택된 configuration을 실행
   - 처음엔 작은 값으로 시작해서 점점 빠르고 큰 폭으로 상승하는 `energy`값을 사용
   - path를 탐색하는 input의 수를 `energy` 값으로 정규화해서 적게 퍼징 된 configuration이 선택되도록 장려

24시간 퍼징 테스트를 해본 결과 AFLFast는 AFL이 찾지 못한 버그 3개를 찾았으며, AFL과 AFLFast 둘다 찾은 버그의 경우에는 AFLFast가 약 7배정도 더 빠르게 찾았습니다.



# 다음 파트 예고

![](fuzz-2/9.png)

이번 파트에선 퍼징 알고리즘의 여섯 단계 중 첫 두 단계인 Preprocess와 Scheduling를 알아봤습니다. 아직 갈 길이 한참 남았지만 여기서 잠시 쉬었다가 갑시다. 다음엔 Input Generation과 Input Evaluation을 알아오겠습니다.