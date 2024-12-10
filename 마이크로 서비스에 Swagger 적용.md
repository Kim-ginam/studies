각 마이크로서비스에서 Swagger UI를 구성하려면 `springdoc-openapi` 라이브러리를 사용하는 것이 가장 쉽습니다. 아래는 Spring Boot 애플리케이션에서 Swagger UI를 구성하는 방법입니다.

---

### **1. Dependency 추가**
`pom.xml` 파일에 `springdoc-openapi`와 관련된 의존성을 추가합니다:

```xml
<dependencies>
    <!-- OpenAPI 3와 Swagger UI 지원 -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-ui</artifactId>
        <version>1.7.0</version> <!-- 최신 버전 확인 후 업데이트 -->
    </dependency>
</dependencies>
```

Gradle을 사용하는 경우:
```gradle
implementation 'org.springdoc:springdoc-openapi-ui:1.7.0'
```

---

### **2. 기본 설정**
`application.properties` 또는 `application.yml` 파일에 Swagger UI 관련 설정을 추가합니다:

#### **application.properties**
```properties
springdoc.api-docs.path=/v3/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
```

#### **application.yml**
```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
```

---

### **3. Controller 작성 예제**
Swagger UI가 동작하는지 확인하기 위해 간단한 REST Controller를 작성합니다:

```java
package com.example.demo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @GetMapping("/hello")
    public String sayHello(@RequestParam(name = "name", defaultValue = "World") String name) {
        return String.format("Hello, %s!", name);
    }
}
```

---

### **4. Swagger UI 확인**
- 애플리케이션을 실행한 후 다음 경로로 Swagger UI에 접속합니다:
  - 기본 경로: `http://localhost:8080/swagger-ui.html`
  - API 문서 경로: `http://localhost:8080/v3/api-docs`

Swagger UI는 자동으로 모든 엔드포인트를 수집하고 사용자 인터페이스에서 제공합니다.

---

### **5. 추가 설정 (선택)**
필요에 따라 OpenAPI 문서를 세부적으로 구성하려면, `@OpenAPIDefinition` 어노테이션을 사용합니다.

#### 예제: OpenAPI 문서 정의
```java
import io.swagger.v3.oas.annotations.OpenAPIDefinition;
import io.swagger.v3.oas.annotations.info.Info;
import io.swagger.v3.oas.annotations.info.Contact;

@OpenAPIDefinition(
    info = @Info(
        title = "Demo API",
        version = "1.0",
        description = "This is the API documentation for the Demo application.",
        contact = @Contact(
            name = "Your Name",
            email = "your.email@example.com"
        )
    )
)
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

---

### **6. 각 서비스에 적용**
- 위 코드를 각 마이크로서비스에 복사하고 적절히 수정합니다.
- Istio Gateway를 통해 각각의 Swagger UI를 연결하거나, Swagger Aggregator를 구성해 통합 Swagger 문서를 제공할 수 있습니다.