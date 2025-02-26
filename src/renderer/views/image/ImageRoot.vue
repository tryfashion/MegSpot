<template>
  <div class="home" flex="dir:top">
    <div class="tool" flex="main:justify cross:center">
      <div class="tool-items">
        <el-button @click="addFolder" type="text" class="tool-item add-folder" title="add folder to root" size="small">
          {{ $t('image.toolbar.addFolder') }}
        </el-button>
        <SelectedBtn :selectedList="imageList" @update="setImages" @remove="removeImages" @click="emptyImages" />
        <el-button
          type="primary"
          round
          :title="`${$t('common.hotKey')}：cmd/ctrl+enter`"
          class="tool-item"
          :disabled="!imageList.length"
          @click="compare"
        >
          {{ $t('general.compare') }}
        </el-button>
        <el-button
          type="primary"
          round
          title="drag-drop-compare"
          class="tool-item"
          :disabled="!imageList.length"
          @click="dragDropCompare"
        >
          {{ $t('general.dragDropCompare') }}
        </el-button>
        <el-button type="primary" round class="tool-item" @click="loadShareProject">
          {{ $t('image.toolbar.loadShareProject') }}
        </el-button>
        <!-- <el-button
          type="text"
          size="small"
          title="line-Browse"
          class="tool-item"
          :disabled="!imageList.length"
          @click="imageBrowser"
        >
          {{ $t('general.imageBrowser') }}
        </el-button> -->
      </div>
    </div>

    <div class="content" flex-box="1">
      <Split :gutterSize="4">
        <SplitArea :size="23" :maxSize="1000" :minSize="200">
          <FileTree
            ref="folderTree"
            id="folderTree"
            :currentPath="currentPath"
            :openedFolders="imageFolders"
            :checkedFiles="checkedFiles"
            @close="onClose"
            :addFolder="addFolder"
            :defaultExpand="expandData"
            @select="onSelect"
            @addExpand="addExpandData"
            @removeExpand="removeExpandData"
          ></FileTree>
        </SplitArea>
        <SplitArea :size="77">
          <ImagePreview ref="ImagePreview" @addFolder="addFolder"></ImagePreview>
        </SplitArea>
      </Split>
    </div>
  </div>
</template>

<script>
import store from '@/store'
const { dialog } = require('@electron/remote')
import FileTree from '@/components/file-tree/FileTree.vue'
import SelectedBtn from '@/components/selected-btn'
import ShowPath from '@/components/show-path'
import ImagePreview from './ImagePreview'
import { SnapshotHelper } from '@/tools/compress'
import { createNamespacedHelpers } from 'vuex'
const { mapGetters, mapActions } = createNamespacedHelpers('imageStore')
import addDragFolderListener from '@/utils/dragFolder.js'
import { handleEvent } from '@/tools/hotkey'

export default {
  name: 'ImageRoot',
  components: { FileTree, ImagePreview, SelectedBtn, ShowPath },
  data() {
    return {
      isDraging: false,
      checkedFiles: [],
      dragFiles: [],
      hotkeyEvents: undefined
    }
  },
  computed: {
    ...mapGetters(['imageList', 'imageFolders', 'expandData', 'currentPath'])
  },
  mounted() {
    this.initHotkeyEvents()
    addDragFolderListener(document.getElementById('folderTree'))
  },
  activated() {
    window.addEventListener('keydown', this.handleHotKey, true)
  },
  deactivated() {
    window.removeEventListener('keydown', this.handleHotKey, true)
  },
  methods: {
    ...mapActions([
      'removeImages',
      'emptyImages',
      'setImageFolders',
      'setImages',
      'setFolderPath',
      'addExpandData',
      'removeExpandData'
    ]),
    initHotkeyEvents() {
      this.hotkeyEvents = new Map([
        [
          'gotoCompare',
          () => {
            this.compare()
          }
        ],
        [
          'emptyAll',
          () => {
            this.emptyImages()
          }
        ]
      ])
    },
    handleHotKey(event) {
      handleEvent(event, this.hotkeyEvents)
    },
    compare() {
      this.$router.push('/image/compare')
    },
    dragDropCompare() {
      this.$router.push('/image/drag-drop-compare')
    },
    imageBrowser() {
      this.$router.push('/image/browser')
    },
    addFolder() {
      dialog
        .showOpenDialog({
          title: 'add folder',
          properties: ['openDirectory']
        })
        .then(({ canceled, filePaths }) => {
          if (!filePaths || !(filePaths.length > 0)) {
            this.$message.info('Cancelled to add folder')
          } else if (this.imageFolders.includes(filePaths[0])) {
            this.$message.info('The folder has been added.')
          } else {
            this.setImageFolders([...this.imageFolders, filePaths[0]])
            this.$nextTick(() => {
              this.$message.success('Successed to add folder')
            })
          }
        })
    },
    onClose(data) {
      const index = this.imageFolders.findIndex((item) => {
        return item === data.path
      })
      if (index >= 0) {
        const tmp = [...this.imageFolders]
        tmp.splice(index, 1)
        this.setImageFolders(tmp)
      }
      const removeList = []
      this.imageList.forEach((item) => {
        if (item.startsWith(data.path)) {
          removeList.push(item)
        }
      })
      this.removeImages(removeList)
      this.setFolderPath('')
    },
    onSelect(data) {
      // 如果选中了文件夹 更新当前激活的文件夹
      if (data.type === 0) {
        this.$nextTick(() => {
          // 等左侧导航树先渲染
          this.setFolderPath(data.path)
        })
      }
    },
    async loadShareProject() {
      const snapshotHelper = new SnapshotHelper()
      const res = await snapshotHelper.load()
      console.log('res', res)
      if (res) {
        const { config, files } = res
        this.$router.push({
          path: '/image/compare',
          query: { snapshotMode: true }
        })
        console.log('store', store, { config, files })
        store.dispatch('imageSnapshotStore/setSnapshotConfig', config)
        store.dispatch('imageSnapshotStore/setFiles', files)
      } else {
        this.$message.info('cancel')
      }
    }
  }
}
</script>

<style scoped lang="scss">
@import '@/styles/variables.scss';

.home {
  color: $labelColor;
  height: 100%;
  width: 100%;
  margin: 0px;
  padding: 0px;
  box-sizing: border-box;
  position: relative;

  .tool {
    padding: 0 10px;
    border-bottom: 1px solid #eee;

    .tool-items {
      position: relative;
    }

    .tool-item + .tool-item {
      margin-left: 10px;
    }
  }

  .content {
    overflow: auto;
  }
}
</style>
