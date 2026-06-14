---
title: Portfolio
icon: fas fa-diagram-project
order: 1
permalink: /portfolio/
toc: false
---

{% include portfolio-style.html %}

<section class="portfolio-hero compact-hero">
  <p class="section-label">Portfolio</p>
  <h1>관리형 서비스와 플랫폼 기능을 제품 단위로 정리한 포트폴리오</h1>
  <p class="hero-copy">
    회사에서 다뤘던 대표적인 서비스 영역을 중심으로 프로젝트 개요, 아키텍처,
    담당 역할, 핵심 기술만 간결하게 정리했습니다.
  </p>
</section>

<p class="section-note">
  이 섹션은 상세 이력서 대신, 어떤 도메인과 제어 흐름을 다뤘는지 빠르게 보여주는 요약판입니다.
</p>

<div class="card-grid">
  <a class="info-card card-link" href="#api-gateway">
    <h3>API Gateway</h3>
    <p>Kubernetes Operator 기반 관리형 API Gateway 서비스</p>
  </a>
  <a class="info-card card-link" href="#secret-manager">
    <h3>Secret Manager</h3>
    <p>보안 중심의 비밀정보 관리 서비스</p>
  </a>
  <a class="info-card card-link" href="#managed-queue">
    <h3>Managed Queue</h3>
    <p>RabbitMQ 기반 메시징 서비스와 운영 자동화</p>
  </a>
  <a class="info-card card-link" href="#devops-code">
    <h3>DevOps Code</h3>
    <p>GitOps와 운영 표준화를 위한 배포/운영 코드</p>
  </a>
</div>

<h2 id="api-gateway">API Gateway</h2>

<table class="portfolio-table">
  <tr>
    <th>개요</th>
    <td>AWS API Gateway를 벤치마킹한 Kubernetes Operator 기반 관리형 API Gateway 서비스</td>
  </tr>
  <tr>
    <th>기술</th>
    <td>Go / Kubernetes / Kong / Gateway API</td>
  </tr>
  <tr>
    <th>내 역할</th>
    <td>Method Controller, Resource Controller, ApiDeployment Controller, Gateway API 전환, Metric Report</td>
  </tr>
  <tr>
    <th>포인트</th>
    <td>선언형 리소스 모델과 제어면 로직을 통해 API 관리 기능을 서비스화한 경험</td>
  </tr>
</table>

<h2 id="secret-manager">Secret Manager</h2>

<table class="portfolio-table">
  <tr>
    <th>개요</th>
    <td>애플리케이션과 플랫폼 운영을 위한 비밀정보 저장, 접근 제어, 라이프사이클 관리를 다루는 서비스</td>
  </tr>
  <tr>
    <th>기술</th>
    <td>Go / Kubernetes / Spring Security / KMS 유사 설계</td>
  </tr>
  <tr>
    <th>내 역할</th>
    <td>리소스 모델 설계, 인증/인가 흐름 정리, Secret 회전과 감사 가능성을 고려한 API 구조 설계</td>
  </tr>
  <tr>
    <th>포인트</th>
    <td>보안 요구사항을 기능 뒤에 두지 않고 서비스 계약과 운영 흐름에 직접 녹여낸 경험</td>
  </tr>
</table>

<h2 id="managed-queue">Managed Queue</h2>

<table class="portfolio-table">
  <tr>
    <th>개요</th>
    <td>RabbitMQ 기반 관리형 메시징 서비스를 목표로 한 운영 자동화 및 리소스 관리 영역</td>
  </tr>
  <tr>
    <th>기술</th>
    <td>Go / Kubernetes / RabbitMQ / Observability</td>
  </tr>
  <tr>
    <th>내 역할</th>
    <td>Broker 수명주기 관리 관점 정리, 운영 편의성 확보를 위한 리소스/상태 관리 방향 설계</td>
  </tr>
  <tr>
    <th>포인트</th>
    <td>메시징 서비스를 단순 배포가 아닌 관리형 제품으로 다루기 위한 제어 흐름과 운영 기준 정리</td>
  </tr>
</table>

<h2 id="devops-code">DevOps Code</h2>

<table class="portfolio-table">
  <tr>
    <th>개요</th>
    <td>서비스 운영을 안정적으로 반복하기 위한 GitOps, 배포 자동화, 운영 코드 자산화 영역</td>
  </tr>
  <tr>
    <th>기술</th>
    <td>ArgoCD / GitOps / Kubernetes / CI/CD</td>
  </tr>
  <tr>
    <th>내 역할</th>
    <td>배포 구조 표준화, 환경 차이 최소화, 운영 변경의 추적 가능성을 높이는 코드 중심 프로세스 정리</td>
  </tr>
  <tr>
    <th>포인트</th>
    <td>플랫폼 기능과 운영 체계를 분리하지 않고 하나의 제품 경험으로 연결한 점</td>
  </tr>
</table>
