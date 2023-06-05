---
title: "Lab 2: Develop a Microservice"
layout: default
theme: just-the-docs
parent: OKE Hands-on
nav_order: 3
---


# Lab 2: Develop a Microservice
- Spring Boot 기반 애플리케이션 구현 및 빌드
    - 애플리케이션은 JSON 메시지를 리턴하는 간단한 애플리케이션입니다.
- 빌드한 애플리케이션 Docker image 생성
- 생성한 이미지 OCIR(Oracle Cloud Infrastructure Registry)에 업로드
- [LiveLabs](https://apexapps.oracle.com/pls/apex/r/dbpm/livelabs/run-workshop?p210_wid=3206&p210_wec=&session=4354810289205)에 **Lab 2: Develop a Microservice** 참고하시어 실습을 진행해주시기바랍니다.

## 유의사항
- (Task 1-1) 기본 프로젝트 소스 파일은 Option #2(커맨드)를 통해 생성하는 방법이 빠릅니다.
```
curl https://start.spring.io/starter.tgz -d type=maven-project -d bootVersion=2.7.7.RELEASE \
-d baseDir=rest-service -d name=rest-service -d artifactId=rest-service \
-d javaVersion=11 -d dependencies=web,actuator | tar -xzvf -
```
- (Task 3-1) OCIR 주소는 아래를 참고해주세요.
1. `oci os ns get` 명령어를 입력합니다.
```
$ oci os ns get
{
    "data": "axjowrxaexxx"
}
```
`axjowrxaexxx`를 복사합니다.
2. OCI에 가입할 때 선택한 지역에 따라 아래와 같이 OCIR 주소가 나뉘어집니다.
- 서울: ap-seoul-1.ocir.io/axjowrxaexxx/oci-hol-xx/spring-boot-greeting:1.0
- 춘천: ap-chuncheon-1.ocir.io/axjowrxaexxx/oci-hol-xx/spring-boot-greeting:1.0