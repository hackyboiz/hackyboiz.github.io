---
title: "[Translation] Siloscape: First Known Malware Targeting Windows Containers to Compromise Cloud Environments"
author: idioth
tags: [idioth, cloud, containers, kubernetes, malware, siloscape]
categories: [Translation]
date: 2021-06-20 18:00:00
cc: false
index_img: /2021/06/20/idioth/siloscape/Untitled.png
---

안녕하세요. idioth입니다. 원래 오늘은 ghidra의 다음 파트가 나오는 날이죠. 하지만 여러 가지 사정으로 인해 번역글을 작성하게 되었어요.



아직 저는 기말 시험이 끝나지 않았고 또 사무실에 있는 노트북에 분석한 VM이 잠들어 있는데 어제 발에 레이저 치료를 받은 탓에 잘 걷지 못하고 있답니다 하하하. 생각보다 발 뒤꿈치로 걷는 거 할만해요 ^^....



아무튼 이런저런 이유로 기말 준비 중인 저에게 싸장님께서 읽으라고 추천해주신 글을 읽을 겸 번역해왔습니다! 후반부에 뺀 부분이 조금 있습니다! Siloscape의 행위를 보고 싶어서 IRC 접속 부분은 제외하고 번역을 했습니다. :)

> 원문 글 : [Siloscape: First Known Malware Targeting Windows Containers to Compromise Cloud Environments](https://unit42.paloaltonetworks.com/siloscape/)



# Executive Summary

2021년 3월에 Windows container를 타깃으로 하는 최초의 악성코드를 발견했다. 지난 몇 년간 클라우드를 타깃으로 한 악성코드가 증가한 것을 생각하면 놀랍지는 않다. 이 악성코드는 container escape가 주목적이고 주로 server silo를 통해 Windows에서 구현되므로 Siloscape (silo escape처럼 들림 ㅎ)라고 이름 지었다.



Siloscape는 Windows container를 통해 Kubernetes clusters를 타깃으로 하는 악성코드이다. 주목적은 악성 컨테이너를 실행하기 위해 취약한 Kubernetes clusters에 백도어를 설치하는 것이다.



각 컨테이너는 하나의 클라우드 애플리케이션을 실행하는 반면에 하나의 클러스터는 여러 개의 클라우드 애플리케이션을 실행할 수 있으므로 개별 컨테이너를 손상시키는 것보다 전체 클러스터를 손상시키는 것이 더 치명적이다. 예시로 해커가 사용자 이름 및 암호, 조직의 기밀 정보와 내부 파일 또는 클러스텡 호스팅 된 전체 데이터베이스와 같은 중요 정보를 훔칠 수 있다. 해커가 랜섬웨어를 통해 조직의 파일을 인질로 잡을 수도 있다. 더 심각한 것은 많은 조직들이 클라우드로 이전하면서 개발, 테스트 환경으로 Kubernetes clusters를 많이 사용하고 있다. 이러한 점에서 광범위한 소프트웨어 공급망 공격이 발생할 수도 있다.



Siloscape는 C2 (Command and Cotrol) 서버에 익명으로 연결하기 위해 토르 프록시와 `.onion` 도메인을 사용한다. 23개(명?, 기관인지 사람인지 애매하네요)의 피해자를 찾았고 총 313명의 사용자를 호스팅 하는데 서버가 사용되고 있음을 발견했다. 이는 Siloscape가 캠페인의 빙산의 일각이라는 부분을 알려준다. 필자는 또한 이 캠페인이 1년 이상 지속되어 왔음을 발견했다.



이 리포트는 Windows container 취약점의 배경 지식, Siloscape의 기술 개요, Windows container 보안 권장 사항을 제공한다.



# Windows Server Container 취약점 개요

2020년 7월 15에 [Windows container boundaries에 관한 게시글](https://unit42.paloaltonetworks.com/windows-server-containers-vulnerabilities/)과 어떻게 우회할 수 있는지 작성했다. 이 게시글에서 컨테이너를 escape 하는 기술을 제공하고 해커가 escape 할 수 있는 위험이 있는 애플리케이션에 대해 이야기했다. 가장 중요한 애플리케이션은 Kubernetes의 Windows container node를 탈출하여 클러스터를 확산시키는 것이었다.



마이크로소프트에서 Windows Server container는 보안 기능이 아니라는 이유로 이 이슈를 취약점으로 생각하지 않았다. 따라서 컨테이너 내부에서 실행되는 각각의 애플리케이션은 호스트에서 직접 실행되는 것처럼 처리되어야 한다.



몇 주 후, Kubernetes가 이러한 이슈에 취약하다고 구글에 전달했다. 구글은 마이크로소프트에 전달하였고 마이크로소프트는 이 이슈를 Windows container에서 호스트로 escape 하는 것으로 결정하였지만 컨테이너에서 관리자 권한 없이 실행되면 취약점으로 간주된다.



그 후에 Windows Server container를 타깃으로 한 악성코드 Siloscape를 발견하였다. Siloscape는 Hyper-V가 아닌 Server Container를 사용하는 Kubernetes을 타깃으로 하는 난독화된 악성코드이며 주목적은 설정이 미흡한 Kubernetes 클러스터에 백도어를 열어 암호 화폐 채굴과 같은 악성 container를 실행하는 것이다.



악성코드의 동작과 기술에 대한 특징은 다음과 같다:

- 알려진 취약점 (1-day)를 사용하여 타깃 클라우드 애플리케이션에 접근
- Windows container escape를 통해 컨테이너를 escape 하고 underlying node에서 코드를 실행
- node의 자격 증명을 통해 클러스터에 전파를 시도
- 토르 네트워크를 통해 IRC 프로토콜을 사용하여 C2 서버에 접속
- 추가 command 대기
- 

이 악성코드는 Kubernetes cluster의 컴퓨팅 리소스를 사용하여 암호화폐 채굴을 수행하고 손상된 클러스터에서 실행되는 수백 개의 애플리케이션의 중요 데이터를 가져올 수 있다.



C2 서버를 조사한 결과 이 악성코드는 큰 네트워크의 일부분이며 이 캠페인은 1년 이상 진행됐다. 게다가 캠페인의 특정 부분이 아직 활성화되어있다.



# 기술 개요

Siloscape의 자세한 기술을 살펴보기 전에 전반적인 행위를 살펴보자.

![](siloscape/Untitled.png)

1. 해커는 1-day 취약점이나 웹페이지, 데이터베이스 취약점을 사용하여 Windows container 내부에 원격 코드 실행 (Remote Code Execution, RCE)을 수행
2. Siloscape (`CloudMalware.exe`)를 C2 접속 정보와 함께 command line으로 실행
3. Siloscape는 `SeTcbPrivilege` 권한을 얻기 위해 `CExecSvc.exe`를 impersonate ([해당 게시글](https://unit42.paloaltonetworks.com/windows-server-containers-vulnerabilities/)에서 기술 상세 내용 확인 가능)
4. 호스트에 global symbolic link를 생성하고 컨테이너화 된 X 드라이브를 호스트의 C 드라이브에 연결
5. global link를 사용하여 호스트에서 이름으로 `kubectl.exe`를 찾고 정규식으로 Kubernetes 설정 파일을 찾음
6. 손상된 노드가 새로운 Kubernetes deployments를 생성할 수 있는 권한이 충분한지 확인
7. unzip을 사용해 압축 파일에서 토르 클라이언트를 디스크로 추출 (Siloscape의 main binary에 파일이 압축되어 있음)
8. 토르 네트워크에 연결
9. command line argument를 통해 C2 서버의 비밀번호를 암호화
10. `.onion` 도메인을 사용하여 C2 서버에 연결
11. C2 서버에서 오는 명령을 기다리고, 명령이 올 경우 실행

컨테이너를 타깃으로 한 다른 암호화폐 채굴이 주 목적인 악성코드들과 다르게 Siloscape는 클러스터를 손상시키지 않는다. 대신에 탐지 및 추적을 방지하고 클러스터에 백도어를 설치하는 데에 집중했다.

# Defense 우회와 난독화

Siloscape는 심하게 난독화되어 있으며 대부분의 문자열을 읽을 수 없다. 난독화 로직은 어렵지 않지만 리버싱 하기 너무 힘들다.



![](siloscape/Untitled%201.png)

단순한 API 호출도 난독화 되어있으며 함수를 단순하게 호출하지 않고 Native API (NTAPI)를 사용하였다.



![](siloscape/Untitled%202.png)

예시로 `CreateFile`을 호출하는 대신에 `NtCreateFile`을 호출한다. `NtCreateFile`을 직접 호출하는 대신에 런타임에서 `ntdll.dll`에서 함수 이름을 찾아 주소로 점프하여 호출한다. 그뿐만 아니라 함수와 모듈 이름을 난독화하고 런타임에서 복호화한다. 정적 분석으로는 분석하기 어렵고 리버스 엔지니어링 하는데 힘든 악성코드이다.



Siloscape는 한 쌍의 key를 사용하여 C2 서버의 비밀번호를 복호화한다. 난독화 중에서 가장 중요한 특징은 키 하나는 바이너리에 하드 코딩되어 있는데, 하나는 command line argument에서 제공된다는 점이다. AutoFocus와 VirusTotal 같은 여러 엔진에서 hash를 검색해도 결과를 찾을 수 없었다. 이를 통해 Siloscape는 새로운 공격을 할 때 고유한 키를 사용해서 컴파일을 하는 것을 유추할 수 있다. 하드 코딩된 키는 바이너리들을 살짝씩 다르게 만들어서 hash를 통해 찾을 수 없음을 설명한다. 또 hash만으로 Siloscape를 탐지할 수 없다.



![](siloscape/Untitled%203.png)

다른 흥미로운 특징은 Visual Studio의 Resource Manager를 사용한 것이다. Visual Studio에 내장된 기능으로 어떤 파일이든 원본 바이너리에 추가할 수 있고 간단한 API 호출을 통해 포인터를 얻을 수 있다. Siloscape는 디스크에 토르 아카이브와 아카이브를 열기 위해 사용되는 unzip 바이너리를 이를 통해 추가했다. 또한 토르를 통해 C2 서버에 안전하게 접속한다.



# The Container Escape

Siloscape의 흥미로운 점 중 하나는 컨테이너를 escape 하는 것이다. escape를 하기 위해 system call `NtSetInformationSymbolicLink`를 실행하려면 `SeTcbPrivilege` 권한을 얻어야 한다. 여기에는 몇 가지 방법이 있다. 예를 들어 권한을 가진 `CExecSvc.exe`에 DLL을 inject 하고 그 context에서 `NtSetInformationSymbolicLink`를 실행할 수 있다. 하지만 Siloscape는 기술 문서나 예시가 거의 없는 Thread Impersonation을 사용하였다. 이 기술의 주요한 함수는 문서화되지 않은 system call `NtImpersonateThread`이다.



Siloscape는 main thread를 impersonating 하여 `CExecSvc.exe`의 권한을 모방하고 컨테이너를 깨기 위해 새로 생성된 symbolic link에서 `NtSetInformationSymbolicLink`를 호출한다. 구체적으로 컨테이너화 된 X 드라이브를 호스트의 C 드라이브에 연결한다.



# 클러스터 선택

호스트에 link를 생성한 뒤에 Kubernetes 노드에 존재하는 `kubectl.exe`와 Kubernetes 설정 파일을 찾는다.

![](siloscape/Untitled%204.png)

이름을 통해 `kubectl.exe`를 찾고 정규식을 사용해 Kubernetes 설정 파일을 찾는다. 검색하는 함수는 검색을 안 할 폴더 이름이 포함된 벡터의 포인터를 추가적인 인수로 사용한다.

![](siloscape/Untitled%205.png)

파일 검색을 위해 `FindFile`을 호출할 때 검색 속도를 빠르게 하기 위해 `Program Files`, `Program Files (x86)`, `Windows`, `Users` 폴더를 제외한다. (해당 폴더에 파일이 존재하지 않을 가능성이 높으므로) 두 파일을 모두 찾으면 전역 변수에 경로가 저장된다. 파일이 없으면 공격을 멈추고 siloscape가 종료된다.

`kubectl`을 실행하는데 필요한 걸 모두 찾은 다음 `%ls auth can-i create deployments --kubeconfig=%ls` 명령어를 통해 손상된 노드가 악성 행위를 할 수 있는 권한이 있는지 확인한다. 

# C2 연결 및 지원 명령어

손상된 노드가 새로운 deployments를 생성할 수 있으면 호스트의 C 드라이브에 토르 아카이브와 unzip 바이너리를 추출한다. 토르를 추출한 후 `tor.exe`를 새 스레드로 시작하고 토르 스레드의 output을 확인할 때까지 대기한다.

토르가 실행되면 `.onion` 도메인을 통해 IRC 서버인 C2에 연결한다.

서버는 비밀번호로 보호되고 있고 command line argument와 바이트 XOR을 통해 복호화한다. C2 서버의 복호화 로직을 간단하게 표현하면 다음과 같다.

```
char hardCodedXor[32] = "HARD_CODED_32_LONG_STRING";
char ircPass[32] = { 0 };
for (int i = 0; i < 32; i++)
     ircPass[i] = hardCodedXor[i] ^ argv[1][i];
```

IRC 서버에 연결이 성공하면 `JOIN #WindowsKubernetes` 명령어를 사용하여 `WindowsKubernetes` IRC 채널에 들어간 후 대기한다.

Siloscape는 kubectl 지원 명령과 Windows `cmd` 명령 두 가지 유형의 명령이 있다.

![](siloscape/Untitled%206.png)

명령을 `admin` 사용자가 받으면 다음 로직을 따른다.

- 메시지가 `K`로 시작하면 `%ls %s --kubeconfig=%ls` 명령을 실행하여 위에서 찾은 경로로 클러스터에 kubectl 명령 실행
    - 첫 번째 파라미터 : 전역 변수의 kubectl의 경로
    - 두 번째 파라미터 : 첫번째 문자를 제외한 admin으로의 메시지
    - 세 번째 파라미터 : 전역 변수의 설정 파일 경로
- 메시지가 `C`로 시작하면 첫 번째 문자를 빼고 Windows cmd 명령을 실행

# 결론

리소스 하이재킹이나 DoS를 목적으로 한 대부분의 클라우드 악성코드와 달리 Siloscape는 하나의 목적에만 국한되지 않고 다양한 악성 행위를 수행하기 위해 백도어를 설치한다.

[필자의 지난 게시글](https://unit42.paloaltonetworks.com/windows-server-containers-vulnerabilities/)에서 얘기했듯 마이크로소프트는 보안 기능으로 Windows container를 사용하지 말라고 했다. 마이크로소프트는 security boundary를 컨테이너화에 의존하지 말고 Hyper-V 컨테이너를 사용하는 걸 추천한다. Windows Server container에서 실행되는 모든 프로세스는 호스트의 관리자 권한을 가진 것으로 생각해야 하며 이번 경우에는 Kubernetes node였다. 보안이 필요한 Windows Server container에서 애플리케이션을 실행하는 경우 Hyper-V 컨테이너에서 실행하는 게 좋다.

게다가 관리자는 Kubernetes cluster를 안전하게 설정해야 한다. secured Kubernetes node의 권한은 해당 악성코드가 실행될 만큼의 충분한 권한이 없어서 Siloscape에 영향을 받지 않는다.

Siloscape는 container escape를 하지 못하면 피해를 유발하지 않는다는 점에서 컨테이너 보안의 중요성을 보여준다.