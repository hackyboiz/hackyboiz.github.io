---
title: "[Research] Windows ALPC (EN)"
author: gongjae
tags: [Windows, ALPC, IPC, gongjae]
date: 2025-11-27 19:00:00
categories: [Research]
cc: true
index_img: /2025/11/27/gongjae/alpc/kr/image01.jpg
---

Hello! I’m `gongjae`, a new member who recently joined HackyBoiz! 👻

During my time as a trainee at HackyBoiz, I had a question related to the Kernel Driver and Named Pipe bug hunting project I had previously worked on.

![image00.jpg](image00.jpg)

Have you also audited ALPC or COM communications in addition to named pipes?

~~Nope~~

Actually, I already knew that ALPC is one of the IPC communication methods, but I decided to take this opportunity to study it properly. 

So today, we’ll explore the concept of ALPC, its communication flow, and how messages are transmitted — all while debugging it directly! :)

![image01.jpg](image01.jpg)

## 1. About ALPC

In the Windows environment, IPC mechanisms that can be used both remotely and locally include Named Pipes and RPC. However, ALPC is unique in that it can only be used locally.

While RPC stands for Remote Procedure Call, ALPC stands for Advanced Local Procedure Call and is sometimes also referred to as Asynchronous Local Call.

The term “asynchronous” reflects the Windows Vista era, during which ALPC was introduced to replace the existing LPC mechanism.

ALPC enables high-speed message communication and is used for data transmission between processes.

### 1.1. LPC to ALPC

![image02.jpg](image02.jpg)

Before Windows Vista, LPC was an undocumented internal IPC mechanism provided by the Microsoft Windows NT kernel, used for lightweight communication between processes on the same machine.

Due to LPC’s synchronous nature, both the client and server had to wait until a message was processed, which resulted in continuous blocking and performance issues.

As a result, from Windows Vista onward, the existing LPC mechanism was effectively reimplemented on top of ALPC, and later versions of Windows internally rely on ALPC as the core IPC mechanism. However, LPC APIs themselves did not completely disappear; rather, their internal implementation was redirected to ALPC.

<aside>
💡

As seen in the image below, existing functions for creating LPC ports remain, but internally they are redirected to create ALPC ports instead. @.@

![image03.jpg](image03.jpg)

</aside>

### 1.2. ALPC Internal Structure

The main component of ALPC communication is the ALPC port object, and its usage is similar to a network socket.

→ The server opens a socket that clients can connect to in order to exchange messages.

Using Sysinternals Suite’s WinObj.exe, we can inspect ALPC ports.

![image04.jpg](image04.jpg)

Although some ALPC ports exist in the root path, most of them are located under the \RPC Control path.

In ALPC communication, three ALPC ports are involved (2 on the server side, 1 on the client side). The ALPC ports shown in WinObj are ALPC Communication Ports that clients can connect to.

Before explaining the communication process, let’s briefly go over some important functions we will repeatedly encounter.

> **NtAlpcCreatePort()**
> 

```cpp
NTSTATUS NTAPI NtAlpcCreatePort(
    OUT PHANDLE PortHandle,
    IN POBJECT_ATTRIBUTES ObjectAttributes,
    IN OUT PALPC_INFO PortInformation OPTIONAL
);
```

- This function is used to create an ALPC port. Unlike LPC, instead of listing options individually, ALPC bundles them into an ALPC_INFO structure and passes it as the final parameter.
- This structure is copied into the ALPC object when the port is created and later referenced internally.

```cpp
typedef struct _ALPC_INFO
{
#define PORT_INFO_LPCMODE               0x001000 // Operate like an LPC port
#define PORT_INFO_CANIMPERSONATE        0x010000 // Allow impersonation
#define PORT_INFO_REQUEST_ALLOWED       0x020000 // Allow message requests
#define PORT_INFO_SEMAPHORE             0x040000 // Enable synchronization mechanism
#define PORT_INFO_HANDLE_EXPOSE         0x080000 // Allow handle exposure
#define PORT_INFO_PARENT_SYSTEM_PROCESS 0x100000 // Kernel ALPC interface (system process parent)

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

- The `PortName` parameter specifies the name of the port to connect to, along with various optional settings. The kernel searches for the ALPC Connection Port object with the given name and then proceeds to request a connection.

**Important ConnectionFlags values**

```cpp
#define ALPC_SYNC_CONNECTION   0x020000 // Synchronous connection
#define ALPC_USER_WAIT_MODE    0x100000 // Wait in user mode
#define ALPC_WAIT_IS_ALERTABLE 0x200000 // Alertable wait
```

- By default, the connection is asynchronous.
    - This means the client can obtain a handle **before** the server has actually processed and accepted the connection request.
    - So if the client sends a message while the server hasn’t handled the connection yet, it can result in an error.

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

- This is the most important function: with this single function, you can send and receive messages at the same time!!
- SendMessage, ReceiveMessage are literally the send/receive message buffers.
- SendMessageAttributes, ReceiveMessageAttributes are used to request or receive extra actions along with the message (section mapping, passing handles, etc.).

## 2. ALPC Communication Flow

Alright, now that we roughly know what each function does, let’s look at the actual communication flow. Earlier, I mentioned that an ALPC communication scenario involves **three** ALPC ports:

1. The ALPC connection port created by the server process
2. The ALPC server communication port newly created by the kernel when a client connects
3. The ALPC client communication port

We use these three, but just listing them like this doesn’t make it very intuitive…

~~At least it didn’t for meㅠ~~

In reality, it *looks* like three logical ports exist during ALPC communication, but under the hood it’s more like multiple endpoints being created on top of a single ALPC port object.

Let’s walk through the communication flow step by step!

![[image00.jpg](https://theori.io/blog/chaining-n-days-to-compromise-all-part-2-windows-kernel-lpe-a-k-a-chrome-sandbox-escape)](image05.jpg)

The function we should pay the most attention to is, again, `NtAlpcSendWaitReceivePort()`. This guy is basically responsible for all the request/response message exchange. In order, the flow looks like this:

1. The server process calls `NtAlpcCreatePort()` to create an ALPC port.
    - Example name: `\RPC Control\HackyBoiz`
2. The kernel creates the ALPC port object and returns a handle to the server.
    
    → This is the **ALPC Connection Port!**
    
3. The server calls `NtAlpcSendWaitReceivePort()` and waits for a client connection.
4. The client calls `NtAlpcConnectPort()`.
    - Name of the server port to connect to: `\RPC Control\HackyBoiz`
    - (Optional) Initial message to send to the server
    - (Optional) Server SID to verify it’s talking to the correct server
    - (Optional) Additional message attributes
5. This connection request is delivered to the server, and the server calls `NtAlpcAcceptConnectPort()` to either accept or reject the connection.
    - The last argument is a boolean: `TRUE` means accept, `FALSE` means reject.
6. If the connection is accepted, the kernel creates a new ALPC Communication Port and returns handles to both the server and the client.
7. From this point on, messages are no longer sent through the Connection Port, but through this new Communication Port.

Now it’s starting to make a bit more sense, right?

Good. Then let’s actually debug it and see it in action.

You can grab an example ALPC implementation here:

https://github.com/DownWithUp/ALPC-Example

We’ll run this code and walk through the communication process ourselves! XD

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

If you build and run Server.c, you can see that a server ALPC port is created under \RPC Control.

![image06.jpg](image06.jpg)

Now let’s debug the server side with WinDbg. First, take a look at NtAlpcCreatePort:

```cpp
NTSTATUS NTAPI NtAlpcCreatePort(
    OUT PHANDLE PortHandle,
    IN POBJECT_ATTRIBUTES ObjectAttributes,
    IN OUT PALPC_INFO PortInformation OPTIONAL
);
```

![image07.jpg](image07.jpg)
After setting a breakpoint, you’ll see that rdx points to the ObjectAttributes. We’ll look at Attributes in more detail later, but inside this structure there is an ObjectName. The OBJECT_ATTRIBUTES structure looks like this:

```cpp
typedef struct _OBJECT_ATTRIBUTES {
    ULONG           Length;
    HANDLE          RootDirectory;
    PUNICODE_STRING ObjectName;   // ← this one
    ULONG           Attributes;
    PVOID           SecurityDescriptor;
    PVOID           SecurityQualityOfService;
} OBJECT_ATTRIBUTES, *POBJECT_ATTRIBUTES;
```

So rdx + 0x10 should be the ObjectName, right? If we inspect that:

![image08.jpg](image08.jpg)

You can see that the port is created with the name \RPC Control\HackyBoiz!

Now let’s switch to the client side and debug the communication with WinDbg.

![image09.jpg](image09.jpg)

Once we run the client, the server accepts the connection, and the client can start sending messages. Let’s look at that part in more detail.

![image10.jpg](image10.jpg)

First, we inspect the initial connection call to `NtAlpcConnectPort`:

```cpp
NTSTATUS NTAPI NtAlpcConnectPort(
    OUT PHANDLE PortHandle,
    IN PUNICODE_STRING PortName, // <- here
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

rdx holds the PortName. If we dump that, we can confirm that it’s pointing to the HackyBoiz port we set up on the server!

Once the Connect request is sent, the server decides whether to accept it or not.

Looking at the NtAlpcAcceptConnectPort call in Server.c:

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

The arguments are set up like this.

- rcx is the handle that the server will receive for the new communication port. After the syscall, you’ll see that hConnectedPort now holds the actual ALPC server communication port handle.

Since the last argument AcceptConnection is TRUE, the connection is accepted, and both sides can now exchange messages over this communication port!

## 3. ALPC Messaging

Now that we’ve seen how ALPC connects, let’s look at the actual message format going back and forth.

In ALPC, messages always have the following layout:

![image11.jpg](image11.jpg)

```cpp
[ PORT_MESSAGE ][ Payload(Data) ]
```

The front is the fixed PORT_MESSAGE header, and after that comes the actual text/binary payload we want to send.

### 3.1. PORT_MESSAGE structure

The header structure looks like this:

```cpp
typedef struct _PORT_MESSAGE
{
    union {
        struct {
            USHORT DataLength;   // Actual payload length
            USHORT TotalLength;  // Total size = PORT_MESSAGE + payload
        } s1;
        ULONG Length;
    } u1;

    union {
        struct {
            USHORT Type;             // Message type
            USHORT DataInfoOffset;   // Attribute area offset
        } s2;
        ULONG ZeroInit;
    } u2;

    union {
        CLIENT_ID ClientId;          // Sender process/thread ID
        double    DoNotUseThisField; // (alignment)
    };

    ULONG MessageId;                 // Message identifier

    union {
        SIZE_T ClientViewSize;       // Section view size
        ULONG  CallbackId;           // Callback ID
    };

} PORT_MESSAGE, *PPORT_MESSAGE;
```

In practice, we usually only care about around four fields:

- u1.s1.DataLength
    
    → Length of the payload
    
- u1.s1.TotalLength
    
    → sizeof(PORT_MESSAGE) + DataLength
    
    → Used by the kernel to know the total size of the message
    
- ClientId
    
    → Process and thread ID of the sender
    
- MessageId / CallbackId
    
    → Used to match requests and responses
    

So the actual format is pretty straightforward.

If we look again at the example client code:

```cpp
#define MSG_LEN 0x100

LPVOID CreateMsgMem(PPORT_MESSAGE PortMessage, SIZE_T MessageSize, LPVOID Message)

    LPVOID lpMem = HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, MessageSize + sizeof(PORT_MESSAGE));
    memmove(lpMem, PortMessage, sizeof(PORT_MESSAGE));
    memmove((BYTE*)lpMem + sizeof(PORT_MESSAGE), Message, MessageSize);
    return(lpMem);
}
```

Here, `lpMem` is exactly:

```cpp
[ PORT_MESSAGE ][ Payload ]
```

A fully-formed ALPC message buffer.

In main() we use it like this:

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

To summarize:

1. We set DataLength and TotalLength inside pmSend.
2. CreateMsgMem() builds a single buffer lpMem with:
    
    [ pmSend | user input string ]
    
3. We pass lpMem as SendMessage to NtAlpcSendWaitReceivePort.

Of course, we should also look at how the message actually looks in memory during debugging.

![image12.jpg](image12.jpg)

Let’s check what the buffer looks like when the client sends **“Hello HackyBoiz!”**.

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

In our code, NtAlpcSendWaitReceivePort is called like this.

lpMem should contain the message we’re sending. Let’s inspect it!

![image13.jpg](image13.jpg)

r8 is the pointer to the full message buffer, and we can clearly see PORT_MESSAGE followed by "Hello HackyBoiz!".

The first DWORD is 00390011 because:

- Lower 2 bytes = 0x0011 → DataLength = 0x11 = 17 (string length)
- Upper 2 bytes = 0x0039 → TotalLength = 0x39 = 57 (PORT_MESSAGE + payload)

This is the buffer that gets copied from user mode to kernel mode, then from the server’s kernel-side buffer to the server’s receive buffer.

## 4. ALPC Message Attribute

So far we’ve looked at **how to create ports, connect them, and send messages**.

But if ALPC were only about sending messages, there’d be no reason for it to be this complex.

The real reason ALPC is complicated is that, besides the message body, it can also attach various **attributes** to a message:

- Security context (impersonation)
- Shared memory sections (views)
- Handles (files, processes, sections, etc.)
- Token information
- User-defined context

ALPC is basically a framework that lets you send all of this together in a single NtAlpcSendWaitReceivePort call.

### 4.1. Attribute buffer structure

If we look again at the `NtAlpcSendWaitReceivePort()` signature, we see two extra parameters for attributes:

```cpp
NTSTATUS NtAlpcSendWaitReceivePort(
    HANDLE PortHandle,
    DWORD Flags,
    PPORT_MESSAGE SendMessage,
    PALPC_MESSAGE_ATTRIBUTES SendMessageAttributes,    // ← send-side attributes
    PPORT_MESSAGE ReceiveMessage,
    PSIZE_T BufferLength,
    PALPC_MESSAGE_ATTRIBUTES ReceiveMessageAttributes, // ← receive-side attributes
    PLARGE_INTEGER Timeout
);
```

- `SendMessageAttributes`
    
    → Attributes we want to attach when sending the message.
    
- `ReceiveMessageAttributes`
    
    → Attributes we want to receive along with the response.
    

So you can think of it as a “mutual agreement” model for sending attributes.

At the beginning of the attribute buffer, there is a common header:

```cpp
typedef struct _ALPC_MESSAGE_ATTRIBUTES {
    ULONG AllocatedAttributes;  // Which attribute types this buffer has space for
    ULONG ValidAttributes;      // Which attributes are actually valid for this message
} ALPC_MESSAGE_ATTRIBUTES, *PALPC_MESSAGE_ATTRIBUTES;
```

- `AllocatedAttributes`
    
    → Bit flags indicating which attribute types this buffer can hold
    
- `ValidAttributes`
    
    → Which attributes are actually used for this specific message
    

To paraphrase it:

> AllocatedAttributes = “This buffer can hold these attributes.”
> 
> 
> **ValidAttributes** = “These are the ones we’re *actually* using for this message.”
> 

### 4.2. Main attribute types

There are several types of attributes, but let’s look at the most important ones.

> 1. Security Attribute
> 

This includes the sender’s security context, which allows the receiver to impersonate the sender.

```c
typedef struct _ALPC_SECURITY_ATTR {
    ULONG Flags;
    PSECURITY_QUALITY_OF_SERVICE pQOS;
    HANDLE ContextHandle;
} ALPC_SECURITY_ATTR, *PALPC_SECURITY_ATTR;
```

- `pQOS`
    - Pointer to a `SECURITY_QUALITY_OF_SERVICE` structure
    - Controls what level of impersonation, context tracking, etc. is allowed
- `ContextHandle`
    - Handle to a kernel-managed security context

---

> 2. View Attribute (Data View Attribute)
> 

Used to pass a shared memory section. This is how you bypass the normal 64KB message size limit.

```c
typedef struct _ALPC_DATA_VIEW_ATTR {
    ULONG  Flags;
    HANDLE SectionHandle;  // Section handle attached to the ALPC port
    PVOID  ViewBase;       // Base address where the section is mapped in this process
    SIZE_T ViewSize;       // Size of the mapped view
} ALPC_DATA_VIEW_ATTR, *PALPC_DATA_VIEW_ATTR;
```

1. Create a shared section and configure a view using `NtAlpcCreatePortSection` / `NtAlpcCreateSectionView`.
2. Fill an `ALPC_DATA_VIEW_ATTR` with that information.
3. Attach it to the attribute buffer and send it along with the message.
4. The receiver maps the same section view in its own address space, effectively sharing memory.

---

> 3. Context Attribute
> 

Used to attach user-defined context to a particular client or message.

The server can use this to track per-client state or per-message state.

```c
typedef struct _ALPC_CONTEXT_ATTR {
    PVOID PortContext;     // Context attached to this port (client)
    PVOID MessageContext;  // Context attached to this specific message
    ULONG Sequence;        // Sequence number
    ULONG MessageId;       // Message ID
    ULONG CallbackId;      // Callback ID
} ALPC_CONTEXT_ATTR, *PALPC_CONTEXT_ATTR;
```

- `PortContext`
    - A pointer to a structure the server associates with the client port (session ID, user info, etc.)
- `MessageContext`
    - Used when you want separate context per message
- `Sequence` / `MessageId` / `CallbackId`
    - Values set by the kernel
    - Help with structured message handling, similar to how TCP tracks state for requests/responses

---

> 4. Handle Attribute
> 

Used to pass object handles along with the message.

```c
typedef struct _ALPC_MESSAGE_HANDLE_INFORMATION {
    ULONG Index;        // Index in the attribute buffer
    ULONG Flags;
    ULONG Handle;       // Handle value (from the sender’s perspective)
    ULONG ObjectType;   // Object type
    ACCESS_MASK GrantedAccess; // Access rights the receiver will get
} ALPC_MESSAGE_HANDLE_INFORMATION, *PALPC_MESSAGE_HANDLE_INFORMATION;
```

- The kernel checks whether the handle is valid, whether the rights make sense, and whether cross-process duplication is allowed.
- The receiver then uses this information to obtain a valid handle for the corresponding object in its own process.

## Wrapping Up

![image14.jpg](image14.jpg)

So that’s the basic flow of ALPC! On the surface it just looks like “yet another IPC mechanism,” but under the hood, a single call to `NtAlpcSendWaitReceivePort()` takes care of:

- Managing port objects
- Handling the `PORT_MESSAGE` header
- Parsing various attributes
- Mapping section views, validating permissions, duplicating handles, etc.

Because of this complexity, a lot of real-world vulnerabilities tend to appear in the logic that interprets and validates these attributes.

If I get the chance, I’d love to write another post just focusing on ALPC-based vulnerabilities.

Thanks for reading this long post — see you next time! 👋

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
