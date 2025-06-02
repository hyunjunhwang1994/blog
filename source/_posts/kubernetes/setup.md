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

### 서버 요구사항
- 최소 2대 이상의 서버 (마스터 노드 1대, 워커 노드 1대 이상)
- 각 서버당 최소 2GB RAM, 2 CPU 코어 
- 이 글에서는 클라우드 환경의 물리 서버 4대를 기준으로 구성


### 방화벽 설정
쿠버네티스 클러스터는 노드 간 다양한 포트를 통해 통신해야 한다. 
설정을 단순화하기 위해 ufw 방화벽을 비활성화 한다.

주의: 프로덕션 환경에서는 필요한 포트만 선별적으로 열어야 합니다.

<br/>

```shell
sudo ufw status
sudo ufw disable
```

## 네트워크 설정
### 커널 모듈 로드 설정
쿠버네티스 네트워킹에 필요한 커널 모듈을 부팅 시 자동으로 로드하도록 설정한다.
```shell
sudo -i
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

#### 각 모듈의 역할

| overlay | br_netfilter |
|---------|--------------|
|서로 다른 호스트의 파드 간 통신을 위한 오버레이 네트워크 생성| 브리지를 통과하는 트래픽에 iptables 규칙 적용 가능|

##### overlay
- overlay는 리눅스 커널의 네트워크 드라이버를 가리킨다. 
- overlay는 서로 다른 호스트에 존재하는 파드 간의 네트워크 연결을 가능하게 하는 기술이다.
- 즉, overlay를 활용하면 여러 개의 독립적인 네트워크 레이어를 겹쳐서 하나로 연결된 네트워크를 생성한다. 
- 즉 overlay를 활용해서 서로 다른 호스트에 존재하는 파드가 동일한 네트워크에 존재하는 것처럼 통신할 수 있게한다. 
- 따라서 overlay를 입력하면 시스템 부팅 시 overlay 네트워크 드라이버를 로드하도록 설정한다.

##### br_netfilter
- br_netfilter는 네트워크 패킷 처리 관련 모듈로써, 
- iptables/netfilter 규칙이 적용되게 한다. 
- 즉, 컨테이너와 호스트 간의 인터페이스 등에서 발생하는 트래픽에 대해 규칙을 적용해 트래픽을 관리한다는 의미이다.

#### 모듈 즉시 로드

```shell
sudo modprobe overlay
sudo modprobe br_netfilter
```

### 네트워크 파라미터 설정
쿠버네티스 네트워킹을 위한 커널 파라미터를 설정한다.

```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

#### 각 설정의 의미

|net.bridge.bridge-nf-call-iptables|브리지를 통과하는 IPv4 트래픽에 iptables 규칙 적용|
|---|---|
|net.bridge.bridge-nf-call-ip6tables = 1|브리지를 통과하는 IPv6 트래픽에 iptables 규칙 적용|
|net.ipv4.ip_forward|IPv4 패킷 포워딩 활성화 (라우터 기능) |

#### 설정 적용
재부팅 없이 변경된 sysctl 설정을 즉시 적용한다.
```shell
sudo sysctl --system
```

중요: 위의 모든 설정은 클러스터에 참여할 모든 서버(마스터 노드, 워커 노드)에 동일하게 적용해야 한다.


