---
sidebar_position: 3
---

# OpenFeign 요청 헤더와 인터셉터

REST API 통신에서 요청 헤더는 인증, 콘텐츠 타입 지정, 캐싱 전략 등 다양한 목적으로 사용됩니다. OpenFeign에서는 여러 방법으로 요청 헤더를 설정할 수 있는데, 이 가이드에서는 OpenFeign에서
요청 헤더를 설정하는 방법과 인터셉터를 활용하는 방법을 설명합니다.

## 정적 헤더 설정

### 메서드 레벨 헤더 설정

메서드 단위로 고정된 헤더를 지정할 수 있습니다. 아래는 예시입니다:

```java title="com/example/client/UserClient.java"
package com.example.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "user-service", url = "https://api.example.com")
public interface UserClient {

    // highlight-start
    @GetMapping(value = "/users/{id}", headers = {
        "Accept=application/json",
        "X-API-Version=1.0"
    })
    // highlight-end
    User getUserById(@PathVariable("id") Long id);
}
```

:::info
`@GetMapping` 어노테이션의 `headers` 속성을 사용하여 특정 메서드에 대한 헤더를 정의합니다.
:::

### 인터페이스 레벨 헤더 설정

`@RequestMapping` 어노테이션을 통해 클라이언트 인터페이스 전체에 적용될 헤더를 설정할 수 있습니다:

```java title="com/example/client/UserClient.java"
package com.example.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeiginClient(name = "user-service", url = "https://api.example.com")
// highlight-start
@RequestMapping(headers = {"X-App-Name=MyApplication", "X-App-Version=1.0.0"})
// highlight-end
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);

    @PostMapping("/users")
    User createUser(@RequestBody User user);
}
```

:::info
`@RequestMapping` 어노테이션의 `headers` 속성을 사용하여 모든 메서드에 공통으로 적용되는 헤더를 정의합니다.
:::

## 동적 헤더 설정

### @RequestHeader 사용하기

메서드 파라미터로 동적인 헤더 값을 전달할 수 있습니다:

```java title="com/example/client/UserClient.java"
package com.example.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestHeader;

@FeignClient(name = "user-service", url = "https://api.example.com")
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUserById(
        @PathVariable("id") Long id,
        // highlight-start
        @RequestHeader("Authorization") String token
        // highlight-end
    );
}
```

호출 예시:

```java title="com/example/service/UserService.java"
package com.example.service;

import com.example.client.UserClient;
import com.example.dto.User;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserService {

    private final UserClient userClient;

    public User getUserWithToken(Long id, String token) {
        // highlight-start
        return userClient.getUserById(id, token);
        // highlight-end
    }
}
```

:::info
`@RequestHeader` 어노테이션을 사용하여 메서드 호출 시 동적으로 헤더를 전달할 수 있습니다.
:::

## RequestInterceptor 활용

여러 API 호출에 공통으로 적용해야 하는 헤더가 있다면, `RequestInterceptor`를 구현하여 모든 요청에 자동으로 헤더를 추가할 수 있습니다.

### 글로벌 인터셉터 설정

모든 Feign 클라이언트에 적용되는 인터셉터를 정의할 수 있습니다:

```java title="com/example/config/FeignConfig.java"
package com.example.config;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FeignConfig {

    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            String token = "sample-token";
            // highlight-start
            requestTemplate.header("Authorization", "Bearer " + token);
            // highlight-end
        };
    }
}
```

:::info
`RequestInterceptor`를 사용하여 모든 Feign 클라이언트의 요청에 공통 헤더를 자동으로 추가합니다.
:::

### 특정 클라이언트에만 인터셉터 적용

특정 Feign 클라이언트에만 인터셉터를 적용하려면 다음과 같이 설정할 수 있습니다:

```java title="com/example/config/feign/CustomFeignConfig.java"
package com.example.config;

import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

public class CustomFeignConfig {

    @Bean
    public RequestInterceptor customRequestInterceptor() {
        return requestTemplate -> {
            requestTemplate.header("X-Custom-Header", "CustomValue");
            // 필요한 추가 헤더
        };
    }
}
```

Feign 클라이언트에 설정 적용:

```java title="com/example/feign/UserClient.java"
package com.example.client;

import com.example.config.CustomFeignConfig;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(
    name = "user-service",
    url = "https://api.example.com",
    // highlight-start
    configuration = CustomFeignConfig.class
    // highlight-end
)
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);
}
```

:::caution
`@Configuration`을 사용하면 인터셉터가 모든 Feign 클라이언트에 적용될 수 있습니다. 특정 클라이언트에만 적용하려면 일반 클래스에 `@Bean`을 사용하여 <b>Feign 클라이언트에 설정을 직접
지정</b>해야 합니다.
:::

## 정리

OpenFeign에서 요청 헤더를 설정하는 방법은 다음과 같이 요약할 수 있습니다:

1. **정적 헤더**: 메서드나 인터페이스 어노테이션에 직접 헤더 지정
2. **동적 헤더**: `@RequestHeader` 어노테이션을 사용하여 메서드 파라미터로 헤더 전달
3. **인터셉터**: `RequestInterceptor`를 구현하여 요청마다 자동으로 헤더 추가

상황에 맞는 방식을 선택하여 효율적인 API 통신을 구현하세요.
