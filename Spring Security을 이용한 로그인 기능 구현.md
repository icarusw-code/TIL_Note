## Spring Security을 이용한 로그인 기능 구현

1. ### Spring Security 세팅

   ------

   **[build.grandle]**

   ```
   // security 관련 의존성
   implementation 'org.springframework.boot:spring-boot-starter-security'
   
   // 테스트 사용
   implementation 'org.springframework.security:spring-security-test'
   	
   // jwt 관련 의존성
   implementation group: 'io.jsonwebtoken', name: 'jjwt-api', version: '0.11.2'
   implementation group: 'io.jsonwebtoken', name: 'jjwt-impl', version: '0.11.2'
   implementation group: 'io.jsonwebtoken', name: 'jjwt-jackson', version: '0.11.2
   ```

   **[application.yml]**

   ```
   jwt:
     secret: {64byte 이상을 Base64로 인코딩한 값} // HS512 알고리즘을 사용하기 때문
   ```

   header, token-validity-in-seconds 등 정보를 미리 입력해 둘 수도 있다.

   512bit(64byte) 시크릿키는 Base64로 인코딩 한 값을 넣어준다.

   터미널에 $ echo '문자열' | base64 를 입력하면 값을 인코딩할 수 있다.

   

2. ### Entity 생성

   ------

   **[User]**

   ```java
   @Entity
   @Getter
   @NoArgsConstructor
   @Table(name = "USER")
   public class User extends BaseTimeEntity {
   
       @Id
       @GeneratedValue(strategy = GenerationType.IDENTITY)
       private Long id;
   
       @Column(nullable = false, length = 40)
       private String email;
   
       @Column(nullable = false)
       private String password;
   
       @Column(nullable = false, length = 20)
       private String nickname;
   
       @Enumerated(EnumType.STRING)
       private Authority authority;
   
       @Builder
       public User(String email, String password, String nickname, Authority authority, String role) {
           this.email = email;
           this.password = password;
           this.nickname = nickname;
           this.authority = authority;
       }
   ```

   **[Authority]**

   ```java
   public enum Authority {
       ROLE_USER, ROLE_ADMIN
   }
   ```

   유정 권한을 주기 위해 Enum 타입으로 클래스를 만들어준다.

   🛑주의🛑

   엔티티의 이름은 Spring Security에서 기본적으로 User를 사용하기 때문에 User보다는 Member로 표현하는것이 좋다. 만약, User로 설정했다면 만들어놓은 User 클래스인지 , Spring Security의 User 클래스인지 잘 구분해서 사용해야 한다.



3. ### TokenDto 생성

   ------

   ```java
   @Getter
   @NoArgsConstructor
   @AllArgsConstructor
   @Builder
   public class TokenDto {
   
       private String grantType;
       private String accessToken;
       private String refreshToken;
       private Long accessTokenExpiresIn;
   }
   ```

4. ### TokenProvider 생성

   ------

   ```java
   @Slf4j
   @Component
   public class TokenProvider {
   
       private static final String AUTHORITIES_KEY = "auth";
       private static final String BEARER_TYPE = "bearer";
       // 2일
       private static final long ACCESS_TOKEN_EXPIRE_TIME = 1000 * 60 * 60 * 24 * 2;
       // 2주
       private static final long REFRESH_TOKEN_EXPIRE_TIME = 1000 * 60 * 60 * 24 * 14;  
   
       private final Key key;
   
       public TokenProvider(@Value("${jwt.secret}") String secretKey) {
           byte[] keyBytes = Decoders.BASE64.decode(secretKey);
           this.key = Keys.hmacShaKeyFor(keyBytes);
       }
   
       /**
        * 유저 정보를 넘겨받아 Access Token 과 Refresh Token 생성
        */
       public TokenDto generateTokenDto(Authentication authentication){
           // authentication 으로 권한들 가져오기
           String authorities = authentication.getAuthorities().stream()
                   .map(GrantedAuthority::getAuthority)
                   .collect(Collectors.joining(","));
   
           long now = (new Date()).getTime();
   
           // Access Token 생성
           Date accessTokenExpiresIn = new Date(now + ACCESS_TOKEN_EXPIRE_TIME);
           String accessToken = Jwts.builder()
                   .setSubject(authentication.getName())  // payload "sub" : "name"
                   .claim(AUTHORITIES_KEY, authorities)   // payload "auth" : "ROLE_USER"
                   .setExpiration(accessTokenExpiresIn)   // payload "exp" : "유효기간"
                   .signWith(key, SignatureAlgorithm.HS512)  // header "alg" : "HS512"
                   .compact();
   
           // Refresh Token 생성
           String refreshToken = Jwts.builder()
                   .setExpiration(new Date(now + REFRESH_TOKEN_EXPIRE_TIME))
                   .signWith(key, SignatureAlgorithm.HS512)
                   .compact();
   
           return TokenDto.builder()
                   .grantType(BEARER_TYPE)
                   .accessToken(accessToken)
                   .accessTokenExpiresIn(accessTokenExpiresIn.getTime())
                   .refreshToken(refreshToken)
                   .build();
       }
   
       public Authentication getAuthentication(String accessToken){
   
           // 토큰 복호화
           Claims claims = parseClaims(accessToken);
   
           if(claims.get(AUTHORITIES_KEY) == null){
               throw new RuntimeException("권한 정보가 없는 토큰입니다.");
           }
   
           // 클레임에서 권한 정보 가져오기
           Collection<? extends GrantedAuthority> authorities =
               Arrays.stream(claims.get(AUTHORITIES_KEY).toString().split(","))
                   .map(SimpleGrantedAuthority::new)
                   .collect(Collectors.toList());
   
           // UserDetails 객체를 만들어서 Authentication 리턴
           UserDetails principal = new User(claims.getSubject(), "", authorities);
   
           return new UsernamePasswordAuthenticationToken(principal, "", authorities);
   
       }
   
       // 토큰 유효성 검증 (Jwts 묘듈이 알아서 예외처리)
       public boolean validateToken(String token){
           try {
               Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token);
               return true;
           } catch (io.jsonwebtoken.security.SecurityException | MalformedJwtException e) {
               log.info("잘못된 JWT 서명입니다.");
           } catch (ExpiredJwtException e) {
               log.info("만료된 JWT 토큰입니다.");
           } catch (UnsupportedJwtException e) {
               log.info("지원되지 않는 JWT 토큰입니다.");
           } catch (IllegalArgumentException e) {
               log.info("JWT 토큰이 잘못되었습니다.");
           }
           return false;
       }
   
       private Claims parseClaims(String accessToken) {
           try{
               return Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(accessToken).getBody();
           } catch (ExpiredJwtException e){
               return e.getClaims();
           }
       }
   }
   ```

   - 생성자

     - application.yml에 정의해놓은 jwt.secret 값을 가져와서 jwt를 만들 때 사용하는 암호화 키값을 생성
     - BASE64 Decode를 이용한다.

   - generateTokenDto

     - Spring Security가 제공하는 Authentication을 이용하여 권한들(유저정보)을 가져온다.
     - 받아 온 유저 정보를 이용하여 Access Token 과 Refresh Token을 생성한다.
     - username으로 User ID를 저장했기 때문에 authentication.getName() 에서 User ID를 가져온다.
     - Access Token 에는 유저와 권환 정보를 담고 Refresh Token에는 아무것도 담지 않는다.

   - getAutentication

     - accessToken에만 유저 정보가 있기 때문에 accessToken을 파라미터로 받는다.
     - Jwt 토큰을 복호화 하여 토큰에 들어 있는 정보를 꺼낸다.
     - SecurityContext가 Authentication 객체를 저장하기 때문에 SecurityContext를 사용하기 위해 UserDetails 객체를 생성해 UsernamePasswordAuthenticationToken 형태로 리턴한다.
     - parseClaims 메소드는 만료된 토큰이어도 정보를 꺼낼 수 있어야 하기 때문에 따로 분리한다.

   - validateToken 

     - 토큰 유효성을 검증한다. 

     - Jwt 모듈이 알아서 Exception을 처리해 준다.

       

5. ### JWT를 위한 커스텀 필터를 만들기 위해 JwtFilter 생성

   ------

   ```java
   @RequiredArgsConstructor
   public class JwtFilter extends OncePerRequestFilter {
   
       public static final String AUTHORIZATION_HEADER = "Authorization";
       public static final String BEARER_PREFIX = "Bearer ";
   
       private final TokenProvider tokenProvider;
   
   // 실제 필터링 로직이 doFilterInternal 에 들어감
   // Jwt 토큰의 인증 정보를 현재 쓰레드의 SecurityContext 에 저장한다.
   // 요청이 정상적으로 Controller까지 도착했으면 SecurityContex 에 UserID 가 존재하는것이 보장
       @Override
       protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
   
           // 1. Request Header에서 토큰을 꺼냄
           String jwt = resolveToken(request);
   
           // 2. validateToken 으로 토큰 유효성 검사
           if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt)){
               Authentication authentication = tokenProvider.getAuthentication(jwt);
           // 정상 토큰이면 해당 토큰으로 Authentication 을 가져와서 SecurityContext 에 저장
               SecurityContextHolder.getContext().setAuthentication(authentication);
           }
           filterChain.doFilter(request, response);
       }
   
       // Request Header 에서 토큰 정보 꺼내오기
       private String resolveToken(HttpServletRequest request) {
   
           String bearerToken = request.getHeader(AUTHORIZATION_HEADER);
           if (StringUtils.hasText(bearerToken) && bearerToken.startsWith(BEARER_PREFIX)){
               return bearerToken.substring(7);
           }
   
           return null;
       }
   
   }
   ```

   - OncePerRequestFilter 를 extends해서 doFilterInternal을 Override 한다. 

     - 실제 필터링 로직은 doFilterInternal에 들어간다.

   - doFilterInternal

     - Jwt 토큰의 인증 정보를 현재 쓰레드의 SecurityContext에 저장한다.

     - 요청이 정상적으로 Controller까지 도착했으면 SecurityContext에 UserID가 존재하는것이 보장된다.

       하지만, DB를 조회한것이 아니라 AccessToken에 있는 UserID를 꺼낸것이라, 탈퇴로 인해 UserID가 DB에 없는 경우 등 예외 상황은 Service에서 구현해야 한다.

     - Request Header에서 AccessToken을 꺼내고, 유효성을 검사한 후 정상 토큰이면 해당 토큰으로

       Authentication을 가져와서 SecurityContext에 저장한다.

     - 가입/로그인/재발급 등 제외한 모든 Request 요청은 이 필터를 거치기 때문에 토큰 정보가 없거나 유효하지 않으면 정상적으로 수행되지 않는다.

       

6. ###  JwtSecurityConfig

   ------

   ```java
   // 만들어 놓은 TokenProvider 와 JwtFilter 를 SecurityConfig에 적용할 때 사용
   @RequiredArgsConstructor
   public class JwtSecurityConfig extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {
   
       private final TokenProvider tokenProvider;
   
       // TokenProvider 를 주입받아서 JwtFilter 를 통해 Security 로직에 필터 등록
       @Override
       public void configure(HttpSecurity httpSecurity){
           JwtFilter customFilter = new JwtFilter(tokenProvider);
           httpSecurity.addFilterBefore(customFilter, UsernamePasswordAuthenticationFilter.class);
       }
   }
   ```

   - SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity>를 extends하고 TokenProvider를 주입받아 JwtFilter를 통해 Security 로직에 필터를 등록한다.

     

7. ### JwtAuthenticationEntryPoint

   ------

   ```java
   @Component
   public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
   
       @Override
       public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
   
           // 유효한 자격증명을 제공하지 않고 접근하려하면 401 에러
           response.sendError(HttpServletResponse.SC_UNAUTHORIZED);
       }
   }
   ```

   - 유저 정보 없이 접근하면 SC_UNAUTHORIZED (401) 응답을 내려준다.



8. ### JwtAccessDeniedHandler

   ------
   ```java
   @Component
   public class JwtAccessDeniedHandler implements AccessDeniedHandler {
   
       @Override
       public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
           // 필요한 권한이 없이 접근하려 할때 403 에러
           response.sendError(HttpServletResponse.SC_FORBIDDEN);
       }
   }
   ```

   - 필요한 권한이 없이 접근하려 할때 SC_FORBIDDEN (403) 응답을 내려준다.

​	

9. ### SecurityConfig

   ------

   ```java
   import DailyFootball.demo.domain.jwt.JwtAccessDeniedHandler;
   import DailyFootball.demo.domain.jwt.JwtAuthenticationEntryPoint;
   import DailyFootball.demo.domain.jwt.TokenProvider;
   import lombok.RequiredArgsConstructor;
   import org.springframework.context.annotation.Bean;
   import org.springframework.security.config.annotation.web.builders.HttpSecurity;
   import org.springframework.security.config.annotation.web.builders.WebSecurity;
   import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
   import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
   import org.springframework.security.config.http.SessionCreationPolicy;
   import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
   import org.springframework.security.crypto.password.PasswordEncoder;
   
   @EnableWebSecurity
   @RequiredArgsConstructor
   //@EnableGlobalMethodSecurity(prePostEnabled = true)
   public class SecurityConfig extends WebSecurityConfigurerAdapter {
   
       private final TokenProvider tokenProvider;
       private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
       private final JwtAccessDeniedHandler jwtAccessDeniedHandler;
   
       @Bean
       public PasswordEncoder passwordEncoder(){
           return new BCryptPasswordEncoder();
       }
   
           /**
           * H2 데이터베이스 사용한다면 테스트 원할하게 하기위해 관련 API전부 무시
           */
   //    @Override
   //    public void configure(WebSecurity web) {
   //        web.ignoring()
   //                .antMatchers("/h2-console/**", "/favicon.ico");
   //    }
   
       @Override
       protected void configure(HttpSecurity http) throws Exception{
           // 세션이 아니므로 csrf 설정 disable
           http
                   .csrf().disable()
                   // 만든 에러 추가
                   .exceptionHandling()
                   .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                   .accessDeniedHandler(jwtAccessDeniedHandler)
              		
                   // h2-console 을 위한 설정을 추가
   //                .and()
   //                .headers()
   //                .frameOptions()
   //                .sameOrigin()
   
                   // 세션 사용하지 않게 설정
                   .and()
                   .sessionManagement()
                   .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
               
   				// 우선 다 허용
                   .and()
                   .authorizeRequests()
                   .antMatchers("/**").permitAll()
            
               //JwtFilter 를 addFilterBefore 로 등록했던 JwtSecurityConfig 클래스를 적용
                   .and()
                   .apply(new JwtSecurityConfig(tokenProvider));
       }
   
   
   }
   ```

   - WebSecurityConfigurerAdapter 를 extends한다. SpringSecurity의 가장 기본적인 설정이며 JWT를 사용하지 않더라도 설정은 기본적으로 들어간다.

   - TokenProvider, JwtAuthenticationEntryPoint, JwtAccessDeniedHandler 를 주입받는다.

   - passwordEncoder는 BCryptPasswordEncoder 를 사용한다.

   - 세션이 아니므로 crsf 설정은 disable 해주도록한다.

   - 세션을 사용하지 않게 세션설정을 STATELESS로 설정한다.

   -  ```java
     http
     	.authorizeReuquests()
     	.antMatchers("/user/**").authenticated() // 해당주소로 들어오면 인증이 필요함
         .antMatchers("manager/**").access("hasRole('ROLE_ADMIN') or hasRole('ROLE_USER')")
         .antMatchers("/admin/**").access("hasRole('ROLE_ADMIN')")
         .anyRequest().permitAll() // 이 3개 주소가 아니라면 권한이 허용된다.
         .and()
         .formLogin()
         .loginPage("/login"); // 권한이 없는 페이지 요청하면 로그인페이지로 이동시킨다.
     ```

     이 외 다양한 설정들이 많으므로 설계에 맞춰서 설정을 적용하면 된다.

   - 컨트롤러에 @PreAuthorize 검증 어노테이션을 사용하려면 SecurityConfig에 @EnableGlobalMethodSecurity(prePostEnabled = true)를 추가해주어야 한다.

     

10. ### SecurityUtil

    ------

    ```java
    @Slf4j
    public class SecurityUtil{
    
        private SecurityUtil(){ }
    
        /**
         * SecurityContext에 유저 정보가 저장되는 시점
         * Request가 들어올 때 JwtFilter의 doFilter에서 저장한다.
         */
        public static Long getCurrentUserId(){
            final Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    
            if(authentication == null || authentication.getName() == null){
                throw new RuntimeException("Security Context 에 인증 정보가 없습니다.");
            }
    
            return Long.parseLong(authentication.getName());
        }
    }
    ```

    - JwtFilter에서 SecurityContext에 세팅한 유저 정보를 꺼낸다.

    - UserId를 저장하게 했으므로 Long타입으로 파싱해서 반환한다.

    - SecurityContext는 ThredLocal에 사용자의 정보를 저장한다.

      

11. ### 외부와의 통신에 사용할 DTO클래스 , 엔티티 추가 생성 

    ------

    **RefreshToken**

    ```java
    @Getter
    @NoArgsConstructor
    @Table(name = "refresh_token")
    @Entity
    public class RefreshToken {
    
        @Id
        private String tokenKey;
        private String tokenValue;
    
        public RefreshToken updateValue(String tokenKey) {
            this.tokenValue = tokenKey;
            return this;
        }
    
        @Builder
        public RefreshToken(String tokenKey, String tokenValue) {
            this.tokenKey = tokenKey;
            this.tokenValue = tokenValue;
        }
    }
    ```

    **TokenRequestDto**

    ```java
    @Getter
    @NoArgsConstructor
    public class TokenRequestDto {
        private String accessToken;
        private String refreshToken;
    }
    ```

    **UserRequestDto**

    ```java
    @Getter
    @AllArgsConstructor
    @NoArgsConstructor
    public class UserRequestDto {
    
        private String email;
        private String password;
    
        public User toUser(PasswordEncoder passwordEncoder){
            return User.builder()
                    .email(email)
                    .password(passwordEncoder.encode(password))
                    .authority(Authority.ROLE_USER)
                    .build();
        }
    
        public UsernamePasswordAuthenticationToken toAuthentication(){
            return new UsernamePasswordAuthenticationToken(email, password);
        }
    
    }
    ```

    **UserResponseDto**

    ```java
    @Getter
    @AllArgsConstructor
    @NoArgsConstructor
    // 바디에 반환할 값을 정해줌
    public class UserResponseDto {
    //    private String email;
        private Long id;
    
        public static UserResponseDto of(User user){
            return new UserResponseDto(user.getId());
        }
    }
    
    ```

    **UserSingupRequestDto**

    ```java
    @Getter
    @NoArgsConstructor
    public class UserSignupRequestDto {
    
        private String email;
        private String nickname;
        private String password;
    
        @Builder
        public UserSignupRequestDto(String email, String nickname, String password){
            this.email = email;
            this.nickname = nickname;
            this.password = password;
        }
    
        public User toUser(PasswordEncoder passwordEncoder){
            return User.builder()
                    .email(email)
                    .nickname(nickname)
                    .password(passwordEncoder.encode(password))
                    .authority(Authority.ROLE_USER)
                    .build();
        }
    
        public User toEntity(){
            return User.builder()
                    .email(email)
                    .nickname(nickname)
                    .password(password)
                    .authority(Authority.ROLE_USER)
                    .build();
        }
    
    }
    ```

12. ### Repository 생성

    ------

    **RefreshTokenRepository**

    ```java
    @Repository
    public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {
        Optional<RefreshToken> findByTokenKey(String tokenKey);
    }
    ```

    User ID 값으로 토큰을 가져오기 위해 findByTokeKey 추가

    

13. ### Service 생성

    ------

    **CustomUserDetailsService**

    ```java
    @Service
    @RequiredArgsConstructor
    public class CustomUserDetailsService  implements UserDetailsService {
    
        private final UserRepository userRepository;
    
        @Override
        @Transactional
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
            return userRepository.findByEmail(username)
                    .map(this::createUserDetails)
                    .orElseThrow(() -> new UsernameNotFoundException(username + " -> 데이터베이스에서 찾을 수 없습니다."));
        }
    
        // DB 에 User 값이 존재하면 UserDetails 객체로 만들어서 리턴
        private UserDetails createUserDetails(User user) {
            GrantedAuthority grantedAuthority = new SimpleGrantedAuthority(user.getAuthority().toString());
    
            return new org.springframework.security.core.userdetails.User(
                    String.valueOf(user.getId()),
                    user.getPassword(),
                    Collections.singleton(grantedAuthority)
            );
        }
    }
    
    ```

    - UserDetailsService 인터페이스를 구현한 클래스

    - loadUserByUsername 메소드를 오버라이드해서 넘겨받은 UserDetails와 Authentication의 패스워드를 비교하고 검증하는 로직을 처리

    - DB에서 username을 기반으로 값을 가져오기 때문에 아이디 존재 여부도 자동 검증

    - lodaUserByUsername 메소드를 어디서 호출하는가?

      1. CustomUserDetailService

         ![1](https://user-images.githubusercontent.com/77170611/156134325-085ce053-5b3b-4ca6-b4ce-2a8eb78fdd07.png)

      2. DaoAuthenticationProvider

         ![image](https://user-images.githubusercontent.com/77170611/156134513-363a3dd1-5e17-4e37-8931-2c507f94bca2.png)

         loadUserByUsername은 username을 받아서 넘겨주는 retrueveUser 메소드 내부에서 호출한다.

      3. AbstractUserDetailsAuthenticationProvider

         ![1](https://user-images.githubusercontent.com/77170611/156135559-49669c45-2bc1-4df1-be2a-e4a77de8d51b.png)

         retrieveUser는 

         DaoAuthenticationProvider의 부모 클래스인AbstractUserDetailsAuthenticationProvider 에서 호출
      
         받아온 user 변수로 additionalAuthenticationChecks 메소드를 호출한다.
      
         additionalAuthenticationChecks 는 추상클래스이고, 오버라이드 해서 구현되어 있다.
      
      4. DaoAuthenticationProvider
      
         ![1](https://user-images.githubusercontent.com/77170611/156136852-7ed63a7a-3c64-45b3-845a-68eaa85b3506.png)
      
         `실제로 password 검증이 이루어지는 부분`
      
         Request로 받아서 만든 authentication 과 DB에서 꺼낸 값인 userDetails 비밀번호를 비교한다.
      
         DB에는 암호화 되어 있지만 passwordEncoder가 인코딩해서 비교해준다.
      
         즉 , password의 검증은 Spring Security가 제공하는 클래스에 의해서 이루어진다.
      
         AuthenticationManager 과의 관계를 살펴보면 AbstractUserDetailsAuthenticationProvider 의 
      
         authenticate는 ProviderManager 단 한곳에서 호출된다.
      
      5. ProviderManager 
      
         ![1](https://user-images.githubusercontent.com/77170611/156138327-eecc9e25-4ac5-43ba-914b-679b16d9004d.png)
      
         authenticate 메소드는 AuthenticationProvider 인터페이스에서 호출한다.
      
         AbstractUserDetailsAuthenticationProvider 의 상위 인터페이스 이다.
      
         ProviderManager.authenticate를 호출하는곳은 만들어놓은 UserService 이다.
      
         ![1](https://user-images.githubusercontent.com/77170611/156139625-3fdd42dc-edd2-4acf-9945-592bab1600b8.png)
      
      6. UserService 
      
         ProviderManger는 AutheticationManger의 구현체이다.
      
      **최종 정리**
      
      ![](https://user-images.githubusercontent.com/77170611/156143105-095e2655-0b88-4b77-accd-0cb0a9327f72.png)
      
      1. UserService에서 AuthenticationManagerBuilder를 주입받음
      
      2. AuthenticationManagerBuilder 에서 AuthenticationManager 를 구현한 ProviderManager 생성
      
      3. ProviderManager 는 AbstractUserDetailsAuthenticationProvider 의 자식 클래스인 DaoAuthenticationProvider를 주입 받아서 호출
      
      4. DaoAuthenticationProvider의 authenticate에서는 retrieveUser로 DB에 있는 사용자 정보를 가져옴
      
         additionalAuthenticationChecks로 비밀번호 비교
      
      5. retrieveUser  내부에서 UserDetailsService 인터페이스를 직접 구현한 CustomUserDetailsService 클래스의 오버라이드 메소드인 loadUserByUsername가 호출됨

    **UserService**

    ```java
        /**
         *  회원 가입
         */
        @Transactional
        public Long signup(UserSignupRequestDto userSignupRequestDto){
            User user = userSignupRequestDto.toUser(passwordEncoder);
            return UserResponseDto.of(userRepository.save(user)).getId();
        }
        
        /**
         * 로그인
         */
        @Transactional
        public TokenDto login(UserRequestDto userRequestDto){
    
            // 1. Login ID/PW 를 기반으로 AuthenticationToken 생성
            UsernamePasswordAuthenticationToken authenticationToken = userRequestDto.toAuthentication();
    
            // 2. 실제로 검증(사용자 비밀번호 체크) 가 이루어짐
            // authenticate 메서드가 실행이 될 때 CustomUserDetailsService 에서 만들었던 loadUserByUsername 메서드가 실행됨
            Authentication authentication = authenticationManagerBuilder.getObject().authenticate(authenticationToken);
    
            // 3. 인증 정보를 기반으로 JMT 토큰 생성
            TokenDto tokenDto = tokenProvider.generateTokenDto(authentication);
    
            // 4. RefreshToken 저장
            RefreshToken refreshToken = RefreshToken.builder()
                    .tokenKey(authentication.getName())
                    .tokenValue(tokenDto.getRefreshToken())
                    .build();
    
            refreshTokenRepository.save(refreshToken);
    
            // 5. 토큰 발급
            return tokenDto;
        }
    
        /**
         * 토큰 재발급
         */
        @Transactional
        public TokenDto reissue(TokenRequestDto tokenRequestDto){
    
            // 1. Refresh Token 검증
            if(! tokenProvider.validateToken(tokenRequestDto.getRefreshToken())){
                throw new RuntimeException("Refresh Token 이 유효하지 않습니다.");
            }
    
            // 2. Access Token 에서 User ID 가져오기
            Authentication authentication = tokenProvider.getAuthentication(tokenRequestDto.getAccessToken());
    
            // 3. 저장소에서 User ID를 기반으로 Refresh Token 값 가져오기
            RefreshToken refreshToken = refreshTokenRepository.findByTokenKey(authentication.getName())
                    .orElseThrow(() -> new RuntimeException("로그아웃 된 사용자입니다."));
    
            // 4. Refresh Token 일치하는지 검사
            if (! refreshToken.getTokenValue().equals(tokenRequestDto.getRefreshToken())){
                throw new RuntimeException("토큰의 유저 정보가 일치하지 않습니다.");
            }
    
            // 5. 새로운 토큰 생성
            TokenDto tokenDto = tokenProvider.generateTokenDto(authentication);
    
            // 6. 저장소 정보 업데이트
            RefreshToken newRefreshToken = refreshToken.updateValue(tokenDto.getRefreshToken());
            refreshTokenRepository.save(newRefreshToken);
    
            // 토큰 발급
            return tokenDto;
        }
    ```

    **[로그인]**

    - Authentication
      - 사용자가 입력한 Login ID, PW 로 인증 정보 객체 UsernamePasswordAuthenticationToken 를 생성
      - 아직 인증이 된 객체가 아니므로 AuthenticationManager에서 authenticate메소드의 파라미터로 넘겨서 검증후 Authentication 를 받는다.
    - AuthenticationManager
      - Spring Security에서 실제로 인증이 이루어지는 곳
      - authenticate 메소드 하나만 정의되어 있는 인터페이스이며 위 코드에서 Builder에서 UserDetails의 유저 정보가 서로 일치하는지 검사 한다.
      - 이 로직은 SpringSecurity 내부적으로 작동한다.
    - 인증이 완료된 authentication 에는 User ID가 들어있다.
    - 인증 객체를 바탕으로 AccessToken 과 Refresh Token을 생성한다.
    - RefreshToken은 저장하고, 생성된 토큰 정보를 클라이언트에 전달한다.

    

    **[재발급]**

    - AccessToken과 RefreshToken을 Request Body에 받아서 검증한다.

    - Refresh Token의 만료 여부를 먼저 검사한다.

    - Access Token을 복호화하여 유저 정보(User ID)를 가져오고 저장소에 있는 Refresh Token과 클라이언트가 전달한 Refresh Token의 일치여부를 검사한다.

    - 만약 일치한다면 로그인 했을 때와 동일하게 새로운 토큰을 생성해서 클라이언트에게 전달한다.

    - Refresh Token은 재사용하지 못하도록 저장소에서 값을 갱신해준다.

      

14. ### UserApiController

    ------

    ```java
    @RestController
    @RequiredArgsConstructor
    public class UserApiController {
    
        private final UserService userService;
        
        // 회원가입
        @PostMapping("/signup")
        public ResponseEntity signup(@RequestBody UserSignupRequestDto userSignupRequestDto){
            Map<String, Object> responseMap = new HashMap<>();
            Long userId = userService.signup(userSignupRequestDto);
            responseMap.put("userId", userId);
            return ResponseEntity.status(HttpStatus.OK).body(responseMap);
        }
        
    	// 로그인
        @PostMapping("/login")
        public ResponseEntity<TokenDto> login(@RequestBody UserRequestDto userRequestDto){
            return ResponseEntity.ok(userService.login(userRequestDto));
        }
    
        // 재발급
        @PostMapping("/reissue")
        public ResponseEntity<TokenDto> reissue(@RequestBody TokenRequestDto tokenRequestDto){
            return ResponseEntity.ok(userService.reissue(tokenRequestDto));
        }
    ```

    





