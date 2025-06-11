# - 프로젝트 개요
이전 React 프로젝트에 사용자 관리 메뉴를 통해 Spring Security의 사용자 권한 제어를 확인 하는 페이지를 구현 하였고 실제 사용자를 관리하는 기능은 Vuejs를 통해 간단하게 구현해 보았다.\
**Vue와 React를 동일 프로젝트에서 구성하는 것은 UI 프레임워크의 목적에 맞지 않으므로 Vuejs를 사용하여 간단한 프로젝트를 별도로 진행 하였다.**\

<br /><br />
# - 개발기간
- 2025.05 ~ 2025.05(약 0.5개월)\
<br /><br />
# - 개발환경
- node.js v18.20.5
- vue v3.5.13, vuetify v3.8.1
- vuex v4.1.0
<br /><br />
# - 주요기능
- 사용자의 리스트를 보여주는 페이지와 페이징 및 검색 기능 
- 사용자의 정보를 변경 및 삭제 하는 기능을 간단히 구현
- Vuejs를 위한 디자인 UI 프레임워크 vuetify를 사용하여 페이지 디자인을 구현 하였으며 백엔드 부분의 데이터 처리를 위해 Axios 라이브러리를 사용

<br /><br />

# - 상세기능
### 1. 데이터 처리
컴포넌트간 데이터 전달을 위해 props와 vuex 라이브러리를 통한 중앙 집중식 저장소 방식 2가지를 사용 해보았다

#### 1.1 props 사용 방식
props 사용 방식을 통해 하위 컴포너트로 데이터를 전달하고 자식과 부모 사이는 하향식 단방향 바인딩 형태 이어야 하므로 하위 컴포넌트의 클릭 이벤트 등에 대한 처리는 **emit** 이벤트를 호출 하여 구현 하였다.

##### 1.1.1 페이징
- App.vue
```js
// currentPage.value를 자식 컴포넌트로 전달하여 UserList 컴포넌트에서 v-pagination에 현재 페이지 값을 전달한다.
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
...

// Axios를 통해 사용자 리스트를 요청하는 api 호출 
const getData = async () => {

await Axios.get('http://localhost:8090/api/v1/user/userList', {
  params: {
        page: currentPage.value,
        size: '5',
        sort: 'createdTime,desc',
        searchKey: searchFiled.value,
        searchValue: searchTxt.value
      },
      headers: {
        access: localStorage.getItem('access')
      },
  }).then((response) => {

      userList.value = response.data.content;
      pageLength.value= response.data.totalPages;

  }).catch(function (error) {
    console.log('error : ' + error)
  });
...
// 자식 컴포넌트에의해 pageClick이 호출되어 백엔드의 요청 페이지에 해당되는 사용자 리스트를 가져온다.
const pageClick = (page) => {
  currentPage.value = page;
  getData();
}
``` 
위 처럼 상위 컴포너트를 통해 자식 컴포넌트로 값을 전달 하고
- MainComp.vue
```js
const props = defineProps(['userList', 'currentPage', 'pageLength', 'showModal'])
```
자식컴포넌트에서도 **defineProps**를 매크로를 사용하여 props를 선언하여 구현 하였다.

- UserList.vue
```js
...
const props = defineProps(['userList', 'currentPage', 'pageLength', 'showModal'])
const emit = defineEmits(['mainPageClick', 'showModalPop'])

// emit을 통해 부모 컴포넌트의 mainPageClick 이벤트를 유발하고 상위 컴포넌트인 MainComp에서 mainPageClick 이벤트를 통해 최상위 컴포넌트의 pageClick 발생 시킨다.
function handlePageClick(pageVal) {
  emit('mainPageClick', pageVal);
}
...
<v-pagination
    v-model="props.currentPage"
    :length="props.pageLength"
    rounded="circle"
    @update:model-value="handlePageClick">
</v-pagination>
...
```
- 페이징 동작화면
![1_page](https://github.com/user-attachments/assets/90e7f1c7-5f58-40ca-82e4-a6911f056d15)


##### 1.1.2 검색

- SearchBar.vue
```js
...
// 검색 대상 필드 선택이나 검색어 입력, 검색 버튼 클릭 시 이벤트를 발생 시키도록 하고
    <v-card-text>
      <v-text-field
        :loading="loading"
        append-inner-icon="mdi-magnify"
        density="compact"
        label="Search templates"
        variant="solo"
        hide-details
        single-line
        @click:append-inner="userSearch"
        @update:modelValue="inputSearch"
        @userSearch="userSearch"
      ></v-text-field>
```
```js
// 발생된 이벤트에서 emit 부모 컴포넌트로 인수 전달, 이벤트 유발
const emit = defineEmits(['pageClick', 'selectFiled', 'inputSearch', 'mainPageClick'])

const selectFiled = (selectSearchFiled) => {
  emit('selectFiled', selectSearchFiled);
}

const inputSearch = (inputSearch) => {
  emit('inputSearch', inputSearch);
}

const userSearch = () => {
  emit('userSearch');
  emit('mainPageClick', 1);
}
```
- App.vue
```js
// emit으로 검색 필드와 검색 텍스트를 전달 받고 사용자 검색 이벤트가 발생된다.
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
...
const userSearch = () => {
  getData();
}
```
- 검색 동작화면
![Image](https://github.com/user-attachments/assets/9b10a027-20aa-40bf-9fba-7a51560418cc)

#### 1.2 Vuex 라이브러리 사용 방식
여러 컴포넌트간에 저장소 공유를 위해 Vuejs 애플리케이션에 대한 상태 관리 패턴 라이브러리인 Vuex 사용해 보았으며 state와 mutations, 비동기 작업 처리를 위한 actions를 store.js에 선언 하여 사용 하였다.  
- 참고  
<https://v3.vuex.vuejs.org/kr/>


##### 1.2.1 store.js
- store.js
```js
export const store = new Vuex.Store({
  state: {
    count: 0,
    count2: 10,
    showModifyModal: false,
    showRoleListModal: false,
    userId: 0,
    ...
    
// state를 통해 선언된 상태를 mutations를 통해서 변경
mutations: {
    showModifyModal (state) {
      state.showModifyModal = !state.showModifyModal
    },
    showRoleListModal (state) {
      state.showRoleListModal = !state.showRoleListModal
    },
    ...
// async 사용하는 비동기 처리 함수는 actions을 통해 처리
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
... 
```
store에 선언된 state와 mutations, actions를 통해 사용자의 상세 정보, 정보 수정, 삭제 등을 처리 하였다


##### 1.2.2 사용자 수정
- UserList.vue
```js
...
// 수정 버튼 클릭 시 store의 showModifyModal mutations을 호출하고 showModifyModal state를 true로 변경
<td><v-btn color="#5865f2"
  size="small"
  variant="flat"
  @click="store.commit('showModifyModal'), store.commit('setUserId', users.id), store.dispatch('getUserData')">Modify</v-btn></td>
<td><v-btn color="#5865f2"
```

- ModifyModal.vue
```js
// click 리스너를 통해 onClick 이벤트를 발생 시키고 setRoleUserSave mutations로 사용자 권한 state를 수정하고 userUpdate action으로 사용자 수정 api를 호출한다.
<v-btn class="mt-2" type="submit" @click="userUpdate" block>Submit</v-btn>
...
// 
function userUpdate(){
  const roleUserSave = ref([]);

  store.state.userDetail.roleUser.map((roles) =>{
  roleUserSave.value.push(roles.roleId)
  })

  store.commit('setRoleUserSave', roleUserSave.value)
  store.dispatch('userUpdate')
}
```

- store.js
```js
// store의 사용자 권한 state를 수정 mutations
setRoleUserSave (state, roleUserSave) {
  state.roleUserSave = roleUserSave;
}
// store의 사용자 수정 api를 호출 action으로
async userUpdate ({ state, commit }) {
  console.log('userUpdate state.userDetail : ' + JSON.stringify(state.userDetail));
  await Axios.post('http://localhost:8090/api/v1/user/userUpdate',
    {
      id: state.userDetail.id,
      loginId: state.userDetail.loginId,
      userName: state.userDetail.userName,
      userPassword: state.userDetail.userPassword,
      roleUserSave: state.roleUserSave
  },
    {
    headers: {
      access: localStorage.getItem('access')
      }
  }
  ).then(response => {
    commit('setRoleList', response.data);
    if(response.status === 200)
    {
      alert('Update Success');
      window.location.reload();
    }
  }).catch(function (error) {
    console.log('error : ' + error)
  });
},
```
- 사용자 수정 동작화면

### 2. 사용자 권한 수정
사용자는 여러 Role을 가질 수 있으며 해당 Role을 가진 사용자가 다수가 존재 할 수 있어 중간 관계 테이블을 추가하여 사용자와 Role 데이터를 관리 하도록 구성 하였다.
```java
.requestMatchers("/api/v1/admin/*").hasAnyRole("ADMIN", "MANAGER")
.anyRequest().authenticated());
```
백엔드 영역에서 Spring Security SecurityConfig에서도 관리자 권한을 여러 권한으로 체크 하도록 하였으므로 사용관 관리 페이지에서 사용자 수정 팝업업 호출 시 기존 사용자가 가진 권한을 DB에서 가져오도록 하였다.

- store.js
```js
// store의 사용자 권한 리스트 요청 action
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
          //유저 정보 업데이트 시 업데이트 될 Role 리스트를 함께 전달 한다
          roleUserSave: state.roleUserSave
      },
```