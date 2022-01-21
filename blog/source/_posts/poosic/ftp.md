---
title: "[해키피디아] FTP, Telnet, SSH, SFTP"
author: poosic
tags: [poosic,  ftp, telnet, ssh, sftp]
categories: [Hackypedia]
date: 2022-01-21 14:00:00
cc: true
index_img: /img/hackypedia.png

---



### FTP(File Transfer Protocol)

FTP란 파일 전송 프로토콜의 줄임말 입니다. 번역 그대로 파일 전송을 위해 사용되는 프로토콜입니다. TCP/IP 기반의 프토콜로써, 네트워크에 연결된 컴퓨터 사이에서 파일 교환을 위해 개발이 되었고 인터넷 프로토콜의 5계층 중 응용 계층에 속해 있습니다.

[GUI](https://hackyboiz.github.io/2021/09/17/poosic/Shell/) 이전 [CLI](https://hackyboiz.github.io/2021/09/17/poosic/Shell/) 환경에서도 널리 사용되던 프로토콜이기 때문에 현재도 GUI가 사용이 불가능한 환경에서는 사용중입니다.

FTP는 파일 전송에 앞서 사용자의 이름과 비밀번호를 인증하는 과정이 필요한데 이 과정에서 암호화 되지않고 평문으로 전달되어 보안성이 좋지 않습니다.

### Telnet

Telnet은 인터넷이나 가까운 지역의 컴퓨터들끼리 통신하는 로컬 영역 네트워크에 사용되는 네트워크 프로토콜입니다. 가상 단말 기능, 즉, 원격으로 컴퓨터를 사용하기 위해 개발되었습니다.

사용자에게 호스트와 클라이언트간 시스템이 다를 경우에도 데이터를 변환해 원활한 통신을 가능하게 해주는 NVT(Network Virtual Terminal)라는 가상의 터미널을 제공합니다.

현재는 FTP와 동일하게 인증 과정에서 암호화를 하지 않아 패킷 스니핑 등의 위험이 있어 사용하지 않고 있습니다.

### SSH(Secure Shell)

SSH는 기존의 원격 통신을 위해 개발된 Telnet, rsh 등을 대체하기 위해 설계된 프로토콜입니다.

이름에도 Secure이 들어가는 것 처럼 기존 프로토콜의 보안상 문제점을 보완하여 더 강도 높은 인증과 암호화 기법을 사용하였습니다. 그래서 Telnet이나 FTP와 다르게 비밀번호 정보가 노출되더라도 복호화를 할 수 없다면 비밀번호를 유추하는 것이 불가능합니다.

### SFTP(Secure File Transfer Protocol)

SFTP는 SSH를 이용해 FTP의 보안 문제를 보완한 프로토콜입니다. 단순히 파일을 전송하던 FTP와는 달리 파일을 암호화해 SSH 프로토콜을 이용하여 전송합니다. SSH의 공개키를 사용해 암호화하고 개인키를 이용해 검증하는 식으로 인증을 진행합니다.

다만 보안을 위해 암호화를 사용하기 때문에 데이터 교환 속도가 느리며 기존 FTP에 비해 SSH를 추가적으로 사용하기 때문에 SSH 인증을 위한 키 관리가 별도로 필요합니다.
