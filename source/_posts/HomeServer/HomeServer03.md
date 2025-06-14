---
title: 홈 서버 구축기 03 - KVM 가상화
date: 2025-06-12 11:00:00
categories: home-server
tags:
  - Home Server
  - Server
  - AMD CPU Server
---

## 1. KVM 설치 준비 (사전 확인)
가장 먼저, CPU가 하드웨어 가상화 기술(Intel VT-x 또는 AMD-V)을 지원하는지 확인해야 한다. 
이 기술이 있어야 KVM이 커널 수준에서 효율적으로 작동한다.

```shell
egrep -c '(vmx|svm)' /proc/cpuinfo
# 결과로 1 이상의 숫자가 출력되면 CPU가 가상화를 지원하는 것이다. 
# 만약 0이 나온다면, 시스템의 BIOS/UEFI 설정에 진입하여 'Virtualization Technology' 또는 유사한 이름의 옵션을 활성화해야 한다.
```

## 2. KVM 및 필수 유틸리티 설치
다음 명령어로 KVM 구동에 필요한 핵심 패키지들을 설치한다.

```shell
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst bridge-utils -y
```

각 패키지의 역할은 다음과 같습니다.

qemu-kvm: KVM 커널 모듈을 활용해 가상 하드웨어를 에뮬레이션하는 핵심 백엔드이다.
libvirt-daemon-system: 가상머신, 스토리지, 네트워크를 관리하는 백그라운드 서비스(데몬)이다.
libvirt-clients: virsh 등 가상머신을 제어하는 클라이언트 도구 모음이다.
virsh: 실행 중인 가상머신을 관리(시작, 종료, 콘솔 접속 등)하는 기본 도구이다.
virtinst: virt-install 명령어를 제공하는 패키지입니다. 가상머신을 새로 생성하고 설치하는 데 필수이다.
bridge-utils: 가상 네트워크 브리지를 생성하고 관리하는 유틸리티이다.


## 3. 사용자 권한 설정
매번 sudo를 사용하는 번거로움을 피하기 위해 현재 사용자를 libvirt와 kvm 그룹에 추가한다. 
이를 통해 일반 사용자 계정으로 가상머신을 관리할 수 있는 권한을 얻게 된다.

```shell
sudo adduser $USER libvirt
sudo adduser $USER kvm
```

✅ 중요: 그룹 변경 사항을 시스템에 적용하려면 로그아웃 후 다시 로그인하거나 시스템을 재부팅해야 한다.

## 4. 설치 미디어(ISO) 준비
가상머신에 설치할 운영체제의 ISO 파일이 필요하다. 
wget을 사용하여 Ubuntu 24.04 서버 ISO 파일을 홈 디렉토리로 다운로드한다.

```shell
cd ~
# 우분투 릴리스 페이지를 먼저 확인
curl -s https://releases.ubuntu.com/24.04/ | grep -i "server.*iso"

# 다운로드
wget https://releases.ubuntu.com/24.04/ubuntu-24.04.2-live-server-amd64.iso
```


libvirt-qemu 사용자가 접근할 수 있도록 /home 폴더로 ISO 파일을 옮긴다.
```shell
sudo mv ubuntu-24.04.2-live-server-amd64.iso /home/

# 스토리지 풀 디렉토리 생성
sudo mkdir -p /var/kvm/images
```



## 5. virt-install로 가상머신 생성 및 VirtIO 적용
이제 virt-install 명령어를 사용하여 모든 설정을 한 번에 적용한 가상머신을 생성한다. 
아래 예제는 2 vCPU, 4GB RAM, 20GB VirtIO 디스크를 갖춘 Ubuntu 서버를 생성하는 명령어이다.

```shell
sudo virt-install \
--name ubuntu2404 \
--ram 4096 \
--disk path=/var/kvm/images/ubuntu2404.img,size=20,bus=virtio,format=qcow2 \
--vcpus 2 \
--os-variant ubuntu24.04 \
--network network=default,model=virtio \
--graphics none \
--console pty,target_type=serial \
--location /home/ubuntu-24.04.2-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
--extra-args 'console=ttyS0,115200n8'
```

--disk ... bus=virtio: 디스크 컨트롤러로 VirtIO를 지정하여 I/O 성능을 극대화한다.
--network ... model=virtio: 네트워크 카드 모델로 VirtIO를 지정하여 네트워크 처리량을 극대화한다.
--graphics none 과 --extra-args 'console=ttyS0...': GUI 없이 시리얼 콘솔만으로 설치와 접속이 가능하도록 설정한다.
명령을 실행하면 즉시 OS 설치 화면으로 진입하며, 화면의 안내에 따라 설치를 완료하면 된다.

이런 화면이 나온다면  Continue in basic mode로 진행한다.
![](/images/HomeServer03-01.png)

그럼 일반적인 우분투 설치화면이 나온다. 이제 설치를 진행하면된다.
![](/images/HomeServer03-02.png)


최종적으로 VM 생성에 성공했다. Reboot Now를 선택하자.
![](/images/HomeServer03-03.png)

그럼 아래와 같이 에러가 뜰 것이다. 실제 서버라면 USB를 뺀후 엔터를 해야하지만, 지금은 엔터만 입력하면된다.
![](/images/HomeServer03-04.png)


최종적으로 가상머신 및 우분투 설치에 성공했다.
![](/images/HomeServer03-05.png)

생각보다 정확한 예제가 없어 힘들었으며 아래 글이 제일 참고에 도움이 되었다.
참고 글: https://www.server-world.info/en/note?os=Ubuntu_24.04&p=kvm&f=2

참고로, 물리적인 호스트 서버를 Restart 하면, 가상머신은 꺼지게 되는데 아래의 설정으로 항상 자동으로 시작하게 할 수 있다.
```shell
virsh autostart ubuntu2404
```

## 6. virsh로 가상머신 관리하기
설치가 완료된 후에는 virsh 명령어로 가상머신을 제어할 수 있다.

모든 가상머신 목록 확인
```shell
virsh list --all
```
가상머신 시작
```shell
virsh start ubuntu-server-2404
```
가상머신 콘솔 접속
```shell
virsh console ubuntu-server-2404
#(콘솔에서 빠져나오려면 Ctrl + ] 키를 누른다.)
```

가상머신 종료
```shell
virsh shutdown ubuntu-server-2404
```

가상머신 삭제
```shell
virsh destroy ubuntu2404 2>/dev/null || true
virsh undefine ubuntu2404 --remove-all-storage 2>/dev/null || true
```

이 과정을 통해 CLI 환경에서 KVM의 구성요소를 이해하고, 고성능 VirtIO 가상머신을 생성 및 관리할 수 있다. 💻
