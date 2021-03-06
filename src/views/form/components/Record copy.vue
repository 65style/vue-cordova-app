<template>
  <div class="about">
    <div>
      <input
        type="file"
        :disabled="status !== Status.wait"
        @change="handleFileChange"
      />
      <van-button @click="handleUpload" :disabled="uploadDisabled"
        >上传</van-button
      >
      <van-button @click="handleResume" v-if="status === Status.pause"
        >恢复</van-button
      >
      <van-button
        v-else
        :disabled="status !== Status.uploading || !container.hash"
        @click="handlePause"
        >暂停</van-button
      >
    </div>
    <div>
      <div>计算文件 hash</div>
      <van-progress :percentage="hashPercentage"></van-progress>
      <div>总进度</div>
      <van-progress :percentage="fakeUploadPercentage"></van-progress>
    </div>
    <el-table :data="data">
      <el-table-column
        prop="hash"
        label="切片hash"
        align="center"
      ></el-table-column>
      <el-table-column label="大小(KB)" align="center" width="120">
        <template v-slot="{ row }">
          {{ row.size | transformByte }}
        </template>
      </el-table-column>
      <el-table-column label="进度" align="center">
        <template v-slot="{ row }">
          <van-progress
            :percentage="row.percentage"
            color="#909399"
          ></van-progress>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
const SIZE = 10 * 1024 * 1024 // 切片大小
const Status = {
  wait: 'wait',
  pause: 'pause',
  uploading: 'uploading',
}
export default {
  name: 'app',
  filters: {
    transformByte(val) {
      return Number((val / 1024).toFixed(0))
    },
  },
  props: {
    rz: {
      type: Object,
      required: true,
    },
    cus: {
      type: Object,
      required: true,
    },
  },
  data: () => ({
    mediaRec: null,
    Status,
    container: {
      file: null,
      hash: '',
      worker: null,
    },
    hashPercentage: 0,
    data: [],
    requestList: [],
    status: Status.wait,
    // 当暂停时会取消 xhr 导致进度条后退
    // 为了避免这种情况，需要定义一个假的进度条
    fakeUploadPercentage: 0,
  }),
  computed: {
    fileName() {
      return new Date().getTime() + '.m4a'
    },
    filePath() {
      if (this.$device) {
        console.log('this.$cordova.file: ', this.$cordova.file)
        if (this.$device.platform === 'iOS') {
          return this.$cordova.file.tempDirectory
        } else {
          return this.$cordova.file.externalRootDirectory
        }
      } else {
        return ''
      }
    },
    uploadDisabled() {
      return (
        !this.container.file ||
        [Status.pause, Status.uploading].includes(this.status)
      )
    },
    uploadPercentage() {
      if (!this.container.file || !this.data.length) return 0
      const loaded = this.data
        .map(item => item.size * item.percentage)
        .reduce((acc, cur) => acc + cur)
      return parseInt((loaded / this.container.file.size).toFixed(2))
    },
  },
  watch: {
    uploadPercentage(now) {
      if (now > this.fakeUploadPercentage) {
        this.fakeUploadPercentage = now
      }
    },
  },
  mounted() {
    this.record()
  },
  methods: {
    record() {
      this.$toast('开始录音')
      console.log(this.fullPath)
      this.mediaRec = new this.$Media(
        this.fileName,
        success => {
          console.log('success: ', success)
        },
        err => {
          console.log('err: ', err)
        },
        this.mediaError,
      )
      // 启动录制音频
      this.mediaRec.startRecord()
    },
    stop() {
      this.$toast('结束录音')
      console.log('this.mediaRec: ', this.mediaRec)
      this.mediaRec.stopRecord()
      this.getFile().then(file => {
        console.log('getfile: ', file)
        const formData = new FormData()
        formData.append('file', file)
        formData.append('cid', this.cus.KEYID)
        formData.append('rzcusname', this.cus.CNAME)
        formData.append('userid', this.$store.state.userInfo.userId)
        formData.append('compid', this.$store.state.userInfo.compid)
        formData.append('tandanbuchong', this.rz.tandanbuchong)

        this.$http.uploadRecord(formData).then(res => {
          console.log('res: ', res)
        })
      })
    },
    getFile() {
      const fullPath = this.filePath + this.fileName
      console.log('fullPath: ', fullPath)
      return new Promise((reslove, reject) => {
        window.resolveLocalFileSystemURL(
          fullPath,
          dirEntry => {
            dirEntry.file(file => {
              console.log('filesssss: ', file)
              const fileReader = new FileReader()
              fileReader.onloadend = function() {
                fileReader.readAsArrayBuffer(file)
                console.log(file)
              }
              reslove(file)
            })
          },
          err => {
            reject(err)
          },
        )
      })
    },

    handlePause() {
      this.status = Status.pause
      this.resetData()
    },
    resetData() {
      this.requestList.forEach(xhr => xhr?.abort())
      this.requestList = []
      if (this.container.worker) {
        this.container.worker.onmessage = null
      }
    },
    async handleResume() {
      this.status = Status.uploading
      const { uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash,
      )
      await this.uploadChunks(uploadedList)
    },
    // xhr
    request({
      url,
      method = 'post',
      data,
      headers = {},
      onProgress = e => e,
      requestList,
    }) {
      return new Promise(resolve => {
        const xhr = new XMLHttpRequest()
        xhr.upload.onprogress = onProgress
        xhr.open(method, url)
        Object.keys(headers).forEach(key =>
          xhr.setRequestHeader(key, headers[key]),
        )
        xhr.send(data)
        xhr.onload = e => {
          // 将请求成功的 xhr 从列表中删除
          if (requestList) {
            const xhrIndex = requestList.findIndex(item => item === xhr)
            requestList.splice(xhrIndex, 1)
          }
          resolve({
            data: e.target.response,
          })
        }
        // 暴露当前 xhr 给外部
        requestList?.push(xhr)
      })
    },
    // 生成文件切片
    createFileChunk(file, size = SIZE) {
      const fileChunkList = []
      let cur = 0
      while (cur < file.size) {
        fileChunkList.push({ file: file.slice(cur, cur + size) })
        cur += size
      }
      return fileChunkList
    },
    // 生成文件 hash（web-worker）
    calculateHash(fileChunkList) {
      return new Promise(resolve => {
        this.container.worker = new Worker('/hash.js')
        this.container.worker.postMessage({ fileChunkList })
        this.container.worker.onmessage = e => {
          const { percentage, hash } = e.data
          this.hashPercentage = percentage
          if (hash) {
            resolve(hash)
          }
        }
      })
    },
    handleFileChange(e) {
      const [file] = e.target.files
      if (!file) return
      this.resetData()
      Object.assign(this.$data, this.$options.data())
      this.container.file = file
    },
    async handleUpload() {
      if (!this.container.file) return
      this.status = Status.uploading
      const fileChunkList = this.createFileChunk(this.container.file)
      this.container.hash = await this.calculateHash(fileChunkList)
      const { shouldUpload, uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash,
      )
      if (!shouldUpload) {
        this.$message.success('秒传：上传成功')
        this.status = Status.wait
        return
      }
      this.data = fileChunkList.map(({ file }, index) => ({
        fileHash: this.container.hash,
        index,
        hash: this.container.hash + '-' + index,
        chunk: file,
        size: file.size,
        percentage: uploadedList.includes(index) ? 100 : 0,
      }))
      await this.uploadChunks(uploadedList)
    },
    // 上传切片，同时过滤已上传的切片
    async uploadChunks(uploadedList = []) {
      const requestList = this.data
        .filter(({ hash }) => !uploadedList.includes(hash))
        .map(({ chunk, hash, index }) => {
          const formData = new FormData()
          formData.append('chunk', chunk)
          formData.append('hash', hash)
          formData.append('filename', this.container.file.name)
          formData.append('fileHash', this.container.hash)
          return { formData, index }
        })
        .map(async ({ formData, index }) =>
          this.request({
            url: 'http://localhost:3000',
            data: formData,
            onProgress: this.createProgressHandler(this.data[index]),
            requestList: this.requestList,
          }),
        )
      await Promise.all(requestList)
      // 之前上传的切片数量 + 本次上传的切片数量 = 所有切片数量时
      // 合并切片
      if (uploadedList.length + requestList.length === this.data.length) {
        await this.mergeRequest()
      }
    },
    // 通知服务端合并切片
    async mergeRequest() {
      await this.request({
        url: 'http://localhost:3000/merge',
        headers: {
          'content-type': 'application/json',
        },
        data: JSON.stringify({
          size: SIZE,
          fileHash: this.container.hash,
          filename: this.container.file.name,
        }),
      })
      this.$message.success('上传成功')
      this.status = Status.wait
    },
    // 根据 hash 验证文件是否曾经已经被上传过
    // 没有才进行上传
    async verifyUpload(filename, fileHash) {
      const { data } = await this.request({
        url: 'http://localhost:3000/verify',
        headers: {
          'content-type': 'application/json',
        },
        data: JSON.stringify({
          filename,
          fileHash,
        }),
      })
      return JSON.parse(data)
    },
    // 用闭包保存每个 chunk 的进度数据
    createProgressHandler(item) {
      return e => {
        item.percentage = parseInt(String((e.loaded / e.total) * 100))
      }
    },
  },
}
</script>
