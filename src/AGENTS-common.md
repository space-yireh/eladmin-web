# AGENTS-common.md -- eladmin-web 공통 설정 & 유틸리티 심층 분석

## 1. Axios 설정 (src/utils/request.js)

### 인스턴스 생성

```js
const service = axios.create({
  baseURL: process.env.NODE_ENV === 'production' ? process.env.VUE_APP_BASE_API : '/',
  timeout: Config.timeout // 1,200,000ms (20분)
})
```

- **프로덕션**: `VUE_APP_BASE_API` 직접 사용 (예: `https://eladmin.vip`)
- **개발**: `/` 사용 → `vue.config.js`의 프록시를 통해 `http://localhost:8000`으로 전달

### 요청 인터셉터

```js
service.interceptors.request.use(config => {
  if (getToken()) {
    config.headers['Authorization'] = getToken()
  }
  config.headers['Content-Type'] = 'application/json'
  return config
})
```

- Cookie에서 토큰을 읽어 `Authorization` 헤더에 주입
- **Bearer 접두사 없음** -- 토큰 문자열 그대로 전달
- 모든 요청의 Content-Type을 `application/json`으로 고정

### 응답 인터셉터

```js
service.interceptors.response.use(
  response => response.data,  // 응답 데이터만 반환 (response.data)
  error => { /* 에러 처리 */ }
)
```

### 에러 처리 흐름

| 상태 코드 | 동작 |
|-----------|------|
| **Blob 응답 에러** | FileReader로 JSON 파싱 → `message` 필드를 Notification.error로 표시 |
| **401** | `store.dispatch('LogOut')` → Cookie에 `point=401` 저장 → `location.reload()` |
| **403** | `router.push({ path: '/401' })` -- 권한 없음 페이지로 이동 |
| **기타** | `error.response.data.message`를 Notification.error로 표시 (5초) |
| **timeout** | "네트워크 요청 타임아웃" Notification 표시 |
| **응답 없음** | "인터페이스 요청 실패" Notification 표시 |

### API 함수 작성 패턴

```js
// src/api/system/role.js
import request from '@/utils/request'

export function add(data) {
  return request({ url: 'api/roles', method: 'post', data })
}
export function edit(data) {
  return request({ url: 'api/roles', method: 'put', data })
}
export function del(ids) {
  return request({ url: 'api/roles', method: 'delete', data: ids })
}
export function get(id) {
  return request({ url: 'api/roles/' + id, method: 'get' })
}
// CRUD mixin에서 사용할 수 있도록 default export 필수
export default { add, edit, del, get }
```

### 공통 데이터 조회/다운로드 (src/api/data.js)

```js
import request from '@/utils/request'
import qs from 'qs'

// GET 목록 조회 (페이지네이션 포함)
export function initData(url, params) {
  return request({
    url: url + '?' + qs.stringify(params, { indices: false }),
    method: 'get'
  })
}

// Blob 다운로드
export function download(url, params) {
  return request({
    url: url + '?' + qs.stringify(params, { indices: false }),
    method: 'get',
    responseType: 'blob'
  })
}
```

---

## 2. 토큰/인증 관리

### 토큰 저장 (src/utils/auth.js)

```js
import Cookies from 'js-cookie'
import Config from '@/settings'

const TokenKey = Config.TokenKey // 'ELADMIN-TOEKN' (오타 주의: TOKEN이 아님)

export function getToken() {
  return Cookies.get(TokenKey)
}
export function setToken(token, rememberMe) {
  if (rememberMe) {
    return Cookies.set(TokenKey, token, { expires: Config.tokenCookieExpires }) // 1일
  } else {
    return Cookies.set(TokenKey, token) // 세션 쿠키
  }
}
export function removeToken() {
  return Cookies.remove(TokenKey)
}
```

### 설정값 (src/settings.js)

| 키 | 값 | 용도 |
|----|----|------|
| `TokenKey` | `'ELADMIN-TOEKN'` | Cookie에 저장되는 토큰 키 (오타 있음) |
| `tokenCookieExpires` | `1` | rememberMe 시 Cookie 만료일 (1일) |
| `passCookieExpires` | `1` | rememberMe 시 비밀번호 Cookie 만료일 |
| `timeout` | `1200000` | 요청 타임아웃 (ms, 20분) |
| `title` | `'ELADMIN'` | 브라우저 탭 제목 접미사 |
| `tagsView` | `true` | 탭 뷰 표시 |
| `fixedHeader` | `true` | 헤더 고정 |
| `sidebarLogo` | `true` | 사이드바 로고 표시 |

### 토큰 처리 전체 흐름

```
로그인 요청 (auth/login)
  → 응답: { token: "Bearer xxx", user: {...} }
  → setToken(token, rememberMe) → Cookie('ELADMIN-TOEKN', token)
  → Vuex: SET_TOKEN, SET_USER, SET_ROLES, SET_LOAD_MENUS(true)

후속 요청
  → request.js 인터셉터: Cookie에서 getToken()
  → headers['Authorization'] = 'Bearer xxx'

401 응답 수신
  → store.dispatch('LogOut') → removeToken() + roles 초기화
  → Cookies.set('point', 401) → location.reload()
  → 로그인 페이지에서 point 쿠키 확인 후 안내 메시지 표시
```

---

## 3. 라우터 설정

### 정적 라우트 (src/router/routers.js)

| 경로 | 컴포넌트 | 비고 |
|------|---------|------|
| `/login` | `views/login` | 로그인 페이지 |
| `/404` | `views/features/404` | 404 페이지 |
| `/401` | `views/features/401` | 권한 없음 페이지 |
| `/redirect/:path*` | `views/features/redirect` | 리다이렉트용 |
| `/` → `/dashboard` | `views/home` | 대시보드 (Layout 래핑) |
| `/user/center` | `views/system/user/center` | 개인 센터 |

- **mode**: `history` (해시 모드 아님)
- **hidden: true**인 라우트는 사이드바에 표시되지 않음

### 네비게이션 가드 (src/router/index.js)

```
beforeEach(to, from, next)
  ├── 토큰 있음 (getToken())
  │   ├── to.path === '/login' → next({ path: '/' }) // 이미 로그인됨
  │   ├── roles.length === 0 (첫 진입)
  │   │   → GetInfo() → loadMenus() → 동적 라우트 추가
  │   ├── loadMenus === true (로그인 직후)
  │   │   → loadMenus() → 동적 라우트 추가
  │   └── 그 외 → next()
  └── 토큰 없음
      ├── whiteList 포함? (/login) → next()
      └── 그 외 → next('/login?redirect=...')
```

### 동적 라우트 생성 흐름

```
1. buildMenus() API 호출 (GET /auth/buildMenus)
2. 응답 받은 메뉴 데이터를 JSON 복제 (사이드바용 + 라우팅용)
3. filterAsyncRouter()로 컴포넌트 문자열 → 실제 컴포넌트 변환
   - 'Layout' → @/layout/index
   - 'ParentView' → @/components/ParentView
   - 그 외 → @/views/{component} (동적 import)
4. rewriteRoutes에 { path: '*', redirect: '/404' } 추가
5. router.addRoutes(rewriteRoutes)로 동적 등록
6. store에 사이드바 라우터 저장
```

### 화이트리스트

```js
const whiteList = ['/login']
```

---

## 4. CRUD 시스템 (핵심)

이 프로젝트에는 **2개의 CRUD 시스템**이 공존합니다:

| 시스템 | 위치 | 사용 방식 | 특징 |
|--------|------|----------|------|
| **mixin CRUD** | `src/mixins/crud.js` | `mixins: [crud]` | 구형, 단순 mixin |
| **CRUD 인스턴스** | `src/components/Crud/crud.js` | `cruds()` + `presenter()/header()/form()/crud()` | **신형, 권장** |

**대부분의 views/ 페이지가 신형 CRUD 인스턴스 방식을 사용하므로, 새 페이지는 반드시 신형 방식을 사용할 것.**

### 4-1. 신형 CRUD 인스턴스 (src/components/Crud/crud.js) -- 권장

#### 기본 구조

```js
import CRUD, { presenter, header, form, crud } from '@crud/crud'
import rrOperation from '@crud/RR.operation'
import crudOperation from '@crud/CRUD.operation'
import udOperation from '@crud/UD.operation'
import pagination from '@crud/Pagination'

const defaultForm = {
  id: null,
  name: null,
  enabled: true
}

export default {
  name: 'ExamplePage',
  components: { pagination, crudOperation, rrOperation, udOperation },
  // CRUD 인스턴스 생성 (cruds 옵션 사용)
  cruds() {
    return CRUD({
      title: '예제',
      url: 'api/examples',
      crudMethod: { ...crudExample }
    })
  },
  // 4가지 mixin 적용
  mixins: [presenter(), header(), form(defaultForm), crud()],
  data() {
    return {
      permission: {
        add: ['admin', 'example:add'],
        edit: ['admin', 'example:edit'],
        del: ['admin', 'example:del']
      },
      rules: {
        name: [{ required: true, message: '이름을 입력하세요', trigger: 'blur' }]
      }
    }
  }
}
```

#### CRUD() 생성자 옵션

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `tag` | `'default'` | 다중 CRUD 사용 시 식별자 |
| `idField` | `'id'` | ID 필드명 |
| `title` | `''` | 제목 (알림, 다이얼로그 제목에 사용) |
| `url` | `''` | API 엔드포인트 |
| `query` | `{}` | 검색 쿼리 초기값 |
| `params` | `{}` | 고정 쿼리 파라미터 |
| `form` | `{}` | 폼 데이터 |
| `defaultForm` | `() => {}` | 폼 초기값 (객체 또는 함수) |
| `sort` | `['id,desc']` | 정렬 (다중 필드 가능) |
| `time` | `0` | 로딩 후 테이블 표시 대기 시간 (ms) |
| `crudMethod` | `{ add, del, edit, get }` | CRUD API 메서드 |
| `optShow` | `{ add, edit, del, download, reset }` | 툴바 버튼 표시 여부 |
| `queryOnPresenterCreated` | `true` | created 시 자동 조회 |
| `debug` | `false` | 디버그 모드 (hook 로그 출력) |

#### 4가지 Mixin 역할

| Mixin | 역할 | VM 타입 |
|-------|------|---------|
| `presenter()` | 메인 페이지. CRUD 인스턴스 소유, created 시 자동 조회, table 연결 | `presenter` (index 0) |
| `header()` | 검색 헤더. `query` 접근 가능 | `header` (index 1) |
| `form(defaultForm)` | 폼 컴포넌트. `form` 접근 가능, defaultForm 설정 | `form` (index 3) |
| `crud()` | 기타 컴포넌트. crud 인스턴스에 접근만 필요할 때 | 커스텀 타입 |

#### CRUD 인스턴스의 주요 속성

```js
crud.data          // 테이블 데이터 배열
crud.loading       // 로딩 상태
crud.page          // { page, size, total }
crud.query         // 검색 조건 객체
crud.params        // 고정 파라미터 객체
crud.form          // 현재 폼 데이터
crud.selections    // 테이블에서 선택된 행들
crud.status        // { add, edit, cu (computed), title (computed) }
crud.status.cu     // 0=NORMAL, 1=PREPARED, 2=PROCESSING
crud.status.title  // '新增예제' 또는 '编辑예제'
crud.dataStatus    // 각 행별 상태 { [id]: { delete, edit } }
crud.optShow       // 툴바 버튼 표시 설정
crud.msg           // 알림 메시지 { submit, add, edit, del }
crud.props         // 확장 속성 (Vue.set으로 반응형)
```

#### CRUD 인스턴스의 주요 메서드

```js
// 조회
crud.toQuery()                    // 검색 (page를 1로 리셋 후 refresh)
crud.refresh()                    // 현재 페이지로 데이터 재조회
crud.resetQuery(toQuery = true)   // query를 초기값으로 리셋

// 신규/편집
crud.toAdd()                      // 신규 모드 진입 (status.add = PREPARED)
crud.toEdit(data)                 // 편집 모드 진입 (form에 data 복사)
crud.submitCU()                   // 폼 유효성 검사 후 doAdd()/doEdit() 실행
crud.cancelCU()                   // 신규/편집 취소

// 삭제
crud.toDelete(data)               // 삭제 준비 (status.delete = PREPARED)
crud.doDelete(data)               // 실제 삭제 실행 (단건 또는 배열)
crud.cancelDelete(data)           // 삭제 취소

// 기타
crud.doExport()                   // 엑셀 다운로드
crud.resetForm(data?)             // 폼을 defaultForm으로 리셋
crud.notify(title, type)          // 알림 표시
crud.updateProp(name, value)      // props 확장 속성 업데이트 (반응형)
crud.getTable()                   // el-table 참조 반환
```

#### CRUD HOOK 목록

Hook은 컴포넌트의 메서드로 정의하면 자동으로 호출됩니다.
`[CRUD.HOOK.xxx]()` 구문을 사용합니다.

| Hook 상수 | 메서드명 | 호출 시점 |
|-----------|---------|----------|
| `beforeRefresh` | `beforeCrudRefresh` | 데이터 조회 전 (false 반환 시 중단) |
| `afterRefresh` | `afterCrudRefresh` | 데이터 조회 후 |
| `beforeToAdd` | `beforeCrudToAdd` | 신규 폼 열기 전 |
| `afterToAdd` | `afterCrudToAdd` | 신규 폼 연 후 |
| `beforeToEdit` | `beforeCrudToEdit` | 편집 폼 열기 전 |
| `afterToEdit` | `afterCrudToEdit` | 편집 폼 연 후 |
| `beforeToCU` | `beforeCrudToCU` | 신규/편집 폼 열기 전 (공통) |
| `afterToCU` | `afterCrudToCU` | 신규/편집 폼 연 후 (공통) |
| `beforeValidateCU` | `beforeCrudValidateCU` | 폼 유효성 검사 전 |
| `afterValidateCU` | `afterCrudValidateCU` | 폼 유효성 검사 후 (false 반환 시 제출 중단) |
| `beforeSubmit` | `beforeCrudSubmitCU` | API 제출 전 (false 반환 시 중단) |
| `afterSubmit` | `afterCrudSubmitCU` | API 제출 성공 후 |
| `afterAddError` | `afterCrudAddError` | 신규 등록 실패 후 |
| `afterEditError` | `afterCrudEditError` | 편집 실패 후 |
| `beforeDelete` | `beforeCrudDelete` | 삭제 API 호출 전 (false 반환 시 중단) |
| `afterDelete` | `afterCrudDelete` | 삭제 성공 후 |
| `beforeDeleteCancel` | `beforeCrudDeleteCancel` | 삭제 취소 전 |
| `afterDeleteCancel` | `afterCrudDeleteCancel` | 삭제 취소 후 |
| `beforeAddCancel` | `beforeCrudAddCancel` | 신규 취소 전 |
| `afterAddCancel` | `afterCrudAddCancel` | 신규 취소 후 |
| `beforeEditCancel` | `beforeCrudEditCancel` | 편집 취소 전 |
| `afterEditCancel` | `afterCrudEditCancel` | 편집 취소 후 |

**tag 기반 Hook**: `crud-tag="job"` 사용 시 `beforeCrudToAdd$job` 형태로 호출 가능

#### CRUD 컴포넌트

```html
<!-- 검색/리셋 버튼 -->
<rrOperation />

<!-- 툴바 (신규/수정/삭제/다운로드 + 컬럼 표시 토글) -->
<crudOperation :permission="permission" />

<!-- 행별 수정/삭제 버튼 -->
<udOperation :data="scope.row" :permission="permission" />

<!-- 페이지네이션 -->
<pagination />
```

#### 완전한 템플릿 예시 (역할 관리 기준)

```html
<template>
  <div class="app-container">
    <!-- 툴바 -->
    <div class="head-container">
      <div v-if="crud.props.searchToggle">
        <el-input v-model="query.blurry" size="small" clearable
          placeholder="검색어 입력" style="width: 200px;"
          class="filter-item" @keyup.enter.native="crud.toQuery" />
        <rrOperation />
      </div>
      <crudOperation :permission="permission" />
    </div>

    <!-- 신규/편집 다이얼로그 -->
    <el-dialog append-to-body :close-on-click-modal="false"
      :before-close="crud.cancelCU"
      :visible.sync="crud.status.cu > 0"
      :title="crud.status.title" width="520px">
      <el-form ref="form" :inline="true" :model="form"
        :rules="rules" size="small" label-width="80px">
        <el-form-item label="이름" prop="name">
          <el-input v-model="form.name" style="width: 380px;" />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="text" @click="crud.cancelCU">취소</el-button>
        <el-button :loading="crud.status.cu === 2"
          type="primary" @click="crud.submitCU">확인</el-button>
      </div>
    </el-dialog>

    <!-- 테이블 -->
    <el-table ref="table" v-loading="crud.loading"
      :data="crud.data"
      @selection-change="crud.selectionChangeHandler">
      <el-table-column type="selection" width="55" />
      <el-table-column prop="name" label="이름" />
      <el-table-column prop="createTime" label="생성일" />
      <el-table-column label="작업" width="130px" align="center">
        <template slot-scope="scope">
          <udOperation :data="scope.row" :permission="permission" />
        </template>
      </el-table-column>
    </el-table>

    <!-- 페이지네이션 -->
    <pagination />
  </div>
</template>
```

#### Hook 사용 예시

```js
methods: {
  // 데이터 조회 후 트리 체크박스 초기화
  [CRUD.HOOK.afterRefresh]() {
    this.$refs.menu.setCheckedKeys([])
  },
  // 신규 폼 열기 전 초기화
  [CRUD.HOOK.beforeToAdd](crud, form) {
    this.deptDatas = []
    form.menus = null
  },
  // 편집 폼 열기 전 데이터 로드
  [CRUD.HOOK.beforeToEdit](crud, form) {
    this.deptDatas = []
    form.depts.forEach(dept => {
      this.deptDatas.push(dept.id)
    })
    form.menus = null
  },
  // 제출 전 추가 유효성 검사
  [CRUD.HOOK.afterValidateCU](crud) {
    if (crud.form.dataScope === '사용자 정의' && this.deptDatas.length === 0) {
      this.$message({ message: '데이터 권한을 선택하세요', type: 'warning' })
      return false  // 제출 중단
    }
    return true
  }
}
```

### 4-2. 구형 mixin CRUD (src/mixins/crud.js) -- 레거시

`src/views/system/user/center.vue`, `src/views/system/timing/log.vue` 등 일부 페이지만 사용.
**새 페이지에서는 사용하지 말 것.**

#### 제공하는 data 속성

| 속성 | 기본값 | 설명 |
|------|--------|------|
| `data` | `[]` | 테이블 데이터 |
| `sort` | `['id,desc']` | 정렬 |
| `page` | `0` | 현재 페이지 (0-based) |
| `size` | `10` | 페이지 크기 |
| `total` | `0` | 전체 데이터 수 |
| `url` | `''` | API URL |
| `params` | `{}` | 고정 파라미터 |
| `query` | `{}` | 검색 조건 |
| `time` | `50` | 로딩 후 대기 시간 |
| `isAdd` | `false` | 신규/편집 구분 |
| `loading` | `true` | 테이블 로딩 |
| `delLoading` | `false` | 삭제 로딩 |
| `delAllLoading` | `false` | 일괄 삭제 로딩 |
| `downloadLoading` | `false` | 다운로드 로딩 |
| `dialog` | `false` | 다이얼로그 표시 |
| `form` | `{}` | 폼 데이터 |
| `resetForm` | `{}` | 폼 리셋용 백업 |
| `title` | `''` | 제목 |

#### 주요 메서드

| 메서드 | 설명 |
|--------|------|
| `init()` | 데이터 조회 (beforeInit 호출 후 initData) |
| `toQuery()` | 검색 (page=0 리셋 후 init) |
| `pageChange(e)` | 페이지 변경 (e-1로 설정 후 init) |
| `sizeChange(e)` | 페이지 크기 변경 후 init |
| `submitMethod()` | `$refs['form'].validate()` → addMethod()/editMethod() |
| `addMethod()` | `crudMethod.add(form)` 호출 |
| `editMethod()` | `crudMethod.edit(form)` 호출 |
| `delMethod(id)` | `crudMethod.del(id)` 호출 (확인 팝업 포함) |
| `delAllMethod()` | 선택된 행들 일괄 삭제 |
| `downloadMethod()` | `url/download`에서 Blob 다운로드 |
| `showAddFormDialog()` | 신규 다이얼로그 표시 |
| `showEditFormDialog(data)` | 편집 다이얼로그 표시 |
| `cancel()` | 다이얼로그 닫기 + 폼 리셋 |

#### 생명주기 훅

| 훅 | 호출 시점 |
|----|----------|
| `beforeInit()` | init() 실행 전 (false 반환 시 중단) |
| `beforeDelMethod()` | 삭제 전 (false 반환 시 중단) |
| `afterDelMethod()` | 삭제 후 |
| `beforeShowAddForm()` | 신규 다이얼로그 표시 전 |
| `beforeShowEditForm(data)` | 편집 다이얼로그 표시 전 |
| `afterAddMethod()` | 신규 등록 성공 후 |
| `afterAddErrorMethod()` | 신규 등록 실패 후 |
| `afterEditMethod()` | 편집 성공 후 |
| `beforeSubmitMethod()` | 제출 전 (false 반환 시 중단) |

---

## 5. 권한 관리

### v-permission 디렉티브

```html
<!-- 버튼에 권한 적용 -->
<el-button v-permission="['admin', 'roles:add']">신규</el-button>

<!-- 조건부 표시 -->
<el-table-column v-if="checkPer(['admin','roles:edit','roles:del'])" label="작업">
```

**동작 방식**: 현재 사용자의 roles 배열에 지정된 권한 중 하나라도 포함되면 표시, 아니면 DOM에서 제거

### checkPer 메서드 (src/utils/permission.js)

```js
// Vue 인스턴스 내에서 사용 가능 (전역 등록됨)
this.checkPer(['admin', 'editor']) // true/false
```

### Permission 컴포넌트

```html
<permission :value="['admin']">
  <el-button>관리자 전용</el-button>
</permission>
```

---

## 6. 데이터 딕셔너리 (src/components/Dict)

### 사용법

```js
export default {
  dicts: ['job_status', 'user_gender'],  // 컴포넌트 옵션에 dicts 배열 선언
  // created에서 자동으로 딕셔너리 데이터 로드
}
```

```html
<!-- 템플릿에서 사용 -->
<el-select v-model="form.status">
  <el-option v-for="item in dict.job_status" :key="item.id"
    :label="item.label" :value="item.value" />
</el-select>
```

- `dictReady` 이벤트로 로드 완료 감지 가능

---

## 7. 공통 유틸 함수 Top 10

### src/utils/index.js

| 함수 | 시그니처 | 용도 |
|------|---------|------|
| `parseTime` | `(time, format)` | 날짜 포맷팅. 기본: `'{y}-{m}-{d} {h}:{i}:{s}'` |
| `downloadFile` | `(obj, name, suffix)` | Blob → 파일 다운로드 (파일명: `날짜-name.suffix`) |
| `formatTime` | `(time, option)` | 상대적 시간 표시 ("방금 전", "3분 전", "2시간 전") |
| `debounce` | `(func, wait, immediate)` | 디바운스 |
| `deepClone` | `(source)` | 깊은 복사 (단순 버전) |
| `uniqueArr` | `(arr)` | 배열 중복 제거 |
| `param` | `(json)` | 객체 → URL 쿼리 문자열 |
| `param2Obj` | `(url)` | URL → 쿼리 객체 |
| `regEmail` | `(email)` | 이메일 마스킹 (`abc***@gmail.com`) |
| `regMobile` | `(mobile)` | 전화번호 마스킹 (`010****1234`) |

### src/utils/validate.js

| 함수 | 용도 |
|------|------|
| `isExternal(path)` | 외부 URL 판별 (`https?:`, `mailto:`, `tel:`) |
| `validURL(url)` | URL 유효성 검사 |
| `validEmail(email)` | 이메일 유효성 검사 |
| `isvalidPhone(phone)` | 휴대폰 번호 유효성 검사 (중국 번호) |
| `validateIP(rule, value, callback)` | IP 주소 유효성 (el-form validator용) |
| `validatePhone(rule, value, callback)` | 전화번호 유효성 (el-form validator용) |
| `validateIdNo(rule, value, callback)` | 주민등록번호 유효성 (el-form validator용) |
| `validatePhoneTwo(rule, value, callback)` | 휴대폰 또는 유선전화 유효성 |

---

## 8. 전역 등록 항목 (src/main.js)

```js
Vue.use(checkPer)      // this.checkPer() 메서드 (권한 체크)
Vue.use(permission)    // v-permission 디렉티브
Vue.use(dict)          // dicts 옵션 활성화 (데이터 딕셔너리)
Vue.use(Element, { size: Cookies.get('size') || 'small' })
```

| 등록 항목 | 타입 | 소스 |
|----------|------|------|
| `checkPer` | 전역 메서드 | `src/utils/permission.js` |
| `v-permission` | 디렉티브 | `src/components/Permission/permission.js` |
| `dicts` 옵션 | 전역 mixin | `src/components/Dict/index.js` |
| Element UI | UI 라이브러리 | 기본 size: `small` |

---

## 9. Vuex Store 구조

### 주요 Getters

| Getter | 소스 | 설명 |
|--------|------|------|
| `roles` | `state.user.roles` | 현재 사용자 권한 배열 |
| `user` | `state.user.user` | 사용자 정보 객체 |
| `token` | `state.user.token` | 현재 토큰 |
| `loadMenus` | `state.user.loadMenus` | 메뉴 로드 필요 여부 |
| `sidebarRouters` | `state.permission.sidebarRouters` | 사이드바 라우터 |
| `sidebar` | `state.app.sidebar` | 사이드바 상태 |
| `size` | `state.app.size` | Element UI 사이즈 |
| `fileUploadApi` | `state.api.fileUploadApi` | 파일 업로드 API URL |
| `imagesUploadApi` | `state.api.imagesUploadApi` | 이미지 업로드 API URL |

### Store 모듈 자동 로드

```js
// src/store/index.js
const modulesFiles = require.context('./modules', true, /\.js$/)
// modules/ 폴더의 모든 .js 파일이 자동 등록됨
```

---

## 10. 하지 말아야 할 것

1. **Authorization 헤더에 `Bearer ` 접두사를 수동으로 붙이지 말 것** -- 백엔드에서 이미 `Bearer xxx` 형태로 토큰을 반환하므로, `getToken()` 값을 그대로 사용하면 됨

2. **TokenKey를 `ELADMIN-TOKEN`으로 수정하지 말 것** -- 기존 값이 `ELADMIN-TOEKN` (오타)이므로, 수정하면 기존 사용자의 세션이 모두 끊김

3. **`response.data` 대신 `response`를 사용하지 말 것** -- 응답 인터셉터에서 이미 `response.data`만 반환하므로, API 호출 결과에서 `.data`를 다시 접근하면 안 됨

4. **새 CRUD 페이지에 `src/mixins/crud.js`를 사용하지 말 것** -- `src/components/Crud/crud.js`의 `CRUD()` 인스턴스 + `presenter()/header()/form()/crud()` mixin 방식을 사용할 것

5. **API 파일에서 `export default`를 빠뜨리지 말 것** -- CRUD 인스턴스의 `crudMethod`에서 `{ ...crudApi }` 스프레드를 사용하므로 default export 필수

6. **`cruds()` 대신 `data()`에서 CRUD 인스턴스를 생성하지 말 것** -- `beforeCreate` 훅에서 `$options.cruds`를 읽으므로 반드시 `cruds()` 옵션으로 정의

7. **`$refs.form`을 수동으로 validate하지 말 것** -- `crud.submitCU()`가 자동으로 `$refs['form'].validate()`를 호출함

8. **page를 1-based로 설정하지 말 것 (구형 mixin)** -- 구형 mixin은 0-based, 신형 CRUD는 1-based. 혼용 시 페이지 오류 발생

9. **`router.addRoutes()` 전에 catch-all 라우트를 수동으로 추가하지 말 것** -- `loadMenus()`에서 자동으로 `{ path: '*', redirect: '/404' }`를 추가함

10. **form에 `createTime`, `updateTime`, `createBy`, `updateBy` 필드를 포함하지 말 것** -- `resetForm()`이 자동으로 이 필드들을 삭제함. defaultForm에도 포함하지 말 것

11. **`Content-Type`을 수동으로 설정하지 말 것** -- 요청 인터셉터에서 모든 요청에 `application/json`을 자동 설정함

12. **딕셔너리 데이터를 수동으로 로드하지 말 것** -- `dicts: ['dict_name']` 옵션만 선언하면 created에서 자동 로드됨

13. **동적 라우트에서 component를 직접 import하지 말 것** -- 백엔드에서 받은 `component` 문자열이 `loadView()`를 통해 `@/views/${view}`로 자동 변환됨

14. **`location.reload()`를 함부로 호출하지 말 것** -- 라우터 인스턴스를 재생성하므로 버그의 원인이 됨. 401 로그아웃 시에만 사용
