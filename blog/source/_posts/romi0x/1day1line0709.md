---
title: "[하루한줄] CVE-2025-6019 : Fedora 및 SUSE Linux에서 udisks, libblockdev와 관련된 Local Privilege Escalation"
author: romi0x
tags: [Fedora, LPE, libblockdev, udisks, Polkit, romi0x]
categories: [1day1line]
date: 2025-07-09 17:00:00
cc: true
index_img: /img/1day1line.png
---

## URL
- https://blog.securelayer7.net/cve-2025-6019-local-privilege-escalation/

## Target
- 대부분의 Linux 배포판에 기본적으로 포함된 udisks 데몬
- Fedora, SUSE, 및 udisks2와 libblockdev를 사용하는  시스템

## Explain
CVE-2025-6019는 Linux 시스템의 디스크 관리 데몬인 udisks와 그 하위 구성 요소인 libblockdev 라이브러리에서 발견된 로컬 권한 상승(Local Privilege Escalation, LPE) 취약점입니다. 이 취약점을 통해 일반 사용자도 루트 권한으로 명령을 실행할 수 있어 문제가 됩니다.

여기서 나오는 libblockdev는 리눅스 시스템에서 블록 장치(디스크 등)를 다루기 위한 C 라이브러리로, 파티션 생성, 파일 시스템 포맷, 마운트 등의 작업을 수행합니다. 일반 사용자는 직접 접근할 수 없고, 주로 udisks 같은 고수준 데몬을 통해 간접적으로 사용됩니다. 이처럼 libblockdev는 스토리지 작업을 루트 권한으로 실행하는 인터페이스이기 때문에, 보안상 매우 중요한 구성요소입니다.

### CVE Flow

udisksd는 D-Bus를 통해 디스크 마운트, 포맷 등의 작업을 처리하며, 이 과정에서 polkit을 통해 권한을 검증합니다. 그러나 문제 버전에서는 사용자 UID를 제대로 확인하지 않고, 단순히 allow_active 상태인지 여부만 검사했습니다.

이로 인해 allow_active 권한만 가진 비루트 사용자도 루트 권한이 필요한 작업을 udisksd를 통해 실행할 수 있게 되며, `udisks_daemon_handle_mount → polkit_check → blkdev_mount` 경로를 따라 루트 권한으로 마운트 작업이 수행됩니다.

libblockdev가 입력을 별다른 검증 없이, 외부에서 들어온 요청을 루트 권한으로 그대로 실행해 LPE 취약점이 발생했습니다.

### Patch Code

- Unpatched Code

    ```
    if (caller_in_allow_active_group()) { 
    
        return ALLOW_MOUNT; 
    
    }
    ```

  caller_in_allow_active_group() 함수는 polkit의 정책을 통해 "active" 사용자(일반적으로 물리적으로 시스템 앞에 있는 사용자)인지 여부만 확인합니다. 실제 호출자가 루트인지 아닌지를 확인하지 않기 때문에, 권한 없는 사용자도 루트 권한 작업을 실행할 수 있게 됩니다.


- Patched Code

    ```
    Patched: 
    
    if (caller_in_allow_active_group() && caller_uid == 0) { 
    
        return ALLOW_MOUNT; 
    
    }
    ```

  패치 후 에는 "allow_active" 조건과 UID가 0(root)인 경우에만 디스크 마운트를 허용하도록 변경되었습니다. 이 조건을 통해 루트가 아닌 사용자의 요청을 차단하고, 루트 사용자만이 udisksd를 통해 작업을 실행할 수 있도록 했습니다.

## Reference
- https://ubuntu.com/blog/udisks-libblockdev-lpe-vulnerability-fixes-available
