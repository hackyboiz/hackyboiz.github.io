---
title: "[해키피디아] Dynamic Link Library"
author: poosic
tags: [poosic, dynamic link library, DLL]
categories: [Hackypedia]
date: 2021-09-03 14:00:00
cc: true
index_img: /img/hackypedia.png

---

![Untitled.jpg](Dynamic-Link-Library/image1.jpg)

**DLL(Dynamic Link Library)**는 마이크로소프트 사에서 구현한 동적 라이브러리입니다. DLL파일의 확장자 명은 `.dll`이고 다른 프로그램들에서 사용 가능한 바이너리 코드들을 내포하고 있습니다.  **Static Link Library**는 컴파일 시점에 라이브러리가 실행 파일의 일부가 되어 프로그램이 라이브러리 내에 있는 함수를 실행할 수 있도록 하지만, DLL은 프로그램이 해당 함수의 위치정보만 가지고 라이브러리 내에 있는 함수를 실행할 수 있도록 합니다.

### 사용방식

DLL의 사용방식에는 **묵시적 링킹(Implicit Linking)**과 **명시적 링킹(Explicit Linking)**,이 두 가지 방식으로 사용됩니다. **묵시적 링킹**은 실행파일 자체에 DLL 사용 정보를 등록해 프로그램 실행시 OS에서 사용 함수들을 프로그램이 사용할 수 있도록 초기화 시켜 주는 방식입니다. **명시적 링킹**은 프로그램이 실행 중일 때 `LoadLibrary()`, `GetProcAddress()`, `FreeLibrary()` 와 같은 API를 이용해 DLL 파일이 있는지 검사하고 원하는 함수만 불러 사용하는 방식입니다..

### 장점

첫 번째 장점은 **리소스를 절약**할 수 있다는 것입니다. Static Link Library를 사용할 경우 해당 라이브러리를 사용하는 프로그램마다 라이브러리가 복제되어 메모리를 차지하게 됩니다. 하지만 DLL을 사용할 경우 DLL 파일 하나만 메모리를 차지하기 때문에 리소스를 훨씬 절약할 수 있습니다.

두 번째 장점은 **유지/보수가 쉽다**는 점입니다. 만약 라이브러리 내의 함수를 업데이트해야 하는 경우 Static Link Library는 배포한 프로그램마다 라이브러리 내용을 수정해줘야 하지만 DLL을 사용할 경우에는 수정한 DLL파일 하나만을 수정하면 되고 프로그램과 라이브러리를 다시 연결하지 않아도 되기 때문에 유지/보수가 쉽습니다.

세 번째 장점은 **다중 언어에 사용이 가능하다**는 점입니다. DLL 파일은 이미 컴파일된 바이너리 코드를 가지고 있기 때문에 다른 언어에서도 사용이 가능합니다. 즉, 델파이나 C# 등에서 Win32 API로 제작된 DLL을 사용 가능하다는 것입니다.