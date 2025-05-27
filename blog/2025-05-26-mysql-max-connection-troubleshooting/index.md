---
slug: mysql-max-connection-troubleshooting
title: MySQL max_connections 초과 문제 해결기
authors: polar
tags: [ mysql, hikaricp, connection-pool, database, spring-boot, troubleshooting ]
---

# MySQL max_connections 초과 문제 해결기: 커넥션 풀 최적화와 인프라 개선

안녕하세요, 원큐 오더 PL 남승현입니다.

오늘은 MySQL에서 발생한 "Too many connections" 오류를 해결한 경험을 공유하려고 해요.

## 문제 상황: 갑작스러운 연결 거부

2025년 5월 26일 오후 4시 25분경, 개발하고 있던 애플리케이션이 데이터베이스에 연결할 수 없다는 오류가 발생했어요.

```text
Caused by: java.sql.SQLNonTransientConnectionException: Connection exception, SQL-server rejected establishment of SQL-connection,  message from server: "Too many connections"

Caused by: jakarta.persistence.PersistenceException: [PersistenceUnit: default] Unable to build Hibernate SessionFactory; nested exception is org.hibernate.exception.JDBCConnectionException: Unable to open JDBC Connection for DDL execution [Connection exception, SQL-server rejected establishment of SQL-connection,  message from server: "Too many connections"] [n/a]
```

위 에러 메시지가 발생하면서 대부분의 팀원들이 개발을 진행할 수 없게 되었죠.

## 원인: 제한된 max_connections

### 1. MySQL 프로세스 리스트 확인

가장 먼저 현재 MySQL에 연결된 프로세스들을 확인해봤어요.

```text
mysql> SHOW PROCESSLIST;
+-------+-----------------+----------------------+------------+---------+--------+------------------------+------------------+
| Id    | User            | Host                 | db         | Command | Time   | State                  | Info             |
+-------+-----------------+----------------------+------------+---------+--------+------------------------+------------------+
|     5 | event_scheduler | localhost            | NULL       | Daemon  | 590667 | Waiting on empty queue | NULL             |
| 17092 | root            | 118.131.63.238:61472 | pg         | Sleep   |   2291 |                        | NULL             |
| 17093 | root            | 118.131.63.238:61473 | pg         | Sleep   |   2291 |                        | NULL             |
| 17961 | root            | 118.131.63.238:64415 | wonq_db    | Sleep   |   7735 |                        | NULL             |
| ...   |                 |                      |            |         |        |                        |                  |
| 18500 | root            | 34.47.94.140:57162   | wonq_db    | Sleep   |     26 |                        | NULL             |
| 18501 | root            | 34.64.193.105:38454  | pg         | Sleep   |      8 |                        | NULL             |
| 18502 | root            | 34.64.193.105:38456  | pg         | Sleep   |      7 |                        | NULL             |
| 18503 | root            | 34.64.193.105:54398  | pg         | Sleep   |      3 |                        | NULL             |
+-------+-----------------+----------------------+------------+---------+--------+------------------------+------------------+
153 rows in set, 1 warning (0.011 sec)
```

확인해보니 **153개의 연결**이 활성화되어 있었고, 대부분이 `Sleep` 상태로 유지되고 있었어요.
이는 커넥션이 사용 후 적절히 반환되지 않거나, 커넥션 풀 설정이 최적화되지 않았음을 나타내는 것이었죠.

### 2. max_connections 설정 확인

그 다음으로 MySQL의 `max_connections` 설정을 확인해봤어요.

```text
mysql> SHOW VARIABLES LIKE 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
+-----------------+-------+
```

`max_connections` 값을 확인해보니 **151**로 설정되어 있었어요.
`max_connections` 값은 MySQL 서버가 동시에 처리할 수 있는 최대 연결 수를 의미하는데, 현재 153개의 연결이 활성화되어 있었으니, 이 **설정을 초과**한 상태였죠.

## 해결 과정

### 1. 임시 조치: MySQL max_connections 증가

먼저 가장 직접적인 해결책으로 MySQL의 최대 연결 수를 늘렸어요:

```sql
SET GLOBAL max_connections = 200;
```

해당 명령어를 실행하고 나니까 애플리케이션이 정상적으로 데이터베이스에 연결할 수 있게 되었어요.
하지만 단순히 연결 한도를 늘리는 것은 시스템 자원을 더 많이 소비하게 되므로, 이것만으로는 근본적인 해결책이 될 수 없었죠.

그래서 저희는 환경별로 커넥션 풀을 최적화하기로 결정했어요.

### 2. 환경별 HikariCP 커넥션 풀 최적화

HikariCP는 [공식적으로 커넥션 풀 크기를 설정하는 방법](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)을 제공하고 있어요.

> connections = ((core_count \* 2) + effective_spindle_count)

이를 기반으로 각 환경에 맞는 커넥션 풀 크기를 계산해보니 다음과 같았어요:

#### 로컬 환경 (개발자 개인 PC)

- **하드웨어**: 일반적으로 4-8 코어 CPU
- **이론적 적정값**:
    - 4코어 기준: (4 \* 2) + n >= 9개 커넥션
    - 8코어 기준: (8 \* 2) + n >= 17개 커넥션
- **커넥션 설정값**: 2개 ~ 3개
- **설정값 결정 이유**: 로컬 환경에서는 서비스가 정상 작동하는지 확인하기 위해 총 5개의 서버를 구동해야 했어요.
  로컬 환경에서 사용할 수 있는 커넥션 수가 약 9개에서 17개 사이로 확인되어, 각 서버에서 2~3개의 커넥션을 사용하도록 설정했어요.

#### 개발 환경 (GCP e2-medium 인스턴스)

- **하드웨어**: 2 vCPU
- **이론적 적정값**: (2 \* 2) + 1 = 5개 커넥션
- **커넥션 설정값**: 5개 ~ 10개
- **설정값 결정 이유**: 개발 환경에서는 인스턴스 1대에 서버 1개가 구동되요.
  따라서 각 서버가 사용할 수 있는 커넥션 수를 5개에서 기본 값인 10개 사이로 설정했어요.

**로컬 환경 설정:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 3 # 최대 커넥션 수
      minimum-idle: 2 # 최소 유지 커넥션 수
      connection-timeout: 30000 # 커넥션 대기 시간 (30초)
      idle-timeout: 600000 # 유휴 커넥션 유지 시간 (10분)
      max-lifetime: 1800000 # 커넥션 최대 생존 시간 (30분)
```

**개발 환경 설정:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10 # 최대 커넥션 수
      minimum-idle: 5 # 최소 유지 커넥션 수
      connection-timeout: 30000 # 커넥션 대기 시간 (30초)
      idle-timeout: 600000 # 유휴 커넥션 유지 시간 (10분)
      max-lifetime: 1800000 # 커넥션 최대 생존 시간 (30분)
```

위 설정값을 통해 각 환경에 맞는 커넥션 풀 크기를 최적화했어요. 이를 통해 불필요한 커넥션을 줄이고, 데이터베이스의 부하를 최소화할 수 있었죠.

## 결론 및 교훈

이번 MySQL "Too many connections" 오류 해결 과정에서 얻은 교훈은 다음과 같아요:

1. **환경별 차별화된 접근**: 기존에는 모든 환경에서 동일한 커넥션 풀 설정을 사용했지만, 앞으로는 각 환경에 맞게 최적화된 설정을 적용하고자 해요.

2. **모니터링 체계 구축**: 프로젝트 개발에 집중한 나머지 데이터베이스 연결 상태를 모니터링하는 체계가 부족했어요.
   앞으로는 데이터베이스 연결 상태를 주기적으로 모니터링하고, 문제가 발생하기 전에 사전 대응할 수 있는 체계를 마련할 계획이에요.

이러한 교훈을 통해 앞으로는 유사한 문제가 발생하지 않도록 예방하고, 안정적인 개발 환경을 유지하는데 노력할 거예요.

혹시 더 궁금한 점이나 다른 질문이 있다면 언제든지 문의해 주세요!

## 참고

### HikariCP 설정값 (일부)

| 설정 값               | 설명                                                         | 기본값                        |
|--------------------|------------------------------------------------------------|----------------------------|
| maximum-pool-size  | 동시에 사용할 수 있는 최대 커넥션 수예요. 너무 크면 메모리 낭비, 너무 작으면 성능 저하가 발생해요. | 10                         |
| minimum-idle       | 항상 유지되는 최소 커넥션 수로, 갑작스러운 요청 증가에 대비해요.                      | maximum-pool-size와 동일하게 설정 |
| connection-timeout | 커넥션을 얻기 위해 대기하는 최대 시간이에요. (ms)                             | 30000 (30초)                |
| idle-timeout       | 사용되지 않는 커넥션이 풀에서 제거되기까지의 시간이에요. (ms)                       | 600000 (10분)               |
| max-lifetime       | 커넥션의 최대 생존 시간으로, 이를 통해 오래된 커넥션을 주기적으로 갱신해요. (ms)           | 1800000 (30분)              |

_[HikariCP Github](https://github.com/brettwooldridge/HikariCP)_

### MySQL 설정값

| 설정 값            | 설명                                                                  | 기본값 |
|-----------------|---------------------------------------------------------------------|-----|
| max_connections | MySQL 서버가 동시에 처리할 수 있는 최대 연결 수예요. 너무 작으면 연결 거부, 너무 크면 메모리 낭비가 발생해요. | 151 |

_[MySQL Documentation](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html#sysvar_max_connections)_
