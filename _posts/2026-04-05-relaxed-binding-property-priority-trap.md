---
title: Relaxed Binding과 Property 우선순위의 함정
date: 2026-04-05 00:48:04 +0900
categories: [Dev, Troubleshooting]
tags: [Spring Boot, Kubernetes, Relaxed Binding, Jasypt]
---

개발 중에 예상치 못한 버그를 만났다. 마이크로서비스 환경에서 A 서비스가 B Server로 API 요청을 보낼 때마다 401 Unauthorized가 나왔다. 코드는 문제없어 보였고, API 키도 제대로 주입된 것처럼 보였는데, 삽질 끝에 알게 된 원인은 Spring Boot의 편의 기능 하나였다. **Relaxed Binding.**

### Spring Boot Relaxed Binding이란?

Relaxed Binding은 Spring Boot가 제공하는 기능으로, 환경변수나 시스템 프로퍼티의 다양한 이름 형식을 자동으로 properties에 매핑해주는 것이다.

예를 들어 다음은 모두 같은 설정을 가리킨다.

| 형식 | 예시 |
| --- | --- |
| KebabCase | `service.api-key` |
| CamelCase | `service.apiKey` |
| UnderScore | `service.api_key` |
| UpperCase | `SERVICE_API_KEY` |

왜 이런 기능이 필요할까?

OS 환경변수는 대문자와 언더스코어 사용이 표준이다. 하지만, YAML 설정 파일은 케밥케이스 관례를 따른다. Relaxed Binding은 이 간극을 자동으로 메워주므로, 개발자는 형식을 신경 쓰지 않아도 된다.

```yaml
# application.yml
server:
  port: 8080

# K8S 환경변수
SERVER_PORT=9090
```

Spring Boot는 `SERVER_PORT`를 `server.port`로 자동 매핑해주므로, 포트는 **9090**으로 실행된다.

### Property Source 우선순위 (문제의 원인)

하지만 여기서 중요한 개념이 하나 있다. **Property Source 우선순위**.

Spring Boot 공식문서에 따르면 우선순위는 다음과 같다 (낮음 → 높음 순으로).

1. 설정 파일 (`application.yml`)
2. OS 환경 변수
3. CommandLine 인자
4. ...

**즉, 같은 설정이 두 곳에 있으면 환경변수가 application.yml을 완전히 덮어쓴다.** 이것이 이번 버그의 원인이다.

### 트러블슈팅

**배경**

- Service A가 Service B로 API 요청
- Service B는 X-Api-Key 헤더로 요청을 검증
- K8S secret에서 암호화된 API 키 주입, Jasypt로 복호화 처리

**증상: 401 계속 발생**

인증 필터 디버그 로그 추가 후 확인.

```text
Service A가 보내는 값: receivedKey=abc123xyz...  (길이=64, 암호화된 raw ciphertext)
Service B가 기대하는 값: expectedKey=plaintext123 (길이=32, 평문 API 키)
```

**Service A가 API 키를 복호화하지 않고 그대로 전송하고 있었다.**

**설정 구조 (문제 당시)**

```yaml
# application.yml
external-api:
  secret-key: ENC(${EXTERNAL_API_KEY})

# K8S Secret
EXTERNAL_API_KEY: base64_encoded_ciphertext
```

이론상 흐름은 다음과 같다.

1. K8S Secret에서 `EXTERNAL_API_KEY` 환경변수 주입
2. YAML에서 `ENC(${EXTERNAL_API_KEY})` 래핑
3. Jasypt가 `ENC()` 감지 후 AES-CBC로 복호화
4. 평문 API 키로 변환
5. Service B로 전송

**하지만, 실제로는 암호화된 값이 그대로 전송되고 있었다.**

### 원인: Relaxed Binding + Property Source 우선순위

**문제의 핵심**

K8S가 `EXTERNAL_API_KEY` (암호화된 값)를 Service A pod의 OS 환경변수로 주입한다. Spring Boot의 Relaxed Binding이 이 환경변수를 `external-api.secret-key`로 직접 매핑한다.

Property Source 우선순위에 의해 환경변수가 application.yml을 완전히 덮어쓴다.

**결과**

```text
YAML: external-api.secret-key: ENC(${EXTERNAL_API_KEY})
↓ (무시됨)
결과: external-api.secret-key = "encrypted_value" (ENC() 래핑 없음)
```

Jasypt는 `ENC()` prefix가 있을 때만 복호화한다. 없으면 그냥 원문 취급해서 그대로 사용한다.

따라서

- 암호화된 값이 그대로 API 키로 사용됨
- Service B로 전송: 암호화된 값
- Service B는 그 값을 복호화하려 해봤자 실패
- → 401 Unauthorized

그림으로 정리하면:

```text
[K8S Secret] EXTERNAL_API_KEY = encrypted_value
    ↓ relaxed binding + 우선순위
[Spring] external-api.secret-key = "encrypted_value" (YAML 무시됨)
    ↓ Jasypt: ENC() 없음 → 복호화 안 함
[결과] Service A에서 encrypted_value 그대로 사용
    ↓
[Service B] 인증 실패 → 401
```

**해결**

환경변수 이름을 변경했다.

**`EXTERNAL_API_KEY` → `SERVICE_A_API_KEY`**

```yaml
# 수정 후 application.yml
external-api:
  secret-key: ENC(${SERVICE_A_API_KEY})
```

이제 `SERVICE_A_API_KEY`는 relaxed binding으로 `service.a.api.key`에 매핑된다. `external-api.secret-key`와 충돌하지 않으므로, YAML의 `ENC()` 설정이 정상적으로 작동한다.

```text
[K8s Secret] SERVICE_A_API_KEY = encrypted_value
    ↓ relaxed binding: service.a.api.key로 매핑 (충돌 없음)
[Spring] external-api.secret-key = ENC(${SERVICE_A_API_KEY}) (YAML 사용됨!)
    ↓ Jasypt: ENC() 감지 → AES-CBC 복호화
[결과] 평문 API 키 생성
    ↓
[Service B] 인증 성공 → 200 OK
```

### 마무리

**Relaxed Binding은 편리한 기능이지만, 함정이 있다.**

환경변수 이름이 properties 이름과 의도치 않게 충돌하면, YAML 설정이 완전히 무시될 수 있다. 특히 **암호화 레이어(Jasypt)가 있을 때는 원인 추적이 매우 어렵다.** 로그에는 명확한 오류가 안 나오고, 단지 인증 실패 같은 상위 에러만 보인다.

**해결 전략**

이번 버그를 통해 배운 것.

1. **환경변수 이름에 서비스 접두사 붙이기**
   - `API_KEY` (❌ - 다른 곳과 충돌 가능)
   - `SERVICE_A_API_KEY` (✅ - 서비스별로 격리)
2. **@ConfigurationProperties 사용**
   - 명시적으로 계층 구조를 정의하면 충돌 여지가 줄어든다.
   - IDE 자동완성과 타입 안전성 제공
3. **설정 로그 확인**
   - Spring Boot 시작 시 설정 출력 활성화: `--debug` 또는 `--trace`
   - 어느 source에서 값이 왔는지 명확히 볼 수 있다.

K8S + Spring Boot 환경에서 비슷한 401/403 에러가 나타났다면,

- 먼저 로그가 아닌 **설정 매핑을 점검**하자.
- Relaxed Binding의 Property Source 우선순위를 의식하자.
- 암호화 설정이 있으면 `ENC()` 래핑 여부를 확인하자.
