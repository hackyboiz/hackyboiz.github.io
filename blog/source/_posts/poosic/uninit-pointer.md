---
title: "[해키피디아] Uninitialized Pointer"
author: poosic
tags: [poosic, uninitialized, pointer]
categories: [Hackypedia]
date: 2022-02-25 14:00:00
cc: true
index_img: /img/hackypedia.png
---

Uninitialized Pointer는 뜻 그대로 초기화되지 않은 포인터를 말합니다. 해당 포인터의 존재로 인해 취약점이 발생할 수 있습니다.

거의 모든 곳에서 포인터에 대해 다룰 때 `NULL`값으로 초기화를 하고 사용하기를 권장합니다. 포인터 변수는 선언이 될 때 초기화가 되지 않으면 무작위로 값이 들어가게 됩니다. 하지만 이 값이 우연히 프로그램에서 중요한 데이터 부분을 가리킨다면 해당 데이터를 덮어쓰거나 유출이 될 수 있습니다. 그로 인해서 심하면 전체 시스템이 다운되는 경우도 발생할 수 있습니다.한번 예제를 통해 살펴보도록 하겠습니다.

```c
#include<stdio.h>

int main()
{
	int* p;
	*p = 123;

	printf("%d", *p);

	return 0;
}
```

위 코드는 포인터 변수 `p`를 초기화시키지 않고 `p`가 가리키는 곳에 값을 넣고 이를 출력하려는 코드입니다.  만약 위 코드와 같은 맥락으로 코드가 작성되거나 실행된다면 프로그램이 의도치 않은 곳에 값이 작성되어 버리게 됩니다.

### How to defense?

보통은 OS상에서 위와 같은 잘못된 메모리 접근의 경우에는 차단하게 됩니다.

```c
//ex1
int main()
{
	int* p = NULL;
}

//ex2
int main()
{
	int* p = (int*)calloc(1,sizeof(int));
}
```

그렇지만  포인터 변수를 사용 시에 `ex1`처럼 `NULL`으로 초기화해 주거나 `ex2`와 같이 `calloc()`, `malloc()` 등 동적할당을 이용해 초기화해 미리 취약점을 방지해 주어야합니다.

