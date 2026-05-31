# AGENTS-store.md — Vuex Store 심층 분석

## 모듈별 역할 요약

| 모듈 | namespaced | 역할 | 주요 state |
|------|-----------|------|-----------|
| `user.js` | false | 인증/사용자 정보 | token, user, roles, loadMenus |
| `permission.js` | false | 동적 라우팅/메뉴 | routers, addRouters, sidebarRouters |
| `app.js` | **true** | UI 상태 (사이드바/디바이스) | sidebar, device, size |
| `settings.js` | **true** | 테마/레이아웃 설정 | theme, tagsView, fixedHeader, sidebarLogo 등 |
| `api.js` | false | API URL 중앙 관리 | deployUploadApi, fileUploadApi, baseApi 등 |
| `tagsView.js` | **true** | 탭 뷰 관리 | visitedViews, cachedViews |

## 모듈 상세

### user.js (`src/store/modules/user.js`)

```
state: {
  token: getToken(),    // Cookie에서 초기화
  user: {},             // 사용자 상세 정보 (id, nickName, gender, phone 등)
  roles: [],            // 권한 역할 배열 (예: ['ROLE_ADMIN'])
  loadMenus: false      // 로그인 직후 메뉴 로드 트리거
}
```

| Action | 역할 |
|--------|------|
| `Login(userInfo)` | 로그인 → setToken(Cookie) → SET_TOKEN → setUserInfo → SET_LOAD_MENUS(true) |
| `GetInfo()` | 백엔드에서 사용자 정보 재조회 → setUserInfo |
| `LogOut()` | 백엔드 logout API 호출 → logOut helper (토큰/roles 초기화) |
| `updateLoadMenus()` | loadMenus를 false로 변경 (메뉴 로드 완료 표시) |

- `setUserInfo(res, commit)`: roles가 빈 배열이면 `'ROLE_SYSTEM_DEFAULT'` 부여 (무한루프 방지)
- `logOut(commit)`: SET_TOKEN(''), SET_ROLES([]), removeToken()
- **namespaced가 없으므로** dispatch 시 `this.$store.dispatch('Login', ...)` 형식

### permission.js (`src/store/modules/permission.js`)

```
state: {
  routers: constantRouterMap,   // 정적 + 동적 전체 라우트
  addRouters: [],               // 동적으로 추가된 라우트만
  sidebarRouters: []            // 사이드바 렌더링용 라우트
}
```

| Action | 역할 |
|--------|------|
| `GenerateRoutes(asyncRouter)` | 동적 라우트 저장 (routers = 정적 + 동적) |
| `SetSidebarRouters(sidebarRouter)` | 사이드바용 라우트 저장 |

- `filterAsyncRouter(routers, lastRouter, type)`: 백엔드 메뉴 데이터를 Vue Router 형식으로 변환
  - `'Layout'` → `@/layout/index` 컴포넌트
  - `'ParentView'` → `@/components/ParentView` 컴포넌트
  - 그 외 → `loadView(component)` = `require([@/views/${view}], resolve)` (lazy load)
- `filterChildren()`: ParentView 중첩 라우트를 평탄화 (path拼接)
- **namespaced가 없으므로** dispatch 시 `this.$store.dispatch('GenerateRoutes', ...)` 형식

### app.js (`src/store/modules/app.js`)

```
state: {
  sidebar: {
    opened: Cookies.get('sidebarStatus') ? !!+Cookies.get('sidebarStatus') : true,
    withoutAnimation: false
  },
  device: 'desktop',    // 'desktop' | 'mobile'
  size: Cookies.get('size') || 'small'  // Element UI 사이즈
}
```

| Action | 역할 | Cookie |
|--------|------|--------|
| `toggleSideBar()` | 사이드바 열기/닫기 토글 | sidebarStatus 저장 |
| `closeSideBar({ withoutAnimation })` | 사이드바 강제 닫기 | sidebarStatus = 0 |
| `toggleDevice(device)` | desktop/mobile 전환 | - |
| `setSize(size)` | Element UI 사이즈 변경 | size 저장 |

- **namespaced: true** → `this.$store.dispatch('app/toggleSideBar')`
- 사이드바 상태는 Cookie로 영속화 (새로고침 후에도 유지)

### settings.js (`src/store/modules/settings.js`)

```
state: {
  theme: variables.theme,       // element-variables.scss에서 가져옴
  showSettings: false,          // 설정 패널 표시 여부
  tagsView: true,               // src/settings.js 기본값
  fixedHeader: true,
  sidebarLogo: true,
  showFooter: true,
  footerTxt: '...',
  caseNumber: ''
}
```

| Action | 역할 |
|--------|------|
| `changeSetting({ key, value })` | state[key] = value (동적 변경) |

- **namespaced: true** → `this.$store.dispatch('settings/changeSetting', { key: 'tagsView', value: false })`
- `CHANGE_SETTING` mutation이 `hasOwnProperty` 체크 후 동적으로 state 업데이트
- Settings 패널(`layout/components/Settings/index.vue`)에서 토글 UI 제공

### api.js (`src/store/modules/api.js`)

```
state: {
  deployUploadApi:   baseUrl + '/api/deploy/upload',
  databaseUploadApi: baseUrl + '/api/database/upload',
  imagesUploadApi:   baseUrl + '/api/localStorage/pictures',
  updateAvatarApi:   baseUrl + '/api/users/updateAvatar',
  s3UploadApi:       baseUrl + '/api/s3Storage',
  sqlApi:            baseUrl + '/druid/index.html',
  swaggerApi:        baseUrl + '/doc.html',
  fileUploadApi:     baseUrl + '/api/localStorage',
  baseApi:           baseUrl
}
```

- **mutations/actions 없음** (읽기 전용 설정 저장소)
- `baseUrl` = `process.env.VUE_APP_BASE_API === '/' ? '' : process.env.VUE_APP_BASE_API` (Nginx 프록시 대응)
- 파일 업로드 URL을 컴포넌트에서 하드코딩하지 않고 store getter로 접근
- **namespaced가 없으므로** getters로만 접근: `mapGetters(['fileUploadApi', 'baseApi'])`

### tagsView.js (`src/store/modules/tagsView.js`)

```
state: {
  visitedViews: [],   // 열린 탭 목록 (route 객체 배열)
  cachedViews: []     // keep-alive 캐시 대상 컴포넌트 name 배열
}
```

| Action | 역할 |
|--------|------|
| `addView(view)` | visitedView + cachedView 동시 추가 |
| `delView(view)` | visitedView + cachedView 동시 삭제 |
| `delOthersViews(view)` | 현재 탭 + affix 외 모든 탭 닫기 |
| `delAllViews()` | affix 태그 외 모든 탭 닫기 |
| `updateVisitedView(view)` | 탭 정보 갱신 (라우트 변경 시) |

- **namespaced: true** → `this.$store.dispatch('tagsView/addView', this.$route)`
- `meta.noCache: true`이면 cachedViews에 추가하지 않음
- `meta.affix: true`이면 delAllViews/delOthers에서 보호됨
- `layout/components/TagsView/index.vue`에서 주로 사용

## 컴포넌트에서 Store 접근 패턴

### 1. mapGetters (가장 흔한 패턴)

```js
// src/layout/components/Navbar.vue
import { mapGetters } from 'vuex'

computed: {
  ...mapGetters([
    'sidebar',      // app 모듈
    'device',       // app 모듈
    'user',         // user 모듈
    'baseApi'       // api 모듈
  ])
}
```

```js
// src/views/system/user/center.vue
computed: {
  ...mapGetters([
    'user',             // user.user
    'updateAvatarApi',  // api.updateAvatarApi
    'baseApi'           // api.baseApi
  ])
}
```

```js
// src/layout/components/Sidebar/index.vue
computed: {
  ...mapGetters([
    'sidebarRouters',   // permission.sidebarRouters
    'sidebar'           // app.sidebar
  ])
}
```

### 2. this.$store.dispatch() 직접 호출

```js
// namespaced 모듈 (app, settings, tagsView)
this.$store.dispatch('app/toggleSideBar')
this.$store.dispatch('settings/changeSetting', { key: 'showSettings', value: val })
this.$store.dispatch('tagsView/addView', this.$route)
this.$store.dispatch('tagsView/delView', view)

// 비-namespaced 모듈 (user, permission)
this.$store.dispatch('Login', user)         // login.vue
this.$store.dispatch('LogOut')              // Navbar.vue
this.$store.dispatch('GetInfo')             // router/index.js, center.vue
this.$store.dispatch('GenerateRoutes', routes)
```

### 3. this.$store.state 직접 접근

```js
// src/layout/components/Sidebar/index.vue
computed: {
  showLogo() {
    return this.$store.state.settings.sidebarLogo
  }
}

// src/layout/components/Navbar.vue
computed: {
  show: {
    get() { return this.$store.state.settings.showSettings },
    set(val) { this.$store.dispatch('settings/changeSetting', { key: 'showSettings', value: val }) }
  }
}
```

### 4. 컴포넌트별 사용 모듈 매핑

| 컴포넌트 | 사용하는 store 모듈 |
|---------|-------------------|
| `layout/components/Navbar.vue` | app (sidebar), user (user, logout), settings (showSettings), api (baseApi) |
| `layout/components/Sidebar/index.vue` | app (sidebar), permission (sidebarRouters), settings (sidebarLogo) |
| `layout/components/TagsView/index.vue` | tagsView (addView, delView, delOthersViews, delAllViews) |
| `layout/components/Settings/index.vue` | settings (changeSetting) |
| `views/login.vue` | user (Login) |
| `views/system/user/center.vue` | user (user, GetInfo), api (updateAvatarApi, baseApi) |
| `views/system/user/index.vue` | api (fileUploadApi, updateAvatarApi) |
| `views/maint/deploy/deploy.vue` | api (deployUploadApi) |
| `views/maint/database/execute.vue` | api (databaseUploadApi) |
| `views/monitor/sql/index.vue` | api (sqlApi) |
| `views/tools/storage/local/index.vue` | api (fileUploadApi, imagesUploadApi) |
| `views/tools/storage/s3/index.vue` | api (s3UploadApi) |
| `components/SizeSelect/index.vue` | app (setSize), tagsView (delAllCachedViews) |
| `components/WangEditor/index.vue` | api (imagesUploadApi) |

## 로그인/로그아웃 전체 흐름

### 로그인 흐름

```
login.vue (submit)
  │
  ├─ this.$store.dispatch('Login', { username, password, code, uuid })
  │   │
  │   ├─ POST /auth/login (src/api/login.js)
  │   ├─ setToken(res.token, rememberMe)     → Cookie 저장 (ELADMIN-TOEKN)
  │   ├─ commit('SET_TOKEN', res.token)      → state.user.token
  │   ├─ setUserInfo(res, commit)            → SET_ROLES, SET_USER
  │   └─ commit('SET_LOAD_MENUS', true)      → 메뉴 로드 플래그
  │
  └─ this.$router.push({ path: redirect || '/' })
      │
      └─ router.beforeEach (src/router/index.js)
          │
          ├─ getToken() 있음 → roles.length === 0?
          │   │
          │   ├─ YES: store.dispatch('GetInfo')
          │   │       ├─ GET /auth/info
          │   │       └─ setUserInfo(res, commit)
          │   │
          │   └─ loadMenus 플래그 확인 → loadMenus === true?
          │       │
          │       ├─ YES: store.dispatch('updateLoadMenus')  → false로 변경
          │       │       │
          │       │       └─ loadMenus(next, to)
          │       │           ├─ buildMenus() → GET /auth/build
          │       │           ├─ filterAsyncRouter(sdata)     → 사이드바용
          │       │           ├─ filterAsyncRouter(rdata, false, true) → 라우터용
          │       │           ├─ store.dispatch('GenerateRoutes', rewriteRoutes)
          │       │           ├─ router.addRoutes(rewriteRoutes)
          │       │           ├─ store.dispatch('SetSidebarRouters', sidebarRoutes)
          │       │           └─ next({ ...to, replace: true })
          │       │
          │       └─ NO: next()
          │
          └─ (이미 roles 있음 + loadMenus false) → next()
```

### 로그아웃 흐름

```
Navbar.vue (logout 버튼)
  │
  ├─ this.$store.dispatch('LogOut')
  │   │
  │   ├─ POST /auth/logout (src/api/login.js)
  │   └─ logOut(commit)
  │       ├─ commit('SET_TOKEN', '')
  │       ├─ commit('SET_ROLES', [])
  │       └─ removeToken()  → Cookie에서 ELADMIN-TOEKN 삭제
  │
  └─ location.reload()  → Vue Router 재인스턴스화 (동적 라우트 제거)
```

### 토큰 만료 (401) 처리

```
Axios 응답 인터셉터 (src/utils/request.js)
  │
  ├─ response.status === 401
  │   │
  │   ├─ store.dispatch('LogOut')
  │   ├─ Cookies.set('point', 401)   → 로그인 페이지에서 만료 안내 표시
  │   └─ location.reload()
  │
  └─ login.vue에서 Cookies.get('point') 확인
      └─ '현재 로그인 상태가 만료되었습니다' 알림 표시
```

## 새 Store 모듈 추가 방법

### 1. 파일 생성

`src/store/modules/` 디렉토리에 `.js` 파일 생성:

```js
// src/store/modules/myModule.js
const state = {
  myData: null
}

const mutations = {
  SET_MY_DATA: (state, data) => {
    state.myData = data
  }
}

const actions = {
  setMyData({ commit }, data) {
    commit('SET_MY_DATA', data)
  }
}

export default {
  namespaced: true,   // 권장
  state,
  mutations,
  actions
}
```

### 2. 자동 등록 확인

`src/store/index.js`의 `require.context('./modules', true, /\.js$/)`가 자동으로 감지하므로 **import나 등록 코드 불필요**.

### 3. Getter 추가 (선택)

`src/store/getters.js`에 전역 getter 추가:

```js
const getters = {
  // ... 기존 getters
  myData: state => state.myModule.myData
}
```

### 네이밍 규칙

| 항목 | 규칙 | 예시 |
|------|------|------|
| 파일명 | camelCase | `myModule.js` |
| state 속성 | camelCase | `myData` |
| mutation | SCREAMING_SNAKE_CASE | `SET_MY_DATA` |
| action | camelCase | `setMyData` |
| 모듈명 = 파일명 (확장자 제외) | | `myModule` |

### namespaced 권장

- `namespaced: true` 설정 권장
- dispatch: `this.$store.dispatch('myModule/setMyData', data)`
- getter 접근: `mapGetters(['myData'])` (getters.js에 등록 시)

## 하지 말아야 할 것

1. **api.js에 mutation/action 추가하지 말 것** — 이 모듈은 읽기 전용 URL 설정 저장소
2. **user.js, permission.js에 namespaced: true를 추가하지 말 것** — `router/index.js`와 `request.js`에서 루트 레벨 dispatch('Login', 'LogOut', 'GetInfo' 등)를 사용 중. namespaced를 추가하면 모든 호출부를 수정해야 함
3. **tagsView를 직접 mutation하지 말 것** — 항상 action을 통해 조작 (addView, delView 등)
4. **settings.js에 새 state를 추가할 때 기본값을 src/settings.js에도 추가할 것** — `defaultSettings`에서 구조분해로 초기화하므로 settings.js와 동기화 필요
5. **store 모듈에서 직접 router를 import하지 말 것** — permission.js만 예외적으로 `constantRouterMap`을 import. 라우팅 로직은 `router/index.js`의 `loadMenus()`에서 처리
6. **Cookie에 민감 정보를 저장하지 말 것** — 현재 login.vue에서 username/password를 Cookie에 저장하는 기능이 있음 (rememberMe). 이는 기존 코드 패턴이지만 새 기능에서는 피할 것
7. **loadView()의 동적 import 경로를 변경하지 말 것** — `require([@/views/${view}], resolve)` 패턴은 Webpack 코드 스플리팅에 의존. 경로 변경 시 모든 동적 라우트가 깨짐
