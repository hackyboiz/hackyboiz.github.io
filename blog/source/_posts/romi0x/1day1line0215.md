---
title: "[하루한줄] CVE-2024-54507: Apple macOS 및 iOS의 Type Confusion"
author: romi0x
tags: [Apple, iOS, Kernel, Type Confusion, romi0x]
categories: [1day1line]
date: 2025-02-15 17:00:00
cc: true
index_img: /img/1day1line.png
---

## URL

https://jprx.io/cve-2024-54507/

## Target

- macOS Sequoia 15.2
- iOS 18.2
- iPadOS 18.2

## Explain

CVE-2024-54507은 Apple macOS 및 iOS 커널에서 발생하는 Type Confusion  취약점입니다. 이 취약점은 BSD sysctl 인터페이스를 통한 잘못된 메모리 접근으로 인해 발생하며, 공격자는 커널 메모리의 일부 내용을 읽을 수 있습니다. `sysctl_udp_log_port` 핸들러 함수의 부적절한 데이터 타입 캐스팅이 이 문제의 원인으로, 2바이트 크기의 `uint16_t` 변수를 4바이트 `int`로 잘못 읽어들이면서 2바이트의 커널 메모리 오버리드(over-read)가 발생합니다.

이 취약점은 `sysctl_udp_log_port` 함수가 `oid_arg1` 변수를 잘못된 데이터 타입으로 캐스팅하면서 발생합니다. 해당 변수는 원래 `uint16_t` 타입으로 정의되어야 하지만, 코드에서는 이를 `int`로 캐스팅하여 2바이트 초과 데이터를 읽어버립니다. 이는 `sysctl_handle_int` 함수 호출 시, 사용자 공간(user space)으로 2바이트의 커널 메모리를 누출할 수 있는 취약점으로 이어집니다.

```
int new_value = *(int *)oidp->oid_arg1;
```

이 코드는 `oid_arg1` 포인터가 `uint16_t`를 가리키고 있음에도 불구하고, 이를 `int`로 캐스팅하여 2바이트 초과 데이터를 읽게 만듭니다.

이 취약점은  `net.inet.udp.log.remote_port_excluded` 라는 sysctl 변수를 `sysctlbyname()` 함수를 사용해 읽을 때 발생합니다. 이 과정에서 sysctl 핸들러 함수가 반환 크기(`len`)를 잘못 처리하거나 메모리 정렬 문제로 인해, 해당 변수 값 바로 뒤에 있는 커널 메모리 2바이트가 추가로 누출될 수 있습니다.

PoC 코드 예시는 다음과 같습니다.

```
void leak() {
    uint64_t val = 0;
    size_t len = sizeof(val);
    sysctlbyname("net.inet.udp.log.remote_port_excluded", &val, &len, NULL, 0);
    printf("leaked: 0x%X 0x%X\n", (val >> 16) & 0x0FF, (val >> 24) & 0x0FF);
}
```

위 코드는 `net.inet.udp.log.remote_port_excluded` 변수를 읽는 과정에서, 커널 메모리 정렬 문제 또는 반환 크기 처리 오류로 인해 해당 변수 뒤의 2바이트 커널 메모리를 누출하는 데 성공합니다.

## Reference

https://nvd.nist.gov/vuln/detail/CVE-2024-54507
https://support.apple.com/en-us/121837