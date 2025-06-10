---
title: 홈 서버 구축기 01 - 우분투 설치
date: 2025-06-09 19:30:00
categories: home-server
tags:
  - Home Server
  - Server
  - AMD CPU Server
---

# 홈 서버를 구현하게 된 이유
풀스택 개발자로서, 서버 및 리눅스 시스템을 다루는 것을 좋아하고 실제로 업무에서도 시스템 엔지니어의 영역까지도 겸하고 있다. <br/>
기존에는 오라클 클라우드의 강력한 프리티어를 활용해서 Ampere A1 cpu 1, 4대 인스턴스를 블로그 및 학습 용도로 사용하곤 했는데, <br/>
저렴한 비용에 더 좋은 성능과 많은 대수의 가상화를 하기 위해, 취미인 서버를 공부하기 위해 서버 컴퓨터를 구매하게 되었다. <br/>

## 고려 사항.
- CPU & Memory
  - 가상화를 8대 이상 할 수 있어야 함.
- Graphic Card
  - 필요 없다.
- 일반 가정의 네트워크 환경 (유동 IP)
- 24시간 서비스
  - 저소음
  - 전기세

해서 아래와 같은 스펙으로 간단하게 맞추게 되었다.
![01](/images/HomeServer01-01.png)




영롱 그잡채...
{% asset_img 01-02.jpg %}
![02](/images/HomeServer01-02.jpg)


우분투가 담긴 usb를 PC에 연결해 OS 설치 작업했으며, 필자는 CLI 환경만 설치했다.

설레는 우분투 설치 과정
![03](/images/HomeServer01-03.jpg)


일부로 WIFI를 지원하는 메인 보드를 사서 편하게 설치했다.
![04](/images/HomeServer01-04.jpg)


스토리지 설정은 아래처럼 진행하였다.
![05](/images/HomeServer01-05.jpg)


마지막쯤에, 꼭 openssh 설치에 체크해주자.