---
title: "[Research] 핵린이의 angr 정복기 - (1) 시작"
author: j0ker
tags: [j0ker, angr, symbolic_execution, exploit, newbie]
categories: [Research]
date: 2021-07-10 18:00:00
cc: true
index_img: /2021/07/10/j0ker/angr_part1/2e17a20210794aa3b3831a38fad04184.jpeg
---

# 인사말

------

안녕하세요! j0ker입니다. 아직 내공이 많이 부족해 그냥 공부만 하고 있다가 이제야 공부하고 있는 내용을 포스팅해볼까 합니다!! 내가 왔다아아아아아악!!!!

![두두등장](./angr_part1/1625546988.gif "우아씨 떨어질 뻔했네;;")

예전에 해킹하기 전에 영화 같은거 보면 해커들이 나와서 키보드를 촤라라라락 누르더니 지도에 점이 찍히고 취약점 알아서 찾고 공격까지 하는 장면을 본 적이 있습니다 ㅋㅋㅋㅋ 지금 보면 말이 안되는데 말이지... 그래도 그 때는 두근두근하면서 "오 해커는 정말 멋지다"라는 생각을 했었습니다. 그리고 이렇게 잘못된 길로 들어와 버렸죠... 허허 그 때 멈췄어야 했습니다.

![2](./angr_part1/2e17a20210794aa3b3831a38fad04184.jpeg "얘도 그 때 '그 반지'만 받지 않았더라면...")

암튼 제가 봤던 그 영화 속에 해커에 좀 더 다가갈 수 있도록 해줄 수 있는 툴이 바로 angr이지 않을까 싶습니다. 그래서 angr를 공부하려고 하니까 다 라이트업만 있지 처음부터 끝까지 다루는 내용은 별로 없더라구요.(있으면 댓글 남겨주시면 감사하겠습니다! 내가 못 찾는건가...?) 시리즈를 일단 4편까지 쓸까 고민하고 있는데 내용을 다루면서 개념적인 부분이나 잘 다루지 않는 angr의 기초적인 내용에 대해서 또 다루다보면 길어질 수도 있겠습니다 ㅎㅎ

사실 이미 많이 유명한 툴이라 설명이 더 필요할까 싶긴한데 오늘 준비한 내용은 많지 않으니 주저리주저리 설명을 하고 가도록 하겠습니다 ^^

# angr란?

------

angr는 다양한 환경에서 활용 가능한 바이너리 분석 플랫폼 입니다. angr가 유명해지기 시작한 것은 2015년 DARPA의 CGC(Cyber Grand Challenge)가 시작하면서죠. CGC는 참여팀이 바이너리를 자동 분석하는 툴을 만들어 제한 시간 내에 출제된 문제들에 취약점이 있음을 증명하고 패치하는 점수를 얻는 대회 입니다. 일반적인 CTF와는 다르죠? angr를 제작한 Shellphish팀이 이 대회에 참가하였고 3등을 차지했습니다. 대회가 끝나고 자동 취약점 분석 분야가 해커들에게 많은 주목을 받으면서 angr라는 툴 역시 python을 기반으로 한 높은 접근성 때문에 많은 인기를 끌기 시작했습니다. 다른 팀들이 제작한 바이너리 자동 분석 툴들도 공개된 것들이 있지만 angr가 python으로 활용 가능하다는 점과 문서화 잘 되어 있다는 점이 확실히 매력적으로 다가오는 거 같습니다. 그러니까 저 같은 핵린이들도 바이너리 자동분석을 찍먹해볼 수 있는거겠죠. CGC 이후로 많은 CTF에서 자동 취약점 분석 및 익스플로잇 혹은 리버싱 분야에서 angr를 활용할 수 있는 문제들이 나오고 해커들이 angr를 익숙하게 다룰 수 있게 되면서 보편적으로 많이 쓰이게 된 거 같습니다.(카더라)

angr, CGC, shellphish에 대해서 좀 더 자세하게 알고 싶으시다면 아래 링크를 참고해보시면 좋을듯 합니다.

- [[번역\] 2016 DARPA CGC 대회 출전팀의 회고](https://cpuu.postype.com/post/4127178)
- [The Cyber Grand Challenge](https://shellphish.net/cgc/)
- [Cyber Grand Challenge 후기](http://kaishackgon.blogspot.com/2016/08/cyber-grand-challenge.html)
- [美 방위고등연구계획국(DARPA)의 사이버 자동 방어시스템 개발을 위한 컨테스트(CGC) 개최](https://www.krcert.or.kr/data/trendView.do?bulletin_writing_sequence=20050)

# angr 설치

------

위에서 말씀드렸다시피 angr는 다양한 환경에서 활용 가능합니다. 이게 뭔 말이냐? python만 깔리면 다 쓸 수 있다는 겁니다! 다 쓸 수 있다고는 하지만 저희는 일단 우분투에서 angr를 사용하도록 하겠습니다.

먼저 아래 명령어를 실행해서 실행에 필요한 기본적인 요소들을 설치합니다.

```bash
sudo apt-get install python3-dev libffi-dev build-essential virtualenvwrapper
```

그리고 다른 모듈과의 버전 충돌을 방지하기 위해 virtualenv를 만들고 그 안에 angr를 설치해줍니다.

```bash
mkvirtualenv --python=$(which python3) angr && pip install angr
```

만약 설치가 잘 안된다!!! 그러면 정직하게 [오피셜 문서](https://docs.angr.io/introductory-errata/install)를 참고해주세요.

# angr 톺아보기

------

이제 angr를 어떻게 사용하는지 알아봐야겠죠? 오늘은 정말 그냥 이런게 있구나~ 정도만 보고 다음에 좀 더 자세하게 앞으로 다뤄볼 예시 문제 상황에 따라 알아보도록 하겠습니다.

## Project

일단 아래와 같이 기본적인 프로젝트를 생성합니다.

```python
>>> import angr
>>> proj = angr.Project('00_angr_find')
```

이렇게 하면 타겟 바이너리에 대한 Project 오브젝트가 생성됩니다. 이 프로젝트가 angr에서 해당 바이너리에 대한 정보를 제어하는 기본 단위가 됩니다.

먼저 바이너리에 대한 기본적인 정보들을 확인할 수 있습니다.

```python
>>> proj.arch
<Arch X86 (LE)>
>>> proj.entry
134513744
>>> import monkeyhex
>>> proj.entry
0x8048450
```

중간에 monkeyhex를 import하면 그냥 10진수로 출력되는 값들이 16진수로 출력되서 좀 더 편하게 확인할 수 있습니다.

당연히 이 뿐만 아니라 다른 정보들도 있겠죠! 함 보겠습니다.

```python
>>> proj.__dict__
{'filename': '00_angr_find',
 'loader': <Loaded 00_angr_find, maps [0x8048000:0x8407fff]>,
 'arch': <Arch X86 (LE)>,
 '_sim_procedures': {0x8100000: <SimProcedure strcmp>,
  0x8100004: <SimProcedure printf>,
  0x8100008: <SimProcedure __stack_chk_fail>,
  0x810000c: <SimProcedure puts>,
  0x8100010: <SimProcedure exit>,
  0x8100014: <SimProcedure __libc_start_main>,
  0x8100018: <SimProcedure __isoc99_scanf>,
  0x10301000: <SimProcedure LinuxLoader>,
  0x10301008: <SimProcedure _dl_rtld_lock_recursive>,
  0x10301010: <SimProcedure _dl_rtld_unlock_recursive>,
  0x10301018: <SimProcedure ReturnUnconstrained>,
  0x10301020: <SimProcedure _dl_initial_error_catch_tsd>,
  0x10301028: <SimProcedure _vsyscall>,
  0x10301038: <SimProcedure CallReturn>,
  0x10301040: <SimProcedure UnresolvableJumpTarget>,
  0x10301048: <SimProcedure UnresolvableCallTarget>},
 'concrete_target': None,
 '_default_analysis_mode': 'symbolic',
 '_exclude_sim_procedures_func': None,
 '_exclude_sim_procedures_list': (),
 'use_sim_procedures': True,
 '_ignore_functions': [],
 '_support_selfmodifying_code': False,
 '_translation_cache': True,
 '_executing': False,
 'entry': 0x8048450,
 'storage': {},
 'store_function': <bound method Project._store of <Project \\\\Mac\\Home\\Desktop\\Angr_Tutorial_For_CTF\\problems\\00_angr_find>>,
 'load_function': <function Project._load at 0x00000225ECEF2CA0>,
 'factory': <angr.factory.AngrObjectFactory object at 0x00000225ECEF4100>,
 '_analyses_preset': None,
 'analyses': <angr.analyses.analysis.AnalysesHub object at 0x00000225ED698A00>,
 'kb': <angr.knowledge_base.knowledge_base.KnowledgeBase object at 0x00000225ED698C70>,
 'simos': <angr.simos.linux.SimLinux object at 0x00000225ED698A90>,
 'is_java_project': False,
 'is_java_jni_project': False}
```

아... 모르는게 많네요... 하하... 저걸 다 공부해야 한다고??

![3](./angr_part1/1625553299.gif "야레야레...")

하나씩 알아가보도록 하죠... 흐흐...

## loader

```python
>>> proj.loader
<Loaded 00_angr_find, maps [0x8048000:0x8407fff]>
>>> proj.loader.shared_objects
{'00_angr_find': <ELF Object 00_angr_find, maps [0x8048000:0x804a03f]>, 
'extern-address space': <ExternObject Object cle##externs, maps [0x8200000:0x8207fff]>, 
'cle##tls': <ELFTLSObjectV2 Object cle##tls, maps [0x8300000:0x8314807]>}
>>> proj.loader.min_addr
0x8048000
>>> proj.loader.max_addr
0x8407fff
>>> proj.loader.main_object
<ELF Object 00_angr_find, maps [0x8048000:0x804a03f]>
>>> proj.loader.main_object.execstack
False
>>> proj.loader.main_object.pic
False
```

loader에서는 바이너리가 메모리에 어떻게 매핑되는지에 대한 정보를 가지고 있습니다. angr에서는 CLE라는 모듈을 통해 바이너리를 메모리 주소에 매핑한 다음, loader에 메모리 주소 정보들을 저장한다고 합니다. 지금은 그저 "loader를 통해 바이너리가 메모리에 매핑된 주소나 미티게이션 적용 여부를 알 수 있다" 정도만 알면 될듯 합니다!

## factory

angr에서는 지원하는 기능이 어~~~~~~~~ㅁ청 많기 때문에 코드들이 난잡해지는 것을 막기 위해서 자주 사용하는 기능들을 factory라는 클래스에 넣어놨습니다.

### block

앞에서는 메모리에 바이너리가 매핑되었는지를 알았으니 이제 해당 주소에 있는 내용을 알아볼겁니다. black 오브젝트는 특정 주소의 명령어 블럭을 확인할 수 있습니다.

```python
>>> block = proj.factory.block(proj.entry)
>>> block.pp()
0x8048450:      xor     ebp, ebp
0x8048452:      pop     esi
0x8048453:      mov     ecx, esp
0x8048455:      and     esp, 0xfffffff0
0x8048458:      push    eax
0x8048459:      push    esp
0x804845a:      push    edx
0x804845b:      push    0x8048710
0x8048460:      push    0x80486b0
0x8048465:      push    ecx
0x8048466:      push    esi
0x8048467:      push    0x80485c7
0x804846c:      call    0x8048420
>>> block.instructions
0xd
>>> block.instruction_addrs
[0x8048450,
 0x8048452,
 0x8048453,
 0x8048455,
 0x8048458,
 0x8048459,
 0x804845a,
 0x804845b,
 0x8048460,
 0x8048465,
 0x8048466,
 0x8048467,
 0x804846c]
```

이처럼 명령어를 보고 싶은 주소를 인자를 주면 원하는 코드 블럭에 있는 명령어들을 확인할 수 있습니다. 명령어의 데이터 형식을 볼 수도 있고 명령어 각각의 주소도 확인할 수 있습니다.

### state

Project 오브젝트는 프로그램 초기상태의 정적인 정보를 담고 있습니다. 그냥 정적인 데이터만 가지고는 프로그램이 실행하면서 값이 어떻게 변하는지를 확인할 수는 없겠죠? 그래서 angr에서는 프로그램이 실행되는 것처럼 시뮬레이션을 할 수 있고 이 과정을 SimState라는 오브젝트를 통해 확인하거나 조작할 수 있습니다.

```python
>>> state = proj.factory.entry_state()
>>> state.regs.eip
<BV32 0x8048450>
>>> state.regs.eax
<BV32 0x1c>
>>> state.mem[proj.entry].int.resolved
<BV32 0x895eed31>
```

지금은 엔트리 포인트에서의 레지스터와 메모리 상태를 알 수 있습니다. 근데 레지스터 값을 확인하면 그냥 인티저 값이 아닌거 같네요?

angr에서는 메모리나 레지스터의 값을 bitvector 형식으로 저장한다고 합니다. 일반적으로 32비트나 64비트 레지스터는 최대값이 넘어가면 0으로 되면서 오버플로우가 발생하잖아요? 근데 아시겠지만 파이썬의 인티저는 그런게 없죠! 오버플로우 따위는 웬만하면 일어나지 않습니다. 그렇기 때문에 32비트와 64비트 bitvector를 활용하게 됩니다.

```python
>>> bv = state.solver.BVV(0x1234, 32)
<BV32 0x1234>
>>> state.solver.eval(bv)
0x1234
>>> state.regs.rsi = state.solver.BVV(3, 64)
>>> state.regs.rsi
<BV64 0x3>
>>> state.mem[0x1000].long = 4
>>> state.mem[0x1000].long.resolved
<BV64 0x4>
```

이런식으로 bitvector↔ integer를 왔다갔다 타입변환을 할 수도 있습니다. 그리고 타입을 지정해서 값을 저장할 수도 있죠.

### Simulation_manager

state를 통해 어디서 시뮬레이션을 할지를 지정했다면 Simulation_manager를 통해서는 지정한 state를 기준으로 실행하면서 어떻게 데이터들이 바뀌어 나가는지 확인해 볼 수 있습니다.

```python
>>> simgr = proj.factory.simulation_manager(state)
>>> simgr.active
[<SimState @ 0x8048450>]
>>> simgr.step()
<SimulationManager with 1 active>
>>> simgr.active[0].regs.eip
<BV32 0x8048420>
>>> simgr.step()
<SimulationManager with 1 active>
>>> simgr.active[0].regs.eip
<BV32 0x8100014>
>>> state.regs.eip
<BV32 0x8048450>
```

이렇게 Simulation Manager에서 step() 함수를 사용하면 eip가 바뀌는 것을 볼 수 있습니다. 호호 factory는 오늘은 이정도만 알아볼텐데... 나중에 많이 만날거 같은 불안한 느낌... 하하...(스포)

## Analyses

angr에서는 사용자가 하나부터 열까지 다 분석하고 삽질하며 시간 낭비를 하지 않도록 수많은 분석 툴들을 제공합니다. 그 툴들이 내장되어 있는 클래스가 바로 Analyses 입니다! 아래와 같이 정말 많은 기능이 있는데, 각각의 기능은 추후에 직접 써보면서 소개해드리도록 하겠습니다. 그래도 다 소개하는 무리데스...

```python
>>> proj.analyses.
proj.analyses.AILBlockSimplifier(              proj.analyses.Decompiler(                      proj.analyses.StructuredCodeGenerator(
proj.analyses.AILCallSiteMaker(                proj.analyses.Disassembly(                     proj.analyses.Structurer(
proj.analyses.AILSimplifier(                   proj.analyses.DivSimplifier(                   proj.analyses.Typehoon(
proj.analyses.BackwardSlice(                   proj.analyses.DominanceFrontier(               proj.analyses.VFG(
proj.analyses.BasePointerSaveSimplifier(       proj.analyses.EagerReturnsSimplifier(          proj.analyses.VSA_DDG(
proj.analyses.BinDiff(                         proj.analyses.Identifier(                      proj.analyses.VariableRecovery(
proj.analyses.BinaryOptimizer(                 proj.analyses.ImportSourceCode(                proj.analyses.VariableRecoveryFast(
proj.analyses.BoyScout(                        proj.analyses.InitFinder(                      proj.analyses.Veritesting(
proj.analyses.CDG(                             proj.analyses.InitializationFinder(            proj.analyses.XRefs(
proj.analyses.CFB(                             proj.analyses.LoopFinder(                      proj.analyses.discard_plugin_preset(
proj.analyses.CFBlanket(                       proj.analyses.ModSimplifier(                   proj.analyses.get_plugin(
proj.analyses.CFG(                             proj.analyses.MultiSimplifier(                 proj.analyses.has_plugin(
proj.analyses.CFGEmulated(                     proj.analyses.Propagator(                      proj.analyses.has_plugin_preset
proj.analyses.CFGFast(                         proj.analyses.Proximity(                       proj.analyses.plugin_preset
proj.analyses.CFGFastSoot(                     proj.analyses.ReachingDefinitions(             proj.analyses.project
proj.analyses.CalleeCleanupFinder(             proj.analyses.Reassembler(                     proj.analyses.register_default(
proj.analyses.CallingConvention(               proj.analyses.RecursiveStructurer(             proj.analyses.register_plugin(
proj.analyses.Clinic(                          proj.analyses.RegionIdentifier(                proj.analyses.register_preset(
proj.analyses.CodeTagging(                     proj.analyses.RegionSimplifier(                proj.analyses.release_plugin(
proj.analyses.CompleteCallingConventions(      proj.analyses.SootClassHierarchy(              proj.analyses.reload_analyses(
proj.analyses.CongruencyCheck(                 proj.analyses.StackCanarySimplifier(           proj.analyses.use_plugin_preset(
proj.analyses.ConstantDereferencesSimplifier(  proj.analyses.StackPointerTracker(
proj.analyses.DDG(                             proj.analyses.StaticHooker(
```

# 00_angr_find

그냥 설명만 하다 끝내기는 아쉬우니까 간단하게 튜토리얼 문제 하나만 보고 마무리할게요. 이 [링크](https://github.com/jakespringer/angr_ctf)로 들어가셔서 레포를 클론해줍니다. 이 레포는 angr의 기능을 ctf 문제 스타일로 하나하나 공부하기에 적합해 보여서 가져와 봤어요. 00_angr_find는 그 첫번째 문제입니다!

```python
${
import random, os
random.seed(os.urandom(8))
userdef_charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'
userdef = ''.join(random.choice(userdef_charset) for _ in range(8))
}$

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define USERDEF "${ userdef }$"
#define LEN_USERDEF ${ write(len(userdef)) }$

char msg[] =
  "${ description }$";

void print_msg() {
  printf("%s", msg);
}

int complex_function(int value, int i) {
#define LAMBDA 3
  if (!('A' <= value && value <= 'Z')) {
    printf("Try again.\\n");
    exit(1);
  }
  return ((value - 'A' + (LAMBDA * i)) % ('Z' - 'A' + 1)) + 'A';
}

int main(int argc, char* argv[]) {
  char buffer[9];

  //print_msg();

  printf("Enter the password: ");
  scanf("%8s", buffer);

  for (int i=0; i<LEN_USERDEF; ++i) {
    buffer[i] = complex_function(buffer[i], i);
  }

  if (strcmp(buffer, USERDEF)) {
    printf("Try again.\\n");
  } else {
    printf("Good Job.\\n");
  }
}
```

코드를 보면 유저가 인풋을 넣으면 인풋을 바탕으로 어떤 연산 작업을 실행하고 그 결과 값을 컴파일 당시 랜덤하게 생성된 문자열과 비교하는 것을 알 수 있습니다. 그러면 우리가 해야하는 것은 연산 결과가 해당 문자열과 일치하는 인풋을 찾아야겠군요! 즉, main 함수에서 printf("Good Job.\n"); 이 실행될 수 있도록 해야한다는 거죠. 쉽죠? 이정도는 angr한테는 껌이겠죠? 이제 위 코드를 컴파일하고 아래 코드를 돌려봅시다!

```python
import angr

def main():
    proj = angr.Project('00_angr_find')
    init_state = proj.factory.entry_state()
    simulation = proj.factory.simgr(init_state)
    print_good = 0x804867d
    simulation.explore(find=print_good)

    if simulation.found:
        solution = simulation.found[0]
        print('flag: ', solution.posix.dumps(0))
    else:
        print('no solution')

if __name__ == '__main__':
    main()
```

컴파일된 바이너리를 IDA로 확인해서 printf("Good Job.\n");의 주소를 알아냅니다.(주소는 바뀔 수 있으니 각자 한번 확인해보세요) 그리고 simulation.explore(find=print_good)를 통해 값을 탐색합니다. 그리고 찾아지면 끝! 진짜 쉽죠? 아 물론 문제가 쉬워서 그럽니다 쿠쿸... 아직 이정도는 찍먹이라고 할 수도 없는 수준이죠.

# 마무리...

------

도대체 angr에서는 저 explore 함수에서 무슨 짓을 했길래 정답을 찾아낼 수 있었던 걸까요? 아마 여러분이 angr하면 떠오르는 키워드가 몇 가지 있을텐데, 바로 Symbolic Execution 덕분 입니다.

예... 다음 글 주제는 Symbolic Execution입니다!! 하하... 이번 편도 재미는 별로 없다고 생각이 드는데 다음 글은 어떡하지...하씨

그래도 마무리되었으니까 기쁜 마음으로 퇴근!!

![4](./angr_part1/1625731445.gif)

