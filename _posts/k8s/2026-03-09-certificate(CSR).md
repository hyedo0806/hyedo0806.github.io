---
title: CSR
date: 2026-03-09 11:11:11 
categories: [k8s]
tags: [TAG]     # TAG names should always be lowercase
---
# 순서
사용자 --> CSR 생성

k8s에 CSR 제출

승인 및 발급

# CSR 생성과정 
1. private key 생성
```
openssl genrsa -out user.key 2048
```

2. CSR 생성
```
openssl req -new -key user.key -out user.csr
```

template 암기
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: USERNAME
spec:
  request: BASE64CSR
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```
(참고) https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/

3. 

# 관련 시험 스타일
1. CSR 승인
```
// pending 상태인 CSR을 찾아 승인
kubectl get csr
kubectl certificate approve john
```
2. CSR 생성
```
CSR을 생성하고 승인
openssl genrsa -out john.key 2048
openssl req -new -key john.key -out john.csr
```
3. 인증서 만료 트러블 슈팅

4. (심화) dev-user가 클러스터에 접근할 수 있도록 구성하시오.
```
1️⃣ key 생성
2️⃣ CSR 생성
3️⃣ CSR Kubernetes에 제출
4️⃣ CSR 승인
5️⃣ certificate 추출
6️⃣ kubeconfig user 등록
7️⃣ context 생성
```