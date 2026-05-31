# AGENTS-components.md — Views & Components 심층 분석

## 1. views/ 폴더 구조

### 도메인별 폴더 구성

```
src/views/
├── system/          # 시스템 관리 (7개 모듈)
│   ├── user/        # 사용자 관리
│   ├── role/        # 역할 관리
│   ├── menu/        # 메뉴 관리
│   ├── dept/        # 부서 관리
│   ├── job/         #岗位 관리
│   ├── dict/        # 데이터 딕셔너리
│   └── timing/      # 스케줄링 작업
├── monitor/         # 모니터링 (4개 모듈)
│   ├── log/         # 로그 조회
│   ├── online/      # 온라인 사용자
│   ├── server/      # 서버 모니터링
│   └── sql/         # SQL 모니터링
├── tools/           # 도구 (3개 모듈)
│   ├── aliPay/      # 알리페이
│   ├── email/       # 이메일
│   └── storage/     # 스토리지
├── maint/           # 유지보수 (5개 모듈)
│   ├── app/         # 애플리케이션
│   ├── database/    # 데이터베이스
│   ├── deploy/      # 배포
│   ├── deployHistory/ # 배포 이력
│   └── server/      # 서버
├── generator/       # 코드 생성기
├── dashboard/       # 대시보드 위젯
├── features/        # 404, 401 등
├── components/      # 뷰 전용 컴포넌트
├── nested/          # 중첩 라우트 예제
├── home.vue         # 대시보드 홈
└── login.vue        # 로그인 페이지
```

### api/ 폴더와 1:1 대응 관계

| api/ 폴더 | views/ 폴더 | 매핑 여부 |
|-----------|-------------|-----------|
| api/system/ | views/system/ | ✓ 완전 일치 |
| api/monitor/ | views/monitor/ | ✓ 완전 일치 |
| api/tools/ | views/tools/ | ✓ 완전 일치 |
| api/maint/ | views/maint/ | ✓ 완전 일치 |
| api/generator/ | views/generator/ | ✓ 완전 일치 |

**규칙**: 새 도메인 추가 시 `api/{도메인}/`과 `views/{도메인}/`을 동시에 생성

---

## 2. 표준 CRUD 페이지 패턴

### 2.1 완전한 템플릿 (복사해서 사용 가능)

```vue
<template>
  <div class="app-container">
    <!-- 도구바 -->
    <div class="head-container">
      <!-- 검색 영역 -->
      <div v-if="crud.props.searchToggle">
        <el-input
          v-model="query.blurry"
          clearable
          size="small"
          placeholder="검색어 입력"
          style="width: 200px;"
          class="filter-item"
          @keyup.enter.native="crud.toQuery"
        />
        <date-range-picker v-model="query.createTime" class="date-item" />
        <el-select
          v-model="query.enabled"
          clearable
          size="small"
          placeholder="상태"
          class="filter-item"
          style="width: 90px"
          @change="crud.toQuery"
        >
          <el-option
            v-for="item in enabledTypeOptions"
            :key="item.key"
            :label="item.display_name"
            :value="item.key"
          />
        </el-select>
        <rrOperation />
      </div>
      <!-- CRUD 버튼 -->
      <crudOperation :permission="permission" />
    </div>

    <!-- 폼 다이얼로그 -->
    <el-dialog
      append-to-body
      :close-on-click-modal="false"
      :before-close="crud.cancelCU"
      :visible.sync="crud.status.cu > 0"
      :title="crud.status.title"
      width="500px"
    >
      <el-form
        ref="form"
        :inline="true"
        :model="form"
        :rules="rules"
        size="small"
        label-width="80px"
      >
        <el-form-item label="이름" prop="name">
          <el-input v-model="form.name" style="width: 370px;" />
        </el-form-item>
        <el-form-item label="상태" prop="enabled">
          <el-radio-group v-model="form.enabled">
            <el-radio
              v-for="item in dict.status_dict"
              :key="item.id"
              :label="item.value"
            >{{ item.label }}</el-radio>
          </el-radio-group>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="text" @click="crud.cancelCU">취소</el-button>
        <el-button :loading="crud.status.cu === 2" type="primary" @click="crud.submitCU">확인</el-button>
      </div>
    </el-dialog>

    <!-- 테이블 -->
    <el-table
      ref="table"
      v-loading="crud.loading"
      :data="crud.data"
      style="width: 100%;"
      @selection-change="crud.selectionChangeHandler"
    >
      <el-table-column type="selection" width="55" />
      <el-table-column prop="name" label="이름" />
      <el-table-column prop="createTime" label="생성일" />
      <el-table-column
        v-if="checkPer(['admin','resource:edit','resource:del'])"
        label="작업"
        width="130px"
        align="center"
        fixed="right"
      >
        <template slot-scope="scope">
          <udOperation
            :data="scope.row"
            :permission="permission"
          />
        </template>
      </el-table-column>
    </el-table>

    <!-- 페이지네이션 -->
    <pagination />
  </div>
</template>

<script>
import crudResource from '@/api/domain/resource'
import CRUD, { presenter, header, form, crud } from '@crud/crud'
import rrOperation from '@crud/RR.operation'
import crudOperation from '@crud/CRUD.operation'
import udOperation from '@crud/UD.operation'
import pagination from '@crud/Pagination'
import DateRangePicker from '@/components/DateRangePicker'

const defaultForm = {
  id: null,
  name: null,
  enabled: 'true'
}

export default {
  name: 'Resource',
  components: {
    crudOperation,
    rrOperation,
    udOperation,
    pagination,
    DateRangePicker
  },
  cruds() {
    return CRUD({
      title: '리소스',
      url: 'api/resources',
      crudMethod: { ...crudResource }
    })
  },
  mixins: [presenter(), header(), form(defaultForm), crud()],
  dicts: ['status_dict'],
  data() {
    return {
      permission: {
        add: ['admin', 'resource:add'],
        edit: ['admin', 'resource:edit'],
        del: ['admin', 'resource:del']
      },
      enabledTypeOptions: [
        { key: 'true', display_name: '정상' },
        { key: 'false', display_name: '비활성' }
      ],
      rules: {
        name: [
          { required: true, message: '이름을 입력하세요', trigger: 'blur' }
        ]
      }
    }
  },
  methods: {
    // CRUD 훅 (필요시 추가)
    [CRUD.HOOK.afterToCU](crud, form) {
      // 다이얼로그 열기 전 초기화 로직
    },
    [CRUD.HOOK.beforeToAdd]() {
      // 추가 전 초기화
    },
    [CRUD.HOOK.beforeToEdit](crud, form) {
      // 수정 전 초기화
    },
    [CRUD.HOOK.afterValidateCU](crud) {
      // 제출 전 검증
      return true
    }
  }
}
</script>

<style rel="stylesheet/scss" lang="scss" scoped>
::v-deep .el-input-number .el-input__inner {
  text-align: left;
}
</style>
```

### 2.2 Script 구조 분석

#### 필수 Imports

```javascript
// 1. API 함수
import crudResource from '@/api/domain/resource'

// 2. CRUD 시스템 (신형)
import CRUD, { presenter, header, form, crud } from '@crud/crud'

// 3. CRUD UI 컴포넌트
import rrOperation from '@crud/RR.operation'      // 검색/리셋 버튼
import crudOperation from '@crud/CRUD.operation'  // 추가/수정/삭제/내보내기 버튼
import udOperation from '@crud/UD.operation'      // 행별 수정/삭제 버튼
import pagination from '@crud/Pagination'         // 페이지네이션

// 4. 공통 컴포넌트
import DateRangePicker from '@/components/DateRangePicker'
```

#### Mixins 순서 (중요!)

```javascript
mixins: [presenter(), header(), form(defaultForm), crud()]
```

- `presenter()`: CRUD 인스턴스 생성 및 관리 (필수, 첫 번째)
- `header()`: 검색 관련 기능 (query 객체 접근)
- `form(defaultForm)`: 폼 데이터 관리 (defaultForm으로 초기화)
- `crud()`: CRUD 데이터/상태 접근 (data, loading, status 등)

#### cruds() 함수

```javascript
cruds() {
  return CRUD({
    title: '리소스',              // 알림/다이얼로그 제목
    url: 'api/resources',         // API 엔드포인트
    sort: 'id,desc',              // 기본 정렬 (선택)
    crudMethod: { ...crudResource } // CRUD 메서드
  })
}
```

### 2.3 검색 조건 처리

#### query 객체 사용법

```javascript
// template에서
<el-input v-model="query.blurry" @keyup.enter.native="crud.toQuery" />
<el-select v-model="query.enabled" @change="crud.toQuery">

// query는 자동으로 API 파라미터로 전달됨
// GET /api/resources?blurry=xxx&enabled=xxx&page=0&size=10
```

#### 검색 조건 추가 방법

1. **단순 텍스트 검색**:
```vue
<el-input v-model="query.blurry" clearable size="small" 
  placeholder="검색어" class="filter-item"
  @keyup.enter.native="crud.toQuery" />
```

2. **날짜 범위 검색**:
```vue
<date-range-picker v-model="query.createTime" class="date-item" />
```

3. **드롭다운 선택**:
```vue
<el-select v-model="query.status" clearable size="small"
  placeholder="상태" class="filter-item" style="width: 90px"
  @change="crud.toQuery">
  <el-option v-for="item in statusOptions" 
    :key="item.key" :label="item.label" :value="item.key" />
</el-select>
```

4. **딕셔너리 사용**:
```vue
<el-select v-model="query.enabled" clearable size="small"
  placeholder="상태" class="filter-item" style="width: 90px"
  @change="crud.toQuery">
  <el-option v-for="item in dict.status_dict" 
    :key="item.value" :label="item.label" :value="item.value" />
</el-select>
```

### 2.4 다이얼로그 커스터마이징

#### 기본 구조

```vue
<el-dialog
  append-to-body
  :close-on-click-modal="false"
  :before-close="crud.cancelCU"
  :visible.sync="crud.status.cu > 0"
  :title="crud.status.title"
  width="500px"
>
```

- `crud.status.cu`: 0=닫힘, 1=추가, 2=수정
- `crud.status.title`: 자동으로 "추가" 또는 "수정" 설정
- `crud.cancelCU`: 다이얼로그 닫기
- `crud.submitCU`: 폼 제출

#### 폼 유효성 검사

```vue
<el-form ref="form" :model="form" :rules="rules" size="small" label-width="80px">
  <el-form-item label="이름" prop="name">
    <el-input v-model="form.name" />
  </el-form-item>
</el-form>
```

```javascript
data() {
  return {
    rules: {
      name: [
        { required: true, message: '필수 입력', trigger: 'blur' },
        { min: 2, max: 20, message: '2-20자', trigger: 'blur' }
      ],
      email: [
        { required: true, message: '이메일 입력', trigger: 'blur' },
        { type: 'email', message: '올바른 이메일 형식', trigger: 'blur' }
      ]
    }
  }
}
```

#### 커스텀 검증 함수

```javascript
data() {
  const validPhone = (rule, value, callback) => {
    if (!value) {
      callback(new Error('전화번호를 입력하세요'))
    } else if (!isvalidPhone(value)) {
      callback(new Error('올바른 11자리 전화번호'))
    } else {
      callback()
    }
  }
  return {
    rules: {
      phone: [{ required: true, trigger: 'blur', validator: validPhone }]
    }
  }
}
```

### 2.5 CRUD 훅 (Lifecycle)

```javascript
methods: {
  // 다이얼로그 열기 전 (추가/수정 공통)
  [CRUD.HOOK.afterToCU](crud, form) {
    // form 데이터로 추가 데이터 로드
  },
  
  // 추가 다이얼로그 열기 전
  [CRUD.HOOK.beforeToAdd]() {
    this.someData = []
  },
  
  // 수정 다이얼로그 열기 전
  [CRUD.HOOK.beforeToEdit](crud, form) {
    // 기존 데이터로 초기화
  },
  
  // 폼 검증 후 제출 전
  [CRUD.HOOK.afterValidateCU](crud) {
    // 추가 검증 또는 데이터 변환
    crud.form.extraData = this.transformData()
    return true // false면 제출 취소
  },
  
  // 데이터 새로고침 후
  [CRUD.HOOK.afterRefresh]() {
    // 테이블 새로고침 후 작업
  },
  
  // 삭제 후
  [CRUD.HOOK.afterDelete](crud, data) {
    // 삭제 후 정리 작업
  }
}
```

---

## 3. 비표준 페이지 패턴

### 3.1 모니터링 페이지 (CRUD 없음)

**예시**: `views/monitor/server/index.vue`

```vue
<template>
  <div class="app-container">
    <el-card class="box-card">
      <!-- 대시보드 UI -->
    </el-card>
    <v-chart :options="chartData" />
  </div>
</template>

<script>
import ECharts from 'vue-echarts'
import { initData } from '@/api/data'

export default {
  name: 'ServerMonitor',
  components: { 'v-chart': ECharts },
  data() {
    return {
      url: 'api/monitor',
      data: {},
      monitor: null
    }
  },
  created() {
    this.init()
    this.monitor = setInterval(() => this.init(), 3500)
  },
  destroyed() {
    clearInterval(this.monitor)
  },
  methods: {
    init() {
      initData(this.url, {}).then(data => {
        this.data = data
      })
    }
  }
}
</script>
```

**특징**:
- CRUD mixin 사용하지 않음
- `initData()` 직접 호출
- 주기적 데이터 새로고침 (setInterval)
- ECharts로 시각화

### 3.2 탭 기반 페이지

**예시**: `views/tools/email/index.vue`

```vue
<template>
  <el-tabs v-model="activeName" style="padding-left: 8px;">
    <el-tab-pane label="설정" name="first">
      <Config />
    </el-tab-pane>
    <el-tab-pane label="전송" name="second">
      <Send />
    </el-tab-pane>
    <el-tab-pane label="도움말" name="third">
      <!-- 정적 콘텐츠 -->
    </el-tab-pane>
  </el-tabs>
</template>

<script>
import Config from './config'
import Send from './send'

export default {
  name: 'Email',
  components: { Config, Send },
  data() {
    return {
      activeName: 'second'
    }
  }
}
</script>
```

**특징**:
- 여러 하위 컴포넌트를 탭으로 구성
- 각 탭이 독립적인 기능

### 3.3 서브 컴포넌트 분리 패턴

**예시**: `views/system/job/`

```
job/
├── index.vue          # 메인 (테이블 + CRUD)
└── module/
    ├── header.vue     # 검색 영역
    └── form.vue       # 폼 다이얼로그
```

**index.vue**:
```vue
<template>
  <div class="app-container">
    <eHeader :dict="dict" :permission="permission" />
    <crudOperation :permission="permission" />
    <el-table>...</el-table>
    <pagination />
    <eForm :job-status="dict.job_status" />
  </div>
</template>

<script>
import eHeader from './module/header'
import eForm from './module/form'
import CRUD, { presenter } from '@crud/crud'

export default {
  components: { eHeader, eForm, crudOperation, pagination, udOperation },
  cruds() {
    return CRUD({ title: '岗位', url: 'api/job', crudMethod: { ...crudJob }})
  },
  mixins: [presenter()] // header(), form()은 서브 컴포넌트에 위임
}
</script>
```

**module/header.vue**:
```vue
<template>
  <div v-if="crud.props.searchToggle">
    <el-input v-model="query.name" ... />
    <rrOperation />
  </div>
</template>

<script>
import { header } from '@crud/crud'
import rrOperation from '@crud/RR.operation'

export default {
  mixins: [header()],
  props: { dict: Object, permission: Object }
}
</script>
```

**module/form.vue**:
```vue
<template>
  <el-dialog :visible="crud.status.cu > 0" ...>
    <el-form ref="form" :model="form" :rules="rules">
      ...
    </el-form>
    <div slot="footer">
      <el-button @click="crud.cancelCU">취소</el-button>
      <el-button @click="crud.submitCU">확인</el-button>
    </div>
  </el-dialog>
</template>

<script>
import { form } from '@crud/crud'

const defaultForm = { id: null, name: '', enabled: true }

export default {
  mixins: [form(defaultForm)],
  props: { jobStatus: Array },
  data() {
    return { rules: { ... } }
  }
}
</script>
```

---

## 4. 공통 컴포넌트 사용 패턴

### 4.1 CRUD 컴포넌트 (@crud/)

#### CRUD.operation.vue

**용도**: 추가/수정/삭제/내보내기 버튼 + 검색 토글 + 컬럼 표시 제어

```vue
<crudOperation :permission="permission" />

<!-- 커스텀 버튼 추가 -->
<crudOperation :permission="permission">
  <el-button slot="right" v-permission="['admin','user:add']"
    :disabled="crud.selections.length === 0"
    size="mini" type="primary" icon="el-icon-refresh-left"
    @click="resetPwd(crud.selections)">
    비밀번호 초기화
  </el-button>
</crudOperation>
```

**Props**:
- `permission`: `{ add: [], edit: [], del: [] }` - 권한 배열
- `hiddenColumns`: 숨길 컬럼 property 배열
- `ignoreColumns`: 컬럼 표시 제어에서 제외할 property 배열

**자동 생성 버튼**:
- 추가 (v-permission="permission.add")
- 수정 (v-permission="permission.edit", selections.length === 1)
- 삭제 (v-permission="permission.del", selections.length > 0)
- 내보내기 (crud.optShow.download)

#### RR.operation.vue

**용도**: 검색/리셋 버튼

```vue
<rrOperation />
```

자동으로 `crud.toQuery()`와 `crud.resetQuery()` 연결

#### UD.operation.vue

**용도**: 테이블 행별 수정/삭제 버튼

```vue
<udOperation
  :data="scope.row"
  :permission="permission"
  :disabled-edit="scope.row.id === 1"
  :disabled-dle="scope.row.id === 1"
  msg="정말 삭제하시겠습니까?"
/>
```

**Props**:
- `data`: 행 데이터 (필수)
- `permission`: `{ edit: [], del: [] }` (필수)
- `disabledEdit`: 수정 버튼 비활성화
- `disabledDle`: 삭제 버튼 비활성화
- `msg`: 삭제 확인 메시지 (기본: "确定删除本条数据吗？")

#### Pagination.vue

**용도**: CRUD와 자동 연동되는 페이지네이션

```vue
<pagination />
```

자동으로 `crud.page`, `crud.size`, `crud.total`과 연동

### 4.2 DateRangePicker

**용도**: 날짜 범위 검색

```vue
<date-range-picker v-model="query.createTime" class="date-item" />
```

**기본 설정**:
- type: 'daterange'
- valueFormat: 'yyyy-MM-dd HH:mm:ss'
- defaultTime: ['00:00:00', '23:59:59']
- shortcuts: calendarShortcuts (오늘, 어제, 최근 7일, 최근 30일 등)

### 4.3 IconSelect

**용도**: SVG 아이콘 선택기 (메뉴 관리에서 사용)

```vue
<el-popover placement="bottom-start" width="450" trigger="click"
  @show="$refs['iconSelect'].reset()">
  <IconSelect ref="iconSelect" @selected="selected" />
  <el-input slot="reference" v-model="form.icon" readonly>
    <svg-icon v-if="form.icon" slot="prefix" :icon-class="form.icon" />
  </el-input>
</el-popover>
```

```javascript
methods: {
  selected(name) {
    this.form.icon = name
  }
}
```

### 4.4 WangEditor

**용도**: 리치 텍스트 에디터

```vue
<WangEditor v-model="form.content" :editor-height="500" />
```

**Props**:
- `value`: HTML 문자열 (v-model)
- `editorHeight`: 에디터 높이 (px, 기본 420)

**특징**:
- 이미지 업로드 자동 처리 (imagesUploadApi 사용)
- simple 모드 툴바

### 4.5 Treeselect (@riophae/vue-treeselect)

**용도**: 계층형 데이터 선택 (부서, 메뉴 등)

```vue
<treeselect
  v-model="form.dept.id"
  :options="depts"
  :load-options="loadDepts"
  style="width: 370px"
  placeholder="부서 선택"
/>
```

```javascript
import Treeselect from '@riophae/vue-treeselect'
import '@riophae/vue-treeselect/dist/vue-treeselect.css'
import { LOAD_CHILDREN_OPTIONS } from '@riophae/vue-treeselect'

data() {
  return {
    depts: []
  }
},
methods: {
  loadDepts({ action, parentNode, callback }) {
    if (action === LOAD_CHILDREN_OPTIONS) {
      getDepts({ enabled: true, pid: parentNode.id }).then(res => {
        parentNode.children = res.content.map(obj => {
          if (obj.hasChildren) obj.children = null
          return obj
        })
        setTimeout(() => callback(), 200)
      })
    }
  }
}
```

**주의**: CSS 반드시 import 필요

---

## 5. 스타일 규칙

### 5.1 SCSS 변수 (element-variables.scss)

```scss
$--color-primary: #1890ff;      // 기본 색상
$--color-success: #13ce66;      // 성공 색상
$--color-warning: #FFBA00;      // 경고 색상
$--color-danger: #ff4949;       // 위험 색상

$--button-font-weight: 400;
$--border-color-light: #dfe4ed;
$--border-color-lighter: #e6ebf5;
$--table-border: 1px solid #dfe6ec;
```

### 5.2 전역 스타일 (index.scss)

```scss
.app-container {
  padding: 20px 20px 45px 20px;
}

.head-container {
  // 검색/버튼 영역 컨테이너
}

.filter-item {
  // 검색 필터 아이템 (margin-right 자동 적용)
}

.pagination-container {
  margin-top: 30px;
}
```

### 5.3 Scoped 스타일 사용

```vue
<style rel="stylesheet/scss" lang="scss" scoped>
::v-deep .el-input-number .el-input__inner {
  text-align: left;
}

::v-deep .vue-treeselect__control,
::v-deep .vue-treeselect__placeholder,
::v-deep .vue-treeselect__single-value {
  height: 30px;
  line-height: 30px;
}
</style>
```

**규칙**:
- 모든 페이지는 `scoped` 사용
- Element UI 내부 요소 스타일링: `::v-deep` 사용
- 전역 스타일 변경 금지 (index.scss 수정)

---

## 6. 새 페이지 추가 체크리스트

### 6.1 단계별 체크리스트

#### 1단계: API 파일 생성

```javascript
// src/api/domain/resource.js
import request from '@/utils/request'

export function add(data) {
  return request({
    url: 'api/resources',
    method: 'post',
    data
  })
}

export function del(ids) {
  return request({
    url: 'api/resources',
    method: 'delete',
    data: ids
  })
}

export function edit(data) {
  return request({
    url: 'api/resources',
    method: 'put',
    data
  })
}

export default { add, edit, del }
```

#### 2단계: Views 폴더 생성

```
src/views/domain/
└── resource/
    └── index.vue
```

#### 3단계: index.vue 작성

표준 CRUD 템플릿 복사 후 수정:
- `name`: 컴포넌트 이름 (PascalCase)
- `url`: API 엔드포인트
- `title`: 표시 제목
- `defaultForm`: 폼 초기값
- `permission`: 권한 배열
- `rules`: 폼 유효성 검사

#### 4단계: 라우터 등록 (백엔드 메뉴)

백엔드에서 메뉴 관리로 동적 라우트 추가:
- 메뉴 제목
- 라우트 경로
- 컴포넌트 경로: `domain/resource/index`
- 권한标识: `resource:list`, `resource:add`, `resource:edit`, `resource:del`

### 6.2 검증 항목

- [ ] API 파일이 `src/api/{도메인}/`에 생성되었는가?
- [ ] API 함수가 `add`, `edit`, `del`을 포함하는가?
- [ ] Views 폴더가 `src/views/{도메인}/`에 생성되었는가?
- [ ] 컴포넌트 `name`이 PascalCase인가?
- [ ] `cruds()` 함수의 `url`이 올바른가?
- [ ] `defaultForm`이 모든 필드를 포함하는가?
- [ ] `permission` 객체가 `add`, `edit`, `del`을 포함하는가?
- [ ] `rules`에 필수 필드 검증이 있는가?
- [ ] 백엔드 메뉴에 등록되었는가?

---

## 7. 하지 말아야 할 것

### 7.1 절대 금지

1. **전역 스타일 직접 수정**
   ```scss
   // ❌ 금지
   <style>
   .el-button { ... }
   </style>
   
   // ✅ 올바른 방법
   <style scoped>
   ::v-deep .el-button { ... }
   </style>
   ```

2. **CRUD mixin 순서 변경**
   ```javascript
   // ❌ 금지
   mixins: [crud(), form(), header(), presenter()]
   
   // ✅ 올바른 방법
   mixins: [presenter(), header(), form(defaultForm), crud()]
   ```

3. **defaultForm 생략**
   ```javascript
   // ❌ 금지
   mixins: [presenter(), header(), form(), crud()]
   
   // ✅ 올바른 방법
   const defaultForm = { id: null, name: null }
   mixins: [presenter(), header(), form(defaultForm), crud()]
   ```

4. **query 직접 조작**
   ```javascript
   // ❌ 금지
   this.query.page = 0
   this.loadData()
   
   // ✅ 올바른 방법
   this.crud.toQuery()
   ```

5. **테이블 데이터 직접 수정**
   ```javascript
   // ❌ 금지
   this.crud.data = newData
   
   // ✅ 올바른 방법
   this.crud.refresh()
   // 또는
   this.crud.toQuery()
   ```

6. **다이얼로그 상태 직접 조작**
   ```javascript
   // ❌ 금지
   this.dialogVisible = true
   
   // ✅ 올바른 방법
   this.crud.toAdd()  // 추가
   this.crud.toEdit(row)  // 수정
   this.crud.cancelCU()  // 닫기
   ```

7. **API 직접 호출 (CRUD 작업)**
   ```javascript
   // ❌ 금지
   crudResource.add(this.form).then(...)
   
   // ✅ 올바른 방법
   this.crud.submitCU()
   ```

8. **permission 객체 생략**
   ```javascript
   // ❌ 금지
   <crudOperation />
   
   // ✅ 올바른 방법
   <crudOperation :permission="permission" />
   
   data() {
     return {
       permission: {
         add: ['admin', 'resource:add'],
         edit: ['admin', 'resource:edit'],
         del: ['admin', 'resource:del']
       }
     }
   }
   ```

9. **v-permission 없이 버튼 추가**
   ```javascript
   // ❌ 금지
   <el-button @click="doSomething">작업</el-button>
   
   // ✅ 올바른 방법
   <el-button v-permission="['admin','resource:edit']" @click="doSomething">작업</el-button>
   ```

10. **checkPer 없이 작업 컬럼 표시**
    ```javascript
    // ❌ 금지
    <el-table-column label="작업">
    
    // ✅ 올바른 방법
    <el-table-column v-if="checkPer(['admin','resource:edit','resource:del'])" label="작업">
    ```

### 7.2 주의사항

1. **el-input-number 스타일**
   ```scss
   // 항상 추가 필요
   <style scoped>
   ::v-deep .el-input-number .el-input__inner {
     text-align: left;
   }
   </style>
   ```

2. **Treeselect CSS**
   ```javascript
   // 반드시 import
   import '@riophae/vue-treeselect/dist/vue-treeselect.css'
   ```

3. **DateRangePicker 클래스**
   ```vue
   <!-- 항상 class="date-item" 추가 -->
   <date-range-picker v-model="query.createTime" class="date-item" />
   ```

4. **검색 입력 필드 스타일**
   ```vue
   <!-- 항상 class="filter-item" 추가 -->
   <el-input v-model="query.blurry" class="filter-item" />
   <el-select v-model="query.status" class="filter-item" />
   ```

5. **폼 크기**
   ```vue
   <!-- 항상 size="small" 사용 -->
   <el-form size="small">
   <el-input size="small">
   ```

6. **다이얼로그 속성**
   ```vue
   <!-- 항상 추가 -->
   <el-dialog
     append-to-body
     :close-on-click-modal="false"
     :before-close="crud.cancelCU"
     :visible.sync="crud.status.cu > 0"
   >
   ```

7. **el-table 속성**
   ```vue
   <!-- 항상 추가 -->
   <el-table
     ref="table"
     v-loading="crud.loading"
     :data="crud.data"
     @selection-change="crud.selectionChangeHandler"
   >
   ```

8. **show-overflow-tooltip**
   ```vue
   <!-- 긴 텍스트가 있을 수 있는 컬럼 -->
   <el-table-column :show-overflow-tooltip="true" prop="description" />
   ```

---

## 8. 디버깅 팁

### 8.1 CRUD 상태 확인

```javascript
// 브라우저 콘솔에서
this.crud.status  // { cu: 0|1|2, title: '추가'|'수정' }
this.crud.data    // 현재 테이블 데이터
this.crud.selections  // 선택된 행들
this.query        // 현재 검색 조건
this.form         // 현재 폼 데이터
```

### 8.2 일반적인 문제

**문제**: 다이얼로그가 열리지 않음
```javascript
// 확인: cruds() 함수가 있는가?
cruds() {
  return CRUD({ title: '...', url: '...', crudMethod: { ...crudApi }})
}
```

**문제**: 검색이 작동하지 않음
```javascript
// 확인: header() mixin이 있는가?
mixins: [presenter(), header(), form(defaultForm), crud()]
```

**문제**: 폼 데이터가 초기화되지 않음
```javascript
// 확인: form(defaultForm)에 defaultForm을 전달했는가?
const defaultForm = { id: null, name: null }
mixins: [presenter(), header(), form(defaultForm), crud()]
```

**문제**: 권한 버튼이 표시되지 않음
```javascript
// 확인: permission 객체를 전달했는가?
<crudOperation :permission="permission" />
```

---

## 9. 고급 패턴

### 9.1 트리 테이블 (계층형 데이터)

```vue
<el-table
  ref="table"
  v-loading="crud.loading"
  lazy
  :load="getDeptDatas"
  :tree-props="{children: 'children', hasChildren: 'hasChildren'}"
  :data="crud.data"
  row-key="id"
  @select="crud.selectChange"
  @select-all="crud.selectAllChange"
  @selection-change="crud.selectionChangeHandler"
>
```

```javascript
methods: {
  getDeptDatas(tree, treeNode, resolve) {
    const params = { pid: tree.id }
    setTimeout(() => {
      crudDept.getDepts(params).then(res => {
        resolve(res.content)
      })
    }, 100)
  }
}
```

### 9.2 상태 변경 (el-switch)

```vue
<el-table-column label="상태" align="center" prop="enabled">
  <template slot-scope="scope">
    <el-switch
      v-model="scope.row.enabled"
      active-color="#409EFF"
      inactive-color="#F56C6C"
      @change="changeEnabled(scope.row, scope.row.enabled)"
    />
  </template>
</el-table-column>
```

```javascript
methods: {
  changeEnabled(data, val) {
    this.$confirm(`"${this.dict.label.status_dict[val]}" ${data.name}, 계속하시겠습니까?`, '확인', {
      confirmButtonText: '확인',
      cancelButtonText: '취소',
      type: 'warning'
    }).then(() => {
      crudResource.edit(data).then(() => {
        this.crud.notify(this.dict.label.status_dict[val] + ' 성공', CRUD.NOTIFICATION_TYPE.SUCCESS)
      }).catch(() => {
        data.enabled = !data.enabled
      })
    }).catch(() => {
      data.enabled = !data.enabled
    })
  }
}
```

### 9.3 일괄 작업

```vue
<crudOperation :permission="permission">
  <el-button
    slot="right"
    v-permission="['admin','user:edit']"
    :disabled="crud.selections.length === 0"
    size="mini"
    type="primary"
    icon="el-icon-refresh-left"
    @click="batchAction(crud.selections)"
  >
    일괄 작업
  </el-button>
</crudOperation>
```

```javascript
methods: {
  batchAction(datas) {
    this.$confirm(`${datas.length}개 항목을 처리하시겠습니까?`, '확인', {
      confirmButtonText: '확인',
      cancelButtonText: '취소',
      type: 'warning'
    }).then(() => {
      const ids = datas.map(val => val.id)
      crudResource.batchAction(ids).then(() => {
        this.crud.notify('처리 성공', CRUD.NOTIFICATION_TYPE.SUCCESS)
      })
    })
  }
}
```

---

## 10. 참고 자료

### 10.1 관련 파일

- CRUD 시스템: `src/components/Crud/crud.js`
- CRUD mixin: `src/mixins/crud.js` (구형, 사용하지 않음)
- API 유틸: `src/api/data.js`
- 권한 유틸: `src/utils/permission.js`
- 스타일: `src/assets/styles/`

### 10.2 예제 페이지

- 표준 CRUD: `src/views/system/dept/index.vue`
- 복잡한 CRUD: `src/views/system/user/index.vue`
- 서브 컴포넌트 분리: `src/views/system/job/`
- 비표준 (모니터링): `src/views/monitor/server/index.vue`
- 비표준 (탭): `src/views/tools/email/index.vue`

---

**마지막 업데이트**: 2026-05-31
**버전**: 1.0
