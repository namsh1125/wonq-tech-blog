# OpenFeign 성능 최적화

마이크로서비스 아키텍처에서 서비스 간 통신 성능은 전체 애플리케이션 성능에 큰 영향을 미칩니다. 이 문서에서는 OpenFeign을 사용할 때 적용할 수 있는 다양한 성능 최적화 기법을 알아봅니다.

## HTTP 클라이언트 최적화

### 기본 클라이언트 vs 최적화된 클라이언트

OpenFeign은 기본적으로 JDK의 `URLConnection`을 사용합니다. 그러나 성능과 기능을 위해 Apache HttpClient나 OkHttp와 같은 다른 HTTP 클라이언트를 사용할 수 있습니다.

#### Apache HttpClient 사용하기

```xml
<!-- Maven 의존성 -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
</dependency>
```

```yaml
# application.yml
feign:
  httpclient:
    enabled: true # Apache HttpClient 활성화
    max-connections: 200 # 최대 연결 수
    max-connections-per-route: 50 # 라우트당 최대 연결 수
    connection-timeout: 2000 # 연결 타임아웃 (ms)
    connection-timer-repeat: 3000 # 연결 타이머 반복 주기 (ms)
```

#### OkHttp 클라이언트 사용하기

```xml
<!-- Maven 의존성 -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

```yaml
# application.yml
feign:
  okhttp:
    enabled: true # OkHttp 클라이언트 활성화
```

OkHttp 클라이언트 추가 구성:

```java
@Configuration
public class OkHttpConfig {

    @Bean
    public OkHttpClient client() {
        return new OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .connectionPool(new ConnectionPool(50, 5, TimeUnit.MINUTES))
            .build();
    }
}
```

> 💡 **성능 팁**: Apache HttpClient와 OkHttp는 모두 연결 풀링을 지원하므로, 기본 JDK HttpURLConnection보다 성능이 뛰어납니다.

## 연결 풀링 최적화

연결 풀을 효과적으로 구성하면 API 호출 성능을 크게 향상시킬 수 있습니다.

### 최적의 풀 사이즈 계산

일반적인 가이드라인:

```
최적 풀 사이즈 = (서비스에 대한 초당 요청 수 * 평균 요청 지연 시간) + 여유분
```

예를 들어:

- 초당 요청 수: 100
- 평균 요청 지연 시간: 0.5초
- 계산: 100 \* 0.5 = 50
- 여유분 추가: 약 70-100 사이의 풀 사이즈 설정이 합리적

### 팁별 연결 풀 구성

특정 서비스나 API에 따라 다른 연결 풀 설정이 필요할 수 있습니다:

```java
@Configuration
public class CustomFeignConfig {

    @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder(CachingCapability cachingCapability) {
        return Feign.builder()
            .client(new ApacheHttpClient(createHttpClient()))
            .addCapability(cachingCapability);
    }

    private CloseableHttpClient createHttpClient() {
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectTimeout(5000)
            .setConnectionRequestTimeout(5000)
            .setSocketTimeout(5000)
            .build();

        PoolingHttpClientConnectionManager connManager =
            new PoolingHttpClientConnectionManager();
        connManager.setMaxTotal(200);
        connManager.setDefaultMaxPerRoute(50);

        return HttpClients.custom()
            .setConnectionManager(connManager)
            .setDefaultRequestConfig(requestConfig)
            .build();
    }
}
```

## 압축 활성화

데이터 전송량을 줄이기 위해 요청 및 응답 압축을 사용할 수 있습니다:

```yaml
feign:
  compression:
    request:
      enabled: true
      mime-types: text/xml,application/xml,application/json # 압축할 MIME 타입
      min-request-size: 2048 # 최소 압축 크기 (바이트)
    response:
      enabled: true # 응답 압축 활성화
```

> ⚠️ **주의사항**: 압축은 CPU 사용량을 증가시킬 수 있으므로, 작은 페이로드나 매우 높은 처리량이 필요한 서비스에서는 주의해서 사용하세요.

## 캐싱 전략

자주 액세스하고 자주 변경되지 않는 데이터에 대해 캐싱을 구현하면 성능을 크게 향상시킬 수 있습니다.

### Spring Cache 통합

```java
@Configuration
@EnableCaching
public class CachingConfig {

    @Bean
    public CacheManager cacheManager() {
        // 다양한 캐시 구현체 사용 가능 (Caffeine, Redis, Ehcache 등)
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(Arrays.asList(
            new ConcurrentMapCache("users"),
            new ConcurrentMapCache("products"),
            new ConcurrentMapCache("configurations")
        ));
        return cacheManager;
    }
}

@FeignClient(name = "user-service")
public interface UserClient {

    @Cacheable(value = "users", key = "#id")
    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);

    @CacheEvict(value = "users", key = "#user.id")
    @PutMapping("/users/{id}")
    User updateUser(@PathVariable("id") Long id, @RequestBody User user);
}
```

### Caffeine 캐시 설정 예제

Caffeine은 고성능 Java 캐시 라이브러리로, 더 세밀한 캐시 설정이 가능합니다:

```java
@Configuration
@EnableCaching
public class CaffeineConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();

        cacheManager.setCaffeine(Caffeine.newBuilder()
            // 최대 항목 수 제한
            .maximumSize(10000)
            // 마지막 접근 후 만료 시간
            .expireAfterAccess(30, TimeUnit.MINUTES)
            // 항목 생성 후 만료 시간
            .expireAfterWrite(1, TimeUnit.HOURS)
            // 항목 제거 시 콜백
            .removalListener((key, value, cause) ->
                log.info("캐시 항목 제거: {}, 원인: {}", key, cause))
            // 통계 활성화
            .recordStats()
        );

        cacheManager.setAllowNullValues(false);
        return cacheManager;
    }
}
```

## 요청 일괄 처리(Batching)

여러 API 요청을 병렬로 처리하거나 일괄 처리하면 성능이 향상될 수 있습니다.

### CompletableFuture를 사용한 병렬 호출

```java
@Service
public class ProductService {

    private final ProductClient productClient;
    private final InventoryClient inventoryClient;
    private final PricingClient pricingClient;

    // 생성자 주입 생략

    public ProductDetail getProductDetail(Long productId) {
        // 비동기 호출 시작
        CompletableFuture<Product> productFuture = CompletableFuture.supplyAsync(
            () -> productClient.getProduct(productId));

        CompletableFuture<Inventory> inventoryFuture = CompletableFuture.supplyAsync(
            () -> inventoryClient.getInventory(productId));

        CompletableFuture<PriceInfo> priceFuture = CompletableFuture.supplyAsync(
            () -> pricingClient.getPrice(productId));

        // 모든 비동기 호출 완료 대기
        return CompletableFuture.allOf(
            productFuture, inventoryFuture, priceFuture)
            .thenApply(v -> {
                // 모든 결과 조합
                return new ProductDetail(
                    productFuture.join(),
                    inventoryFuture.join(),
                    priceFuture.join()
                );
            }).join();
    }
}
```

### 일괄 API 호출 구현

다수의 항목을 개별적으로 조회하는 대신 일괄 API를 구현하면 네트워크 오버헤드를 줄일 수 있습니다:

```java
@FeignClient(name = "user-service")
public interface UserClient {

    // 단일 사용자 조회 대신
    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);

    // 다중 사용자 일괄 조회
    @PostMapping("/users/batch")
    List<User> getUsersByIds(@RequestBody List<Long> ids);
}

@Service
public class UserService {

    private final UserClient userClient;

    public List<User> getUsersByIds(List<Long> ids) {
        // 작은 배치로 나누어 호출
        return Lists.partition(ids, 100).stream()
            .map(batch -> userClient.getUsersByIds(batch))
            .flatMap(List::stream)
            .collect(Collectors.toList());
    }
}
```

## 지연 시간 모니터링 및 최적화

### Feign Logger 설정

Feign 클라이언트의 로깅을 활성화하여 API 호출 시간을 모니터링할 수 있습니다:

```yaml
logging:
  level:
    com.example.clients: DEBUG # Feign 클라이언트 패키지

feign:
  client:
    config:
      default:
        loggerLevel: FULL # NONE, BASIC, HEADERS, FULL 중 선택
```

```java
@Configuration
public class FeignLoggingConfig {

    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

### 커스텀 성능 로깅 인터셉터

더 상세한 성능 메트릭을 수집하기 위한 커스텀 인터셉터:

```java
public class PerformanceLoggingInterceptor implements RequestInterceptor {

    private static final Logger log = LoggerFactory.getLogger(PerformanceLoggingInterceptor.class);
    private final MeterRegistry meterRegistry;

    public PerformanceLoggingInterceptor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void apply(RequestTemplate template) {
        String requestId = UUID.randomUUID().toString();
        template.header("X-Request-ID", requestId);

        // 요청 시작 시간 기록
        template.header("X-Request-Start-Time", String.valueOf(System.currentTimeMillis()));

        // 메트릭 태그 준비
        String apiEndpoint = template.path();
        String method = template.method();

        // 카운터 증가
        meterRegistry.counter("api.calls",
            "endpoint", apiEndpoint,
            "method", method).increment();
    }
}

@Configuration
public class FeignMetricsConfig {

    @Bean
    public FeignClientMetricsInterceptor metricsInterceptor(MeterRegistry meterRegistry) {
        return new PerformanceLoggingInterceptor(meterRegistry);
    }
}
```

## 타임아웃 및 재시도 최적화

### 타임아웃 설정

적절한 타임아웃을 설정하여 느린 서비스가 전체 시스템에 영향을 미치지 않도록 합니다:

```yaml
feign:
  client:
    config:
      default: # 모든 클라이언트에 적용
        connectTimeout: 5000 # 5초
        readTimeout: 5000 # 5초
      user-service: # 특정 서비스에 다른 값 적용
        connectTimeout: 3000
        readTimeout: 3000
```

### 재시도 전략 구성

일시적인 오류에 대한 자동 재시도 설정:

```java
@Configuration
public class FeignRetryConfig {

    @Bean
    public Retryer retryer() {
        // 초기 대기 시간(ms), 최대 대기 시간(ms), 최대 재시도 횟수
        return new Retryer.Default(100, 2000, 3);
    }
}
```

또는 Resilience4j와 통합하여 더 세밀한 재시도 설정:

```java
@Bean
public RetryPolicy retryPolicy() {
    return RetryPolicy.builder()
        .handle(ConnectException.class, SocketTimeoutException.class)
        .withDelay(Duration.ofMillis(100))
        .withMaxRetries(3)
        .withBackoff(100, 1000, 1.5, TimeUnit.MILLISECONDS)
        .build();
}

@Bean
public Retry retry(RetryPolicy retryPolicy) {
    return Retry.of("apiRetry", retryPolicy);
}
```

## 성능 테스트 및 벤치마킹

### 부하 테스트 도구 활용

JMeter, Gatling 또는 k6와 같은 도구를 사용하여 Feign 클라이언트의 성능을 테스트합니다:

```java
// Gatling 시나리오 예제
public class FeignClientSimulation extends Simulation {

    HttpProtocolBuilder httpProtocol = http
        .baseUrl("http://localhost:8080")
        .acceptHeader("application/json")
        .userAgentHeader("Gatling/Performance Test");

    ScenarioBuilder getUserScenario = scenario("Get User API")
        .exec(http("Get User")
            .get("/api/users/1")
            .check(status().is(200))
        );

    {
        setUp(
            getUserScenario.injectOpen(
                rampUsers(100).during(30),
                constantUsersPerSec(10).during(60)
            )
        ).protocols(httpProtocol);
    }
}
```

## 종합적인 성능 최적화 전략

마이크로서비스 환경에서 Feign 클라이언트 성능을 최적화하는 종합 전략:

1. **최적화된 HTTP 클라이언트 사용**: Apache HttpClient 또는 OkHttp
2. **연결 풀링 튜닝**: 서비스 특성에 맞는 풀 사이즈 설정
3. **압축 활성화**: 큰 페이로드에 대해 요청/응답 압축 사용
4. **효과적인 캐싱 구현**: 자주 조회되고 변경이 적은 데이터를 캐싱
5. **병렬 처리 및 일괄 처리**: 여러 API 호출을 동시에 처리
6. **타임아웃 최적화**: 서비스별 적절한 타임아웃 설정
7. **재시도 전략 구현**: 일시적인 오류에 대한 지능적 재시도
8. **모니터링 및 로깅**: 성능 메트릭 수집 및 분석
9. **적절한 직렬화/역직렬화 방식 선택**: 필요에 따라 JSON, Protocol Buffers 등 선택
10. **부하 테스트 및 튜닝**: 실제 트래픽 패턴을 시뮬레이션하여 설정 최적화

## 결론

OpenFeign 클라이언트의 성능은 다양한 요소에 의해 영향을 받습니다. 이 가이드에서 설명한 최적화 기법을 적용하여 마이크로서비스 간 통신 성능을 향상시킬 수 있습니다. 하지만 최적화는 항상 실제 워크로드와 요구사항에 맞게 조정되어야 함을 명심하세요.

서비스의 특성, 트래픽 패턴, 응답 시간 요구사항 등을 고려하여 가장 효과적인 최적화 전략을 선택하고 적용하세요.

## 참고 리소스

- [OpenFeign 성능 최적화 가이드](https://github.com/OpenFeign/feign/tree/master/benchmark)
- [Apache HttpClient 문서](https://hc.apache.org/httpcomponents-client-4.5.x/index.html)
- [OkHttp 문서](https://square.github.io/okhttp/)
- [Spring Cloud OpenFeign 구성 참조](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/)
- [Caffeine 캐시 문서](https://github.com/ben-manes/caffeine)
