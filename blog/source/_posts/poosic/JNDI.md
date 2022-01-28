---
title: "[해키피디아] JNDI(Java Naming and Directory Interface)"
author: poosic
tags: [poosic, JNDI, log4j]
categories: [Hackypedia]
date: 2022-01-26 14:00:00
cc: true
index_img: /img/hackypedia.png

---

# JNDI(Java Naming and Directory Interface)

오늘은 log4j의 RCE 취약점에서 사용된 기능인 JNDI에 대해 살펴보겠습니다! 

> JNDI(Java Naming and Directory Interface)는 디렉터리 서비스에서 제공하는 데이터 및 객체를 발견(discover)하고 참고(lookup)하기 위한 자바 API다.

위는 위키백과의 정의입니다. 먼저 디렉터리 서비스와 네이밍 서비스에 대해 알아보겠습니다.



## What is Naming Service?

네이밍 서비스는 직역해보면 말 그대로 작명 서비스입니다. 조금 자세히 말하자면, 컴퓨터 이름, 주소, 유저 이름 등의 정보를 특정 공간에 저장해두고 유저나 애플리케이션이 해당 컴퓨터의 데이터를 이용할 수 있게 해주는 서비스입니다.

대표적인 예시로 DNS를 들 수 있겠네요! 우리가 hackyboiz 블로그를 접속하고 싶다면 hackyboiz의 블로그에 대한 IP주소를 입력하지 않아도 우리는[`hackyboiz.github.io`](http://www.hackyboiz.github.io)만 입력해도 간단히 접속할 수 있습니다! 사람이 기억하긴 어려운 IP주소에 자연어로 이루어진 문자열을 네이밍해준 것 입니다.



## What is Directory Service?

> 디렉토리 서비스(directory service, DS)는 컴퓨터 네트워크의 사용자와 네트워크 자원에 대한 정보를 저장하고 조직하는 응용 소프트웨어 (응용 프로그램들의 모임)이다.

즉, 네트워크 환경에서 데이터 베이스에 대해 등록,조회,삭제 등의 관리가 가능하게 해주는 소프트웨어입니다. 여기에서 디렉토리는 이름이 있는 객체에 대한 정보를 가지고 있는 데이터베이스를 말합니다.

네이밍 서비스와 디렉토리 서비스가 어떻게 맞물려 작동할지 감이 오시나요? 그럼, 간단히 JNDI가 무엇인지 다시 살펴볼까요? JNDI는 사용자가 다른 데이터 베이스를 참고하고 싶을 때 네이밍 서비스를 통해 이름 지어진 객체를 찾아 디렉토리 서비스를 통해 데이터베이스의 데이터를 이용하는 것을 도와주는 인터페이스입니다!

## Component and Flow of JNDI

![](JNDI/image.png)

1.네이밍 서비스에서는 각 네트워크 서비스에 맞게 이름과 객체를 바인드하여 또 다른 객체인 컨텍스트 생성

2.프로그램 측에서 특정 개체의 데이터 요청

3.디렉토리 서비스에서 JNDI name을 통해서 요청한 객체를 찾아서 SPI 호출

4.찾은 객체에 맞는 서비스 공급자와 연결하여 데이터 소스 획득

**SPI와 API는 뭐가 다른가요?**

사실 API와 SPI는 다르지 않습니다! 위의 그림처럼 JNDI는 API와 SPI 두 가지로 구분됩니다. API는 프로그램에서 네이밍 서비스나 디렉터리 서비스에 접근하는 데에 호출되어 사용됩니다.

[SPI(Service Provider Interface)](https://blog.seulgi.kim/2014/08/service-provider-interface.html)는 서비스의 확장을 위해 서비스 제공자 측에서 제공하는 API로서, 서버 계층에서 LDAP,DNS,NDS 등의 서비스 제공자들과 연결해주는 역할을 합니다. 정리해보자면 API는 클라이언트 측에서 서비스에 대한 접근을 위해 JNDI에서 제공되어 사용되는 것이고 SPI는 서버 측에서 서비스 공급자에 대한 접근을 위해 서비스 공급자 측에서 제공되어 사용된다는 차이가 있습니다.