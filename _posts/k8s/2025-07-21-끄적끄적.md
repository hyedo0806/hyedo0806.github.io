---
layout: post
title: "VirtualBox CentOS 설정"
date: 2025-07-19 12:00:00 +0900
categories: [VirtualBox, CentOS, SSH]
---

VirtualBox에서 CentOS를 설치하고 macOS 호스트에서 SSH로 접속하는 방법을 정리했습니다.

## 1. CentOS에 SSH 서버 설치 및 실행
```bash
sudo yum install -y openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
sudo firewall-cmd --permanent --zone=public --add-service=ssh
sudo firewall-cmd --reload
```

## 2. 네트워크 모드 설정 
VirtualBox에서는 VM(가상 머신)과 호스트 및 외부 네트워크 간 통신을 위해 여러 네트워크 모드를 제공합니다.  
대표적으로 NAT, Bridged Adapter, Host-Only Adapter, Internal Network 네 가지가 있습니다.


### 1. NAT (Network Address Translation)
- **동작 원리**: VM이 호스트를 통해 외부 인터넷에 연결됩니다.  
- **사용 사례**: 외부 인터넷 연결만 필요할 때.
- **장점**:
  - 별도의 설정 없이 인터넷 사용 가능.
  - 호스트와 외부에서 VM이 보이지 않아 보안성이 높음.
- **단점**:
  - 호스트에서 VM으로 직접 접근하려면 포트 포워딩 필요.

**SSH 접속 예시 (포트 포워딩)**:
- VirtualBox → VM → **설정** → **네트워크** → **어댑터 1** → **고급 → 포트 포워딩**:
  - 호스트 포트: `2222`, 게스트 포트: `22`

호스트에서 접속:
```bash
ssh -p 2222 hailey@127.0.0.1
```

### 2. Bridged Adapter
Bridged Adapter는 VM이 호스트와 동일한 물리 네트워크에 직접 연결되어 **실제 장치처럼 동작**하는 네트워크 모드입니다.
- **특징**
  - VM이 물리 네트워크에서 고유한 IP 주소를 할당받음.
  - 호스트와 같은 네트워크 내의 다른 장치에서 VM에 접근 가능.
  - NAT와 달리 포트 포워딩이 필요 없음.
- **장점**
  - 외부 장치에서 VM에 직접 접근할 수 있음.
  - 호스트와 VM이 대등한 네트워크 위치에 있음.
- **단점**
  - 회사/공용 네트워크에서는 접근이 제한될 수 있음.
  - IP 충돌 위험이 있음.

**설정 방법**
1. VirtualBox → VM → **설정** → **네트워크**
2. 어댑터 연결 방식: **Bridged Adapter**
3. 네트워크 인터페이스 선택 (Wi-Fi 또는 Ethernet)
4. VM에서 IP 주소 확인:
    ```bash
    ip addr
    ```
5. 호스트에서 VM에 SSH 접속:
    ```bash
    ssh hailey@<VM_IP>
    ```

**사용 사례**
- VM을 동일 네트워크 내 서버처럼 활용할 때
- 외부에서 접근해야 하는 테스트 서버 구축 시

### 3. Host-Only Adapter
- **특징**
  - 외부 인터넷과 완전히 분리됨.
  - 호스트와 VM 간의 안전한 개발/테스트 환경에 적합.
- **장점**
  - 안전한 전용 네트워크 제공.
  - NAT와 조합하면 VM의 인터넷 접속도 가능.
- **단점**
  - 단독으로는 인터넷 연결이 불가능.

**설정 방법**
1. VirtualBox → VM → **설정** → **네트워크**
2. 어댑터 1: **NAT** (인터넷 접속용)
3. 어댑터 2: **Host-Only Adapter** (호스트 전용 통신)
4. VM에서 IP 확인:
    ```bash
    ip addr
    ```
5. 호스트에서 SSH 접속:
    ```bash
    ssh user@<VM_IP>
    ```

**사용 사례**
- 호스트와 VM 간 데이터 전송
- 안전한 개발 및 테스트 환경
- 인터넷 연결이 필요 없는 폐쇄형 네트워크

**proxy server**

Host-Only 네트워크는 VM 과 호스트 간에만 통신이 가능한 격리 네트워크이다. VM은 기본적으로 외부 인터넷과 연결되지 않는다. 
호스트 PC 또는 별도의 VM에 Proxy 서버를 설치해 vm 들의 인터넷 접속 gateway 역할을 하도록 구성할 수 있다.(마치 회사망처럼...)

**squid**

가장 널리 쓰이는 오픈 소스 proxy server이다. (참고참고)