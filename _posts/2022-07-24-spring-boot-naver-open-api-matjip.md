---
title: Spring Boot - Naver Open API를 통해서 맛집 리스트 만들기
date: 2022-07-24 01:16:17 +0900
categories: [Dev, Troubleshooting]
tags: [Spring Boot, Java, error]
---

![오류 사진](/assets/img/posts/2022-07-24-naver-open-api/error.png)

이렇게 알 수 없는 오류가 떴다.

한참의 구글링 끝에, Spring Boot 2.6 버전 이후로 몇몇 라이브러리들이 충돌하면서 오류가 발생하고 있다는 걸 짐작하고 그 쪽으로 또 한참 구글링을 해보았다.

java, jdk 버전을 최신 버전으로도 해보고 11 버전까지 낮춰서 해봤는데도 달라지는 건 없다.

java complier 버전도 11로 맞춰도..

또, Run/Debug Configurations 에 있는 Modify options - Enable debug output 도 체크하라는 해답을 따라 체크도 해보았지만 여전히 오류가 발생했다.

그러다가, 프로젝트 내의 src - main - resources - `application.properties` 에다가

`spring.mvc.pathmatch.matching-strategy=ant_path_matcher` 를 적어주고 서버를 재실행 해보았다.

![결과 화면](/assets/img/posts/2022-07-24-naver-open-api/result.png)

결과는.... 정상 작동..!!

최신 버전에서의 라이브러리 오류들을 잡아주는 문구인듯하다.

> 출처: [https://goyunji.tistory.com/137](https://goyunji.tistory.com/137) [개발하는 농부:티스토리]
