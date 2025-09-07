---
title: "[Research] MCP (Model Context Protocol) Part 2 (ko)"
author: romi0x
tags: [romi0x, research, mcp, llm]
categories: [Research]
date: 2025-09-02 17:00:00
cc: false
index_img: /2025/09/02/romi0x/MCP_Part2_ko/index.png
---

안녕하세요! romi0x입니다~

이번 글은 [저번 글](https://hackyboiz.github.io/2025/07/24/romi0x/MCP_Part1_ko/)에 이어 MCP Part2 입니다~

지난 글에서는 MCP의 개념과 등장 배경, 기본 구조와 도입 사례를 다뤘습니다. 이번 글에서는 **MCP가 내부적으로 어떻게 작동하는지**에 집중해서 작성해 봤어요.

# MCP의 핵심 컴포넌트 (Host / Client / Server)

저번 글에서도 언급했듯이 MCP는 여러 LLM들이 문맥 객체를 정의하고, 이를 표준화된 방식으로 주고받을 수 있게 해주는 프로토콜입니다. 또한, 저번 글에서도 MCP의 간단한 작동 방식에 대해 소개한 바가 있죠!

구성요소는 **MCP 호스트, MCP 클라이언트, MCP 서버** 등으로 구성되어 있다고 덧붙었습니다. 그럼, 이 구성 요소들이 정확히 어떻게 동작되고 어떤 구조를 가지고 있는지 살펴볼게요.

3가지의 구성요소를 간단하게 요약하면 다음과 같아요.

| 컴포넌트 | 설명 | 예시 |
| --- | --- | --- |
| **MCP Server** | 실제 리소스를 보유한 백엔드 | Google Drive MCP, GitHub MCP 등 |
| **MCP Client** | Host ↔ Server 사이의 통신 중계자 | OpenAI의 Function Router 등 |
| **MCP Host** | LLM이나 LLM 앱 (Claude, GPT 등) | Claude Desktop, AI IDE |

이 구성요소를 하나씩 뜯어봅시다!

## (1) MCP Server

MCP 서버는 **AI 애플리케이션이 외부 세계와 연결되어 동작할 수 있도록 기능을 제공하는 백엔드 컴포넌트**입니다. 이 서버는 파일 시스템, 이메일, 여행 일정, 데이터베이스 등 특정 도메인에 특화된 기능을 제공합니다.

예를 들면, 문서 관리를 위한 파일 시스템 서버, 메시지 처리를 위한 이메일 서버, 여행 계획을 위한 여행 서버 등이 있습니다.

AI 모델은 MCP 서버를 통해 리소스에 접근하고, 툴을 실행하며, 명시된 인터페이스에 따라 다양한 액션을 수행할 수 있습니다.

이러한 구조는 LLM을 단순한 언어 모델이 아닌, **앱처럼 동작할 수 있는 환경**으로 확장시킵니다.

MCP 서버는 세 가지 주요 구성 요소로 기능을 제공합니다. 그 3가지 구성 요소는 **Tools, Resources, Prompts** 입니다.

| 구성 요소 | 목적 | 제어 주체 | 예시 |
| --- | --- | --- | --- |
| **Tool** | 모델이 수행할 수 있는 액션 | 모델 | 항공편 검색, 이메일 전송, 일정 등록 |
| **Resource** | 문맥 데이터를 제공 | 애플리케이션 | 캘린더, 문서, 여행 이력, 날씨 데이터 |
| **Prompt** | 상호작용 템플릿 제공 | 사용자 | "휴가 계획 짜줘", "회의 요약해줘" |

각 요소는 표준화된 스키마와 URI 구조를 따르며, JSON-RPC를 통해 통신합니다.

### (1-1) Tools: AI가 실행하는 액션

Tool은 AI 모델이 외부 세계에서 **작업을 수행하기 위해 호출하는 실행 가능한 기능**입니다.

예를 들어, 항공편 검색, 이메일 전송, 캘린더 일정 등록과 같은 액션은 모두 MCP Tool로 구현할 수 있습니다.

- 구조

  각 Tool은 JSON Schema로 명확하게 정의된 입력/출력 구조를 가집니다.

  이는 LLM이 도구를 사용할 때 형식적 오류 없이 정확한 입력을 생성할 수 있도록 돕습니다.

    ```bash
    {
      "name": "searchFlights",
      "description": "Search for available flights",
      "inputSchema": {
        "type": "object",
        "properties": {
          "origin": { "type": "string" },
          "destination": { "type": "string" },
          "date": { "type": "string", "format": "date" }
        },
        "required": ["origin", "destination", "date"]
      }
    }
    
    ```

- 프로토콜 메서드


    | 메서드 | 설명 |
    | --- | --- |
    | `tools/list` | 사용 가능한 툴 목록 조회 |
    | `tools/call` | 특정 툴 실행 요청 |

Tool 실행은 반드시 **사용자의 명시적 승인**을 거칩니다. 사용자는 실행 전 해당 Tool의 설명, 입력값, 예상 결과를 확인하고 승인 여부를 결정합니다. 신뢰할 수 있는 Tool에 대해 **자동 승인 설정**도 가능합니다.

### (1-2) Resources – 문맥(Context) 데이터 제공

Resource는 LLM이 이해할 수 있는 형태로 제공되는 **외부 데이터 소스**입니다. 이는 문서, 캘린더 일정, 날씨 정보, 이메일, DB 레코드 등 다양할 수 있으며, 모델은 이를 문맥으로 받아들이고 처리할 수 있습니다.

- 구조

  Resource는 고유한 URI를 통해 식별되며, `file://`, `calendar://`, `travel://` 등의 스킴을 사용합니다. 또한 MIME 타입, 제목, 설명 등의 메타데이터를 포함합니다.


```bash
{
  "uriTemplate": "weather://forecast/{city}/{date}",
  "name": "weather-forecast",
  "title": "Weather Forecast",
  "description": "Get weather forecast for any city and date",
  "mimeType": "application/json"
}

```

- 프로토콜 메서드


    | 메서드 | 설명 |
    | --- | --- |
    | `resources/list` | 사용 가능한 리소스 조회 |
    | `resources/templates/list` | 리소스 템플릿 조회 |
    | `resources/read` | 특정 리소스 데이터 읽기 |
    | `resources/subscribe` | 리소스 변경 구독 (옵션) |

리소스는 일반적으로 **파일 브라우저**, **검색창**, **카테고리 필터링 UI**를 통해 탐색할 수 있습니다. LLM은 애플리케이션이 선택한 리소스를 **자동으로 불러오거나**, 사용자가 수동으로 선택할 수 있습니다. 선택된 리소스는 AI에게 컨텍스트로 전달되어 질문 응답, 분석 등에 사용됩니다.

### (1-3) Prompts – 상호작용 템플릿

Prompt는 사용자가 자주 수행하는 작업을 **정형화된 형식으로 재사용할 수 있도록 정의한 템플릿**입니다. 프롬프트를 통해 모델은 자연어 입력보다 더 명확한 구조화된 요청을 받을 수 있으며, 특정 도메인에 맞는 워크플로우를 유도할 수 있습니다.

- 구조

    ```bash
    {
      "name": "plan-vacation",
      "title": "Plan a vacation",
      "description": "Guide through vacation planning process",
      "arguments": [
        { "name": "destination", "type": "string", "required": true },
        { "name": "duration", "type": "number" },
        { "name": "budget", "type": "number" },
        { "name": "interests", "type": "array", "items": { "type": "string" } }
      ]
    }
    
    ```

- 프로토콜 메서드


    | 메서드 | 설명 |
    | --- | --- |
    | `prompts/list` | 사용 가능한 프롬프트 목록 조회 |
    | `prompts/get` | 특정 프롬프트 세부 정보 조회 |

사용자는 `/plan-vacation`과 같은 슬래시 명령 또는 UI 버튼을 통해 프롬프트를 실행할 수 있습니다. 프롬프트는 구조화된 입력폼(텍스트, 숫자, 드롭다운 등)으로 나타나며, 입력된 값은 모델에게 직접 전달됩니다. 이를 통해 **일관된 결과**, **자동완성**, **워크플로우 자동화**가 가능해집니다.

---

## **(2) MCP Client**

MCP 클라이언트는 **호스트 애플리케이션인 Claude, AI IDE 등이 MCP 서버와 통신하기 위해 생성하는 컴포넌트**입니다. 즉, 사용자는 직접 눈에 보거나 다루는 대상은 아니지만, 모든 서버 연결은 반드시 클라이언트를 거쳐야 합니다.

서버에서 데이터를 읽어오거나, 도구를 실행하거나, 사용자에게 추가 정보를 요청할 때, 모든 요청이 클라이언트를 통과해야 합니다.

- MCP Client 가 중요한 이유

  MCP 클라이언트의 가장 큰 역할은 **서버가 직접 LLM이나 사용자의 환경에 접근하지 못하도록 차단하는 것**입니다. 모든 데이터 접근과 도구 실행은 반드시 사용자 눈앞에서 승인 과정을 거치도록 설계되어 있어, 사용자는 AI가 어떤 작업을 시도하는지 명확히 확인하고 통제할 수 있습니다.

  특히 파일 시스템 접근처럼 위험할 수 있는 기능은 클라이언트가 **루트 경계(roots)를 설정**해 “여기까지만 접근 가능하다”는 제한을 두어 안전성을 확보합니다. 또한 서버가 모델 호출이 필요할 때도 직접 실행하지 않고, **클라이언트를 통해 위임하게 함**으로써 비용, 보안, 승인 절차까지 모두 사용자 중심에서 제어됩니다.

- 주요 기능


    | 기능명 | 설명 |
    | --- | --- |
    | **Sampling** | 서버가 AI 모델 호출을 클라이언트에게 위임 |
    | **Roots** | 서버가 접근 가능한 파일 시스템 경계 설정 |
    | **Elicitation** | 서버가 사용자에게 추가 입력을 요청할 수 있도록 지원 |

3가지 주요 기능을 하나씩 알아봅시다.

### **(2-1) Sampling: 모델 호출 위임**

Sampling은 서버가 직접 LLM API를 호출하지 않고, 클라이언트를 통해 **AI 모델 호출을 위임**할 수 있는 기능입니다.

- 예시 흐름

  ![출처: [https://modelcontextprotocol.io/docs/learn/client-concepts](https://modelcontextprotocol.io/docs/learn/client-concepts)](MCP_Part2_ko/image.png)

  출처: [https://modelcontextprotocol.io/docs/learn/client-concepts](https://modelcontextprotocol.io/docs/learn/client-concepts)

    1. 서버는 클라이언트에 "이 데이터 분석해줘"라는 요청을 보냅니다.
    2. 클라이언트는 요청 내용을 사용자에게 명확히 보여주고, **승인 여부를 확인**합니다.
    3. 사용자가 승인하면 클라이언트가 LLM을 호출하고 응답을 서버에 전달합니다.
    4. **모든 응답은 사용자 검토 후 전달됩니다.**
- 샘플 요청 예시

    ```bash
    {
      "messages": [
        {
          "role": "user",
          "content": "다음 항공편 옵션을 분석하고 최적의 선택을 추천해주세요..."
        }
      ],
      "modelPreferences": {
        "hints": [{ "name": "claude-3-5-sonnet" }],
        "costPriority": 0.3,
        "speedPriority": 0.2,
        "intelligencePriority": 0.9
      },
      "systemPrompt": "여행 전문가로서 항공편 추천을 수행해주세요",
      "maxTokens": 1500
    }
    ```


### **(2-2) Roots: 파일 시스템 경계 설정**

Roots는 서버가 접근할 수 있는 **파일 시스템의 범위(디렉터리)를 정의**하는 기능입니다. 모든 접근은 `file://` URI로 구성되며, 클라이언트가 이 경계를 서버에 명확히 전달합니다.

- 샘플 요청 예시

    ```bash
    {
      "uri": "file:///Users/agent/travel-planning",
      "name": "Travel Planning Workspace"
    }
    ```


서버는 이 루트 내부의 파일만 접근 가능하며, 외부 경로는 클라이언트 정책에 따라 차단됩니다. 사용자가 폴더를 열면 클라이언트는 자동으로 roots를 업데이트할 수 있으며, `roots/list_changed` 이벤트로 서버에 알립니다.

### (2-3) Elicitation: 사용자 추가 입력 요청

Elicitation은 서버가 작업 중 필요한 정보를 사용자에게 **동적으로 요청할 수 있게 해주는 기능**입니다.

- 사용 흐름

  ![출처: [https://modelcontextprotocol.io/docs/learn/client-concepts](https://modelcontextprotocol.io/docs/learn/client-concepts)](MCP_Part2_ko/image1.png)
  출처: [https://modelcontextprotocol.io/docs/learn/client-concepts](https://modelcontextprotocol.io/docs/learn/client-concepts)

  1. 서버가 사용자에게 입력 요청을 생성합니다.
  2. 클라이언트는 UI를 통해 메시지 및 입력 양식을 제공합니다.
  3. 사용자가 입력을 완료하면 해당 정보를 서버에 전달하고 작업을 계속 진행합니다.
- 샘플 요청 예시

    ```bash
    {
      "method": "elicitation/requestInput",
      "params": {
        "message": "바르셀로나 여행 예약을 확정해 주세요:",
        "schema": {
          "type": "object",
          "properties": {
            "confirmBooking": { "type": "boolean" },
            "seatPreference": {
              "type": "string",
              "enum": ["window", "aisle", "no preference"]
            },
            "roomType": {
              "type": "string",
              "enum": ["sea view", "city view"]
            }
          },
          "required": ["confirmBooking"]
        }
      }
    }
    
    ```


---

## (3) 전체 통신 구조

![출처: 본인 제작](MCP_Part2_ko/image2.png)
출처: 본인 제작

**사용자(User)**는 호스트 애플리케이션(예: Claude, GPT 기반 IDE 등)을 통해 AI 모델과 상호작용합니다.

**AI 모델(Host)**은 사용자의 요청을 해석하고 필요한 작업을 결정합니다.

**MCP 클라이언트(Client)**는 호스트와 MCP 서버 사이에서 **중계자 역할**을 수행합니다.

모든 통신은 **JSON-RPC** 형식으로 주고받으며, 클라이언트는 서버와의 연결을 관리하면서 사용자 승인, 보안 제어, 모델 호출 위임(sampling) 등을 담당합니다.

MCP 서버(Server)는 실제 기능을 제공합니다. 서버는 세 가지 핵심 빌딩 블록을 노출합니다.

- **Tool**: AI가 실행할 수 있는 구체적인 액션 (예: 항공편 검색, 이메일 전송, 일정 등록)
- **Resource**: AI에게 제공되는 문맥 데이터 (예: 파일, 캘린더, 날씨 정보)
- **Prompt**: 사용자가 재사용할 수 있는 구조화된 상호작용 템플릿 (예: "휴가 계획 짜줘")

이러한 구조를 통해 MCP는 AI 모델이 외부 데이터와 도구에 안전하고 일관되게 접근할 수 있도록 하며, 사용자는 모든 액션을 직접 승인하고 제어할 수 있는 신뢰 기반의 환경을 제공합니다.

이번 연구글은 MCP에 대해서 좀 더 자세하게 다뤄봤습니다 -!

실제로 사용하면 이해가 확 될 것 같네요!

MCP를 손쉽게 사용할 수 있는 [튜토리얼](https://modelcontextprotocol.io/docs/tutorials/use-remote-mcp-server)이 있으니 함께보면 좋을 것 같아요~

### Ref

[https://modelcontextprotocol.io/docs/getting-started/intro](https://modelcontextprotocol.io/docs/getting-started/intro)