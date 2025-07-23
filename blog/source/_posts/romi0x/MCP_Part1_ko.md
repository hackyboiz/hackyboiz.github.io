---
title: "[Research] MCP (Model Context Protocol) Part 1 (ko)"
author: romi0x
tags: [romi0x, research, mcp, llm]
categories: [Research]
date: 2025-07-24 17:00:00
cc: false
index_img: /2025/07/24/romi0x/MCP_Part1_ko/index.png
---
안녕하세요 romi0x입니다!

최근 AI 모델, 특히 LLM 환경에서 문맥(Context) 관리와 상호작용 효율성을 높이기 위한 새로운 접근법으로 MCP (Model Context Protocol)이 제안되고 있습니다. 오늘은 이 MCP에 대해서 간단하게 소개하겠습니다.

## MCP란 무엇일까

사람들이 많이 사용하고 있는 AI 모델인 GPT, Claude, Gemini 등은 이미 코드 작성, 문서 요약, 질의응답 등 다양한 작업에서 높은 성능을 보여주고 있습니다. 하지만 이 모델들도 한계가 있기 마련이죠.

바로 **"문맥(Context)을 완전히 이해하거나 유지하지 못한다"**는 점입니다.

이 문제를 해결하기 위한 시도 중 하나가 바로 Model Context Protocol (MCP)입니다. MCP는 여러 AI 모델, 에이전트, 플러그인들이 공유할 수 있는 문맥 객체(Context Object)를 정의하고, 이를 표준화된 방식으로 주고받을 수 있게 해주는 프로토콜입니다.

MCP는 2024년 11월 Claude를 개발한 [Anthropic](https://www.anthropic.com/news/model-context-protocol)에 의해 처음 공개되었습니다. AI가 다양한 시스템과 데이터를 표준 방식으로 주고받고, 문맥을 일관되게 유지하며, 실질적 문제 해결에 더욱 밀접하게 다가갈 수 있도록 만들어진 오픈 프로토콜입니다.

잘 이해가 가지 않죠? MCP 개발 가이드의 introduction에서 설명하고 있는 비유를 가져와 설명해볼게요.

> "MCP는 AI 애플리케이션을 위한 USB-C 포트이다.”


![출처: [https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/](https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/)](MCP_Part1_ko/image.png)
출처: [https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/](https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/)

> MCP is an open protocol that standardizes how applications provide context to LLMs. Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect your devices to various peripherals and accessories, MCP provides a standardized way to connect AI models to different data sources and tools. 
> 
> 출처 :[modelcontextprotocol.io](https://modelcontextprotocol.io/introduction)


USB-C가 노트북, 스마트폰, 마우스, 모니터 등 다양한 기기를 하나의 포트로 연결할 수 있는 것처럼, MCP는 다양한 데이터 소스(Google Drive, GitHub, Slack 등)를 하나의 방식으로 AI 모델에 연결할 수 있게 해줍니다.

즉, 개발자는 MCP 방식으로 한 번만 연결을 구현해두면, 새로운 LLM이 나와도 다시 만들 필요 없이 재사용이 가능하고, AI 모델 제공자 입장에서도 애플리케이션을 직접 연결하지 않고도 다양한 도구와 연동할 수 있습니다.

![출처: [https://techblog.woowahan.com/22342/](https://techblog.woowahan.com/22342/)](MCP_Part1_ko/image1.png)
출처: [https://techblog.woowahan.com/22342/](https://techblog.woowahan.com/22342/)

우아한형제들 기술 블로그에서는 MCP에 대한 용어를 이렇게 풀어서 설명했어요.

AI 모델에게 프로그램의 사용법을 알려주는 규약, 이제 어려운 용어가 조금 와닿는거 같네요.

## 왜 MCP가 필요할까?

LLM은 일반적으로 입력 토큰 내에서만 문맥을 유지합니다. GPT-4의 경우 최대 128K 토큰이라는 꽤 긴 컨텍스트 윈도우를 지원하지만, 이 역시 단일 세션 한정이며 외부 기억이나 장기적 문맥을 처리하는 데는 한계가 있습니다.

AI 애플리케이션은 하나의 모델이 아닌 **다수의 에이전트와 플러그인이 함께 작동**합니다.

- 사용자 → 요약 플러그인 → 이메일 분석 플러그인 → 캘린더 입력 플러그인

이런 과정에서 동일한 사용자의 의도나 작업 흐름을 각 에이전트가 파악하려면, 문맥이 표준화된 방식으로 공유되어야 합니다. 이를 위해서 MCP가 필요한거죠.

## MCP **작동 방식**

MCP는 간단한 클라이언트-서버 아키텍처를 따릅니다.

![출처: [https://modelcontextprotocol.io/introduction](https://modelcontextprotocol.io/introduction)](MCP_Part1_ko/image2.png)
출처: [https://modelcontextprotocol.io/introduction](https://modelcontextprotocol.io/introduction)

구성요소는 다음과 같아요

- **MCP 호스트**: 외부 데이터에 접근하려는 주체 (예: Claude Desktop, AI 기반 IDE 등)
- **MCP 클라이언트**: MCP 서버와 전담 1:1 연결을 유지하며 요청을 보냅니다
- **MCP 서버**: 실제 데이터(Google Drive, GitHub 등)와 연결되어 정보를 제공하거나 작업을 수행
- **로컬/원격 데이터 소스**: 파일, 데이터베이스, API 등 MCP 서버가 접근할 수 있는 자원

비유로 풀어보자면, 호스트는 CEO, 서버는 각각의 부서, 클라이언트는 각 부서의 전담 비서, MCP는 통일된 문서양식입니다.  CEO는 문서 양식에 맞춰 요청을 작성하고, 각 부서의 비서(클라이언트)가 이를 전달하며, 부서는 처리 후 다시 정해진 방식으로 회신합니다.

덕분에 정보 누락 없이, 혼선 없이 효율적인 협업이 가능 해지는 것이죠.

## MCP의 실제 활용 예시

MCP는 아직 초기 단계이지만, 이미 여러 기업과 개발 도구에서 도입되고 있습니다.

- 사례 1: Claude + Google Drive 문서 요약

사용자는 Claude Desktop 앱에 "지난 회의록을 요약해줘"라고 입력합니다. Claude는 MCP 클라이언트를 통해 로컬에 설치된 Google Drive MCP 서버에 연결하고, 회의록 문서를 찾아 컨텍스트에 로드합니다. 이후 요약 결과를 사용자에게 보여주고, 사용자가 원한다면 다시 Google Drive에 요약 결과를 저장합니다.

> 과거에는 Google Drive API를 수동으로 연동하고 사용자 인증, 파일 구조 파싱을 직접 구현해야 했지만, MCP를 활용하면 이 모든 흐름이 표준 컨텍스트 포맷에 따라 자동화됩니다.


- 사례 2: Claude + GitHub 코드 리뷰

개발자가 Claude에게 "이 Pull Request에 어떤 문제점이 있을까?"라고 묻습니다. Claude는 GitHub MCP 서버를 통해 PR의 변경사항, 리뷰 코멘트, 커밋 히스토리를 읽어들이고 문맥으로 삼아 분석합니다. 이 정보를 기반으로 리뷰 피드백을 제공하죠.

> 이 과정에서 Claude는 단순히 '텍스트 요약'이 아닌, 코드 변경 의도와 전체 맥락을 이해한 고급 분석을 수행할 수 있습니다.


## 기존 기술과 무엇이 다를까?

MCP는 사실 완전히 새롭기보다는, 기존 기술들의 한계를 극복하고자 나온 프로토콜입니다.

기존 기술과는 달리 MCP는 "문맥을 공유하고 지속시키는 연결 구조"에 초점을 둔 기술입니다.

| 구분 | Function Calling (OpenAI) | LangChain Memory | Vector Search + RAG | **MCP** |
| --- | --- | --- | --- | --- |
| 목적 | 외부 기능 호출 | 세션 내 기억 유지 | 문서 기반 질의응답 | 컨텍스트 표준화 및 공유 |
| 데이터 접근 | 실시간 API 호출 | 과거 대화 기록 유지 | 벡터 검색 기반 문서 검색 | 파일, API, DB 등 광범위 |
| 공유 범위 | 단일 모델 세션 | 단일 모델 내부 | 모델 자체만 사용 | **여러 도구/모델 간 공유** |
| 표준화 | X (모델 종속) | 프레임워크 의존 | 문서 포맷 비표준 | JSON 기반 명세 존재 |
| 보안 관리 | 없음 (직접 구현) | 없음 | 없음 | 클라이언트-서버 기반 제어 가능 |
| 확장성 | 낮음 | 낮음 | 보통 | **높음 (플러그형 구조)** |

오늘은 MCP의 전체적인 개념을 알아보았는데요. 다음 시리즈에서는 MCP 관련한 논문을 가져올게요-!!(아마두_ㅎㅎ)


## Ref

[https://techblog.woowahan.com/22342/](https://techblog.woowahan.com/22342/)

[https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/](https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/)

[https://channel.io/ko/blog/articles/what-is-mcp-52c77e72](https://channel.io/ko/blog/articles/what-is-mcp-52c77e72)