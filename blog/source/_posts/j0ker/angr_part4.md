# 핵린이의 angr 정복기 - (4) angr_ctf part.2

검수 날짜: August 15, 2021
검수 여부: No
업로드 날짜: August 15, 2021
업로드 여부: No
작성자: 최광준
진행 상황: 진행중
카테고리: Research

![5jnh8a.jpg](angr_part4/5jnh8a.jpg)

# 이전 글 바로가기

[[Research] 핵린이의 angr 정복기 - (1) 시작](https://hackyboiz.github.io/2021/07/10/j0ker/angr_part1/)

[[Research] 핵린이의 angr 정복기 - (2) Symbolic Execution](https://hackyboiz.github.io/2021/07/21/j0ker/angr_part2/)

[[Research] 핵린이의 angr 정복기 - (3) angr_ctf part.1](https://hackyboiz.github.io/2021/08/04/j0ker/angr_part3/)

[Research] 핵린이의 angr 정복기 - (4) angr_ctf part.2 ← Here!

# 인사말

안녕하세요. j0ker 입니다! 이번에도 angr_ctf 문제들을 풀어보겠습니다. 근데 시리즈가 매우 길어질거 같은 예감은 틀리지 않았다는거... 생각보다 시리즈가 더 길어질 수 있겠습니다... 그리고 생각보다 더 고통스럽.... 하하

# 06_angr_symbolic_dynamic_memory

앞에 5번 문제에서는 전역변수에 대한 심볼을 지정했습니다. 그리고 제목에 다이나믹 메모리가 나온것을 보니 자연스럽게 이번에는 힙에 심볼을 지정하는 구나라고 추측해볼 수 있겠네요. 바로 바이너리 까보겠습니다.

![angr_part4/Untitled.png](angr_part4/Untitled.png)

역시 힙을 사용합니다. 두 개의 변수가 힙을 사용하고 있습니다. 앞에서와 마찬가지고 분석은 scanf 뒤에서부터 시작해야겠네요. 이 문제를 들여다보면 앞에서 스택관련된 문제를 풀었을 때와 같이 angr는 프로그램을 분석할 때 `malloc`의 결과값, 즉 힙이 어디에 할당되어 있는지 그 주소를 모른다는 것입니다. 그렇기 때문에 이 문제에서는 가상의 힙 주소를 우리가 임의로 지정을 해줘야 합니다.

```python
  fake_heap_address0 = 0x41414140
  pointer_to_malloc_memory_address0 = 0x0ABCC8A4
  initial_state.memory.store(pointer_to_malloc_memory_address0, fake_heap_address0, endness=project.arch.memory_endness)
  fake_heap_address1 = 0x41414150
  pointer_to_malloc_memory_address1 = 0x0ABCC8AC
  initial_state.memory.store(pointer_to_malloc_memory_address1, fake_heap_address1, endness=project.arch.memory_endness)
```

각각 변수의 크기가 9이기 때문에 alignment로 인해 아마 16바이트씩 할당이 될겁니다. 저는 일단 0x41414140과 0x41414150으로 16바이트 간격으로 설정해 주었고 힙 주소를 저장하고 있는 전역 변수의 주소들을 각각 `initial_state`에 설정해주었습니다. 아마 주소는 다르게 바꾸고 간격만 신경쓰면 큰 문제는 없을 듯 슾습니다. 다른 부분은 앞 문제들과 똑같네요!

## sol06.py

```python
import angr
import claripy
import sys

def main(argv):
  path_to_binary = argv[1]
  project = angr.Project(path_to_binary)

  start_address = 0x08048696
  initial_state = project.factory.blank_state(addr=start_address)

  password0 = claripy.BVS('password0', 64)
  password1 = claripy.BVS('password1', 64)
  
  fake_heap_address0 = 0x41414140
  pointer_to_malloc_memory_address0 = 0x0ABCC8A4
  initial_state.memory.store(pointer_to_malloc_memory_address0, fake_heap_address0, endness=project.arch.memory_endness)
  fake_heap_address1 = 0x41414150
  pointer_to_malloc_memory_address1 = 0x0ABCC8AC
  initial_state.memory.store(pointer_to_malloc_memory_address1, fake_heap_address1, endness=project.arch.memory_endness)
  

  initial_state.memory.store(fake_heap_address0, password0)
  initial_state.memory.store(fake_heap_address1, password1)
  

  simulation = project.factory.simgr(initial_state)

  def is_successful(state):
    stdout_output = state.posix.dumps(sys.stdout.fileno())
    return ('Good Job' in stdout_output.decode('ascii'))

  def should_abort(state):
    stdout_output = state.posix.dumps(sys.stdout.fileno())
    return ('Try again' in stdout_output.decode('ascii'))

  simulation.explore(find=is_successful, avoid=should_abort)

  if simulation.found:
    solution_state = simulation.found[0]

    solution0 = solution_state.se.eval(password0, cast_to=bytes)
    solution1 = solution_state.se.eval(password1, cast_to=bytes)
    
    solution = '{} {}'.format(solution0, solution1)

    print(solution)
  else:
    raise Exception('Could not find the solution')

if __name__ == '__main__':
  main(sys.argv)
```

```python
j0ker@angr:~/angr_ctf/dist$ python3 sol06.py 06_angr_symbolic_dynamic_memory 
WARNING | 2021-08-15 05:35:25,145 | angr.storage.memory_mixins.bvv_conversion_mixin | Unknown size for memory data 0x41414140. Default to arch.bits.
WARNING | 2021-08-15 05:35:25,145 | angr.storage.memory_mixins.bvv_conversion_mixin | Unknown size for memory data 0x41414150. Default to arch.bits.
WARNING | 2021-08-15 05:35:25,150 | angr.storage.memory_mixins.default_filler_mixin | The program is accessing memory or registers with an unspecified value. This could indicate unwanted behavior.
WARNING | 2021-08-15 05:35:25,150 | angr.storage.memory_mixins.default_filler_mixin | angr will cope with this by generating an unconstrained symbolic variable and continuing. You can resolve this by:
WARNING | 2021-08-15 05:35:25,150 | angr.storage.memory_mixins.default_filler_mixin | 1) setting a value to the initial state
WARNING | 2021-08-15 05:35:25,150 | angr.storage.memory_mixins.default_filler_mixin | 2) adding the state option ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to make unknown regions hold null
WARNING | 2021-08-15 05:35:25,150 | angr.storage.memory_mixins.default_filler_mixin | 3) adding the state option SYMBOL_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to suppress these messages.
WARNING | 2021-08-15 05:35:25,150 | angr.storage.memory_mixins.default_filler_mixin | Filling register ebp with 4 unconstrained bytes referenced from 0x8048699 (main+0x8d in 06_angr_symbolic_dynamic_memory (0x8048699))
WARNING | 2021-08-15 05:35:26,815 | angr.storage.memory_mixins.default_filler_mixin | Filling memory at 0x41414148 with 8 unconstrained bytes referenced from 0xac8a520 (strncmp+0x0 in libc.so.6 (0x8a520))
WARNING | 2021-08-15 05:35:26,816 | angr.storage.memory_mixins.default_filler_mixin | Filling memory at 0x41414158 with 105 unconstrained bytes referenced from 0xac8a520 (strncmp+0x0 in libc.so.6 (0x8a520))
WARNING | 2021-08-15 05:35:28,576 | angr.storage.memory_mixins.default_filler_mixin | Filling memory at 0x414141c1 with 16 unconstrained bytes referenced from 0xac8a520 (strncmp+0x0 in libc.so.6 (0x8a520))
CRITICAL | 2021-08-15 05:35:30,932 | angr.sim_state | The name state.se is deprecated; please use state.solver.
b'UBDKLMBV' b'UNOERNYS'
j0ker@angr:~/angr_ctf/dist$ ./06_angr_symbolic_dynamic_memory 
Enter the password: UBDKLMBV
UNOERNYS
Good Job.
j0ker@angr:~/angr_ctf/dist$
```

# 07_angr_symbolic_file

이제 다들 눈치 채시겠죠? 너무 뻔하게 파일을 심볼로 지정하는 문제겠네요 ㅋㅋ 하하 이쯤되면 눈감고도 문제 유형은 맞추실 수 있으실 겁니다.

![angr_part4/Untitled%201.png](angr_part4/Untitled%201.png)

일단 먼저 64바이트 입력값을 받고 파일에 씁니다. 중간에 `ignore_me`라는 함수가 있는 걸 보니 저 함수 뒤부터 분석하도록 만들면 될거 같네요. 그리고 파일을 쓰고 지운 다음 그 값으로 뭔가 한 다음에 문자열과 비교합니다. 처음에 생각했던거는 메모리 문제처럼 `buffer` 주소를 메모리 심볼로 지정하고 풀까했는데, 문제 스크립트를 보니 그렇게 풀 수 있지만 제발 문제 의도에 따라 달라고 합니다 ㅋㅋㅋ 그럼 스크립트에 따라 문제를 풀어보도록 합시다. 일단 파일 이름과 사이즈를 설정합니다. 이거는 바이너리를 까면 다 확인할 수 있죠?

그런 다음 이제 파일에 심볼을 설정합니다.

```python
symbolic_file_backing_memory = angr.state_plugins.SimSymbolicMemory()
symbolic_file_backing_memory.set_state(initial_state)

password = claripy.BVS('password', symbolic_file_size_bytes * 8)
symbolic_file_backing_memory.store(address_of_buffer, password)

file_options = 'r'
password_file = angr.storage.SimFile(filename, file_options, content=symbolic_file_backing_memory, size=symbolic_file_size_bytes)

symbolic_filesystem = {
    filename : password_file
  }
initial_state.posix.fs = symbolic_filesystem

```

`SimSymbolicMemory`를 통해 메모리에 심볼을 설정합니다? 왜죠???

설명을 보면 리눅스에서 파일은 시퀀셜 데이터의 스트림이라고 합니다. 꼭 파일일 필요는 없고 네트워크나 다른 프로그램의 결과값이 될 수도 있다고 하는데요. 암튼 무엇인가를 메모리에 매핑해주는 느낌인거 같습니다. 그리고 해당 메모리에 심볼을 지정하구요. `SimFile` 함수를 통해 파일을 시뮬레이션해주고 파일 이름과 매핑을 시켜줍니다. 이렇게하면 끝인거 같네요! 그럼 돌려봅시다!

```python
j0ker@angr:~/angr_ctf/dist$ ./06_angr_symbolic_dynamic_memory 
Enter the password: UBDKLMBV
UNOERNYS
Good Job.
j0ker@angr:~/angr_ctf/dist$ python3 scaffold07.py 07_angr_symbolic_file 
WARNING | 2021-08-15 05:39:36,180 | angr.storage.memory_mixins.bvv_conversion_mixin | Encoding unicode string for memory as utf-8. Did you mean to use a bytestring?
Traceback (most recent call last):
  File "scaffold07.py", line 130, in <module>
    main(sys.argv)
  File "scaffold07.py", line 47, in main
    symbolic_file_backing_memory = angr.state_plugins.SimSymbolicMemory()
AttributeError: module 'angr.state_plugins' has no attribute 'SimSymbolicMemory'
```

???????????????????????????????????? 없다고?????? 왜없는데???????

하아 이 때부터 멘탈이 나가기 시작했습니다. 하하 이런 고난 없이도 내 인생은 충분히 고통스러운거 같은데...

암튼 이 문제를 고치기 위해 검색을 좀 해보다가 아래 글을 찾게 되었습니다.

[Angr 9 SimFile without SimSymbolicMemory](https://jasper.la/posts/angr-9-simfile-without-simsymbolicmemory/)

이 글을 보면 이 angr_ctf가 만들어진지 매우 오래되었고 angr도 업데이트되면서 기능이 통합되고 구조화되면서 `SimFile`이 옮겨졌다고 합니다. 그래서 위에서 설명한 부분을 날려주고 간단하게 아래와 같이 수정해주면 됩니다.

```python
password_file = angr.SimFile(filename,
          content = password,
          size = symbolic_file_size_bytes)

initial_state = project.factory.blank_state(
          addr = start_address,
          fs = {filename: password_file}
)
```

이렇게만 봐도 훨씬더 쓰기 편해진거 같네요. 일단 `SimFile`을 통해 바로 파일을 시뮬레이션해줄 수 있게 되었습니다. 그리고 `black_state`의 `fs` 인자에 파일 이름과 파일 시뮬레이터만 매핑해서 전달만 해주면 끝!!!

## sol07.py

```python
import angr
import claripy
import sys

def main():
  project = angr.Project('07_angr_symbolic_file')
  start_address = 0x080488D6

  filename = 'OJKSQYDP.txt'
  symbolic_file_size_bytes = 0x40

  password = claripy.BVS('password', symbolic_file_size_bytes * 8)

  password_file = angr.SimFile(filename,
          content = password,
          size = symbolic_file_size_bytes)

  initial_state = project.factory.blank_state(
          addr = start_address,
          fs = {filename: password_file}
  )

  simulation = project.factory.simgr(initial_state)

  def is_successful(state):
    stdout_output = state.posix.dumps(sys.stdout.fileno())
    return b'Good Job.' in stdout_output

  def should_abort(state):
    stdout_output = state.posix.dumps(sys.stdout.fileno())
    return b'Try again.' in stdout_output

  simulation.explore(find=is_successful, avoid=should_abort)

  if simulation.found:
    solution_state = simulation.found[0]

    solution = solution_state.solver.eval(password,cast_to=bytes).decode()
    print(solution)
  else:
    raise Exception('Could not find the solution')

if __name__ == '__main__':
  main()
```

```python
j0ker@angr:~/angr_ctf/dist$ python3 sol07.py 07_angr_symbolic_file 
WARNING | 2021-08-15 05:56:08,302 | angr.storage.memory_mixins.default_filler_mixin | The program is accessing memory or registers with an unspecified value. This could indicate unwanted behavior.
WARNING | 2021-08-15 05:56:08,302 | angr.storage.memory_mixins.default_filler_mixin | angr will cope with this by generating an unconstrained symbolic variable and continuing. You can resolve this by:
WARNING | 2021-08-15 05:56:08,302 | angr.storage.memory_mixins.default_filler_mixin | 1) setting a value to the initial state
WARNING | 2021-08-15 05:56:08,302 | angr.storage.memory_mixins.default_filler_mixin | 2) adding the state option ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to make unknown regions hold null
WARNING | 2021-08-15 05:56:08,302 | angr.storage.memory_mixins.default_filler_mixin | 3) adding the state option SYMBOL_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to suppress these messages.
WARNING | 2021-08-15 05:56:08,302 | angr.storage.memory_mixins.default_filler_mixin | Filling register ebp with 4 unconstrained bytes referenced from 0x804893c (main+0xc2 in 07_angr_symbolic_file (0x804893c))
AZOMMMZM
j0ker@angr:~/angr_ctf/dist$ ./07_angr_symbolic_file 
Enter the password: AZOMMMZM
Good Job.
```

# 08_angr_constraints

이번에는 경로제약조건에 대한 문제일듯 싶습니다. 프로그램을 까보면 그냥 일반적인 문제입니다.

![Untitled](angr_part4/Untitled%202.png)

`password`에 문자열이 있고 16바이트 입력값을 받은 뒤 입력값과 `password`를 짬짜미하여 특정 문자열과 같은지 비교하는 거겠죠. 근데 연산 결과값을 비교하는 부분이 함수로 되어 있네요. 이 함수도 봅시다.

![Untitled](angr_part4/Untitled%203.png)

함수를 보면 두 문자열을 비교하는 건 맞지만 틀리는 즉시 리턴하는게 아니라 문자 몇개가 맞는지 센다음 몇개가 맞는지 16바이트가 맞는지를 비교하고 리턴하네요. angr에서 이런 함수를 분석하게 되면 무수히 많은 state가 생성될게 뻔합니다. 중간에 멈출수가 없으니 말이죠. 거기에다가 16바이트짜리 문자열이면 엄청난 경우의 수(branch)가 있으니 이 함수는 애초에 분석하면 안될듯 싶습니다. 따라서 이 함수에 들어가기 전에 연산 결과값을 우리가 직접 비교하는게 좀 더 효율적일듯 싶네요.

일단 먼저 `scanf` 뒤에서 시작해서 `check_equals_` 함수까지 분석해 도달할 수 있도록 스크립트를 작성합니다. 그리고 `check_equals_`함수에 들어가기 전에 문자열을 체크합니다.

```python
    constrained_parameter_address = 0x0804A050
    constrained_parameter_size_bytes = 16
    constrained_parameter_bitvector = solution_state.memory.load(
      constrained_parameter_address,
      constrained_parameter_size_bytes
    )

    constrained_parameter_desired_value = 'AUPDNNPROEZRJWKB'
    solution_state.add_constraints(constrained_parameter_bitvector == constrained_parameter_desired_value)
```

일단 입력값의 주소와 사이즈를 지정해주고 메모리에서 로드합니다. 그리고 해당 메모리에 있는 값과 우리가 결과적으로 맞아야할 값이 같아야한다는 제약 조건을 추가해줍니다. 이렇게 되면 z3에서 결과값을 도출할 때 우리가 설정한 제약조건을 추가하여 계산을 해줍니다! 그러면 끝!

이 문제에서 조금 의문이었던 점은 처음에 입력값이 하나인 것을 보고 그냥 `entry_state`부터 분석하도록 프로젝트를 설정했는데 안되더라고요? 그래서 저는 제가 문제를 잘못푼 줄 알았는데, `scanf` 뒤로 설정하니까 문제가 풀렸습니다... 처음에는 angr에서 특정 크기 이상의 입력값을 인식하지 못하나? 했는데, 다음 문제에서는 또 정상적으로 됩니다 ㅋㅋㅋ 이게 뭔지...

## sol08.py

```python
import angr
import claripy
import sys

def main(argv):
  path_to_binary = argv[1]
  project = angr.Project(path_to_binary)

  start_address = 0x08048625
  initial_state = project.factory.blank_state(addr=start_address)

  # initial_state = project.factory.entry_state()

  password = claripy.BVS('password', 8*16)

  password_address =  0x0804A050
  initial_state.memory.store(password_address, password)

  simulation = project.factory.simgr(initial_state)

  address_to_check_constraint = 0x08048673
  simulation.explore(find=address_to_check_constraint)

  if simulation.found:
    solution_state = simulation.found[0]

    constrained_parameter_address = 0x0804A050
    constrained_parameter_size_bytes = 16
    constrained_parameter_bitvector = solution_state.memory.load(
      constrained_parameter_address,
      constrained_parameter_size_bytes
    )

    constrained_parameter_desired_value = 'AUPDNNPROEZRJWKB'
    solution_state.add_constraints(constrained_parameter_bitvector == constrained_parameter_desired_value)

    solution = solution_state.se.eval(password, cast_to=bytes)

    print(solution)
  else:
    raise Exception('Could not find the solution')

if __name__ == '__main__':
  main(sys.argv)
```

```python
j0ker@angr:~/angr_ctf/dist$ python3 sol08.py 08_angr_constraints 
WARNING | 2021-08-15 06:22:55,285 | angr.storage.memory_mixins.default_filler_mixin | The program is accessing memory or registers with an unspecified value. This could indicate unwanted behavior.
WARNING | 2021-08-15 06:22:55,285 | angr.storage.memory_mixins.default_filler_mixin | angr will cope with this by generating an unconstrained symbolic variable and continuing. You can resolve this by:
WARNING | 2021-08-15 06:22:55,285 | angr.storage.memory_mixins.default_filler_mixin | 1) setting a value to the initial state
WARNING | 2021-08-15 06:22:55,285 | angr.storage.memory_mixins.default_filler_mixin | 2) adding the state option ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to make unknown regions hold null
WARNING | 2021-08-15 06:22:55,285 | angr.storage.memory_mixins.default_filler_mixin | 3) adding the state option SYMBOL_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to suppress these messages.
WARNING | 2021-08-15 06:22:55,285 | angr.storage.memory_mixins.default_filler_mixin | Filling register ebp with 4 unconstrained bytes referenced from 0x8048625 (main+0x72 in 08_angr_constraints (0x8048625))
WARNING | 2021-08-15 06:22:57,291 | claripy.ast.bv | BVV value is being coerced from a unicode string, encoding as utf-8
WARNING | 2021-08-15 06:22:57,292 | claripy.ast.bv | BVV value is being coerced from a unicode string, encoding as utf-8
CRITICAL | 2021-08-15 06:22:57,310 | angr.sim_state | The name state.se is deprecated; please use state.solver.
b'LGCRCDGJHYUNGUJB'
j0ker@angr:~/angr_ctf/dist$ ./08_angr_constraints 
Enter the password: LGCRCDGJHYUNGUJB
Good Job.
```

# 09_angr_hooks

제목만 보면 뭔갈 후킹해야하는거 같네요. 뭘 후킹해야할 지 봅시다.

![Untitled](angr_part4/Untitled%204.png)

보면 입력을 두번 받는데, 일단 첫번째 받는걸 아까와 같은 함수로 체크를 하네요. 이 부분을 후킹해서 뛰어넘어야할듯 보입니다. 뒤에는 뭐 그냥 비교하는거니 별거 없구요. 후킹은 해본적이 없으니 문제 스크립트를 보면서 풀어보죠!

```python
  check_equals_called_address = 0x080486B3
  instruction_to_skip_length = 5
```

일단 어디서부터 후킬할 것인지 지정을 해줍니다. 저 같은 경우에는 딱 check_equals_ 함수가 호출되는 부분부터 후킹하도록 주소를 지정했습니다. 해당 인스트럭션의 길이가 5이기 때문에 사이즈는 5로 지정을 해주었구요. 그리고 이제 후킹 함수를 지정해 줍니다.

```python
  @project.hook(check_equals_called_address, length=instruction_to_skip_length)
  def skip_check_equals_(state):
    user_input_buffer_address = 0x0804A054
    user_input_buffer_length = 16

    user_input_string = state.memory.load(
      user_input_buffer_address, 
      user_input_buffer_length
    )
    
    check_against_string = 'XYMKBKUHNIQYNQXE' # :string

    state.regs.eax = claripy.If(
      user_input_string == check_against_string, 
      claripy.BVV(1, 32), 
      claripy.BVV(0, 32)
    )

  simulation = project.factory.simgr(initial_state)
```

hook 함수에 위에서 설정한 내용들을 인자로 넘겨주게 됩니다. 이 때부터 후킹하도록 지시를 하는거겠죠. 그런 다음 함수를 하나 선언해주는데 이게 이제 후킹할 대상을 건너뛰는 대신 실행할 내용인듯 합니다.

여기에서는 연산된 결과값이 특정 문자열인지 비교해주고 결과 값이 1이 되는도록 해야겠죠? 먼저 비교해야할 메모리에서 값을 로트하고 최종적으로 비교해야할 문자열을 지정해줍니다. 아시다시피 함수의 리턴값은 보통 eax에 저장됩니다. 따라서 마지막에 claripy를 사용해서 인자로 넣은 조건, 즉 연산값이 우리가 비교해야할 문자열과 같은지 여부에 따라 eax를 1로 설정할 것이냐 0으로 설정할 것이냐를 지정해줍니다. 그러면 시뮬레이터가 알아서 시뮬레이션을 해주겠죠. 나머지 뒤에 부분은 앞 문제들고 같습니다. 

## sol09.py

```python
import angr
import claripy
import sys
import traceback
import logging

def main(argv):
  path_to_binary = argv[1]
  project = angr.Project(path_to_binary)

  initial_state = project.factory.entry_state()

  check_equals_called_address = 0x080486B3
  instruction_to_skip_length = 5
  print(type(project.hook))
  @project.hook(check_equals_called_address, length=instruction_to_skip_length)
  def skip_check_equals_(state):
    for line in traceback.format_stack():
      print(line.strip())
    user_input_buffer_address = 0x0804A054
    user_input_buffer_length = 16

    user_input_string = state.memory.load(
      user_input_buffer_address, 
      user_input_buffer_length
    )
    
    check_against_string = 'XYMKBKUHNIQYNQXE'

    state.regs.eax = claripy.If(
      user_input_string == check_against_string, 
      claripy.BVV(1, 32), 
      claripy.BVV(0, 32)
    )

  simulation = project.factory.simgr(initial_state)

  def is_successful(state):
    stdout_output = state.posix.dumps(sys.stdout.fileno())
    return ('Good Job' in stdout_output.decode('ascii'))

  def should_abort(state):
    stdout_output = state.posix.dumps(sys.stdout.fileno())
    return ('Try again' in stdout_output.decode('ascii'))

  simulation.explore(find=is_successful, avoid=should_abort)

  if simulation.found:
    solution_state = simulation.found[0]
    solution = solution_state.posix.dumps(sys.stdin.fileno())
    print(solution)
  else:
    raise Exception('Could not find the solution')

if __name__ == '__main__':
  main(sys.argv)
```

```python
j0ker@angr:~/angr_ctf/dist$ python3 sol09.py 09_angr_hooks 
WARNING | 2021-08-15 06:44:36,774 | angr.storage.memory_mixins.default_filler_mixin | The program is accessing memory or registers with an unspecified value. This could indicate unwanted behavior.
WARNING | 2021-08-15 06:44:36,774 | angr.storage.memory_mixins.default_filler_mixin | angr will cope with this by generating an unconstrained symbolic variable and continuing. You can resolve this by:
WARNING | 2021-08-15 06:44:36,774 | angr.storage.memory_mixins.default_filler_mixin | 1) setting a value to the initial state
WARNING | 2021-08-15 06:44:36,774 | angr.storage.memory_mixins.default_filler_mixin | 2) adding the state option ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to make unknown regions hold null
WARNING | 2021-08-15 06:44:36,774 | angr.storage.memory_mixins.default_filler_mixin | 3) adding the state option SYMBOL_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to suppress these messages.
WARNING | 2021-08-15 06:44:36,775 | angr.storage.memory_mixins.default_filler_mixin | Filling register edi with 4 unconstrained bytes referenced from 0x8048791 (__libc_csu_init+0x1 in 09_angr_hooks (0x8048791))
WARNING | 2021-08-15 06:44:36,776 | angr.storage.memory_mixins.default_filler_mixin | Filling register ebx with 4 unconstrained bytes referenced from 0x8048793 (__libc_csu_init+0x3 in 09_angr_hooks (0x8048793))
WARNING | 2021-08-15 06:44:38,681 | claripy.ast.bv | BVV value is being coerced from a unicode string, encoding as utf-8
b'ZXIDRXEORJOTFFJNWUFAOUBLOGLQCCGK'
j0ker@angr:~/angr_ctf/dist$ ./09_angr_hooks 
Enter the password: ZXIDRXEORJOTFFJNWUFAOUBLOGLQCCGK
Good Job.
```

## 데코레이터 @ 그리고 hook 작동원리?

사실 이렇게 문제는 풀렸지만 저는 이 후킹이 어떻게 동작하는지 매우 궁금했습니다. 데코레이터라는 것도 코드를 볼 때 어렴풋이 알고만 있었지 제대로는 몰라서 이참에 좀 더 공부해 보기로 했습니다.

### 데코레이터

데코레이터는 아래 함수를 `wrapping`해서 함수가 실행될 때 자신이 원하는 코드를 추가하여 실행할 수 있는 기능이라고 합니다. 주로 디버깅할 때나 코드 간결화를 위해 많이 쓰인다고 합니다. 

간단한 예시를 한번 보겠습니다.

Code:

```python
def decoratorFunctionWithArguments(arg1, arg2, arg3):
    def wrap(f):
        print "Inside wrap()"
        def wrapped_f(*args):
            print "Inside wrapped_f()"
            print "Decorator arguments:", arg1, arg2, arg3
            f(*args)
            print "After f(*args)"
        return wrapped_f
    return wrap

@decoratorFunctionWithArguments("hello", "world", 42)
def sayHello(a1, a2, a3, a4):
    print 'sayHello arguments:', a1, a2, a3, a4

print "After decoration"

print "Preparing to call sayHello()"
sayHello("say", "hello", "argument", "list")
print "after first sayHello() call"
sayHello("a", "different", "set of", "arguments")
print "after second sayHello() call"
```

Output:

```python
Inside wrap()
After decoration
Preparing to call sayHello()
Inside wrapped_f()
Decorator arguments: hello world 42
sayHello arguments: say hello argument list
After f(*args)
after first sayHello() call
Inside wrapped_f()
Decorator arguments: hello world 42
sayHello arguments: a different set of arguments
After f(*args)
after second sayHello() call
```

> 출처 : [Python Decorators II: Decorator Arguments](https://www.artima.com/weblogs/viewpost.jsp?thread=240845)

문제에서 `hook` 함수를 사용하는 것과 비슷한 예시를 가져와 봤습니다. 일단 데코레이터 @를 사용하고 뒤에 함수를 호출해줍니다. 이 때 wrap 함수의 포인터를 반환하지만 실행은 하지 않습니다. 반면에 `sayHello` 함수가 실행되면 이 때 wrap 함수가 실행이 되면서 `wrapped_f` 함수를 리턴하는데, 이 함수 포인터를 다시 받아서 `sayHello`에 전달된 인자들이 `wrapped_f` 함수에 전달되면서 실행됩니다. 그리고 `wrapped_f` 함수 안에서 `f(*args)`를 통해 `sayHello` 함수가 실행됩니다.

즉 데코레이터는 @ 뒤에 붙은 함수를 실행하고 리턴된 함수 포인터를 받아 데코레이터 밑에 있는 함수가 실행되면 해당 함수를 인자로 데코레이터에서 리턴받은 함수를 실행한다는 것을 알 수 있습니다. 뭔가 좀 복잡하네요....

```python
sayHello(a,b,c,d) -> wrap(sayHello) -> wrapped(a,b,c,d) -> sayHello(a,b,c,d)
```

저도 공부하면서 정리를 하고 있는데 좀 더 자세한 설명은 아래 링크를 확인하시면 될거 같습니다.

[Decorators I: Introduction to Python Decorators](https://www.artima.com/weblogs/viewpost.jsp?thread=240808)

데코레이터를 공부하고 나서 제가 가장 의문이었던 점은 모든 예시에서는 데코레이터 아래 선언된 함수들이 호출이 되는데, 왜 솔루션 스크립트에서는 호출이 안됐음에도 불구하고 잘 동작을 하는가? 였습니다. 이 부분을 확인하기 위해 angr 코드를 좀 살펴봤습니다.

```python
def hook(self, addr, hook=None, length=0, kwargs=None, replace=False):
        """
        Hook a section of code with a custom function. This is used internally to provide symbolic
        summaries of library functions, and can be used to instrument execution or to modify
        control flow.

        When hook is not specified, it returns a function decorator that allows easy hooking.
        Usage::

            # Assuming proj is an instance of angr.Project, we will add a custom hook at the entry
            # point of the project.
            @proj.hook(proj.entry)
            def my_hook(state):
                print("Welcome to execution!")

        :param addr:        The address to hook.
        :param hook:        A :class:`angr.project.Hook` describing a procedure to run at the
                            given address. You may also pass in a SimProcedure class or a function
                            directly and it will be wrapped in a Hook object for you.
        :param length:      If you provide a function for the hook, this is the number of bytes
                            that will be skipped by executing the hook by default.
        :param kwargs:      If you provide a SimProcedure for the hook, these are the keyword
                            arguments that will be passed to the procedure's `run` method
                            eventually.
        :param replace:     Control the behavior on finding that the address is already hooked. If
                            true, silently replace the hook. If false (default), warn and do not
                            replace the hook. If none, warn and replace the hook.
        """
        if hook is None:
            # if we haven't been passed a thing to hook with, assume we're being used as a decorator
            return self._hook_decorator(addr, length=length, kwargs=kwargs)

        if kwargs is None: kwargs = {}

        l.debug('hooking %s with %s', self._addr_to_str(addr), str(hook))

        if self.is_hooked(addr):
            if replace is True:
                pass
            elif replace is False:
                l.warning("Address is already hooked, during hook(%s, %s). Not re-hooking.", self._addr_to_str(addr), hook)
                return
            else:
                l.warning("Address is already hooked, during hook(%s, %s). Re-hooking.", self._addr_to_str(addr), hook)

        if isinstance(hook, type):
            raise TypeError("Please instanciate your SimProcedure before hooking with it")

        if callable(hook):
            hook = SIM_PROCEDURES['stubs']['UserHook'](user_func=hook, length=length, **kwargs)

        self._sim_procedures[addr] = hook
```

`hook` 함수를 보면 먼저 인자 `hook`이 `None`이면 `_hook_decorator` 함수가 실행이 됩니다.

```python
def _hook_decorator(self, addr, length=0, kwargs=None):
        """
        Return a function decorator that allows easy hooking. Please refer to hook() for its usage.

        :return: The function decorator.
        """

        def hook_decorator(func):
            self.hook(addr, func, length=length, kwargs=kwargs)
            return func

        return hook_decorator

    #
    # Pickling
    #
```

이 함수는 위 예제에서 본 `wrap` 함수가 안에 들어 있는것처럼 보이네요. 그리고 안에서 선언한 `hook_decorator` 함수 포인터를 리턴합니다. 그리고 `hook_decorator` 함수가 불리면 다시 `hook` 함수를 호출하는데, 이 때는 전달된 데코레이터 아래 함수의 포인터도 같이 전달이 됩니다. 그러면 `hook` 함수에서는 `SIM_PROCEDURES['stubs']['UserHook'](user_func=hook, length=length, **kwargs)`에서 해당 함수를 실행하는 것처럼 보이고요. 흠 일단 데코레이션 포맷?에 맞춰 제작되어 있는 것은 알았습니다. 그래서 skip_check_equals_ 함수는 어디서 호출하냐고!!! 

일단 콜스택을 봐야할거 같아서 아래 코드를 추가해보았습니다.

```python
for line in traceback.format_stack():
      print(line.strip())
```

```python
File "scaffold09.py", line 99, in <module>
    main(sys.argv)
File "scaffold09.py", line 86, in main
    simulation.explore(find=is_successful, avoid=should_abort)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/sim_manager.py", line 250, in explore
    self.run(stash=stash, n=n, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/sim_manager.py", line 280, in run
    self.step(stash=stash, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/misc/hookset.py", line 75, in __call__
    result = current_hook(self.func.__self__, *args, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/exploration_techniques/explorer.py", line 96, in step
    return simgr.step(stash=stash, extra_stop_points=base_extra_stop_points | self._extra_stop_points, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/misc/hookset.py", line 80, in __call__
    return self.func(*args, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/sim_manager.py", line 365, in step
    successors = self.step_state(state, successor_func=successor_func, **run_args)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/sim_manager.py", line 402, in step_state
    successors = self.successors(state, successor_func=successor_func, **run_args)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/sim_manager.py", line 441, in successors
    return self._project.factory.successors(state, **run_args)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/factory.py", line 60, in successors
    return self.default_engine.process(*args, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/engines/vex/light/slicing.py", line 19, in process
    return super().process(*args, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/engines/engine.py", line 149, in process
    self.process_successors(self.successors, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/engines/failure.py", line 21, in process_successors
    return super().process_successors(successors, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/engines/syscall.py", line 18, in process_successors
    return super().process_successors(successors, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/engines/hook.py", line 61, in process_successors
    return self.process_procedure(state, successors, procedure, **kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/engines/procedure.py", line 37, in process_procedure
    inst = procedure.execute(state, successors, ret_to=ret_to, arguments=arguments)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/sim_procedure.py", line 230, in execute
    r = getattr(inst, inst.run_func)(*sim_args, **inst.kwargs)
File "/home/j0ker/.local/lib/python3.8/site-packages/angr/procedures/stubs/UserHook.py", line 8, in run
    result = user_func(self.state)
File "scaffold09.py", line 42, in skip_check_equals_
    for line in traceback.format_stack():
```

사실 이 때부터 감이 왔습니다. 내가 감당할 수 없는 영역을 건드렸구나... 이걸 언제 다 분석하고 있냐... 마감이 코앞인데... 하아...

그래서 오늘은 여기서 끝!!!! 캬캬 이 부분은 제가 분석을 다 해서 시리즈 중간에 따로 정리해 보겠습니다. 언제 끝날지는 장담을 못하지만... 그래도 궁금한 점이 있으면 어떻게든 해결은 해야겠죠.

# 마무리

예상보다 삽질하는 부분이 많아서 시리즈가 좀 루즈해지고 있습니다. 조회수가 많이 나올만한 글도 아니어서 후딱 진행해야하는데... 다음이나 다다음에는 angr_ctf 끝내고 리얼 CTF 문제를 풀도록 하겠습니다. 답답하신 분들이 있으면 죄송합니다. 궁금한 부분은 그냥 넘어갈 수가 없었어요 ㅠㅠ 아마 나머지 문제 푸는 과정에서도 궁금한게 많이 있을거 같은데, 최대한 다 제대로 짚고 넘어가면서 다뤄보겠습니다(앞으로도 답답할 예정이라는 뜻).

그럼 조-바!