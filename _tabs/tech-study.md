---
title: Tech & Study
icon: fas fa-book-open
order: 3
permalink: /tech-study/
toc: false
---

{% include portfolio-style.html %}

<section class="portfolio-hero compact-hero">
  <p class="section-label">Tech & Study</p>
  <h1>기술 학습, 세미나 정리, 자격증 공부 메모를 쌓아가는 기록</h1>
  <p class="hero-copy">
    실무에서 바로 써먹기 위한 메모부터, 컨퍼런스/세미나를 정리한 글,
    특정 주제를 깊게 파고드는 학습 기록까지 한곳에 모읍니다.
  </p>
</section>

<div class="chip-row">
  <span class="chip">Kubernetes</span>
  <span class="chip">Operator</span>
  <span class="chip">Security</span>
  <span class="chip">Cloud</span>
  <span class="chip">Web</span>
  <span class="chip">AI Infra</span>
</div>

## 최근 글

<ul class="compact-posts">
{% for post in site.posts limit: 12 %}
  <li>
    <a href="{{ post.url | relative_url }}">{{ post.title | default: post.slug }}</a>
    <span>{{ post.date | date: "%Y-%m-%d" }}</span>
  </li>
{% endfor %}
</ul>

## 주로 다루는 주제

<div class="card-grid">
  <article class="info-card">
    <h3>Kubernetes / Operator</h3>
    <p>CRD, Controller, Reconcile 패턴과 테스트 코드 작성, 운영 관점의 구조를 다룹니다.</p>
  </article>
  <article class="info-card">
    <h3>Security / Access Control</h3>
    <p>JWT, Spring Security, 비밀정보 관리처럼 플랫폼 보안에 직접 연결되는 주제를 정리합니다.</p>
  </article>
  <article class="info-card">
    <h3>Cloud Service Design</h3>
    <p>AWS 유사 서비스 구조, 플랫폼 기능 분리, 관리형 서비스 관점의 모델링 메모를 남깁니다.</p>
  </article>
  <article class="info-card">
    <h3>Seminar and Study Notes</h3>
    <p>스터디, 발표 자료, 자격증 공부 과정에서 남긴 요약과 인사이트를 모읍니다.</p>
  </article>
</div>
