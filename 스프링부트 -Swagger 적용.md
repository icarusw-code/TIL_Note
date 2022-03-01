## REST 개념

Representational State Transfer 의 약자로 자원을 이름으로 구분하여 해당 자원(Resource)의 상태를 주고 받는 모든것을 의미한다.

1. HTTP URI(Uniform Resource Identifier)를 통해 자원(Resource)를 명시하고

2. HTTP Method(POST, GET, PUT, DELETE)를 통해

3. 해당 자원(URI)에 대한 CRUD Operation을 적용하는 것을 의미한다.

   - CRUD Operation

     Create, Read, Update, Delete를 묶어서 일컫는 말이다.

     Create - 데이터 생성(POST)

     Read - 데이터 조회(GET)

     Update - 데이터 수정(PUT)

     Delete - 데이터 삭제(DELETE)

### REST의 구성 요소

1. 자원(Resource) : HTTP URI
2. 자원에 대한 행위(Verb) : HTTP Method
3. 자원에 대한 행위의 내용(Representaions) : HTTP Message Pay Load



### REST API

REST의 원리를 따르는 API를 의미한다.



### RESTful 

REST의 원리를 따르는 시스템, 그 중에서 **REST API**의 설계 규칙을 올바르게 지킨 시스템을 의미한다.

모든 CRUD 기능을 POST로 처리 하는 API 혹은 URI 규칙을 올바르게 지키지 않은 API, REST API의 설계 규칙을 올바르게 지키지 못한 시스템은 REST API를 사용하였지만, RESTful하지 못한 시스템이다.



## Swagger

### Swagger 개념

서버로 요청되는 API리스트를 HTML 화면으로 문서화하여 테스트 할 수 있는 라이브러리

@RestController를 읽어 API를 분석하여 HTML문서를 작성한다.



### 필요성

REST API의 스펙을 문서화 하는 것은 중요하다

API를 변경할 때마다 Reference문서를 계속 바꿔야하는 불편함을 해결



### 설정방법

https://mvnrepository.com/artifact/io.springfox/springfox-swagger-ui 에서 버전을 확인하자. 

**Swagger 2.9.2버전이 가장 많이 사용된다.**

**주의: 스프링부트 2.6.2와 호환되는 Swagger가 아직 없으므로 버전을 낮추어 사용하자. **

**(Spring Boot2.4.2)**



#### 1.**[build.gradle] 에 임포트 **

```gradle
implementation group: 'io.springfox', name: 'springfox-swagger-ui', version: '2.9.2'
implementation group: 'io.springfox', name: 'springfox-swagger2', version: '2.9.2'
```



#### 2.**[basePackage] - config - SwaggerConfig**

![image](https://user-images.githubusercontent.com/77170611/155145446-a0885088-b8ea-4572-a6bc-3ccda297bb73.png)

**[SwaggerConfig]**

```java
package jpabook.jpashop.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

@Configuration
@EnableSwagger2
public class SwaggerConfig {

    @Bean
    public Docket api(){
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("jpabook.jpashop"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("icarusw의 API Swagger 입니다.")
                .description("설명 부분")
                .version("1.0.0")
                .build();
    }
}

```

**[http://localhost:8080/swagger-ui.html 접속 화면]**

![image](https://user-images.githubusercontent.com/77170611/155146361-889ac631-45b9-4db2-9690-cce9eafca461.png)

**주요 옵션**

- apis : 대상 패키지를 설정 한다. 

  basePackage에는 어디를 범위로 스캔을 할 것인지 작성하면 된다.

  .apis(RequestHandlerSelectors.any()) 를 사용하면 RestController 전부를 스캔한다.

  

- paths:  어떤 식으로 시작하는 api를 보여줄 것인가

  any는 전체를 볼 수 있게 한다.

  만약, member/라고 설정하면 member로 시작하는 api만 볼 수 있게 설정 가능하다.



#### 2-1. 기본 화면 확인

<img src="https://user-images.githubusercontent.com/77170611/155148284-0d0a3b9d-0f9b-4f47-b9bc-c0a1e64cddf2.png" alt="image"  />

기본적으로 모든 @RestController를 검색해서 보여준다.



#### 3. 문서상 API 설명 작성

```java
@Api(tags = {"API 이름"})
@Controller
@RequiredArgsConstructor
public class ItemController {
    @ApiOperation(value = "resource 제목", notes = "resource 설명")
    @GetMapping("/items")
    public String list(Model model) {
        List<Item> items = itemService.findItems();
        model.addAttribute("items", items);
        return "items/itemList";
    }
}
```

- @Api(tags={"API 이름"})

- @ApiOperation(value="resource 제목", notes="resource 설명")

- @ApiResponses(

  ​		{ @ApiResponse(code = 200, message = "API 정상 작동"), 

  ​		  @ApiResponse(code = 500, message = "서버 에러") }

  )![제목 없음](https://user-images.githubusercontent.com/77170611/155154553-0bee45c8-d901-4324-bcee-aba6515ca890.png)



#### 3-1. 내부 parameter 설명도 작성 가능

```

```

**[변경 전]**

![image](https://user-images.githubusercontent.com/77170611/155155434-b1ec1d95-6f48-4ff3-a46a-fa333ae18f82.png)

**[변경 후]**

https://nahwasa.com/entry/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-Swagger-UI-300-%EC%A0%81%EC%9A%A9-%EB%B0%A9%EB%B2%95-%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-22-%EC%9D%B4%EC%83%81-Spring-Boot-Swagger-UI
