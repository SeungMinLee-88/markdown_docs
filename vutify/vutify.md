# - 프로젝트 개요
기존 React 프로젝트에 사용자 관리 메뉴를 통해 Spring Security의 사용자 Role 기능을 구현 하였고 실제 사용자를 관리하는 기능은 Vue.js를 통해 간단하게 구현해 보았다.

# - 주요 구현 기능
기능은 사용자의 리스트를 보여주는 페이지와 사용자의 정보를 변경하는 기능(사용자 이름과, Role 정보만 변경), 사용자를 삭제 하는 기능을 간단히 구현 하였다.

또한 페이지의 Vue.js를 위한 디자인 UI 프레임워크 vuetify를 사용하여 구현으며 백엔드 부분의 데이터 처리를 위해 Axios 라이브러리를 사용 하였다.하였다.

- 참고  
<https://vuetifyjs.com/en/>  
<https://axios-http.com/kr/docs/intro>

![Image](https://github.com/user-attachments/assets/bfcaae53-b5cf-4950-beff-1a2242f587a9)

# - 주요 특징
1. 컴포넌트간 데이터 전달을 위해 props와 vuex 라이브러리를 통한 중앙 집중식 저장소 방식 2가지를 사용 해보았다

#### 1.1 props 사용 방식

```js
<MainComp
        :userList=userList
        :currentPage=currentPage
        :pageLength=pageLength
        @pageClick='pageClick'
        @selectFiled='selectFiled'
        @inputSearch='inputSearch'
        :searchFiled='searchFiled'
        :searchTxt='searchTxt'
        @userSearch='userSearch'
        @showModalPop='showModalPop'
      />
``` 
위 처럼 상위 컴포너트를 통해 자식 컴포넌트로 값을 전달 하고
```js
const props = defineProps(['userList', 'currentPage', 'pageLength', 'showModal'])
```
자식컴포넌트에서도 **defineProps**를 매크로를 사용하여 props를 선언하여 구현 하였으며 페이징 처리, 검색 처리 등에 props를 사용 하여 구현 하였다.

```js
   <v-pagination
        v-model="props.currentPage"
        :length="props.pageLength"
        rounded="circle"
        @update:model-value="handlePageClick"
      ></v-pagination>
```
```js
function handlePageClick(pageVal) {
  emit('mainPageClick', pageVal);
}
```
![1_page](https://github.com/user-attachments/assets/90e7f1c7-5f58-40ca-82e4-a6911f056d15)

자식과 부모 사이는 하향식 단방향 바인딩 형태 이어야 하므로 클릭 이벤트 등에 대한 처리는 **emit** 이벤트를 호출 하여 구현 하였다.

#### 1.2 vuex 라이브러리 사용 방식
여러 컴포넌트간에 저장소 공유를 위해 Vue.js 애플리케이션에 대한 상태 관리 패턴, 라이브러리인 vuex를 사용해 보았으며 state와 mutations, 비동기 작업 처리를 위한 actions를 store.js에 선언 하여 사용 하였다.  
- 참고  
<https://v3.vuex.vuejs.org/kr/>

store.js
```js
export const store = new Vuex.Store({
  state: {
    count: 0,
    count2: 10,
    showModifyModal: false,
    showRoleListModal: false,
    userId: 0,
    ... 중략
    
    /* state를 통해 선언된 상태를 mutations를 통해서 변경 */
mutations: {
    showModifyModal (state) {
      state.showModifyModal = !state.showModifyModal
    },
    showRoleListModal (state) {
      state.showRoleListModal = !state.showRoleListModal
    },
    
    ... 중략
    
    /* async 사용하는 비동기 함수는 actions을 통해 처리 */
    actions: {
    async getUserData ({ state, commit }) {
      await Axios.post('http://localhost:8090/api/v1/user/userDetail',
      {
        id: state.userId
      },
        {
            headers: {
            access: localStorage.getItem('access')
            }
      }).then((response) => {
        commit('setUserDetail', response.data);
    ... 이하 생략
```
store에 선언된 state와 mutations, actions를 통해 사용자의 상세 정보, 정보 수정, 삭제 등을 처리 하였다

```js
<!-- click 리스너를 통해 onClick 이벤트를 발생 시킨 후 처리 -->
<v-btn class="mt-2" type="submit" @click="userUpdate" block>Submit</v-btn>
```
```js
function userUpdate(){
  const roleUserSave = ref([]);

  store.state.userDetail.roleUser.map((roles) =>{
  roleUserSave.value.push(roles.roleId)
  })

  store.commit('setRoleUserSave', roleUserSave.value)
  store.dispatch('userUpdate')
}
```
1. 사용자는 여러 Role을 가질 수 있으며 해당 Role을 가진 사용자가 다수가 존재 할 수 있어 중간 관계 테이블을 추가하여 사용자와 Role 데이터를 관리 하도록 구성하였으며
```java
.requestMatchers("/api/v1/admin/*").hasAnyRole("ADMIN", "MANAGER")
.anyRequest().authenticated());
```
백엔드 영역에서 Spring Security SecurityConfig에서도 관리자 권한을 여러 권한으로 체크 하도록 하였으므로 사용관 관리 페이지에서 사용자 수정 팝업업 호출 시 기존 사용자가 가진 권한을 DB에서 가져오도록 하였다.
```js
    /* 추가 대상 Role 리스트 */
    async getRoleList ({ state, commit }) {
      await Axios.post('http://localhost:8090/api/v1/user/roleList',
        state.exceptRoleList,
        {
          headers: {
            access: localStorage.getItem('access')
          },
        }

      ).then(response => {
        commit('setRoleList', response.data);
```
Role리스트를 호출 시 현재 팝업창에서 가져온 사용자의 Role 리스트를 제외한 리스트를 보여 주도록 하였다. Role 추가 후 저장 시 Role 데이터를 가진 state를 request 값으로 전달하여 사용자의 Role 정보가 업데이트 되도록 하였다.

![2_modify](https://github.com/user-attachments/assets/77c1f9c9-1256-437e-84dd-0ac3ed586c92)

```js
      await Axios.post('http://localhost:8090/api/v1/user/userUpdate',
        {
          id: state.userDetail.id,
          loginId: state.userDetail.loginId,
          userName: state.userDetail.userName,
          userPassword: state.userDetail.userPassword,
          roleUserSave: state.roleUserSave
    /* 유저 정보 업데이트 시 업데이트 될 Role 리스트를 함께 전달 */
      },
```