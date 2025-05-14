---
sidebar_position: 2
---

# OpenFeign 기본 사용법

OpenFeign은 선언적 방식으로 API 클라이언트를 정의하여 복잡한 로직 없이 간단한 선언만으로 외부 API와 통신이 가능하게 합니다.이 가이드는 OpenFeign을 사용하여 REST API를 호출하는 기본적인 방법을 설명합니다.

## 의존성 추가

프로젝트에 OpenFeign을 사용하기 위해서는 먼저 의존성을 추가해야 합니다.

**Maven**:

```xml title="pom.xml"
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

**Gradle**:

```groovy title="build.gradle"
implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
```

:::info
Spring Cloud 버전에 맞는 OpenFeign 버전을 선택해야 합니다. 호환성 확인은 [Spring Cloud 공식 문서](https://spring.io/projects/spring-cloud)를 참조하세요.
:::

## Spring Boot 애플리케이션에 Feign 활성화

Feign을 활성화하려면 `@EnableFeignClients` 어노테이션을 사용해야 합니다.

```java title="com/example/config/FeignConfig.java"
package com.example.config; // 예시 패키지

import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.Configuration;

@Configuration
// highlight-start
@EnableFeignClients(basePackages = "com.example.feign") // Feign 클라이언트 인터페이스가 있는 패키지
// highlight-end
public class FeignConfig {
    // 추가적인 Feign 설정은 여기서 할 수 있습니다.
}
```

:::warning
`@EnableFeignClients`는 메인 애플리케이션 클래스가 아닌, `FeignConfig`와 같은 별도의 설정 클래스에 선언하는 것을 권장합니다.

이렇게 하면 코드의 모듈성과 가독성이 향상될 뿐만 아니라, 테스트 코드에 불필요한 영향을 줄일 수 있습니다.
:::

## DTO(Data Transfer Object) 모델 정의

API 통신에 사용할 데이터 객체를 정의합니다:

```java title="com/example/feign/dto/User.java"
package com.example.feign.dto; // 예시 패키지

public class User {
    private Long id;
    private String name;
    private String email;

    // getter, setter, constructor 등
}
```

## Feign 클라이언트 인터페이스 정의하기

OpenFeign의 핵심은 인터페이스 정의를 통한 API 호출입니다. 아래와 같이 인터페이스를 정의하여 HTTP API를 호출할 수 있습니다:

```java title="com/example/feign/UserClient.java"
package com.example.feign; // 예시 패키지

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.RequestBody;

@FeignClient(name = "user-service", url = "https://api.example.com")
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUserById(@PathVariable("id") Long id);

    @PostMapping("/users")
    User createUser(@RequestBody User user);

    @PutMapping("/users/{id}")
    User updateUser(@PathVariable("id") Long id, @RequestBody User user);

    @DeleteMapping("/users/{id}")
    void deleteUser(@PathVariable("id") Long id);
}
```

:::info
`@FeignClient` 어노테이션의 `name` 속성은 클라이언트를 식별하는 데 사용되며, `url` 속성은 호출할 API의 기본 URL을 지정합니다.
:::

## 서비스에서 Feign 클라이언트 사용하기

정의한 Feign 클라이언트는 일반 Spring Bean처럼 주입받아 사용할 수 있습니다:

```java title="com/example/service/UserService.java"
package com.example.service; // 예시 패키지

import com.example.feign.UserClient;
import com.example.feign.dto.User;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserService {

    // highlight-start
    private final UserClient userClient;
    // highlight-end

    public User getUser(Long id) {
        return userClient.getUserById(id);
    }

    public User createNewUser(User user) {
        return userClient.createUser(user);
    }

    // 기타 메서드...
}
```

## 기본 요청 파라미터 전달

### Path Variable

URL 경로에 변수를 포함할 때는 `@PathVariable` 어노테이션을 사용합니다:

```java
@GetMapping("/users/{id}")
User getUserById(@PathVariable("id") Long id);
```

### Query Parameter

쿼리 파라미터를 전달할 때는 `@RequestParam` 어노테이션을 사용합니다:

```java
@GetMapping("/users")
List<User> searchUsers(@RequestParam("name") String name);
```

### Request Body

요청 본문을 전달할 때는 `@RequestBody` 어노테이션을 사용합니다:

```java
@PostMapping("/users")
User createUser(@RequestBody User user);
```
