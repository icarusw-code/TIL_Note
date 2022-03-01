## Spring Boot 서비스 구조

<img src="https://user-images.githubusercontent.com/77170611/155713487-184f8c6a-9ade-4e46-a376-3e6714d89824.png" alt="image"  />

Service: 비지니스 로직의 전반적인 내용을 처리  

Repository : DB와 직접 통신 

둘다 Interface로 되어있기 때문에 실제 구현체인 Impl을 이용한다.



**Entity(Domain)**

데이터베이스에 쓰일 컬럼과 여러 엔티티 간의 연관관계를 정의

데이터베이스의 테이블을 하나의 엔티티로 생각해도 무방

실제 데이터베이스의 테이블과 1:1 로 매핑됨

이 클래스의 필드는 각 테이블 내부의 컬럼(Column)을 의미



**DAO**

- Repository

​		Entity에 의해 생성된 DB에접근하는 메소드를 사용하기 위한 인터페이스

​		Service와 DB를 연결하는 고리의 역할을 수행

​		데이터베이스에 적용하고자 하는 CRUD를 정의하는 영역

- DAO(Data Access Object) - Repository를 활용

  데이터베이스에 접근하는 객체를 의미 (Persistance Layer)

  Service가 DB에 연결할 수 있게 해주는 역할

  DB를 사용하여 데이터를 조회하거나 조작하는 기능을 전담



**DTO(Data Transfer Object**

DTO는 VO(Value Object)로 불리기도 하며, 계층간 데이터 교환을 위한 객체를 의미

(클라이언트 - Controller - Service) 각 계층이 구분되어있는곳에 데이터를 옮겨주기위해 사용되는 객체

===========================================================================

### DTO 예시

![image](https://user-images.githubusercontent.com/77170611/155718185-43e567e8-106a-476a-a217-885675e11b1c.png)



### Entity 예시

![image](https://user-images.githubusercontent.com/77170611/155718396-874d96ae-afb2-42c2-b422-31ae65e30f2b.png)



### Repository 예시

![image](https://user-images.githubusercontent.com/77170611/155719580-de23172b-c990-4a7e-b5e2-a9ede568252a.png)

JpaRepository<사용할 Entity , PK의 데이터 타입>



### DAO 예시

![image](https://user-images.githubusercontent.com/77170611/155719804-6a72b74c-e5fb-477b-ba50-ee5ba2b2bc5e.png)

메소드의 선언만 함



### ProductDAOImpl(구현체) 예시

![image](https://user-images.githubusercontent.com/77170611/155719960-605e55e1-c259-41c4-aa68-f271a78ea8db.png)

의존성 주입을 사용해서 Reopsitory를 끌어와서 주입받아서 사용한다.



### Service 예시

![image](https://user-images.githubusercontent.com/77170611/155727285-238e3920-d091-497d-83e8-3ac28f506a6c.png)

인터페이스로 만들고 Impl로 구체화 시킨다.



### ServiceImpl 예시

![image](https://user-images.githubusercontent.com/77170611/155727555-c48bc829-21d4-4ed3-b4fe-0020f2105d6b.png)

