# - 프로젝트 개요
예전 PHP 프레임워크인 Laravel을 통해 개발한 회의실 예약 프로젝트의 일부 기능을 프론트 영역은 react와 react 프레임워크인 Next.js, 백엔드 영역은 Spring Boot, Spring Data JPA, Spring Security 등을 통해 구현해 보았으며 해당 문서에서는 백엔드 부분에 대해 중점적으로 다루었다.
# - 개발기간
- 25.04 ~ 25.05(약 1.5개월)
# - 개발환경
- JAVA v1.8
- Spring Boot, Spring Data Jpa v3.4.3
- Spring Security v3.4.3
- Jsonwebtoken v0.12.3
- gradle v8.13
- lombok 등 라이브러리 및 Mysql DB
# - 주요기능
- 사용자인증: \
Spring Security를 통한 사용자 인증 및 권한 제어, JWT 인증 토큰 및 refresh 토큰 발급, 재발급 기능
- 게시판 :\
게시판 CRUD 기능, Pageable 인터페이스를 통한 페이징 처리, Specification을 통한 검색 기능, 첨부 파일 처리
- 코멘트 :\
코멘트 CRUD 기능, 코멘트 트리 리스트 구현
- 예약 :\
예약 CRUD 기능
# - 특이사항
- Spring Security를 통한 인증 처리, 권한 제어 JWT 인증 토큰 발급/재발급 기능을 통한 사용자 접근 제어
- Pageble, Specification 인터페이스를 통한 쿼리 리스트 조회 처리
- 코멘트에 자기 참조 관계 설정 및 리스트 트리 구현
- N:N 관계 처리를 위해 중간 테이블 추가 및 데이터 처리

## 1. DB구조
### 1.1 사용자
사용자는 여러개의 Role을 가질 수 있고 Role 역시 여러 사용자에게 할당 될 수 있으므로 사용자와 Role은 N:N 관계이며 이를 표현 하기 위해 중간 테이블인 role_user 테이블을 두어 사용자가 추가 될 시 role_user 테이블에 사용자 아이디와 Role 아이디를 가진 데이터가 추가 되어야 한다.

![Image](https://github.com/user-attachments/assets/bd1fd004-4c3c-45c7-a4e3-b89ac4b3c782)

### 1.2 게시판
게시판은 게시글을 작성하는 사용자와 N:1, 게시글에 첨부되는 첨부파일과 1:N, 게시글의 코멘트와 1:N 관계이다. 코멘트의 경우 답글 기능으로 자기 참조 데이터가 생성되므로 자기자신을 1:N으로 참조 하게 된다.

![Image](https://github.com/user-attachments/assets/45dccb1d-cf3d-4d53-b10e-c414cc9fc6e0)

### 1.3 예약
사용자는 여러 예약을 가질 수 있으므로 사용자와 예약은 1:N 관계이며 예약은 여러개의 예약시간을 가질 수 있고 예약시간 역시 여러 예약에 할당 될 수 있으므로 예약과 예약시간은 N:N 관계이며 이를 표현 하기 위해 중간 테이블인 reserve_time 테이블을 두어 예약이 추가 될 시 reserve_time 테이블에 예약 아이디와 예약시간 아이디를 가진 데이터가 추가 되어야 한다.

![Image](https://github.com/user-attachments/assets/9fa9f89c-41be-4505-93e1-3622724819aa)


## 2. 프로젝트 기본 구조
### 2.1 생성자 패턴
프로젝트는 특정한 형태의 생성자가 필요한 경우 직접 생성자를 선언하거나 Lombok 어노테이션을 사용하여 빌터 패턴을 사용하여 구현 하였다.

- ex)BoardDTO.class
```java
// 페이징 처리를 위해 직접 선언
public BoardDTO(Long id, String boardWriter, String boardTitle, int boardHits, LocalDateTime boardCreatedTime) {
    this.id = id;
    this.boardWriter = boardWriter;
    this.boardTitle = boardTitle;
    this.boardHits = boardHits;
    this.boardCreatedTime = boardCreatedTime;
  }
// 상세보기 페이지를 위해 직접 선언
  public static BoardDTO toBoardDTO(BoardEntity boardEntity) {
    BoardDTO boardDTO = new BoardDTO();
    boardDTO.setId(boardEntity.getId());
    boardDTO.setBoardWriter(boardEntity.getBoardWriter());
    boardDTO.setBoardTitle(boardEntity.getBoardTitle());
    boardDTO.setBoardContents(boardEntity.getBoardContents());
    boardDTO.setBoardHits(boardEntity.getBoardHits());
    boardDTO.setBoardCreatedTime(boardEntity.getCreatedTime());
    boardDTO.setBoardUpdatedTime(boardEntity.getUpdatedTime());
    boardDTO.setFileAttached(boardEntity.getFileAttached());
    return boardDTO;
  }

- ex)BoardEntity.class
```java
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Table(name = "board")
public class BoardEntity extends BaseEntity {
        ...
  // 빌더패턴을 통한 구현
    public static BoardEntity toSaveEntity(BoardDTO boardDTO) {
    return BoardEntity.builder()
            .id(boardDTO.getId())
            .boardWriter(boardDTO.getBoardWriter())
            .boardTitle(boardDTO.getBoardTitle())
            .boardContents(boardDTO.getBoardContents())
            .fileAttached(boardDTO.getFileAttached())
            .boardHits(0)
            .build();
```

### 2.2 ORM 기반 구현
Entity 객체는 BaseEntity를 상속 받도록 하였고 Spring Data JPA 모듈을 사용하여 관계 어노테이션을 통해 엔티티간에 관계를 정의 하였다.
- ex)BoardEntity.class
```java
public class ReserveEntity extends BaseEntity {
...
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String reserveReason;
  private String reserveDate;
  private String reservePeriod;

  private String reserveUserId;
  private String userName;


  // 관계 어노테이션을 통한 엔티티 매핑
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "user_id")
  private UserEntity userEntity;


  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "hall_id")
  private HallEntity hallEntity;


  @OneToMany(mappedBy = "reserveEntity", cascade = CascadeType.REMOVE, orphanRemoval = true, fetch = FetchType.LAZY)
  private final List<ReserveTimeEntity> reserveTimeEntity = new ArrayList<>();
  ...

```

### 2.3 JpaRepository 상속 구현
각 리포지토리 인터페이스는 JpaRepository를 상속받아 기본 쿼리 메서드를 통해 테이블에 접근하며 인터페이스에 쿼리 메서드를 정의, @Query 어노테이션 사용하여 직접 쿼리 선언하는 방식 등을 통해 구현해 보았다.

- ex)UserServiceImpl.class
```java
// UserRepository 인터페이스 JpaRepository 상속 받아 기본 제공 메서드를 사용
userRepository.findById(userDto.getId());

```
- ex)UserRepository.class
```java

public interface UserRepository extends JpaRepository<UserEntity, Long>, JpaSpecificationExecutor<UserEntity> {
        
  // JPA에서 제공되는 쿼리 메서드를 통해 사용자 정보 조회
  Boolean existsByLoginId(String loginId);

  UserEntity findByLoginId(String loginId);

  UserEntity findByLoginIdAndUserPassword(String loginId, String userPassword);

  // @Query 어노테이션을 통해 엔티티에 대한 쿼리를 직접 선언
  // 네이티브 쿼리로 직접 작성하고 싶다면 nativeQuery = true로 설정 
  @Query("SELECT u FROM UserEntity u LEFT JOIN FETCH u.roleUserEntities")
  Page<UserEntity> findAllWithRelated(Pageable pageable);

  @Query("SELECT u FROM UserEntity u")
  Page<UserEntity> findAllWithPageble(Specification specification, Pageable pageable);
}
```
- ex)BoardRepository.class
```java
public interface BoardRepository extends JpaRepository<BoardEntity, Long>, JpaSpecificationExecutor<BoardEntity> {
  // @Modifying 어노테이션을 붙여 @Query에서 작성된 변경 쿼리 직접 수행
  @Modifying
  @Query(value = "update BoardEntity b set b.boardHits=b.boardHits+1 where b.id=:id")
  void updateHits(@Param("id") Long id);

  @Modifying
  @Query(value = "update BoardEntity b set b.fileAttached=0 where b.id=:id")
  void updatefileAttached(@Param("id") Long id);
}
```
이외 @Transactional 어노테이션을 통해 메서드 레벨 트랜잭션 관리 등을 구현 해보았으며 Spring Data JPA를 통해 JPA 기반으로 데이터 액세스 계층을 추상화하며 ORM 기반한 구조로 프로젝트를 구현하였다.
(Spring Data JPA가 디폴트 구현체로 Hibernate를 제공하기에 구현체는 Hibernate를 사용)


## 2. 사용자인증
사용자 인증에는 Spring에서 제공하는 Security 라이브러리를 사용 하였다.

### 2.1 스프링 시큐리티 설정
- SecurityConfig.class
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

  private final AuthenticationConfiguration authenticationConfiguration;
  private final JWTUtil jwtUtil;
  private final RefreshRepository refreshRepository;
  private final RoleRepository roleRepository;
  private final RoleUserRepository roleUserRepository;
  private final UserRepository userRepository;

  public SecurityConfig(AuthenticationConfiguration authenticationConfiguration, JWTUtil jwtUtil, RefreshRepository refreshRepository, RoleRepository roleRepository, RoleUserRepository roleUserRepository, UserRepository userRepository) {
        
  ...
  // authenticationManager 를 Bean로 등록하고 DaoAuthenticationProvider에 customUserDetailsService를 등록
  // 패스워드 암호화를 위해 bCryptPasswordEncoder 선언 후 등록
 @Bean
  public AuthenticationManager authenticationManager(
          BCryptPasswordEncoder bCryptPasswordEncoder) {
    DaoAuthenticationProvider authenticationProvider = new DaoAuthenticationProvider();
    authenticationProvider.setUserDetailsService(customUserDetailsService);
    authenticationProvider.setPasswordEncoder(bCryptPasswordEncoder);
    return new ProviderManager(authenticationProvider);
  }
  
  // 패스워드 암호화를 위해 Bean으로 등록
  @Bean
  public BCryptPasswordEncoder bCryptPasswordEncoder() {
    return new BCryptPasswordEncoder();
  }
  ...
    http.csrf((auth) -> auth.disable());

    http.formLogin((auth) -> auth.disable());
  
    http
            .authorizeHttpRequests((auth) -> auth
                    // permitAll() 인증 토큰이 없어도 접근이 가능
                    .requestMatchers(
                                "/"
                            , "/join"
                            ,"/api/v1/user/login"
                            ,"/api/v1/user/reIssueToken"
                            , "/api/v1/board/boardList"
                            , "/api/v1/board/detal/*"
                            , "/api/v1/comment/commentList"
                            , "/api/v1/user/userJoin"
                            , "/error").permitAll()
                    // hasAnyRole() 인증된 사용자의 권한을 확인
                    .requestMatchers("/api/v1/admin/*").hasAnyRole("ADMIN", "MANAGER")
                    .anyRequest().authenticated());

    // 로그인 필터 호출 전 JWTFilter 호츌
    http
            .addFilterBefore(new JWTFilter(jwtUtil), LoginFilter.class);

    // UsernamePasswordAuthenticationFilter를 커스텀 LoginFilter로 교체
    http
            .addFilterAt(new LoginFilter(authenticationManager(bCryptPasswordEncoder()), jwtUtil, refreshRepository, roleUserRepository, userRepository), UsernamePasswordAuthenticationFilter.class);
            
    // 로그아웃 필터 호출 전 CustomLogoutFilter 호츌
    http
            .addFilterBefore(new CustomLogoutFilter(jwtUtil, refreshRepository), LogoutFilter.class);
    ...
```
Security 라이브러리를 사용하기 위해 SecurityConfig 클래스를 만들고 인증이 필요한 페이지와 인증 후에도 특정 권한이 필요한 페이지를 추가 해주었다.
로그인을 위해 UsernamePasswordAuthenticationFilter 상속받는 커스텀 LoginFilter를 추가해 주었다.

### 2.2 사용자 인증 및 토큰 발급
- 스프링 시큐리티의 인증 구조

![Image](https://github.com/user-attachments/assets/995ff7ee-1dfd-488a-9178-c73ba92abee4)

- LoginFilter.class
```java
@Override
  public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
    ServletInputStream inputStream = null;
    try {
      inputStream = request.getInputStream();
    } catch (IOException e) {
      throw new RuntimeException(e);
    }
    String messageBody = null;
    try {
      messageBody = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8);
    } catch (IOException e) {
      throw new RuntimeException(e);
    }
  ...
  
    // 사용자 인증 시도 시 LoginFilter 클래스에서 attemptAuthentication를 호출하고 UsernamePasswordAuthenticationToken에 사용자 아이디와 패스워드, 권한을 저장하고 authenticationManager를 통해 인증을 시도
      RoleDTO roleDTO = new RoleDTO();
      ModelMapper mapper = new ModelMapper();

      List<SimpleGrantedAuthority> updatedAuthorities = new ArrayList<SimpleGrantedAuthority>();
      for(int i=0; i < roleUserEntityList.size(); i++) {
        roleDTO = mapper.map(roleUserEntityList.get(i).getRoleEntity()
                , new TypeToken<RoleDTO>() {
                }.getType());
        SimpleGrantedAuthority authority = new SimpleGrantedAuthority(roleDTO.getRoleName());
        updatedAuthorities.add(authority);
      }

    UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(loginId, userPassword, updatedAuthorities);
      
    return authenticationManager.authenticate(authToken);
```

- CustomUserDetailsService.class
```java
// authenticationManager를 통해 인증을 시도 하면 SecurityConfig에서 authenticationProvider.setUserDetailsService(customUserDetailsService)를 통해 customUserDetailsService를 등록 해두었으니 CustomUserDetailsService.class를 호출하게 되고 사용자를 인증 처리 한다.
 @Override
  public UserDetails loadUserByUsername(String loginId) throws UsernameNotFoundException {

    UserEntity userEntity = userRepository.findByLoginId(loginId);

    if (userEntity != null) {
      List<RoleEntity> roleEntity = new ArrayList<>();

      List<String> roles = new ArrayList<>();
      List<RoleUserEntity> roleUserEntityList = roleUserRepository.findByUserEntity(userEntity);
      ModelMapper mapper = new ModelMapper();
      for(int i=0; i < roleUserEntityList.size(); i++) {
        roles.add(roleUserEntityList.get(i).getRoleEntity().getRoleName());
      }
      return new CustomUserDetails(userEntity, roles);
    }

    return null;
  }
```

```java
// AbstractAuthenticationProcessingFilter의 successfulAuthentication를 재정의 하여 인증이 성공하면 JWT 토큰을 반환 하도록 하였다..
protected void successfulAuthentication (HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authentication) {

    String userName = authentication.getName();
    
    // 인증이 완료된 사용자의 권한들을 리스트로 만들어 인증 토큰에 담아 준다.
    List<String> role = new ArrayList<>();
    Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
    Iterator<? extends GrantedAuthority> iterator = authorities.iterator();
    while ( iterator.hasNext()){
      role.add(String.valueOf(iterator.next()));
    }
    String access = jwtUtil.createJwt("access", userName, role, 600000L);
    String refresh = jwtUtil.createJwt("refresh", userName, role, 86400000L);
    addRefreshEntity(userName, refresh, 20000L);

    response.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization, userName, Response-Header, access" );
    response.setHeader("Access-Control-Allow-Methods", "GET, POST, PATCH, PUT, DELETE, OPTIONS" );
    response.setHeader("Access-Control-Allow-Origin", "localhost:3000" );
    // 사용자 화면에서 사용자 이름을 사용하기 위해 Expose-Headers를 추가하여 Header에 값을 담아 리턴하도록 하였다.
    // Expose-Headers를 추가하지 않으면 사용자 아이디와 인증 토큰만 리턴 되기 때문.
    response.setHeader("Access-Control-Expose-Headers", "userName, access" );
    response.setHeader("access", access );
    response.setHeader("userName", userName );
    response.addCookie(createCookie("refresh", refresh));
    response.setStatus(HttpStatus.OK.value());
  }

```
- 인증 성공 시

![Image](https://github.com/user-attachments/assets/9a62388e-b4d3-4c7c-987e-3497450beb91)

![Image](https://github.com/user-attachments/assets/f0492bae-53af-4bed-8e00-088ce1d23df1)

- 인증 실패 시

![Image](https://github.com/user-attachments/assets/c756ec0f-a4bb-4305-a50a-8444e520d4c8)

### 2.3 인증 사용자 처리
인증 성공 시 액세스 토큰과 리프레시 토큰을 발급하여 헤더에 인증 정보와 쿠키로 리프레시 토큰을 리턴한다. 인증 토큰을 가진 사용자가 페이지 접근 시 addFilterBefore(new JWTFilter(jwtUtil), LoginFilter.class) 부분을 통해 JWTFilter 필터를 통해 인증 여부를 확인 한다.

- 스프링 시큐리티의 인증 구조

![Image](https://github.com/user-attachments/assets/7c7eb81d-a52f-46d8-bd1c-00a544b01bc0)

- JWTFilter.class
```java

// Filter 클래스 implements 방식이 아닌 OncePerRequestFilter 상속 받는 방식으로 구현
public class JWTFilter extends OncePerRequestFilter {

  private final JWTUtil jwtUtil;

  public JWTFilter(JWTUtil jwtUtil) {
    this.jwtUtil = jwtUtil;
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

    // 요청 헤더에서 액세스 토큰 존재 여부를 확인 한다.
    String accessToken = request.getHeader("access");
    if (accessToken == null) {
      filterChain.doFilter(request, response);
      return;
    }

    // 토큰 존재 시 토큰의 만료 여부를 확인한다.
    try {
      jwtUtil.isExpired(accessToken);
    } catch (ExpiredJwtException e) {
      PrintWriter writer = response.getWriter();
      writer.print("accessToken expired");
      response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
      return;
    }catch (JwtException e) {
      PrintWriter writer = response.getWriter();
      writer.print("accessToken not valid");
      response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
      return;
    }
    ...
    
    // 유효기간 이내의 유효한 인증 토큰이라면 토큰에서 사용자 정보와 권한 정보를 가져온다.
    UserEntity userEntity = new UserEntity();
    userEntity.setUserName(userName);
    List<RoleEntity> roleEntity = new ArrayList<>();

    List<String> userRoles = new ArrayList<>();
    userRoles = userRole;

    CustomUserDetails customUserDetails = new CustomUserDetails(userEntity, userRoles);

    Authentication authToken = new UsernamePasswordAuthenticationToken(customUserDetails, null, customUserDetails.getAuthorities());
    
    // 토큰에서 가져온 정보를 SecurityContextHolder 클래스의 getContext 통해 인증 값으로 넘기고 doFilter 메서드를 호출하여 권한 값을 확인 한다.
    SecurityContextHolder.getContext().setAuthentication(authToken);
    filterChain.doFilter(request, response);
  }
}
```
```java
                    .requestMatchers(
                                "/"
                            , "/join"
                            ,"/api/v1/user/login"
                            ,"/api/v1/user/reIssueToken"
                            , "/api/v1/board/boardList"
                            , "/api/v1/board/detal/*"
                            , "/api/v1/comment/commentList"
                            , "/api/v1/user/userJoin"
                            , "/error").permitAll()
                    // hasAnyRole() 인증된 사용자의 권한을 확인
                    .requestMatchers("/api/v1/admin/*").hasAnyRole("ADMIN", "MANAGER")
```

인증된 사용자는 requestMatchers를 통해 권한을 확인 후 페이지의 접근을 제어한다

- 권한이 있는 사용자 접근 시(토큰 내용)

![Image](https://github.com/user-attachments/assets/4d4ff1a5-0732-40fd-be85-dc0b5e654cd6)

![Image](https://github.com/user-attachments/assets/b1516e1b-d936-47eb-a04b-f0ed5fd917bb)

- 권한이 없는 사용자 접근 시(토큰 내용)

![Image](https://github.com/user-attachments/assets/e6e88283-0ed0-493a-9b6d-0b2cc573a6bc)

![Image](https://github.com/user-attachments/assets/0fc2100f-24cc-479f-9d69-31e77c574b70)

### 2.3 토큰 재발급
사용자 인증 액세스 토큰이 만료될 경우 리프레시 토큰을 통해 토큰을 재발급 받을 수 있는 기능도 구현 해보았다. 서버의 경우 리프레시을 DB 상에 저장 되도록 구현 하였다.

- DB의 리프레시 토큰

![Image](https://github.com/user-attachments/assets/54973936-1feb-47ac-b0b5-9bdecc2d7a80)

- ReissueController
```java
@PostMapping("/reIssueToken")
    public ResponseEntity<?> reIssueToken(HttpServletRequest request, HttpServletResponse response) {
        String refresh = null;
        Cookie[] cookies = request.getCookies();
        for (Cookie cookie : cookies) {
            if (cookie.getName().equals("refresh")) {
                refresh = cookie.getValue();
            }
        }
        ...
        
        // 요청된 리프레시 토큰이 DB에 존재하는지 확인
         Boolean isExist = refreshRepository.existsByRefresh(refresh);
        if (!isExist) {
            return new ResponseEntity<>("invalid refresh token", HttpStatus.BAD_REQUEST);
        }
        
        String username = jwtUtil.getUsername(refresh);
        List<String> role = jwtUtil.getRole(refresh);

        // 새로운 인증 토큰과 리프레시 토큰을 발급
        String newAccess = jwtUtil.createJwt("access", username, role, 600000L);
        String newRefresh = jwtUtil.createJwt("refresh", username, role, 86400000L);

        refreshRepository.deleteByRefresh(refresh);
        addRefreshEntity(username, newRefresh, 86400000L);

        response.setHeader("access", newAccess);
        response.addCookie(createCookie("refresh", newRefresh));

        return new ResponseEntity<>(HttpStatus.OK);
    }
```
- 인증 토큰 만료 시

![Image](https://github.com/user-attachments/assets/a809dbe6-c1cb-4375-9bf1-8ca30058a61c)

- 쿠키에 저장된 리프레시 토큰

![Image](https://github.com/user-attachments/assets/05e57d22-95c2-45b0-970c-6ae3464fae43)

- 토큰 재발급 결과

![Image](https://github.com/user-attachments/assets/1984dc46-c432-46e2-9742-af5ae9af2e18)

## 3. 게시판
게시판은 기본적인 CRUD 기능을 구현 하였으며 게시판 구현 시 특이사항에 대해서만 다루어 보겠다.

### 3.1 게시판 - Pageable, Specification
게시판의 페이징과 검색은 Pageable, Specification 인터페이스를 통해 구현 하였다.

- BoardSpecification.class
```java
@AllArgsConstructor
// Specification을 레퍼런스 문서를 참고하여 구현하였다.
// https://docs.spring.io/spring-data/jpa/reference/jpa/specifications.html
public class BoardSpecification implements Specification<BoardEntity> {

    private SearchCriteria criteria;

    @Override
    public Predicate toPredicate
            (Root<BoardEntity> root, CriteriaQuery<?> query, CriteriaBuilder builder) {

        if(criteria.getSearchKey() != null){
            if (root.get(criteria.getSearchKey()).getJavaType() == String.class) {
                   return builder.like(
                   root.<String>get(criteria.getSearchKey()), "%" + criteria.getSearchValue() + "%");
               } else {
                  return builder.equal(root.get(criteria.getSearchKey()), criteria.getSearchValue());
            }
        }
        return null;
    }
}
```
SearchCriteria.class
``` java
// BoardSpecification 객체 선언 시 사용되는 인자로 전달 하기 위한 클래스
@Data
@AllArgsConstructor
public class SearchCriteria {

    private String searchKey;
    private String searchValue;

}
```

BoardServiceImpl.class
```java
@Override
  public Page<BoardDTO> boardList(Pageable pageable, Map<String, String> params){

    // SearchCriteria 객체에 컨트롤러로 부터 받은 params를 인자로 주고 BoardSpecification 객체를 SearchCriteria 객체를 인자로 주어 생성
    Specification<BoardEntity> specification = new BoardSpecification(new SearchCriteria(params.get("searchKey"), params.get("searchValue")));
    
    // Pageable로 전달 받은 값과, 현제 페이지 값을 PageRequest으 매개변수로 담아 리포지토리로 넘겨주면 페이징 처리된 결과가 리턴된다.
    Page<BoardEntity> boardEntities = boardRepository.findAll(specification, PageRequest.of(page, pageable.getPageSize(), pageable.getSort()));

    Page<BoardDTO> boardDTOList = boardEntities.map(board -> new BoardDTO(board.getId(), board.getBoardWriter(), board.getBoardTitle(), board.getBoardHits(), board.getCreatedTime()));

    return boardDTOList;

  }
```
```java
// 리포지토리 인터페이스에서 JpaSpecificationExecutor를 상속받고 쿼리 메서드를 요청하면 검색 대상 필드와 검색어가 적용된 결과가 리턴 된다.
public interface BoardRepository extends JpaRepository<BoardEntity, Long>, JpaSpecificationExecutor<BoardEntity> {
  @Modifying
  @Query(value = "update BoardEntity b set b.boardHits=b.boardHits+1 where b.id=:id")
  void updateHits(@Param("id") Long id);

  @Modifying
  @Query(value = "update BoardEntity b set b.fileAttached=0 where b.id=:id")
  void updatefileAttached(@Param("id") Long id);
}
```


- Pageable 적용 결과

![Image](https://github.com/user-attachments/assets/96e13ac1-45e1-49b5-a4e4-23580ab85536)
```json
{
    "content": [
      ...
        {
            "id": 2,
            "boardWriter": "111",
            "boardTitle": "aaa",
            "boardContents": null,
            "boardHits": 0,
            "boardCreatedTime": "2025-06-04T12:05:46.316532",
            "boardUpdatedTime": null,
            "fileList": null,
            "originalFileName": null,
            "storedFileName": null,
            "fileAttached": 0,
            "boardFileDTO": null
        }
    ],
    // 페이징 데이터도 함께 리턴 된다.
    "pageable": {
        "pageNumber": 0,
        "pageSize": 2,
        "sort": {
            "empty": true,
            "unsorted": true,
            "sorted": false
        },
        "offset": 0,
        "paged": true,
        "unpaged": false
    },
    "last": false,
    "totalElements": 12,
    "totalPages": 6,
    "first": true,
    "size": 2,
    "number": 0,
    "sort": {
        "empty": true,
        "unsorted": true,
        "sorted": false
    },
    "numberOfElements": 2,
    "empty": false
}
```

- Specification 적용 결과

![Image](https://github.com/user-attachments/assets/6482f22f-2db9-4d63-920f-502610814933)
```json
{
            "id": 2,
            "boardWriter": "111",
            "boardTitle": "aaa",
            "boardContents": null,
            "boardHits": 0,
            "boardCreatedTime": "2025-06-04T12:05:46.316532",
            "boardUpdatedTime": null,
            "fileList": null,
            "originalFileName": null,
            "storedFileName": null,
            "fileAttached": 0,
            "boardFileDTO": null
        },
        {
            "id": 7,
            "boardWriter": "testid1",
            "boardTitle": "aaa",
            "boardContents": null,
            "boardHits": 0,
            "boardCreatedTime": "2025-06-09T16:43:19.596313",
            "boardUpdatedTime": null,
            "fileList": null,
            "originalFileName": null,
            "storedFileName": null,
            "fileAttached": 0,
            "boardFileDTO": null
        },
...
```

- Pageable, Specification은 혼합이 가능하다.

![Image](https://github.com/user-attachments/assets/6d213532-952d-42a4-bea1-da49c16a66fc)
```json
{
    "content": [
...
        {
            "id": 9,
            "boardWriter": "testid1",
            "boardTitle": "acaaa",
            "boardContents": null,
            "boardHits": 0,
            "boardCreatedTime": "2025-06-09T16:43:29.665184",
            "boardUpdatedTime": null,
            "fileList": null,
            "originalFileName": null,
            "storedFileName": null,
            "fileAttached": 0,
            "boardFileDTO": null
        }
    ],
    "pageable": {
        "pageNumber": 1,
        "pageSize": 2,
        "sort": {
            "empty": true,
            "unsorted": true,
            "sorted": false
        },
        "offset": 2,
        "paged": true,
        "unpaged": false
    },
    "last": true,
    "totalElements": 4,
    "totalPages": 2,
    "first": false,
    "size": 2,
    "number": 1,
    "sort": {
        "empty": true,
        "unsorted": true,
        "sorted": false
    },
    "numberOfElements": 2,
    "empty": false
}
```

### 3.2 게시판 특이사항 - 첨부파일 처리
게시판에는 첨부파일을 첨부하고 처리하는 기능을 구현 하였다.

- RestBoardController.class
```java
@PostMapping("/boardSave")
  // 사용자 화면에서 multipart/form-data 형식의 form 데이터가 전송 될 것이므로 MultipartFile[] boardFile 형식의 데이터를 RequestParam 받아 주어야 한다.
    public ResponseEntity<BoardPostResponse> boardSave(@RequestParam("boardTitle") String boardTitle, @RequestParam("boardWriter") String boardWriter, @RequestParam("boardContents") String boardContents, @RequestParam(name="boardFile", required = false) MultipartFile[] boardFile) throws IOException {
        BoardDTO boardDTO = new BoardDTO();
        boardDTO.setBoardTitle(boardTitle);
        boardDTO.setBoardWriter(boardWriter);
        boardDTO.setBoardContents(boardContents);
        boardDTO.setFileList(boardFile);
        boardService.boardSaveAtta(boardDTO);

        return ResponseEntity.ok(BoardPostResponse
                .builder()
                .resultMessage("save success")
                .resultCode("200")
                .id(1L)
                .build());
    }
```

- BoardServiceImpl.class
```java
@Override
  public BoardDTO boardSaveAtta(BoardDTO boardDTO) throws IOException {
    // 게시글 저장 시 첨부파일 존재 여부에 따라 분기하여 처리 한다.
    if (boardDTO.getFileList() == null) {
    ... 중략
    } else {
      // 첨부파일 존재 시 게시판 테이블의 파일 첨부여부를 true로 insert 한다.
      boardDTO.setFileAttached(1);
      BoardEntity saveBoardEntity = BoardEntity.toSaveEntity(boardDTO);
      BoardEntity boardEntitys = boardRepository.save(saveBoardEntity);
      Long savedId = boardRepository.save(saveBoardEntity).getId();
      BoardEntity board = boardRepository.findById(savedId).get();

      if(boardDTO.getFileList().length > 0) {
        for (MultipartFile boardFile : boardDTO.getFileList()) {
          String originalFilename = boardFile.getOriginalFilename();
          String storedFileName = System.currentTimeMillis() + "_" + originalFilename;
          String savePath = "..." + storedFileName;
          String mimeType = boardFile.getContentType().substring(0, boardFile.getContentType().indexOf("/"));
          boardFile.transferTo(new File(savePath));
          BoardFileEntity boardFileEntity = BoardFileEntity.toBoardFileEntity(board, originalFilename, storedFileName, mimeType);
          // 첨부파일 존재 시 리스트를 저장 한다.
          boardFileRepository.save(boardFileEntity);
        }
      }
      ...
  }
```
게시글의 첨부파일 삭제 시 해당 게시글의 첨부파일 존재 여부를 확인하고 모든 첨부 파일 삭제 시 파일 첨부여부를 false로 업데이트 하여 사용자 화면의 첨부파일 리스트 노출 여부를 결정 할 수 있도록 하였다. 
- RestBoardController.class
```java
// 파일 삭제 처리 컨트롤러
@GetMapping("/fileDelete/{fileId}&{boardId}")
    public List<BoardFileDTO> fileDelete(@PathVariable Long fileId, @PathVariable Long boardId) {
        List<BoardFileDTO> boardFileDTOList = boardService.fileDelete(fileId, boardId);

        return boardFileDTOList;
    }
```

- BoardServiceImpl.class
```java
@Override
  @Transactional
  public List<BoardFileDTO> fileDelete(Long fileId, Long boardId) {
    boardFileRepository.deleteById(fileId);

    // 게시판 아이디로 첨부 파일 테이블 조회
    List<BoardFileEntity> boardFileEntityList = boardFileRepository.findByBoardId(boardId);

    ModelMapper mapper = new ModelMapper();
    List<BoardFileDTO> fileDTOList = mapper.map(boardFileEntityList, new TypeToken<List<BoardFileDTO>>() {
    }.getType());

    if(boardFileEntityList.size() == 0)
    {
      // 해당 게시글의 첨부파일이 없다면 첨부 파일 존재 여부 false로 업데이트
      boardRepository.updatefileAttached(boardId);
    }

    return fileDTOList;
  }
```

- 첨부파일이 있는 게시글 데이터 및 조회 화면

![Image](https://github.com/user-attachments/assets/df69a95e-a6fe-484e-b440-4ffd5753b97f)

![Image](https://github.com/user-attachments/assets/406b7a8d-919d-43ba-9e04-f3b0fb6253fd)

## 4. 코멘트
코멘트도 기본적인 CRUD 기능을 구현 하였으며 게시판과 같이 Pageable 인터페이스를 통해 페이징 처리를 하였으며 코멘트의 경우 답글 기능으로 자기 참조 관계 설정 및 리스트 트리를 구현 하였다. 

### 4.1 코멘트 - 리스트 트리

![Image](https://github.com/user-attachments/assets/bd8e617c-1ffa-48d2-9d44-435d81091a3e)

코멘트는 부모 아이디가 없는 루트 코멘트가 있으며 루트 코멘트에 답글 달기 기능으로 차일드 코멘트가 발생되며 차일드 코멘트 역시 답글이 달리게 되어 부모 코멘트가 되며 해당 답글은 같은 루트 코멘트를 가진 차일드 코멘트가 된다.

![Image](https://github.com/user-attachments/assets/6c2f6080-a7b5-484d-aab3-9acf170563be)

CommentEntity.class
```java
  // 코멘트의 엔티티 관계 매팅 루트 코멘트와 부모 코멘트는 여러 자식 코멘트를 가지게 된다.
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "parent_comment_id")
  //@JsonIgnore 어노테이션으로 직렬화에서 제외
  @JsonIgnore
  private CommentEntity parentCommentEntity;


  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "root_comment_id")
  @JsonIgnore
  private CommentEntity rootCommentEntity;

  // @Transient 어노테이션을 통해 영속 대상에서 제외
  @Transient
  public final List<CommentEntity> childrenComments = new ArrayList<CommentEntity>();

```

RestCommentController.class
```java

    // 부모 코멘트가 없는 코멘트인 루트 코멘트 리스트를 가져오고
    BoardEntity boardEntity = boardRepository.findById(boardId).get();
    Page<CommentEntity> rootCommentEntity = commentRepository.findByBoardEntityAndParentCommentEntityIsNull(boardEntity, PageRequest.of(page, pageable.getPageSize()));


    // 가져온 루트 코멘트 리스트들의 자식 코멘트를 가져온다
    List<Long> rootCommentIds = rootCommentEntity.stream().map(CommentEntity::getId).collect(Collectors.toList());
    List<CommentEntity> subComments = commentRepository.findAllSubCommentEntitysInRoot(rootCommentIds);

    // 자식 코멘트 리스트들을 반복문을 돌며 부모와 자식 코멘트의 자식 코멘트를 가져온다.
    subComments.forEach(subComment -> {
        subComment.getParentCommentEntity().getChildrenComments().add(subComment); // no
    });

```
코멘트 엔티티들의 리스트들을 반복문을 통해 처리 하면 아래와 같은 트리 구조의 JSON 데이터를 리턴 받을 수 있으며 리턴 받은 코멘트 리스트를 사용자 화면에서 재귀적으로 처리하여 표현 할 수 있도록 구현 하였다.

```json
"createdTime": "2025-06-10T09:26:10.76783",
            "updatedTime": null,
            "id": 8,
            "commentWriter": "testid1",
            "commentContents": "11",
            "childrenComments": [
                {
                    "createdTime": "2025-06-10T09:26:33.413461",
                    "updatedTime": null,
                    "id": 11,
                    "commentWriter": "testid2",
                    "commentContents": "4444",
                    "childrenComments": [
                        {
                            "createdTime": "2025-06-10T09:26:41.891861",
                            "updatedTime": null,
                            "id": 12,
                            "commentWriter": "testid1",
                            "commentContents": "5555",
                            "childrenComments": []
                        }
                    ]
                },
                {
                    "createdTime": "2025-06-10T09:26:53.023516",
                    "updatedTime": null,
                    "id": 14,
                    "commentWriter": "testid1",
                    "commentContents": "7777",
                    "childrenComments": []
                }
            ]
        },
        {
            "createdTime": "2025-06-10T09:26:25.226668",
            "updatedTime": null,
            "id": 9,
            "commentWriter": "testid2",
            "commentContents": "222",
            "childrenComments": [
                {
                    "createdTime": "2025-06-10T09:26:46.694357",
                    "updatedTime": null,
                    "id": 13,
                    "commentWriter": "testid1",
                    "commentContents": "6666666",
                    "childrenComments": []
                }
            ]
        },
        {
            "createdTime": "2025-06-10T09:26:28.957493",
            "updatedTime": null,
            "id": 10,
            "commentWriter": "testid2",
            "commentContents": "333",
            "childrenComments": []
        }
```

- 코멘트 사용자 화면

![Image](https://github.com/user-attachments/assets/fc42b747-e65f-43d6-9069-d35d87f4478d)

## 5. 예약
예약 역시 기본적인 CRUD 기능을 구현 하였으며 예약의 경우 1개 예약에 여러 예약 시간이 존재 할 수 있으며 예약 시간 역시 다수의 예약에 할당 될 수 있으므로 N:N 관계가 되며 N:N 관계가 가진 이슈로 인에 중간 테이블인 reserve_time 테이블로 관리 되도록 구현 하였다.


### 5.1 예약 - 예약과 예약 시간
![Image](https://github.com/user-attachments/assets/3cfe9ac9-be13-4dd2-ab44-20720f8537db)

- ReserveTimeEntity.class
```java
ReserveTimeEntity extends BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 중간 엔티티인 예약시간 엔티티 예약과 시간을 N:1로 가지도록 매칭
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reserve_id")
    private ReserveEntity reserveEntity;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "time_id")
    private TimeEntity timeEntity;
    ...
```
- ReserveServiceImpl.class
```java
@Override
    public ReserveDTO reserveSave(ReserveDTO reserveDTO){
        Optional<UserEntity> optionalUserEntity = Optional.ofNullable(userRepository.findByLoginId(reserveDTO.getReserveUserId()));
        Optional<HallEntity> optionalHallEntity = hallRepository.findById(reserveDTO.getHallId());
        if (optionalUserEntity.isPresent() && optionalHallEntity.isPresent()) {
            UserEntity userEntity = optionalUserEntity.get();
            HallEntity hallEntity = optionalHallEntity.get();
            ReserveEntity reserveEntity = ReserveEntity.toSaveEntity(reserveDTO, userEntity, hallEntity);
            ReserveEntity reserveEntitys = reserveRepository.save(reserveEntity);

            // 예약을 저장 할때는 선택된 시간 리스트들을 반복문으로 돌며 중간 테이블인 예약시간 테이블에 저장 하게 된다.
            for(int i = 0; i < reserveDTO.getReserveTimeSave().size(); i++) {
                TimeEntity timeEntity = timeRepository.findById(reserveDTO.getReserveTimeSave() .get(i)).get();
                ReserveTimeEntity reserveTimeEntity = ReserveTimeEntity.toSaveEntity(reserveEntity, timeEntity,reserveDTO);
                reserveTimeRepository.save(reserveTimeEntity);
            }
            ModelMapper mapper = new ModelMapper();
            ReserveDTO reserveDTO1  = mapper.map(reserveEntitys, new TypeToken<ReserveDTO>(){}.getType());

            return reserveDTO1;

        } else {
            return null;
        }
    }
```

- ReserveRepository.class
```java
public interface ReserveRepository extends JpaRepository<ReserveEntity, Long> {

  ...
  // 예약과 예약시간의 중간 테이블을 두고 쿼리 메소드를 통해 예약 데이터를 가져올 때 예약 시간도 가져 올 수 있도록 하였다.
    List<ReserveEntity> findByReserveDateContainingAndReserveUserIdContaining(String reserveDate, String reserveUserId);
}

```
- 예약 데이터 리턴 형태
```json
{
        "id": 6,
        "reserveReason": "test",
        "reserveDate": "20250611",
        "reservePeriod": "2",
        "userName": "testusername1",
        "reserveUserId": "testid1",
        "hallId": 1,
        "reserveTimeSave": null,
        "reserveTime": [
            {
                "id": 1,
                "reserveId": "testid1",
                "timeId": null,
                "reserveDate": "20250611",
                "time": {
                    "id": 4,
                    "time": "13",
                    "reserved": 0,
                    "reserveUserId": null
                }
            },
            {
                "id": 2,
                "reserveId": "testid1",
                "timeId": null,
                "reserveDate": "20250611",
                "time": {
                    "id": 5,
                    "time": "14",
                    "reserved": 0,
                    "reserveUserId": null
                }
            }
        ],
        "reserveCreatedTime": "2025-06-10T12:01:18.003458",
        "reserveUpdatedTime": null
    }
```
리턴된 데이터를 통해 사용자 화면에서 캘린터에 예약 리스트를 예약 시간을 포함하여 보여 줄수 있고 특정 예약 수정 시 예약된 시간과 예약 이유와 같은 예약 데이터를 수정 폼에서 보여 주어 수정 기능을 구현 할 수 있도록 하였다.

- 예약 사용자 화면

![Image](https://github.com/user-attachments/assets/da3477e0-5c6d-48c8-8760-7c52a60b647a)

## 5. 기타사항
Spring Boot와 Spring Data 라이브러리의 모듈을 통해 프로젝트를 구현 해보았으며 일반적인 세션기반 CRUD 게시판 프로젝트의 내용은 해당 문서에서 설명하지 않았으며 생성자 패턴, ORM 기반 구현 등도 사용자, 게시판, 코멘트, 예약의 내용에 중복되어 구현 되었기에 특이 사항만 기술 하였다. 프로젝트의 프론트 부분은 Next.js 기반으로 구축하여 상세한 기능에 대해서는 아래 URL을 통해 확인이 가능하다.

- GithHub Url : 
<https://github.com/SeungMinLee-88/nextjs_prj>