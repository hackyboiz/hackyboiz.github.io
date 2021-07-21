---
title: "[Research] 핵린이의 angr 정복기 - (2) Symbolic Execution"
author: j0ker
tags: [j0ker, angr, symbolic_execution, exploit, newbie]
categories: [Research]
date: 2021-07-21 18:00:00
cc: true
index_img: /2021/07/21/j0ker/angr_part2/5gxdyj.jpg
---

# 핵린이의 angr 정복기 - (2) Symbolic Execution

![./angr_part2/5gxdyj.jpg](./angr_part2/5gxdyj.jpg)



# 서론

안녕하세요. j0ker 입니다... 저번주부터 날씨가 좋네요... 하지만 저는 집에만 쳐박혀서 일이랑 공부만 했습니다 하하 왜냐고요? 쓸데없이 주제를 어렵게 잡았기 때문이죠! 

![./angr_part2/joker_sad_by_darknight7_d1hjnno-fullview.jpeg](./angr_part2/joker_sad_by_darknight7_d1hjnno-fullview.jpeg)

> 쭈굴...

사실 주제는 잘못이 없습니다. 제가 못하기 때문에 이렇게 고통받는 것이죠. 처음에는 "음... 그냥 Symbolic Execution만 간단하게 공부해서 정리하면 되겠지?"라고 생각한게 가장 큰 오산이었습니다. 하다보니 다른 것도 공부해야하고 공부하다보니 모르는게 많고 궁금한 것도 많이 생기고... 이것저것 공부하다보니 생각보다 공부할 양이 많아서 시간이 오래걸렸습니다. 하하 닉네임을 고커(고통받는 조커)로 바꿔야겠네요.

아무튼 이제 Symbolic Execution에 대해 알아보도록 하겠습니다!



# 자동 소프트웨어 분석 기법

Symbolic Execution은 자동 소프트웨어 분석 기법 중 하나입니다. 타겟 소프트웨어을 지정해주면 자동으로 분석해서 취약점을 찾아주는거죠. 자동 소프트웨어 분석 기법은 동적 분석과 정적 분석 두 가지고 나뉩니다. 사람이 분석할 때도 정적 분석과 동적 분석으로 나뉘죠. 예를 들어, CTF에서 Pwnable 문제가 나오면 먼저 IDA에 넣어서 어셈블리 코드를 디컴파일 한 다음, 코드를 보면서 어떤 동작으로 하는지 분석합니다. 이게 타겟 프로그램을 실행하지 않고 분석하는 정적 분석 기법입니다. 그렇다면 반대로, 동적 분석은 프로그램을 실행해서 분석하는 기법이겠죠? IDA에서 분석을 완료하면 분석한 내용을 바탕으로 Exploit 시나리오를 세운 후, 프로그램을 실행하고 디버깅하면서 예상대로 메모리와 레지스터가 바뀌는지 확인한 다음 Exploit을 진행하겠죠. 이게 실제 프로그램을 실행하면서 분석하는 동적 분석 기법입니다.

여러분은 취약점 자동 분석하면 어떤 키워드들이 떠오르시나요? 많은 사람들이 아시거나 사용하는 기법들을 나열해보면 아래와 같습니다.

- Fuzzing : 타겟 소프트웨어에 랜덤한 값을 입력하여 프로그램의 버그가 발생하는지 확인하는 기법
- Taint Anaysis : 특정 값을 입력하고 입력한 값이 프로그램을 거치면서 레지스터나 메모리 등에 어떤 변화가 발생하는 지 추적하는 기법
- Dynamic Binary Instrumentation : 실행되고 있는 프로그램에 코드를 삽입하여 프로그램의 동작을 분석하는 기법

최근에는 이런 기법들이 단독으로 쓰이기 보다는 서로의 단점을 보완하면서 복합적으로 쓰이고 있는거 같습니다. 최근에 [Fabulous](https://hackyboiz.github.io/tags/Fabu1ous/)님이 포스팅한 [WinAFL로 마구 퍼징하기 시리즈](https://hackyboiz.github.io/2021/05/23/fabu1ous/winafl-1/)에서 다루고 있는 WinAFL도 Fuzzing+Dynamic Binary Instrumentation을 결합해 사용하고 있죠.

위에서 얘기한 기법들의 공통점은 무엇일까요? 모두 동적 분석 기법이라는 겁니다. 반면에 Symbolic Execution은 정적 분석 기법입니다.

![./angr_part2/5gy87e.jpg](./angr_part2/5gy87e.jpg)

아니... 이름에서부터 "Execution"이 들어가있고, 메모리랑 레지스터가 어떻게 바뀌는지 알아가면서 하는거 아냐? 라고 저는 생각했습니다... 그럼 이제 Symbolic Execution의 개념에 대해 알아보면서 왜 정적 분석인지도 보도록 하겠습니다.



# Symbolic Execution

Symbolic Execution에서는 먼저 타겟 소프트웨어에 대한 인풋값을 미지수로 가정하고 이를 심볼화(기호화)합니다. 그리고 모든 실행 경로를 탐색하여 모든 경로마다 이 경로까지 실행하기 위해서는 인풋값이 뭐가 되어야 하는지 계산할 수 있는 수식을 만듭니다. 버그가 존재하는 코드를 발견한다면 해당 실행 경로를 그대로 실행하기 위해서 이제까지 만든 수식을 풀어 최종적으로 어떤 인풋값을 넣어야하는지를 구합니다. 어렵죠... 이게 뭔 소린지...

예시를 보면서 다시 보도록 하겠습니다!

```c
void foobar(int a, int b) { 
	int x = 1, y = 0;
	if (a != 0) {
		y = 3+x;
		if (b == 0)
			x = 2*(a+b);
	}
	assert(x-y != 0); 
} 
```

예를 들어 foobar라는 함수를 분석한다고 가정해봅시다. 인풋은 a와 b가 있고 assert 함수를 통해 프로그램을 종료시키는 a와 b를 찾으려면 어떻게 해야할까요? 보면 x는 1로 y는 0으로 초기화를 하는데, x-y가 0이 되어야하니까 if 문 안에 들어가서 x와 y의 값을 조절해야할 거 같네요. if문을 보면 일단 a가 0이 아니여야 if문 안으로 들어갈 수 있을거 같구요. 그러면 y는 x+3=4가 됩니다. 그러면 x와 y가 같아야 하니까 두번째 if문 안으로 들어가서 x가 4가 될 수 있도록 해야겠네요. 그러기 위해서는 b가 0이어야 하고 2 * (a+b)가 4가 되어야합니다.

위에서 분석한 내용을 토대로 조건들을 나열해보면 다음과 같습니다.

1. $$a\ne0$$
2. $$0=2*(a+b)-4$$
3. $$b=0$$

그리고 이 조건들을 모두 만족해야하니까 합치면 다음과 같은 수식을 만들 수 있겠네요.

$$a\ne0 \wedge 0=2*(a+b)-4 \wedge b=0$$

이게 이걸 풀면 원하는 값을 얻을 수 있습니다! 근데 이거는 또 어떻게 풀어... 허허 안그래도 수학도 못하는데... 하지만... 

![./angr_part2/zzal.jpeg](./angr_part2/zzal.jpeg)

> 솔버쨩 도와줘!!!



## Solver

이왕 자동으로 분석하기로 했으니 이런 수식도 자동으로 분석해야겠죠! 저 같은 수포자도 이럴 때 사용할 수 있는 솔버라는게 있습니다.(역시 세상은 빠르게 발전하고 있습니다... 저만 시대에 맞춰 나가는게 이렇게 버거운 건지...) 솔버에는 SAT solver와 SMT solver가 있는데, 먼저 SAT solver에 대해 알아보도록 하죠.



### SAT solver

SAT는 SATisfiability problem의 앞 세글자를 따온 약자입니다. 한국어로는 충족 가능성 문제인데 이 문제를 풀어주는게 SAT solver입니다. 충족 가능성 문제는 논리 연산자와 논리 변수로 이루어진 논리식이 참이 되는 변수값이 존재하는지를 찾는 문제입니다.

SAT의 논리식은 **논리 변수**와 **논리 연산자**로만 이루어져있습니다. 이런걸 명제논리식이라고 합니다. 프로그래밍 언어에서 사용되는 논리연산자에는 not, and, or이 있죠. 곱셈이나 더하기 등 연산자 없이 이런 연산자들과 이 연산을 하기 위한 true, flase 같은 논리 변수들이 존재합니다. 예를 들면 아래와 같습니다.

$x_1 \wedge \neg x_2 \vee x_3$

여기서 $x_3$가 참이어서 논리식이 참이된다면, 이 논리식은 satisfiable하다라고 볼 수 있습니다. 반면에 식이 참이 되는 변수 조합을 찾을 수 없어 거짓이 된다면 이 논리식은 unsatisfiable하다고 할 수 있습니다. 이렇게 명제논리식으로만 이루어진 문제를 풀어주는 것이 SAT solver라고 합니다.



### SMT solver

SMT는 Satisfiability Modulo Theories의 약자로 SAT solver에서 해결하던 명제논리식뿐만 아니라 1차 논리식을 해결하기 위한 solver입니다. SAT solver에서 한단계 업그레이드된 놈이라 볼 수 있습니다. 1차 논리식에서는 논리변수와 논리연산자 뿐만 아니라 다양한 연산자와 변수를 사용할 수 있습니다. 

$(a * 10) > 20 \wedge (b + 20) < 100 \wedge (c > 0)$ 

이 논리식은 좀 전에 foobar 함수를 분석해서 나온 식과 좀 더 비슷하죠? 네. Symbolic Execution을 통해 만들어진 논리식을 바로 SMT solver에 전달되고 이런 논리식을 SAT solver가 논리식을 풀 수 있도록 명제논리식으로 만들어 줍니다. 그럼 또 SAT sovler에서 뚝딱뚝딱해서 결과값을 계산해주죠!

사실 SMT solver가 내부적으로 어떻게 동작하는지도 궁금해서 공부하고 정리하려고 했는데 내용이 너무 많고 이것저것 다 하다가는 시간 내에 다 공부를 못 할거 같아서 다음에 기회가 있으면 다뤄보도록 하겠습니다. 관심 있으신 분들은 Reference 6번을 참고해 보세요!



## 예제

이제 Symbolic Execution 엔진에서는 어떻게 코드를 분석하는지 알아보도록 하겠습니다. 하지만 일단 그 전에 Symbolic Execution 엔진에서 코드를 분석할 때 어떤 것들을 저장하는지 알아볼게요.

일단 코드를 분석할 때 총 세 가지 요소를 저장하고 이를 합쳐 state라고 부릅니다.

1. $stmt$ : statement의 약자로, 다음에 평가할 statement를 저장합니다. C언어를 분석한다고 하면 다음에 분석할 코드를 저장한다고 생각하시면 됩니다. statement에는 조건부 분기나 점프 혹은 수행해야할 assignment(이거는 동작이라고 이해하면 편할거 같네요)가 존재할 수 있습니다.
2. $\sigma$ : Symbolic Store로 프로그램의 변수를 심볼릭 값과 concrete 값(고정값)을 매핑한 내용들이 저장됩니다. 예를 들어, foobar 함수에서 a와 b는 각각 심볼릭 값인 $\alpha_a$와 $\alpha_b$로 매핑이 되고, x와 y는 1과 0으로 처음에 매핑됩니다. 그리고 이런 매핑관계(association)를 $\mapsto$ 기호로 표현합니다.(ex. $b \mapsto \alpha_b$)
3. $\pi$ : Path Constraint로 경로 제약 조건이 저장됩니다. 실행 경로를 탐색하면서 조건부 분기나 명령어들을 거쳐 $stmt$까지 실행되기 위해 앞에서 설정한 심볼릭 값 $a_i$가 어떤 값이 되어야 하는지에 대한 논리식이 저장되어 있습니다. 처음에는 true로 초기화됩니다.

그럼 이제 앞에서 설명했던 foobar 함수를 다시 가져와서 Symbolic Execution 엔진에서 어떻게 분석을 하는지 알아보겠습니다. 앞에서는 assert 함수를 통해 프로그램을 종료시킬 수 있는 값을 찾는데 집중했지만 Symbolic Execution은 원래 모든 경로를 탐색하기 때문에 assert 함수에서 프로그램이 종료가 되는 경로, 종료가 되지 않는 경로 모두 탐색해 보도록 하겠습니다.

```c
1.  void foobar(int a, int b) { 
2. 	int x = 1, y = 0;
3. 	if (a != 0) {
4. 		y = 3+x;
5. 		if (b == 0)
6. 			x = 2*(a+b);
7. 	}
8. 	assert(x-y != 0); 
9. }
```

먼저 함수 분석을 시작하면서 state를 초기화해줍니다. 일단 Symbolic Store에서 인자 a와 b를 각각 심볼릭 값인 $\alpha_a$와 $\alpha_b$로 매핑합니다.

$\sigma = \{ a \mapsto \alpha_a , b \mapsto \alpha_b \}$

앞에서 말했던 것처럼 경로 제약 조건은 기본적으로 true로 초기화됩니다.

$\pi = true$

$stmt$는 분석해야할 코드로 설정됩니다.

$stmt =$ `2. int x=1, y=0`

정리를 하자면 처음으로 초기화된 state A는 아래와 같습니다.

state A. $\sigma = \{ a \mapsto \alpha_a , b \mapsto \alpha_b \}$,  $\pi = true$, $stmt =$ `2. int x=1, y=0`

다음에 분석할 코드가 `int x=1, y=0`이기 때문에 새로운 변수 x,y가 추가됩니다. 그리고 각각 1과 0으로 초기화되기 때문에 Symbolic Store $\sigma$에 x와 y에 대한 정보가 추가되고 $stmt$에 별 다른 조건 분기나 심볼에 대한 조작이 존재하지 않기 때문에 경로 제약 조건 $\pi$은 업데이트되지 않습니다. $stmt$는 앞으로도 자동적으로 다음에 분석할 코드로 업데이트 됩니다.

state B. $\sigma = \{ a \mapsto \alpha_a , b \mapsto \alpha_b, x \mapsto 1, y \mapsto 0 \}$,  $\pi = true$, $stmt =$ `3. if(a != 0)`

이제 조건 분기문이 나왔습니다. 심볼릭 변수 $a$가 0이 아니라면 if문 안으로 들어가고 $a$가 0이라면 if문을 패스하고 넘어가겠네요. 이렇게 되면 Symbolic Execution 엔진에서는 fork를 통해 경로 제약 조건이 다른 두 가지 state를 생성합니다.

state C. $\sigma = \{ a \mapsto \alpha_a , b \mapsto \alpha_b, x \mapsto 1, y \mapsto 0 \}$,  $\pi = \alpha_a \neq 0$, $stmt =$ `4. y = 3+x`

state D. $\sigma = \{ a \mapsto \alpha_a , b \mapsto \alpha_b, x \mapsto 1, y \mapsto 0 \}$,  $\pi = \alpha_a = 0$, $stmt =$ `8. assert(x-y !=0)`

state가 생성이 되면 각각 별개로 다시 탐색에 들어갑니다. 먼저 state D를 보도록 하죠. 다음에 실행할 코드는 함수의 마지막인 assert 함수입니다. 조건으로 `x-y != 0`이 존재합니다. 그리고 현재 $x$는 1이고 $y$는 0이기 때문에 $x-y = 1-0 = 1 \neq 0$이 되어서 assert 함수로 인해 프로그램이 종료되지 않습니다. 그러면 이제 우리는 $\alpha_a = 0$이면,  즉 인자 $a$가 0이라면 프로그램이 정상적으로 실행될 수 있는 경로를 하나 알아낸 것이네요!

이번에는 state C를 보겠습니다. $stmt$가 `y = 3+x`이고 $x$가 1이기 때문에 Symbolic Store $\sigma$에 있는 $y$를 4로 업데이트 해줍니다.

state E.  $\sigma = \{ a \mapsto \alpha_a , b \mapsto \alpha_b, x \mapsto 1, y \mapsto 4 \}$,  $\pi = \alpha_a \neq 0$, $stmt =$ `3. if(b == 0)`

또 다시 조건 분기문이 나왔네요. if문 안에 있는 조건인 `b == 0`에 따라 경로 제약 조건 $\pi$를 업데이트하여 두 개의 state를 생성합니다.

state F. $\sigma = \{ a \mapsto \alpha_a , b \mapsto \alpha_b, x \mapsto 1, y \mapsto 4 \}$,  $\pi = (\alpha_a \neq 0) \wedge (\alpha_b \neq 0)$, $stmt =$ `8. assert(x-y != 0)`

state G. $\sigma = \{ a \mapsto \alpha_a , b \mapsto \alpha_b, x \mapsto 1, y \mapsto 4 \}$,  $\pi = (\alpha_a \neq 0) \wedge (\alpha_b = 0)$, $stmt =$ `6. x = 2*(a+b)`

먼저 state F를 보죠. Symbolic Store $\sigma$에 있는 $x$가 1이고 $y$가 4이기 때문에 $x-y=-3$이 됩니다. assert 함수의 조건이 $x-y \neq 0$이기 때문에 참이네요! 따라서 우리는 프로그램을 정상적으로 실행시킬 수 있는 두번째 조건 $\pi = (\alpha_a \neq 0) \wedge (\alpha_b = 0)$을 알아냈네요!(하.. 힘들다...)

다음으로는 state G 입니다. $stmt$가 `x = 2*(a+b)` 이네요. 이에 맞춰서 Symbolic Store $\sigma$에 있는 $x$를 $x \mapsto 2*(\alpha_a + \alpha_b)$로 업데이트 해줍니다.

state H. $\sigma = \{ a \mapsto \alpha_a , b \mapsto \alpha_b, x \mapsto 2*(\alpha_a+\alpha_b), y \mapsto 4 \}$,  $\pi = (\alpha_a \neq 0) \wedge (\alpha_b = 0)$, $stmt =$ `8. assert(x-y != 0)`

이제 드디어 마지막 state 입니다! assert 함수를 프로그램 종료를 시키지 않고 넘어가기 위해서는 `x-y != 0`가 참이 되어야 합니다. 이에 맞게 Symbolic Store $\sigma$에 있는 $x$와 $y$를 참고하여 새로운 경로 제약 조건을 추가해주면 됩니다. $x$는 $2*(\alpha_a + \alpha_b)$이고 $y$는 4이기 때문에 이를 반영하면 최종적으로 아래와 같은 경로 제약 조건 $\pi$가 나오게 됩니다!

$\pi = (\alpha_a \neq 0) \wedge (\alpha_b = 0) \wedge 2*(\alpha_a + \alpha_b) + 4 \neq 0$

이 논리식을 풀면 프로그램을 정상적으로 실행할 수 있는 a와 b 값을 얻을 수 있겠네요!(나머지는 SMT solver가 알아서 해줄 겁니다 ㅋㅋ)



## Symbolic Execution의 한계

여기까지만 보시면 "우와 Symbolic Execution은 우리가 분석하기 귀찮아하는 것도 알아서 다 분석해서 정말 자동으로 취약점을 찾을 수 있겠네?"라고 생각하실 수 있겠지만... 이론적으로는 가능하나 현실적인 제약들이 많이 존재합니다 ㅠㅠ (이게 가능했으면 저는 벌써 길바닥에 앉아 구걸을 하고 있을지도...)

1. 메모리의 제약이 존재합니다. 

   예시로든 foobar 함수는 매우 짧아서 분석이 가능하지만 우리가 현실에서 사용하는 프로그램들은 크기가 매우 큽니다. 일반적인 오피스나 플레이어 같은 프로그램들은 작게는 몇 백 MB 혹은 몇 십 GB까지 합니다. 이런 프로그램들을 분석하기 위해 state를 계속해서 쌓아나간다고 생각해보세요. 최소 프로그램 크기의 몇 배에서 몇 백배의 메모리가 필요할 겁니다.(뇌피셜) 이 문제는 Symbolic Execution이 제안된 70년대에도 똑같이 지적됐었던 문제입니다. 그 당시에는 지금만큼 복잡한 프로그램이 없었을텐데도 말이죠. 물론 지금 메모리 값이 싸지고 클라우드 플랫폼이 생기면서 좀 더 큰 메모리를 확보하는게 쉬워졌지만 아직까지도 이는 Symbolic Execution의 대표적인 문제점으로 지목 받고 있습니다. 물론 마이크로소프트나 구글 같은 대기업에서는 몇 만 코어에 몇 백 테라바이트로 돌리고 있을지도 모르겠지만... 저 같은 개인이 그냥 Symbolic Execution에만 의존해서 취약점을 찾는 엔진을 만들었다고 해도... 돌아가도 효율이 안 좋겠죠... 아마 그냥 Dumb Fuzzing이 더 빠를 수 있습니다 ㅠㅠ

2. 컴퓨팅 파워가 부족합니다.

   메모리 문제가 해결이 됐다고 쳐도 분석해야할 실행 경로가 너무 많습니다. 상용 프로그램에서는 그냥 분석해야 할 코드도 이미 많은데, 이 안에서 조건분기문, 루프문도 많이 존재하고 이에 따라 같은 취약점이더라도 다른 실행 경로를 통해 트리거될 수도 있습니다. 모든 실행 경로를 분석해야 취약점을 제일 exploit하기 좋은 실행 경로를 찾을 수 있는 것이죠. 심지어 특정 실행 경로에서는 취약점을 트리거할 수 있어도 exploit하지는 못할 수도 있습니다. 근데 이런 실행 경로를 fork까지 해가면서 모든 state를 다 저장하고 계산하고... 일단 CPU의 코어 개수도 많아야할 것이고 심지어 빨라야할 것입니다. 하지만... 우리에게는 리사수느님이 있다!!! 킹갓스레드리퍼로 돌리면 되지 않겠느냐!!! 예... 누가 함 구해주신다면 어떻게든 온몸을 바쳐 돌려보도록 하겠습니다 ㅎ허헣ㅎ(생각만 해도 좋네)

   ![./angr_part2/22HD9O4TZ2_4.jpeg](./angr_part2/22HD9O4TZ2_4.jpeg)

   > 이쯤에서 영접해보는 리사수느님

3. 실행 환경과의 interaction은 분석할 수 없습니다.

   프로그램을 실행하면 기본적으로 운영체제 위에 올라가기 때문에 운영체제와 상호작용을 해야합니다. 힙을 할당해야할 수도 있고 시스템 콜을 보내야할 수 있습니다. 이렇게 요청을 한 것들은 커널에서 알아서 해줍니다. 하지만 Symbolic Execution 엔진에서는 커널에서 동작하는 내용을 분석하지는 않습니다. 아 커널도 분석해버리면 안되냐구요? 하하 그렇게 하지 않아도 이미 분석할 거리가 많은데... 그렇다면 네트워크 통신은 어떨까요? 이것도 분석할 수 있을까요? 당연히 안되겠죠. 이렇게 프로그램 외적인 대상에 대한 분석이 빠져있기 때문에 분석을 완벽하게 진행할 수 없습니다.

   

## 한계점 보완

그렇다면 Symbolic Execution은 리얼월드에서는 쓸모 없는 것 일까요? 그렇다면 이렇게 얘기하고 있지 않겠죠 ㅋㅋ 맨 앞에 동적 분석 기법들이 서로의 단점을 보완하면서 활용되고 있는 것처럼 Symbolic Execution 역시 단점을 보완해줄 수 있는 기법들과 같이 사용되고 있습니다.

### Concolic Execution

대표적인게 바로 Concolic Execution입니다. Concolic은 Concrete + Symbolic을 합친 합성어로 Concrete Execution과 Symbolic Execuition을 동시에 실행하는 겁니다. Concrete Execution은 Symbolic Execution과 반대로 미지수가 아닌 구체적인 값을 인풋으로 주고 실행 과정을 분석하는 기법입니다. 따라서 이 두 기법을 합쳐서 사용하게 된다면 기존 Symbolic Store $\sigma$에 Concrete 값이 추가됩니다. Concrete 값을 넣어보면서 특정 분기로 빠지는 값(Statisfiable한 값)을 미리 알아내게 된다면 SMT solver의 계산 없이 해당 실행 경로에 좀 더 빨리 진입할 수 있는 장점을 가지고 있습니다.

이 외에도 선별적인 실행 경로 분석이라던가 fuzzing과 같이 활용한다던가 여러 활용 기법들이 있지만 아직은 더 공부해야할 부분들이기 때문에 나중에 또 다뤄보도록 하겠습니다!



# Symbolic Execution 과정에서 프로그램은 실행되지 않는다

처음에 제가 가지고 있는 의문점을 이제는 풀 수 있겠네요. Symbolic Execution 과정에서 프로그램을 실행이 되지 않고 코드만 분석을 하는 것이기 때문에 동적 분석이라고 볼 수 없습니다. 프로그램이 실행되면 로더에서 프로그램을 메모리에 매핑하고, 프로세스 ID를 할당하여 프로세스를 생성하고, CPU에 명령어를 전달하여 기계어를 실행합니다. 하지만 Symbolic Execution 엔진에서는 이런 일련의 과정없이 엔진에 프로그램을 로드하고 코드를 엔진이 분석하기 쉬운 IL(Intermediate language)로 바꾸어 위에서 얘기한 일련의 작업들을 진행합니다. CPU가 실재로 기계어를 실행하는 것도 아니고 프로그램이 메모리에 매핑되지도 않습니다. IDA가 프로그램을 로드하는 것과 같다고 생각하면 되겠네요. 



# Part 2 마무리

사실 Symbolic Execution을 공부해야겠다라고 처음 생각했을 때, 대략적으로 뭔지는 알아서 정리하는게 뭐 그렇게 어렵겠어라고 생각했습니다. 하하 역시 무식하면 용감하다고 무작정 뛰어들었다가 뒤지게 쳐맞았네요. 공부하면서 느낀건데 정말 공부할 내용이 많은거 같습니다. 마무리하는 와중에도 이거 좀 더 공부할걸 저거 좀 더 공부할걸 하는 아쉬움이 좀 남아있습니다. 하지만 시간은 제한되어 있고... 글은 올려야하고... 잠을 좀 더 줄여서 공부 좀 더 하고... 해야겠습니다...

그럼 이제 다음 편을 쓰기 위해 또 공부하러 가야겠네요... 근데 나 다음 글 올릴 수 있겠지...?

![./angr_part2/5h1k8f.jpg](./angr_part2/5h1k8f.jpg)



# Reference

1. [A Survey of Symbolic Execution Techniques](https://arxiv.org/pdf/1610.00502.pdf)
2. [[HACKING] Symbolic Execution(symbolic evaluation)을 이용한 취약점 분석](https://www.hahwul.com/2017/06/19/hacking-symbolic-executionsymbolic/)
3. [2017 CodeEngn Conference 14, 솔버가 너희를 자유롭게 하리라(feat. Symbolic Execution) [강정진]](https://github.com/codeengn/codeengn-conference/blob/master/14/2017%20CodeEngn%20Conference%2014%2C%20%EC%86%94%EB%B2%84%EA%B0%80%20%EB%84%88%ED%9D%AC%EB%A5%BC%20%EC%9E%90%EC%9C%A0%EB%A1%AD%EA%B2%8C%20%ED%95%98%EB%A6%AC%EB%9D%BC(feat.%20Symbolic%20Execution)%20%5B%EA%B0%95%EC%A0%95%EC%A7%84%5D.pdf)
4. [Awesome Symbolic Execution](https://github.com/pwnbatman-joker/awesome-symbolic-execution)
5. [03.Symbolic execution(feat.Concrete execution)](http://www.lazenca.net/pages/viewpage.action?pageId=6324534)
6. [SAT/SMT tutorial SMT 해결기 내부](http://rosaec.snu.ac.kr/meet/file/20110626o2.pdf)
