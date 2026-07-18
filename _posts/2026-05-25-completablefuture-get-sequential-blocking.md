---
title: 순차적 작업 처리 시, CompletableFuture.get() 사용에 대해
date: 2026-05-25 23:34:46 +0900
categories: [Dev, Troubleshooting]
tags: [Java, Spring Boot, CompletableFuture, Async]
---

**CompletableFuture**는 Java 8부터 표준 라이브러리에 포함된 클래스로, 비동기 작업을 처리하는 데에 주로 사용되는 클래스입니다.

이걸 순차적인 작업이 필요한 곳에 사용하게 되면 자칫 성능을 악화시킬 가능성이 있습니다.

다음과 같은 코드가 그 예시입니다.

```java
CompletableFuture.supplyAsync(() -> {
    result = doStep1();         // 300ms
    result = doStep2(result);   // 200ms
    result = doStep3(result);   // 150ms
    return result;              // 총 650ms
}, executor).get();
```

각 단계는 이전 단계의 결과에 순차적으로 의존합니다.

별도 스레드로 처리되는 것처럼 보이지만, **실제로는 아닐 수 있습니다.**

**supplyAsync()**는 별도 스레드에 할당되고 **.get()**은 그 스레드가 작업을 완료할 때까지 현재 스레드를 **블로킹**합니다.

결과적으로,

> 할당된 스레드: 650ms 작업중
>
> 메인 스레드: **.get()**에서 650ms 기다림
>
> **두 스레드 모두 점유하는 현상 발생**

초당 100개 요청이 들어오면, 100개 스레드가 동시에 점유됩니다.

지연 발생 시 500개 이상의 스레드가 필요할 수도 있습니다.

이를 또 대비하기 위해, 스레드풀을 과다 설정하게 되고 그것이 **메모리 낭비와 성능 악화**로 이어집니다.

### 그럼 CompletableFuture.get()을 사용하지 않아야 하나?

**future.get() 자체가 문제되지는 않습니다.**

분명 필요한 케이스도 있습니다.

```java
// 3개의 독립적 API를 동시에 호출
CompletableFuture f1 = supplyAsync(() -> callAPI1());   // 300ms
CompletableFuture f2 = supplyAsync(() -> callAPI2());   // 200ms
CompletableFuture f3 = supplyAsync(() -> callAPI3());   // 150ms

// 결과를 수집
Result1 r1 = f1.get();
Result2 r2 = f2.get();
Result3 r3 = f3.get();
```

이 경우 최대 소요 시간이 650ms → 300ms으로 단축되며, 의미있는 **CompletableFuture.get()** 사용으로 볼 수 있습니다.

저의 경우에는 **CompletableFuture.get()** 적용 전에 아래 이유들을 따져볼 것 같습니다.

1. 작업들이 동시에 처리되어야 하는가?
2. 코드에 적용했을 때, 분명한 성능 효과가 눈에 보이는가?
3. 로그, 캐시, 알림 등의 응답을 기다리지 않아도 되는 작업들인가?

이 중 어느 것도 해당되지 않는다면, CompletableFuture.get() 사용은 불필요하며 **단순한 동기 처리로도 충분**할 것으로 보입니다.
