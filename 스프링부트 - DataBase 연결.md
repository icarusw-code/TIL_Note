# 스프링부트 - DataBase 연결

## 1. build.gradle 설정

```
plugins {
	id 'org.springframework.boot' version '2.6.3'
	id 'io.spring.dependency-management' version '1.0.11.RELEASE'
	id 'java'
}

group = 'com.icarusw'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	compileOnly 'org.projectlombok:lombok'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	runtimeOnly 'org.mariadb.jdbc:mariadb-java-client' // MariaDB
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
	useJUnitPlatform()
}
```

- MariaDB를 사용하기 위해 mariadb client 의존성을 다운 받는다.

  ```
  	runtimeOnly 'org.mariadb.jdbc:mariadb-java-client' // MariaDB
  ```

- JPA를 사용하기 위해 JPA 의존성도 다운 받는다.

  ```
  	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  ```

**Spring Boot 초기 설정에서 Dependencies에서 넣어줘도 된다.**

![image](https://user-images.githubusercontent.com/77170611/154085650-6f2ccb66-c698-4765-9962-fa8f8c102ac8.png)



## 2. application.yml 설정

```
spring:
  datasource:
    driver-class-name: org.mariadb.jdbc.Driver
    url: jdbc:mariadb://localhost:3306/test?serverTimezone=Asia/Seoul
    username: project
    password: project1234

  jpa:
    open-in-view: true
    hibernate:
      ddl-auto: create
      use-new-id-generate-mappings: false
    show-sql: true
    properties:
      hibernate.format_sql: true
```

```
url: jdbc:mariadb://localhost:3306/test?serverTimezone=Asia/Seoul
```

- **3306 : DataBase의 Port 번호**

  ​          **[MySQL Workbench로 확인 하는 방법]**

![image](https://user-images.githubusercontent.com/77170611/154087513-122f2a8c-b594-4a4b-8b88-70d0477fd81a.png)

​					MariaDB는 커넥션(세션) 이름이므로 설정값과 아무런 연관이 없다. 

​					프로젝트 이름이라고 생각하자.

- **test: Database의 이름**

![image-20220216000520663](C:\Users\CHOI\AppData\Roaming\Typora\typora-user-images\image-20220216000520663.png)

​		Schema생성으로 Database를 생성하고 이름을 맞춰준다.

- **database 계정 입력**

```
username: project
password: project1234
```

root 계정을 사용하면 중요한 정보이기 때문에 따로 정보를 만들어서 해주는것이 좋다.

![image](https://user-images.githubusercontent.com/77170611/154092378-72576120-7d59-4db7-9eda-0a7663a55336.png)

1. **Users and Privileges**에서 계정 추가를 위해 **Add Account**를 해준다.
2. 사용할 이름과 비밀번호를 입력한다.
3. 특정 스키마를 사용할 수 있게 권한 설정을 위해 **Selected schema**를 선택후 **Add Enty...**를 눌러준다.

![image](https://user-images.githubusercontent.com/77170611/154093052-fe8be5c1-f0d2-425c-af88-d3d16aa21af1.png)

4. 스키마를 선택한다.

![image](https://user-images.githubusercontent.com/77170611/154093219-a11a4865-4aa3-4672-be94-3d022a139b26.png)

5. **Select ALL**을 이용해 권한을 모두 주거나 조절해서 줄 수 있다.

​	**참고 [Heidi를 사용해서 계정 만들기]**

![image-20220216003003468](C:\Users\CHOI\AppData\Roaming\Typora\typora-user-images\image-20220216003003468.png)











