---
title: "[해키피디아] ACL(Access Control List)"
author: y00n_nms
tags: [y00n_nms, acl, ace, dacl, sacl]
categories: [Hackypedia]
date: 2021-09-08 14:00:00
cc: true
index_img: /img/hackypedia.png
---

Access Control(접근 제어)은 적절한 권한을 가진 사람만이 특정 시스템이나 파일에 접근할 수 있도록 제어합니다. 따라서 권한이 없는 사람은 함부로 해당 파일에 접근하여 읽기, 쓰기, 실행 등을 수행할 수 없습니다.

그리고 파일 및 디렉터리에 대한 접근은 **ACL(접근 제어 목록, Access Control List)**을 통해 관리되어 인증된 사용자만 접근할 수 있습니다.

하루 한줄의 권한 상승 취약점을 보면 ACL의 구성이 미흡하거나, ACL 검사를 수행하지 않아 발생하는 경우가 종종 있습니다.

![image](access-control-list/image.png)

[[하루한줄] CVE-2021-21551: 수억 대의 Dell PC에 영향을 주는 권한 상승 취약점](https://hackyboiz.github.io/2021/05/07/l0ch/2021-05-07/)

![image1](access-control-list/image1.png)

[[하루한줄] CVE-2021-23874: McAfee COM-objects 권한 상승 취약점](https://hackyboiz.github.io/2021/05/20/idioth/2021-05-20/)

**ACE(접근 제어 항목, Access Control Entry)**는 ACL의 항목으로 0개 이상의 ACE가 있을 수 있습니다. 각 ACE는 SID(보안 식별자), 접근 권한을 지정하는 마스크, 형식을 나타내는 플래그, 개체가 ACE를 상속할 수 있는지 여부를 결정하는 비트 플래그를 포함하며 보안 주체를 식별하고 해당 보안 주체에 대해 허용, 거부, 또는 감사된 접근 권한을 지정합니다.

ACE에는 다음 세 가지 형식이 있습니다.

- 액세스 거부 ACE: DACL에서 보안 주체에 대한 접근 권한을 거부하는데 사용됩니다.
- 액세스 허용 ACE: DACL에서 보안 주체에 대한 접근 권한을 허용하는데 사용됩니다.
- 시스템 감사 ACE: SACL에서 보안 주체가 지정한 접근 권한을 수행하려고 할 때 감사 레코드를 생성하는데 사용됩니다.

**DACL(임의 접근 제어 목록, Discretionary Access Control List)**은 보안 개체에 대해 접근이 허용되거나 거부된 사용자, 사용자 그룹, 또는 프로세스를 식별합니다. 접근을 요청하면 DACL에 있는 ACE를 확인하여 권한을 부여합니다.

이때 개체에 DACL이 없는 경우에는 모든 사용자가 접근 권한을 가지고, DACL이 있지만 ACE가 없을 경우에는 모든 접근을 거부합니다. 그 외 경우에는 ACE에서 허용하는 접근만 허용합니다. 주의해야 할 것은 ACE를 순서대로 읽기 때문에 액세스 거부 ACE와 허용 ACE를 동시에 명시할 때 거부 ACE부터 명시해야 합니다.

**SACL(시스템 접근 제어 목록, System Access Control List)**는 보안 개체에 접근하려는 시도를 기록합니다. SACL의 ACE는 접근 시도가 실패하거나 성공할 때 감사 레코드를 생성합니다.