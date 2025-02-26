<template>
  <div class="sort-toolbar-container">
    <el-tooltip :content="$t(this.allSelectd ? 'general.clearAll' : 'general.selectAll')" :open-delay="300">
      <vxe-checkbox :value="allSelectd" :indeterminate="oneOrMoreSelected" @change="handleSelectAll"></vxe-checkbox>
    </el-tooltip>
    <el-button v-show="generateVisible" v-tip="'Generate Sorting File'" @click="generateSortFile">
      {{ $t('sortFile.generate') }}
    </el-button>
    <span v-show="generateVisible" class="tip">{{ $t('sortFile.generateTip') }}</span>
    <!-- <el-button
      v-show="!generateVisible"
      :disabled="btnDisabled"
      v-tip="'Apply Sorting File'"
      @click="applySortFile"
    >
      {{ $t('sortFile.apply') }}
    </el-button> -->
    <el-button v-show="!generateVisible" v-tip="'Edit Sorting File'" @click="editSortFile">
      {{ $t('sortFile.edit') }}
    </el-button>
  </div>
</template>

<script>
import fse from 'fs-extra'
import getFileName from '@/filter/get-file-name'
const { dialog } = require('@electron/remote')
import { EOF, DELIMITER, SORTING_FILE_NAME } from '@/constants'

export default {
  components: {},
  props: {
    currentPath: String,
    btnDisabled: {
      type: Boolean,
      default: true
    },
    allSelectd: {
      type: Boolean,
      default: false
    },
    oneOrMoreSelected: {
      type: Boolean,
      default: false,
      // 监听当前文件夹下排序文件的变化, 变化后改变按钮状态
      wacther: undefined
    }
  },
  data() {
    return {
      generateVisible: false
    }
  },
  created() {},
  mounted() {},
  beforeDestroy() {
    this.wacther && this.wacther.close()
    this.wacther = null
  },
  methods: {
    handleSelectAll(...rest) {
      this.$emit('change', ...rest)
    },
    saveFile(file) {
      dialog
        .showOpenDialog({
          title: 'select folder',
          defaultPath: this.currentPath,
          filters: [{ name: 'Custom File Type', extensions: ['gif'] }]
        })
        .then((result) => {
          fse.writeFileSync(result.filePath, dataBuffer)
          this.$message.success('success')
        })
    },
    async applySortFile() {
      this.$bus.$emit('applySortFile')
    },
    editSortFile() {
      this.$emit('showDialog')
    },
    async generateSortFile() {
      const path = this.sortFilePath
      let exist = await fse.pathExists(path)
      let cancel = false
      if (exist) {
        this.$confirm('排序文件已存在，是否覆盖', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).catch(() => {
          cancel = true
          this.$message({
            type: 'info',
            message: '已取消覆盖'
          })
        })
      }
      if (cancel) return
      // 获取当前表格排序列表数据
      const data = await new Promise((resolve) =>
        this.$emit('getSortData', null, (data) => resolve(data.map((item) => getFileName(item.path, false)).join(EOF)))
      )
      // 写入文件
      try {
        await fse.outputFile(path, data)
        this.$message({
          type: 'success',
          message: `The sorting file is successfully generated !<br /> and its path is <a style="color: blue;">${path}</a>`,
          dangerouslyUseHTMLString: true
        })
        this.generateVisible = false
        const _data = await fse.readFile(path, 'utf8')
      } catch (err) {
        console.error(err)
        this.$message.error(err)
      }
    },
    async confirmBtnVisible() {
      const exists = await fse.pathExists(this.sortFilePath)
      this.generateVisible = !exists
      // 更改dialog中的排序列表
      if (exists) {
        this.$bus.$emit('fileChanged')
      }
    }
  },
  computed: {
    sortFilePath() {
      return this.currentPath + DELIMITER + SORTING_FILE_NAME
    }
  }
}
</script>

<style lang="scss" scoped>
@import '@/styles/public.scss';
.sort-toolbar-container {
  margin: 5px 0;
}
</style>
