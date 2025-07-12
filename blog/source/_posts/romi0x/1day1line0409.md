---
title: "[하루한줄] CVE-2025-0924: WP Activity Log의 Stored Cross-Site Scripting (XSS)취약점"
author: romi0x
tags: [WordPress, WP Activity Log, xss, romi0x]
categories: [1day1line]
date: 2025-04-09 17:00:00
cc: true
index_img: /img/1day1line.png
---

## URL

https://nvd.nist.gov/vuln/detail/CVE-2025-0924

## Target

- WP Activity Log v5.2.2 이하 모든 버전

## Explain

WP Activity Log는 워드프레스 사이트의 사용자 활동을 기록하고 추적하는 기능을 제공하는 인기 플러그인입니다. 관리자 및 보안 담당자들이 시스템 변경 사항과 사용자 이벤트를 파악하는 데 사용됩니다.

취약점은 `class-alert-manager.php`의 `LogCustomEvent()` 함수에서 발생합니다. 이 함수는 외부에서 전달된 사용자 입력값인 `message`를 처리하는데, `sanitize_text_field()`만 적용하고 있어 불완전한 필터링이 이 취약점의 핵심입니다.

```php
$message = isset( $_POST['message'] ) ? sanitize_text_field( wp_unslash( $_POST['message'] ) ) : '';
```

이때 사용된 `sanitize_text_field()`는 `<script>`와 같은 명백한 HTML 태그는 제거하지만, HTML 속성 기반의 페이로드(예: `<img onerror=alert(1)>`)까지는 제거하지 못하는 한계가 있습니다. 이 불완전한 필터링이 취약점의 핵심입니다.

이렇게 처리된 `message` 값은 내부적으로 `TriggerCustomEvent()`를 통해 경고(alert)로 저장되며, 이후 `class-alert.php`의 `GetMessage()` 메서드를 통해 다시 출력됩니다:

```php
public function GetMessage()
{
    return $this->_message;
```

`$this->_message`는 DB에 저장된 사용자 입력이 그대로 담긴 값입니다. 문제는 이 값을 출력하는 시점에서 다음과 같은 방식으로 **escaping 없이 그대로 HTML에 삽입**된다는 것입니다:

```php
echo '<div class="log-entry">' . $alert->GetMessage() . '</div>';
```

이처럼 출력 시점에서 `esc_html()` 또는 `wp_kses()` 등 적절한 escaping 처리가 누락되어 있어, 공격자가 삽입한 스크립트가 로그를 열람하는 관리자 또는 사용자의 브라우저에서 실행될 수 있습니다.


