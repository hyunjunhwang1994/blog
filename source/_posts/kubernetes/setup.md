---
title: Kubernetes 환경 구축
date: 2025-05-31 19:30:00
categories: k8s
tags:
  - Docker
  - DevOps
  - kubernetes
---

## 사전 준비 사항

### 서버
- 실제 다수의 물리 서버
- 혹은 가상 머신으로 이루어진 다수의 서버
- 이 글에서는 클라우드에 올라간 각각의 물리서버 4대를 기준으로 구현합니다.


### 방화벽 확인.
쿠버네티스 클러스터를 구성하기 위한 노드는 각 노드별로 포트 통신이 원활해야 한다. 따라서 이를 위해 기존에 사용하던 ufw를 정지시킨다.
```shell
sudo ufw status
sudo utf disable
```

### 네트워크 설정
먼저, IPv4를 포워딩하여 iptables가 연결된 트래픽을 볼 수있도록 처리해보자.
```shell
ubuntu@ubuntu22-arm-1-50gb:~$ sudo -i
root@ubuntu22-arm-1-50gb:~# cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
overlay
br_netfilter
```
overlay는 리눅스 커널의 네트워크 드라이버를 가리킨다. overlay는 서로 다른 호스트에 존재하는 파드 간의 네트워크 연결을 가능하게 하는 기술이다.
즉, overlay를 활용하면 여러 개의 독립적인 네트워크 레이어를 겹쳐서 하나로 연결된 네트워크를 생성한다.
즉 overlay를 활용해서 서로 다른 호스트에 존재하는 파드가 동일한 네트워크에 존재하는 것처럼 통신할 수 있게한다. 따라서 overlay를 입력하면
시스템 부팅 시 overlay 네트워크 드라이버를 로드하도록 설정한다.

br_netfilter는 네트워크 패킷 처리 관련 모듈로써, iptables/netfilter 규칙이 적용되게 한다. 즉, 컨테이너와 호스트 간의
인터페이스 등에서 발생하는 트래픽에 대해 규칙을 적용해 트래픽을 관리한다는 의미이다.

아래 명령어로 해당 모듈을 로드한다.
```shell
root@ubuntu22-arm-1-50gb:~# sudo modprobe overlay
root@ubuntu22-arm-1-50gb:~# sudo modprobe br_netfilter
```


```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

아래 명령어를 입려하면, 재부팅하지 않고 sysctl 매개변수를 적용할 수 있다.

```shell
sudo sysctl --system
```

위 과정을 진행하고자 하는 서버에 전부 적용해 준다. 필자의 경우 물리 서버 4대
### 설치


