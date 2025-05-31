---
slug: spring-security-filter-chain-deep-dive
title: Spring Security 필터 체인 등록 문제 해결기
authors: polar
tags: [spring-security, filter-chain, trouble-shooting, spring-boot, java]
---

# Spring Security 필터 체인 등록 문제 해결하기: 필터 순서의 중요성

안녕하세요, 원큐 오더 PL 남승현입니다.

오늘은 결제를 요청하는데 사용되는 토큰을 검증하는 `PaymentTokenAuthenticationFilter`와 `JwtAuthenticationFilter`를 구성하다 마주친 문제와 이 오류의 원인을 파악하면서 Spring Security의 내부 동작에 대해 깊이 이해하게 된 경험을 공유하려고 해요.

## 문제 상황: 왜 갑자기 필터 체인이 동작하지 않을까?

2025년 5월 16일 로그인 API를 개발하며 JWT 인증 방식을 추가하기 위해 커스텀 필터를 설정했는데, 다음과 같은 오류가 발생했어요:

```
[org.springframework.security.web.SecurityFilterChain]: Factory method 'securityFilterChain' threw exception with message: The Filter class com.fisa.pg.config.security.filter.PaymentTokenAuthenticationFilter does not have a registered order and cannot be added without a specified order
```

## 문제의 원인: 필터 순서는 생각보다 중요해요

먼저 오류가 발생했던 코드는 다음과 같아요:

```java title="SecurityConfig.java"
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final PaymentAuthenticationFilter paymentAuthenticationFilter;

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                // ...중략...
                // highlight-start
                .addFilterBefore(jwtAuthenticationFilter, PaymentTokenAuthenticationFilter.class) // PaymentTokenAuthenticationFilter 필터 전에 JWT 필터 추가
                .addFilterBefore(paymentTokenAuthenticationFilter, UsernamePasswordAuthenticationFilter.class) // UsernamePasswordAuthenticationFilter 필터 전에 PaymentTokenAuthenticationFilter 필터 추가
                // highlight-end
                .build();

    }

    // ...중략...

}
```

그리고 해결한 코드는 다음과 같아요:

```java title="SecurityConfig.java"
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final PaymentAuthenticationFilter paymentAuthenticationFilter;

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                // ...중략...
                // highlight-start
                .addFilterBefore(paymentTokenAuthenticationFilter, UsernamePasswordAuthenticationFilter.class) // UsernamePasswordAuthenticationFilter 필터 전에 PaymentTokenAuthenticationFilter 필터 추가
                .addFilterBefore(jwtAuthenticationFilter, PaymentTokenAuthenticationFilter.class) // PaymentTokenAuthenticationFilter 필터 전에 JWT 필터 추가
                // highlight-end
                .build();

    }

    // ...중략...

}
```

왜 이전 코드에서 오류가 발생했을까요?

간단히 말하면, Spring Security는 필터 체인을 구성할 때 참조되는 필터가 이미 체인에 등록되어 있어야 해요.
이전 코드에서는 `PaymentTokenAuthenticationFilter`가 아직 체인에 추가되지 않은 상태에서 참조했기 때문에 오류가 발생했어요.

## Spring Security 내부 들여다보기: 필터 체인은 어떻게 구성될까?

Spring Security가 어떻게 필터 체인을 구성하는지 코드 레벨에서 깊게 살펴볼게요. 이 과정을 이해하면 왜 필터 추가 순서가 중요한지 명확해질 거예요.

### 1. 필터 순서 관리 방식

Spring Security는 `FilterOrderRegistration`이라는 클래스를 통해 필터의 순서를 관리해요.

이 클래스는 각 필터의 순서를 `filterToOrder`라는 Map에 저장하는데, 표준 필터만 기본으로 등록되어 있어요.

```java title="FilterOrderRegistration.java"
final class FilterOrderRegistration {

    private static final int INITIAL_ORDER = 100;

    private static final int ORDER_STEP = 100;

    private final Map<String, Integer> filterToOrder = new HashMap<>();

    FilterOrderRegistration() {
        put(DisableEncodeUrlFilter.class, order.next());
        put(ForceEagerSessionCreationFilter.class, order.next());
        put(ChannelProcessingFilter.class, order.next());
        // ...중략...
        put(UsernamePasswordAuthenticationFilter.class, order.next());
        // ...중략...
        put(FilterSecurityInterceptor.class, order.next());
        put(AuthorizationFilter.class, order.next());
        put(SwitchUserFilter.class, order.next());
    }

    void put(Class<? extends Filter> filter, int position) {
        this.filterToOrder.putIfAbsent(filter.getName(), position);
    }

    // ...중략...

}
```

그리고 커스텀 필터는 이 `filterToOrder`라는 Map에 등록되어 있지 않죠.

### 2. 커스텀 필터 등록 과정

이제 표준 필터가 어떻게 등록되어 있는지 알았으니, 그 다음으로 우리가 직접 만든 커스텀 필터를 등록하는 과정을 살펴볼게요.

커스텀 필터를 등록할 때 사용하는 `HttpSecurity`의 `addFilterBefore` 메서드는 다음과 같이 동작해요:

```java title="HttpSecurity.java"
public final class HttpSecurity extends AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity> implements SecurityBuilder<DefaultSecurityFilterChain>, HttpSecurityBuilder<HttpSecurity> {

    // ...중략...

    @Override
    public HttpSecurity addFilterBefore(Filter filter, Class<? extends Filter> beforeFilter) {
        return addFilterAtOffsetOf(filter, -1, beforeFilter);
    }

    private HttpSecurity addFilterAtOffsetOf(Filter filter, int offset, Class<? extends Filter> registeredFilter) {
        Integer registeredFilterOrder = this.filterOrders.getOrder(registeredFilter);
        if (registeredFilterOrder == null) {
            throw new IllegalArgumentException("The Filter class " + registeredFilter.getName() + " does not have a registered order");
        }
        int order = registeredFilterOrder + offset;
        this.filters.add(new OrderedFilter(filter, order));
        this.filterOrders.put(filter.getClass(), order);
        return this;
    }

    // ...중략...
}
```

실제 코드를 보면 `addFilterBefore` 메서드는 내부적으로 `addFilterAtOffsetOf` 메서드를 호출하고 있어요. 이 메서드의 동작 방식을 단계별로 살펴볼게요:

1. `addFilterBefore`는 새 필터를 특정 필터 '앞에' 추가하려 할 때 `addFilterAtOffsetOf`를 offset `-1`로 호출해요. (앞에 위치하려면 순서 값이 더 작아야 하니까요)

2. `addFilterAtOffsetOf` 메서드는 다음 작업을 수행해요:

   - `filterOrders.getOrder(registeredFilter)`를 호출해서 참조하는 필터의 순서를 조회해요.
   - 만약 참조하는 필터가 등록되지 않았다면(순서가 null이면) 예외를 던져요.
   - 참조 필터의 순서에 offset을 더해 새 필터의 순서를 계산해요.
   - 새 필터를 `OrderedFilter`로 감싸서 `filters` 목록에 추가해요.
   - 새 필터의 클래스와 계산된 순서를 `filterOrders`에 등록해요. 이렇게 하면 나중에 이 필터를 참조할 수 있게 돼요!

### 3. 커스텀 필터 적용 과정

앞에서 필터가 등록되는 과정을 살펴봤으니, 이제 이렇게 등록된 커스텀 필터가 Spring Security의 실제 필터 체인에 어떻게 추가되는지 자세히 알아볼게요.

Spring Security는 `SecurityBuilder` 패턴을 사용해서 필터 체인을 구성해요. 이 과정에서 핵심 역할을 하는 클래스가 바로 `AbstractConfiguredSecurityBuilder`예요.
이 클래스의 `doBuild()` 메소드는 필터 체인 구축 과정을 단계별로 관리해요:

```java title="AbstractConfiguredSecurityBuilder.java"
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>> extends AbstractSecurityBuilder<O> {

    // ...중략...

    /**
     * Executes the build using the {@link SecurityConfigurer}'s that have been applied
     * using the following steps:
     *
     * <ul>
     * <li>Invokes {@link #beforeInit()} for any subclass to hook into</li>
     * <li>Invokes {@link SecurityConfigurer#init(SecurityBuilder)} for any
     * {@link SecurityConfigurer} that was applied to this builder.</li>
     * <li>Invokes {@link #beforeConfigure()} for any subclass to hook into</li>
     * <li>Invokes {@link #performBuild()} which actually builds the Object</li>
     * </ul>
     */
    @Override
    protected final O doBuild() throws Exception {
        synchronized (this.configurers) {
            this.buildState = BuildState.INITIALIZING;
            beforeInit();
            init();
            this.buildState = BuildState.CONFIGURING;
            beforeConfigure();
            configure();
            this.buildState = BuildState.BUILDING;
            O result = performBuild();
            this.buildState = BuildState.BUILT;
            return result;
        }
    }

    // ...중략...

}
```

이 빌드 과정은 네 가지 주요 상태로 나뉘어요:

#### 1) INITIALIZING (초기화) 단계

이 단계에서 `beforeInit()` 메소드가 호출되는데, 이 메소드는 `SecurityConfigurer`를 사용하지 않고도 빌드 생명주기에 연결할 수 있게 해줘요:

```java title="AbstractConfiguredSecurityBuilder.java"
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>> extends AbstractSecurityBuilder<O> {

    // ...중략...

    /**
     * Invoked prior to invoking each {@link SecurityConfigurer#init(SecurityBuilder)}
     * method. Subclasses may override this method to hook into the lifecycle without
     * using a {@link SecurityConfigurer}.
     */
    protected void beforeInit() throws Exception {

    }

    // ...중략...

}
```

그 다음 `init()` 메소드가 호출되어 모든 등록된 `SecurityConfigurer`의 `init()` 메소드가 실행돼요:

```java title="AbstractConfiguredSecurityBuilder.java"
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>> extends AbstractSecurityBuilder<O> {

    // ...중략...

    private final LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>> configurers = new LinkedHashMap<>();

    // ...중략...

    private void init() throws Exception {
        Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
        for (SecurityConfigurer<O, B> configurer : configurers) {
            configurer.init((B) this);
        }
        for (SecurityConfigurer<O, B> configurer : this.configurersAddedInInitializing) {
            configurer.init((B) this);
        }
    }

    // ...중략...

    private Collection<SecurityConfigurer<O, B>> getConfigurers() {
        List<SecurityConfigurer<O, B>> result = new ArrayList<>();
        for (List<SecurityConfigurer<O, B>> configs : this.configurers.values()) {
            result.addAll(configs);
        }
        return result;
    }
}
```

이 시점에서 각 `SecurityConfigurer`는 기본 설정을 준비하고, 필요한 객체들을 생성해요.
예를 들어, `FormLoginConfigurer`는 로그인 페이지 URL, 인증 성공/실패 핸들러 등을 초기화해요.

#### 2) CONFIGURING (설정) 단계

초기화가 완료된 후, `beforeConfigure()` 메소드가 호출돼요. 이 메소드도 `SecurityConfigurer`를 사용하지 않고 빌드 생명주기에 연결할 수 있게 해줘요:

```java title="AbstractConfiguredSecurityBuilder.java"
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>> extends AbstractSecurityBuilder<O> {

    // ...중략...

    /**
     * Invoked prior to invoking each
     * {@link SecurityConfigurer#configure(SecurityBuilder)} method. Subclasses may
     * override this method to hook into the lifecycle without using a
     * {@link SecurityConfigurer}.
     */
    protected void beforeConfigure() throws Exception {

    }

    // ...중략...

}
```

그 다음 `configure()` 메소드를 통해 모든 `SecurityConfigurer`의 `configure()` 메소드가 호출돼요:

```java
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>> extends AbstractSecurityBuilder<O> {

    // ...중략...

    private final LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>> configurers = new LinkedHashMap<>();

    // ...중략...

    private void configure() throws Exception {
        Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
        for (SecurityConfigurer<O, B> configurer : configurers) {
            configurer.configure((B) this);
        }
    }

    private Collection<SecurityConfigurer<O, B>> getConfigurers() {
        List<SecurityConfigurer<O, B>> result = new ArrayList<>();
        for (List<SecurityConfigurer<O, B>> configs : this.configurers.values()) {
            result.addAll(configs);
        }
        return result;
    }
}
```

이 단계에서 실제 필터 설정이 적용되고, 여러분이 `HttpSecurity`에 추가했던 커스텀 필터들이 구성돼요.
예를 들어, `WebSecurityConfigurerAdapter`의 `configure(HttpSecurity http)` 메소드나 `SecurityConfig` 클래스의 `securityFilterChain` 메소드에서 정의한 설정들이 이 시점에 적용돼요.

#### 3) BUILDING (빌드) 단계

모든 설정이 완료된 후, `performBuild()` 메소드가 호출돼요. 이 메소드는 실제 빌드 과정을 수행하는 추상 메소드로, 구현 클래스에 따라 다르게 동작해요.

```java title="AbstractConfiguredSecurityBuilder.java"
public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>> extends AbstractSecurityBuilder<O> {

    // ...중략...

    protected abstract O performBuild() throws Exception;

    // ...중략...

}
```

`HttpSecurity`의 `performBuild()` 메소드는 다음과 같이 최종 필터 체인을 생성해요:

```java title="HttpSecurity.java"
public final class HttpSecurity extends AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity> implements SecurityBuilder<DefaultSecurityFilterChain>, HttpSecurityBuilder<HttpSecurity> {

    // ...중략...

    @SuppressWarnings("unchecked")
    @Override
    protected DefaultSecurityFilterChain performBuild() {
    	ExpressionUrlAuthorizationConfigurer<?> expressionConfigurer = getConfigurer(ExpressionUrlAuthorizationConfigurer.class);
    	AuthorizeHttpRequestsConfigurer<?> httpConfigurer = getConfigurer(AuthorizeHttpRequestsConfigurer.class);
    	boolean oneConfigurerPresent = expressionConfigurer == null ^ httpConfigurer == null;
    	Assert.state((expressionConfigurer == null && httpConfigurer == null) || oneConfigurerPresent, "authorizeHttpRequests cannot be used in conjunction with authorizeRequests. Please select just one.");
    	this.filters.sort(OrderComparator.INSTANCE);
    	List<Filter> sortedFilters = new ArrayList<>(this.filters.size());
    	for (Filter filter : this.filters) {
    	    sortedFilters.add(((OrderedFilter) filter).filter);
    	}
    	return new DefaultSecurityFilterChain(this.requestMatcher, sortedFilters);
    }

}
```

이 구현에서 가장 중요한 부분은 필터들이 `OrderComparator`를 사용해서 정렬된다는 점이에요. 즉, `Ordered` 인터페이스나 `@Order` 어노테이션을 통해 지정된 순서에 따라 필터가 정렬되어 최종
필터 체인에 추가돼요.

#### 4) BUILT (완료) 단계

빌드가 완료된 상태로, 이제 `SecurityFilterChain`이 완전히 구성되었고 사용할 준비가 됐어요.

### 필터 체인 구성의 핵심 포인트

이러한 빌드 과정을 이해하면 몇 가지 중요한 점을 알 수 있어요:

1. **순서의 중요성**: 필터는 등록된 순서대로 실행돼요. 필터 A가 필터 B보다 먼저 등록되면, 요청 처리 시 A가 B보다 먼저 실행돼요.

2. **빌드 시점의 검증**: 필터 체인이 구성될 때 필터 순서가 최종적으로 검증돼요. 이 시점에서 잘못된 필터 구성이 발견되면 예외가 발생해요.

3. **한 번 빌드, 여러 번 사용**: `SecurityFilterChain`은 애플리케이션 시작 시 한 번 빌드되고, 이후 모든 요청에 재사용돼요. 따라서 빌드 과정의 효율성과 정확성이 중요해요.

4. **계층적 구성**: Spring Security는 여러 `SecurityFilterChain`을 URL 패턴별로 다르게 구성할 수 있어요. 이렇게 하면 다양한 보안 요구사항에 맞게 유연하게 필터를 적용할 수 있어요.

바로 이런 빌드 프로세스 때문에, 필터를 추가하는 순서가 중요해져요.
`addFilterBefore`나 `addFilterAfter`를 호출할 때, 참조하는 필터가 이미 등록되어 있어야 하는 이유가 이것 때문인 거죠.

## 우리가 만난 오류 다시 살펴보기: 근본 원인 분석

이제 Spring Security의 내부 코드를 분석했으니, 처음 만났던 오류를 다시 살펴볼게요:

```
The Filter class com.fisa.pg.config.security.filter.PaymentTokenAuthenticationFilter does not have a registered order and cannot be added without a specified order
```

이 오류 메시지는 `PaymentTokenAuthenticationFilter` 클래스가 등록된 순서(registered order)를 가지고 있지 않아서 발생했다고 말하고 있어요.

우리가 `.addFilterBefore(jwtAuthenticationFilter, PaymentTokenAuthenticationFilter.class)`를 호출했을 때,
`PaymentTokenAuthenticationFilter`가 아직 등록되지 않았기 때문에 `filterOrders`에서 순서를 찾을 수 없었고, 그 결과 예외가 발생한 거예요.

해결 방법은 간단했어요. `PaymentTokenAuthenticationFilter`를 `UsernamePasswordAuthenticationFilter` 필터 앞에 추가하고,
`JwtAuthenticationFilter`를 `PaymentTokenAuthenticationFilter` 앞에 추가하는 것이었죠.

## 실용적인 해결책: 필터 체인 구성 가이드라인

이런 문제를 방지하기 위한 몇 가지 가이드라인을 공유할게요:

1. **의존성 순서 고려하기**: 필터 간에 의존성이 있다면, 의존하는 필터가 먼저 추가되도록 순서를 조정하세요.
2. **명시적 순서 설정 사용하기**: 커스텀 필터에 `@Order` 어노테이션을 사용하거나, `Ordered` 인터페이스를 구현하는 것도 좋은 방법이에요.

## 마무리: 깊은 이해가 보안을 강화해요

Spring Security의 필터 체인을 설정할 때는 단순히 코드를 작성하는 것 이상으로, 내부 메커니즘을 이해하는 것이 중요해요.
이번 경험을 통해 Spring Security의 내부 동작 방식을 더 깊이 이해하게 되었고, 이를 바탕으로 더 안전한 결제 시스템을 구축할 수 있게 되었어요.

여러분도 이 글을 통해 Spring Security의 필터 체인에 대한 이해를 높이고, Security 설정을 더 효과적으로 관리할 수 있기를 바라요.

## 참고 자료

- [FilterOrderRegistration.java](https://github.com/spring-projects/spring-security/blob/main/config/src/main/java/org/springframework/security/config/annotation/web/builders/FilterOrderRegistration.java)
- [HttpSecurity.java](https://github.com/spring-projects/spring-security/blob/main/config/src/main/java/org/springframework/security/config/annotation/web/builders/HttpSecurity.java)
- [AbstractConfiguredSecurityBuilder.java](https://github.com/spring-projects/spring-security/blob/main/config/src/main/java/org/springframework/security/config/annotation/AbstractConfiguredSecurityBuilder.java)
