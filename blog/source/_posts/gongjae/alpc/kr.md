---
title: "[Research] Windows ALPC (KR)"
author: gongjae
tags: [Windows, ALPC, IPC, gongjae]
categories: [Research]
date: 2025-11-27 19:00:00
cc: true
index_img: /2025/11/27/gongjae/alpc/kr/image01.jpg
---

안녕하세요! 이번에 HackyBoiz로 새롭게 들어온 `gongjae`라고 합니다! 👻

HackyBoiz 견습생시절.. 제가 진행했었던 Kernel Driver와 Named Pipe 버그헌팅 프로젝트에 대한 질문이 하나 있었습니다.

![image00.jpg](image00.jpg)

~~아니요~~

사실 ALPC가 IPC 통신 방법 중 하나라는 것은 알고 있었지만 이번 기회에 제대로 공부해보자는 마음으로 이 주제를 가지고 왔습니다!! 오늘은 ALPC의 개념과 통신 흐름, 메시지는 어떻게 전송되는지 직접 디버깅 해보며 알아보겠습니다 :)

![image01.jpg](image01.jpg)

## 1. About ALPC

Windows 환경에서 원격과 로컬 모두에서 사용할 수 있는 IPC 통신은 Named Pipe, RPC 등이 있습니다. 그러나 ALPC는 특이하게도 로컬에서만 사용 가능한 기술입니다.

RPC가 Remote Procedure Call을 의미하는 것과 달리, ALPC는 Advanced Local Procedure Call을 뜻하며, 때때로는 Asynchronous(비동기) Local Call로도 불린답니다!

비동기라는 명칭은 Windows Vista 시절을 반영해 Vista에서 기존의 LPC를 대체하기 위해 ALPC를 도입한 것이기 때문입니다.

 ALPC는 빠른 메시지 통신이 가능하고 프로세스 간 데이터 송수신에 사용되게 됩니다.

### 1.1. LPC to ALPC

![image02.jpg](image02.jpg)

Windows Vista 이전 LPC는 동일한 컴퓨터의 프로세스간 경량 IPC를 통해 Microsoft Windows NT 커널에서 제공하는 문서화되지 않은 내부 프로세스간 통신 기능입니다.

LPC의 동기적 특성으로 인해 클라이언트와  서버는 메시지가 처리될 때까지 서로 대기해야만 했고, 이 때문에 실행이 지속적으로 Block 되는 문제가 발생합니다.

따라서 Vista 시점부터 기존 LPC 메커니즘은 사실상 ALPC 기반으로 재구현되었고, 이후 버전에서는 내부적으로ALPC를 중심으로 IPC가 동작하게 됩니다.
다만 LPC 관련 API 자체가 “완전히 사라진 것”은 아니고, 내부 구현이 ALPC로 리디렉션된다고 보는 게 더 정확합니다. 

<aside>
💡

아래 사진을 보면 기존에 LPC 포트를 생성하던 함수는 유지되었으며, 해당 함수 호출은 실제로 LPC가 아닌 ALPC 포트를 생성하도록 리디렉션하고 있죠 @.@

![image03.jpg](image03.jpg)

</aside>

### 1.2. ALPC 내부 구조

APLC 통신의 주요 구성 요소는 ALPC 포트 객체이며, 사용법은 네트워크 소캣과 유사합니다.

→ 서버가 클라이언트와 메시지를 교환하기 위해 연결할 수 있는 소캣을 여는 것

Sysinternals Suite의 WinObj.exe를 통해 ALPC 포트들을 확인할 수 있는데요!

![image04.jpg](image04.jpg)

루트 경로에도 ALPC 포트가 있지만, 대부분은 \RPC Control 경로에 존재합니다.

ALPC 통신에는 총 3개의 ALPC port(서버측 2개, 클라이언트측 1개)가 관여하게 되는데, 위 WinObj 사진에서 보인 ALPC 포트들은 ALPC Communication Port로 클라이언트가 연결할 수 있는 포트입니다.

ALPC 통신에 대해 설명하기 전에, 계속해서 보게 될 함수 몇개를 먼저 짚고 넘어가겠습니다.

> **NtAlpcCreatePort()**
> 

```cpp
NTSTATUS NTAPI NtAlpcCreatePort(
    OUT PHANDLE PortHandle,
    IN POBJECT_ATTRIBUTES ObjectAttributes,
    IN OUT PALPC_INFO PortInformation OPTIONAL
);
```

- 이 함수는 ALPC 포트를 생성할 때 사용하며, 포트 생성 방식은 기존 LPC와 크게 다르지 않지만 옵션들을 따로 나열하는 대신 ALPC_INFO 구조체로 묶어서 마지막 인자로 보내는게 특징!
- 이 구조체는 포트 생성 시 ALPC 객체에 복사되어 추 후에 내부에서 참조

```cpp
typedef struct _ALPC_INFO
{
#define PORT_INFO_LPCMODE               0x001000 // LPC 포트처럼 동작
#define PORT_INFO_CANIMPERSONATE        0x010000 // 임퍼소네이션 허용
#define PORT_INFO_REQUEST_ALLOWED       0x020000 // 메시지 허용
#define PORT_INFO_SEMAPHORE             0x040000 // 동기화 시스템
#define PORT_INFO_HANDLE_EXPOSE         0x080000 // 핸들 노출 허용
#define PORT_INFO_PARENT_SYSTEM_PROCESS 0x100000 // 커널 ALPC 인터페이스

    ULONG Flags;
    SECURITY_QUALITY_OF_SERVICE PortQos;
    ULONG MaxMessageSize;
    ULONG unknown1;
    CHAR  cReserved1[8];
    ULONG MaxViewSize;
    CHAR  cReserved2[8];
} ALPC_INFO, *PALPC_INFO;

```

> **NtAlpcConnectPort()**
> 

```cpp
NTSTATUS NTAPI NtAlpcConnectPort(
    OUT PHANDLE PortHandle,
    IN PUNICODE_STRING PortName,
    IN POBJECT_ATTRIBUTES ObjectAttributes,
    IN PALPC_INFO PortInformation OPTIONAL,
    IN DWORD ConnectionFlags,
    IN PSID pSid OPTIONAL,
    IN PLPC_MESSAGE ConnectionMessage OPTIONAL,
    IN OUT PULONG ConnectMessageSize OPTIONAL,
    IN PVOID InMessageBuffer OPTIONAL,
    IN PVOID OutMessageBuffer OPTIONAL,
    IN PLARGE_INTEGER Timeout OPTIONAL
);
```

- PortName으로 연결하려는 포트 이름, 각종 옵션들을 넣을 수 있고 커널이 해당 이름의 ALPC Connection Port 객체를 탐색하고 추후에 연결을 요청

**ConnectionFlags 중요 Flag**

```cpp
#define ALPC_SYNC_CONNECTION   0x020000 // 동기식 연결
#define ALPC_USER_WAIT_MODE    0x100000 // 유저 모드에서 Wait
#define ALPC_WAIT_IS_ALERTABLE 0x200000 // Alertable wait
```

- default 값으로는 비동기 연결(Async)
    - 연결 요청을 서버가 처리해서 실제로 받아들이기 전에, 클라이언트는 핸들을 하나 받아버릴 수 있음
    - 만약 서버가 아직 연결 요청을 처리하지 않은 상태에서 클라이엍느가 메시지를 보내면 에러가 날 수 있음

> **NtAlpcSendWaitReceivePort()**
> 

```cpp
NTSTATUS NtAlpcSendWaitReceivePort(
    HANDLE PortHandle,
    DWORD Flags,
    PPORT_MESSAGE SendMessage,
    PALPC_MESSAGE_ATTRIBUTES SendMessageAttributes,
    PPORT_MESSAGE ReceiveMessage,
    PSIZE_T BufferLength,
    PALPC_MESSAGE_ATTRIBUTES ReceiveMessageAttributes,
    PLARGE_INTEGER Timeout
);
```

- 가장 중요한 함수로, 이 함수 하나로 메시지 보내기, 받기 동시에 가능!!
- SendMessage, ReciveBuffer는 말 그대로 송/수신 메시지 버퍼
- InMessageBuffer, OutMessageBuffer는 메시지와 함께 추가 액션(섹션 매핑, 핸들 전달 등)을 요청/응답하는데 쓰임

## 2. ALPC 통신 흐름

좋아요, 대충 함수들이 어떤 역할을 하는지 알았으니, 제대로된 통신 흐름을 알아볼까요? 앞서 ALPC 통신 시나리오에서 3개의 ALPC 포트를 사용한다고 했었죠! 

1. 서버 프로세스가 생성하는 ALPC connection port
2. 클라이언트가 연결할 때 커널이 새로 만드는 ALPC server communication port
3. ALPC client communication port

이렇게 3가지를 사용하는데요, 이렇게만 보면 무슨 소리인지 이해가 잘 안되죠..? ~~전 그랬습니다ㅠ~~

사실 ALPC 통신 과정에서 논리적으로 3개의 포트가 존재하는 것처럼 보이지만, 실제로는 하나의 ALPC 포트 객체를 기반으로 여러 엔드포인트가 생성되는 구조입니다.

이제 통신 흐름을 순서대로 짚어가며 이해해봅시다!

![[image00.jpg](https://theori.io/blog/chaining-n-days-to-compromise-all-part-2-windows-kernel-lpe-a-k-a-chrome-sandbox-escape)](image05.jpg)

가장 주목할 함수는 역시나 `NtAlpcSendWaitReceivePort()` 함수입니다. 사실상 이 함수가 요청이나 승인을 주고받는 역할을 전부 해주기 때문이죠! 순서대로 흐름을 말하자면 이렇습니다.

1. 서버 프로세스가 `NtAlpcCreatePort()` 를 호출하여 ALPC 포트를 생성합니다.
    - 이름 예시 : “\RPC Control\HackyBoiz”
2. 커널은 ALPC 포트 객체를 생성하고 서버에세 핸들을 반환합니다.
    
    → 이것이 **ALPC Connection Port!**
    
3. 서버는 `NtAlpcSendWaitRecievePort()` 를 호출하여 클라이언트 연결을 기다립니다.
4. 클라이언트는 `NtAlpcConnectPort()` 를 호출합니다.
    - 연결할 서버 포트 이름 : “\RPC Control\HackyBoiz”
    - (선택) 서버에게 보낼 초기 메시지
    - (선택) 서버 SID 지정하여 올바른 서버인지 검증
    - (선택) 메시지 속성(Attribute) 추가
5. 이 요청은 서버에 전달되며, 서버는 `NtAlpcAcceptConnectPort()` 를 호출해 연결을 승인하거나 거절합니다.
    - 마지막 인자가 boolean 으로 true면 수락, false면 거절을 뜻함
6. 연결을 수락 시, 커널은 새로운 ALPC Communication Port를 생성하여 서버와 클라이언트 양쪽에 각각 핸들을 반환합니다.
7. 이후 메시지는 Connection Port가 아닌, 이 Communication Port를 통해 송수신 됩니다.

이제는 조금 이해가 되는군요!! 좋아요 그렇다면 이제 실습을 통해 직접 디버깅해서 확인해봅시다.

https://github.com/DownWithUp/ALPC-Example

위 github에서 예제 ALPC 코드를 확인 할 수 있는데요, 직접 코드를 실행하여 통신 과정을 확인해봅시다!XD

> Server.c
> 

```cpp
#include <Windows.h>
#include <winternl.h>
#include <stdio.h>
#include "ntalpcapi.h"
#pragma comment(lib, "ntdll.lib")

#define MAX_MSG_LEN 0x500

LPVOID AllocMsgMem(SIZE_T Size)
{
    return(HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, Size + sizeof(PORT_MESSAGE)));
}

void CreatePortAndListen(LPCWSTR PortName)
{
    ALPC_PORT_ATTRIBUTES    serverPortAttr;
    OBJECT_ATTRIBUTES       objPort;
    UNICODE_STRING          usPortName;
    PORT_MESSAGE            pmRequest;
    PORT_MESSAGE            pmReceive;
    NTSTATUS                ntRet;
    BOOLEAN                 bBreak;
    HANDLE                  hConnectedPort;
    HANDLE                  hPort;
    SIZE_T                  nLen;
    LPVOID                  lpMem;
    BYTE                    bTemp;
	
    RtlInitUnicodeString(&usPortName, PortName);
    InitializeObjectAttributes(&objPort, &usPortName, 0, 0, 0);
    RtlSecureZeroMemory(&serverPortAttr, sizeof(serverPortAttr));
    serverPortAttr.MaxMessageLength = MAX_MSG_LEN;

    ntRet = NtAlpcCreatePort(&hPort, &objPort, &serverPortAttr);
    printf("[i] NtAlpcCreatePort: 0x%X\n", ntRet);
    if (!ntRet)
    {
        nLen = sizeof(pmReceive);
        ntRet = NtAlpcSendWaitReceivePort(hPort, 0, NULL, NULL, &pmReceive, &nLen, NULL, NULL);
        if (!ntRet)
        {
            RtlSecureZeroMemory(&pmRequest, sizeof(pmRequest));
            pmRequest.MessageId = pmReceive.MessageId;
            pmRequest.u1.s1.DataLength = 0x0;
            pmRequest.u1.s1.TotalLength = pmRequest.u1.s1.DataLength + sizeof(PORT_MESSAGE);
            ntRet = NtAlpcAcceptConnectPort(&hConnectedPort, hPort, 0, NULL, NULL, NULL, &pmRequest, NULL, TRUE); // 0
            printf("[i] NtAlpcAcceptConnectPort: 0x%X\n", ntRet);
            if (!ntRet)
            {
                bBreak = TRUE;
                while (bBreak)
                {	
                    nLen = MAX_MSG_LEN;
                    lpMem = AllocMsgMem(nLen);
                    NtAlpcSendWaitReceivePort(hPort, 0, NULL, NULL, (PPORT_MESSAGE)lpMem, &nLen, NULL, NULL);
                    pmReceive = *(PORT_MESSAGE*)lpMem;
                    if (!strcmp((BYTE*)lpMem + sizeof(PORT_MESSAGE), "exit\n"))
                    {
                        printf("[i] Received 'exit' command\n");
                        HeapFree(GetProcessHeap(), 0, lpMem);
                        ntRet = NtAlpcDisconnectPort(hPort, 0);
                        printf("[i] NtAlpcDisconnectPort: %X\n", ntRet);
                        CloseHandle(hConnectedPort);
                        CloseHandle(hPort);
                        ExitThread(0);
                    }
                    else
                    {
                        printf("[i] Received Data: ");
                        for (int i = 0; i <= pmReceive.u1.s1.DataLength; i++)
                        {
                            bTemp = *(BYTE*)((BYTE*)lpMem + i + sizeof(PORT_MESSAGE));
                            printf("0x%X ", bTemp);
                        }
                        printf("\n");
                        HeapFree(GetProcessHeap(), 0, lpMem);
                    }
                }
            }
        }
    }
    ExitThread(0);
}

void main()
{
    HANDLE hThread;

    printf("[i] ALPC-Example Server\n");
    hThread = CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE)&CreatePortAndListen, L"\\RPC Control\\HackyBoiz", 0, NULL);
    WaitForSingleObject(hThread, INFINITE);
    printf("[!] Shuting down server\n");
    getchar();
    return;
}

```

> Client.c
> 

```cpp
#include <Windows.h>
#include <winternl.h>
#include <stdio.h>
#include "ntalpcapi.h"
#pragma comment(lib, "ntdll.lib")

#define MSG_LEN 0x100

LPVOID CreateMsgMem(PPORT_MESSAGE PortMessage, SIZE_T MessageSize, LPVOID Message)
{
    LPVOID lpMem = HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, MessageSize + sizeof(PORT_MESSAGE));
    memmove(lpMem, PortMessage, sizeof(PORT_MESSAGE));
    memmove((BYTE*)lpMem + sizeof(PORT_MESSAGE), Message, MessageSize);
    return(lpMem);
}

void main()
{
    UNICODE_STRING  usPort;
    PORT_MESSAGE    pmSend;
    PORT_MESSAGE    pmReceive;
    NTSTATUS        ntRet;
    BOOLEAN         bBreak;
    SIZE_T          nLen;
    HANDLE          hPort;
    LPVOID          lpMem; 
    CHAR            szInput[MSG_LEN];

    printf("ALPC-Example Client\n");
    RtlInitUnicodeString(&usPort, L"\\RPC Control\\HackyBoiz");
    RtlSecureZeroMemory(&pmSend, sizeof(pmSend));
    pmSend.u1.s1.DataLength = MSG_LEN;
    pmSend.u1.s1.TotalLength = pmSend.u1.s1.DataLength + sizeof(pmSend);
    lpMem = CreateMsgMem(&pmSend, MSG_LEN, L"Hello HackyBoiz!");
    ntRet = NtAlpcConnectPort(&hPort, &usPort, NULL, NULL, 0, NULL, NULL, NULL, NULL, NULL, NULL);

    printf("[i] NtAlpcConnectPort: 0x%X\n", ntRet);
    if (!ntRet)
    {
        printf("[i] type 'exit' to disconnect from the server\n");
        bBreak = TRUE;
        while (bBreak)
        {
            RtlSecureZeroMemory(&pmSend, sizeof(pmSend));
            RtlSecureZeroMemory(&szInput, sizeof(szInput));
            printf("[.] Enter Message > ");
            fgets(&szInput, MSG_LEN, stdin);
            pmSend.u1.s1.DataLength = strlen(szInput);
            pmSend.u1.s1.TotalLength = pmSend.u1.s1.DataLength + sizeof(PORT_MESSAGE);
            lpMem = CreateMsgMem(&pmSend, pmSend.u1.s1.DataLength, &szInput);
            ntRet = NtAlpcSendWaitReceivePort(hPort, 0, (PPORT_MESSAGE)lpMem, NULL, NULL, NULL, NULL, NULL);
            printf("[i] NtAlpcSendWaitReceivePort: 0x%X\n", ntRet);
            HeapFree(GetProcessHeap(), 0, lpMem);
        }
    }
    getchar();
    return;
}

```

일단 Server.c 코드를 빌드하여 실행시키면 \RPC Control 위치에 Server ALPC Port가 생긴 것을 확인할 수 있죠!

![image06.jpg](image06.jpg)

Windbg로 Server 부분을 디버깅 해보겠습니다. 먼저 NtAlpcCreatePort 함수를 볼까요!

```cpp
NTSTATUS NTAPI NtAlpcCreatePort(
    OUT PHANDLE PortHandle,
    IN POBJECT_ATTRIBUTES ObjectAttributes,
    IN OUT PALPC_INFO PortInformation OPTIONAL
);
```

![image07.jpg](image07.jpg)

bp를 건 뒤, rdx 부분이 ObjectAttributes 부분입니다. 여기서 Attributes에 대해서는 조금 이따가 더 자세하게 알아보도록 하고, 이 안에 ObjectName이 존재할 겁니다. POBJECT_ATTRIBUTES 구조체는 다음과 같습니다.

```cpp
typedef struct _OBJECT_ATTRIBUTES {
    ULONG           Length;
    HANDLE          RootDirectory;
    PUNICODE_STRING ObjectName;   // ← 이거
    ULONG           Attributes;
    PVOID           SecurityDescriptor;
    PVOID           SecurityQualityOfService;
} OBJECT_ATTRIBUTES, *POBJECT_ATTRIBUTES;
```

그렇다면 rdx + 0x10 부분이 ObjectName 이겠죠? 이를 확인해보면

![image08.jpg](image08.jpg)

이렇게 \RPC Control\HackyBoiz 이름으로 Port가 생성되는 것을 확인할 수 있습니다!

이제 Windbg로 Client 부분을 디버깅하여 통신을 확인해보겠습니다.

![image09.jpg](image09.jpg)

일단 Client를 실행하면 Server에서 Accept를 해준 뒤, Client에서 메시지를 보낼 수 있게됩니다. 이 부분을 좀 더 자세하게 확인해볼게요!

![image10.jpg](image10.jpg)

일단 처음에 연결하는 부분인 NtAlpcConnectPort를 확인해보면 

```cpp
NTSTATUS NTAPI NtAlpcConnectPort(
    OUT PHANDLE PortHandle,
    IN PUNICODE_STRING PortName, // <- 이곳
    IN POBJECT_ATTRIBUTES ObjectAttributes,
    IN PALPC_INFO PortInformation OPTIONAL,
    IN DWORD ConnectionFlags,
    IN PSID pSid OPTIONAL,
    IN PLPC_MESSAGE ConnectionMessage OPTIONAL,
    IN OUT PULONG ConnectMessageSize OPTIONAL,
    IN PVOID InMessageBuffer OPTIONAL,
    IN PVOID OutMessageBuffer OPTIONAL,
    IN PLARGE_INTEGER Timeout OPTIONAL
);
```

rdx에 PortName이 들어가죠, 이를 출력해보면 제가 Server로 올려놓은 HakcyBoiz port가 뜨는 걸 확인할 수 있습니다!

이렇게 Connect 요청을 보내면 Server에서 Accept 할지 말지 결정하게 되겠죠?

일단 Server.c 코드에서의 AlpcAcceptConnectPort 함수를 보면

```cpp
ntRet = NtAlpcAcceptConnectPort(
    &hConnectedPort,   // OUT PHANDLE PortHandle  → rcx
    hPort, 
    0,
    NULL,   
    NULL,   
    NULL,   
    &pmRequest,     
    NULL,           
    TRUE              
);
```

이런식으로 인자가 세팅됩니다.

- rcx의 경우 서버가 새로 채워줄 통신 포트 핸들이므로 syscall이 진행된 이후에 확인해보면 hConnectedPort에 실제 ALPC 서버 communication port 핸들이 들어가게 됩니다.

이후에 마지막 인자인 AcceptConnection 부분이 TRUE로 되어 있으므로 Accpet 되어 서로 통신이 되어 메시지를 보낼 수 있게 됩니다!

## 3. ALPC Messaging

이제 ALPC가 어떻게 통신하는지 알아봤으니, 실제로 왔다 갔다 하는 메시지 포맷을 까봐야겠죠.

ALPC에서 메시지는 항상 다음과 같은 형태를 가지는데요!

![image11.jpg](image11.jpg)

```cpp
[ PORT_MESSAGE ][ Payload(Data) ]
```

앞부분은 고정된 헤더 PORT_MESSAGE, 그 뒤는 저희가 실제로 보내고 싶은 text/binary 데이터 입니다.

### 3.1. PORT_MESSAGE 구조

헤더 구조체는 다음과 같습니다.

```cpp
typedef struct _PORT_MESSAGE
{
    union {
        struct {
            USHORT DataLength;   // 실제 Payload 길이
            USHORT TotalLength;  // PORT_MESSAGE + Payload 전체 크기
        } s1;
        ULONG Length;
    } u1;

    union {
        struct {
            USHORT Type;             // 메시지 타입
            USHORT DataInfoOffset;   // Attribute 영역 오프셋
        } s2;
        ULONG ZeroInit;
    } u2;

    union {
        CLIENT_ID ClientId;          // 송신 프로세스/스레드 ID
        double    DoNotUseThisField; // (정렬용)
    };

    ULONG MessageId;                 // 메시지 식별자

    union {
        SIZE_T ClientViewSize;       // 섹션 뷰 크기
        ULONG  CallbackId;           // 콜백 ID
    };

} PORT_MESSAGE, *PPORT_MESSAGE;
```

실제로 우리가 자주 만지는 필드는 거의 네 개 정도라고 보면 됩니다.

- u1.s1.DataLength
    
    → payload 길이
    
- u1.s1.TotalLength
    
    → sizeof(PORT_MESSAGE) + DataLength
    
    → 커널이 “메시지 전체 크기”를 알기 위해 사용
    
- ClientId
    
    → 이 메시지를 보낸 프로세스/스레드의 ID
    
- MessageId / CallbackId
    
    → 요청/응답 매칭할 때 쓰는 ID
    

즉, 포맷 자체는 되게 심플합니다.

제가 사용한 예제 Client 코드를 다시 보면:

```cpp
#define MSG_LEN 0x100

LPVOID CreateMsgMem(PPORT_MESSAGE PortMessage, SIZE_T MessageSize, LPVOID Message)

    LPVOID lpMem = HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, MessageSize + sizeof(PORT_MESSAGE));
    memmove(lpMem, PortMessage, sizeof(PORT_MESSAGE));
    memmove((BYTE*)lpMem + sizeof(PORT_MESSAGE), Message, MessageSize);
    return(lpMem);
}
```

여기서 `lpMem`이 바로

```cpp
[ PORT_MESSAGE ][ Payload ]
```

형태의 완성된 하나의 ALPC 메시지 버퍼입니다.

main() 쪽에서는 이렇게 쓰죠.

```cpp
RtlSecureZeroMemory(&pmSend, sizeof(pmSend));

printf("[.] Enter Message > ");
fgets(&szInput, MSG_LEN, stdin);

pmSend.u1.s1.DataLength  = strlen(szInput);
pmSend.u1.s1.TotalLength = pmSend.u1.s1.DataLength + sizeof(PORT_MESSAGE);

lpMem = CreateMsgMem(&pmSend, pmSend.u1.s1.DataLength, &szInput);

ntRet = NtAlpcSendWaitReceivePort(
    hPort,                // HANDLE PortHandle
    0,                    // ULONG Flags
    (PPORT_MESSAGE)lpMem, // SendMessage
    NULL,                 // SendMessageAttributes
    NULL,                 // ReceiveMessage
    NULL,                 // BufferLength
    NULL,                 // ReceiveMessageAttributes
    NULL                  // Timeout
);
```

정리하면

1. pmSend 안에 DataLength, TotalLength 세팅
2. CreateMsgMem()으로
    
    [pmSend | 사용자가 입력한 문자열] 한 덩어리 버퍼 lpMem 생성
    
3. lpMem 포인터를 NtAlpcSendWaitReceivePort의 SendMessage로 전달

메시지 내용도 디버깅 해봐야겠죠?

![image12.jpg](image12.jpg)

자 Client에서 **“Hello HackyBoiz!”를 보냈을 때, 버퍼가 실제로 어떻게 생겼는지** 확인해보겠습니다!

```cpp
ntRet = NtAlpcSendWaitReceivePort(
    hPort,          // HANDLE PortHandle
    0,              // ULONG Flags
    (PPORT_MESSAGE)lpMem, // SendMessage
    NULL,           // SendMessageAttributes
    NULL,           // ReceiveMessage
    NULL,           // BufferLength
    NULL,           // ReceiveMessageAttributes
    NULL            // Timeout
);
```

저희 코드에서 NtAlpcSendWaitReceivePort 함수는 이렇게 진행됩니다.

lpMem 변수에 아마 저희가 보낸 메시지가 담겨있겠죠? 직접 확인해봅시다!

![image13.jpg](image13.jpg)

r8 레지스터가 바로 메시지 전체 버퍼인데, PORT_MESSAGE + “Hello HackyBoiz!” 가 들어있는 것을 확인할 수 있습니다.

첫 DWORD 가 00390011 인 이유는

- 하위 2바이트 = `0x0011` → DataLength = 0x11 = 17 (문자열 길이)
- 상위 2바이트 = `0x0039` → TotalLength = 0x39 = 57 (PORT_MESSAGE + payload)

이기 때문이죠! 

이렇게 Send Buffer에서 커널 공간으로 복사한 뒤, Server의 Receive Buffer로 복사되는 구조입니다!

## 4. ALPC Message Attribute

자 여기까지는 “포트를 어떻게 만들고, 어떻게 연결하고, 어떻게 메시지를 보내는지”를 봤습니다. 근데 ALPC가 그냥 메시지 보내기만 하는 IPC였다면 굳이 이렇게까지 복잡하게 만들 이유가 없겠죠?

ALPC가 복잡한 이유는 메시지 본문 말고도 각종 부가 정보(Attribute**)**를 같이 붙여서 보낼 수 있기 때문입니다!

- 보안 컨텍스트 (Impersonation)
- 공유 메모리 섹션(View)
- 핸들(파일/프로세스/섹션 등)
- 토큰 정보
- 사용자 정의 컨텍스트

이런 것들을 한 번의 NtAlpcSendWaitReceivePort 호출로 같이 움직이는 프레임워크라고 보면 됩니다.

### 4.1. Attribute 버퍼 구조

앞에서 봤던 `NtAlpcSendWaitReceivePort()`함수 시그니처를 다시 보면, 중간에 Attribute 관련 인자가 두 개 더 있는 걸 볼 수 있습니다.

```cpp
NTSTATUS NtAlpcSendWaitReceivePort(
    HANDLE PortHandle,
    DWORD Flags,
    PPORT_MESSAGE SendMessage,
    PALPC_MESSAGE_ATTRIBUTES SendMessageAttributes,    // ← 송신 Attribute
    PPORT_MESSAGE ReceiveMessage,
    PSIZE_T BufferLength,
    PALPC_MESSAGE_ATTRIBUTES ReceiveMessageAttributes, // ← 수신 Attribute
    PLARGE_INTEGER Timeout
);
```

- `SendMessageAttributes`
    
    → 내가 메시지를 보낼 때 같이 붙이고 싶은 Attribute
    
- `ReceiveMessageAttributes`
    
    → 나는 응답 받을 때 ****어떤 Attribute를 받고 싶은지를 지정
    

즉, 쌍방 합의 기반 Attribute 전달 구도라고 이해할 수 있을 것 같네요!

Attribute 버퍼 맨 앞에는 공통 헤더가 하나 붙어있는데요,

```cpp
typedef struct _ALPC_MESSAGE_ATTRIBUTES {
    ULONG AllocatedAttributes;  // 이 버퍼가 어떤 Attribute 공간을 가지고 있는지
    ULONG ValidAttributes;     
} ALPC_MESSAGE_ATTRIBUTES, *PALPC_MESSAGE_ATTRIBUTES;
```

- `AllocatedAttributes`
    
    → 이 버퍼에 어떤 Attribute 타입들 공간을 확보했는지 비트 플래그로 표현
    
- `ValidAttributes`
    
    → 이번 메시지에서 실제로 유효한 Attribute
    

정리하자면,

<aside>
💡

AllocatedAttributes = “이 버퍼에 뭐까지 담을 수 있음!”

ValidAttributes = “이번 메시지에서 실제로 쓰는 것”

</aside>

라고 볼 수 있습니다!

### 4.2 주요 Attribute 종류

Attribute도 여러가지가 존재하는데, 대표적인 몇 개만 확인해봅시다.

> **1. 보안 속성 (Security Attribute)**
> 

발신자의 보안 컨텍스트 정보를 포함하며, 메시지 수신 측은 이를 통해 발신자를 Impersonation할 수 있습니다.

```c
typedef struct _ALPC_SECURITY_ATTR {
    ULONG Flags;
    PSECURITY_QUALITY_OF_SERVICE pQOS;
    HANDLE ContextHandle;
} ALPC_SECURITY_ATTR, *PALPC_SECURITY_ATTR;
```

- pQos
    - SECURITY_QUALITY_OF_SERVICE 구조체 포인터
    - 어떤 수준의 impersonation, context tracking 등을 허용할지 설정
- ContextHandle
    - 커널이 관리하는 security context 핸들

---

> **2. View 속성 (Data View Attribute)**
> 

공유 메모리 섹션을 전달하는 데 사용되며, 일반 메시지 크기 제한(64KB)을 초과할 때 사용됩니다.

```c
typedef struct _ALPC_DATA_VIEW_ATTR {
    ULONG  Flags;
    HANDLE SectionHandle;  // ALPC 포트에 attach된 섹션 핸들
    PVOID  ViewBase;       // 이 프로세스에 매핑된 베이스 주소
    SIZE_T ViewSize;       // 매핑 크기
} ALPC_DATA_VIEW_ATTR, *PALPC_DATA_VIEW_ATTR;
```

1. `NtAlpcCreatePortSection` / `NtAlpcCreateSectionView` 로 공유 섹션 생성 + 뷰 설정
2. 그 정보를 `ALPC_DATA_VIEW_ATTR` 에 넣고
3. Attribute 버퍼에 포함시켜 메시지와 같이 전송
4. 상대방은 자신의 주소 공간에 같은 섹션 뷰를 매핑해서 같은 메모리를 공유

---

> **3. Context 속성**
> 

특정 클라이언트 또는 메시지에 사용자 정의 컨텍스트를 부여할 수 있습니다.

서버는 이 속성을 이용해 각 클라이언트를 식별하고 메시지를 상태 기반으로 처리할 수 있죠.

```c
typedef struct _ALPC_CONTEXT_ATTR {
    PVOID PortContext;     // 이 포트(클라이언트)에 붙여둔 컨텍스트
    PVOID MessageContext;  // 이 메시지에 붙여둔 컨텍스트
    ULONG Sequence;        // 시퀀스 번호
    ULONG MessageId;       // 메시지 ID
    ULONG CallbackId;      // 콜백 ID
} ALPC_CONTEXT_ATTR, *PALPC_CONTEXT_ATTR;
```

- `PortContext`
    - 서버가 클라이언트 포트에 심어둔 구조체 포인터 (세션 ID, 유저 정보 등)
- `MessageContext`
    - 특정 메시지별로 별도 정보 저장하고 싶을 때 사용
- `Sequence` / `MessageId` / `CallbackId`
    - 커널이 설정하는 값들
    - TCP처럼 “구조화된 메시지 처리”를 하기 위한 개념 (요청–응답 매칭 등)

---

> **4. Handle 속성**
> 

객체 핸들을 메시지와 함께 전달할 수 있습니다.

```c
typedef struct _ALPC_MESSAGE_HANDLE_INFORMATION {
    ULONG Index;        // Attribute 버퍼 내 인덱스
    ULONG Flags;
    ULONG Handle;       // 넘기려는 핸들 값 (sender 기준)
    ULONG ObjectType;   // 객체 타입
    ACCESS_MASK GrantedAccess;// 수신 측에서 사용할 권한
} ALPC_MESSAGE_HANDLE_INFORMATION, *PALPC_MESSAGE_HANDLE_INFORMATION;
```

- 커널은 핸들이 진짜 유효한지, 권한은 맞는지, cross-process dup 이 가능한지 검사
- 수신자는 이 정보를 기반으로 자신의 프로세스에서 해당 핸들로 객체를 참조

## 마치며

![image14.jpg](image14.jpg)

여기까지가 ALPC의 기본적인 흐름이었습니다!! 겉에서 보면 그냥 IPC 한 덩어리 같지만, 실제로는

- 포트 객체 관리
- PORT_MESSAGE 헤더 처리
- 각종 Attribute 파싱
- 섹션 뷰 매핑, 권한 검증, 핸들 중복 생성 등

이런 것들이 한번의 `NtAlpcSendWaitReceivePort()` 호출 안에서 다 돌아가는 구조입니다!!

그래서 실제 취약점들도 Attribute 해석 및 검증 단계에서 터지는 경우가 많다고 하네요!

기회가 된다면 ALPC 기반 취약점에 대해서도 설명드리고 싶군요! 긴 글 읽어주셔서 감사합니다 다음에 만나요~! 👋

## Reference

https://ti.qianxin.com/blog/articles/the-tragedy-of-alpc-unknown-windows-privilege-escalation-nday-vulnerability-research-in-august-en/

https://csandker.io/2022/05/24/Offensive-Windows-IPC-3-ALPC.html

https://csandker.io/2022/05/29/Debugging-And-Reversing-ALPC.html

https://recon.cx/2008/a/thomas_garnier/LPC-ALPC-paper.pdf

https://theori.io/blog/chaining-n-days-to-compromise-all-part-2-windows-kernel-lpe-a-k-a-chrome-sandbox-escape

https://infocon.org/cons/SyScan/SyScan%202014%20Singapore/SyScan%202014%20presentations/SyScan2014_AlexIonescu_AllabouttheRPCLRPCALPCandLPCinyourPC.pdf

https://en.wikipedia.org/wiki/Local_Inter-Process_Communication

https://processhacker.sourceforge.io/doc/struct___a_l_p_c___s_e_c_u_r_i_t_y___a_t_t_r.html

https://ntdoc.m417z.com/
