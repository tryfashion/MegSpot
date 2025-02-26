<template>
  <vxe-table
    ref="xTable"
    :data="showFile"
    auto-resize
    border
    align="center"
    highlight-hover-row
    highlight-current-row
    :scroll-y="{ gt: 0, rHeight: 40, oSize: 5 }"
    :checkbox-config="{
      trigger: 'cell',
      range: true,
      checkMethod: checkSelectable
    }"
    :edit-config="{ trigger: 'dblclick', mode: 'cell', showIcon: false }"
    :sort-config="{ sortMethod: customSortMethod }"
    show-overflow="tooltip"
    :tooltip-config="{ enterable: true }"
    class="table"
    header-cell-class-name="width_adaptive"
    :height="tableHeight"
    v-on="$listeners"
    @checkbox-all="selectAll"
    @checkbox-change="select"
    @checkbox-range-end="handleRangeSelect"
    @checkbox-range-change="handleRangingRender"
  >
    <template #empty>
      <span v-if="!currentPath">
        {{ $t('common.please') }}
        <el-button type="text" size="small" @click="$emit('addFolder')">
          {{ $t('sortFile.addFolder') }}
        </el-button>
        {{ $t('sortFile.afterAddFolder') }}
      </span>
      <span v-else>There is no available files in current directory.</span>
    </template>
    <vxe-column type="checkbox" width="48"></vxe-column>
    <vxe-column align="left" show-overflow="tooltip" field="name" title="Name" sortable>
      <template #header>
        <div flex="cross:center">
          <el-tooltip :content="$t('general.enableRegular')">
            <vxe-checkbox v-model="regexpEnabled"></vxe-checkbox>
          </el-tooltip>
          <search-input
            v-model="search"
            size="mini"
            placeholder="filter file name according to input"
            autofocus
            clearable
          >
            <span v-if="regexpEnabled" slot="prepend" style="font-size: 16px; color: black">/</span>
            <span v-if="regexpEnabled" slot="append" style="color: black">/ig</span>
          </search-input>
        </div>
      </template>
    </vxe-column>
    <vxe-column
      sortable
      show-overflow
      field="lastModifyTime"
      :title="$t('general.lastModifyTime')"
      :formatter="formatter"
    ></vxe-column>
    <vxe-column sortable show-overflow :formatter="formatter" :title="$t('general.size')" field="size"></vxe-column>
  </vxe-table>
</template>

<script>
import fse from 'fs-extra'
import dayjs from 'dayjs'
import { arraySortByName } from '@/utils/file'
import { createNamespacedHelpers } from 'vuex'
import {  debounce } from '@/utils'
import { formatFileSize } from '@/utils/file'
import SearchInput from '../search-input'
import { isImage, isVideo } from '@/components/file-tree/lib/util'
const { mapActions: imageMapActions } = createNamespacedHelpers('imageStore')
const { mapActions: videoMapActions } = createNamespacedHelpers('videoStore')
import { isDirectory, readDir, getFileStatSync } from '@/utils/file'
import { EOF, DELIMITER, SORTING_FILE_NAME } from '@/constants'
import chokidar from 'chokidar'

export default {
  name: 'FileTable',
  components: { SearchInput },
  props: {
    defaultSort: {
      required: true,
      type: Object
    },
    showAll: {
      type: Boolean,
      default: false
    },
    fileList: {
      type: Array,
      default: () => {
        return []
      }
    },
    currentPath: {
      type: String,
      default: ''
    },
    checkItem: {
      type: Function,
      default: () => {}
    },
    addVuexItem: {
      type: Function,
      default: () => {}
    },
    removeVuexItem: {
      type: Function,
      default: () => {}
    },
    emptyVuexItems: {
      type: Function,
      default: () => {}
    }
  },
  data() {
    return {
      tableHeight: 1000,
      searchString: '',
      regexpEnabled: false, // 采用正则表达式方法搜索
      fileInfoList: [],
      sortList: [],
      origin: -1,
      pin: false, // 按下shift
      // 监听当前文件夹的变化,变化后手动刷新目录
      wacther: undefined,
      canApply: false
    }
  },
  computed: {
    search: {
      get() {
        return this.searchString
      },
      set: debounce(100, function (newVal) {
        this.searchString = newVal
      })
    },
    showFile: {
      get() {
        // 过滤排序文件，根据当前是否showAll以及支持文件类型，正则等进行过滤
        const data = this.fileInfoList.filter((item) => {
          if (item.name === SORTING_FILE_NAME) return false
          let result1 = false,
            result2 = false
          if (this.showAll || (!this.showAll && this.checkItem(item))) {
            result1 = true
          }

          try {
            result2 =
              !this.search ||
              (this.regexpEnabled
                ? new RegExp(this.search, 'ig').test(item.name)
                : item.name.toString().toLowerCase().indexOf(this.search.toLowerCase()) > -1)
          } catch {
            result2 = item.name.toString().toLowerCase().indexOf(this.search.toLowerCase()) > -1
          }
          return result1 && result2
        })
        return this.mySort(data, this.defaultSort?.field || 'name', this.defaultSort?.order || 'asc')
      },
      set(data) {
        console.log('setData', data)
      }
    },
    sortFilePath() {
      return this.currentPath + DELIMITER + SORTING_FILE_NAME
    }
  },
  async mounted() {
    window.addEventListener('resize', debounce(300, this.updateTableHeight))
    window.addEventListener('keydown', (code) => {
      // 开启多选
      if (code.keyCode === 16 && code.shiftKey) {
        this.pin = true
      }
    })
    window.addEventListener('keyup', (code) => {
      // 关闭多选
      if (code.keyCode === 16) {
        this.pin = false
      }
    })
    this.$bus.$on('applySortFile', this.applySortFile)
    this.$nextTick(() => {
      this.updateTableHeight()
    })
  },
  updated() {
    // 打开同步选中
    this.fileList.forEach((path) => {
      const item = this.showFile.find((item) => item.path === path)
      if (item) {
        this.$refs.xTable.setCheckboxRow(item, true)
      }
    })
  },
  beforeDestroy() {
    this.$bus.$off('applySortFile', this.applySortFile)
    this.wacther && this.wacther.close()
    this.wacther = null
  },
  watch: {
    currentPath: {
      handler: async function (newVal, oldVal) {
        if (oldVal) {
          this.wacther.close()
        }
        if (newVal) {
          this.wacther = chokidar
            .watch(newVal, {
              // 持续监听
              persistent: true,
              // 忽略初始化的目录检测（即：认为监听时目录是从空变为当前目录的过程 会触发很多的addDir,add file回调）
              ignoreInitial: true,
              // 等待写入完成
              awaitWriteFinish: {
                stabilityThreshold: 2000,
                pollInterval: 100
              }
            })
            .on('all', async (event, path) => {
              // console.log('FileTable watcher:', event, path);
              if (path === this.sortFilePath) {
                const exist = await fse.pathExists(path)
                if (exist) {
                  const data = await fse.readFile(path, 'utf8')
                  this.sortList = data.split(EOF)
                } else this.sortList = []
              }
              this.refreshFileList()
            })
        }
        const exist = await fse.pathExists(this.sortFilePath)
        if (exist) {
          const data = await fse.readFile(this.sortFilePath, 'utf8')
          this.sortList = data.split(EOF)
        } else this.sortList = []
        this.refreshFileList()
      },
      immediate: true
    },
    fileList(newVal, oldVal) {
      oldVal.forEach((path) => {
        // 同步删除
        if (!newVal.includes(path)) {
          const item = this.showFile.find((item) => item.path === path)
          if (item) {
            this.$refs.xTable.setCheckboxRow(item, false)
          }
        }
      })
      newVal.forEach((path) => {
        // 同步新增
        if (!oldVal.includes(path)) {
          const item = this.showFile.find((item) => item.path === path)
          if (item) {
            this.$refs.xTable.setCheckboxRow(item, true)
          }
        }
      })
    },
    canApply(newVal, oldVal) {
      if (newVal !== oldVal) {
        this.$emit('canApply', newVal)
      }
    }
  },
  methods: {
    ...imageMapActions(['addImages', 'removeImages']),
    ...videoMapActions(['addVideos', 'removeVideos']),
    mySort(data, property, order) {
      let list = []
      // whether to sort by filename
      if (property === 'name') {
        list = data.sort((a, b) => arraySortByName(a.name, b.name))
        // other sort filed (size,lastModifyTime)
      } else {
        list = data.sort((a, b) => a[property] - b[property])
      }
      if (this.sortList.length) {
        list = data.sort((a, b) => {
          let indexA = this.sortList.indexOf(a[property])
          let indexB = this.sortList.indexOf(b[property])
          if (indexA === -1 && indexB === -1) return a[property] < b[property]
          if (indexA === -1) return 1
          if (indexB === -1) return -1
          return indexA - indexB
        })
      }
      // reverse sort
      if (order === 'desc') {
        list.reverse()
      }
      console.log(`sort by ${property}, result:`, list)
      return list
    },
    async customSortMethod({ data, sortList, ...rest }) {
      const exist = await fse.pathExists(this.sortFilePath)
      if (!this.sortList.length && exist) {
        // if (exist) {
        const data = await fse.readFile(this.sortFilePath, 'utf8')
        this.sortList = data.split(EOF)
      }

      const sortItem = sortList[0]
      // 取出第一个排序的列
      const { property, order } = sortItem

      const list = this.mySort(data, property, order)

      if (this.searchString !== '') {
        this.canApply = false
      } else {
        new Promise(async (resolve) => {
          for (let i = 0, len = this.sortList.length; i < len; i++) {
            if (list[i].name !== this.sortList[i]) {
              resolve(true)
              break
            }
          }
          resolve(false)
        }).then((res) => {
          this.canApply = res
        })
      }

      return list
    },
    async applySortFile(data, callback) {
      await this.$refs.xTable.clearSort()
      // 重新触发排序
      this.$nextTick(() => {
        this.$refs.xTable.sort('name', 'asc').then(() => {
          // 向父级反馈 最新的顺序
          this.$emit('sort-change', this.$refs.xTable.getSortColumns()[0])
        })
      })
    },
    checkSelectable({ row }) {
      return this.checkItem(row)
    },
    formatter({ cellValue, column }) {
      let formatVal = cellValue
      switch (column.property) {
        case 'lastModifyTime':
          formatVal = dayjs(cellValue).format('YYYY-MM-DD HH:mm:ss')
          break
        case 'size':
          formatVal = formatFileSize(cellValue)
          break
        case 'duration':
          formatVal = cellValue.toFixed(2) + 's'
          break
        default:
      }
      return formatVal
    },
    updateTableHeight() {
      this.tableHeight = this.$refs.xTable.$el.parentElement.clientHeight || this.tableHeight
    },
    // 由于vxe-table 在range选择过程中 会清空已选，所以需要在选择过程中不断更新已选中。
    handleRangingRender() {
      this.fileList.forEach((path) => {
        const item = this.showFile.find((item) => item.path === path)
        if (item) {
          this.$refs.xTable.setCheckboxRow(item, true)
        }
      })
    },
    handleRangeSelect({ records }) {
      this.addVuexItem(records.map((item) => item.path))
    },
    commonSelect(row) {
      const sortData = this.getSortData() // 获取最终渲染的table数据
      const origin = this.origin // 起点行
      const currentRowIndex = sortData.indexOf(row) // 当前点击行
      let flag = sortData[origin] ? this.fileList.includes(sortData[origin].path) : false
      if (this.pin && flag) {
        // 选中 起点行 -- 到当前点击行
        let minIndex = Math.min(origin, currentRowIndex)
        let maxIndex = Math.max(origin, currentRowIndex)
        this.addVuexItem(sortData.slice(minIndex, maxIndex + 1).map((item) => item.path))
      } else {
        // 未按住shift 或无起点 只增删该行并初始化起点
        this.origin = sortData.indexOf(row)
        if (this.fileList.includes(row.path)) {
          // 已选 则删
          this.removeVuexItem(row.path)
        } else {
          // 选中
          this.addVuexItem(row.path)
        }
      }
    },
    select({ row }) {
      if (this.checkItem(row)) {
        this.commonSelect(row)
      }
    },
    selectAll({ records: items }) {
      if (items.length) {
        this.addVuexItem(items.map((item) => item.path))
      } else {
        this.emptyVuexItems()
      }
    },
    // 可外部直接调用触发逻辑
    async refreshFileList() {
      console.info(`-----------refresh file list-----------`)
      if (this.currentPath && isDirectory(this.currentPath)) {
        const files = await readDir(this.currentPath).catch((err) => {
          throw err
        })
        this.fileInfoList = files.map((item) => {
          const path = this.currentPath + DELIMITER + item
          const status = getFileStatSync(path)
          return {
            path,
            name: item,
            isFile: status.isFile(),
            lastModifyTime: status.mtime.getTime(),
            size: status.size
          }
        })
      } else {
        this.fileInfoList = []
      }
      // 重新触发排序
      this.$nextTick(() => {
        this.$refs.xTable.sort(this.defaultSort.field, this.defaultSort.order ?? 'null').then(() => {
          // 向父级反馈 最新的顺序
          this.$emit('sort-change', this.$refs.xTable.getSortColumns()[0])
        })
      })
    },
    // 供外部直接调用获取 通过排序处理后的tableData
    getSortData() {
      return this.$refs.xTable.getTableData().visibleData
    }
  }
}
</script>

<style lang="scss" scoped>
@import '@/styles/variables.scss';
.table {
  width: 100%;
  background-color: #f0f3f6;
  .del-btn:hover {
    color: red;
  }
  ::v-deep {
    .el-input-group__prepend,
    .el-input-group__append {
      padding: 0;
    }
    .width_adaptive {
      /*表头与单元格统一高度*/
      padding: 0 !important;
      height: 37px !important;
      .vxe-cell {
        display: flex;
        justify-content: center;
      }
    }
    th {
      background-color: #f6f6f6 !important;
    }
  }
}
</style>
