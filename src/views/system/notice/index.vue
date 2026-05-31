<template>
  <div class="app-container">
    <!--工具栏-->
    <div class="head-container">
      <div v-if="crud.props.searchToggle">
        <!-- 搜索 -->
        <el-input v-model="query.title" clearable size="small" placeholder="输入标题搜索" style="width: 200px;" class="filter-item" @keyup.enter.native="crud.toQuery" />
        <el-select v-model="query.enabled" clearable size="small" placeholder="状态" class="filter-item" style="width: 90px" @change="crud.toQuery">
          <el-option v-for="item in enabledTypeOptions" :key="item.key" :label="item.display_name" :value="item.key" />
        </el-select>
        <rrOperation />
      </div>
      <crudOperation :permission="permission" />
    </div>
    <!--表单组件-->
    <el-dialog append-to-body :close-on-click-modal="false" :before-close="crud.cancelCU" :visible.sync="crud.status.cu > 0" :title="crud.status.title" width="700px">
      <el-form ref="form" :model="form" :rules="rules" size="small" label-width="80px">
        <el-form-item label="标题" prop="title">
          <el-input v-model="form.title" style="width: 570px;" />
        </el-form-item>
        <el-form-item label="内容" prop="content">
          <WangEditor v-model="form.content" :editor-height="300" />
        </el-form-item>
        <el-form-item label="状态" prop="enabled">
          <el-radio v-model="form.enabled" label="true">激活</el-radio>
          <el-radio v-model="form.enabled" label="false">禁用</el-radio>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="text" @click="crud.cancelCU">取消</el-button>
        <el-button :loading="crud.status.cu === 2" type="primary" @click="crud.submitCU">确认</el-button>
      </div>
    </el-dialog>
    <!--表格渲染-->
    <el-table ref="table" v-loading="crud.loading" :data="crud.data" style="width: 100%;" @selection-change="crud.selectionChangeHandler">
      <el-table-column type="selection" width="55" />
      <el-table-column prop="title" label="标题" show-overflow-tooltip />
      <el-table-column prop="enabled" label="状态" align="center" width="100">
        <template slot-scope="scope">
          <el-switch
            v-model="scope.row.enabled"
            active-color="#409EFF"
            inactive-color="#F56C6C"
            @change="changeEnabled(scope.row, scope.row.enabled)"
          />
        </template>
      </el-table-column>
      <el-table-column prop="createBy" label="创建人" width="120" />
      <el-table-column prop="createTime" label="创建日期" width="180" />
      <el-table-column v-if="checkPer(['admin','notice:edit','notice:del'])" label="操作" width="130px" align="center" fixed="right">
        <template slot-scope="scope">
          <udOperation
            :data="scope.row"
            :permission="permission"
          />
        </template>
      </el-table-column>
    </el-table>
    <!--分页组件-->
    <pagination />
  </div>
</template>

<script>
import crudNotice from '@/api/system/notice'
import CRUD, { presenter, header, form, crud } from '@crud/crud'
import rrOperation from '@crud/RR.operation'
import crudOperation from '@crud/CRUD.operation'
import udOperation from '@crud/UD.operation'
import pagination from '@crud/Pagination'
import WangEditor from '@/components/WangEditor'

const defaultForm = { id: null, title: null, content: '', enabled: 'true' }
export default {
  name: 'Notice',
  components: { crudOperation, rrOperation, udOperation, pagination, WangEditor },
  cruds() {
    return CRUD({ title: '公告', url: 'api/notice', crudMethod: { ...crudNotice }})
  },
  mixins: [presenter(), header(), form(defaultForm), crud()],
  data() {
    return {
      rules: {
        title: [
          { required: true, message: '请输入标题', trigger: 'blur' }
        ],
        content: [
          { required: true, message: '请输入内容', trigger: 'blur' }
        ]
      },
      permission: {
        add: ['admin', 'notice:add'],
        edit: ['admin', 'notice:edit'],
        del: ['admin', 'notice:del']
      },
      enabledTypeOptions: [
        { key: 'true', display_name: '激活' },
        { key: 'false', display_name: '禁用' }
      ]
    }
  },
  methods: {
    changeEnabled(data, val) {
      this.$confirm('此操作将 "' + (val === 'true' || val === true ? '激活' : '禁用') + '" ' + data.title + '公告, 是否继续？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        crudNotice.edit(data).then(() => {
          this.crud.notify((val === 'true' || val === true ? '激活' : '禁用') + '成功', CRUD.NOTIFICATION_TYPE.SUCCESS)
        }).catch(err => {
          data.enabled = !data.enabled
          console.log(err.response.data.message)
        })
      }).catch(() => {
        data.enabled = !data.enabled
      })
    }
  }
}
</script>
