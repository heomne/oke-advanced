---
title: OKE Hands-on
layout: home
theme: just-the-docs
has-children: true
---

# OKE Hands-on (심화)

[LiveLabs 링크](https://apexapps.oracle.com/pls/apex/r/dbpm/livelabs/run-workshop?p210_wid=3206&p210_wec=&session=4354810289205)

## Get Started
- Oracle Cloud Infrastructure (OCI) 회원가입

## Lab 1: Setup Cloud Environment
- OCI 콘솔 로그인 및 OKE(Oracle Kubernetes Engine) 클러스터 생성
- 클러스터 생성 시 10분~15분 소요

## Lab 2: Develop a Microservice
- Spring Boot 기반 간단한 애플리케이션 구현
- 어플리케이션 소스 빌드 후 Docker Image 생성
- 생성한 이미지 OCIR(Oracle Cloud Infrastructure Registry)에 업로드

## Lab 3: Create a Helm Chart
- Lab 2에서 OCIR에 업로드한 컨테이너 이미지 토대로 Helm Chart 생성
- Helm Chart 생성 후 쿠버네티스에 배포

## Lab 4: Deploy the MuShop App
- 오라클에서 제공하는 샘플 애플리케이션 MuShop을 Helm Chart로 자동화 배포, 배포한 애플리케이션 확인
- Mushop Chart에 Prometheus/Grafana 모니터링 툴이 포함됨(Lab 5에서 사용)

## Lab 5: Monitor the deployment
- Lab 4에서 배포한 MuShop 애플리케이션을 OCI 콘솔, OSS Grafana로 모니터링
- 부하 시뮬레이션을 통한 Autoscaling 및 모니터링
