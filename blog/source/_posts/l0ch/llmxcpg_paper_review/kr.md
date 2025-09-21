---
title: "[Research] “LLMxCPG: Context-Aware Vulnerability Detection Through Code Property Graph-Guided Large Language Models” Paper Review (KR)"
author: L0ch
tags: [L0ch, llm, cpg, vulnerability detection, paper, review, usenix]
categories: [Research]
date: 2025-09-22 19:00:00
cc: false
index_img: 2025/09/22/l0ch/llmxcpg_paper_review/kr/thumbnail.jpeg
---

# 서론

안녕하세요, L0ch입니다! 최근 CPG(Code Property Graph)에 흥미가 생겨서 여기저기 기웃대고 있는데요. 이번 글에서는 USENIX Security ‘25 논문인 LLMxCPG: Context-Aware Vulnerability Detection Through Code Property Graph-Guided Large Language Models를 리뷰해보겠습니다.

> 원본 논문: [https://arxiv.org/pdf/2507.16585](https://arxiv.org/pdf/2507.16585)
> 

# Background

모든 분야가 그렇듯이, 최근 취약점 탐지에서도 LLM이 많이 활용되고 있는데요. 저도 요즘 하루한줄을 쓰는 등 1-day 취약점을 분석할 때나 제로데이를 찾을 때 LLM을 많이 활용하죠.

그런데 막상 큰 규모의 코드베이스인 프로그램에서 분석이 실패한다거나, 아예 컨텍스트 크기를 넘어 수동 개입을 해야 하는 문제를 종종 겪은 경험이 있습니다. 물론 요즘 gemini나 gpt같은 대규모 모델들은 컨텍스트 크기가 전과 비교하면 엄청나게 커졌지만, 파인 튜닝이나 학습을 위한 소규모 임베딩 모델들은 ~~가난한 GPU~~ 하드웨어 성능이 뒷받침되기 어려워 여전히 컨텍스트 윈도우 크기에 제약이 따르죠. 

기존 LLM 기반 취약점 탐지 방법론의 접근 방식은 실제 취약점과 관련된 코드가 LLM이 분석하는 전체 코드 세그먼트에서 차지하는 비중이 작다는 문제가 있습니다. 다시 말하면, 취약점과 관련 없는 코드가 세그먼트의 대부분을 차지한다는 거죠. 이는 불필요한 토큰 사용량 증가, 모델이 취약점과 관련 없는 코드 패턴 정보에 의존하는 등 여러 문제의 원인이 됩니다. 

오늘 리뷰할 논문에서는 이러한 문제를 해결하고 대규모 코드베이스 취약점 분석에서의 LLM의 한계를 극복하기 위해 CPG 기반의 코드 슬라이싱 기법을 도입합니다.

> Code Property Graphs(CPG): Abstract Syntax Tree(AST), Control Flow Graph(CFG), Program Dependence Graph(PDG)와 같은 다양한 코드의 표현 방식들을 하나의 그래프로 나타내는 방식
> 

# Methodology

## 0. System Overview

아래 그림은 논문에서 제안한, CPG가 제공하는 코드 표현을 활용하여 취약점과 관련된 코드를 슬라이싱해 크기를 줄이고 취약점 존재 여부를 판단하는 시스템의 개요입니다.

![image.png](image.png)

크게 Query Generation, Slice Construction, Code Classification 세 단계로 이루어져 있습니다.

## 1. Query Generation

![image.png](image%201.png)

`LLMxCPG-Q` 모델은 분석할 코드를 입력받아 후술할 CPGQL 쿼리를 생성하도록 `Qwen2.5-Coder-32B Instruct`를 그림과 같이 fine tuning한 모델입니다. 이렇게 학습된 모델은 취약점과 관련된 주요 코드 세그먼트 추출과 코드 슬라이싱 전반에 걸친 쿼리 생성을 자동화하는 데 사용되죠.

## 2. Slice Construction

1에서 생성한 CPGQL 쿼리는 [Joern](https://github.com/joernio/joern)에서 처리됩니다. Joern은 CPG를 생성하고, 이를 Scala 기반의 CPGQL로 쿼리할 수 있도록 지원하는 오픈소스 SAST 도구입니다.

Joern 및 CPGQL 쿼리를 기반으로 아래 세 단계를 통해 코드 슬라이스를 구성합니다.

1. execution path를 중심으로 코드 내 잠재적인 취약점 root cause 식별
2. execution path와 상호작용하는 변수 식별 
3. 1과 2에 영향을 미치는 모든 코드 요소를 찾아 최종 슬라이스 구성

### 2-1) Taint Path 추출

예제 취약점으로는 [CVE-2011-3359](https://github.com/torvalds/linux/commit/c85ce65ecac078ab1a1835c87c4a6319cf74660a) 을 활용합니다.

- 버퍼 길이 `len`에 대한 검증 부족으로 발생하는 buffer overflow 취약점
- `skb_put` 함수 호출 시 `len`만큼 복사하는 과정에서 overflow 트리거

```graphql
val source = cpg.identifier.name("len")
val sink = cpg.call.name("skb_put").where(_.argument.order(2).codeExact("len + ring->frameoffset"))
val execution_paths = sink.reachableByFlows(source)
```

위 CPGQL 쿼리를 통해 source (변수 `len`), sink( `skb_put` 호출), execution paths 등 취약점의 Taint Path를 추출합니다.

### 2-2) Execution Path 상호작용 변수 추출

```graphql
val execution_path_nodes = <실행 경로 추출 쿼리(LLMxCPG-Q에서 생성됨)>
cpg.identifier.filter(id => execution_path_nodes.lineNumber.toSet.intersect(id.lineNumber.l.toSet).size.equals(1))
```

`execution_path_nodes` 와 상호작용하는 노드를 CPGQL 쿼리로 탐색합니다.

### 2-3) 역방향 슬라이싱 및 최종 코드 스니펫 구성

```graphql
실행 경로와 상호작용을 추출하기 위한 쿼리
execution_path_and_interacters.reachableByFlows(cpg.all)
```

이 단계에서는 내부적으로 PDG(Program Dependency Graph)을 사용하는 Joern의 `reachableByFlows` API를 통해 backward slice를 구성합니다. 

backward slice를 통해 루프, 변수 선언 및 초기화 등 취약점의 상호작용, 실행 경로와 관련된 모든 컨텍스트를 가져옵니다.

```graphql
static void dma_rx(struct b43_dmaring *ring, int *slot)
{
	u16 len;
	len = le16_to_cpu(rxhdr->frame_len);
	if (unlikely(len > ring->rx_buffersize)) {
		s32 tmp = len;
		while (1) {
			tmp -= ring->rx_buffersize;
			if (tmp <= 0)
				break;
		}
		goto drop;
	}

	skb_put(skb, len + ring->frameoffset);
	drop:
	return;
}
```

슬라이싱 결과 85줄인 원본 `dma_rx()` 함수가 CVE-2011-3359 취약점의 실행 경로와 의미론적으로 일치하는 18줄의 코드 스니펫으로 압축되었습니다. 

## 3. Code Classification (Vulnerability Detection)

![image.png](image%202.png)

슬라이싱을 통해 압축된 코드는 `QwQ-32B-Preview` 모델을 fine tuning한 `LLMxCPG-D` 모델을 통해 Vuln 또는 Safe로 이진 분류됩니다. `LLMxCPG-D`는 분류 정확도를 위해 취약한 코드와 그렇지 않은 코드 모두에서 슬라이싱되고 라벨링된 코드 스니펫 데이터셋으로 fine tuning된 모델입니다.

이러한 접근법은 취약점 탐지 뿐만 아니라, 패치된 코드 버전에서 취약점 패치 코드 또한 식별하고 강조할 수 있어 이후 모델 학습을 위한 데이터셋 생성 자동화에도 활용될 수 있습니다.

# Performance Analysis

구현된 시스템에 대한 평가는 크게 6가지로 이루어졌는데요. 하나씩 결과를 보면 다음과 같습니다. (자세한 수치 및 비교 표는 원문 참고)

### Query Generation

- 1278개의 테스트 샘플에서 모든 쿼리를 유효하게 생성해 CPGQL을 성공적으로 학습
- 50개 샘플에 대한 human-auditing 결과, 76% 샘플에서 슬라이싱된 코드가 의도된 취약점과 의미론적으로 일치
- 오분류된 25개 샘플
    - 28%는 쿼리가 의미론적으로 정확했으나 LLMxCPG-D가 잘못 분류
    - 40%는 잘못된 CWE 도출
    - 32%는 CWE를 올바르게 식별했지만 중요 컨텍스트 요소 검색 실패

### Code Reduction

- 여러 데이터셋에서 최소 67.87%, 최대 90.93%의 코드 압축률

### Function-level Vulnerability Detection

- 특히 함수 레벨 취약점 탐지에 높은 성능
- 특정 메모리 커럽션 버그 타입([CWE-119](https://cwe.mitre.org/data/definitions/119.html), [CWE-415](https://cwe.mitre.org/data/definitions/415.html), [CWE-416](https://cwe.mitre.org/data/definitions/416.html), [CWE-190](https://cwe.mitre.org/data/definitions/190.html)) 에 대해 높은 탐지율
    
    ![image.png](image%203.png)
    

### Generalizability

- 함수 레벨에서의 Generalizability는 기존의 VulSim, VulBERTA-CNN, VulBERTA-MLP, ReGVD 모델 대비 정확도 20% 향상
- 프로젝트 레벨에서는 복잡성 증가에도 성능 저하 없이 일관된 성능을 유지
- 지식 단절 이후 새로운 취약점 패턴에 대한 일반화 성능 또한 준수해 알려진 CVE뿐만 아니라 취약점의 근본적인 특성을 학습한 것을 확인

### Misclassification Analysis

- CWE-120 (Classic Buffer Overflow) 및 CWE-125 (Out-of-bounds Read)에서의 성능 저하
    - 훈련 데이터셋에 해당 CWE의 샘플 수가 제한적이었기 때문으로, 이는 고품질 데이터셋으로 해결 가능

### Robustness to Code Augmentation

- 의미론적 보존에 대한 높은 성능(주석 영향, 데이터셋 의존성, 코드 변환 노이즈)
- 단, T3 변환(함수 추출)은 모델 성능, 그중에서도 재현율(recall)에 큰 영향을 줌 → 모델이 T3 변환에 다소 민감
    - 함수 추출은 함수의 코드를 분리하여 새로운 함수를 만들거나, 여러 함수의 코드를 하나로 합치는 등의 변환
    - 이러한 변환이 발생하면 CPG 내의 함수 및 데이터의 흐름 또는 다른 함수와의 호출 관계 구조 자체가 재구성되기 때문에 다른 변환에 비해 모델의 성능에 큰 영향을 미침

# Conclusion

LLMxCPG는 종합적으로 CPG 기반의 코드 슬라이싱을 통해 불필요한 코드를 필터링하여 코드 컨텍스트를 최적화합니다. 이러한 최적화는 LLM의 학습 효율을 높이고, 기존 모델의 한계를 극복할 수 있었습니다. 특히 코드 변환에도 의미론적 일관성을 유지하며, 복잡한 코드베이스에 대한 일반화 성능을 크게 향상시켰습니다.

# 마치며

저자가 Limitations 항목에서도 언급했지만, 쿼리 모델(`LLMxCPG-Q`)이 대규모 코드베이스를 전처리할 때의 컴퓨팅 파워 의존성을 클리어하게 해결하기는 어렵고, CPG 자체의 한계인 런타임 속성의 race condition이나 비즈니스 로직 에러와 같은 타입의 취약점 탐지의 어려움도 그대로 계승해 한계가 명확하긴 합니다.

그래도 원본 코드에서 취약점과 직접적으로 관련된 실행 경로를 CPG 기반으로 의미론적으로 동일하게 생성 및 압축해 분석하는 아이디어와 실현이 흥미로웠네요. LLM의 추론 능력과 컨텍스트 크기는 갈수록 발전하고 있지만, 모델 성능 향상만 기다리는 것 보단 이러한 최적화 연구를 고도화하면 더 빠르고 정확한 취약점 탐지 자동화를 앞당길 수 있을 것 같습니다.

그 외 구현 파트는 다루지 못했는데, 공개된 [소스코드](https://github.com/qcri/llmxcpg)와 [모델](https://huggingface.co/collections/QCRI/llmxcpg-6855f80e601774b43eba2d14)을 써본 뒤 리뷰 파트 2로 돌아올 수도 있을 것 같아요.관심 있는 분들은 먼저 살펴보시면 좋을 것 같습니다. 

앞으로 종종 재밌는 논문 리뷰로 또 돌아오겠습니다~