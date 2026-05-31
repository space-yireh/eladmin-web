# AGENTS.md — eladmin-web

## 프로젝트 개요

**ELADMIN** 관리 시스템 프론트엔드 (v2.7.0)

| 기술 | 버전 |
|------|------|
| Vue | 2.7.16 |
| Vue Router | 3.0.2 (history mode) |
| Vuex | 3.1.0 |
| Element UI | 2.15.14 |
| Axios | 1.8.2 |
| Webpack | 4.47.0 |
| Sass | 1.32.13 |
| ECharts | 4.2.1 |

- **Lint**: `npm run lint` (`eslint --ext .js,.vue src`)
- **테스트**: `npm run test:unit` (Jest + @vue/test-utils)

## src/ 폴더 구조

```
src/
├── api/            # 백엔드 API 호출 함수 (도메인별 분류)
│   ├── data.js     # 공통 데이터 조회/다운로드 (initData, download)
│   ├── login.js    # 인증 API (login, logout, getInfo, getCodeImg)
│   ├── system/     # 시스템 관리 (user, role, menu, dept, job, dict, timing, code)
│   ├── monitor/    # 모니터링 (online, log)
│   ├── tools/      # 도구 (email, alipay, localStorage, s3Storage)
│   ├── maint/      # 유지보수 (app, database, deploy, deployHistory, serverDeploy, connect)
│   └── generator/  # 코드 생성기 (generator, genConfig)
├── assets/         # 정적 자원
│   ├── icons/      # SVG 아이콘 (svg-sprite-loader로 번들링)
│   ├── images/     # 이미지
│   └── styles/     # SCSS 스타일 (element-variables.scss, index.scss)
├── components/     # 공통 재사용 컴포넌트 (23개)
│   ├── Crud/       # CRUD 시스템 핵심 (crud.js, CRUD.operation, RR/UD.operation, Pagination)
│   ├── Dict/       # 데이터 딕셔너리 (dicts 옵션으로 자동 로드)
│   ├── Pagination/ # 페이지네이션
│   ├── WangEditor/ # 리치 텍스트 에디터
│   ├── Permission/ # 권한 컴포넌트 + v-permission 디렉티브
│   ├── IconSelect/ # 아이콘 선택기
│   ├── HeaderSearch/ # 헤더 검색
│   ├── DateRangePicker/ # 날짜 범위 선택기
│   └── ...         # Breadcrumb, SvgIcon, Screenfull, SizeSelect, ParentView 등
├── layout/         # 앱 레이아웃
│   ├── index.vue   # 메인 레이아웃
│   └── components/ # AppMain, Navbar, Settings/, Sidebar/(6), TagsView/(2)
├── mixins/         # Vue 믹스인
│   └── crud.js     # 구형 CRUD mixin (레거시, 새 페이지에서 사용 금지)
├── router/         # 라우팅
│   ├── routers.js  # 정적 라우트 (login, 404, 401, dashboard, user center)
│   └── index.js    # 네비게이션 가드 + 동적 메뉴 로드
├── store/          # Vuex 스토어
│   ├── index.js    # require.context로 modules 자동 로드
│   ├── getters.js  # 전역 getters
│   └── modules/    # user, permission, app(NS), settings(NS), tagsView(NS), api
├── utils/          # 유틸리티
│   ├── request.js  # Axios 인스턴스 (인터셉터, 에러 핸들링)
│   ├── auth.js     # 토큰 관리 (Cookie 기반)
│   ├── permission.js # v-permission 지시어 + checkPer()
│   ├── validate.js # 유효성 검사
│   ├── rsaEncrypt.js # RSA 암호화 (비밀번호 전송용)
│   ├── upload.js   # 파일 업로드 (raw axios 사용, request.js 아님)
│   ├── datetime.js # 날짜/시간 포맷
│   ├── clipboard.js # 클립보드
│   ├── shortcuts.js # 단축키
│   └── index.js    # 공통 유틸 (parseTime, downloadFile, debounce, deepClone 등)
├── views/          # 페이지 컴포넌트 (api/와 1:1 매핑)
│   ├── login.vue   # 로그인 페이지
│   ├── home.vue    # 대시보드 홈
│   ├── system/     # 시스템 관리 (user, role, menu, dept, job, dict, timing)
│   ├── monitor/    # 모니터링 (log, online, server, sql)
│   ├── tools/      # 도구 (aliPay, email, storage)
│   ├── maint/      # 유지보수 (app, database, deploy, deployHistory, server)
│   ├── generator/  # 코드 생성기
│   ├── dashboard/  # 대시보드 위젯
│   ├── features/   # 404, 401, redirect
│   ├── components/ # 뷰 전용 컴포넌트
│   └── nested/     # 중첩 라우트 예제
├── App.vue         # 루트 컴포넌트
├── main.js         # 앱 진입점 (전역 등록: checkPer, v-permission, dicts, Element UI)
└── settings.js     # 전역 설정 (TokenKey, timeout, title, tagsView 등)
```

## 처음 투입된 개발자를 위한 가이드

### 가장 먼저 읽어야 할 파일 3개

1. **`src/components/Crud/crud.js`** — 이 프로젝트의 핵심. CRUD 인스턴스 생성, 4가지 mixin (presenter/header/form/crud), Hook 시스템, VM 등록 메커니즘이 모두 여기에 있음
2. **`src/utils/request.js`** — Axios 인터셉터. 토큰 주입, 401/403 처리, Blob 에러 처리 등 모든 HTTP 요청/응답이 이곳을 통과
3. **`src/router/index.js`** — 네비게이션 가드 + 동적 라우팅. 로그인 상태 확인, 메뉴 로드, `router.addRoutes()` 흐름을 이해해야 함

### 새 페이지 추가 시 참고 문서

1. `src/api/{도메인}/{리소스}.js` 생성 (add/edit/del + default export 필수)
2. `src/views/{도메인}/{리소스}/index.vue` 생성 (표준 CRUD 템플릿 복사)
3. 백엔드 메뉴 관리에서 동적 라우트 등록 (컴포넌트 경로: `도메인/리소스/index`)
4. **참고할 표준 페이지**: `src/views/system/dept/index.vue` (단순 CRUD), `src/views/system/user/index.vue` (복잡한 CRUD)

### 절대 하지 말아야 할 것 Top 5

1. **`src/store/modules/user.js`와 `permission.js`에 `namespaced: true`를 추가하지 말 것** — `router/index.js`, `request.js`, `login.vue` 등에서 루트 레벨 `dispatch('Login')`, `dispatch('LogOut')`, `dispatch('GetInfo')`를 사용 중. namespaced를 추가하면 인증 시스템 전체가 멈춤. 가장 치명적

2. **새 CRUD 페이지에 `src/mixins/crud.js`(구형)를 사용하지 말 것** — 반드시 `src/components/Crud/crud.js`의 `CRUD()` 인스턴스 + `presenter()/header()/form()/crud()` mixin 방식을 사용할 것. 구형은 일부 레거시 페이지만 유지 중

3. **`cruds()` 옵션 대신 `data()`에서 CRUD 인스턴스를 생성하지 말 것** — `presenter()` mixin의 `beforeCreate` 훅이 `this.$options.cruds`를 읽어서 인스턴스를 생성함. `data()`에서 만들면 VM 등록 타이밍이 맞지 않아 Hook이 작동하지 않음

4. **Authorization 헤더에 `Bearer ` 접두사를 수동으로 붙이지 말 것** — 백엔드가 이미 `Bearer xxx` 형태로 토큰을 반환하므로 `getToken()` 값을 그대로 사용. 중복 Bearer가 되면 인증 실패

5. **`response.data` 대신 `response` 전체를 사용하지 말 것** — 응답 인터셉터에서 이미 `response.data`만 반환하도록 처리됨. API 호출 결과에 `.data`를 다시 접근하면 `undefined`가 됨 (단, `src/utils/upload.js`의 raw axios는 예외)

## 전체 데이터 흐름

### CRUD 작업 흐름 (신규 등록 예시)

```
사용자: "신규" 버튼 클릭
  │
  ├─ crud.toAdd()
  │   ├─ crud.resetForm()                    → form을 defaultForm으로 초기화
  │   ├─ callVmHook(HOOK.beforeToAdd, form)  → 뷰의 beforeCrudToAdd() 호출
  │   ├─ callVmHook(HOOK.beforeToCU, form)   → 뷰의 beforeCrudToCU() 호출
  │   ├─ crud.status.add = PREPARED (1)      → 다이얼로그 열림 (visible.sync)
  │   ├─ callVmHook(HOOK.afterToAdd, form)
  │   └─ callVmHook(HOOK.afterToCU, form)
  │
사용자: 폼 입력 후 "확인" 버튼 클릭
  │
  ├─ crud.submitCU()
  │   ├─ callVmHook(HOOK.beforeValidateCU)   → false면 중단
  │   ├─ $refs['form'].validate(valid => {   → Element UI 폼 유효성 검사
  │   │   ├─ callVmHook(HOOK.afterValidateCU) → false면 중단
  │   │   └─ crud.doAdd()
  │   │       ├─ callVmHook(HOOK.beforeSubmit) → false면 중단
  │   │       ├─ crud.status.add = PROCESSING (2) → 버튼 로딩 표시
  │   │       │
  │   │       ├─ crud.crudMethod.add(crud.form)   → API 함수 호출 (예: crudRole.add)
  │   │       │   │
  │   │       │   └─ request({ url: 'api/roles', method: 'post', data: form })
  │   │       │       │
  │   │       │       ├─ 요청 인터셉터: headers['Authorization'] = getToken()
  │   │       │       ├─ POST http://localhost:8000/api/roles (프록시 경유)
  │   │       │       └─ 응답 인터셉터: response.data 반환
  │   │       │
  │   │       ├─ crud.status.add = NORMAL (0)     → 다이얼로그 닫힘
  │   │       ├─ crud.resetForm()
  │   │       ├─ crud.addSuccessNotify()           → "新增成功" Notification
  │   │       ├─ callVmHook(HOOK.afterSubmit)
  │   │       └─ crud.toQuery()                    → page=1 리셋 후 refresh()
  │   │           │
  │   │           └─ crud.refresh()
  │   │               ├─ callVmHook(HOOK.beforeRefresh) → false면 중단
  │   │               ├─ initData(url, getQueryParams())
  │   │               │   └─ GET api/roles?page=0&size=10&sort=id,desc
  │   │               ├─ crud.page.total = data.totalElements
  │   │               ├─ crud.data = data.content       → 테이블 자동 갱신
  │   │               ├─ crud.resetDataStatus()
  │   │               └─ callVmHook(HOOK.afterRefresh)
  │   │
  │   │   (실패 시)
  │   │   └─ .catch() → crud.status.add = PREPARED, callVmHook(HOOK.afterAddError)
  │   })
```

### CRUD 작업 흐름 (편집/삭제)

```
편집:
  crud.toEdit(row) → form에 row 깊은 복사 → status.edit = PREPARED
  crud.submitCU() → validate → doEdit()
    → crudMethod.edit(form) → PUT api/{resource}
    → 성공: refresh() (현재 페이지 유지)

삭제:
  crud.doDelete(data) → ids 배열 수집
    → callVmHook(HOOK.beforeDelete) → false면 중단
    → crudMethod.del(ids) → DELETE api/{resource} (body: ids)
    → 성공: dleChangePage() + refresh()
```

### 페이지네이션 파라미터 자동 조립

```js
// crud.getQueryParams() 내부
{
  page: crud.page.page - 1,  // 1-based → 0-based 변환
  size: crud.page.size,      // 기본값: 10
  sort: crud.sort,           // 기본값: ['id,desc']
  ...crud.query,             // v-model로 바인딩된 검색 조건
  ...crud.params             // 고정 파라미터
}
```

## 백엔드 API 연결 방식

### 프록시 설정 (vue.config.js)

| 프록시 경로 | 대상 | 비고 |
|------------|------|------|
| `/api` | `VUE_APP_BASE_API` (dev: `http://localhost:8000`) | pathRewrite: `^/api` → `api` |
| `/auth` | `VUE_APP_BASE_API` (dev: `http://localhost:8000`) | pathRewrite: `^/auth` → `auth` |

### 환경변수

| 변수 | 개발 (.env.development) | 프로덕션 (.env.production) |
|------|------------------------|--------------------------|
| `VUE_APP_BASE_API` | `http://localhost:8000` | `https://eladmin.vip` (또는 Nginx 프록시 시 `/`) |
| `VUE_APP_WS_API` | `ws://localhost:8000` | `wss://eladmin.vip` |

- `.env.staging` 파일은 존재하지 않음

### Axios 설정 (src/utils/request.js)

- **baseURL**: 프로덕션 = `VUE_APP_BASE_API`, 개발 = `/` (프록시 경유)
- **timeout**: 1,200,000ms (settings.js `timeout`, 20분)
- **요청 인터셉터**: `Authorization` 헤더에 토큰 자동 주입 (Bearer 접두사 포함됨, 수동 추가 금지), `Content-Type: application/json` 고정
- **응답 인터셉터**: `response.data`만 반환

| 상태 코드 | 동작 |
|-----------|------|
| 401 | `store.dispatch('LogOut')` → Cookie에 `point=401` 저장 → `location.reload()` |
| 403 | `router.push('/401')` — 권한 없음 페이지 |
| Blob 에러 | FileReader로 JSON 파싱 → Notification.error |
| 기타 | `error.response.data.message`를 Notification.error (5초) |

### 토큰 관리

- **저장**: Cookie (`ELADMIN-TOEKN` 키 — 오타 주의, TOKEN이 아님)
- **로그인**: `POST auth/login` → `{ token: "Bearer xxx" }` → `setToken(token)` → Cookie 저장
- **후속 요청**: 인터셉터가 Cookie에서 `getToken()` → `headers['Authorization']`에 그대로 주입
- **만료**: 401 응답 → LogOut → Cookie 삭제 → reload

## CRUD 시스템 (핵심)

### 2개의 CRUD 시스템 공존

| 시스템 | 위치 | 사용 방식 | 상태 |
|--------|------|----------|------|
| **CRUD 인스턴스** | `src/components/Crud/crud.js` | `cruds()` + `presenter()/header()/form()/crud()` | **신형, 권장** |
| **mixin CRUD** | `src/mixins/crud.js` | `mixins: [crud]` | 구형, 레거시 유지보수만 |

**새 페이지는 반드시 신형 CRUD 인스턴스 방식을 사용할 것.**

### CRUD 인스턴스 내부 동작 (src/components/Crud/crud.js)

#### 생성 과정

```js
CRUD(options)
  → mergeOptions(defaultOptions, options)    // 옵션 병합
  → data 객체 생성 (status, msg, page, loading 등)
  → methods 객체 생성 (refresh, toAdd, toEdit, submitCU, doDelete 등)
  → Object.assign(crud, data)
  → Vue.observable(crud)                     // 반응형 객체화
  → Object.assign(crud, methods)             // 메서드 추가
  → Object.freeze(crud)                      // 메서드 추가/변경 차단
  → return crud
```

- `Vue.observable()`로 반응형 처리 → `crud.data`, `crud.loading` 등 변경 시 뷰 자동 갱신
- `Object.freeze()`로 동결 → 메서드 오버라이드 불가. 확장 속성은 `crud.updateProp(name, value)`로 `crud.props`에 추가

#### VM 등록 시스템 (4 슬롯)

```
vms = Array(4)  // [presenter, header, pagination, form]

presenter()  → registerVM('presenter', vm, 0)  // 메인 페이지 (index.vue)
header()     → registerVM('header', vm, 1)     // 검색 헤더
pagination() → registerVM('pagination', vm, 2) // 페이지네이션
form()       → registerVM('form', vm, 3)       // 폼 다이얼로그
crud()       → registerVM(type, vm)            // 커스텀 (index >= 4)
```

- `findVM(type)`: 등록된 VM 중 해당 타입의 Vue 인스턴스 반환
- `crud.getTable()` = `findVM('presenter').$refs.table`
- `crud.submitCU()` = `findVM('form').$refs['form'].validate()` 사용

#### Hook 시스템 (callVmHook)

```js
callVmHook(crud, hook, ...args)
  → crud.vms의 모든 VM을 Set으로 중복 제거
  → 각 VM에서 vm[hook]() 호출
  → 하나라도 false 반환하면 전체 false (작업 중단)
  → tag 기반 Hook: hook + '$' + tag (예: 'beforeCrudToAdd$job')
```

- Hook은 뷰 컴포넌트의 `methods`에 `[CRUD.HOOK.xxx]()` 형태로 정의
- `false` 반환 시 해당 작업 중단 (beforeRefresh, beforeSubmit, afterValidateCU 등)

#### STATUS 상태 머신

```
NORMAL (0) → PREPARED (1) → PROCESSING (2) → NORMAL (0)
                 ↓ (cancel)
              NORMAL (0)
```

- `crud.status.add`: 신규 상태
- `crud.status.edit`: 편집 상태
- `crud.status.cu`: computed — add 또는 edit 중 높은 값
- `crud.status.title`: computed — add > NORMAL이면 "新增{title}", edit > NORMAL이면 "编辑{title}"

### CRUD 컴포넌트

```html
<rrOperation />                              <!-- 검색/리셋 버튼 -->
<crudOperation :permission="permission" />   <!-- 툴바: 신규/수정/삭제/다운로드 -->
<udOperation :data="scope.row" :permission="permission" />  <!-- 행별 수정/삭제 -->
<pagination />                               <!-- 페이지네이션 -->
```

### CRUD 생성자 옵션

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `tag` | `'default'` | 다중 CRUD 식별자 |
| `idField` | `'id'` | ID 필드명 |
| `title` | `''` | 제목 (알림, 다이얼로그) |
| `url` | `''` | API 엔드포인트 (GET 목록 조회용) |
| `query` | `{}` | 검색 쿼리 초기값 |
| `params` | `{}` | 고정 파라미터 |
| `sort` | `['id,desc']` | 정렬 |
| `time` | `0` | 로딩 후 테이블 표시 대기 (ms) |
| `crudMethod` | `{ add, del, edit, get }` | CUD API 메서드 |
| `optShow` | `{ add, edit, del, download, reset }` | 툴바 버튼 표시 |
| `queryOnPresenterCreated` | `true` | created 시 자동 조회 |
| `debug` | `false` | Hook 로그 출력 |

### CRUD HOOK 전체 목록

| Hook 상수 | 메서드명 | 호출 시점 |
|-----------|---------|----------|
| `beforeRefresh` | `beforeCrudRefresh` | 데이터 조회 전 (false → 중단) |
| `afterRefresh` | `afterCrudRefresh` | 데이터 조회 후 |
| `beforeToAdd` | `beforeCrudToAdd` | 신규 폼 열기 전 |
| `afterToAdd` | `afterCrudToAdd` | 신규 폼 연 후 |
| `beforeToEdit` | `beforeCrudToEdit` | 편집 폼 열기 전 |
| `afterToEdit` | `afterCrudToEdit` | 편집 폼 연 후 |
| `beforeToCU` | `beforeCrudToCU` | 신규/편집 폼 열기 전 (공통) |
| `afterToCU` | `afterCrudToCU` | 신규/편집 폼 연 후 (공통) |
| `beforeValidateCU` | `beforeCrudValidateCU` | 폼 유효성 검사 전 |
| `afterValidateCU` | `afterCrudValidateCU` | 폼 유효성 검사 후 (false → 제출 중단) |
| `beforeSubmit` | `beforeCrudSubmitCU` | API 제출 전 (false → 중단) |
| `afterSubmit` | `afterCrudSubmitCU` | API 제출 성공 후 |
| `afterAddError` | `afterCrudAddError` | 신규 등록 실패 후 |
| `afterEditError` | `afterCrudEditError` | 편집 실패 후 |
| `beforeDelete` | `beforeCrudDelete` | 삭제 API 호출 전 (false → 중단) |
| `afterDelete` | `afterCrudDelete` | 삭제 성공 후 |
| `beforeDeleteCancel` | `beforeCrudDeleteCancel` | 삭제 취소 전 |
| `afterDeleteCancel` | `afterCrudDeleteCancel` | 삭제 취소 후 |
| `beforeAddCancel` | `beforeCrudAddCancel` | 신규 취소 전 |
| `afterAddCancel` | `afterCrudAddCancel` | 신규 취소 후 |
| `beforeEditCancel` | `beforeCrudEditCancel` | 편집 취소 전 |
| `afterEditCancel` | `afterCrudEditCancel` | 편집 취소 후 |

## 도메인별 API 엔드포인트 전체 목록

### URL 접두사 규칙

| 접두사 | 사용 도메인 |
|--------|------------|
| `api/` | system, tools, maint, generator, monitor/log |
| `auth/` | login, monitor/online, buildMenus |

### login.js (인증)

| 메서드 | 엔드포인트 | 비고 |
|--------|-----------|------|
| POST | `auth/login` | 로그인 (username, password, code, uuid) |
| GET | `auth/info` | 사용자 정보 조회 |
| GET | `auth/code` | 캡차 이미지 조회 |
| DELETE | `auth/logout` | 로그아웃 |

### system/

| 파일 | 메서드 | 엔드포인트 | 비고 |
|------|--------|-----------|------|
| user.js | POST | `api/users` | 신규 |
| | PUT | `api/users` | 수정 |
| | DELETE | `api/users` | 일괄 삭제 (body: ids) |
| | PUT | `api/users/center` | 개인 정보 수정 |
| | POST | `api/users/updatePass/` | 비밀번호 변경 (RSA) |
| | PUT | `api/users/resetPwd` | 비밀번호 초기화 (관리자) |
| | POST | `api/users/updateEmail/{code}` | 이메일 변경 (RSA) |
| role.js | GET | `api/roles` | 목록 (CRUD) |
| | GET | `api/roles/all` | 전체 조회 (페이지네이션 없음) |
| | GET | `api/roles/{id}` | 단건 조회 |
| | GET | `api/roles/level` | 역할 레벨 조회 |
| | POST | `api/roles` | 신규 |
| | PUT | `api/roles` | 수정 |
| | PUT | `api/roles/menu` | 메뉴 할당 수정 |
| | DELETE | `api/roles` | 일괄 삭제 |
| menu.js | GET | `api/menus` | 목록 |
| | GET | `api/menus/lazy?pid=` | 지연 로딩 (트리) |
| | GET | `api/menus/build` | 사용자 메뉴 트리 빌드 |
| | GET | `api/menus/child?id=` | 하위 메뉴 조회 |
| | POST | `api/menus` | 신규 |
| | POST | `api/menus/superior` | 상위 메뉴 조회 (body: ids) |
| | PUT | `api/menus` | 수정 |
| | DELETE | `api/menus` | 일괄 삭제 |
| dept.js | GET | `api/dept` | 목록 |
| | POST | `api/dept` | 신규 |
| | POST | `api/dept/superior?exclude=` | 상위 부서 조회 |
| | PUT | `api/dept` | 수정 |
| | DELETE | `api/dept` | 일괄 삭제 |
| job.js | GET | `api/job` | 목록 |
| | POST | `api/job` | 신규 |
| | PUT | `api/job` | 수정 |
| | DELETE | `api/job` | 일괄 삭제 |
| dict.js | GET | `api/dict/all` | 전체 딕셔너리 |
| | POST | `api/dict` | 신규 |
| | PUT | `api/dict` | 수정 |
| | DELETE | `api/dict/` | 일괄 삭제 (trailing slash 있음) |
| dictDetail.js | GET | `api/dictDetail` | 목록 (dictName 파라미터) |
| | GET | `api/dictDetail/map` | 맵 조회 |
| | POST | `api/dictDetail` | 신규 |
| | PUT | `api/dictDetail` | 수정 |
| | DELETE | `api/dictDetail/{id}` | 단건 삭제 (URL에 id) |
| timing.js | POST | `api/jobs` | 신규 |
| | PUT | `api/jobs` | 수정 |
| | PUT | `api/jobs/{id}` | 일시정지/재개 토글 |
| | PUT | `api/jobs/exec/{id}` | 수동 실행 |
| | DELETE | `api/jobs` | 일괄 삭제 |
| code.js | POST | `api/code/resetEmail?email=` | 이메일 인증코드 발송 |
| | GET | `api/users/updatePass/{pass}` | 비밀번호 업데이트 |

### monitor/

| 파일 | 메서드 | 엔드포인트 | 비고 |
|------|--------|-----------|------|
| online.js | DELETE | `auth/online` | 온라인 사용자 강제 퇴출 |
| log.js | GET | `api/logs/error/{id}` | 에러 로그 상세 |
| | DELETE | `api/logs/del/error` | 전체 에러 로그 삭제 |
| | DELETE | `api/logs/del/info` | 전체 정보 로그 삭제 |

### tools/

| 파일 | 메서드 | 엔드포인트 | 비고 |
|------|--------|-----------|------|
| email.js | GET | `api/email` | 설정 조회 |
| | PUT | `api/email` | 설정 수정 |
| | POST | `api/email` | 이메일 발송 |
| alipay.js | GET | `api/aliPay` | 설정 조회 |
| | PUT | `api/aliPay` | 설정 수정 |
| | POST | `api/{url}` | 결제 (동적 URL) |
| localStorage.js | POST | `api/localStorage` | 신규 |
| | PUT | `api/localStorage` | 수정 |
| | DELETE | `api/localStorage/` | 일괄 삭제 |
| s3Storage.js | GET | `api/s3Storage/download/{id}` | 다운로드 |
| | DELETE | `api/s3Storage` | 일괄 삭제 |

### maint/

| 파일 | 메서드 | 엔드포인트 | 비고 |
|------|--------|-----------|------|
| app.js | POST | `api/app` | 신규 |
| | PUT | `api/app` | 수정 |
| | DELETE | `api/app` | 일괄 삭제 |
| database.js | POST | `api/database` | 신규 |
| | PUT | `api/database` | 수정 |
| | POST | `api/database/testConnect` | 연결 테스트 |
| | DELETE | `api/database` | 일괄 삭제 |
| deploy.js | POST | `api/deploy` | 신규 |
| | PUT | `api/deploy` | 수정 |
| | POST | `api/deploy/startServer` | 서비스 시작 |
| | POST | `api/deploy/stopServer` | 서비스 중지 |
| | POST | `api/deploy/serverStatus` | 서비스 상태 조회 |
| | DELETE | `api/deploy` | 일괄 삭제 |
| deployHistory.js | DELETE | `api/deployHistory` | 일괄 삭제 |
| | POST | `api/deploy/serverReduction` | 버전 롤백 |
| serverDeploy.js | POST | `api/serverDeploy` | 신규 |
| | PUT | `api/serverDeploy` | 수정 |
| | POST | `api/serverDeploy/testConnect` | 연결 테스트 |
| | DELETE | `api/serverDeploy` | 일괄 삭제 |
| connect.js | POST | `api/database/testConnect` | DB 연결 테스트 |
| | POST | `api/serverDeploy/testConnect` | 서버 연결 테스트 |

### generator/

| 파일 | 메서드 | 엔드포인트 | 비고 |
|------|--------|-----------|------|
| generator.js | GET | `api/generator/tables/all` | 전체 테이블 조회 |
| | POST | `api/generator/{tableName}/{type}` | 코드 생성 (type=2: Blob) |
| | PUT | `api/generator` | 설정 저장 |
| | POST | `api/generator/sync` | 테이블 동기화 |
| genConfig.js | GET | `api/genConfig/{tableName}` | 설정 조회 |
| | PUT | `api/genConfig` | 설정 수정 |

## WebSocket 사용처

프로젝트 전체에서 **단 한 곳**만 사용:

**파일**: `src/views/maint/deploy/deploy.vue` (배포 알림)

```js
const wsUri = (process.env.VUE_APP_WS_API === '/'
  ? '/' : (process.env.VUE_APP_WS_API + '/')) + 'webSocket/deploy'

this.websock = new WebSocket(wsUri)
// 수신: JSON.parse(e.data) → msgType: 'INFO' | 'ERROR' → $notify
// 송신: this.websock.send(agentData)
```

| 환경 | URL |
|------|-----|
| 개발 | `ws://localhost:8000/webSocket/deploy` |
| 프로덕션 | `wss://eladmin.vip/webSocket/deploy` |

## 공통 규칙 및 패턴

### Webpack 별칭

- `@` → `src/`
- `@crud` → `src/components/Crud`

### 동적 라우팅

- 정적 라우트: `src/router/routers.js` (login, 404, dashboard 등)
- 동적 라우트: 로그인 후 `buildMenus()` (GET `auth/buildMenus`) → `filterAsyncRouter()` → `router.addRoutes()`
- 컴포넌트 매핑: `'Layout'` → `@/layout/index`, `'ParentView'` → `@/components/ParentView`, 그 외 → `@/views/{component}` (lazy load)

### Vuex Store 구조

| 모듈 | namespaced | 역할 |
|------|-----------|------|
| `user.js` | false | 인증/사용자 (token, user, roles, loadMenus) |
| `permission.js` | false | 동적 라우팅/메뉴 (routers, addRouters, sidebarRouters) |
| `app.js` | **true** | UI 상태 (sidebar, device, size) |
| `settings.js` | **true** | 테마/레이아웃 설정 |
| `api.js` | false | 업로드 URL 중앙 관리 (읽기 전용) |
| `tagsView.js` | **true** | 탭 뷰 관리 (visitedViews, cachedViews) |

- `src/store/index.js`에서 `require.context('./modules', true, /\.js$/)`로 자동 로드
- 새 모듈: `store/modules/`에 파일만 생성하면 자동 등록

### 권한 관리

- `v-permission` 지시어: `src/utils/permission.js` (전역 등록)
- `this.checkPer(['admin', 'roles:add'])` — 전역 메서드
- `<permission :value="['admin']">` — 컴포넌트

### 네이밍 패턴

- API 파일: `src/api/{도메인}/{리소스}.js` (camelCase)
- 뷰 폴더: `src/views/{도메인}/` (api/ 폴더와 1:1 대응)
- 컴포넌트 폴더: `src/components/{PascalCase}/`
- Store 모듈: `src/store/modules/{camelCase}.js`

### 전역 설정 (src/settings.js)

| 키 | 값 | 용도 |
|----|----|------|
| `TokenKey` | `'ELADMIN-TOEKN'` | Cookie 토큰 키 (오타 있음, 수정 금지) |
| `tokenCookieExpires` | `1` | rememberMe 시 Cookie 만료일 (1일) |
| `timeout` | `1200000` | 요청 타임아웃 (ms, 20분) |
| `title` | `'ELADMIN'` | 브라우저 탭 제목 |
| `tagsView` | `true` | 탭 뷰 표시 |
| `fixedHeader` | `true` | 헤더 고정 |
| `sidebarLogo` | `true` | 사이드바 로고 표시 |

### ESLint 설정 (.eslintrc.js)

- **extends**: `plugin:vue/recommended`, `eslint:recommended`
- **parser**: `babel-eslint`
- **핵심 규칙**: `semi: never`, `quotes: single`, `indent: 2`, `comma-dangle: never`, `vue/name-property-casing: PascalCase`
- **프로덕션**: `no-debugger: error`

### Plop 코드 생성기

- `plopfile.js`에 `view`와 `component` 제너레이터가 정의됨
- `plop-templates/` 폴더가 필요하나 **현재 존재하지 않음** (불완전한 설정)
- `plop` 명령어는 실행 불가 상태

### layout/components/ 하위 구조

```
layout/components/
├── AppMain.vue          # <router-view> 래퍼 (keep-alive 포함)
├── index.js             # 배럴 export
├── Navbar.vue           # 상단 네비게이션 (사이드바 토글, 사용자 메뉴, 설정)
├── Settings/
│   └── index.vue        # 설정 패널 (테마, tagsView, header, footer 토글)
├── Sidebar/
│   ├── index.vue        # 사이드바 컨테이너
│   ├── SidebarItem.vue  # 재귀 메뉴 아이템
│   ├── Item.vue         # 아이콘 + 제목 렌더링
│   ├── Link.vue         # 라우터 링크 / 외부 링크 분기
│   ├── Logo.vue         # 사이드바 로고
│   └── FixiOSBug.js     # iOS 서브메뉴 버그 수정 mixin
└── TagsView/
    ├── index.vue        # 탭 뷰 (우클릭 컨텍스트 메뉴 포함)
    └── ScrollPane.vue   # 가로 스크롤 래퍼
```

### 파일 업로드 패턴

업로드 URL은 `src/store/modules/api.js`에서 중앙 관리 (`mapGetters`로 접근):

| Getter | URL |
|--------|-----|
| `deployUploadApi` | `{baseUrl}/api/deploy/upload` |
| `databaseUploadApi` | `{baseUrl}/api/database/upload` |
| `imagesUploadApi` | `{baseUrl}/api/localStorage/pictures` |
| `updateAvatarApi` | `{baseUrl}/api/users/updateAvatar` |
| `s3UploadApi` | `{baseUrl}/api/s3Storage` |
| `fileUploadApi` | `{baseUrl}/api/localStorage` |

**주의**: `src/utils/upload.js`는 `request.js` 인스턴스가 아닌 **raw axios**를 사용 (multipart/form-data 전송을 위해 `application/json` 강제 우회). 따라서 `response.data`가 아닌 `response` 전체를 반환.

### API 함수 작성 컨벤션

```js
import request from '@/utils/request'

export function add(data) {
  return request({ url: 'api/{resource}', method: 'post', data })
}
export function edit(data) {
  return request({ url: 'api/{resource}', method: 'put', data })
}
export function del(ids) {
  return request({ url: 'api/{resource}', method: 'delete', data: ids })
}
export default { add, edit, del }  // default export 필수 (crudMethod 스프레드용)
```

### 응답 데이터 구조 (서버 → 클라이언트)

```js
// 페이지네이션 응답
{ content: [...], totalElements: 100 }

// CRUD 인스턴스 자동 처리:
// crud.page.total = data.totalElements
// crud.data = data.content
```

### 페이지네이션 없이 전체 조회 패턴

드롭다운/셀렉트 박스용: `size: 9999`로 전체 데이터 조회 (예: `getAllJob()`, `getDictMap()`)

## 하지 말아야 할 것 (종합)

### 치명적 (시스템 전체 영향)

1. **`user.js`, `permission.js`에 `namespaced: true` 추가** — 인증/라우팅 시스템 전체 멈춤
2. **`TokenKey`를 `ELADMIN-TOKEN`으로 수정** — 기존 사용자 세션 모두 끊김 (`ELADMIN-TOEKN`이 원본 오타)
3. **`loadView()`의 동적 import 경로 변경** — `require([@/views/${view}], resolve)` 패턴. 모든 동적 라우트가 깨짐
4. **`location.reload()`를 함부로 호출** — 라우터 인스턴스 재생성. 401 로그아웃 시에만 사용

### CRUD 관련

5. **새 페이지에 `src/mixins/crud.js` 사용** — `src/components/Crud/crud.js`의 인스턴스 방식 사용할 것
6. **`cruds()` 대신 `data()`에서 CRUD 인스턴스 생성** — `beforeCreate` 훅이 `$options.cruds`를 읽음
7. **mixin 순서 변경** — 반드시 `[presenter(), header(), form(defaultForm), crud()]`
8. **`defaultForm` 생략** — `form()` mixin에 반드시 전달
9. **`$refs.form`을 수동 validate** — `crud.submitCU()`가 자동 호출
10. **form에 `createTime`, `updateTime`, `createBy`, `updateBy` 포함** — `resetForm()`이 자동 삭제
11. **query 직접 조작 후 수동 로드** — `crud.toQuery()` 사용할 것
12. **테이블 데이터 직접 수정** — `crud.refresh()` 사용할 것
13. **다이얼로그 상태 직접 조작** — `crud.toAdd()`, `crud.toEdit()`, `crud.cancelCU()` 사용할 것

### API/HTTP 관련

14. **Authorization 헤더에 `Bearer ` 수동 추가** — 토큰에 이미 포함됨
15. **`response.data` 대신 `response` 사용** — 인터셉터에서 이미 `response.data`만 반환 (upload.js 예외)
16. **`Content-Type` 수동 설정** — 인터셉터에서 `application/json` 자동 설정
17. **페이지네이션 파라미터 수동 조합** — `getQueryParams()`가 자동 조립
18. **`sort`를 문자열로 전송** — `['id,desc']` 배열 형태 유지
19. **default export에 CRUD 기본 함수 외 포함** — `editUser`, `updatePass` 등은 named export로만
20. **URL에 trailing slash 추가** — 새 API에서는 trailing slash 없이 통일
21. **GET 요청에서 쿼리 문자열 수동 조합** — `params` 객체 사용 권장
22. **업로드 URL을 API 파일에 하드코딩** — `store/modules/api.js`에서 중앙 관리

### Store 관련

23. **`api.js`에 mutation/action 추가** — 읽기 전용 URL 설정 저장소
24. **`tagsView`를 직접 mutation** — 항상 action 사용 (addView, delView 등)
25. **store 모듈에서 직접 router import** — 라우팅 로직은 `router/index.js`의 `loadMenus()`에서 처리
26. **Cookie에 민감 정보 저장** — rememberMe의 username/password 저장은 레거시 패턴

### UI/기타

27. **전역 스타일 직접 수정** — `scoped` + `::v-deep` 사용
28. **`v-permission` 없이 버튼 추가** — 모든 액션 버튼에 권한 적용
29. **`checkPer` 없이 작업 컬럼 표시** — `v-if="checkPer([...])"` 필수
30. **딕셔너리 데이터를 수동 로드** — `dicts: ['dict_name']` 옵션으로 자동 로드
31. **동적 라우트에서 component를 직접 import** — 백엔드 `component` 문자열이 자동 변환됨
32. **`router.addRoutes()` 전에 catch-all 수동 추가** — `loadMenus()`에서 자동 추가
33. **`size: 9999` 패턴 남용** — 드롭다운용 전체 조회에만 사용
34. **`src/utils/upload.js`에서 `request.js` 인스턴스 사용 시도** — 의도적으로 raw axios 사용
