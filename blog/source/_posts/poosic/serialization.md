---
title: "[해키피디아] Serialization"
author: poosic
tags: [poosic, serialization , deserialization]
categories: [Hackypedia]
date: 2021-10-29 14:00:00
cc: true
index_img: /img/hackypedia.png

---

![Untitled.jpg](serialization/image1.jpg)

Serialization은 직렬화라는 뜻을 가지고 있습니다. 데이터 개체를 메모리, 데이터 베이스 등 다른 곳에 전송해 사용하기 위해 나중에 재구성할 수 있는 일련의 바이트 포맷으로 변환하는 프로세스를 일컬어 말합니다.

Serialization과 반대 프로세스는 Deserialization이라고 불립니다. 역직렬화라는 의미를 가지고 있습니다. 위에서 Serialization이 재구성 가능한 일련의 바이트 포맷으로 변환한다고 언급했었는데, Deserialization은 Serialization의 결과인 일련의 바이트를 다시 직렬화 전의 데이터 객체로 변환하는 프로세스를 일컬어 말합니다.

### 왜 사용할까요?

Serialization를 사용하는 이유는 데이터 전송과 저장을 위함입니다. 우리가 사용하는 데이터의 형식에는 크게 값 형식 데이터(Value Type)와 참조 형식 데이터(Reference Type), 이 2가지가 있습니다. 값 형식 데이터는 말 그대로 우리가 스택에 쌓여 직접 접근이 가능한 값들을 의미하는 변수가 값을 담는 데이터 형식입니다. 참조 형식 데이터는 힙에 할당되어 변수가 값을 담는 대신 값이 있는 곳의 주소를 담는 데이터 형식입니다.

PC마다 환경이 다르기 때문에 참조 형식 데이터 즉 주소 값을 준다고 해서 해당 메모리 주소에 같은 데이터가 있다는 보장을 하기는 어렵습니다. 그래서 데이터 통신을 위해서는 이런 참조 형식 데이터가 아닌 값 형식 데이터만을 이용해 통신이 가능합니다. 그렇기 때문에 참조 형식 데이터를 이용해 전송하기 위해서는 모두 값 형식 데이터로 바꾸어주는 Serialization이 필요로 하게 되는 것입니다.