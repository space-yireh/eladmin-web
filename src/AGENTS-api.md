# AGENTS-api.md -- src/api/ 폴더 심층 분석

> **전제**: Axios 인스턴스(`request.js`), 기본 API 함수 패턴(`add/edit/del/get + default export`), `initData()`, `download()` 공통 함수는 AGENTS-common.md에 기술됨. 여기서는 중복하지 않음.

---

## 1. api/ 폴더 구조와 파일별 담당 도메인

```
src/api/
├── data.js                    # 공통: initData(), download()
├── login.js                   # 인증: login, logout, getInfo, getCodeImg
├── system/                    # 시스템 관리
│   ├── user.js                # 사용자 (api/users)
│   ├── role.js                # 역할 (api/roles)
│   ├── menu.js                # 메뉴 (api/menus)
│   ├── dept.js                # 부서 (api/dept)
│   ├── job.js                 #岗位 (api/job)
│   ├── dict.js                # 딕셔너리 (api/dict)
│   ├── dictDetail.js          # 딕셔너리 상세 (api/dictDetail)
│   ├── timing.js              #定时任务 (api/jobs)
│   └── code.js                # 이메일 인증코드 (api/code)
├── monitor/                   # 모니터링
│   ├── online.js              # 온라인 사용자 (auth/online)
│   └── log.js                 # 로그 (api/logs)
├── tools/                     # 도구
│   ├── email.js               # 이메일 설정 (api/email)
│   ├── alipay.js              # 알리페이 (api/aliPay)
│   ├── localStorage.js        # 로컬 저장소 (api/localStorage)
│   └── s3Storage.js           # S3 저장소 (api/s3Storage)
├── maint/                     # 유지보수
│   ├── app.js                 # 앱 관리 (api/app)
│   ├── database.js            # DB 관리 (api/database)
│   ├── deploy.js              # 배포 (api/deploy)
│   ├── deployHistory.js       # 배포 이력 (api/deployHistory)
│   ├── serverDeploy.js        # 서버 배포 (api/serverDeploy)
│   └── connect.js             # 연결 테스트 (api/database/testConnect, api/serverDeploy/testConnect)
└── generator/                 # 코드 생성기
    ├── generator.js           # 생성기 (api/generator)
    └── genConfig.js           # 생성 설정 (api/genConfig)
```

### 파일별 제공 함수 요약

| 파일 | 함수 | default export |
|------|------|----------------|
| **login.js** | `login`, `logout`, `getInfo`, `getCodeImg` | 없음 |
| **system/user.js** | `add`, `edit`, `del`, `editUser`, `updatePass`, `resetPwd`, `updateEmail` | `{ add, edit, del, resetPwd }` |
| **system/role.js** | `getAll`, `add`, `get`, `getLevel`, `del`, `edit`, `editMenu` | `{ add, edit, del, get, editMenu, getLevel }` |
| **system/menu.js** | `getMenusTree`, `getMenus`, `getMenuSuperior`, `getChild`, `buildMenus`, `add`, `del`, `edit` | `{ add, edit, del, getMenusTree, getMenuSuperior, getMenus, getChild }` |
| **system/dept.js** | `getDepts`, `getDeptSuperior`, `add`, `del`, `edit` | `{ add, edit, del, getDepts, getDeptSuperior }` |
| **system/job.js** | `getAllJob`, `add`, `del`, `edit` | `{ add, edit, del }` |
| **system/dict.js** | `getDicts`, `add`, `del`, `edit` | `{ add, edit, del }` |
| **system/dictDetail.js** | `get`, `getDictMap`, `add`, `del`, `edit` | `{ add, edit, del }` |
| **system/timing.js** | `add`, `del`, `edit`, `updateIsPause`, `execution` | `{ del, updateIsPause, execution, add, edit }` |
| **system/code.js** | `resetEmail`, `updatePass` | 없음 |
| **monitor/online.js** | `del` | 없음 |
| **monitor/log.js** | `getErrDetail`, `delAllError`, `delAllInfo` | 없음 |
| **tools/email.js** | `get`, `update`, `send` | 없음 |
| **tools/alipay.js** | `get`, `update`, `toAliPay` | 없음 |
| **tools/localStorage.js** | `add`, `del`, `edit` | `{ add, edit, del }` |
| **tools/s3Storage.js** | `download`, `del` | `{ del, download }` |
| **maint/app.js** | `add`, `del`, `edit` | `{ add, edit, del }` |
| **maint/database.js** | `add`, `del`, `edit`, `testDbConnection` | `{ add, edit, del, testDbConnection }` |
| **maint/deploy.js** | `add`, `del`, `edit`, `getApps`, `getServers`, `startServer`, `stopServer`, `serverStatus` | `{ add, edit, del, stopServer, serverStatus, startServer, getServers, getApps }` |
| **maint/deployHistory.js** | `del`, `reducte` | 없음 |
| **maint/serverDeploy.js** | `add`, `del`, `edit` | `{ add, edit, del }` |
| **maint/connect.js** | `testDbConnect`, `testServerConnect` | 없음 |
| **generator/generator.js** | `getAllTable`, `generator`, `save`, `sync` | 없음 |
| **generator/genConfig.js** | `get`, `update` | 없음 |

---

## 2. 도메인별 API 엔드포인트 패턴

### URL 접두사 규칙

| 접두사 | 사용 도메인 | 비고 |
|--------|------------|------|
| `api/` | system, tools, maint, generator, monitor/log | 대부분의 CRUD API |
| `auth/` | login, monitor/online | 인증/인가 관련 |

### 전체 엔드포인트 맵

#### system/

| 파일 | 메서드 | 엔드포인트 | 비고 |
|------|--------|-----------|------|
| user.js | POST | `api/users` | 신규 |
| | PUT | `api/users` | 수정 |
| | DELETE | `api/users` | 일괄 삭제 (body: ids 배열) |
| | PUT | `api/users/center` | 개인 정보 수정 |
| | POST | `api/users/updatePass/` | 비밀번호 변경 (RSA 암호화) |
| | PUT | `api/users/resetPwd` | 비밀번호 초기화 (관리자) |
| | POST | `api/users/updateEmail/{code}` | 이메일 변경 (RSA 암호화) |
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
| | POST | `api/menus/superior` | 상위 메뉴 조회 (body: ids 배열) |
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
| | DELETE | `api/dictDetail/{id}` | 단건 삭제 (URL에 id 포함) |
| timing.js | POST | `api/jobs` | 신규 |
| | PUT | `api/jobs` | 수정 |
| | PUT | `api/jobs/{id}` | 일시정지/재개 토글 |
| | PUT | `api/jobs/exec/{id}` | 수동 실행 |
| | DELETE | `api/jobs` | 일괄 삭제 |
| code.js | POST | `api/code/resetEmail?email=` | 이메일 인증코드 발송 |
| | GET | `api/users/updatePass/{pass}` | 비밀번호 업데이트 (인증코드) |

#### monitor/

| 파일 | 메서드 | 엔드포인트 | 비고 |
|------|--------|-----------|------|
| online.js | DELETE | `auth/online` | 온라인 사용자 강제 퇴출 |
| log.js | GET | `api/logs/error/{id}` | 에러 로그 상세 |
| | DELETE | `api/logs/del/error` | 전체 에러 로그 삭제 |
| | DELETE | `api/logs/del/info` | 전체 정보 로그 삭제 |

#### tools/

| 파일 | 메서드 | 엔드포인트 | 비고 |
|------|--------|-----------|------|
| email.js | GET | `api/email` | 이메일 설정 조회 |
| | PUT | `api/email` | 이메일 설정 수정 |
| | POST | `api/email` | 이메일 발송 |
| alipay.js | GET | `api/aliPay` | 설정 조회 |
| | PUT | `api/aliPay` | 설정 수정 |
| | POST | `api/{url}` | 결제 (동적 URL) |
| localStorage.js | POST | `api/localStorage` | 신규 |
| | PUT | `api/localStorage` | 수정 |
| | DELETE | `api/localStorage/` | 일괄 삭제 (trailing slash) |
| s3Storage.js | GET | `api/s3Storage/download/{id}` | 다운로드 |
| | DELETE | `api/s3Storage` | 일괄 삭제 |

#### maint/

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

#### generator/

| 파일 | 메서드 | 엔드포인트 | 비고 |
|------|--------|-----------|------|
| generator.js | GET | `api/generator/tables/all` | 전체 테이블 조회 |
| | POST | `api/generator/{tableName}/{type}` | 코드 생성 (type=2: Blob 다운로드) |
| | PUT | `api/generator` | 설정 저장 |
| | POST | `api/generator/sync` | 테이블 동기화 |
| genConfig.js | GET | `api/genConfig/{tableName}` | 설정 조회 |
| | PUT | `api/genConfig` | 설정 수정 |

#### login.js

| 메서드 | 엔드포인트 | 비고 |
|--------|-----------|------|
| POST | `auth/login` | 로그인 (username, password, code, uuid) |
| GET | `auth/info` | 사용자 정보 조회 |
| GET | `auth/code` | 캡차 이미지 조회 |
| DELETE | `auth/logout` | 로그아웃 |

---

## 3. 페이지네이션 파라미터 전달 방식

### CRUD 인스턴스 자동 생성 (신형)

CRUD 인스턴스의 `getQueryParams()`가 자동으로 파라미터를 조립합니다:

```js
// src/components/Crud/crud.js 내부
getQueryParams() {
  return {
    page: crud.page.page - 1,  // 1-based → 0-based 변환
    size: crud.page.size,      // 기본값: 10
    sort: crud.sort,           // 기본값: ['id,desc']
    ...crud.query,             // 검색 조건
    ...crud.params             // 고정 파라미터
  }
}
```

### 전송 형태

`initData()`에서 `qs.stringify(params, { indices: false })`로 직렬화:

```
GET api/roles?page=0&size=10&sort=id,desc&blurry=admin
```

- **page**: 0-based (서버에 0부터 전달)
- **size**: 페이지 크기 (기본 10)
- **sort**: `['id,desc']` → `sort=id,desc` (다중 필드: `sort=id,desc&sort=createTime,asc`)
- **검색 조건**: `crud.query` 객체가 그대로展開됨

### 응답 데이터 구조 (서버 → 클라이언트)

```js
{
  content: [...],       // 현재 페이지 데이터 배열
  totalElements: 100    // 전체 데이터 수
}
```

CRUD 인스턴스가 이를 자동으로 처리:
```js
crud.page.total = data.totalElements
crud.data = data.content
```

### 페이지네이션 없이 전체 조회하는 패턴

일부 API는 `size: 9999`로 전체 데이터를 가져옵니다:

```js
// system/job.js - getAllJob()
const params = { page: 0, size: 9999, enabled: true }

// system/dictDetail.js - get(), getDictMap()
const params = { dictName, page: 0, size: 9999 }
```

이 패턴은 드롭다운/셀렉트 박스용 데이터 로딩에 사용됩니다.

---

## 4. 파일 업로드 API 패턴

### 업로드 URL 중앙 관리 (src/store/modules/api.js)

모든 업로드 URL은 Vuex `api` 모듈에서 관리되며, `mapGetters`로 접근합니다:

| Getter | URL | 용도 |
|--------|-----|------|
| `deployUploadApi` | `{baseUrl}/api/deploy/upload` | 배포 패키지 업로드 |
| `databaseUploadApi` | `{baseUrl}/api/database/upload` | SQL 스크립트 업로드 |
| `imagesUploadApi` | `{baseUrl}/api/localStorage/pictures` | 이미지 업로드 |
| `updateAvatarApi` | `{baseUrl}/api/users/updateAvatar` | 아바타 변경 |
| `s3UploadApi` | `{baseUrl}/api/s3Storage` | S3 파일 업로드 |
| `fileUploadApi` | `{baseUrl}/api/localStorage` | 일반 파일 업로드 |

### 패턴 A: el-upload + Vuex URL (드래그 앤 드롭)

```html
<el-upload
  :action="deployUploadApi"
  :data="deployInfo"
  :headers="headers"
  :on-success="handleSuccess"
  drag
>
```

```js
computed: {
  ...mapGetters(['deployUploadApi'])
},
data() {
  return {
    headers: { Authorization: getToken() },
    deployInfo: { /* 추가 폼 데이터 */ }
  }
}
```

사용처: `views/maint/deploy/deploy.vue`, `views/maint/database/execute.vue`

### 패턴 B: el-upload + 수동 제출 (CRUD 폼 내부)

```html
<el-upload
  ref="upload"
  :limit="1"
  :auto-upload="false"
  :before-upload="beforeUpload"
  :headers="headers"
  :on-success="handleSuccess"
  :action="fileUploadApi + '?name=' + form.name"
>
```

```js
methods: {
  upload() {
    this.$refs.upload.submit()
  },
  handleSuccess(response, file, fileList) {
    this.crud.notify('업로드 성공', CRUD.NOTIFICATION_TYPE.SUCCESS)
    this.$refs.upload.clearFiles()
    this.crud.toQuery()
  }
}
```

사용처: `views/tools/storage/local/index.vue`

### 패턴 C: 프로그래밍 방식 업로드 (에디터 이미지)

```js
import { upload } from '@/utils/upload'

// WangEditor, MavonEditor 등 서드파티 에디터와 연동
upload(this.imagesUploadApi, file).then(res => {
  const data = res.data
  const url = this.baseApi + '/file/' + data.type + '/' + data.realName
  insert(url)
})
```

**upload 헬퍼** (`src/utils/upload.js`):
```js
export function upload(api, file) {
  var data = new FormData()
  data.append('file', file)
  const config = { headers: { 'Authorization': getToken() } }
  return axios.post(api, data, config)
}
```

**주의**: 이 헬퍼는 `request.js` 인스턴스가 아닌 **raw axios**를 사용합니다. 따라서 응답 인터셉터가 적용되지 않아 `response.data`가 아닌 `response` 전체를 반환합니다.

사용처: `views/tools/email/send.vue`, `views/components/MarkDown.vue`

---

## 5. WebSocket 연결 방식

프로젝트 전체에서 **단一处**만 사용합니다:

**파일**: `src/views/maint/deploy/deploy.vue`

```js
// WebSocket URL 구성
const wsUri = (process.env.VUE_APP_WS_API === '/'
  ? '/'
  : (process.env.VUE_APP_WS_API + '/')) + 'webSocket/deploy'

// 네이티브 WebSocket 생성
this.websock = new WebSocket(wsUri)
this.websock.onerror = this.webSocketOnError
this.websock.onmessage = this.webSocketOnMessage

// 수신 메시지 처리 (JSON)
webSocketOnMessage(e) {
  const data = JSON.parse(e.data)
  if (data.msgType === 'INFO') {
    this.$notify({ title: data.msg, type: 'success', duration: 2500 })
  } else if (data.msgType === 'ERROR') {
    this.$notify({ title: data.msg, type: 'error', duration: 2500 })
  }
}

// 송신
webSocketSend(agentData) {
  this.websock.send(agentData)
}
```

| 환경 변수 | 값 | 비고 |
|----------|-----|------|
| `VUE_APP_WS_API` (dev) | `ws://localhost:8000` | 개발 |
| `VUE_APP_WS_API` (prod) | `wss://eladmin.vip` | 프로덕션 |

---

## 6. CRUD 인스턴스와 API 연결 방식

### 연결 구조

```
View 컴포넌트
  ├── cruds()에서 CRUD 인스턴스 생성
  │   ├── url: 'api/roles'          → GET 목록 조회에 사용
  │   └── crudMethod: { ...crudRole } → POST/PUT/DELETE에 사용
  │
  ├── CRUD.refresh()
  │   └── initData(url, getQueryParams())  → GET api/roles?page=0&size=10&sort=id,desc
  │
  ├── CRUD.doAdd()
  │   └── crudMethod.add(form)       → POST api/roles (body: form)
  │
  ├── CRUD.doEdit()
  │   └── crudMethod.edit(form)      → PUT api/roles (body: form)
  │
  ├── CRUD.doDelete(data)
  │   └── crudMethod.del(ids)        → DELETE api/roles (body: [id1, id2])
  │
  └── CRUD.doExport()
      └── download(url + '/download', getQueryParams()) → GET api/roles/download (Blob)
```

### 두 가지 방식 비교

| 방식 | 사용 시기 | 예시 |
|------|----------|------|
| `url` + `initData` | CRUD 인스턴스가 자동으로 목록 조회 | 대부분의 CRUD 페이지 |
| `crudMethod` | CRUD 인스턴스가 CUD 작업 시 호출 | add/edit/del |
| `initData` 직접 호출 | CRUD를 사용하지 않는 특수 페이지 | `views/monitor/server/index.vue` (서버 모니터링 폴링) |

### crudMethod 바인딩 규칙

```js
// 1. API 모듈 import
import crudUser from '@/api/system/user'

// 2. cruds()에서 스프레드로 바인딩
cruds() {
  return CRUD({
    title: '사용자',
    url: 'api/users',
    crudMethod: { ...crudUser }  // default export 객체가展開됨
  })
}
```

**default export에 포함된 함수만 crudMethod로 연결됩니다.** 추가 함수(`editUser`, `updatePass` 등)는 뷰에서 직접 import하여 호출합니다:

```js
import crudUser, { editUser, updatePass } from '@/api/system/user'

// crudMethod에는 add, edit, del, resetPwd만 연결됨
// editUser, updatePass는 메서드에서 직접 호출
methods: {
  changePassword() {
    updatePass({ oldPass: '...', newPass: '...' })
  }
}
```

### 부분 crudMethod (읽기 전용 페이지)

```js
import { del } from '@/api/maint/deployHistory'

cruds() {
  return CRUD({
    title: '배포 이력',
    url: 'api/deployHistory',
    crudMethod: { del },  // del만 제공
    optShow: { add: false, edit: false }  // 툴바에서 신규/수정 버튼 숨김
  })
}
```

---

## 7. 로그인/인증 API (src/api/login.js)

| 함수 | 엔드포인트 | 파라미터 | 반환값 |
|------|-----------|---------|--------|
| `login(username, password, code, uuid)` | POST `auth/login` | `{ username, password, code, uuid }` | `{ token, user }` |
| `getInfo()` | GET `auth/info` | 없음 (토큰으로 인증) | 사용자 정보 |
| `getCodeImg()` | GET `auth/code` | 없음 | `{ img, uuid, code }` (캡차) |
| `logout()` | DELETE `auth/logout` | 없음 (토큰으로 인증) | 없음 |

**로그인 흐름**:
1. `getCodeImg()` → 캡차 이미지 + uuid 수신
2. 사용자가 아이디/비번/캡차 입력
3. `login(username, password, code, uuid)` → 토큰 수신
4. `setToken(token)` → Cookie 저장
5. `getInfo()` → 사용자 정보 + roles 수신
6. `buildMenus()` (from `api/system/menu.js`) → 동적 메뉴 로드

---

## 8. 특수 API 패턴

### RSA 암호화 (비밀번호 전송)

`src/utils/rsaEncrypt.js`의 `encrypt()` 함수로 비밀번호를 RSA 공개키로 암호화:

```js
// system/user.js
import { encrypt } from '@/utils/rsaEncrypt'

export function updatePass(user) {
  const data = {
    oldPass: encrypt(user.oldPass),
    newPass: encrypt(user.newPass)
  }
  return request({ url: 'api/users/updatePass/', method: 'post', data })
}
```

사용처: `updatePass()`, `updateEmail()` -- 비밀번호가 포함된 요청만 암호화

### 단건 삭제 vs 일괄 삭제

| 방식 | 엔드포인트 패턴 | 사용 파일 |
|------|---------------|----------|
| **일괄 삭제** (body: ids 배열) | DELETE `api/{resource}` | 대부분의 파일 |
| **단건 삭제** (URL에 id) | DELETE `api/{resource}/{id}` | `dictDetail.js` |

### 동적 URL 패턴

```js
// generator/generator.js - tableName과 type이 동적으로 결정
export function generator(tableName, type) {
  return request({
    url: 'api/generator/' + tableName + '/' + type,
    method: 'post',
    responseType: type === 2 ? 'blob' : ''  // type=2일 때 Blob 다운로드
  })
}

// tools/alipay.js - URL 자체가 동적
export function toAliPay(url, data) {
  return request({ url: 'api/' + url, data, method: 'post' })
}
```

### GET 요청에서 params 객체 vs 쿼리 문자열 수동 조합

```js
// params 객체 사용 (axios가 자동 직렬화)
export function getMenus(params) {
  return request({ url: 'api/menus', method: 'get', params })
}

// 수동 쿼리 문자열 조합
export function getMenusTree(pid) {
  return request({ url: 'api/menus/lazy?pid=' + pid, method: 'get' })
}
```

두 방식이 혼재되어 있습니다. 새 API 작성 시 `params` 객체 방식을 권장합니다.

### trailing slash 불일치

일부 파일에서 DELETE URL에 trailing slash가 있습니다:

```js
// dict.js
url: 'api/dict/'    // trailing slash 있음

// localStorage.js
url: 'api/localStorage/'  // trailing slash 있음

// role.js
url: 'api/roles'    // trailing slash 없음
```

백엔드에서 둘 다 허용하므로 기능적 문제는 없으나, **새 API에서는 trailing slash 없이 통일**하는 것을 권장합니다.

---

## 9. 새 API 파일 작성 템플릿

### 기본 CRUD API 파일

```js
import request from '@/utils/request'

export function add(data) {
  return request({
    url: 'api/{resource}',
    method: 'post',
    data
  })
}

export function del(ids) {
  return request({
    url: 'api/{resource}',
    method: 'delete',
    data: ids
  })
}

export function edit(data) {
  return request({
    url: 'api/{resource}',
    method: 'put',
    data
  })
}

export function get(id) {
  return request({
    url: 'api/{resource}/' + id,
    method: 'get'
  })
}

export default { add, edit, del, get }
```

### 특수 함수가 포함된 API 파일

```js
import request from '@/utils/request'

export function add(data) {
  return request({ url: 'api/{resource}', method: 'post', data })
}

export function del(ids) {
  return request({ url: 'api/{resource}', method: 'delete', data: ids })
}

export function edit(data) {
  return request({ url: 'api/{resource}', method: 'put', data })
}

export function getAll() {
  return request({ url: 'api/{resource}/all', method: 'get' })
}

export function doAction(id) {
  return request({ url: 'api/{resource}/action/' + id, method: 'put' })
}

export default { add, edit, del, getAll, doAction }
```

---

## 10. 새 도메인 API 파일 추가 시 체크리스트

1. **폴더 위치**: `src/api/{도메인}/{리소스}.js` (camelCase 파일명)
2. **import**: `import request from '@/utils/request'` (공통 Axios 인스턴스)
3. **함수 네이밍**: `add`, `edit`, `del`, `get` 기본 4종 + 도메인 특수 함수
4. **default export**: CRUD 인스턴스와 연동할 파일이면 반드시 `{ add, edit, del }` 형태의 default export 포함
5. **URL 접두사**: CRUD API는 `api/`, 인증 관련은 `auth/`
6. **URL 형식**: `api/{resource}` (trailing slash 없음, 소문자 또는 camelCase)
7. **DELETE 방식**: 일괄 삭제는 body에 ids 배열, 단건 삭제는 URL에 `/{id}`
8. **뷰 폴더 대응**: `src/views/{도메인}/`에 1:1 대응 폴더 생성
9. **업로드가 필요한 경우**: `src/store/modules/api.js`에 업로드 URL 추가 + `src/store/getters.js`에 getter 등록
10. **RSA 암호화 필요 시**: `src/utils/rsaEncrypt.js`의 `encrypt()` 사용 (비밀번호만)

---

## 11. 하지 말아야 할 것

1. **`src/utils/upload.js`에서 `request.js` 인스턴스를 사용하지 말 것** -- upload 헬퍼는 raw axios를 사용함. 이는 의도적인 설계로, `Content-Type: application/json` 강제 설정을 우회하여 `multipart/form-data`를 전송하기 위함. 이 헬퍼를 사용할 때는 `response.data`가 아닌 `response` 전체가 반환됨에 주의

2. **업로드 URL을 API 파일에 하드코딩하지 말 것** -- 모든 업로드 URL은 `src/store/modules/api.js`에서 중앙 관리. `mapGetters`로 접근할 것

3. **`Content-Type: multipart/form-data`를 수동으로 설정하지 말 것** -- `FormData` 객체를 사용하면 axios가 자동으로 `Content-Type`과 `boundary`를 설정함

4. **페이지네이션 파라미터를 수동으로 조합하지 말 것** -- CRUD 인스턴스가 `getQueryParams()`에서 자동으로 `page`(0-based), `size`, `sort`를 조립함

5. **`sort` 파라미터를 문자열로 보내지 말 것** -- 기본값은 `['id,desc']` 배열. `qs.stringify`의 `{ indices: false }` 옵션으로 `sort=id,desc` 형태로 직렬화됨

6. **`size: 9999` 패턴을 남용하지 말 것** -- 드롭다운/셀렉트 박스용 전체 조회에만 사용. 목록 페이지에서는 반드시 페이지네이션 적용

7. **default export에 CRUD 기본 4종 외의 함수를 포함하지 말 것** -- `editUser`, `updatePass` 같은 특수 함수는 named export로만 제공. default export에는 CRUD 인스턴스가 사용할 `add`, `edit`, `del`(+필요 시 `get`)만 포함

8. **URL에 trailing slash를 추가하지 말 것** -- 기존 코드에 `api/dict/`, `api/localStorage/` 등 불일치가 있으나, 새 API에서는 trailing slash 없이 통일

9. **GET 요청에서 쿼리 문자열을 수동으로 조합하지 말 것** -- `url: 'api/menus/lazy?pid=' + pid` 같은 수동 조합이 일부 존재하나, 새 API에서는 `params` 객체를 사용할 것: `request({ url: 'api/menus/lazy', method: 'get', params: { pid } })`

10. **WebSocket을 새 기능에 쉽게 도입하지 말 것** -- 현재 프로젝트에서 WebSocket은 배포 알림 한 곳에만 사용됨. 새 WebSocket 연결 추가 시 `VUE_APP_WS_API` 환경 변수를 사용하고, 에러 핸들링과 재연결 로직을 반드시 구현할 것
