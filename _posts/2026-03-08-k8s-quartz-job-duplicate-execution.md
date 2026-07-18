---
title: Kubernetes + Quartz 환경 Job 중복 실행된 원인
date: 2026-03-08 23:17:04 +0900
categories: [Dev, Troubleshooting]
tags: [Kubernetes, Quartz, Cluster]
---

### 요약

Kubernetes, 멀티 파드 환경에서 Quartz 스케줄러를 클러스터 모드로 잘 사용하고 있다고 생각했다. 하지만, `instanceId`가 구분되어 있지 않아, 실제로는 정상 동작하지 않고 있었다.

확인해보니 **Job이 파드마다 한 번씩 실행**되고 있었으며, `instanceId=AUTO`와 `clusterCheckinInterval` 설정을 통해 정상 복구하였다.

### 문제 상황

Kubernetes 환경에서 Quartz 스케줄러를 다음과 같이 구성하여 사용 중이었다.

- `isClustered=true`
- 동일 서비스 파드 2개
- 동일 DB를 사용하는 JDBC JobStore

Quartz 클러스터 모드에서는 동일 Job이 **클러스터 내 한 노드에서만 실행**되는 것이 정상이다.

하지만 운영 중 다음과 같은 현상이 확인되었다.

- 동일 Job이 **두 파드에서 각각 실행**
- 실행 시점은 완전히 동일하지 않고 **수 초 차이 발생**

클러스터 설정이 되어 있기 때문에, 처음에는 단순 스케줄링 타이밍 문제라고 생각하였지만, 동일 Job이 반복적으로 두 번 실행되면서 설정을 다시 확인하게 되었다.

### 원인

Quartz 클러스터는 여러 Scheduler 인스턴스가 **같은 DB를 공유**하면서 동작한다. Trigger 실행 시 DB lock을 통해 **한 노드만 Job을 실행**하도록 설계되어 있다.

Quartz 공식 문서에서도 다음과 같이 설명한다.

> Only one node will fire the job for each firing.

하지만 Quartz 관련 테이블을 확인해보니 클러스터 상태 관리가 정상적으로 이루어지지 않고 있었다. **문제는 `instanceId` 설정이었다.**

Quartz 클러스터 환경에서는 **각 Scheduler 인스턴스가 반드시 고유한 `instanceId`를 가져야 한다.**

> Each node in the cluster MUST have a unique instanceId.

하지만, 기존 설정에서는 모든 파드가 **동일한 `instanceId`**를 사용하고 있었으며, Quartz가 클러스터 노드를 구분하지 못하는 상태였다.

결과적으로 `isClustered=true`가 설정되어 있어도 실제로는 클러스터처럼 동작하지 않는 상태였다.

### 해결 방법

**1. instanceId 자동 생성**

```properties
org.quartz.scheduler.instanceId=AUTO
```

AUTO 설정을 사용하면 Scheduler 인스턴스마다 고유 ID가 자동 생성된다. Kubernetes처럼 파드가 동적으로 생성되는 환경에서는 일반적으로 이 설정을 사용한다.

**2. clusterCheckinInterval 설정**

```properties
org.quartz.jobStore.clusterCheckinInterval=20000
```

Scheduler가 주기적으로 **DB에 상태(heartbeat)를 기록**하는 간격이다. 이를 통해 Quartz는 다른 노드 상태 확인, 장애 노드 감지, Job takeover 같은 클러스터 기능을 수행한다.

### 결과

설정 적용 후 다음과 같이 동작이 정상화 되었다.

- 파드별 Scheduler 인스턴스 정상 구분
- Quartz 클러스터 모드 정상 동작
- 동일 Job이 한 파드에서만 실행

### 정리

Kubernetes 환경에서 Quartz 클러스터를 사용할 때 최소한 다음 설정은 확인하는 것이 좋다.

```properties
org.quartz.jobStore.isClustered=true
org.quartz.scheduler.instanceId=AUTO
org.quartz.jobStore.clusterCheckinInterval=20000
```

특히, instanceId는 클러스터 노드마다 반드시 unique 해야 한다. 이 설정이 잘못되면 `isClustered=true`가 설정되어 있어도 의도한 클러스터 동작이 보장되지 않을 수 있다.

### Ref.

- [Quartz JDBC JobStore Clustering](https://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigJDBCJobStoreClustering.html)
- [Quartz Configuration](https://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/)
