---
title: "[Research] Firmware Emulation with FirmAE Part 2 (ko)"
author: newp1ayer48
tags: [Embedded, Emulation, firmware, newp1ayer48]
date: 2025-07-10 21:00:00
categories: [Research]
cc: true
index_img: /2025/07/10/newp1ayer48/emulation2/ko/image001.jpg
---

안녕하세요! Hackyboiz에서 가장 낮은 곳(low-level)을 맡고 있는 `newp1ayer48` 입니다! 🤸🏻‍♂️

오랜만에 Firmware Emulation with FirmAE Part 2로 무사히(?) 돌아왔습니다!

![image001.jpg](image001.jpg)

FirmAE는 펌웨어 에뮬레이팅을 자동화 해주는 편리한 도구입니다! 펌웨어를 에뮬레이팅하고 동적으로 분석하면 취약점을 찾기 더욱 좋죠! 에뮬레이션과 FirmAE 설치는 이전 글인 [Firmware Emulation with FirmAE Part 1](https://hackyboiz.github.io/2025/05/08/newp1ayer48/emulation1/ko/)을 참고해주시면 됩니다!

저번 글에 이어서 이번 글에서는 FirmAE의 구성과 실행, 그리고 IP 문제 발생 시 해결 방법까지 다뤄보도록 하겠습니다. 🙌🏻

## 1. FirmAE 구성

FirmAE의 주요 디렉토리와 파일은 아래처럼 구성되어 있습니다. 🖇️

| 디렉토리 및 파일 | 설명 |
| --- | --- |
| `FirmAE/analyses/` | `run.sh` 실행 모드 중 analyses 모드의 관련된 파일과 로그가 저장되는 곳입니다. |
| `FirmAE/firmwares/` | 분석할 펌웨어(`.bin`)를 위치 시키는 곳입니다. |
| `FirmAE/images/` | 에뮬레이션을 위한 OS 이미지가 존재하는 곳입니다. |
| `FirmAE/scratch/` | 각 에뮬레이션 실행 시, IID(Image ID)로 하위 디렉토리가 생성되며, 그 안에는 **추출된 파일 시스템, 커널, 생성된 디스크 이미지, 로그 등 해당 실행에 대한 모든 중간 산출물과 결과가 저장**됩니다. 하위 디렉토리는 `FirmAE/scratch/{IID}` 형태로 생성됩니다. |
| `FirmAE/scripts/` | 메인 스크립트들이 호출하는 각종 하위 스크립트들이 존재하는 곳입니다. |
| `FirmAE/scripts/makeNetwork.py` | 펌웨어의 네트워크 동작을 동적으로 분석하고, 관련 네트워크 설정을 최종 QEMU 실행 스크립트 `FirmAE/scratch/{IID}/run.sh`을 생성하고 전달하는 역할을 합니다. |
| `FirmAE/sources/scraper/firmware/spiders/` | `run.sh` 실행에 필요한 `<brand>` 인자에 해당되는 파일들이 존재하는 디렉토리 입니다. 만약 해당 이름이 없거나 자동으로 브랜드 이름을 추출하는 경우 `scripts/util.py`에서 `get_brand()`를 통해 진행됩니다. |
| `FirmAE/run.sh` | **메인 실행 스크립트**로, 기본적으로 5가지 모드를 지원합니다. |

`FirmAE/run.sh` 의 5가지 모드는 아래와 같습니다.

| `-c`(check) | 네트워크/웹 접속 가능 여부를 확인하는 모드입니다. **에뮬레이팅에 앞서 check 모드로 확인**하는 것이 일반적입니다. |
| --- | --- |
| `-r`(run) | 에뮬레이션을 실행하는 모드입니다. |
| `-d`(debug) | 디버깅 모드로 펌웨어를 에뮬레이션 할 수 있습니다.  |
| `-b`(boot debug mode) | kernel 수준에서 부팅 단계부터 디버깅을 할 수 있는 모드입니다. |
| `-a`(analyze) | 취약점 분석 모드입니다. 웹 서비스를 목표로 사전에 정의된 공격들을 진행하고 발생 가능한 취약점을 찾습니다. 분석 결과는 `analyses/`와 `scratch/{IID}/result`에서 확인할 수 있습니다. |

![image002.jpg](image002.jpg)

이외에도 파일이 정말 많지만… 😮 전부 보기에는 한계가 있으니, 중요하다고 판단되는 것만 소개했습니다. 이중에서 자주 보는 중요한 파일과 폴더는 아래로 정리할 수 있습니다.

- `scripts/makeNetwork.py`
- `run.sh`
- `scratch/`

다른 파일들도 영향을 미치지만, 크게 `scripts/makeNetwork.py`의 설정 값을 참고하여 `run.sh`이 실행되고 `scratch/`에 해당 에뮬레이션 이미지와 파일들이 생성되는 흐름으로 FirmAE가 동작한다고 할 수 있습니다. 에뮬레이션의 과정에서 발생하는 **모든 로그는 `scratch/` 내에 존재**하니 로그 정보도 확인하면 실행 이후에 발생하는 문제도 확인이 가능합니다! 🔍

## 2. FirmAE 실행 및 FirmAE Debugger

`-c` Check 옵션으로 네트워크 및 에뮬레이션 확인이 완료되었다면, 이제 에뮬레이션을 실행할 차례입니다.

에뮬레이션은 필요와 용도에 따라 옵션을 부여하여 사용하면 됩니다. 잘 모르겠다면 `-d` 옵션으로 debug 모드로 디버깅을 수행하는 것이 좋습니다.

![image003.png](image003.png)

에뮬레이션이 잘 동작하면 위처럼 FirmAE Debugger 화면을 확인할 수 있습니다! 🎉 이제 에뮬레이션이 동작하는 상태에서 FirmAE Debugger의 기능들을 살펴보겠습니다. 기능은 크게 5가지가 있습니다.

- connect to socat: python socat을 통해 접속을 제공하는 기능입니다. 1번 기능을 사용하지 않더라도, netcat 연결이 잘 되었다면 생성된 주소와 포트로 가상 기기 웹 페이지를 접속할 수 있습니다! 🌐

- connect to shell: shell로 에뮬레이션 된 가상 기기에 접속하여 제어할 수 있도록 하는 기능입니다!
busybox 등으로 해당 펌웨어에 정의된 linux 명령어를 사용할 수 있습니다. 이를 통해 동적으로 기기 내에서 분석과 테스트를 수행할 수 있습니다. 🧪

![image004.png](image004.png)
    

- tcpdump: 생성한 가상 네트워크 인터페이스를 tcpdump로 분석할 수 있습니다. ✉️ 기입된 명령어는 따로 수정이 안되니 생성된 인터페이스를 대상으로만 분석이 가능합니다.

![image005.png](image005.png)

- run gdbserver: 가상 기기 내부에 gdb server 열어 동적 분석을 제공하는 기능입니다.
    
    이 기능을 사용하기 위해서는 준비가 필요합니다. 임베디드 기기 펌웨어는 ARM, MIPS 등의 RISC 아키텍처를 사용하는 경우가 대부분이기 때문에, 일반 gdb로는 송수신되는 데이터를 인식하지 못하는 경우가 많습니다. ⚠️ 그렇기 때문에 gdb-multiarch를 통해서 gdb server에 접속해야 합니다.
    
    ```bash
    sudo apt-get install gdb-multiarch
    gdb-multiarch
    set sysroot
    set follow-fork-mode child
    set solib-search-path PATH
    target remote IP:PORT
    ```
    
    설치 후에는 위처럼 gdb-multiarch 실행 후 몇 가지 설정을 진행해줍니다. 사전에 binwalk를 통해 해제한 펌웨어 lib 주소를 path로 등록하면 됩니다. 📖
    
    예제 펌웨어 프로세스 pid를 대상으로 설정 된 gdb server에 접속하면 접속이 되는 것을 확인 할 수 있습니다. httpd 같은 웹 서비스 프로세스에 적용하면 해당 웹 페이지 기능을 대상으로 디버깅이 가능합니다.
    
![image006.png](image006.png)
    

- file transfer: 로컬에 있는 파일을 가상 기기로 전송할 수 있습니다.
    
    전송된 파일이 저장되는 경로는 펌웨어 내에 생성된 `firmadyne/` 내에 위치합니다. 그 이유는 `FirmAE/debug.py`에서 `file_transfer()`에 정의되어 있습니다. 가상 기기에 파일을 전송할 수 있기 때문에, 리버스 쉘 같은 프로그램을 밀어 넣거나, `xinetd.conf`, `sshd_config` 같은 설정 파일을 전송할 수도 있습니다. ✈️ 필요에 따라서는 익스플로잇 코드와 관련 파일을 전송하여 테스트 할 수도 있습니다.
    
    ![image007.png](image007.png)
    

## 3. IP 및 네트워크 문제 발생 시 해결 방법

FirmAE는 기기 펌웨어에 설정된 IP 주소를 기반으로 네트워크를 설정하고 에뮬레이션을 구동합니다. 그러나 기기 펌웨어에 따라서 IP가 설정이 되어 있지 않거나, localhost로만 접속하는 등의 여러 가지 이유로 FirmAE 에뮬레이션이 동작하지 않는 경우가 종종 있습니다. 😭

![image008.jpg](image008.jpg)

이 문제는 여러가지 원인과 해결 방법이 존재합니다. 그중 `scripts/makeNetwork.py` 를 수정하는 방법으로 해결하는 방법에 대해 소개하겠습니다.

FirmAE에서는 기본 네트워크 설정(브리지 모드)에 실패했을 때, 에뮬레이션을 계속 진행하기 위해 대체 네트워크 방식(QEMU 사용자 모드)으로 자동 전환하는 기능이 있습니다. 🚸 자동 전환이 되면, **localhost로 접속 환경이 구성**이 됩니다. 아래 사진의 펌웨어도 같은 경우이기 때문에, localhost로 접속 환경이 구성되는 것을 확인 할 수 있습니다.

![image009.png](image009.png)

이것은 `scratch/{IID}/run.sh` 내에 `QEMU_NET_OPTS` 변수를 통해 QEMU 에뮬레이션에 들어가는 네트워크 브리지 설정을 조정하는 곳에서 벌어집니다.

앞서 FirmAE 구성에 대해 설명할 때, `scratch/{IID}` 폴더와 파일들은 `scripts/makeNetwork.py`에서 생성하고, 네트워크 설정을 한다고 했습니다. 그렇기 때문에 이 파이썬 스크립트를 수정하면 에뮬레이션 시, 무조건 localhost로 열어버릴 수 있습니다. 🔓

- `scripts/makeNetwork.py` 수정
    
    ```bash
    cp scripts/makeNetwork.py scripts/makeNetwork.py.bak
    ```
    
    먼저 원본 파일을 백업해둡니다.
    
    ```python
    isUserNetwork = any(isDhcpIp(ip) for ip in ips)
    with open(SCRATCHDIR + "/" + str(iid) + "/isDhcp", "w") as out:
    ```
    
    ```python
    # isUserNetwork = any(isDhcpIp(ip) for ip in ips)
    isUserNetwork = True
    with open(SCRATCHDIR + "/" + str(iid) + "/isDhcp", "w") as out:
    ```
    
    IP 설정 부분을 첫 번째 코드를 두 번째처럼 수정해줍니다.
    
    ```python
    portfwd = "hostfwd=tcp::80-:80,hostfwd=tcp::443-:443,"
    for (proto, ip, port) in ports:
    ```
    
    ```python
    portfwd = "hostfwd=tcp::8888-:80,"
    # portfwd = "hostfwd=tcp::80-:80,hostfwd=tcp::443-:443,"
    
    for (proto, ip, port) in ports:
    ```
    
    port 설정 부분도 첫 번째 코드를 두 번째처럼 수정해줍니다.
    

![image010.png](image010.png)

파일 수정 후 실행을 하면 IP가 localhost로 변경되는 것을 확인할 수 있습니다! 👍🏻

이 방법은 단순히 설정되는 네트워크 만을 변경하는 것이기 때문에 근본적인 해결 방법이 아닐 수도 있습니다. 계속 동작하지 않으면 직접 펌웨어를 분석하여 정확히 네트워크 설정을 맞춰 주는 것이 좋습니다. 앞서 설명 드린 것처럼 `scratch/{IID}/`에 에뮬레이팅 진행 중에 발생한 모든 로그가 존재하기에, 로그를 분석하면서도 문제를 해결하는 것도 좋은 방법입니다.

QEMU를 통해 직접 에뮬레이션을 하는 것도 추천 드립니다! 사용해보신 분들은 알겠지만 FirmAE는 정말 느리기 때문이죠…

![image011.png](image011.png)

FirmAE는 느리긴 하지만, 잘만 사용하면 분명히 유용한 도구입니다! 이 글로 여러분의 펌웨어 에뮬레이팅에 조금이나마 도움이 되었으면 좋겠습니다! 🙏🏻

다음에는 다른 임베디드 주제로 돌아오도록 하겠습니다! 감사합니다! 👋🏻