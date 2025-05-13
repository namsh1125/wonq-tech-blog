---
sidebar_position: 1
---

# OpenFeign 소개

## OpenFeign이란?

**OpenFeign**은 Java에서 선언적 방식으로 REST API 클라이언트를 작성할 수 있도록 도와주는 HTTP 클라이언트입니다. Netflix에서 시작된 Feign 프로젝트가 오픈소스 커뮤니티로 이전되어 OpenFeign으로 발전했으며, 현재는 Spring Cloud OpenFeign을 통해 Spring 생태계와 통합되어 널리 사용됩니다.

## 핵심 특징

### 선언적 인터페이스 기반

OpenFeign은 **선언적 방식**으로 API 클라이언트를 정의합니다. 복잡한 로직 없이 간단한 선언만으로 외부 API와 통신이 가능합니다.

```Java title="com/example/feign/UserClient.java"
package com.example.feign;

@FeignClient(name = "user-service", url = "https://api.example.com")
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);
}
```

### Spring Cloud OpenFeign의 확장 기능

Spring Cloud는 OpenFeign에 다양한 기능을 추가하여 마이크로서비스 환경에 최적화된 통신 기능을 제공합니다.

| 기능                  | 설명                                                             |
| --------------------- | ---------------------------------------------------------------- |
| **서비스 디스커버리** | `Eureka`와 연동하여 서비스 이름 기반으로 동적 URL을 해석         |
| **로드 밸런싱**       | `Spring Cloud LoadBalancer`로 클라이언트 사이드 로드 밸런싱 제공 |
| **서킷 브레이커**     | `Resilience4j` 또는 `Spring Cloud Circuit Breaker`와 통합 가능   |
| **인증 지원**         | `Spring Security OAuth2`와 연동하여 토큰 자동 부착               |
| **GZIP 압축**         | 요청/응답에 GZIP 압축 적용 가능 (속도 향상)                      |
| **모니터링**          | `Micrometer`와 연동하여 호출 지표 수집 가능                      |
| **캐싱**              | `@Cacheable` 등을 통한 응답 캐싱 가능 (선택 사항)                |

### 다른 HTTP 클라이언트와 비교

| 클라이언트    | 특징                         | 코드 복잡성 | 선언적 여부 |
| ------------- | ---------------------------- | ----------- | ----------- |
| **OpenFeign** | 인터페이스 기반, Spring 통합 | 낮음        | 높음        |
| RestTemplate  | 명령형, 템플릿 패턴          | 중간        | 낮음        |
| WebClient     | 비동기, 리액티브 프로그래밍  | 높음        | 중간        |

## 왜 OpenFeign을 사용하는가?

OpenFeign은 마이크로서비스 환경에서 다음과 같은 이유로 선택됩니다:

- **간소화된 코드**: 반복 코드를 줄여 개발 생산성 향상.
- **명확한 가독성**: 인터페이스로 API 의도 명확히 표현.
- **테스트 용이성**: Mock 객체로 단위 테스트 간편.
- **확장성**: Spring Cloud 기능과 통합으로 유연한 확장

## OpenFeign의 아키텍처

OpenFeign은 모듈화된 아키텍처로, 다음 구성 요소로 구성됩니다:

| 구성 요소              | 역할                                                        |
| ---------------------- | ----------------------------------------------------------- |
| **Contract**           | 인터페이스 어노테이션을 HTTP 요청으로 변환                  |
| **Client**             | 실제 HTTP 요청 수행 (예: OkHttp, Apache HttpClient)         |
| **Encoder/Decoder**    | 요청/응답의 JSON을 직렬화/역직렬화 수행 (예: Jackson, Gson) |
| **Logger**             | 요청/응답 로깅 (NONE, BASIC, HEADERS, FULL 등)              |
| **ErrorDecoder**       | 예외 처리용 커스텀 에러 디코더                              |
| **Retryer**            | 실패한 요청 재시도 전략 설정                                |
| **RequestInterceptor** | 요청 인터셉터 (예: 헤더에 인증 토큰 추가)                   |

Spring Cloud OpenFeign은 이러한 구성 요소를 Eureka, LoadBalancer 등과 통합하여 마이크로서비스에 최적화합니다.
