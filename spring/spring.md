# - 프로젝트 개요
예전 PHP 프레임워크인 Laravel을 통해 개발한 회의실 예약 프로젝트의 일부 기능을 프론트 영역은 react와 react 프레임워크인 Next.js, 백엔드 영역은 Spring Boot, Spring Data JPA, Spring Security 등을 통해 구현해 보았으며 해당 문서에서는 백엔드 부분에 대해 중점적으로 다루었다.
<br /><br />
# - 개발기간
- 25.04 ~ 25.05(약 1.5개월)\
<br /><br />
# - 개발환경
- JAVA v1.8
- Spring Boot, Spring Data Jpa v3.4.3
- Spring Security v3.4.3
- Jsonwebtoken v0.12.3
- lombok 등 라이브러리 및 Mysql DB
<br /><br />
# - 주요기능
- 사용자인증
Spring Security 통한 사용자 인증 및 권한 제어, JWT 인증 토큰 및 refresh 토큰 발급, 재발급 기능
- 게시판 :\
게시판 CRUD 기능, Pageable 인터페이스를 통한 페이징 처리, Specification을 통한 검색 기능, 첨부 파일 처리
- 코멘트 :\
코멘트 CRUD 기능, 코멘트 트리 리스트 구현
- 예약 :\
예약 CRUD 기능
<br /><br />
# - 특이사항
- Spring Security를 통한 인증 처리, 권한 제어 JWT 인증 토큰 발급/재발급 기능을 통한 사용자 접근 제어
- Pageble, Specification 인터페이스를 통한 쿼리 리스트 조회 처리
- 코멘트에 자기 참조 관계 설정 및 리스트 트리 구현
- N:N 관계 처리를 위해 중간 테이블 추가 및 데이터 처리

## 1. DB구조
### 1.2 사용자
사용자는 여러개의 Role을 가질 수 있고 Role 역시 여러 사용자에게 할당 될 수 있으므로 사용자와 Role은 N:N 관계이며 이를 표현 하기 위해 중간 테이블인 role_user 테이블을 두어 사용자가 추가 될 시 role_user 테이블에 사용자 아이디와 Role 아이디를 가진 데이터가 추가 되어야 한다.

![Image](https://github.com/user-attachments/assets/3ff89efb-cb24-4edf-8514-f769328fc6f7)

### 1.2 게시판
게시판은 게시글을 작성하는 사용자와 N:1, 게시글에 첨부되는 첨부파일과 1:N, 게시글의 코멘트와 1:N 관계이다. 코멘트의 경우 Reply 기능으로 자기 참조 데이터가 생성되므로 자기자신을 1:N으로 참조 하게 된다.

![Image](https://github.com/user-attachments/assets/f08b4e4f-d24a-4b90-baa9-914944126c7c)

### 1.3 예약
사용자는 여러 예약을 가질 수 있으므로 사용자와 예약은 1:N 관계이며 예약은 여러개의 예약시간을 가질 수 있고 예약시간 역시 여러 예약에 할당 될 수 있으 예약과 예약시간 N:N 관계이며 이를 표현 하기 위해 중간 테이블인 reserve_time 테이블을 두어 예약이 추가 될 시 reserve_time 테이블에 예약 아이디와 예약시간 아이디를 가진 데이터가 추가 되어야 한다.

![Image](https://github.com/user-attachments/assets/283df42e-6218-4ca5-a86a-8a8dfd5828d3)


## 2. 프로젝트 기본 구조
### 2.1 생성자 패턴
프로젝트는 특정한 형태의 생성자가 필요한 경우 직접 생성자를 선언하거나 Lombok 어노테이션을 사용하여 빌터 패턴을 사용하여 구현 하였다.

- BoardDTO.class
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

- BoardEntity.class
```java
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Table(name = "board")
public class BoardEntity extends BaseEntity {
        ... 중략
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
- BoardEntity.class
```java
public class ReserveEntity extends BaseEntity {
... 중략
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
  ... 생략

```

### 2.3 JpaRepository 상속 구현
각 리포지토리 인터페이스는 JpaRepository를 상속받아 기본 쿼리 메서드를 통해 테이블에 접근하며 인터페이스에 쿼리 메서드를 정의, @Query 어노테이션 사용하여 직접 쿼리 선언하는 방식 등을 통해 구현해 보았다.

- UserServiceImpl.class
```java
// UserRepository 인터페이스 JpaRepository 상속 받아 기본 제공 메서드를 사용
userRepository.findById(userDto.getId());

```
- UserRepository.class
```java

public interface UserRepository extends JpaRepository<UserEntity, Long>, JpaSpecificationExecutor<UserEntity> {
        
// JPA에서 제공되는 쿼리 메서드를 통해 사용자 정보 조회
  Boolean existsByLoginId(String loginId);

  UserEntity findByLoginId(String loginId);

  UserEntity findByLoginIdAndUserPassword(String loginId, String userPassword);

// @Query 어노테이션을 통해 엔티티에 대한 쿼리를 직접 선언
  @Query("SELECT u FROM UserEntity u LEFT JOIN FETCH u.roleUserEntities")
  Page<UserEntity> findAllWithRelated(Pageable pageable);

  @Query("SELECT u FROM UserEntity u")
  Page<UserEntity> findAllWithPageble(Specification specification, Pageable pageable);
}
```
BoardRepository.class
```java
public interface BoardRepository extends JpaRepository<BoardEntity, Long>, JpaSpecificationExecutor<BoardEntity> {
// @Modifying 엔티티를 직접 접근하여 수정하는 방식도 사용해 보았다.
  @Modifying
  @Query(value = "update BoardEntity b set b.boardHits=b.boardHits+1 where b.id=:id")
  void updateHits(@Param("id") Long id);

  @Modifying
  @Query(value = "update BoardEntity b set b.fileAttached=0 where b.id=:id")
  void updatefileAttached(@Param("id") Long id);
}
```
이외 @Transactional 어노테이션을 통해 메서드 레벨 트랜잭션 관리 등을 구현 해보았으며 Spring Data JPA를 통해 JPA 기반으로 데이터 액세스 계층을 추상화하며 ORM 기반한 구조로 프로젝트를 구현하였다.
(Spring Data JPA가 디폴트 구현체로 Hibernate를 제공하기에 구현체는 Hibernate를 사용 하였다.)


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
        
  ...중략
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
  
  ...중략
  
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

        // UsernamePasswordAuthenticationFilter 커스텀 LoginFilter로 교체
    http
            .addFilterAt(new LoginFilter(authenticationManager(bCryptPasswordEncoder()), jwtUtil, refreshRepository, roleUserRepository, userRepository), UsernamePasswordAuthenticationFilter.class);
            
        // 로그아웃 필터 호출 전 CustomLogoutFilter 호츌
    http
            .addFilterBefore(new CustomLogoutFilter(jwtUtil, refreshRepository), LogoutFilter.class);
    ...생략
```
Security 라이브러리를 사용하기 위해 SecurityConfig 클래스를 만들고 인증이 필요한 페이지와 인증 후에도 특정 권한이 필요한 페이지를 추가 해주었다.
로그인을 위해 UsernamePasswordAuthenticationFilter 상속받는 커스텀 LoginFilter를 추가해 주었다.

### 2.2 사용자 인증 및 토큰 발급
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
  ...중략
  
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
// authenticationManager를 인증을 시도 하면 앞서 SecurityConfig 에서 authenticationProvider.setUserDetailsService(customUserDetailsService) 부분을 통해 customUserDetailsService를 등록 해두었으니 CustomUserDetailsService.class를 호출하게 되고 사용자를 인증 처리 한다.
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
// 인증이 성공하면 AbstractAuthenticationProcessingFilter의 successfulAuthentication를 재정의 하여 JWT 토큰을 반환 한다.

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
    response.setHeader("Access-Control-Expose-Headers", "userName, access" );
    response.setHeader("access", access );
    response.setHeader("userName", userName );
    response.addCookie(createCookie("refresh", refresh));
    response.setStatus(HttpStatus.OK.value());
  }

```

### 2.3 인증 사용자 처리
인증 성공 시 액세스 토큰과 리프레시 토큰을 발급하여 헤더에 인증 정보와 쿠키로 리프레시 토큰을 리턴한다. 인증 토큰을 가진 사용자가 페이지 접근 시 addFilterBefore(new JWTFilter(jwtUtil), LoginFilter.class) 부분을 통해 JWTFilter 필터를 통해 인증 여부를 확인 한다.

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
    ... 중략
    
    // 유효기간 이내의 액세스 토큰이라면 토큰에서 사용자 정보와 권한 정보를 가져온다.
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
- 권한이 있는 사용자

- 권한이 없는 사용자

### 2.3 토큰 재발급
사용자 인증 액세스 토큰이 만료될 경우 리프레시 토큰을 통해 토큰을 재발급 받을 수 있는 기능도 구현 해보았다. 서버의 경우 리프레시을 DB 상에 저장 되도록 구현 하였다.
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
        ... 중략
        
        // 요청된 리프레시 토큰이 서버에 존재하는지 확인
         Boolean isExist = refreshRepository.existsByRefresh(refresh);
        if (!isExist) {
            return new ResponseEntity<>("invalid refresh token", HttpStatus.BAD_REQUEST);
        }
        
        String username = jwtUtil.getUsername(refresh);
        List<String> role = jwtUtil.getRole(refresh);

// 새로운 액세스 토큰과 리프레시 토큰을 발급
        String newAccess = jwtUtil.createJwt("access", username, role, 600000L);
        String newRefresh = jwtUtil.createJwt("refresh", username, role, 86400000L);

        refreshRepository.deleteByRefresh(refresh);
        addRefreshEntity(username, newRefresh, 86400000L);

        response.setHeader("access", newAccess);
        response.addCookie(createCookie("refresh", newRefresh));

        return new ResponseEntity<>(HttpStatus.OK);
    }
```
- 토큰 만료 시
- 리프레시 토큰
- 토큰 재발급

## 3. 게시판
게시판은 기본적인 CRUD 기능을 구현 하였으며 게시판 구현 시 특이사항에 대해서만 설명 하겠다.

### 3.1 게시판 - Pageable, Specification
게시판의 페이징과 검색은 Pageable, Specification 인터페이스를 통해 구현 하였다. 

- BoardSpecification.class
```java
@AllArgsConstructor
// Specification을 구현 한다. 레퍼런스 문서를 참고하여 구현한다
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
```java
@Override
  public Page<BoardDTO> boardList(Pageable pageable, Map<String, String> params){

    // 컨트롤러에서 Pageable과 검색 Rarams를 받아 서비스 검색 파라미터들은 Specification를 BoardSpecification 객체를 만들고 

    Specification<BoardEntity> specification = new BoardSpecification(new SearchCriteria(params.get("searchKey"), params.get("searchValue")));
    
    // Pageable과 값들은 필요한 PageRequest으 매개변수로 담아 리포지토리로 넘겨주면 페이징 처리된 결과가 리턴된다.
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
```json
{
    "content": [
      ...중략
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
...생략
```

- Pageable, Specification 혼합이 가능하다.
```json
{
    "content": [
...중략
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
          // 첨부파일 리스트를 저장 처리 한다.
          boardFileRepository.save(boardFileEntity);
        }
      }
      ...생략
  }
```
게시글의 첨부파일 삭제 시 해당 게시글의 첨부파일 존재 여부를 확인하고 모든 첨부 파일 삭제 시 파일 첨부여부를 false로 업데이트 하여 사용자 화면의 첨부리스트 노출 여부를 결정 할 수 있도록 하였다. 
- RestBoardController.class
```java
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

## 4. 코멘트
코멘트도 기본적인 CRUD 기능을 구현 하였으며 코멘트의 


### 3.1 게시판 특이사항 - Pageable, Specification


## 5. 결론 및 향후 계획
JavaScript 라이브러리인 react와 react 기반 프레임워크인 nextjs를 통해 예전에 진행했던 예약 플젝트의 일부를 구현 해보았다. 이번 프로젝트는 react와 nextjs를 처음 접해보고 사용기에 Pages Router를 통해 구현 하였으며