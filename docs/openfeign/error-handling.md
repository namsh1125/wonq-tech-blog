# OpenFeign 오류 처리 및 폴백

분산 시스템에서 네트워크 통신은 항상 실패할 가능성이 있습니다. OpenFeign에서는 다양한 방식으로 API 호출 오류를 처리할 수 있습니다.

## 기본 오류 처리 방식

### FeignException 처리

기본적으로 Feign은 HTTP 오류 응답(4xx, 5xx)을 `FeignException`으로 변환합니다:

```java
try {
    User user = userClient.getUserById(1L);
} catch (FeignException e) {
    // HTTP 상태 코드 확인
    if (e.status() == 404) {
        log.error("사용자를 찾을 수 없습니다.");
    } else {
        log.error("API 호출 중 오류 발생: {}", e.getMessage());
    }
}
```

### FeignException 서브클래스 사용

Feign은 특정 HTTP 상태 코드에 대한 예외 서브클래스를 제공합니다:

```java
try {
    User user = userClient.getUserById(1L);
} catch (FeignException.NotFound e) {
    // 404 오류 처리
    log.error("사용자를 찾을 수 없습니다.");
} catch (FeignException.Unauthorized e) {
    // 401 오류 처리
    log.error("인증되지 않은 요청입니다.");
} catch (FeignException.BadRequest e) {
    // 400 오류 처리
    log.error("잘못된 요청입니다: {}", e.getMessage());
} catch (FeignException e) {
    // 그 외 모든 Feign 예외 처리
    log.error("API 호출 중 오류: {}", e.getMessage());
}
```

## 사용자 정의 ErrorDecoder 구현

복잡한 오류 처리가 필요한 경우 `ErrorDecoder` 인터페이스를 구현하여 HTTP 오류 응답을 커스텀 예외로 변환할 수 있습니다:

```java
@Component
public class CustomErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        switch (response.status()) {
            case 400:
                return new BadRequestException("잘못된 요청입니다.");
            case 401:
                return new UnauthorizedException("인증되지 않은 접근입니다.");
            case 403:
                return new ForbiddenException("접근 권한이 없습니다.");
            case 404:
                return new ResourceNotFoundException("리소스를 찾을 수 없습니다.");
            case 500:
                return new InternalServerException("서버 내부 오류가 발생했습니다.");
            default:
                // 응답 본문에서 오류 메시지 추출
                String errorMessage = extractErrorMessage(response);
                return new FeignClientException(response.status(), errorMessage);
        }
    }

    private String extractErrorMessage(Response response) {
        try {
            Reader reader = response.body().asReader(StandardCharsets.UTF_8);
            String body = Util.toString(reader);

            // JSON 응답에서 오류 메시지 추출 로직
            // 여기서는 단순화를 위해 전체 본문을 반환
            return body;
        } catch (IOException e) {
            return "오류 메시지를 추출할 수 없습니다.";
        }
    }
}
```

### ErrorDecoder 등록

구현한 ErrorDecoder는 다음과 같이 등록할 수 있습니다:

```java
@Configuration
public class FeignConfig {

    @Bean
    public ErrorDecoder errorDecoder() {
        return new CustomErrorDecoder();
    }
}

// 특정 클라이언트에만 적용
@FeignClient(
    name = "user-service",
    url = "https://api.example.com",
    configuration = FeignConfig.class
)
public interface UserClient {
    // 메서드 정의
}
```

## 폴백(Fallback) 메커니즘 구현

### 기본 폴백 클래스

서킷 브레이커 패턴을 구현하여 서비스 장애 시 대체 응답을 제공할 수 있습니다:

```java
@FeignClient(
    name = "user-service",
    url = "https://api.example.com",
    fallback = UserClientFallback.class
)
public interface UserClient {
    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);

    @GetMapping("/users")
    List<User> getAllUsers();
}

@Component
public class UserClientFallback implements UserClient {
    @Override
    public User getUserById(Long id) {
        // 기본값 반환 또는 대체 로직 실행
        User fallbackUser = new User();
        fallbackUser.setId(id);
        fallbackUser.setName("Unknown");
        fallbackUser.setEmail("unknown@example.com");
        return fallbackUser;
    }

    @Override
    public List<User> getAllUsers() {
        // 빈 목록 반환
        return Collections.emptyList();
    }
}
```

### 폴백 공장(Factory) 패턴

예외 정보를 활용하여 더 상세한 폴백 처리가 필요한 경우 `FallbackFactory`를 사용할 수 있습니다:

```java
@FeignClient(
    name = "user-service",
    url = "https://api.example.com",
    fallbackFactory = UserClientFallbackFactory.class
)
public interface UserClient {
    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);
}

@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    private static final Logger log = LoggerFactory.getLogger(UserClientFallbackFactory.class);

    @Override
    public UserClient create(Throwable cause) {
        // 예외 정보 로깅
        log.error("UserClient 폴백 발생: ", cause);

        return new UserClient() {
            @Override
            public User getUserById(Long id) {
                // 예외 유형에 따른 다른 처리
                if (cause instanceof FeignException.NotFound) {
                    log.warn("사용자 ID {}를 찾을 수 없습니다.", id);
                } else if (cause instanceof FeignException.ServiceUnavailable) {
                    log.error("사용자 서비스를 사용할 수 없습니다.");
                }

                // 기본 폴백 사용자 생성
                User fallbackUser = new User();
                fallbackUser.setId(id);
                fallbackUser.setName("Unknown (Service Error)");
                return fallbackUser;
            }
        };
    }
}
```

### Hystrix 또는 Resilience4j 설정

Spring Cloud 2020 이전 버전에서는 Hystrix를, 이후 버전에서는 Resilience4j를 서킷 브레이커로 사용할 수 있습니다.

**Resilience4j 설정 예:**

```yaml
feign:
  circuitbreaker:
    enabled: true

resilience4j:
  circuitbreaker:
    configs:
      default:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 5s
        failureRateThreshold: 50
        slowCallRateThreshold: 50
        slowCallDurationThreshold: 2s
    instances:
      user-service:
        baseConfig: default
```

## 실전 예제: 종합적인 오류 처리

아래 예제는 ErrorDecoder, 폴백, 그리고 로깅을 조합하여 견고한 오류 처리 전략을 구현하는 방법을 보여줍니다:

```java
// 커스텀 예외 클래스들
public class ApiException extends RuntimeException {
    private final int statusCode;

    public ApiException(String message, int statusCode) {
        super(message);
        this.statusCode = statusCode;
    }

    public int getStatusCode() {
        return statusCode;
    }
}

public class ResourceNotFoundException extends ApiException {
    public ResourceNotFoundException(String message) {
        super(message, 404);
    }
}

// ErrorDecoder 구현
@Component
public class DetailedErrorDecoder implements ErrorDecoder {

    private final ObjectMapper objectMapper;

    public DetailedErrorDecoder(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public Exception decode(String methodKey, Response response) {
        try {
            // 응답 본문 추출
            String responseBody = IOUtils.toString(response.body().asInputStream(), StandardCharsets.UTF_8);

            // 오류 응답의 세부 정보 추출 시도
            ErrorResponse errorResponse = parseErrorResponse(responseBody);

            switch (response.status()) {
                case 400:
                    return new BadRequestException(errorResponse.getMessage());
                case 404:
                    return new ResourceNotFoundException(errorResponse.getMessage());
                // 기타 상태 코드 처리...
                default:
                    return new ApiException(errorResponse.getMessage(), response.status());
            }
        } catch (Exception e) {
            // 응답 파싱 실패 시 기본 예외 반환
            return new ApiException("API 호출 오류: " + response.status(), response.status());
        }
    }

    private ErrorResponse parseErrorResponse(String responseBody) {
        try {
            return objectMapper.readValue(responseBody, ErrorResponse.class);
        } catch (Exception e) {
            // JSON 파싱 실패 시 기본 오류 응답 생성
            return new ErrorResponse("알 수 없는 오류", "세부 정보를 파싱할 수 없습니다");
        }
    }

    // 오류 응답 DTO
    @Data
    static class ErrorResponse {
        private String message;
        private String details;

        public ErrorResponse() {}

        public ErrorResponse(String message, String details) {
            this.message = message;
            this.details = details;
        }
    }
}

// 폴백 팩토리 구현
@Component
public class SmartFallbackFactory implements FallbackFactory<UserClient> {

    private static final Logger log = LoggerFactory.getLogger(SmartFallbackFactory.class);

    @Override
    public UserClient create(Throwable cause) {
        log.error("UserClient 폴백 발생", cause);

        return new UserClient() {
            @Override
            public User getUserById(Long id) {
                if (cause instanceof ResourceNotFoundException) {
                    // 캐시에서 사용자 조회 시도
                    User cachedUser = userCacheService.getUserFromCache(id);
                    if (cachedUser != null) {
                        return cachedUser;
                    }
                }

                // 기본 폴백 응답
                return createFallbackUser(id, cause);
            }

            // 기타 메서드 구현...
        };
    }

    private User createFallbackUser(Long id, Throwable cause) {
        User fallbackUser = new User();
        fallbackUser.setId(id);

        // 예외 유형에 따른 다른 메시지 설정
        if (cause instanceof ResourceNotFoundException) {
            fallbackUser.setName("[사용자를 찾을 수 없음]");
        } else if (cause instanceof TimeoutException) {
            fallbackUser.setName("[시간 초과]");
        } else {
            fallbackUser.setName("[서비스 오류]");
        }

        return fallbackUser;
    }
}

// Feign 클라이언트 설정
@Configuration
public class FeignClientErrorConfig {

    @Bean
    public ErrorDecoder errorDecoder(ObjectMapper objectMapper) {
        return new DetailedErrorDecoder(objectMapper);
    }
}

// Feign 클라이언트 정의
@FeignClient(
    name = "user-service",
    configuration = FeignClientErrorConfig.class,
    fallbackFactory = SmartFallbackFactory.class
)
public interface UserClient {
    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);

    // 기타 메서드...
}
```

## 모범 사례 및 팁

1. **명확한 예외 계층 구조**: 서비스별로 구체적인 예외 클래스를 설계하세요.
2. **폴백 전략의 상세화**: 단순히 기본값을 반환하는 것을 넘어서, 상황에 따른 다양한 대체 전략을 고려하세요.
3. **타임아웃 설정**: API 호출 타임아웃을 적절하게 설정하여 장시간 대기를 방지하세요.
4. **재시도 정책**: 일시적인 오류에 대해 자동 재시도 메커니즘을 구현하세요.
5. **로깅과 모니터링**: 오류 발생 시 충분한 정보를 로깅하고, 필요하면 모니터링 시스템에 알림을 보내세요.

## 정리

OpenFeign에서 오류 처리는 크게 세 가지 방식으로 구현할 수 있습니다:

1. **기본 예외 처리**: `FeignException`과 그 서브클래스를 사용한 예외 처리
2. **커스텀 ErrorDecoder**: HTTP 응답을 애플리케이션별 예외로 변환
3. **폴백 메커니즘**: 서비스 장애 시 대체 응답 제공

마이크로서비스 아키텍처에서는 견고한 오류 처리와 폴백 전략이 필수적이므로, 이러한 패턴을 적극적으로 활용하세요.

## 참고 리소스

- [OpenFeign GitHub - ErrorDecoder](https://github.com/OpenFeign/feign/blob/master/core/src/main/java/feign/codec/ErrorDecoder.java)
- [Spring Cloud OpenFeign 오류 처리 가이드](https://cloud.spring.io/spring-cloud-openfeign/reference/html/)
- [Resilience4j 문서](https://resilience4j.readme.io/docs/circuitbreaker)
- [마이크로서비스 패턴 - 서킷 브레이커](https://microservices.io/patterns/reliability/circuit-breaker.html)
