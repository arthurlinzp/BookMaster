<template>
  <div>
    <div style="margin-bottom: 10px">
      <el-input style="width: 260px" placeholder="请输入商品名称" v-model="name"></el-input>
      <el-button style="margin-left: 10px" type="primary" @click="load">搜 索</el-button>
      <el-button type="info" @click="reset">重 置</el-button>
      <el-span style="margin-left: 20px; color: orange; font-weight: bold" v-if="user.role === 'USER' ">您目前剩余{{ user?.account }}个积分</el-span>

    </div>

    <div style="margin: 10px 0" v-if="user.role === 'ADMIN'">
      <el-button type="primary" @click="handleAdd">新 增</el-button>
      <el-popconfirm
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
      <el-table-column align="center" label="商品图片">
        <template v-slot="scope">
          <el-image style="width: 50px; height: 50px" :src="scope.row.img"
                    :preview-src-list="[scope.row.img]"></el-image>
        </template>
      </el-table-column>
      <el-table-column align="center" prop="name" label="商品名称" show-overflow-tooltip></el-table-column>
      <el-table-column align="center" prop="price" label="所需积分">
        <template v-slot="scope">
          <span style="color: red">{{ scope.row.price }}</span>
        </template>
      </el-table-column>
      <el-table-column align="center" prop="store" label="剩余数量"></el-table-column>
      <el-table-column label="操作" width="100" align="center" v-if="user.role === 'USER'">
        <template v-slot="scope">
          <el-button type="success" @click="borrow(scope.row)" size="mini">积分兑换</el-button>
        </template>
      </el-table-column>
      <el-table-column label="上架" width="100" align="center" v-if="user.role === 'ADMIN'">
        <template v-slot="scope">
          <el-switch v-model="scope.row.active" @change="changeActive(scope.row)"
                     style="margin-right: 10px"></el-switch>
        </template>
      </el-table-column>
      <el-table-column label="操作" width="180" align="center" v-if="user.role === 'ADMIN'">
        <template v-slot="scope">
          <el-button type="primary" @click="handleEdit(scope.row)" size="mini">编辑</el-button>
          <el-popconfirm
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

    <el-dialog title="输入兑换数量" :visible.sync="dialogFormVisible1" width="30%" :close-on-click-modal="false">
      <el-form label-width="100px" style="padding-right: 40px" :model="exchangeForm" ref="exchangeFormRef">
        <el-form-item prop="num" label="兑换数量">
          <el-input-number :min="1" :max="10" v-model="exchangeForm.num" autocomplete="off"></el-input-number>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible1 = false">取 消</el-button>
        <el-button type="primary" @click="saveBorrow">确 定</el-button>
      </div>
    </el-dialog>


    <el-dialog title="信息" :visible.sync="dialogFormVisible" width="30%" :close-on-click-modal="false">
      <el-form label-width="100px" style="padding-right: 40px" :model="form" :rules="rules" ref="ruleForm">
        <el-form-item prop="name" label="商品名称">
          <el-input v-model="form.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item prop="price" label="所需积分">
          <el-input v-model="form.price" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item prop="img" label="封面">
          <el-upload :action="$baseUrl +'/file/upload'" ref="img" :on-success="handleImgUploadSuccess">
            <el-button size="small" type="primary">点击上传</el-button>
          </el-upload>
        </el-form-item>
        <el-form-item prop="store" label="剩余数量">
          <el-input v-model="form.store" autocomplete="off"></el-input>
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
  name: "Book",
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
        name: [
          {required: true, message: '请输入必填项', trigger: 'blur'}
        ],
      },
      exchangeForm: {
        num: 1
      },
      dialogFormVisible1: false,
    }
  },
  created() {
    this.load()
    this.$request.get('/user/' + this.user.id).then(res => this.user = res.data)
  },
  methods: {
    changeActive(row) {
      this.$request.put('/goods', row).then(res => {
        if (res.code === '200') {
          this.$message.success('设置成功')
        } else {
          this.$message.error(res.msg)
        }
      })
    },
    saveBorrow() {
      this.$request.post('/goodsExchange', this.exchangeForm).then(res => {
        if (res.code === '200') {
          this.$message.success('兑换成功')
          this.load()
          this.dialogFormVisible1 = false
        } else {
          this.$message.error(res.msg)
        }
      })
    },
    borrow(row) {
      this.dialogFormVisible1 = true
      this.exchangeForm = {num: 1, goodsId: row.id}
    },
    load() {
      this.$request.get("/goods/page", {
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
          this.$request({
            method: this.form.id ? 'PUT' : 'POST',
            url: "/goods",
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
      this.$request.delete("/goods/" + id).then(res => {
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
      this.$request.post("/goods/del/batch", ids).then(res => {
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
      window.open($baseUrl + "/goods/export")
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
