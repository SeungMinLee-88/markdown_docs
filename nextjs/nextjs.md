# - 프로젝트 개요
예전 PHP 프레임워크인 Laravel을 통해 개발한 회의실 예약 프로젝트의 일부 기능을 프로트 영역은 react와 react 프레임워크인 nextjs, 백엔드 영역은 Spring Boot, Spring Data JPA, Spring Security 등을 통해 구현해 보았으며 해당 문서에서는 프론트 부분에 대해 중점적으로 다루었다.


# - 개발기간
- 25.04 ~ 25.05(약 1.5개월)

# - 개발환경
- node.js v18.20.5
- react v19.0.0
- nextjs v15.2.4
- Semantic UI React, Axios, FullCalendar 등 라이브러리

# - 주요기능
- 사용자인증
기본 사용자 인증, JWT 토큰 발급, 재발급localStorage와 sessionStorage를 통한 사용자 접근 제어, react Context를 통한 값 전달 처리
- 게시판
기본 게시판 CRUD 기능 및 검색, Semantic UI를통한 페이징 처리, react reducer를 통한 검색 기능, react useRef를 이용한 렌더링 제어 및 DOM 엘리먼트 처리 등
- 코멘트
코멘트 CRUD, 코멘트 트리 UI 표현
- 예약
특정일자 시간대 예약 및 수정, 사용자 권한에 따른 예약 제한,
FullCalendar 라이브러리를 통한 달력 UI 표현, react useRef를 이용한 렌더링 제어 및 DOM 엘리먼트 처리 등

# - 특이사항
- nextjs 환경에서 기본 App을 오버라이드 하여 Custom App 

## 1. 사용자 인증처리
사용자 인증 및 접근 제어는 Spring Security와 JWT 라이브러리를 통해서 구현 하였으며