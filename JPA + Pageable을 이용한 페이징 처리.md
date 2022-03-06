## JPA + Pageable을 이용한 페이징 처리

### 개요

Spring Boot JPA 를 이용하여 API 개발 시 Pagination과 Sorting 처리를 할 수 있는 Pageable을 이용

Pageable의 이점

1. 요건에 맞는 Pagination을 구현할 수 있다.
2. 정렬이 필요한 데이터를 쉽게 Sorting 할 수 있다.



### 구현

#### **[Ariticle] 엔티티**

```java
@Entity
@Getter
@NoArgsConstructor
@Table(name = "ARTICLE")
public class Article extends BaseTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 100)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String content;

    @Column(name = "read_count")
    private Long readCount;

    @Column(name = "likes_count")
    private Long likesCount;

    @PrePersist
    public void initializer() {
        readCount = 0L;
        likesCount = 0L;
    }
```

#### **[ArticleRepository]**

```java
public interface ArticleRepository extends JpaRepository<Article, Long> {
    Page<Article> findAll(Pageable pageable);
}
```

![image](https://user-images.githubusercontent.com/77170611/156720745-eeaedf64-9693-46fe-9517-124475687bdb.png)

ArticleRepository 는 JpaRepository를 상속 받는데, JpaRepository는 PagingAndSortingRepository를 

상속받는다. 

![image](https://user-images.githubusercontent.com/77170611/156720973-c5e7f0a4-0fb6-43d1-b067-5cebc07143c8.png)

PagingAndSortingRepository 에는 pageable을 매개변수로 사용하는

```java
Page<T> findAll(Pageable pageable); 
```

![image](https://user-images.githubusercontent.com/77170611/156721460-4849950a-2686-4f80-ab0b-b47f8c8ccd42.png)
![image](https://user-images.githubusercontent.com/77170611/156721499-f5fc0e8b-48cc-4820-a866-26c8bd230e18.png)

Page<T> findAll(Pageable pageable); 

가 이미 있으므로 Pagination 과 Sorting을 처리할 준비가 된 상태이다.



#### [AritcleApiController]

```java
@GetMapping("/article")
public ResponseEntity getArticle(@RequestParam("page") int page, ModelMap model){
    Page<ArticleFindDto> articles = articleService.findAllArticle(page);
    Pageable pageable = articles.getPageable();
    model.addAttribute("page", PageUtils.getPages(pageable, articles.getTotalPages()));
    model.addAttribute("articles", articles);
    return ResponseEntity.status(HttpStatus.OK).body(model);
}
```

필요한 정보만 사용하기 위해 ArticleFindDto 를 사용한다. 

**[ArticleFindDto]**

ResponseEntity body에 넣어줄 요소들을 만들어 준다. 받아온 정보는 생성자를 만들어서 넣어준다.

```java
@Getter
@NoArgsConstructor
public class ArticleFindDto {

    private Long id;
    private String title;
    private String content;
    private Long readCount;
    private Long likesCount;
    private Long userId;
    private LocalDateTime modifiedDateTime;
    
    public void convertEntityToDto(Article article) {
        this.id = article.getId();
        this.title = article.getTitle();
        this.content = article.getContent();
        this.readCount = article.getReadCount();
        this.likesCount = article.getLikesCount();
        this.userId = article.getUser().getId();
        this.modifiedDateTime = article.getModifiedDate();
    }
}
```



Response body

```json
{
  "page": {
    "EndPage": 2,
    "StartPage": 0
  },
  "articles": {
    "content": [
      {
        "id": 1,
        "title": "string",
        "content": "string",
        "readCount": 0,
        "likesCount": 0,
        "userId": 1,
        "modifiedDateTime": "2022-03-04T01:23:31.709726"
      },
      {
        "id": 2,
        "title": "string",
        "content": "string",
        "readCount": 0,
        "likesCount": 0,
        "userId": 2,
        "modifiedDateTime": "2022-03-04T01:29:20.939937"
      },
      {
        "id": 3,
        "title": "string",
        "content": "string",
        "readCount": 0,
        "likesCount": 0,
        "userId": 2,
        "modifiedDateTime": "2022-03-04T01:29:26.176915"
      },
        ...(생략)
        ,
      {
        "id": 10,
        "title": "바르셀로나 433",
        "content": "가나다라마바사 3줄요약",
        "readCount": 0,
        "likesCount": 0,
        "userId": 1,
        "modifiedDateTime": "2022-03-04T15:55:10.034801"
      }
    ],
    "pageable": {
      "sort": {
        "empty": true,
        "sorted": false,
        "unsorted": true
      },
      "offset": 0,
      "pageSize": 10,
      "pageNumber": 0,
      "unpaged": false,
      "paged": true
    },
    "last": false, // 마지막 페이지 여부
    "totalElements": 11, // 전체 article 수
    "totalPages": 2, // 전체 페이지 개수 총 11개의 데이터이고 size 가 10 이므로 2페이지 존재
    "size": 10, // 페이지당 출력 개수
    "number": 0,
    "sort": {
      "empty": true,
      "sorted": false,
      "unsorted": true
    },
    "first": true, // 첫 페이지 존재 여부
    "numberOfElements": 10, // 요청 페이지에서 조회 된 데이터 개수
    "empty": false
  }
}
```

실제 요청을 전달할때 파라미터 값이 존재하면 Pagealbe 객체를 생성하려고 시도한다. 

파라미터의 종류는 3가지이다.

-  page: 요청할 페이지 번호
- size: 한 페이지 당 조회 할 개수(deafult: 20)

- sort: 기본적으로 오름차순, 표기는 정렬한 필드명, 정렬기준 

  ex) createdDate, desc



#### **[ArticleService]**

```java
    @Transactional
    public Page<ArticleFindDto> findAllArticle(int page) {
        // Pageable의 page는 0부터 시작하기 때문에 화면에서 올라온 값에서 1을 뺀다.
        // 화면의 Pagenation은 1부터 시작
        int pageNumber = page - 1;
        Pageable pageable = PageRequest.of(pageNumber, 10); // 한화면 사이즈 10
        Page<Article> article = articleRepository.findAll(pageable);

        Page<ArticleFindDto> articleFindDto = article.map(new Function<Article, ArticleFindDto>() {
            @Override
            public ArticleFindDto apply(Article article) {
                ArticleFindDto dto = new ArticleFindDto();
                dto.convertEntityToDto(article);
                return dto;
            }
        });
        return articleFindDto;
```



#### [PageUtils]

```java
public class PageUtils {

    static int pageScale = 5;

    public static Map<String, Object> getPages(Pageable page, int totalPage){
        Map<String, Object> pageMap = new HashMap<String, Object>();
        int size = page.getPageSize(); // ArticleService에서 정해준 size를 가져옴 (10)
        int pageNumber = page.getPageNumber() + 1; // pageNumber 시작은 0부터
        int startPage = ((pageNumber - 1) / pageScale) * pageScale;
        int endPage = startPage + pageScale - 1;

        if ( endPage >= totalPage){
            endPage = totalPage;
        }

        int inPage = (pageNumber - 1) / size + 1;

        pageMap.put("StartPage", startPage);
        pageMap.put("EndPage", endPage);
        return pageMap;
    }
}
```



#### [WebConfig]를 이용해서 Pageable 커스터마이징

```java
@Configuration
public class WebConfig implements WebMvcConfiurer{
    
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        SortHandlerMethodArgumentResolver sortArgumentResolver = new SortHandlerMethodArgumentResolver();
        sortArgumentResolver.setSortParameter("sortBy");
        sortArgumentResolver.setPropertyDelimiter("-");

        PageableHandlerMethodArgumentResolver pageableArgumentResolver = new PageableHandlerMethodArgumentResolver(sortArgumentResolver);
        pageableArgumentResolver.setOneIndexedParameters(true);
        pageableArgumentResolver.setMaxPageSize(500);
        pageableArgumentResolver.setFallbackPageable(PageRequest.of(0,10));
        argumentResolvers.add(pageableArgumentResolver);

    }
}
```

1. SortHandlerMethodArgumentResolver : Sorting에 대한 설정을 수정하는 Resolver

   - setSortParameter : sort 요청 시 요청 파라미터를 수정할 수 있다. 

     Default는 sort이며 예제에서는 sortBy로 수정

   - setPropertyDelimiter : 정렬조건 값을 전달할 때 정렬조건필드 property와 정렬기준 property를 구분하는 구분자를 설정할 수 있다. Default는 ",' 로 구분하며, 예제에서는 "-"로 수정

   - setSortFallback : 기본 정렬 조건을 설정하는 부분 

     예제에는 없지만, sort 요청이 없는 경우 여기서 설정한 Sort 정보로 정렬을 하게된다.



2. PageableHandlerMethodArgumentResolver : Paging에 대한 설정을 수정하는 Resolver

   - setOneIndexParameters : page 기본값을 1로 설정하게 한다. Pageable 파라미터 중 page는 페이지 번호를 뜻하는데, 1페이지가 0으로 인식된다. 그래서 화면에서 쓰이는 페이지 번호랑 상이한 케이스가 있어 이 값을 true로 설정하면 화면에서 쓰이는 페이지 번호와 동일하게 1페이지부터 시작할 수 있다.

   - setMaxPageSize : Paging 요청에 대해 한 번에 많은 갯수를 요청하는 경우를 대비하여, 최대 요청 가능한 size를 설정할 수 있다.

   - setFallbackPageable : 페이지 요청이 없는 경우 기본적으로 요청되는 페이징 정보를 설정할 수 있다. Default는 page 0, size 20 

   - setPageParameterName : page 파라미터명을 수정할 수 있다.

   - setSizeParameterName : size 파라미터명을 수정할 수 있다.





