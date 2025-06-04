# - 프로젝트 개요
예전 PHP 프레임워크인 Laravel을 통해 개발한 회의실 예약 프로젝트의 일부 기능을 프로트 영역은 react와 react 프레임워크인 Next.js, 백엔드 영역은 Spring Boot, Spring Data JPA, Spring Security 등을 통해 구현해 보았으며 해당 문서에서는 프론트 부분에 대해 중점적으로 다루었다.
<br /><br />
# - 개발기간
- 25.04 ~ 25.05(약 1.5개월)\
<br /><br />
# - 개발환경
- node.js v18.20.5
- react v19.0.0
- Next.js v15.2.4
- Semantic UI React, Axios, FullCalendar 등 라이브러리
<br /><br />
# - 주요기능
- 사용자인증
기본 사용자 인증, JWT 토큰 발급, 재발급, localStorage와 sessionStorage를 통한 사용자 접근 제어, react Context를 통한 값 전달 처리
- 게시판 :\
기본 게시판 CRUD 기능 및 검색, Semantic UI를통한 페이징 처리, react reducer를 통한 검색 기능, react useRef를 이용한 렌더링 제어 및 DOM 엘리먼트 처리 등
- 코멘트 :\
코멘트 CRUD 기능, 코멘트 트리 UI 표현
- 예약 :\
특정일자 시간대 예약 및 수정, 사용자 권한에 따른 예약 제한,
FullCalendar 라이브러리를 통한 달력 UI 표현, react createRef를 이용한 클래스 컴포넌트 DOM 오브젝트 처리 등
<br /><br />
# - 특이사항
- Next.js의 Pages Router를 통해 구현 하였으며 공식 문서의 경우 App Router 사용을 권장 하나 Next.js를 처음 접할 경우 Pages Router부터 사용하는 것이 추천되어 Pages Router를 통해 구현 하였으며 향후 App Router 구조로 마이그레이션 진행 예정
참고 -\
<https://dev.to/dcs-ink/nextjs-app-router-vs-pages-router-3p57>\ <https://stackoverflow.com/questions/76570208/what-is-different-between-app-router-and-pages-router-in-next-js>\
<https://www.reddit.com/r/nextjs/comments/1gdxcg5/why_do_you_still_prefer_page_router_over_app/>

- 기본 App 재정의 하여 _app.js를 통한 커스텀 앱 형태로 구현\
참고 -\
<https://www.dhiwise.com/post/the-power-of-nextjs-custom-routes-in-modern-web-development>\
<https://medium.com/@farihatulmaria/what-is-the-purpose-of-the-app-js-and-document-js-files-in-a-next-js-application-397f22fed69e>

- UI는 Semantic UI React 라이브러리를 사용하여 구현
- 백엔드 부분과 데이터 요청, 응답을 위해 Axios 라이브러리를 사용 

<!-- <details>
<summary>제목</summary> -->
<!-- </details> -->
## 1. 테이블 구조
### 1.1 사용자
### 1.1 게시판, 코멘트, 첨부파일
### 1.1 게시판, 코멘트, 첨부파일