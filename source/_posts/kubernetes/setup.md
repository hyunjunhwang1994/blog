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

### Docker 설치

#### 1단계: 시스템 업데이트 및 필수 패키지 설치

```shell
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
```

- apt-get update: 패키지 목록을 최신 상태로 업데이트 
- ca-certificates: HTTPS 연결을 위한 인증서 관리 
- curl: 웹에서 파일을 다운로드하는 도구 
- gnupg: GPG 암호화/서명 검증을 위한 도구 
- lsb-release: Linux 배포판 정보를 확인하는 도구

#### 2단계: Docker 공식 GPG 키 추가

```shell
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

- /etc/apt/keyrings 디렉토리 생성 (권한 755)
- Docker 공식 GPG 키를 다운로드하여 저장
- 이 키는 패키지의 무결성을 검증하는 데 사용된다.

#### 3단계: Docker 저장소 추가
```shell
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

- Docker 공식 저장소를 APT 소스 목록에 추가
- $(dpkg --print-architecture): 현재 시스템의 아키텍처 (amd64, arm64 등)
- $(lsb_release -cs): Ubuntu 버전 코드명 (jammy, focal 등)
- signed-by: 앞서 추가한 GPG 키로 서명 검증

#### 4단계: Docker 설치
```shell
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- docker-ce: Docker Community Edition 엔진 
- docker-ce-cli: Docker 명령줄 인터페이스 
- containerd.io: 컨테이너 런타임 
- docker-buildx-plugin: 멀티 플랫폼 이미지 빌드 플러그인 
- docker-compose-plugin: Docker Compose V2 플러그인

#### 5단계: Docker 서비스 확인 및 사용자 권한 설정
````shell
systemctl status docker
sudo usermod -aG docker $USER
````

- systemctl status docker: Docker 서비스 실행 상태 확인
- usermod -aG docker $USER: 현재 사용자를 docker 그룹에 추가 (sudo 없이 docker 명령 사용 가능)

#### 6단계: 설치 확인
```shell
docker version
docker compose version
```

- Docker 클라이언트와 서버 버전 정보 출력
- Docker compose 버전 정보 출력
- 설치가 제대로 되었는지 최종 확인

참고: usermod 명령 후에는 로그아웃 후 다시 로그인하거나 newgrp docker 명령을 실행해야 권한이 적용된다.


### 서버 요구사항
- 최소 2대 이상의 서버 (마스터 노드 1대, 워커 노드 1대 이상)
- 각 서버당 최소 2GB RAM, 2 CPU 코어 
- 이 글에서는 클라우드 환경의 물리 서버 4대를 기준으로 구성


### 방화벽 설정
쿠버네티스 클러스터는 노드 간 다양한 포트를 통해 통신해야 한다. 
설정을 단순화하기 위해 ufw 방화벽을 비활성화 한다.

주의: 프로덕션 환경에서는 필요한 포트만 선별적으로 열어야 한다.

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

##### overlay
- 서로 다른 호스트의 파드 간 통신을 위한 오버레이 네트워크 생성
- overlay는 리눅스 커널의 네트워크 드라이버를 가리킨다. 
- overlay는 서로 다른 호스트에 존재하는 파드 간의 네트워크 연결을 가능하게 하는 기술이다.
- 즉, overlay를 활용하면 여러 개의 독립적인 네트워크 레이어를 겹쳐서 하나로 연결된 네트워크를 생성한다. 
- 즉 overlay를 활용해서 서로 다른 호스트에 존재하는 파드가 동일한 네트워크에 존재하는 것처럼 통신할 수 있게한다. 
- 따라서 overlay를 입력하면 시스템 부팅 시 overlay 네트워크 드라이버를 로드하도록 설정한다.

##### br_netfilter
- 브리지를 통과하는 트래픽에 iptables 규칙 적용 가능
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
net.bridge.bridge-nf-call-iptables = 1 : 브리지를 통과하는 IPv4 트래픽에 iptables 규칙 적용
net.bridge.bridge-nf-call-ip6tables = 1 : 브리지를 통과하는 IPv6 트래픽에 iptables 규칙 적용
net.ipv4.ip_forward = 1 : IPv4 패킷 포워딩 활성화 (라우터 기능)


#### 설정 적용
재부팅 없이 변경된 sysctl 설정을 즉시 적용한다.
```shell
sudo sysctl --system
```

중요: 위의 모든 설정은 클러스터에 참여할 모든 서버(마스터 노드, 워커 노드)에 동일하게 적용해야 한다.

## containerd 설정
- 앞서 containerd는 도커 설치 과정에서 설치함.
- 이 containerd는 도커 관련 작업을 할 때 사용하는데, containerd를 쿠버네티스에서 컨테이너 런타임으로 사용할 수 있도록 설정을 변경한다.


```shell
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

```shell
sudo vi /etc/containerd/config.toml
#/runc.options  검색
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            .
            .
            SystemdCgroup = true # true로 변경
```

```shell
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
● containerd.service - containerd container runtime
     Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-06-04 12:33:38 UTC; 11s ago
```

마찬가지로 나머지 서버에 전부 적용해준다.

## swap 메모리 비활성화
아래 명령어로 스왑 메모리가 할당되어 있다면 비활성화 해야함. Swap 영역이 0이라면 생략
```shell
free -h
               total        used        free      shared  buff/cache   available
Mem:           5.8Gi       454Mi       2.8Gi       5.0Mi       2.6Gi       5.1Gi
Swap:             0B          0B          0B
```

## 쿠버네티스 설치

### 모든 노드에 쿠버네티스 설치
```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

```shell
sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```shell
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```shell
sudo apt-get update
sudo apt-get install -y kubelet=1.29.0-1.1 kubeadm=1.29.0-1.1 kubectl=1.29.0-1.1
```

```shell
sudo -i
kubelet --version
kubeadm version
kubectl version --output=yaml
```

마찬가지로 모든 서버에 설치한다.

## 노드 설정

### 마스터 노드 설정
- 일단은 서버 중 하나의 서버를 마스터 노드로 설정하면 되며,
- 필자의 경우 4대의 서버 중 첫 번째 서버를 마스터 노드로 설정하려고 한다.

```shell
ubuntu@ubuntu22-arm-1-50gb:~$ kubeadm certs check-expiration
CERTIFICATE                          EXPIRES   RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
!MISSING! admin.conf                                                                   
!MISSING! apiserver                                                                    
!MISSING! apiserver-etcd-client                                                        
!MISSING! apiserver-kubelet-client                                                     
!MISSING! controller-manager.conf                                                      
!MISSING! etcd-healthcheck-client                                                      
!MISSING! etcd-peer                                                                    
!MISSING! etcd-server                                                                  
!MISSING! front-proxy-client                                                           
!MISSING! scheduler.conf                                                               
!MISSING! super-admin.conf                                                             

CERTIFICATE AUTHORITY      EXPIRES   RESIDUAL TIME   EXTERNALLY MANAGED
!MISSING! ca                                         
!MISSING! etcd-ca                                    
!MISSING! front-proxy-ca  
```
위는 쿠버네티스 인증서 상태를 확인하는 명렁어이며 인증 관련 설정이 하나도 되지 않은 것을 확인할 수 있다.

아래 명령어를 사용하여, kubeadm이 사용할 수 있는 이미지 리스트를 출력한다.
```shell
ubuntu@ubuntu22-arm-1-50gb:~$ kubeadm config images list
I0604 12:59:33.050775  123892 version.go:256] remote version is much newer: v1.33.1; falling back to: stable-1.29
registry.k8s.io/kube-apiserver:v1.29.15
registry.k8s.io/kube-controller-manager:v1.29.15
registry.k8s.io/kube-scheduler:v1.29.15
registry.k8s.io/kube-proxy:v1.29.15
registry.k8s.io/coredns/coredns:v1.11.1
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.10-0
```

루트 권한을 획득한 뒤, 쿠버네티스 설치에 필요한 이미지를 다운로드 한다.
```shell
ubuntu@ubuntu22-arm-1-50gb:~$ sudo -i
root@ubuntu22-arm-1-50gb:~# kubeadm config images pull
I0604 13:01:33.428353  123929 version.go:256] remote version is much newer: v1.33.1; falling back to: stable-1.29
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.29.15
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.29.15
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.29.15
[config/images] Pulled registry.k8s.io/kube-proxy:v1.29.15
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.11.1
[config/images] Pulled registry.k8s.io/pause:3.9
[config/images] Pulled registry.k8s.io/etcd:3.5.10-0
```

아래 명령어로 ip를 확인한다. 필자의 경우 10.0.0.169
```shell
oot@ubuntu22-arm-1-50gb:~# ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:00:17:03:a8:74 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.169/24 metric 100 brd 10.0.0.255 scope global enp0s6
       valid_lft forever preferred_lft forever
    inet6 fe80::17ff:fe03:a874/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 82:cd:0a:3b:80:c1 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: br-c09904005d4c: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 9e:83:77:e4:85:bd brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-c09904005d4c
       valid_lft forever preferred_lft forever
    inet6 fe80::9c83:77ff:fee4:85bd/64 scope link 
       valid_lft forever preferred_lft forever
5: veth44ad33c@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-c09904005d4c state UP group default 
    link/ether 42:22:14:ae:d8:cc brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::4022:14ff:feae:d8cc/64 scope link 
       valid_lft forever preferred_lft forever
6: vethbdc31c2@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-c09904005d4c state UP group default 
    link/ether 5e:bf:44:95:2d:b5 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::5cbf:44ff:fe95:2db5/64 scope link 
       valid_lft forever preferred_lft forever
```


apiserver-advertise-address는 위에서 파악한 ip를 넣자.
```shell
root@ubuntu22-arm-1-50gb:~# kubeadm init --apiserver-advertise-address=10.0.0.169 --pod-network-cidr=192.168.0.0/16 --cri-socket /run/containerd/containerd.sock
W0604 13:53:52.075108    6289 initconfiguration.go:125] Usage of CRI endpoints without URL scheme is deprecated and can cause kubelet errors in the future. Automatically prepending scheme "unix" to the "criSocket" with value "/run/containerd/containerd.sock". Please update your configuration!
I0604 13:53:53.079512    6289 version.go:256] remote version is much newer: v1.33.1; falling back to: stable-1.29
[init] Using Kubernetes version: v1.29.15
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W0604 13:53:53.780127    6289 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local ubuntu22-arm-1-50gb] and IPs [10.96.0.1 10.0.0.169]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost ubuntu22-arm-1-50gb] and IPs [10.0.0.169 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost ubuntu22-arm-1-50gb] and IPs [10.0.0.169 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 7.002369 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node ubuntu22-arm-1-50gb as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node ubuntu22-arm-1-50gb as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: 9stuxl.eydqlpwzqp1g2wv0
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.169:6443 --token 9stuxl.eydqlpwzqp1g2wv0 \
        --discovery-token-ca-cert-hash sha256:3fc95769754c24ab16bd3e7cd26aca82e36f3159135f6fa1fb8dfbf0b7fbd25d 
```
여기서 바로 위의 토큰값은 꼭 저장해놓도록 하자, 이후에 워커 노드에서 마스터 노드에 연결할때 사용할 구문이다. (이후에 따로 나오지 않는다.)


다시한번 쿠버네티스 인증서 상태를 확인해보면, 인증이 되있는 것을 확인할 수 있다.
```shell
root@ubuntu22-arm-1-50gb:~# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jun 04, 2026 13:53 UTC   364d            ca                      no      
apiserver                  Jun 04, 2026 13:53 UTC   364d            ca                      no      
apiserver-etcd-client      Jun 04, 2026 13:53 UTC   364d            etcd-ca                 no      
apiserver-kubelet-client   Jun 04, 2026 13:53 UTC   364d            ca                      no      
controller-manager.conf    Jun 04, 2026 13:53 UTC   364d            ca                      no      
etcd-healthcheck-client    Jun 04, 2026 13:53 UTC   364d            etcd-ca                 no      
etcd-peer                  Jun 04, 2026 13:53 UTC   364d            etcd-ca                 no      
etcd-server                Jun 04, 2026 13:53 UTC   364d            etcd-ca                 no      
front-proxy-client         Jun 04, 2026 13:53 UTC   364d            front-proxy-ca          no      
scheduler.conf             Jun 04, 2026 13:53 UTC   364d            ca                      no      
super-admin.conf           Jun 04, 2026 13:53 UTC   364d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Jun 02, 2035 13:53 UTC   9y              no      
etcd-ca                 Jun 02, 2035 13:53 UTC   9y              no      
front-proxy-ca          Jun 02, 2035 13:53 UTC   9y              no  
```

이번엔 루트 권한뿐만 아니라, 사용자 권한으로도 쿠버네티스를 사용할 수 있도록 설정한다.
```shell
root@ubuntu22-arm-1-50gb:~# exit
logout
ubuntu@ubuntu22-arm-1-50gb:~$ mkdir -p $HOME/.kube
ubuntu@ubuntu22-arm-1-50gb:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
ubuntu@ubuntu22-arm-1-50gb:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```shell
ubuntu@ubuntu22-arm-1-50gb:~$ kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/tigera-operator.yaml
namespace/tigera-operator created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/apiservers.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/imagesets.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/installations.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/tigerastatuses.operator.tigera.io created
serviceaccount/tigera-operator created
clusterrole.rbac.authorization.k8s.io/tigera-operator created
clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
deployment.apps/tigera-operator created
```

```shell
ubuntu@ubuntu22-arm-1-50gb:~$ curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/custom-resources.yaml -O
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   824  100   824    0     0   1339      0 --:--:-- --:--:-- --:--:--  1339

ubuntu@ubuntu22-arm-1-50gb:~$ ls
custom-resources.yaml
ubuntu@ubuntu22-arm-1-50gb:~$ kubectl create -f custom-resources.yaml 
installation.operator.tigera.io/default created
apiserver.operator.tigera.io/default created
```

```shell
Every 2.0s: kubectl get pods -n calico-system                                           ubuntu22-arm-1-50gb: Wed Jun  4 14:02:26 2025

NAME                                       READY   STATUS              RESTARTS   AGE
calico-kube-controllers-5bd446cb9d-7bz9g   0/1     Pending             0          49s
calico-node-ltk7l                          0/1     PodInitializing     0          49s
calico-typha-f9847b87d-ml9xq               1/1     Running             0          49s
csi-node-driver-24nrs                      0/2     ContainerCreating   0          49s


Every 2.0s: kubectl get pods -n calico-system                                           ubuntu22-arm-1-50gb: Wed Jun  4 14:03:10 2025

NAME                                       READY   STATUS              RESTARTS   AGE
calico-kube-controllers-5bd446cb9d-7bz9g   1/1     Running             0          93s
calico-node-ltk7l                          1/1     Running             0          93s
calico-typha-f9847b87d-ml9xq               1/1     Running             0          93s
csi-node-driver-24nrs                      0/2     ContainerCreating   0          93s
```

```shell
ubuntu@ubuntu22-arm-1-50gb:~$ watch kubectl get pods -n calico-system
ubuntu@ubuntu22-arm-1-50gb:~$ watch kubectl get pods -n calico-system
ubuntu@ubuntu22-arm-1-50gb:~$ kubectl get node -o wide
NAME                  STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
ubuntu22-arm-1-50gb   Ready    control-plane   9m52s   v1.29.0   10.0.0.169    <none>        Ubuntu 22.04.5 LTS   6.8.0-1026-oracle   containerd://1.7.27
ubuntu@ubuntu22-arm-1-50gb:~$ kubectl get node
NAME                  STATUS   ROLES           AGE   VERSION
ubuntu22-arm-1-50gb   Ready    control-plane   10m   v1.29.0
ubuntu@ubuntu22-arm-1-50gb:~$ 
```