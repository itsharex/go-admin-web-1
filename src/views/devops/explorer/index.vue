<template>
  <div id="container" class="container">
    <div class="head">
      <el-icon class="arrow-icon" size="24" @click="goBack" style="margin-left: 10px;"><i class="iconfont icon-arrow-left arrow" :class="x.pathStack.length === 0 ? 'disable': ''"></i></el-icon>
      <span class="path">{{x.crumbs}}</span>
      <span style="flex: auto;"></span>
      <div class="refresh" @click="refresh">
        <el-icon :class="x.isRefreshing ? 'is-loading' : ''" color="#8a8a8a" size="18">
          <Refresh/>
        </el-icon>
      </div>
      <el-button class="upload_btn" type="primary" size="small" @click="uploadDialog" :disabled="!hasPermission('explorer:upload')">上传</el-button>
    </div>
    <div class="list" v-if="x.entries.length > 0">
      <div class="item_v" v-for="item in x.entries" :key="item.name">
        <div class="item" v-if="item.type === 'd'">
          <el-dropdown trigger="contextmenu" @command="handleCommand">
            <el-icon size="60" @dblclick="openDir(item.name)"><i class="iconfont icon-ex-dir dir"></i></el-icon>
            <template #dropdown>
              <el-dropdown-menu>
                <el-dropdown-item :command="'OPEN:'+item.name">打开</el-dropdown-item>
                <el-dropdown-item :command="'DELETE:'+item.name" :disabled="!hasPermission('explorer:delete')">删除</el-dropdown-item>
              </el-dropdown-menu>
            </template>
          </el-dropdown>
          <span :title="item.name" class="name">{{item.name}}</span>
        </div>
        <div class="item" v-else>
          <el-dropdown trigger="contextmenu" @command="handleCommand">
            <el-icon size="60"><i class="iconfont icon-ex-file file"></i></el-icon>
            <template #dropdown>
              <el-dropdown-menu>
                <el-dropdown-item :command="'DELETE:'+item.name" :disabled="!hasPermission('explorer:delete')">删除</el-dropdown-item>
                <el-dropdown-item :command="'DOWNLOAD:'+item.name" :disabled="!hasPermission('explorer:download')">下载</el-dropdown-item>
              </el-dropdown-menu>
            </template>
          </el-dropdown>
          <span :title="item.name" class="name">{{item.name}}</span>
        </div>
      </div>
    </div>
    <div class="empty-dir" v-else>空目录</div>
    <!--删除弹框-->
    <el-dialog v-model="x.showDeleteDialog" title="操作提示" width="30%">
      <span>删除后不可恢复，是否确定删除 ？</span>
      <template #footer>
        <el-button @click="x.showDeleteDialog = false">取消</el-button>
        <el-button type="primary" :loading="x.btnState.isLoading" @click="doDelete">确定</el-button>
      </template>
    </el-dialog>
    <!--上传弹框-->
    <el-dialog v-model="x.showUploadDialog" title="上传提示" width="30%">
      <el-upload ref="upload" drag :action="x.uploadUrl" name="file" method="post" v-model:file-list="x.uploadFileList" :data="x.formUploadData" :auto-upload="true" :limit="1" :on-exceed="handleExceed" :before-upload="beforeUpload" :on-success="handleUploadSuccess" :on-error="handleUploadError">
        <el-icon class="el-icon--upload"><upload-filled /></el-icon>
        <div class="el-upload__text">
          拖拽上传或 <em>点击上传</em>
          <div style="margin-top: 10px; color: #e6a23c; font-size: 12px;">注意：上传重名文件会自动覆盖</div>
        </div>
      </el-upload>
    </el-dialog>
  </div>
</template>
<script setup>
import {onMounted, reactive, ref} from "vue"
import {hasPermission} from "@/utils/permissions"
import "@xterm/xterm/css/xterm.css"
import {serverPaths, config} from "@/settings"
import uiUtils from "@/utils/ui-utils"
import httpUtils from "@/utils/http-utils"
import {ElLoading} from "element-plus"

const loadingConfig = {
  background: "rgba(255,255,255,0.3)",
  target: "#container"
}
const x = reactive({
  btnState: uiUtils.buttonState(),
  root: "/",
  crumbs: "",
  uploadUrl: serverPaths.explorerUpload,
  formUploadData: {
    dir: ""
  },
  uploadFileList: [],
  pathStack: [],
  searchItems: {
    dir: "",
    keyword: ""
  },
  entries: [],
  showDeleteDialog: false,
  showUploadDialog: false,
  opFileName: "",
  isRefreshing: false
})
const upload = ref()

onMounted(() => {
  let parentDir = getParentDir();
  search(parentDir, () => {
    showCrumbs()
  })
})

function showCrumbs() {
  x.crumbs = getParentDir().replaceAll("/", " / ")
}

function search(parentDir, successCallback) {
  x.searchItems.dir = parentDir
  let params = x.searchItems
  const loadingInstance = ElLoading.service(loadingConfig)
  httpUtils.get(serverPaths.explorerEntries, params, function (resList) {
    if (resList) {
      x.entries = resList
    } else {
      x.entries = []
    }
    loadingInstance.close()
    if (typeof successCallback === "function") {
      successCallback()
    }
  }, () => {
    loadingInstance.close()
  })
}

function refresh() {
  if (x.isRefreshing) {
    return
  }
  x.isRefreshing = true
  let parentDir = getParentDir();
  search(parentDir, () => {
    showCrumbs()
    x.isRefreshing = false
    uiUtils.showToast("success", "刷新成功")
  })
}

function handleCommand(command) {
  let parts = command.split(":");
  let op = parts[0]
  x.opFileName = parts[1]
  if (op === "OPEN") {
    openDir(x.opFileName)
    return
  }
  if (op === "DELETE") {
    x.showDeleteDialog = true
    return
  }
  if (op === "DOWNLOAD") {
    downloadFile(x.opFileName)
  }
}

function openDir(dirName) {
  let parentDir = getParentDir();
  if (parentDir === x.root) {
    search(parentDir + dirName, () => {
      x.pathStack.push({
        index: x.pathStack.length + 1,
        name: dirName
      })
      showCrumbs()
    })
  } else {
    search(parentDir + "/" + dirName, () => {
      x.pathStack.push({
        index: x.pathStack.length + 1,
        name: dirName
      })
      showCrumbs()
    })
  }
}

function downloadFile(fileName) {
  let parentDir = getParentDir();
  let filePath = ""
  if (parentDir === x.root) {
    filePath = parentDir + fileName
  } else {
    filePath = parentDir + "/" + fileName
  }
  const link = document.createElement("a")
  link.href = serverPaths.explorerDownload + "?path=" + filePath
  document.body.appendChild(link);
  link.click()
  document.body.removeChild(link);
}

function getParentDir() {
  if (x.pathStack.length === 0) {
    return x.root
  }
  let names = []
  for (const item of x.pathStack) {
    names.push(item.name)
  }
  return x.root + names.join("/")
}

function doDelete() {
  let parentDir = getParentDir();
  let filePath = ""
  if (parentDir === x.root) {
    filePath = parentDir + x.opFileName
  } else {
    filePath = parentDir + "/" + x.opFileName
  }
  let params = {
    path: filePath
  }
  httpUtils.delete(serverPaths.explorerDelete, params, function () {
    x.showDeleteDialog = false
    uiUtils.showToast("success", "删除成功")
    search(parentDir)
  })
}

function goBack() {
  if (x.pathStack.length > 0) {
    let parentDir = getParentDir();
    let backDir = parentDir.substring(0, parentDir.lastIndexOf("/"))
    if (backDir + "/" === x.root) {
      backDir = x.root
    }
    search(backDir, () => {
      x.pathStack.pop()
      showCrumbs()
    })
  }
}

function uploadDialog() {
  if (upload.value) {
    upload.value.clearFiles()
  }
  x.showUploadDialog = true
}

function beforeUpload () {
  x.formUploadData.dir = getParentDir()
  return true
}

function handleExceed(files) {
  upload.value.clearFiles()
  upload.value.handleStart(files[0])
  upload.value.submit()
}

function handleUploadSuccess(res) {
  if (res.code === 200) {
    x.showUploadDialog = false
    uiUtils.showToast("success", "上传成功")
    search(getParentDir())
  } else {
    uiUtils.showToast("error", res.message)
  }
}

function handleUploadError(err) {
  console.error(err)
  uiUtils.showToast("error", err.toString())
}

</script>
<style scoped src="../../../assets/css/devops/explorer.css">

</style>