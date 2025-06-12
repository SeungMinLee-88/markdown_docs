# - 프로젝트 개요
예전 PHP 프레임워크인 Laravel을 통해 개발한 회의실 예약 프로젝트의 일부 기능을 프론트 영역은 react와 react 프레임워크인 Next.js, 백엔드 영역은 Spring Boot, Spring Data JPA, Spring Security 등을 통해 구현해 보았으며 해당 문서에서는 프론트 부분에 대해 중점적으로 다루었다.
# - 개발기간
- 25.04 ~ 25.05(약 1.5개월)
# - 개발환경
- node.js v18.20.5
- react v19.0.0
- Next.js v15.2.4
- Semantic UI React, Axios, FullCalendar 등 라이브러리
# - 주요기능
- 사용자인증 :\
기본 사용자 인증, JWT 토큰 발급, 재발급, localStorage와 sessionStorage를 통한 사용자 정보 저장, react Context를 통한 컴포넌트 트리에 값 공유
- 게시판 :\
기본 게시판 CRUD 기능 및 검색, Semantic UI를 통한 페이징 처리, react reducer를 통한 검색 관련 state 로직 통합, react useRef를 이용한 렌더링 제어 및 DOM 엘리먼트 처리 등
- 코멘트 :\
코멘트 CRUD 기능, 코멘트 트리 UI 표현
- 예약 :\
특정일자 시간대 예약 및 수정, 사용자 권한에 따른 예약 제한,
FullCalendar 라이브러리를 통한 달력 UI 표현, react createRef를 이용한 클래스 컴포넌트 DOM 오브젝트 처리 등
<br /><br />
# - 특이사항
- Next.js의 Pages Router를 통해 구현 하였으며 공식 문서의 경우 App Router 사용을 권장 하나 Next.js를 처음 접할 경우 Pages Router를 통한 구현이 추천되어 Pages Router를 통해 구현 하였으며 향후 App Router 구조로 마이그레이션 진행 해볼 예정
- 참고 URL -\
<https://dev.to/dcs-ink/nextjs-app-router-vs-pages-router-3p57>
<https://stackoverflow.com/questions/76570208/what-is-different-between-app-router-and-pages-router-in-next-js>
<https://www.reddit.com/r/nextjs/comments/1gdxcg5/why_do_you_still_prefer_page_router_over_app/>

- 기본 App 재정의 하여 _app.js를 통한 커스텀 앱 형태로 구현
- 참고 URL -\
<https://www.dhiwise.com/post/the-power-of-nextjs-custom-routes-in-modern-web-development>\
<https://medium.com/@farihatulmaria/what-is-the-purpose-of-the-app-js-and-document-js-files-in-a-next-js-application-397f22fed69e>

- UI는 Semantic UI React 라이브러리를 사용하여 구현
- 백엔드 부분과 데이터 요청, 응답을 위해 Axios 라이브러리를 사용 

## 1. 사용자인증
### 1.1 인증처리
사용자 인증 및 접근 제어는 Spring Security와 JWT 라이브러리를 통해서 구현 하였으며 로그인 성공시 localStorage, sessionStorage에 인증과 권한 확인에 필요한 값을 저장한다.

![Image](https://github.com/user-attachments/assets/1011ad0f-a204-42e5-a304-18fc4ce52063)


login.js
```js
// Axios를 통해 사용자 인증을 요청하는 api url 호출 
await Axios.post(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/user/login`, 
          {
            loginId: loginId,
            userPassword: userPassword
          },
          {
            withCredentials: true
          }
        )
        .then(function (response) {
          //인증 성공 시 localStorage, sessionStorage에 인증 정보를 저장한다.
          if (response.headers.access) {
            localStorage.setItem("access", response.headers.access);
            window.sessionStorage.setItem("loginId", loginId);
            window.sessionStorage.setItem("userName", response.headers["username"]); 
            setAccessToken(localStorage.getItem("access"));
            setLoginUserId(loginId);
            setLoginUserName(response.headers["username"]);
          }
          alert("Login Success");
          router.push(`/`);
        })
```

### 1.2 사용자 정보 처리
로컬과 세션 스토리지에 인증 정보와 사용자 정보를 저장 후 로그인 컴포넌트에서 _app.js로 부터 전달 받은 state setter를 통해 액세스 토큰과 사용자 아이디 값을 state에 저장하고 _app.js의 useEffect를 통해 state의 변경을 감지하여 컴포넌트를 리렌더링 한다

- _app.js
```js
export default function MyApp({ Component, pageProps }) {
  const [accessToken, setAccessToken] = useState();
  const [loginUserId, setLoginUserId] = useState();
  const [loginUserName, setLoginUserName] = useState("");
  const [reissueResult, setReissueResult] = useState(false);
...
// _app.js의 useEffect를 통해 렌더링 시 state setter로 state에 값을 할당한다.
  useEffect(() => {
    setAccessToken(localStorage.getItem("access"));
    setLoginUserId(window.sessionStorage.getItem("loginId"));
    setLoginUserName(window.sessionStorage.getItem("userName"));
  }, [accessToken, loginUserId, loginUserName, reissueResult]);
```

### 1.3 react Context를 통한 자식 컴포넌트로 값 전달
useEffect를 통한 state 변경 감지 부분 추가는 공유 레이아웃 컴포넌트에서 변경된 state를 값을 사용하기 위함과 이후 react Context를 통한 값 공유 방식을 구현해 보기 위해서이다.  
(<span style="color:red">**Next.js 13 이후 App Router의 Server Component는 Context Provider를 미지원 하므로 향후 마이그레이션 시에는 Client Component를 이용해 구성해 볼 예정.**</span>)  

- UserContext.js
```js
// createContext로 Context를 상속받는 페이지에서 받을 컨텍스트들을 선언
import { createContext } from 'react';
export const UserIdContext = createContext("userIdContext");
export const UserNameContext = createContext("userNameContext");
```

- _app.js
```js
  return (
    <div style={{ width: 800, margin: "0 auto" }}>
      <UserIdContext value={loginUserId}>
        <UserNameContext value={loginUserName}>
          <Top setAccessToken={setAccessToken}
          setLoginUserId={setLoginUserId} 
          setLoginUserName={setLoginUserName} 
          accessToken={accessToken}/>
          <Component {...pageProps} 
          setAccessToken={setAccessToken} 
          setLoginUserId={setLoginUserId} 
          setLoginUserName={setLoginUserName} 
          reissueAccessToken={reissueAccessToken}/>
          <Footer />
        </UserNameContext>
      </UserIdContext>
    </div>
  );
```
Context를 선언하고 _app.js에서 Context를 provider로 하위 컴포넌트로 전달하여 다수 컴포넌트나 여러 단계를 거치는 하위 컴포넌트에서 사용자 정보를 사용할 수 있도록 하였다.

- UserContext 구조
```mermaid
flowchart TB
  subgraph _app.js
  UserContext-->UserIdContext
  UserContext-->UserNameContext
    subgraph Reserve.js
      subgraph ReserveForm.js
        UserIdContext
        UserNameContext
      end
    end
  end
  style _app.js text-align:left
```
- Context 구현 예시, ReserveForm.js
```js
import { UserIdContext } from './UserContext.js';
import { UserNameContext } from './UserContext.js';
...
const userId = useContext(UserIdContext);
const userName = useContext(UserNameContext);
```
_app.js에서 Context를 제공하여 하위 ReserveForm 컴포넌트에서 로그인된 사용자의 아이디와 이름 정보를 제공된 Context에서 가져와 사용 할 수 있다.

![Image](https://github.com/user-attachments/assets/1a6b8144-b66a-4281-92bc-848544665c5f)

### 1.4 인증토큰 재발급
로그인이 필요한 페이지에서는 사용의 인증토큰을 헤더 값으로 서버에 전달하여 인증 토크의 만료 여부를 확인 후 페이지를 보여주도록 하였다.

- 화면 페이지
```js
async function getData() {
  await Axios.get(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/reserve/reserveList`, {
      headers: {
        "Content-Type": "application/json", 
        access: localStorage.getItem("access") 
        // 인증토큰 값 헤더 데이터로 전달
      },

```

- 서버의 인증 상태 확인 부분
```java
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
```

사용자 인증 성공 시 인증 jwt 토큰과 토큰 만료 시 재발급을 위한 리프레시 토큰이 발급되며 사용자 화면에서 유효한 토큰이 요구되는 페이지를 만료된 토큰을 가지고 접근 시 서버를 통해서  401에러가 리턴되며 해당 코드 리턴 시 토큰 재발급 여부를 확인 후 재발급 되도록 구현 하였다.

- 사용자 페이지의 인증 만료 여부 및 재발급 여부 확인 부분
```js
async function getData() {
  await Axios.get(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/reserve/reserveList`, {
      headers: {
        "Content-Type": "application/json", 
        access: localStorage.getItem("access")
        // 인증토큰 값 헤더 데이터로 전달
      },
      params: {
        reserveDate: toolBarState,
        reserveUserId: userId
      },
    }
  ).then((response) => {
    
... 

.catch(async function (error) {
      console.log("error : " + error);
      // 토큰을 통한 인증 실패 시 리턴 코드 확인
      if(error.response.status === 401){
        // 토큰 만료 시 서버에서 401에러를 리턴하여 재발급 여부 확인
        if(confirm("Session is expired. Do you want Reissue?"))
          {
            setTimeout(() => console.log("after"), 3000);
            const reissueResult = await reissueAccessToken();
            if(reissueResult){
              alert("Reissue success")
            }else{
              alert("Reissue false");
              router.push(`/`);
            }
          }
          else
          {
            console.log("Reissue false")
            router.push(`/`);
          }
      }
    });
```
- 인증토큰 만료 시 토큰 재발급 여부 확인

![Image](https://github.com/user-attachments/assets/a088e33d-c9c0-4136-b814-fea65de9994b)


- _app.js의 토큰 재발급 부분
```js
async function reissueAccessToken()
  {
    let result = "";
    // 리프레시 토큰으로 인증 토큰 재발급 요청하는 api url을 호출한다.
    await Axios.post(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/user/reIssueToken` ,
      {},
      {
        withCredentials: true
        // 리프레시 토큰은 쿠키로 저장되어 있어 withCredentials을 true로 설정
      }
      )
      .then(function (response) {
        if(response.status === 200){
          // 리프레시 토큰이 유효하다면 헤더 데이터로 새로운 인증 토큰과 쿠키 데이터로 리프레시 토큰이 발급된다.
          localStorage.removeItem("access");
          localStorage.setItem("access", response.headers.access);
        }
        result = true;
      })
      .catch(function (error) {
            result = false;
      });
      return result;
  }
```
![Image](https://github.com/user-attachments/assets/1f1d07ca-827b-4430-ba38-90de112fea32)

![Image](https://github.com/user-attachments/assets/7fa061bb-ab1c-48df-8024-37b19261a1f1)


### 1.5 사용자 권한 제어
Spring Security의 권한 제어 기능을 서버상에 구현 하였으며 해당 기능 확인을 위한 사용자 권한을 확인 후 접근을 제어하는 기능을 구현 하였으며 해당 페이지에서는 권한 제어 기능만 구현하고 간략한 사용자 관리 기능은 Vuejs를 통해 별개 프로젝트로 구현 하였다.

- 서버의 SecurityConfig 클래스
```java
 http
            .authorizeHttpRequests((auth) -> auth
                    .requestMatchers(
                             "/"
                            , "/join"
                            ,"/api/v1/user/login"
                            ,"/api/v1/user/reIssueToken"
                            , "/api/v1/board/**"
                            , "/api/v1/board/detal/*"
                            , "/api/v1/comment/commentList"
                            , "/api/v1/user/userJoin"
                            , "/error").permitAll()
                    .requestMatchers("/api/v1/admin/*").hasAnyRole("ADMIN", "MANAGER")
                    // /api/v1/admin/로 시작되는 경로 접근은 ADMIN, MANAGER 권한이 있는 사용자만 접근 가능
                    .anyRequest().authenticated());

    http.addFilterBefore(new JWTFilter(jwtUtil), LoginFilter.class);
```

- COMMON, TEMP 권한이 있는 사용자가 권한이 없는 페이지 접근 시

![Image](https://github.com/user-attachments/assets/2fad2355-eada-4d6f-a96a-a9d2b24403bb)

![Image](https://github.com/user-attachments/assets/650dc53c-b3e7-45d2-8752-a562db09232a)

![Image](https://github.com/user-attachments/assets/61cb545e-e20d-4c3a-bb9a-a66d0b0820be)

- ManagerUser.js
```js
async function chkAuthor(){
      // 서버에 ADMIN, MANAGER 권한이 필요한 api url을 호출
      await Axios.get(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/admin/manageUser` ,
      {
        headers: {
            'access' : accessToken
          }
      },
        {
        withCredentials: true
        }
      )
      .then(function (response) {
      })
      .catch(function (error) {
        // COMMON, TEMP 권한은 접근이 불가 하므로 서버에서 403 에러를 리턴 하게 되며 인덱스 페이지로 리다이렉트 처리한다.
        if(error.response.status === 403){
                alert("You are not authorized");
                router.push(`/`); 
        }
      });
    }
    // ManagerUser 컴포터넌트 접근 시 마운트가 완료되면 권한을 체크하는 chkAuthor를 호출 한다.
    useEffect(() => {
        chkAuthor()
    }, []);
```
- ADMIN이나 MANAGER 권한이 있는 사용자가 페이지 접근 시

![Image](https://github.com/user-attachments/assets/0533548a-98b7-40fc-8fb3-5704ff3a9307)

정상적으로 페이지에 접근 할 수 있다.

## 2. 게시판
### 2.1 기본기능 및 페이징, 검색 기능
게시판 부분은 기본 CRUD 기능을 구현 하였으며 페이징 처리를 Semantic UI의 Pagination 컴포넌트를 통해 구현 하였다.

- 페이징 기능 동작
![Image](https://github.com/user-attachments/assets/72f8d7fe-9367-43be-96d6-b02b62397878)

```js
<Pagination
  boundaryRange={0}
  defaultActivePage={1}
  ellipsisItem={null}
  firstItem={null}
  lastItem={null}
  siblingRange={1}
  totalPages={TotalPage}
  onPageChange={(_, { activePage }) => goToPage(activePage)}
/>
// Pagination 컴포넌트에 Props 값을 설정하면 원하는 형태의 페이징 UI를 보여 줄 수 있다.
```
또한 게시판의 검색기능 구현에는 react reducer 함수를 사용해 보았다.

- 검색 기능 동작
![Image](https://github.com/user-attachments/assets/6d8c14cf-c5cf-45ed-a91e-38806bde2838)

- BoardList.js
```js
// reducer 사용하기 위해 useReducer로 선언하고 initialState로 초기 값을 설정한다.
const [state, dispatch] = React.useReducer(searchReducer, initialState);
// reducer로 관리 될 대상들
const { loading, value, searchKey } = state;
...
const timeoutRef = React.useRef()
const handleSearchChange = (e, data) => {
  clearTimeout(timeoutRef.current)
  // handleSearchChange 호출 시 START_SEARCH 타입으로 searchReducer가 호출된다.
  dispatch({ type: 'START_SEARCH', query: data.value })
  changeSearchValue(data.value);
  setCurrentPage(1);
  // 검색 요청이 완료되면 FINISH_SEARCH 타입으로 searchReducer가 호출되며 검색 필드가 비워지면 CLEAN_QUERY 타입으로 호출한다.
  timeoutRef.current = setTimeout(() => {
    if (data.value.length === 0) {
      dispatch({ type: 'CLEAN_QUERY' })
      return
    }
    dispatch({
      type: 'FINISH_SEARCH',
    })
  }, 300)
}
const handleSearchKey = (e) => {
  // 검색 대상 필드 변경 시 UPDATE_SELECTION 타입으로 searchReducer 호출
  dispatch({ type: 'UPDATE_SELECTION', query: e.target.value });
  changeSearchKey(e.target.value);
  setCurrentPage(1);
}
...
  <select
    value={searchKey}
    // 검색 대상 필드 변경 시 handleSearchKey를 호출
    onChange={handleSearchKey} style={{width: 100}}>
    <option value="boardTitle">Title</option>
    <option value="boardWriter">Writer</option>
  </select>
    
    <Search
        loading={loading}
        placeholder='Search...'
        value={value}
        // 검색 입력칸에 검색어 입력 시 handleSearchChange를 호출
        onSearchChange={handleSearchChange}
        showNoResults={false}
      />
  </div>
...
function searchReducer(state, action) {
  // searchReducer 호출되면 reducer에서 action.type에 따라 분기하여 처리한다.
  switch (action.type) {
    case 'CLEAN_QUERY':
      return initialState
    case 'START_SEARCH':
      return { ...state, loading: true, value: action.query }
    case 'FINISH_SEARCH':
      return { ...state, loading: false}
    case 'UPDATE_SELECTION':
      return { ...state, searchKey: action.query }
    default:
      throw new Error()
  }
}
const initialState = {
  loading: false,
  value: '',
  searchKey: ''
}
```
reducer를 사용하지 않았다면 3개의 state를 만들고 각 state의 변경에 대한 이벤트 핸들러 만들어 구현해야 하는데 reducer를 기능을 통해 loading ,value, searchKey 값을 업데이트하는 로직들을 통합하여 관리 가능하다.

### 2.2 첨부 파일 처리
게시판 글쓰기, 수정의 경우 게시글에 첨부 파일을 첨부 하고 이미지 표시, 다운로드 할 수 있는 기능을 추가 했으며 파일 업로드 기능에 react의 useRef를 사용하여 react가 관리하는 DOM 노드에 접근하는 기능을 간단히 구현 해보았다.

- 파일 첨부 동작
![Image](https://github.com/user-attachments/assets/ae2dc1f8-c5de-4eb6-9f34-5b62f5f92f2f)

#### 2.2.1 파일 첨부
- BoardWrite.js
```js
// file 타입 DOM노드를 제어하기 위해 useRef를 선언
const fileInputRef1 = useRef();
useEffect(() => {
  setUserName(window.sessionStorage.getItem("loginId"))
}, [fileList]);
...
const renderFileList = () => (
<div>
  <li>
    Attached File : {fileList.length}
  </li>
  <ol>
    {[...fileList].map((f, i) => (
        <li key={i}>{f.name} - {f.type}</li>
    ))}
  </ol>
</div>  )
...
<Form.Field>
/* file input의 ref 속성에 fileInputRef1을 할당한다. */
<input type="file" name='files' multiple onChange={fileChange} ref={fileInputRef1} hidden/>
/* 파일이 첨부되면 fileChange를 통해 fileList state를 변경하고 renderFileList를 통해 jsx를 리턴한다 useEffect에서 fileList가 변경되면 다시 렌더링 하게되어 화면에서 파일 리스트를 확인 할 수 있다.*/
{renderFileList()}
<button type="button"
    name = "fileBtn"
    className="ui icon left labeled button"
    labelposition="left"
    icon="file"
    /* 버튼을 클릭하면 fileInputRef1를 통해 file DOM에 접근할 수 있다.*/
    onClick={() => fileInputRef1.current.click()}
  ><i aria-hidden="true" className="file icon"></i>Choose File</button>
</Form.Field>
```
![Image](https://github.com/user-attachments/assets/f0e0e9f1-28cc-4135-9945-33dc1efc8f92)

file input을 hidden으로 숨김 처리하고 fileInputRef1 선언 후 선언한 fileInputRef1 &lt;input ref={fileInputRef1}> 처럼 어트리뷰트로 전달하여 fileInputRef1.current에서 input DOM 노드 읽게하여 fileInputRef1.current.click() 부분으로 click 이벤트를 발생 시키는 방식으로 구현하였다.

#### 2.2.2 파일 전송
- BoardWrite.js
```js
const [fileList, setFileList] = useState([]);

...
  const fileChange = e => {
    const newFiles = Array.from(e.target.files);
    setFileList(newFiles)
  };
...
          // 서버로 전송할 formData를 선언한다.
          const formData = new FormData();
          formData.append("boardTitle", boardTitle);
          formData.append("boardWriter", boardWriter);
          formData.append("boardContents", boardContents);
          // 파일을 첨부하면 formData에 boardFile을 추가하여 서버에 전송 한다.
          if(fileList.length === 0) {
          }else{
          fileList.forEach((fileList) => {
            formData.append('boardFile', fileList);
           });
          }
...
await Axios.post(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/board/boardSave`,
            formData,
            {
              headers: {
                'Content-Type': 'multipart/form-data',
                'access' : accessToken
              }
            }
          )
```
react 렌더링한 요소를 서버로 전송할 경우 기존 html 양식 처럼 form을 submit 하는 형태가 아니기에 FormData 객체를 선언 후 전송할 필드와 데이터를 append 후 POST 요청으로 첨부 파일을 포함하여 데이터를 전송 하도록 구현 하였다.

#### 2.2.3 상세 보기의 첨부파일
- /board/detail/[id].js
```js
useEffect(() => {
  if(board["fileAttached"] === 1){
      setFileList(board["boardFileDTO"]);
      // filter 함수를 통해 기존 state의 복사본을 생성하여 imageFileList에 할당한다.
      setImageFileList(fileList.filter(a => a.mimeType === "image"));
    }
}, [fileList]);
...
<p>
{board.boardContents}
</p>
// imageFileList에 이미지 형식 파일 데이터만 존재하여 상세보기 화면에서 이미지 파일들만 미리보기를 보여 줄 수 있다.
{imageFileList.map((imageFiles) => (
  <div key={imageFiles.id}>
    <img src={`${process.env.NEXT_PUBLIC_API_URL}/api/v1/board/download/`+imageFiles.storedFileName} className="ui medium bordered image"/>                     
  </div>
  ))}
<List bulleted horizontal link>
  <ListItem active>Attached | </ListItem>
  // 모든 첨부 파일은 다운로드 요청을 할 수 있도록 구현 하였다.
    {fileList.map((files) => (
        
        <a key={files.id} role="listitem" id={files.id} className="item"  href={`${process.env.NEXT_PUBLIC_API_URL}/api/v1/board/download/`+files.storedFileName} target="_blank">{files.originalFileName}{files.type}</a>                   
      
      ))}
</List>
```

BoardServiceImpl.class
```java
@Override
  public Resource fetchFileAsResource(String fileName) throws FileNotFoundException {
    Path UPLOAD_PATH;
    try {
        UPLOAD_PATH = Paths.get("file path...");
        // 첨부파일 처리를 위해 UrlResource 클래스를 선언 후 filePath를 할당 후 return하여 처리
        Path filePath = UPLOAD_PATH.resolve(fileName).normalize();
        Resource resource = new UrlResource(filePath.toUri());
      if (resource.exists()) {
        return resource;
      } else {
        throw new FileNotFoundException("File not found " + fileName);
      }
    } catch (MalformedURLException ex) {
      throw new FileNotFoundException("File not found " + fileName);
    }
  }
```
- 게시판 상세 보기 동작
![Image](https://github.com/user-attachments/assets/fdd102f9-1d0f-4a23-bce1-f4fbefa848d4)

상세보기에서 첨부된 파일의 타입을 체크하여 이미지일 경우 화면상에 보여 줄수 있도록 state를 만들어 react의 filter 함수를 통해 새로운 새로운 배열을 만들어 할당 할 수 있도록 하였다.

#### 2.2.4 게시글 수정시 첨부파일 처리
- /board/update/[id].js
```js
...
// 게시글 수정또한 파일을 첨부하면 formData에 boardFile을 추가하여 서버에 전송 한다.
if(fileUpdateList.length === 0) {
  }else{
    fileUpdateList.forEach((fileUpdate) => {
    formData.append('boardFile', fileUpdate);
    });
  }
  await Axios.post(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/board/updateBoard`,
    formData,
    {
      headers: 
      {
        'Content-Type': 'multipart/form-data' 
      }
    })
...
// fileDelete는 파일 삭제를 요청하는 api url에 삭제대상 파일 아이디를 파라미터 값으로 요청한다.
const fileDelete = async function (fileId, boardId) {
    if(window.confirm('Delete attached file?')){
      await Axios.get(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/board/fileDelete/${fileId}&${boardId}`, {
        headers: {
          "Content-Type": "application/json", 
          access: localStorage.getItem("access") 
        },
        // params는 URL 파라미터
        params: {
          fileId: fileId,
          boardId: boardId
        },
      }
    ).then((response) => {

      setFileList(response.data);
      alert("Delete Success");
      router.refresh();

    }).catch(function (error) {
      console.log("error", error);
    });
    };
  };
  ...
<div>
<div role="list" className="ui bulleted horizontal link list">
  <ListItem active>Attached | </ListItem>
  {fileList.map((files) => (
  <div key={files.id} id={files.id} role="listitem" className="item"  href={"http://localhost:8090/api/v1/board/download/"+files.storedFileName} target="_blank">{files.originalFileName}
  // 수정페이지 에서 파일 삭제 버튼을 클릭 시 fileDelete를 호출한다.
    <i id={files.id} aria-hidden="true" className="delete icon" style={{hover: "background-color: #ff0000"}} onClick={() => fileDelete(files.id, files.boardId)}></i>
  </div>
  ))}
  </div>
</div>
```

BoardServiceImpl.class
```java
@Transactional
public List<BoardFileDTO> fileDelete(Long fileId, Long boardId) {
  boardFileRepository.deleteById(fileId);
  // 요청을 받은 서버는 삭제대상 아이디의 첨부 파일을 삭제하고
  List<BoardFileEntity> boardFileEntityList = boardFileRepository.findByBoardId(boardId);

  ModelMapper mapper = new ModelMapper();
  List<BoardFileDTO> fileDTOList = mapper.map(boardFileEntityList, new TypeToken<List<BoardFileDTO>>() {
  }.getType());

  if(boardFileEntityList.size() == 0)
  {
    // 첨부된 파일이 없는지 확인하여 게시글의 첨부 여부를 업데이트 한다.
    boardRepository.updatefileAttached(boardId);
  }

  return fileDTOList;
}
```
![Image](https://github.com/user-attachments/assets/a03cf549-8680-4709-b89f-1755674c431f)

게시판의 수정또한 신규로 첨부되는 파일은 FormData 객체에 append하여 처리 되도록 구현 하였고 게시글의 모든 첨부 파일이 삭제되면 게시글의 파일 첨부여부를 false로 업데이트 되도록 하였다.


### 2.3 동적 라우팅을 통한 접근
![Image](https://github.com/user-attachments/assets/f03e5b8f-224e-4c67-add9-74df4f0be14d)

![Image](https://github.com/user-attachments/assets/abf91a2a-88ce-41ab-b1f2-29d1f9ad2175)

게시판의 상세보기와 수정 페이지는 Next.js의 동적 라우트로 생성 하여 pages/blog/[slug].js url 형태로 접속 가능 하도록 하였다.

- /board/detail/[id].js
```js
...
// getStaticPaths에서 fallback을 true로 설정하면 Loading 페이지를 보여지도록 할 수 있다.
  if (router.isFallback) {
    return (
      <div style={{ padding: "100px 0" }}>
        <Loader active inline="centered">
          Loading
        </Loader>
      </div>
    );
  }
...

export async function getStaticPaths() {
  //getStaticPaths를 통해 사전 렌더링이 필요한 경로를 지정
  const apiUrl =  `${process.env.NEXT_PUBLIC_API_URL}/api/v1/board/list`;
  const res = await Axios.get(apiUrl);
  const data = res.data;
  return {
    // 리턴 시 getStaticProps로 paths 값을 넘긴다.
    paths: data.slice(0, 50).map((item) => ({
      params: {
        id: item.id.toString(),
      },
    })),
    // 그리고 fallback 값에따라 사전 렌더링 범위가 아닌 페이지 요청 시의 getStaticProps의 동작을 지정 할 수 있다.(404 리턴 등)
    fallback: true,
  };
}

export async function getStaticProps(context) {
  const id = context.params.id;
  const apiUrl = `${process.env.NEXT_PUBLIC_API_URL}/api/v1/board/detail/${id}`;
  const res = await Axios.get(apiUrl);
  const data = res.data;
  return {
    props: {
      board: data,
      id: id
    },
  };
}
```
getStaticPaths를 통해 동적 라우트를 사용하는 페이지를 정적으로 사전 렌더링 처리 하게되면 
next build시에 데이터를 가져와 Static Page를 미리 생성하는것을 볼 수 있다.

![Image](https://github.com/user-attachments/assets/f361ae7a-294d-42b5-afa6-809d7cf0bc51)


## 3. 코멘트
### 3.1 페이징
코멘트의 경우 로그인 시 코멘트 입력 폼을 볼 수 있도록 하였고 페이징은 게시판의 페이징과 동일한 방식으로 Pagination 컴포넌트를 통해 구현 하였다.
또한 자신이 작성한 코멘트일 경우에만 수정 삭제가 가능 하며 다른 사용자가 작성한 코멘트에는 덧글 달기가 가능 하도록 하였다.

- CommentList.js
```js
// 로그인된 사용자의 정보와 코멘트의 작성자 정보를 비교하여 버튼을 노출 시킨다.
{userId === commentList["commentWriter"] && <CommentAction commentid={commentList["id"]} onClick={addEdit}>Edit</CommentAction>}
{userId === commentList["commentWriter"] && <CommentAction commentid={commentList["id"]} onClick={addDelete}>Delete</CommentAction>}
...
<div>
  <span>Comments</span>
  <Divider />
  {userId !== null &&
// 로그인 사용자의 정보가 존재 하여야 코멘트 작성 폼이 노출된다.
<Form onSubmit={addFormSubmit} reply>
  <FormField name='commentContents' label='Comments' as="" control='textarea' rows='3' />
  <button type="submit" className="ui icon primary left labeled button" color="blue">
  <i aria-hidden="true" className="edit icon"></i>
  Add Comment
  </button>
</Form>
...

<Pagination
  activePage={currentPage}
  boundaryRange={0}
  ellipsisItem={null}
  firstItem={null}
  lastItem={null}
  siblingRange={1}
  totalPages={totalPage}
  onPageChange={(_, { activePage }) => goToPage(activePage)}
  // 게시판과 마찬가지로 Pagination 컴포넌트에 Props 값을 설정하면 원하는 형태의 페이징 UI를 보여 줄 수 있다.
  
/>

```
- 코멘트 페이징 동작 및 입력 폼

![Image](https://github.com/user-attachments/assets/1eebe5de-80f7-4895-b9c9-4e851c1dfdc8)

![Image](https://github.com/user-attachments/assets/42c5fde2-4371-4747-8052-1c99b823a9b1)


### 3.2 코멘트 리스트

![Image](https://github.com/user-attachments/assets/b0a399c4-cf8e-4ee9-99a2-e79b69c4b4d6)

- 코멘트 리스트 요청 시 리턴 형태
```json
[
        {
            "createdTime": "2025-05-30T11:45:27.858694",
            "updatedTime": null,
            "id": 46,
            "commentWriter": "testid",
            "commentContents": "1111",
            "childrenComments": [
                {
                    "createdTime": "2025-05-30T11:48:54.690654",
                    "updatedTime": null,
                    "id": 53,
                    "commentWriter": "testid2",
                    "commentContents": "888",
                    "childrenComments": [
                        {
                            "createdTime": "2025-05-30T11:48:59.234274",
                            "updatedTime": null,
                            "id": 54,
                            "commentWriter": "testid2",
                            "commentContents": "999",
                            "childrenComments": []
                        },
                        {
                            "createdTime": "2025-05-30T11:49:21.175973",
                            "updatedTime": null,
                            "id": 57,
                            "commentWriter": "testid",
                            "commentContents": "ccc",
                            "childrenComments": []
                        }
                    ]
                },
                {
                    "createdTime": "2025-05-30T11:49:04.657264",
                    "updatedTime": null,
                    "id": 55,
                    "commentWriter": "testid2",
                    "commentContents": "aaa",
                    "childrenComments": []
                }
            ]
        },
    ]
```
코멘트의 경우는 게시판 아이디를 부모키로 가지며 또한 덧글 달기로 부모 코멘트와 자식 코멘트를 가질 수 있어 리스트가 트리 형태로 리턴 되기에 재귀 함수를 통해 리스트 컴포넌트를 만들어 화면에 보여 주도록 하였다.

- CommentList.js
```js
function recursiveMap(commentLists, level, depthVal) {
    commentLists.map((commentList) => {
      ...
      // 코멘트에 자식 코멘트 존재 여부 확인
      if(commentList["childrenComments"] !== "" && commentList["childrenComments"] !== null 
        && commentList["childrenComments"].length > 0
      ){
        
        renderVal.push(<Comment key={commentList["id"]} style={{ paddingLeft: depthStyle }}>
          <CommentContent>
              ... 
          </CommentContent>
         </Comment>
         );
        // 배열 전개 구문 ...로 기존 배열에 새로운 렌더링 대상 값을 추가한다.
        setCommentListRender([...commentListRender, 
          renderVal]);
        // 자식 코멘트 존재 시 코멘트 depth 값을 증가 시키기고 재귀 호출로 코멘트 리스트를 다시 만든다.
        recursiveMap(commentList["childrenComments"], "child", depthVal+1)
      }else{
        renderVal.push(<Comment key={commentList["id"]} style={{ paddingLeft: depthStyle }}>
          <CommentContent>
            ...
         </CommentContent>
         </Comment>);
          setCommentListRender([...commentListRender, 
           renderVal]);
      }
    });
  }
```


## 4. 예약
### 4.1 기본기능
![Image](https://github.com/user-attachments/assets/59e3a5f2-6d17-4cbc-8b13-3ce09b4431c0)

예약 페이지는 주말이 아닌 현재 일자 이후만 예약이 가능 하도록 구성 하였다

![Image](https://github.com/user-attachments/assets/20e8e41d-80af-471a-b49d-2d3a85cadc90)

원하는 일자 선택 시 예약자 아이디와 이름은 세션에서 가져오도록 하고 예약 시간을 선택한 만큼 예약 기간 값은 업데이트 된다.

![Image](https://github.com/user-attachments/assets/471a61eb-c517-48dc-b38f-f67e8889408b)

![Image](https://github.com/user-attachments/assets/e6779d37-a772-492c-9364-1f7e31332ba3)

자신이 선택한 예약 리스트 선택 시 예약 시간등을 업데이트 할 수 있으며 리스트가 아닌 일자 선택 시 기존 예약된 시간은 예약이 불가능 하도록 disable 처리 되도도록 하였다.
또한 예약 리스트는 자신이 예약한 리스트만 보여주도록 구현해 보았다.

### 4.2 FullCalendar 컴포넌트 사용
![Image](https://github.com/user-attachments/assets/b037c63e-cee5-498f-8f87-8c66c2eb7ff4)

예약 페이지를 구현하기 위해 직접 달력 UI를 만들지 않고 오픈 소스 캘린더 라이브러리인 FullCalendar를 이용해 보았다 Premium 버전 등이 있지만 Standard 버전으로 원하는 기능 구현이 충분 하기에 Standard 버전으로 구성 하였다.
FullCalendar는 next, prev 버튼에 대한 이벤트 props가 없으므로 customButtons
를 만들어 headerToolbar에 버튼이 보이도록 하였고 calendarRef를 선언하여 버튼 클릭 시 FullCalendar 클래스 컴포넌트에 접근하여 DOM 오브젝트를 제어하고 
setToolBarState를 통해 리렌더링을 발생 시켜 다음월, 이전월의 예약 데이터를 가져오고 화면에 보여 줄 수 있도록 하였다.

- Reserve.js
``` js
...
async function getData() {
  await Axios.get(`${process.env.NEXT_PUBLIC_API_URL}/api/v1/reserve/reserveList`, {
      headers: {
        "Content-Type": "application/json", 
        access: localStorage.getItem("access") 
      },
      params: {
        reserveDate: toolBarState,
        // 예약 리스트를 요청하는 api url에 년월(ex- 202507) 값을 전달하여 원하는 월에 해당하는 예약 리스트만 가져오게 하였다.
        reserveUserId: userId
      },
    }
...
// FullCalendar는 클래스 컴포넌트이므로 createRef를 통해 calendarRef를 선언한다.
const calendarRef = createRef(null);
const handleNextButtonClick = () => {
if (calendarRef.current) {
  // calendarRef를 통해 FullCalendar 클래스 컴포넌트 객체에 접근하여 현재 FullCalendar의 currentData 값을 가져올 수 있다.
  const currentMonth = moment(calendarRef.current.calendar.currentData.currentDate).format('YYYYMM');
    // 또한 calendarRef를 통해 getApi()를 호출하여 calendarApi.next()로 다음 월로 이동 할 수 있다.
  const calendarApi = calendarRef.current.getApi();
  calendarApi.next();
  // 서버에서 현재 FullCalendar 보여주고 있는 월의 예약 리스트를 가져오기 위해 toolBarState state를 변경 한다.
  setToolBarState(parseInt(currentMonth)+1);
}
};
const handlePrevButtonClick = () => {
if (calendarRef.current) {
  const currentMonth = moment(calendarRef.current.calendar.currentData.currentDate).format('YYYYMM');
  const calendarApi = calendarRef.current.getApi();
  calendarApi.prev();
  setToolBarState(parseInt(currentMonth)-1);
}
};
... 
<FullCalendar
    plugins={[dayGridPlugin, timeGridPlugin, interactionPlugin]}
    // next, prev 버튼을 클릭 시 예약 년월 state를 변경 해주기 위해 customButtons 추가
    headerToolbar={{
      left: 'prev,next today',
      center: 'title',
      right: 'dayGridMonth'
    }}
    customButtons= {{
      prev: {
        text: 'prev',
        click: handlePrevButtonClick
      },
      next: {
        text: 'next',
        click: handleNextButtonClick
      }
    }}
    ref={calendarRef}
    initialView='dayGridMonth'
    select={handleSelectedDates}
    eventClick={handleEventClick}
    editable={false}
    selectable={true}
    selectMirror={true}
    dayMaxEvents={true}
    weekends={true}
    // events prop에 reserveData state를 설정
    events={{events: reserveData}}
    // FullCalendar 컴포넌트를 렌더링하고 events 데이터에는 사용자가 예약한 데이터들을 커스텀하게 만들어 할당해 주었다.
    eventTimeFormat={{
      hour: '2-digit',
      minute: '2-digit',
      hour12: false
    }}
    displayEventEnd={true}
    />
```
서버로 부터 리턴 받은 예약 리스트를 FullCalendar를 events prop에 할당 가능한 형태로 가공하여 FullCalendar의 events={{events: reserveData}}로 이벤트를 할당하여 화면에 보여 줄 수 있도록 하였다.

- Reserve.js
```js

for (var timeKey in response.data[responseKey]["reserveTime"]) {
  reserveTotalList.push(
  {
    id: response.data[responseKey]["id"],
    title: response.data[responseKey]["reserveReason"],
    reserveReason: response.data[responseKey]["reserveReason"],
    reserveDate: response.data[responseKey]["reserveDate"],
    hallId: response.data[responseKey]["hallId"],
    reservePeriod: response.data[responseKey]["reservePeriod"],
    start: moment(response.data[responseKey]["reserveDate"]).format("YYYY-MM-DD")+"T"+response.data[responseKey]["reserveTime"][timeKey]["time"]["time"]+":00:00",
    end: moment(response.data[responseKey]["reserveDate"]).format("YYYY-MM-DD")+"T"+response.data[responseKey]["reserveTime"][timeKey]["time"]["time"]+":00:00",
    time: response.data[responseKey]["reserveTime"][timeKey]["time"]["time"],
    userId: response.data[responseKey]["userId"],
    allDay: false
    }
    );
}
```

### 4.3 예약 시간 처리(react reducer)
![Image](https://github.com/user-attachments/assets/f9193c9b-940e-47bb-b8ab-ab352d1205a8)

예약 시간 처리도 게시판의 검색 기능 처럼 react reducer를 이용해 구현 해보 았다. handleTimeChange에서 체크박스의 상태에 따라 times 값을 처리 하도록 하였고 체크된 상태에서 이미 예약이 있는 일자 선택 시 times 값을 false로 처리 하기 위해 useEffect에서 렌더링 시 action.type INITIAL로 reducer를 호출 하도록 하였다.

- Reserve.js
```js
const [times, dispatch] = React.useReducer(reserveTimeReducer, initialTimes); 

useEffect(() => {
  getData();
  formMode === "update" ? getDetailData() : 
  dispatch({
    type: 'INITIAL',
    times: initialTimes
  });
  // 렌더링 시 action.type INITIAL로 reducer 호출하여 기존 예약과 중복된 시간으로 예약이 불가 하도록 처리
  setReserveDetail("");
  clearTextInput();

}, [selectDate, reserveDetailId]);

// 예약 시간 리스트 action.type 별로 분기하여 처리 후 리턴
function reserveTimeReducer(times, action) {
  switch (action.type) {
    case 'INITIAL':
      return action.times;
    case 'CHECK':
      return [...times, action.timeId];
    case 'UNCHECK':
      return times.filter(t => t !== action.timeId);
    default:
      throw new Error()
  }
}
...
// 예약 시간 체크 시 checked 상태에 따라 다른 actionType으로 dispatch 호출
const handleTimeChange = (e) => {
  let actionType = "";
  e.target.checked ? actionType = "CHECK" : actionType = "UNCHECK";
  
  e.target.checked ? setIsChecked(true) : setIsChecked(false);
  dispatch({ type: actionType, timeId: e.target.tabIndex})
}

```