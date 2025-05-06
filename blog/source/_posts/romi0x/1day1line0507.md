---
title: "[하루한줄] CVE-2025-29774, CVE-2025-29775 : xml-crypto의 XML Signature Wrapping 취약점"
author: romi0x
tags: [WorkOS, xml-crypto, XML Signature Wrapping, romi0x]
categories: [1day1line]
date: 2025-05-07 17:00:00
cc: true
index_img: /img/1day1line.png
---

## URL

https://workos.com/blog/samlstorm

## Target

- xml-crypto 라이브러리 (버전 <= 6.0.0)

## Explain
이번 취약점은 XML 디지털 서명 검증 로직의 허점을 악용하여 SAML 응답 또는 XML 서명 기반 인증 메커니즘을 우회할 수 있는 취약점입니다. 해당 취약점은 `xml-crypto` 라이브러리의 서명 검증 방식의 구조적 문제에서 비롯되었습니다. CVE-2025-29774와 CVE-2025-29775 두 건으로 나뉘어 공개됐습니다.

**CVE-2025-29774**는 "SignedInfo 노드 중복"을 이용한 취약점입니다. 정상적인 서명된 XML 문서는 `<Signature>` 노드 내에 단 하나의 `<SignedInfo>` 노드만 존재해야 합니다. 그러나 `xml-crypto`의 취약 버전은 다수의 `<SignedInfo>` 노드가 있을 경우, 공격자가 삽입한 가짜 `<SignedInfo>`가 아닌 정상적인 것을 대상으로 서명 검증을 수행합니다. 이로 인해 공격자는 하나는 유효한 SignedInfo로 남겨두고, 다른 하나는 권한 상승이나 사용자 변조와 같은 악의적 조작을 포함해 서명을 우회할 수 있습니다. 실제 공격 예시에서는 `<Signature>` 블록 내에 두 개의 `<SignedInfo>`를 삽입하고, 하나에 위조된 Digest 값을 포함해 인증 우회를 시도합니다.

**CVE-2025-29775**는 DigestValue 조작을 기반으로 합니다. 이 취약점은 DigestValue에 HTML 주석 (`<!-- -->`)을 삽입하는 방식으로 동작합니다. 정상적인 DigestValue는 Base64로 인코딩된 순수 데이터여야 하지만, xml-crypto의 검증 로직은 이 값을 노드 기준으로 파싱하지 않고, 문자열 기준으로 처리해버립니다. 이 때문에 공격자는 다음과 같이 주석과 위조된 값을 삽입할 수 있습니다.

```xml
<DigestValue>
    <!--TBlYWE0ZWM4ODI1NjliYzE3NmViN2E1OTlkOGDhhNmI=-->
    c7RuVDYo83z2su5uk0Nla8DXcXvKYKgf7tZklJxL/LZ=
</DigestValue>
```

검증 로직은 주석을 무시하고 실제 DigestValue만을 검증하기 때문에 서명이 유효하다고 판단되지만, 해석하는 애플리케이션은 주석과 값을 결합해 잘못된 데이터로 처리하게 됩니다. 이로써 공격자는 사용자의 신원 정보나 권한 정보를 교묘히 변경하면서도 서명 검증을 우회할 수 있습니다.

공격자는 이 취약점을 악용해 인증 우회 및 권한 상승 공격을 수행할 수 있습니다. 이는 `xml-crypto`를 기반으로 SAML SSO를 구현한 모든 서비스에 광범위한 영향을 미치며, 특히 중요한 신원 정보나 권한 정보가 담긴 XML 메시지를 조작해도 여전히 서명 검증을 통과하는 것이 문제입니다.

이 취약점은 `xml-crypto` 라이브러리 6.0.1 버전에서 패치되었습니다. 모든 6.0.0 이하 버전은 영향을 받습니다.
1. **SignedInfo 노드 중복 여부 (CVE-2025-29774 탐지법)**

   SAML 응답이나 XML 문서 내에서 `<Signature>` 안에 `<SignedInfo>` 노드가 두 개 이상 있는지 확인해야 합니다. 정상 문서는 반드시 하나만 있어야 합니다.

    ```xml
    <Signature>
        <SomeNode>
          <SignedInfo>
            <Reference URI="somefakereference">
              <DigestValue>forgeddigestvalue</DigestValue>
            </Reference>
          </SignedInfo>
        </SomeNode>
        <SignedInfo>
            <Reference URI="realsignedreference">
              <DigestValue>realdigestvalue</DigestValue>
            </Reference>
        </SignedInfo>
    </Signature>
    ```

   탐지용 JavaScript 코드는 다음과 같습니다.

    ```jsx
    const signedInfoNodes = xpath.select(".//*[local-name(.)='SignedInfo']", signatureNode);
    if (signedInfoNodes.length > 1) {
      // Compromise detected
    }
    ```

2. **DigestValue 내 주석 존재 여부 (CVE-2025-29775 탐지법)**

   `<DigestValue>` 블록 안에 `<!-- -->` 형태의 주석이 삽입된 경우는 정상적인 XML 서명에서는 절대 있어선 안 됩니다. 발견 시 조작된 메시지로 간주해야 합니다.

    ```xml
    <DigestValue>
        <!-- malicious comment -->
        validDigestValue==
    </DigestValue>
    ```

   탐지용 JavaScript 코드는 다음과 같습니다:

    ```jsx
    const digestValues = xpath.select("//*[local-name()='DigestValue'][count(node()) > 1]", decryptedDocument);
    if (digestValues.length > 0) {
      // Compromise detected
    }
    ```


## Reference

https://github.com/node-saml/xml-crypto/security/advisories/GHSA-x3m8-899r-f7c3

https://github.com/node-saml/xml-crypto/security/advisories/GHSA-9p8x-f768-wp2g


