---
layout: post
title: "CentOS에서 k8s 설치하기"
date: 2025-07-20 12:00:00 +0900
categories: [CentOS, k8s]
---

## 1. container runtime
k8s 클러스터를 사용하기 위해서는 각 node에 container runtime을 설치해야한다. Kubernetes **1.32**에서는 CRI를 사용하는 것을 요구한다. **v1.24** 이전에는 docker engine과 직접 연동해서 컨테이너를 실행했다. 이때 사용하는 특별한 컴포넌트가 dockershim 이다. 

### Kubernetes의 container runtime
- containerd
- CRI-O
- Docker Engine 
### How to Install
```
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

## 2. cgroup drivers
### group 이란?
