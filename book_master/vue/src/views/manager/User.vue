<template>
  <div>
    <div style="margin-bottom: 10px">
      <el-input style="width: 260px" placeholder="请输入..." v-model="name"></el-input>
      <el-button style="margin-left: 10px" type="primary" @click="load">搜 索</el-button>
      <el-button type="info" @click="reset">重 置</el-button>
    </div>

    <div style="margin: 10px 0">
      <el-button type="primary" @click="handleAdd" v-if="user.role === 'ADMIN'">新 增</el-button>
      <el-popconfirm
          v-if="user.role === 'ADMIN'"
          style="margin-left: 10px"
          confirm-button-text='确 定'
          cancel-button-text='我再想想'
          icon="el-icon-info"
          icon-color="red"
          title="您确定批量删除这些数据吗？"
          @confirm="delBatch"
      >
        <el-button type="danger" slot="reference">批量删除</el-button>
      </el-popconfirm>
    </div>

    <el-table :data="tableData" stripe :header-cell-style="{background:'#f4f3f9',color:'#606266'}"
              @selection-change="handleSelectionChange">
      <el-table-column type="selection" width="55" align="center" v-if="user.role === 'ADMIN'"></el-table-column>
<!--      <el-table-column prop="id" label="序号" width="80" align="center" sortable></el-table-column>-->
      <el-table-column align="center" prop="username" label="用户名"></el-table-column>
      <!--      <el-table-column align="center" prop="password" label="密码"></el-table-column>-->
      <el-table-column align="center" prop="name" label="名称"></el-table-column>
      <el-table-column align="center" prop="sex" label="性别"></el-table-column>
      <el-table-column align="center" prop="birth" label="生日"></el-table-column>
      <el-table-column align="center" prop="phone" label="手机"></el-table-column>
      <el-table-column align="center" prop="address" label="地址"></el-table-column>
      <el-table-column align="center" prop="account" label="积分"></el-table-column>
      <el-table-column align="center" prop="role" label="身份">
        <template slot-scope="scope">
                  <span v-if="scope.row.role == 'ADMIN'">管理员</span>
                  <span v-if="scope.row.role == 'USER'">用户</span>
        </template>
      </el-table-column>
<!--      <el-table-column label="禁用" v-if="user.role === 'ADMIN'">-->
<!--        <template v-slot="scope">-->
<!--          <el-switch v-model="scope.row.disable" active-color="red" @change="disableUser(scope.row)"></el-switch>-->
<!--        </template>-->
<!--      </el-table-column>-->
      <el-table-column label="操作" width="240" align="center">
        <template v-slot="scope">
          <el-button type="primary" @click="handleEdit(scope.row)" size="mini" v-if="user.role === 'ADMIN'">编辑
          </el-button>
          <el-button type="warning" @click="resetPass(scope.row)" size="mini" v-if="user.role === 'ADMIN'">重置密码
          </el-button>
          <el-popconfirm
              v-if="user.role === 'ADMIN'"
              style="margin-left: 10px"
              confirm-button-text='确定'
              cancel-button-text='我再想想'
              icon="el-icon-info"
              icon-color="red"
              title="您确定删除吗？"
              @confirm="del(scope.row.id)"
          >
            <el-button type="danger" slot="reference" size="mini">删除</el-button>
          </el-popconfirm>
        </template>
      </el-table-column>
    </el-table>

    <div style="margin: 10px 0">
      <el-pagination
          style="padding: 0"
          background
          layout="total, prev, pager, next"
          @current-change="handleCurrentChange"
          :current-page="pageNum"
          :total="total">
      </el-pagination>
    </div>

    <el-dialog title="信息" :visible.sync="dialogFormVisible" width="30%" :close-on-click-modal="false">
      <el-form label-width="100px" style="padding-right: 40px" :model="form" :rules="rules" ref="ruleForm">
        <el-form-item prop="username" label="用户名">
          <el-input v-model="form.username" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item prop="name" label="名称">
          <el-input v-model="form.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item prop="sex" label="性别">
          <el-input v-model="form.sex" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item prop="birth" label="生日">
          <el-input v-model="form.birth" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item prop="phone" label="手机">
          <el-input v-model="form.phone" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item prop="address" label="地址">
          <el-input type="textarea" v-model="form.address" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="身份">
          <el-select v-model="form.role" placeholder="身份">
            <el-option label="用户" value="USER"></el-option>
            <el-option label="管理员" value="ADMIN"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="积分">
          <el-input v-model="form.account" autocomplete="off"></el-input>
        </el-form-item>

      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button type="primary" @click="save">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
export default {
  name: "User",
  data() {
    return {
      tableData: [],
      total: 0,
      pageNum: 1,
      name: "",
      form: {},
      dialogFormVisible: false,
      multipleSelection: [],
      user: localStorage.getItem("user") ? JSON.parse(localStorage.getItem("user")) : {},
      rules: {
        username: [
          {required: true, message: '请输入必填项', trigger: 'blur'}
        ],
        name: [
          {required: true, message: '请输入必填项', trigger: 'blur'}
        ],
      }
    }
  },
  created() {
    this.load()
  },
  methods: {
    load() {
      this.$request.get("/user/page", {
        params: {
          pageNum: this.pageNum,
          pageSize: 10,
          name: this.name,
        }
      }).then(res => {
        this.tableData = res.data?.records
        this.total = res.data?.total
      })
    },
    save() {
      this.$refs['ruleForm'].validate((valid) => {
        if (valid) {
          if (!this.form.password) {
            this.form.password = '123'
          }
          this.$request({
            method: this.form.id ? 'PUT' : 'POST',
            url: "/user",
            data: this.form
          }).then(res => {
            if (res.code === '200') {
              this.$message.success("操作成功")
              this.dialogFormVisible = false
              this.load()
            } else {
              this.$message.error(res.msg)
            }
          })
        }
      })
    },
    handleAdd() {
      this.dialogFormVisible = true
      this.form = {}
      this.$nextTick(() => {
        if (this.$refs.img) {
          this.$refs.img.clearFiles();
        }
        if (this.$refs.file) {
          this.$refs.file.clearFiles();
        }
      })
    },
    handleEdit(row) {
      this.form = JSON.parse(JSON.stringify(row))
      this.dialogFormVisible = true
      this.$nextTick(() => {
        if (this.$refs.img) {
          this.$refs.img.clearFiles();
        }
        if (this.$refs.file) {
          this.$refs.file.clearFiles();
        }
      })
    },
    del(id) {
      this.$request.delete("/user/" + id).then(res => {
        if (res.code === '200') {
          this.$message.success("操作成功")
          this.load()
        } else {
          this.$message.error(res.msg)
        }
      })
    },
    handleSelectionChange(val) {
      console.log(val)
      this.multipleSelection = val
    },
    delBatch() {
      if (!this.multipleSelection.length) {
        this.$message.error("请选择需要删除的数据")
        return
      }
      let ids = this.multipleSelection.map(v => v.id)  // [{}, {}, {}] => [1,2,3]
      this.$request.post("/user/del/batch", ids).then(res => {
        if (res.code === '200') {
          this.$message.success("操作成功")
          this.load()
        } else {
          this.$message.error(res.msg)
        }
      })
    },
    reset() {
      this.name = ""
      this.load()
    },
    handleCurrentChange(pageNum) {
      this.pageNum = pageNum
      this.load()
    },
    resetPass(row) {
      row.password = '123'
      this.updateUser(row)
    },
    disableUser(row) {
      let obj = Object.assign({}, row)
      obj.disable = !!obj.disable
      this.updateUser(obj)
    },
    updateUser(user) {
      this.$request({
        method: 'PUT',
        url: "/user",
        data: user
      }).then(res => {
        if (res.code === '200') {
          this.$message.success("操作成功")
          this.load()
        } else {
          this.$message.error(res.msg)
        }
      })
    },
    handleFileUploadSuccess(res) {
      this.form.file = res
    },
    handleImgUploadSuccess(res) {
      this.form.img = res
    },
    download(url) {
      window.open(url)
    },
    exp() {
      window.open($baseUrl + "/user/export")
    },
    handleExcelImportSuccess() {
      this.$message.success("导入成功")
      this.load()
    }
  }
}
</script>


<style scope>

</style>
