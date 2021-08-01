---
title: "[Translation] Windows Command-Line Obfuscation"
author: idioth
tags: [idioth, windows, cmd, obfuscation]
categories: [Translation]
date: 2021-08-01 18:00:00
cc: false
index_img: /2021/08/01/idioth/windows_cmd_obfuscation/Untitled 2.png
---


안녕하세요. idioth 입니다. 원래 오늘부터 Hooking에 대한 시리즈를 진행하려고 하였으나 모아둔 자료를 취합하고 구상하던 중에 꼬여 버려서 읽으려고 가져다 놓았던 것 중 하나를 번역해왔습니다.

잠깐 휴식하면서 시리즈 구상을 제대로 하고 시작하도록 하겠습니다.

> 원문 글 : [Windows Command-Line Obfuscation](https://www.wietzebeukema.nl/blog/windows-command-line-obfuscation)

**TL;DR** - 많은 Windows 애플리케이션은 편의성이나 호환성을 위해 같은 command line을 표현하는 여러 가지 방법이 있다. command-line 인자가 여러 가지로 표현되다 보니 특정 명령어를 탐지하기가 매우 어렵다. 이 게시글은 자주 사용되는 40개 이상의 기본 Windows 애플리케이션이 command-line 난독화에 얼마나 취약한지 보여주고 다른 실행 파일을 분석하기 위한 툴을 소개한다.

# Command-line arguments

> He who wishes to be obeyed must know how to commad
- Machiavelli (Discourses on Livy III, Chapter XXII)

컴퓨터에게는 16세기의 이 인용문이 많이 적용된다. 대부분의 운영체제와 프로세스는 부모 프로세스가 자식 프로세스에게 정보를 전달할 수 있는 'command line' 구성요소를 가진다. command line은 새로 생성된 프로세스에 접근할 수 있으며 프로세스의 흐름을 바꿀 수 있다. 명령어 집합, 프로그램 실행, 입력 기능은 컴퓨터를 만드는 핵심 개념이다.

컴퓨팅에 대한 근본적인 역할에도 불구하고 command line에서 볼 수 있는 다양한 부분을 어떻게 불러야하는 지에 대한 합의점이 없다. 어느 부분에서는 command line arguments, parameters, options, flags, switches가 서로 다른 의미를 갖는다고 하지만 이 게시글에서는 다음 용어를 사용한다.

![](windows_cmd_obfuscation/Untitled.png)

라인 전체가 command line이며 공백으로 인자가 구분된다. 모든 인자는 역할이 다르다. 예시로 첫 번째 인자는 보통 프로세스이며 명령 프롬프트의 경우 명령어가 된다. 특히 처음 세 가지는 일반적으로 특수 문자로 시작하는 command line 옵션이다. 처음 두 가지 (`/f /unicode`)는 특별한 입력 값을 요하지 않으므로 switch이고 세 번째 인자 (`/decode`)는 뒤의 인자 (`folder\encoded.b64`)가 이 인자에 대한 입력 값인 flag이다.

`option char`는 command-line options에서 prefix로 사용되는 문자이다. Windows에서 보통 슬래시 (`/`)를 사용하며 유닉스 계열 운영체제에서는 하이푼 (`-`)을 쓴다. command line은 실행 중인 프로그램에 의해서 구문 분석되므로 개발자들은 어떤 형식의 인자를 전달해야 하는지에 대해서 고민하지 않아도 된다.

컴퓨터 역사에서 표준화가 이루어지지 않으면 매우 혼란스러운 경우가 있는데 command line에도 적용된다.

# 호환성 (Compatibility)

command-line 구문 분석 규칙에 대한 표준화가 미흡하여 많은 사용자에게 혼란을 유발한다. 사용자를 돕기 위해서 몇몇 프로그램은 다양한 규칙이 허용되도록 설계되었다. 예시로 슬래시와 하이푼을 옵션 문자로 모두 사용할 수 있다. 이에 대해서는 다음 부분에서 자세히 설명한다.

혼란의 다른 요인은 다양한 문자열 인코딩이다. 이로 인해 오래된 프로그램 (보통 ASCII만 허용)과 최신 프로그램 (UTF-8과 유니코드 허용) 간의 호환성 문제가 발생한다. 일부 프로그램에서는 이러한 문제를 특정 문자 필터링이나 특정 문자를 ASCII 문자로 변환하여 해결하고자 한다.

이를 통해서 일부 프로그램에서 여러 개의 command line 인자를 하나로 다루는 문제가 생긴다. 프로세스에 전달된 데이터가 다름에도 실행 및 결과는 동일한데, 이런 인스턴스를 synonymous command line이라고 부른다.

# 탐지 문제 (Problems for detection)

같은 명령어를 다른 방법으로 표현하는 것은 일부 사용자에게는 도움이 되지만 탐지나 감지를 어렵게 한다. 대부분 위협 탐지 소프트웨어 (AV, EDR 등)는 프로세스 실행 및 command-line 인자의 악의적 사용을 감시한다. 해커는 탐지를 피하고 싶을 것이고 command line의 유연성을 통해 탐지를 우회할 수 있다. 이는 키워드 기반 탐지 우회부터 명령어를 다 난독화하는 방법까지 다양하다.

command-line 난독화의 좋은 예시 중 하나는 Windows 명령 프롬프트 CMD에 초점을 맞춘 Daniel Bohannon의 DOSfuscation [[1](https://github.com/danielbohannon/Invoke-DOSfuscation), [2](https://www.fireeye.com/content/dam/fireeye-www/blog/pdfs/dosfuscation-report.pdf)]이다. 난독화된 명령어를 CMD가 이해하고 실행하더라도 사람이 이해할 수 없을 수 있고, 모니터링 소프트웨어 또한 탐지하지 못할 수 있다.

synonymous command line은 CMD 뿐만 아니라 더 많은 프로그램에 적용된다. 중요한 차이점은 명령 프롬프트가 아닌 실행 프로그램 자체를 속이는 것이다. 뒤에서 알 수 있듯이 DOSfuscation은 탐지 소프트웨어에 의해 난독화되지 않은 형태로 남을 수 있지만 synonymous command line은 그렇지 않다.

# Synonymous command lines: methods

실제도 작동하는지 확인하기 위해 synonymous command line을 유발할 수 있는 5 가지 메서드를 살펴보자.

## Option Char substitution

Windows 실행 파일 `ping`을 생각해보자. 원본이 유닉스 버전이므로 command-line option에 하이푼을 사용해야 한다. ( e.g. `ping -n 127.0.0.1`) 이것은 슬래시 (`/`)를 사용하는 다른 Windows command-line 툴과는 다르다. 사용자의 혼란을 줄이기 위해 option character로 슬래시를 사용하는 것 (`ping /n 127.0.0.1`) 또한 허용한다.

하이푼을 사용하는 대부분의 기본 내장 Windows 실행 파일들은 슬래시도 사용할 수 있지만, 그 반대는 아닌 경우가 많다. 'keyword' 단어가 포함된 모든 파일을 보여주는 `find /i keyword` 명령어는 `find -i keyword`로 동작하지 않는다.

슬래시와 하이푼이 일반적으로 사용 가능한 옵션이지만 일부 프로그램은 더 많은 option character를 지원한다. `certutil`은 하이푼, 슬래시 뿐만 아니라 division slash (0x2215), fraction slash (0x2044) 같은 유니코드 슬래시도 허용한다.

확인할 수 있듯이 대체로 사용되는 command-line option에 대해서 실행 파일들은 문서화를 잘 하지 않으므로 무차별 대입 (brute force)나 리버싱을 통해 알아내야 한다.

## Character substitution

또 다른 접근법은 command line에 있는 다른 문자 (option char가 아닌 문자)를 비슷한 문자로 바꾸는 것이다. 특히 유니코드의 범위를 고려할 때 ASCII 범위에도 프로세스가 허용하는 여러 문자가 있다.

유니코드에는 공백 문자 (0x02B0 ~ 0x02FF)가 포함되어 있고, ˪, ˣ, ˢ 같은 문자도 포함한다. 일부 command-line parser는 이러한 문제를 l, x, s로 변환한다. 예시로 `reg export HKCU out.reg`와 `reg eˣport HKCU out.reg`는 같다.

![](windows_cmd_obfuscation/Untitled%201.png)

> `eˣport HKCU out.reg`가 성공적으로 실행되는 예시

이를 통해 일부 프로그램에서 사용할 수 있는 유니코드 범위가 더 많아졌다.

## Character insertion

비슷한 맥락에서 실행 프로그램에서 무시하는 문자를 command-line에 삽입할 수 있다. 일부 실행 파일은 출력할 수 없는 문자를 제거하며 특정 문자를 필터링할 수도 있다.

Windows 이벤트 로그 툴 `wevtutil`은 특정 범위의 유니코드 문자열을 허용한다. `wevtutil gli hardwareevents`와 `wevtutil gࢯli hardwareevents`는 첫 번째 인자의 중간에 아랍어가 포함되어 있지만 같은 결과를 출력한다. 

![](windows_cmd_obfuscation/Untitled%202.png)

> `wevtutil gࢯli hardwareevents`가 성공적으로 실행되는 예시

이 기법에 사용할 수 있는 문자가 명령 프롬프트의 stdin에서 지원하지 않을 수 있으므로 바이트 표기법을 사용해 문자를 삽입할 수도 있다.

## Quotes insertion

quote 쌍을 삽입하여 프로세스의 흐름을 유지하면서 command-line을 조작할 수 있다. 인자에 quote를 사용하는 것은 익숙할 것이다. 예를 들어 `dir "C:\Windows\"`는 `dir c:\Windows\`와 같다. 대부분 프로그램은 이를 허용하며 임의의 위치에서 quote를 받아들인다. `dir c:"win"d""ow"s"`도 작동한다.

![](windows_cmd_obfuscation/Untitled%203.png)

> `netsh ad"vfi"rewall show currentprofile state`가 성공적으로 실행되는 예시

명령 프롬프트에서 quote를 사용하면 프로그램으로 전달하기 전에 자체적으로 처리하여 까다로울 수 있다. cmd에서 이 문제를 해결하는 방법은 모든 quote를 두 배로 늘려 위와 같이 실행하려면 다음과 같다.

```jsx
netsh ad""vfi""rewall show currentprofile state
```

## Shorthands

문자를 삽입하고 교체하고 나면 문자를 제거해야 한다. 일부 애플리케이션은 'shorthands'를 허용하므로 쉽게 입력할 수 있다.

이는 Unix 계열 툴에서는 잘 알려져 있지만 Windows에는 그렇지 않다. (`grep -i keyword` vs `grep --ignore-case keyword`) 하지만 Windows의 일부 프로그램은 단축어가 있다. Unix와 유사한 방식을 통해 한 글자로 줄이며 (`cmdkey /l` , `cmdkey /list`), 약어를 사용한다. (`wevtutil gli`, `wevtutil get-loginfo`) 또 다른 프로그램에서는 wildcard 접근법을 사용한다. Powershell을 예로 들면 powershell의 키워드 끝에 하나 이상의 문자를 남길 수 있다.

![](windows_cmd_obfuscation/Untitled%204.png)

> 13 가지의 다른 shorthand가 있다.

소수의 사람들만이 이를 이해하고 사용하고, 나머지는 더 복잡해진다. Powershell은 다른 명령 사이에서 모호성이 없는 경우에만 단축어를 사용한다. `/noprofile`의 가장 짧은 단축은 `/no`는 `/noexit`와 충돌 때문에 `/nop`이다. Powershell의 경우 앞의 두 문자로 축약할 수 있으므로 `/ec`도 가능하다.

# 난독화된 command line이 있는 프로그램 찾기

위에서 설명한대로 command line 인자를 핸들링하는데 표준이 없다. 일부 프로그램은 엄격하지만 어떤 프로그램은 유연하다. 특정한 프로그램에서 command line이 어떻게 동작하는지 알아내는 것도 문제다. 

이를 알아내는 가장 확실한 방법은 리버싱이지만 프로그램의 복잡함을 생각하면 시간이 많이 든다. 좀 더 효과적인 방법은 모든 우회 기법을 반복적으로 시도하고 결과를 확인하는 것이다.

이 접근법의 PoC 구현을 통해 위의 다섯 가지 기법 중 어느 것이 적용되는지 분석할 수 있다. PoC의 자세한 설명은 [github](https://github.com/wietze/windows-command-line-obfuscation)에서 확인할 수 있다.

# 탐지 문제 (Problems for detection)

특정 command line 인자가 있는 프로그램 실행을 탐지하려면 synonymous command-line 인자가 문제다. 예시로 `urlcache`와 `f` 인자를 사용하는 `certutil` 실행 탐지 Sigma 룰을 작성하면 다음과 같다.

```
detection:
    selection:
        Image: "*\\certutil.exe"
        CommandLine|contains|all:
            - "/urlcache"
            - "/f"
    condition: selection
```

문제가 발생하는데, 해커가 `urlcache`, `/urᴸcache`, `/₞urlcache`, `/url"cach"e`, 파멸적으로 `₞u₞rᴸ"c₞a₞ch"e` 등을 사용할 경우 command는 동작하지만 행위를 탐지할 수 없다. 규칙에 모든 변형을 추가하는 것도 불가능한데, 다양한 우회 기법을 조합하면 수백만 개가 발생할 수 있기 때문이다.

![](windows_cmd_obfuscation/Untitled%205.png)

> `certutil -f -urlcache -split [https://wietze.github.io/robots.txt](https://wietze.github.io/robots.txt) output.txt`의 난독화 버전

시도할 수 있는 한 가지 방법은 보편적인 규칙을 만드는 것이다. option char를 제거하여 문제를 해결할 수 있다.

```
detection:
    selection:
        Image: "*\\certutil.exe"
        CommandLine|contains|all:
            - "urlcache"
            - "f"
    condition: selection
```

하지만 완벽한 해결책은 아닌데, `f`만 찾는 거는 오탐률이 많이 발생할 확률이 있다.

두 번째 방법은 matching option을 사용하는 것이다. 프로세스 실행을 검색하는 기능이 있는 많은 EDR 도구는 위의 예시와 같이 단순한 문자열 포함 기능을 사용한다. 정규식을 사용하는 경우는 드물다. Sigma가 regex를 지원하지만 30개 중 7개만 구현한다. 제조 업체가 처음부터 이 옵션을 제공하지 않는 이유는 성능 때문이다.

하지만 정규식을 지원하는 툴의 경우 option char 문제를 해결할 수 있다.

```
detection:
    selection:
        Image: "*\\certutil.exe"
        CommandLine|re: "^(?=.*[-/\\]urlcache)(?=.*[-/\\]f).+.*"
    condition: selection
```

정규식은 또 shorthand를 탐지하는 데 도움이 되지만, character instertion과 substituion은 불가능하다.

# Robust and resilient detctions

규칙을 합리적으로 넓게 정의하여 모든 변형을 탐지하고 오탐률을 고려해야 한다. PoC를 사용하여 규칙이 허용할 수 있는 부분을 찾을 수 있다.

두번째로 모니터링 툴의 출력을 표준화하는 것도 도움이 된다. 데이터가 분석 플랫폼에서 처리되기 전에 파이프라인에서 처리되면 모든 특수 문자를 ASCII로 변환하고 알파벳이 아닌 모든 값을 제거하는 필드를 추가하면 좋다.

또 데이터 분석을 사용하여 난독화 시도 자체를 탐지할 수 있다. IT 전반에 사용률이 낮은 문자가 포함된 command line을 확인하거나 문자 밀도, ASCII가 아닌 문자가 포함된 인자 탐지 등이 도움이 된다.