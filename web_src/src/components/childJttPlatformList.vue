<template>
  <div id="app" style="width: 100%">
    <div class="page-header">
      <div class="page-title">下级平台列表</div>
      <div class="page-header-btn">
        <el-button icon="el-icon-plus" size="mini" style="margin-right: 1rem;" type="primary" @click="openAddDialog">添加</el-button>
        <el-button icon="el-icon-refresh-right" circle size="mini" @click="refresh()"></el-button>
      </div>
    </div>

    <el-table :data="platformList" style="width: 100%" :height="winHeight" v-loading="loading">
      <el-table-column prop="name" label="名称"></el-table-column>
      <el-table-column prop="serverGBId" label="平台编号" min-width="200"></el-table-column>
      <el-table-column label="是否启用" min-width="80">
        <template slot-scope="scope">
          <div slot="reference" class="name-wrapper">
            <el-tag size="medium" v-if="scope.row.enable">已启用</el-tag>
            <el-tag size="medium" type="info" v-if="!scope.row.enable">未启用</el-tag>
          </div>
        </template>
      </el-table-column>
      <el-table-column label="状态" min-width="80">
        <template slot-scope="scope">
          <div slot="reference" class="name-wrapper">
            <el-tag size="medium" v-if="scope.row.status">在线</el-tag>
            <el-tag size="medium" type="info" v-if="!scope.row.status">离线</el-tag>
          </div>
        </template>
      </el-table-column>
      <el-table-column label="地址" min-width="160">
        <template slot-scope="scope">
          <div slot="reference" class="name-wrapper">
            <el-tag size="medium">{{ scope.row.serverIP }}:{{ scope.row.serverPort }}</el-tag>
          </div>
        </template>
      </el-table-column>
      <el-table-column prop="commandPort" label="信令端口" min-width="120"></el-table-column>
      <el-table-column prop="protocolVersion" label="协议版本" min-width="120"></el-table-column>
      <el-table-column label="操作" min-width="240" fixed="right">
        <template slot-scope="scope">
          <el-button size="medium" icon="el-icon-edit" type="text" @click="handleEdit(scope.row)">编辑</el-button>
          <el-button size="medium" icon="el-icon-delete" type="text" style="color: #f56c6c" @click="handleDelete(scope.row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <el-pagination
      style="float: right"
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
      :current-page="currentPage"
      :page-sizes="[15, 25, 35, 50]"
      :page-size="pageSize"
      layout="total, sizes, prev, pager, next"
      :total="total">
    </el-pagination>

    <!-- 添加/编辑对话框 -->
    <el-dialog :title="dialogTitle" :visible.sync="dialogVisible" width="40%">
      <el-form :model="platformForm" :rules="rules" ref="platformForm" label-width="100px">
        <el-form-item label="平台名称" prop="name">
          <el-input v-model="platformForm.name" placeholder="请输入平台名称"></el-input>
        </el-form-item>
        <el-form-item label="平台编号" prop="code">
          <el-input v-model="platformForm.code" placeholder="请输入平台编号"></el-input>
        </el-form-item>
        <el-form-item label="IP地址" prop="ip">
          <el-input v-model="platformForm.ip" placeholder="请输入IP地址"></el-input>
        </el-form-item>
        <el-row>
          <el-col :span="12">
            <el-form-item label="信令端口" prop="serverPort">
              <el-input v-model.number="platformForm.serverPort" placeholder="请输入信令端口"></el-input>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="协议版本" prop="protocolVersion">
              <el-select v-model="platformForm.protocolVersion" placeholder="请选择协议版本" default-first-option>
                <el-option label="2011" value="2011" selected></el-option>
                <el-option label="2019" value="2019"></el-option>
              </el-select>
            </el-form-item>
          </el-col>
        </el-row>
        <el-row>
          <el-col :span="12">
            <el-form-item label="是否加密" prop="enableEncrypt">
              <el-switch v-model="platformForm.enableEncrypt"></el-switch>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="是否校验" prop="enableValidate">
              <el-switch v-model="platformForm.enableValidate"></el-switch>
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitForm">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
export default {
  name: 'childJttPlatformList',
  data() {
    // IP地址验证规则
    const validateIP = (rule, value, callback) => {
      const ipRegex = /^(\d{1,3}\.){3}\d{1,3}$/
      if (!value) {
        callback(new Error('请输入IP地址'))
      } else if (!ipRegex.test(value)) {
        callback(new Error('请输入正确的IP地址格式'))
      } else {
        // 验证每个数字是否在0-255范围内
        const parts = value.split('.')
        const valid = parts.every(part => {
          const num = parseInt(part, 10)
          return num >= 0 && num <= 255
        })
        if (!valid) {
          callback(new Error('IP地址的每个段必须在0-255之间'))
        } else {
          callback()
        }
      }
    }

    return {
      loading: false,
      platformList: [],
      dialogVisible: false,
      dialogTitle: '添加平台',
      winHeight: window.innerHeight - 260,
      platformForm: {
        name: '',
        code: '',
        ip: '',
        serverPort: null,
        protocolVersion: '2011',
        enableEncrypt: false,
        enableValidate: false
      },
      rules: {
        name: [
          { required: true, message: '请输入平台名称', trigger: 'blur' }
        ],
        code: [
          { required: true, message: '请输入平台编号', trigger: 'blur' }
        ],
        ip: [
          { required: true, message: '请输入IP地址', trigger: 'blur' },
          { validator: validateIP, trigger: 'blur' }
        ],
        serverPort: [
          { required: true, message: '请输入信令端口', trigger: 'blur' },
          { type: 'number', message: '端口必须为数字', trigger: 'blur' },
          { type: 'number', min: 1, max: 65535, message: '端口必须在1-65535之间', trigger: 'blur' }
        ],
        protocolVersion: [
          { required: true, message: '请选择协议版本', trigger: 'change' }
        ]
      },
      currentPage: parseInt(this.$route.params.page) || 1,
      pageSize: parseInt(this.$route.params.count) || 15,
      total: 0
    }
  },
  watch: {
    '$route.params': {
      handler(newVal) {
        this.currentPage = parseInt(newVal.page) || 1
        this.pageSize = parseInt(newVal.count) || 15
        this.getPlatformList()
      },
      immediate: true
    }
  },
  mounted() {
    this.initData();
    this.updateLooper = setInterval(this.initData, 10000);
  },
  destroyed() {
    clearTimeout(this.updateLooper);
  },
  methods: {
    initData() {
      this.getPlatformList();
    },
    refresh() {
      this.initData();
    },
    getPlatformList() {
      this.loading = true
      this.$axios.get(`/api/platform/child/query/${this.pageSize}/${this.currentPage}`).then(res => {
        if (res.data.code === 0) {
          this.platformList = res.data.data.list
          this.total = res.data.data.total
        }
      }).finally(() => {
        this.loading = false
      })
    },
    openAddDialog() {
      this.dialogTitle = '添加平台'
      this.platformForm = {
        name: '',
        code: '',
        ip: '',
        serverPort: null,
        protocolVersion: '2011',
        enableEncrypt: false,
        enableValidate: false
      }
      this.dialogVisible = true
    },
    handleEdit(row) {
      this.dialogTitle = '编辑平台'
      this.platformForm = {...row}
      this.dialogVisible = true
    },
    handleDelete(row) {
      this.$confirm('确认删除该平台?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        // 这里添加删除平台的API调用
        this.$axios.delete(`/api/platform/child/${row.serverGBId}`).then(res => {
          if (res.data.code === 0) {
            this.$message.success('删除成功')
            this.getPlatformList()
          }
        })
      })
    },
    submitForm() {
      this.$refs.platformForm.validate((valid) => {
        if (valid) {
          const api = this.dialogTitle === '添加平台' ? 
            this.$axios.post('/api/platform/child/add', this.platformForm) :
            this.$axios.put('/api/platform/child/update', this.platformForm)
          
          api.then(res => {
            if (res.data.code === 0) {
              this.$message.success(`${this.dialogTitle}成功`)
              this.dialogVisible = false
              this.getPlatformList()
            }
          })
        }
      })
    },
    handleSizeChange(val) {
      this.pageSize = val
      this.$router.push(`/childJttPlatformList/${val}/${this.currentPage}`)
    },
    handleCurrentChange(val) {
      this.currentPage = val
      this.$router.push(`/childJttPlatformList/${this.pageSize}/${val}`)
    },
    chooseChannel(row) {
      // TODO: 实现选择通道功能
      console.log('选择通道:', row)
    }
  }
}
</script>

<style>
.subscribe-on {
  color: #409EFF;
  font-size: 18px;
}
.subscribe-off {
  color: #afafb3;
  font-size: 18px;
}
</style>
