---
title: 홈 서버 구축기 04 - Nginx 포워딩 & 웹 페이지 서빙
date: 2025-06-12 14:40:00
categories: home-server
tags:
  - Home Server
  - Server
  - AMD CPU Server
---

## 개요
이번 시간에는 홈 서버의 호스트에 Nginx를 설치하여, 최초 가비아에서 설정한 서브도메인에 따라 다른 서비스를 응답하도록 한다.

필자의 도메인은 hyunjun.kr 이며 blog.hyunjun.kr 요청 시, 내부의 가상 머신으로 연결하도록 한다.

## Nginx 설치
패키지 업데이트 및 Nginx 설치를 진행한다.

```shell
# 패키지 목록 업데이트
sudo apt update
```

```shell
sudo apt install nginx -y

# Nginx 서비스 시작
sudo systemctl start nginx

# 부팅 시 자동 시작 설정
sudo systemctl enable nginx

# 상태 확인
sudo systemctl status nginx
```

## 가상머신 IP 확인
먼저 생성한 가상머신의 IP를 확인해야 한다.
```shell
# 가상머신 IP 확인
virsh domifaddr ubuntu2404

# 또는 가상머신 내부에서 확인
virsh console ubuntu2404
# 가상머신 내부에서: ip addr show
```


## Nginx 설정 변경
Nginx 가상 호스트 설정 파일 생성한다.

```shell
# blog 서브도메인용 설정 파일 생성
sudo vi /etc/nginx/sites-available/blog.hyunjun.kr
```

```nginx
server {
   listen 80;
   server_name blog.hyunjun.kr;

   # 로깅 설정
   access_log /var/log/nginx/blog.hyunjun.kr.access.log;
   error_log /var/log/nginx/blog.hyunjun.kr.error.log;

   # 가상머신으로 프록시
   location / {
   proxy_pass http://가상머신IP:80;
   proxy_set_header Host $host;
   proxy_set_header X-Real-IP $remote_addr;
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   proxy_set_header X-Forwarded-Proto $scheme;

        # 타임아웃 설정
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
   }
}
```


해당 사이트 설정을 활성화 해준다.
```shell
# 심볼릭 링크 생성 (사이트 활성화)
sudo ln -s /etc/nginx/sites-available/blog.hyunjun.kr /etc/nginx/sites-enabled/

# 기본 사이트 비활성화 (선택사항)
# sudo rm /etc/nginx/sites-enabled/default
```

설정 파일 문법 검사 및 재시작
```shell
# Nginx 설정 문법 검사
sudo nginx -t

# 설정이 올바르면 리로드
sudo systemctl reload nginx
```

## 가상머신에 웹 서빙용 Nginx 설치 및 설정
가상머신의 콘솔로 접속한다.
```shell
# 가상머신 콘솔 접속
virsh console ubuntu2404
```

가상머신에 Nginx 설치
```shell
# 패키지 업데이트
sudo apt update

# Nginx 설치
sudo apt install nginx -y

# Nginx 서비스 시작 및 활성화
sudo systemctl start nginx
sudo systemctl enable nginx

# 상태 확인
sudo systemctl status nginx
```

테스트 페이지 생성

# 기본 HTML 페이지 수정
```shell
sudo vi /var/www/html/index.html
```
   
다음 내용으로 수정한다.
```html
<!DOCTYPE html>
<html>
<head>
   <title>Blog Server</title>
   <style>
      body {
         font-family: Arial, sans-serif;
         margin: 40px;
         background: #f5f5f5;
      }
      .container {
         background: white;
         padding: 30px;
         border-radius: 10px;
         box-shadow: 0 2px 10px rgba(0,0,0,0.1);
         max-width: 600px;
      }
      h1 { color: #333; margin-bottom: 20px; }
      .status {
         background: #d4edda;
         padding: 15px;
         border-radius: 5px;
         margin: 15px 0;
         border-left: 4px solid #28a745;
      }
      .info { color: #666; font-size: 14px; }
      .success { color: #28a745; font-weight: bold; }
   </style>
</head>
<body>
<div class="container">
   <h1>Blog Server Running</h1>

   <div class="status">
      <strong>Virtual Machine Active</strong><br>
      Ubuntu 24.04 VM with Nginx
   </div>

   <p>This is a test page served from KVM virtual machine.</p>

   <div class="info">
      <p><strong>Domain:</strong> blog.hyunjun.kr</p>
      <p><strong>Server:</strong> Ubuntu 24.04 VM</p>
      <p><strong>Web Server:</strong> Nginx</p>
      <p><strong>Proxy:</strong> Working correctly</p>
   </div>

   <p class="success">Subdomain routing is working!</p>
</div>
</body>
</html>

```

먼저 가상머신에서 직접 테스트 해본다.

```shell
# 가상머신 내부에서 로컬 테스트
curl http://localhost
```

호스트에서 가상머신으로 테스트 해본다.

```shell
# 호스트로 돌아가기 (Ctrl + ])
# 호스트에서 가상머신 IP로 테스트
curl http://가상머신IP
```

마지막으로 http://blog.hyunjun.kr 로 접속하여 index page를 확인한다.