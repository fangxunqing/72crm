<template>
  <create-view :body-style="{height: '100%'}">
    <div
      v-loading="newLoading"
      class="new-journal">
      <div
        slot="header"
        class="header">
        <span class="text">{{ dialogTitle }}</span>
        <img
          class="el-icon-close rt"
          src="@/assets/img/task_close.png"
          alt=""
          @click="close">
      </div>
      <div class="content">
        <el-tabs
          v-model="activeName"
          @tab-click="tabClick">
          <el-tab-pane
            v-for="(item, index) in tabsData"
            :label="item.label"
            :name="item.key"
            :key="index"/>
        </el-tabs>
        <div class="form">
          <div
            v-for="(item, index) in formList"
            :key="index"
            class="row-list">
            <label class="item-label">{{ item.label }}：</label>
            <el-input
              :autosize="{ minRows: 4}"
              v-model="formData[item.model]"
              type="textarea"
              placeholder="请输入内容"
              resize="none"/>
          </div>
          <!-- 图片附件 -->
          <div class="img-accessory">
            <div class="img-box">
              <el-upload
                ref="imageUpload"
                :action="crmFileSaveUrl"
                :headers="httpHeader"
                :on-preview="handleFilePreview"
                :before-remove="beforeRemove"
                :on-success="imgFileUploadSuccess"
                :file-list="imageFileList"
                name="img[]"
                multiple
                accept="image/*"
                list-type="picture-card">
                <p class="add-img">
                  <span class="el-icon-picture"/>
                  <span>添加图片</span>
                </p>
                <i class="el-icon-plus"/>
              </el-upload>
            </div>
            <p class="add-accessory">
              <el-upload
                ref="fileUpload"
                :action="crmFileSaveUrl"
                :headers="httpHeader"
                :on-preview="handleFilePreview"
                :before-remove="handleFileRemove"
                :on-success="fileUploadSuccess"
                :file-list="fileList"
                name="file[]"
                multiple
                accept="*.*">
                <p>
                  <img
                    src="@/assets/img/relevance_file.png"
                    alt="">
                  添加附件
                </p>
              </el-upload>
            </p>
          </div>
          <div class="sent-who">
            <span class="label">发送给谁:</span>
            <div
              v-photo="k"
              v-lazy:background-image="$options.filters.filterUserLazyImg(k.thumb_img)"
              v-for="(k, j) in formData.sentWhoList"
              :key="j"
              class="div-photo k-img header-circle"/>
            <span>
              <span
                v-for="(item, index) in formData.depData"
                :key="index"
                class="item-name">{{ item.name }}</span>
            </span>
            <members-dep
              :user-checked-data="formData.sentWhoList"
              :dep-checked-data="formData.depData"
              :content-block="false"
              @popoverSubmit="popoverSubmit">
              <img
                slot="membersDep"
                class="sent-img"
                src="@/assets/img/task_add.png">
            </members-dep>
          </div>
          <related-business
            :margin-left="'0'"
            :all-data="allData"
            @checkInfos="checkInfos"/>
        </div>
      </div>
      <div class="btn-group">
        <el-button
          type="primary"
          @click="submitBtn">提交</el-button>
        <el-button @click="close">取消</el-button>
      </div>
    </div>
  </create-view>
</template>

<script>
import axios from 'axios'
import { crmFileDelete, crmFileSaveUrl } from '@/api/common'
import CreateView from '@/components/CreateView'
// 部门员工优化版
import membersDep from '@/components/selectEmployee/membersDep'
// 关联业务 - 选中列表
import relatedBusiness from '@/components/relatedBusiness'
export default {
  components: {
    CreateView,
    membersDep,
    relatedBusiness
  },
  props: {
    formData: {
      type: Object,
      default: () => {
        return {}
      }
    },
    dialogTitle: {
      type: String,
      default: '写日志'
    },
    // 附件
    accessoryFileList: {
      type: Array,
      default: () => {
        return []
      }
    },
    imgFileList: {
      type: Array,
      default: () => {
        return []
      }
    },
    newLoading: Boolean
  },
  data() {
    return {
      activeName: '1',
      tabsData: [
        { label: '日报', key: '1' },
        { label: '周报', key: '2' },
        { label: '月报', key: '3' }
      ],
      // 表格数据
      formList: [],
      dateList: [
        { label: '今日工作内容', model: 'content' },
        { label: '明日工作内容', model: 'tomorrow' },
        { label: '遇到的问题', model: 'question' }
      ],
      weekList: [
        { label: '本周工作内容', model: 'content' },
        { label: '下周工作内容', model: 'tomorrow' },
        { label: '遇到的问题', model: 'question' }
      ],
      monthList: [
        { label: '本月工作内容', model: 'content' },
        { label: '下月工作内容', model: 'tomorrow' },
        { label: '遇到的问题', model: 'question' }
      ],
      imageFileList: [],
      fileList: [],
      // 发送给谁
      dialogVisible: false,
      // 获取选择的数据id数组
      relevanceAll: {},
      allData: {}
    }
  },
  computed: {
    crmFileSaveUrl() {
      return window.BASE_URL + crmFileSaveUrl
    },
    httpHeader() {
      return {
        authKey: axios.defaults.headers.authKey,
        sessionId: axios.defaults.headers.sessionId
      }
    }
  },
  mounted() {
    document.body.appendChild(this.$el)
    this.formList = this.dateList

    // 确定显示哪一种日志
    if (this.formData.category_id) {
      switch (this.formData.category_id) {
        case 1:
          this.tabsData = [{ label: '日报', key: '1' }]
          this.formList = this.dateList
          break
        case 2:
          this.tabsData = [{ label: '周报', key: '1' }]
          this.formList = this.weekList
          break
        case 3:
          this.tabsData = [{ label: '月报', key: '1' }]
          this.formList = this.monthList
          break
      }
    }
    // 编辑时引用 - 自动勾选
    var allData = {}
    allData.business = this.formData.businessList
      ? this.formData.businessList
      : []
    allData.contacts = this.formData.contactsList
      ? this.formData.contactsList
      : []
    allData.contract = this.formData.contractList
      ? this.formData.contractList
      : []
    allData.customer = this.formData.customerList
      ? this.formData.customerList
      : []
    this.allData = allData
    var relevanceAll = {}
    relevanceAll.business_ids = this.formData.businessList
      ? this.formData.businessList.map((item, index, array) => {
        return item.business_id
      })
      : []
    relevanceAll.contacts_ids = this.formData.contactsList
      ? this.formData.contactsList.map((item, index, array) => {
        return item.contacts_id
      })
      : []
    relevanceAll.contract_ids = this.formData.contractList
      ? this.formData.contractList.map((item, index, array) => {
        return item.contract_id
      })
      : []
    relevanceAll.customer_ids = this.formData.customerList
      ? this.formData.customerList.map((item, index, array) => {
        return item.customer_id
      })
      : []
    this.relevanceAll = relevanceAll

    this.imageFileList = this.imgFileList.map(function(item, index, array) {
      item.url = item.file_path_thumb
      return item
    })
    this.fileList = this.accessoryFileList.map(function(item, index, array) {
      item.url = item.file_path_thumb
      return item
    })
  },
  destroyed() {
    // remove DOM node after destroy
    if (this.$el && this.$el.parentNode) {
      this.$el.parentNode.removeChild(this.$el)
    }
  },
  methods: {
    close() {
      if (this.$route.query.routerKey == 1) {
        this.$router.go(-1)
      } else {
        this.$emit('close')
      }
    },
    tabClick() {
      switch (this.activeName) {
        case '1':
          this.formList = this.dateList
          break
        case '2':
          this.formList = this.weekList
          break
        case '3':
          this.formList = this.monthList
          break
      }
    },
    // 提交按钮
    submitBtn() {
      if (
        this.formData.content ||
        this.formData.tomorrow ||
        this.formData.question
      ) {
        this.$emit(
          'submitBtn',
          this.activeName,
          this.fileList,
          this.imageFileList,
          this.relevanceAll
        )
      } else {
        this.$message.error('内容至少填写一项')
      }
    },
    // 图片和附件
    // 上传图片
    imgFileUploadSuccess(response, file, fileList) {
      this.imageFileList = fileList
    },
    // 查看图片
    handleFilePreview(file) {
      if (file.response || file.file_id) {
        let perviewFile
        if (file.response) {
          perviewFile = {
            url: file.response.data[0].path,
            name: file.response.data[0].name
          }
        } else {
          perviewFile = {
            url: file.file_path,
            name: file.name
          }
        }
        this.$bus.emit('preview-image-bus', {
          index: 0,
          data: [perviewFile]
        })
      }
    },
    beforeRemove(file, fileList) {
      if (file.response || file.file_id) {
        let save_name
        if (file.response) {
          save_name = file.response.data[0].save_name
        } else {
          save_name = file.save_name
        }
        this.$confirm('您确定要删除该文件吗?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        })
          .then(() => {
            crmFileDelete({
              save_name: save_name
            })
              .then(res => {
                this.$message.success(res.data)
                var removeIndex = this.getFileIndex(
                  this.$refs.imageUpload.uploadFiles,
                  save_name
                )
                if (removeIndex != -1) {
                  this.$refs.imageUpload.uploadFiles.splice(removeIndex, 1)
                }
                removeIndex = this.getFileIndex(this.imgFileList, save_name)
                if (removeIndex != -1) {
                  this.imgFileList.splice(removeIndex, 1)
                }
              })
              .catch(() => {})
          })
          .catch(() => {
            this.$message({
              type: 'info',
              message: '已取消操作'
            })
          })
        return false
      } else {
        return true
      }
    },
    // 附件索引
    getFileIndex(files, save_name) {
      var removeIndex = -1
      for (let index = 0; index < files.length; index++) {
        const item = files[index]
        let item_save_name
        if (item.response) {
          item_save_name = item.response.data[0].save_name
        } else {
          item_save_name = item.save_name
        }
        if (item_save_name == save_name) {
          removeIndex = index
          break
        }
      }
      return removeIndex
    },
    fileUploadSuccess(response, file, fileList) {
      this.fileList = fileList
    },
    handleFileRemove(file, fileList) {
      if (file.response || file.file_id) {
        let save_name
        if (file.response) {
          save_name = file.response.data[0].save_name
        } else {
          save_name = file.save_name
        }
        this.$confirm('您确定要删除该文件吗?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        })
          .then(() => {
            crmFileDelete({
              save_name: save_name
            })
              .then(res => {
                this.$message.success(res.data)
                var removeIndex = this.getFileIndex(
                  this.$refs.fileUpload.uploadFiles,
                  save_name
                )
                if (removeIndex != -1) {
                  this.$refs.fileUpload.uploadFiles.splice(removeIndex, 1)
                }
                removeIndex = this.getFileIndex(this.fileList, save_name)
                if (removeIndex != -1) {
                  this.fileList.splice(removeIndex, 1)
                }
              })
              .catch(() => {})
          })
          .catch(() => {
            this.$message({
              type: 'info',
              message: '已取消操作'
            })
          })
        return false
      } else {
        return true
      }
    },
    popoverSubmit(members, dep) {
      this.$set(this.formData, 'sentWhoList', members)
      this.$set(this.formData, 'depData', dep)
    },
    // 取消
    handleClose() {
      this.dialogVisible = false
    },
    checkInfos(val) {
      this.relevanceAll = val
    }
  }
}
</script>

<style scoped lang="scss">
.new-journal {
  display: flex;
  flex-direction: column;
  height: 100%;
  .header {
    height: 40px;
    line-height: 40px;
    padding: 0 0 0 10px;
    .el-icon-close {
      margin-right: 0;
      width: 40px;
      line-height: 40px;
      padding: 10px;
      cursor: pointer;
    }
    .text {
      font-size: 17px;
    }
  }
  .content {
    flex: 1;
    display: flex;
    flex-direction: column;
    overflow: auto;
    padding: 20px;
  }
  .btn-group {
    text-align: right;
    padding-right: 20px;
  }
  .content /deep/ .el-tabs {
    .el-tabs__header {
      .el-tabs__item {
        height: 50px;
        line-height: 50px;
      }
      .el-tabs__nav {
        margin-left: 20px;
        font-size: 13px;
      }
      .el-tabs__nav-wrap::after {
        height: 1px;
      }
    }
    .el-tabs__content {
      padding: 0 20px;
    }
  }
}

.form {
  flex: 1;
  margin-top: 10px;
  padding: 0 20px;
  overflow-y: scroll;
  .row-list {
    margin-bottom: 20px;
    padding-bottom: 10px;
    .item-label {
      margin-bottom: 9px;
      display: block;
      font-size: 12px;
      // padding-bottom: 10px;
    }
    .el-textarea {
      .el-textarea__inner {
        resize: none;
      }
    }
  }
  .img-accessory {
    font-size: 12px;
    img {
      vertical-align: middle;
    }
    .img-box /deep/ .el-upload {
      width: 80px;
      height: 80px;
      line-height: 90px;
    }
    .img-box /deep/ .el-upload-list {
      .el-upload-list__item {
        width: 80px;
        height: 80px;
      }
    }
    .img-box {
      position: relative;
      margin-top: 40px;
      .add-img {
        position: absolute;
        left: 0;
        top: -30px;
        height: 20px;
        line-height: 20px;
        margin-bottom: 10px;
        color: #3e84e9;
      }
    }
    .add-accessory {
      margin-top: 25px;
      margin-bottom: 20px;
      color: #3e84e9;
    }
  }
  .sent-who {
    margin-bottom: 15px;
    .label,
    img,
    .k-img {
      vertical-align: middle;
    }
    .label {
      margin-right: 15px;
      font-size: 12px;
    }
    img {
      cursor: pointer;
    }
    .item-name {
      margin-right: 7px;
    }
    .k-img {
      width: 25px;
      height: 25px;
      border-radius: 17.5px;
      margin-right: 7px;
    }
    .must {
      color: #f56c6c;
      font-size: 12px;
      margin-top: 5px;
    }
    .sent-img {
      width: 24px;
      height: 24px;
    }
  }
}
</style>
