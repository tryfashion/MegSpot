<template>
  <div :class="['image-canvas', { selected: selected }]" @click.stop>
    <div ref="header" class="header" flex="cross:center">
      <CoverMask :mask="maskDom" class="cover-mask">
        <HistContainer ref="hist-container" v-tip="$t('general.histogram')" @changeVisible="handleHistVisible" />
      </CoverMask>
      <el-tooltip placement="bottom" :open-delay="800">
        <span class="compare-name" flex-box="1" v-html="getTitle"></span>
        <div slot="content">
          {{ path }}
          <br />
          <br />
          <span class="size">{{ bitMap && bitMap.width }} x {{ bitMap && bitMap.height }}</span>
        </div>
      </el-tooltip>
      <RGBAExhibit :RGBAcolor="RGBAcolor"></RGBAExhibit>
      <EffectPreview @change="changeCanvasStyle" />
    </div>
    <div ref="container" class="canvas-container" id="canvas-container" :style="canvasStyle">
      <OperationContainer
        id="canvas-container"
        ref="canvas-container"
        @drag="handleDrag"
        @zoom="handleZoom"
        @scrollEnd="handleZoomEnd"
        @dbclick="handleDbclick"
        @mouseMove="handleMove"
      >
        <div v-loading="loading" element-loading-background="rgba(0, 0, 0, 0)" class="canvas-item" @contextmenu.prevent>
          <ScaleEditor
            v-if="preference.showScale"
            class="scale-editor"
            :scale="imgScale"
            :scaleEditorVisible.sync="scaleEditorVisible"
            @show="showScaleEditor"
            @reset="resetZoom"
            @update="setZoom"
          />
          <canvas ref="canvas" :width="_width" :height="_height"></canvas>
          <div ref="feedback" id="feedback" v-show="traggerRGB"></div>
          <div v-if="preference.showMousePos" class="mouse-position" v-show="showPosition && mousePosInfo.x !== ''">
            <span>x={{ mousePosInfo.x }},y={{ mousePosInfo.y }}</span>
          </div>
        </div>
      </OperationContainer>
    </div>
  </div>
</template>
<script>
import Vue from 'vue'
const cv = Vue.prototype.$cv
import fse from 'fs-extra'
import OperationContainer from '@/components/operation-container'
import HistContainer from '@/components/hist-container'
import CoverMask from '@/components/cover-mask'
import RGBAExhibit from '@/components/rgba-exhibit'
import ScaleEditor from '@/components/scale-editor'
import EffectPreview from '@/components/effect-preview'
import { createNamespacedHelpers } from 'vuex'
const { mapGetters, mapActions } = createNamespacedHelpers('imageStore')
const { mapGetters: preferenceMapGetters } = createNamespacedHelpers('preferenceStore')
import { getImageUrlSyncNoCache } from '@/utils/image'
import { throttle } from '@/utils'
import { SCALE_CONSTANTS, DRAG_CONSTANTS } from '@/constants'
import chokidar from 'chokidar'
import { getFileName } from '@/filter/get-file-name'
import { decode, decodeImage, toRGBA8 } from 'utif2'

export default {
  components: {
    OperationContainer,
    HistContainer,
    CoverMask,
    RGBAExhibit,
    ScaleEditor,
    EffectPreview
  },
  props: {
    _width: {
      type: Number,
      default: 500
    },
    _height: {
      type: Number,
      default: 500
    },
    path: {
      type: String,
      default: ''
    },
    // name: {
    //   type: String,
    //   default: ''
    // },
    // imageData: {
    //   type: [Object, Blob],
    //   default: () => {}
    // },
    snapInfo: {
      type: Object,
      default: () => {}
    }
  },
  inject: ['getSnapshotMode'],
  data() {
    return {
      // 监听图像文件的变化,变化后自动刷新图像
      wacther: undefined,
      loading: false,
      header: null,
      canvas: null,
      maskDom: undefined,
      currentHist: undefined,
      histImage: null,
      cs: null,
      image: {
        width: 0,
        height: 0
      },
      bitMap: null,
      imagePosition: null,
      cachedPositionData: null,
      imgScale: 'N/A',
      scaleEditorVisible: false,
      afterFullSize: false,
      selectedId: null,
      // 原图的mat 缓存用于便捷获取
      syncCanvasActions: [
        {
          event: 'imageCenter_resetCanvas',
          action: 'reset'
        },
        {
          event: 'imageCenter_rotate',
          action: 'rotate'
        },
        {
          event: 'imageCenter_reverse',
          action: 'reverse'
        }
      ],
      // 广播调度事件
      scheduleCanvasActions: [
        {
          event: 'image_broadcast',
          action: 'handleBroadcast'
        },
        {
          event: 'image_handleSelect',
          action: 'handleSelect'
        },
        {
          event: 'imageCenter_getSelectedPosition',
          action: 'getSelectedPosition'
        },
        {
          event: 'imageCenter_align',
          action: 'align'
        },
        {
          event: 'radius',
          action: 'setRadius'
        }
      ],
      histVisible: true,
      traggerRGB: false,
      radius: 10,
      RGBAcolor: {
        R: 0,
        G: 0,
        B: 0,
        A: 0
      },
      showPosition: true,
      mousePosInfo: {
        x: '',
        y: ''
      }
    }
  },
  computed: {
    ...mapGetters(['imageConfig', 'imageList']),
    ...preferenceMapGetters(['preference']),
    snapshotMode() {
      return this.getSnapshotMode() || false
    },
    selected() {
      return this.path === this.selectedId
    },
    getTitle() {
      return this.preference.showTitle
        ? (this.selected ? `<span style='color: red'>(✔)</span>` : ``) + this.getName()
        : ' '
    },
    canvasStyle() {
      return this.preference.background.style
    }
  },
  async mounted() {
    this.header = this.$refs.header
    this.canvas = this.$refs.canvas
    this.initCanvas()
    this.initImage()
    this.listenEvents()
  },
  beforeDestroy() {
    this.removeEvents()
    this.bitMap && this.bitMap.close()
  },
  watch: {
    path: {
      handler: function (newVal, oldVal) {
        if (oldVal) {
          this.wacther && this.wacther.close()
        }
        if (newVal) {
          this.wacther = chokidar
            .watch(newVal, {
              // 持续监听
              persistent: true,
              // 忽略初始化的目录检测
              ignoreInitial: true,
              // 等待写入完成
              awaitWriteFinish: {
                stabilityThreshold: 2000,
                pollInterval: 100
              }
            })
            .on('change', (path, details) => {
              console.log('image--change', path, details)
              this.initImage(false)
            })
            .on('unlink', (path, details) => {
              console.log('image--remove', path, details)
              this.removeImages(path)
            })
        }
      },
      immediate: true
    },
    'imageConfig.smooth': {
      handler(newVal, oldVal) {
        this.setSmooth()
      },
      immediate: true
    }
  },
  methods: {
    ...mapActions(['removeImages']),
    // 检查边界， 保证图像至少部分在canvas内(显示大小至少为当前图像大小的DRAG_CONSTANTS)
    checkBorder(transX, transY, _width, _height) {
      const cw = this._width,
        ch = this._height,
        iw = _width ?? this.imagePosition.width,
        ih = _height ?? this.imagePosition.height
      const constantsW = DRAG_CONSTANTS * (cw < iw ? cw : iw),
        constantsH = DRAG_CONSTANTS * (ch < ih ? ch : ih)

      let isFullFilled = transX <= constantsW && transY <= constantsH && transX + iw >= cw && transY + ih >= ch // 判断图像是否占据整个canvas
      if (isFullFilled) return true

      let isTooLeft = transX + iw < constantsW // 图像是否过于偏左
      let isTooRight = transX > cw - constantsW // 图像是否过于偏右
      let isTooTop = transY + ih < constantsH // 图像是否过于偏上
      let isTooBottom = transY > ch - constantsH // 图像是否过于偏下
      return !(isTooLeft || isTooRight || isTooTop || isTooBottom)
    },
    // 约束缩放，保证图像的宽(或高)不小于canvas宽(或高)的SCALE_CONSTANTS
    checkSize(transW, transH) {
      let isTooSmall = transW < this.canvas.width * SCALE_CONSTANTS || transH < this.canvas.height * SCALE_CONSTANTS
      if (this.afterFullSize && !isTooSmall) {
        this.afterFullSize = false
      }
      return this.afterFullSize || !isTooSmall
    },
    setCanvasStyle({ style }) {
      this.canvas.style.filter = style
    },
    setSmooth() {
      if (this.cs) {
        this.cs.imageSmoothingEnabled = this.imageConfig.smooth
        this.reDraw()
      }
    },
    getSnapshot() {
      // 叠加显示时候 生成快照
      return {
        snapShot: this.canvas,
        hist: this.currentHist
      }
    },
    getName(filter = true) {
      return this.snapshotMode
        ? this.snapInfo.name
        : filter
        ? this.$options.filters.getFileName(this.path)
        : getFileName(this.path)
    },
    listenEvents() {
      // 广播调度事件
      this.scheduleCanvasActions.forEach((item) => {
        this.$bus.$on(item.event, this[item.action])
      })
      // 分发事件 执行各个子组件的方法 同步状态
      this.syncCanvasActions.forEach((item) => {
        this.$bus.$on(item.event, ({ name, data }) => {
          this.dispatchCanvasAction({ name, data })
        })
      })
      // 鼠标点击时隐藏编辑scale的输入框
      this.$refs.container.addEventListener(
        'click',
        () => {
          this.scaleEditorVisible = false
        },
        false
      )
    },
    removeEvents() {
      this.scheduleCanvasActions.forEach((item) => {
        this.$bus.$off(item.event, this[item.action])
      })
      // 分发事件 执行各个子组件的方法 同步状态
      this.syncCanvasActions.forEach((item) => {
        this.$bus.$off(item.event, this[item.action])
      })
    },
    dispatchCanvasAction({ name, data }) {
      if (!this.selectedId || this.selectedId === this.path) {
        this[name](data)
      }
    },
    handleBroadcast({ name, data }) {
      if (this.selectedId) {
        if (data.id === this.path || this.selectedId === this.path) {
          this[name](data)
        }
      } else {
        this[name](data)
      }
    },
    initHist() {
      if (this.image && this.image.width) {
        this.currentHist = this.$refs['hist-container'].generateHist(cv.imread(this.image))
      }
    },
    async initBitMap(imageData) {
      if (this.bitMap) {
        this.bitMap.close()
      }
      this.bitMap = await createImageBitmap(
        imageData
          ? new ImageData(new Uint8ClampedArray(imageData), this.image.width, this.image.height, { colorSpace: 'srgb' })
          : this.image
      )
    },
    resolvePath() {
      return new Promise((resolve) => {
        if (this.snapshotMode) {
          fetch(this.path)
            .then(() => {
              resolve(this.path)
            })
            .catch((err) => {
              // TODO: A: reload后imageData为空，（从vuex）取imageData
              // B: 或退回文件浏览器页
              console.log(err)

              // method A
              // const path = URL.createObjectURL(
              //   new Blob([this.snapInfo.imageData])
              // );
              // this.image.src = path;

              // method B
              this.$router.push({
                path: '/image/index'
              })
            })
        } else {
          resolve(getImageUrlSyncNoCache(this.path)) //        'C:/Demo/1-1%20-%20副本.jpg'
        }
      })
    },
    async initImage(initPosition = true) {
      this.loading = true
      if (/tiff?$/.test(this.path)) {
        const file = await fse.readFile(this.path)
        const arraybuffer = file.buffer
        const ifds = decode(arraybuffer)
        const ifd = ifds[0]
        decodeImage(file, ifd)
        const imageData = toRGBA8(ifd)
        this.image = new Image(ifd.width, ifd.height)
        await this.initBitMap(imageData)
        this.loading = false
        this.reDraw(initPosition)
      } else {
        this.image = new Image()
        this.image.onload = async () => {
          this.initBitMap()
          this.loading = false
          this.reDraw(initPosition)
        }
        this.image.src = await this.resolvePath()
      }
    },
    initCanvas() {
      this.cs = this.canvas.getContext('2d')
      this.$nextTick(() => {
        this.cs.imageSmoothingEnabled = this.imageConfig.smooth
      })
    },
    // 供外部直接调用 待测试
    reMount() {
      this.initCanvas()
      this.initImage()
    },
    drawImage() {
      let { x, y, width, height } = this.imagePosition
      this.cs.clearRect(0, 0, this.canvas.width, this.canvas.height)
      this.cs.drawImage(this.bitMap, x, y, width, height)
    },
    handleDbclick() {
      if (!this.selected) {
        this.$bus.$emit('image_handleSelect', this.path)
      } else {
        this.$bus.$emit('image_handleSelect', null)
      }
    },
    handleSelect(selectedId) {
      this.selectedId = selectedId
    },
    handleHistVisible(visible) {
      this.broadCast({
        name: 'doHistVisible',
        data: { visible }
      })
    },
    async doHistVisible({ visible }) {
      if (!this.currentHist) {
        await this.initHist()
      }
      this.$refs['hist-container'].setVisible(visible)
    },
    handleScaleDbClick(data) {
      console.log('handleScaleDbClick', Object.values(arguments))
      this.scaleEditorVisible = !this.scaleEditorVisible
    },
    handleScaleReset() {
      console.log('handleScaleReset')
    },
    changeZoom(data) {
      console.log('changeZoom', data)
    },
    pickColor({ status }) {
      this.traggerRGB = status
    },
    changeCanvasStyle(newStyle) {
      // FIXME: 具体的值没有被广播
      this.broadCast({
        name: 'setCanvasStyle',
        data: { style: newStyle }
      })
    },
    setRadius(radius) {
      this.radius = radius
    },
    handleMove: throttle(40, function (mousePos) {
      this.broadCast({
        name: 'doHandleMove',
        data: { mousePos }
      })
    }),
    doHandleMove({ mousePos }) {
      this.preference.showMousePos && this.changeMousePosInfo(mousePos)
      this.preference.showScale && this.changeRGBA(mousePos)
      // console.log(`${this.getName()}-changeMousePosInfo`, mousePos, {
      //   ...this.mousePosInfo
      // });
    },
    changeRGBA(mousePos) {
      if (!this.traggerRGB) return
      const { x, y } = mousePos
      const feedback = this.$refs.feedback
      feedback.style.left = x - this.radius + 'px'
      feedback.style.top = y - this.radius + 'px'
      feedback.style.width = this.radius * 2 + 'px'
      feedback.style.height = this.radius * 2 + 'px'
      Promise.resolve().then(() => {
        const data = this.cs.getImageData(x, y, this.radius, this.radius).data
        const count = data.length / 4
        let r = 0,
          g = 0,
          b = 0,
          a = 0
        for (var i = 0; i < count; i++) {
          r += data[i * 4]
          g += data[i * 4 + 1]
          b += data[i * 4 + 2]
          a += data[i * 4 + 3]
        }
        this.RGBAcolor = {
          R: parseInt(r / count),
          G: parseInt(g / count),
          B: parseInt(b / count),
          A: parseInt(a / count)
        }
      })
    },
    changeMousePosInfo(mousePos) {
      if (!this.imagePosition?.x || !this.imagePosition?.width) {
        this.mousePosInfo = {
          x: '',
          y: ''
        }
        return
      }
      const { x, y } = mousePos
      const { x: imageX, y: imageY, width, height } = this.imagePosition
      const isOutside = x < imageX || y < imageY || x > imageX + width || y > imageY + height
      const originWidth = this.image.width
      const originHeight = this.image.height
      if (!isOutside && originWidth && originHeight) {
        const posX = ((x - imageX) * originWidth) / width
        const posY = ((y - imageY) * originHeight) / height
        this.mousePosInfo = {
          x: Number(posX).toFixed(2),
          y: Number(posY).toFixed(2)
        }
      } else {
        this.mousePosInfo = {
          x: '',
          y: ''
        }
      }
    },
    // 外部直接调用
    setCoverStatus({ snapShot, hist }, status) {
      if (status) {
        this.cs.clearRect(0, 0, this.canvas.width, this.canvas.height)
        this.cs.drawImage(snapShot, 0, 0)
        if (this.$refs['hist-container'].visible) {
          this.maskDom = hist
        }
      } else {
        this.maskDom = null
        this.drawImage()
      }
    },
    reset(val) {
      if (val) {
        let x = 0,
          y = 0
        let width = this.image.width
        let height = this.image.height
        this.afterFullSize = true
        this.imagePosition = { x, y, width, height }
        this.doZoomEnd()
        this.drawImage()
      } else {
        this.imagePosition = this.getImageInitPos(this.canvas, this.bitMap)
        this.doZoomEnd()
        this.drawImage()
      }
    },
    reDraw(init = false) {
      window.requestAnimationFrame(() => {
        if (init) {
          this.imagePosition = this.snapshotMode
            ? this.snapshotModeInitPos()
            : this.getImageInitPos(this.canvas, this.bitMap)
          this.doZoomEnd()
        }
        this.drawImage()
      })
    },
    snapshotModeInitPos() {
      const positionInfo = {}
      const { _width: snap_width, _height: snap_height, imagePosition } = this.snapInfo
      const { x, y, width, height } = imagePosition
      const base = this._width / this._height <= snap_width / snap_height ? '_width' : '_height'
      const baseLength = this.snapInfo[base] // _width or _height
      positionInfo.x = Number(x / baseLength) * this[base]
      positionInfo.y = Number(y / baseLength) * this[base]
      positionInfo.width = Number(width / baseLength) * this[base]
      positionInfo.height = Number(height / baseLength) * this[base]
      // console.log('snapshotModeInitPos', positionInfo);
      return positionInfo
    },
    getImageInitPos(canvas, image) {
      const cw = canvas.width
      const ch = canvas.height

      const iw = image.width
      const ih = image.height

      const canvasRadio = cw / ch
      const imageRadio = iw / ih

      let x = 0
      let y = 0
      let height = ch
      let width = cw

      if (canvasRadio > imageRadio) {
        //比较高，所以高占100%,宽居中
        width = canvas.height * imageRadio
        x = (canvas.width - width) / 2
      } else {
        //比较宽，所以宽占100%,高居中
        height = canvas.width / imageRadio
        y = (canvas.height - height) / 2
      }
      return {
        x,
        y,
        width,
        height
      }
    },
    doDrag(data) {
      if (this.imagePosition == null) return
      let offset = data.offset
      let transX = this.imagePosition.x + offset.x
      let transY = this.imagePosition.y + offset.y
      // console.log('offset', offset)
      if (this.checkBorder(transX, transY)) {
        // 判断是否只在指定范围内拖动
        this.imagePosition.x = transX
        this.imagePosition.y = transY
        this.drawImage()
      }
    },
    // 判断图像是否占据整个canvas
    isFullFilled() {
      const cw = this.canvas.width,
        ch = this.canvas.height,
        iw = this.imagePosition.width,
        ih = this.imagePosition.height
      const constantsW = DRAG_CONSTANTS * iw,
        constantsH = DRAG_CONSTANTS * ih
      const { x: transX, y: transY } = this.imagePosition
      return transX <= constantsW && transY <= constantsH && transX + iw >= cw && transY + ih >= ch
    },
    /******************ScaleEditor START******************/
    showScaleEditor() {
      this.scaleEditorVisible = true
      this.cachedPositionData = {
        scale: this.imgScale,
        imagePosition: { ...this.imagePosition }
      }
    },
    resetZoom() {
      const { scale, imagePosition } = this.cachedPositionData
      this.imagePosition = imagePosition
      this.cachedPositionData = {
        scale: scale,
        imagePosition: { ...imagePosition }
      }
      this.imgScale = scale
      this.drawImage()
    },
    setZoom(scale) {
      if (scale == this.imgScale) {
        this.imgScale = scale // 必需, 触发scaleEditor组件更新
        return
      }
      const oldScale = this.imgScale ?? scale
      const scaleRatio = scale / oldScale
      // 默认从画布中心放大
      const position = { x: this.canvas.width / 2, y: this.canvas.height / 2 }
      // 画像不占据整个画布时，从画像中心放大
      // const isFullFilled = this.isFullFilled();
      // const position = isFullFilled
      //   ? { x: this.canvas.width / 2, y: this.canvas.height / 2 }
      //   : {
      //       x: (this.imagePosition.x + this.imagePosition.width) / 2,
      //       y: (this.imagePosition.y + this.imagePosition.height) / 2
      //     };
      this.doZoom({
        rate: scaleRatio,
        mousePos: position
      })
      this.imgScale = scale
    },
    /******************ScaleEditor END******************/
    doZoom(data) {
      if (this.imagePosition == null) return
      let mousePos = data.mousePos
      let rate = data.rate
      let x = mousePos.x - (mousePos.x - this.imagePosition.x) * rate
      let y = mousePos.y - (mousePos.y - this.imagePosition.y) * rate
      let height = this.imagePosition.height * rate
      let width = this.imagePosition.width * rate
      // 缩小时才检查显示图片是否过小
      if ((rate > 1 || this.checkSize(width, height)) && this.checkBorder(x, y, width, height)) {
        this.imagePosition = {
          x,
          y,
          height,
          width
        }
        this.imgScale = 'N/A'
        this.drawImage()
      }
    },
    doZoomEnd() {
      this.imgScale = Number(this.imagePosition.width / this.bitMap.width).toFixed(2)
    },
    handleDrag(offset) {
      this.broadCast({
        name: 'doDrag',
        data: { offset: offset, id: this.path }
      })
    },
    handleZoom(rate, mousePos) {
      this.broadCast({
        name: 'doZoom',
        data: { rate: rate, mousePos: mousePos, id: this.path }
      })
    },
    handleZoomEnd() {
      this.imagePosition &&
        this.broadCast({
          name: 'doZoomEnd',
          data: {}
        })
    },
    broadCast({ name, data }) {
      if (!this.selected) {
        throttle(16, () => {
          this.$bus.$emit('image_broadcast', {
            name: name,
            data: data
          })
        })()
      } else {
        this[name](data)
      }
    },
    rotate(degree) {
      let offscreenWidth = this.bitMap.height
      let offscreenHight = this.bitMap.width
      let offsreen = new OffscreenCanvas(offscreenWidth, offscreenHight)
      let offCtx = offsreen.getContext('2d')
      if (degree < 0) {
        offCtx.translate(0, this.bitMap.width)
        offCtx.rotate((-90 * Math.PI) / 180)
        ;[this.imagePosition.width, this.imagePosition.height] = [this.imagePosition.height, this.imagePosition.width]
      } else if (degree > 0) {
        offCtx.translate(this.bitMap.height, 0)
        offCtx.rotate((90 * Math.PI) / 180)
        ;[this.imagePosition.width, this.imagePosition.height] = [this.imagePosition.height, this.imagePosition.width]
      }
      offCtx.drawImage(this.bitMap, 0, 0)
      this.bitMap.close()
      this.bitMap = offsreen.transferToImageBitmap()
      this.drawImage()
    },
    reverse(direction) {
      console.log('reverse', direction)
      let offscreenWidth = this.bitMap.width
      let offscreenHight = this.bitMap.height
      let offsreen = new OffscreenCanvas(offscreenWidth, offscreenHight)
      let offCtx = offsreen.getContext('2d')
      if (direction > 0) {
        //左右翻转
        offCtx.translate(this.bitMap.width, 0)
        offCtx.scale(-1, 1)
      } else if (direction < 0) {
        //上下翻转
        offCtx.translate(0, this.bitMap.height)
        offCtx.scale(1, -1)
      }
      offCtx.drawImage(this.bitMap, 0, 0)
      this.bitMap.close()
      this.bitMap = offsreen.transferToImageBitmap()
      this.drawImage()
    },
    align({ name, data }) {
      const { beSameSize, position } = data
      if (this.selectedId || (!this.selectedId && this.path !== this.imageList[0])) {
        if (beSameSize) {
          let { x, y, width, height } = position
          this.imagePosition = { x, y, width, height }
          this.afterFullSize = true
          this.doZoomEnd()
        } else {
          this.imagePosition.x = position.x
          this.imagePosition.y = position.y
        }
        this.drawImage()
      }
    },
    getSelectedPosition(data, callback) {
      if (this.selected || this.path === this.selectedId || (!this.selectedId && this.path === this.imageList[0])) {
        callback({
          id: this.path,
          position: this.imagePosition
        })
      }
    }
  }
}
</script>
<style scoped lang="scss">
@import '@/styles/variables.scss';

.image-canvas {
  .header {
    height: 18px;
    line-height: 16px;
    background-color: #f6f6f6;
    padding-right: 10px;
  }

  .canvas-container {
    .canvas-item {
      position: relative;
      width: 100%;
      height: 100%;

      img {
        object-fit: contain;
        vertical-align: middle;
      }

      ::v-deep {
        input,
        .el-input-group__append {
          border-radius: 0;
        }

        .el-input-group__append,
        .el-input-group__prepend {
          padding: 0 6px;

          &:hover {
            color: green;
          }
        }

        .el-input__inner {
          padding: 0 2px;
        }
      }

      .canvas {
        vertical-align: middle;
        font-size: 0;
      }

      /** default canvas background */
      // .canvas-style {
      //   background: #e3e7e9;
      //   background-image: linear-gradient(45deg, #f6fafc 25%, transparent 0),
      //     linear-gradient(45deg, transparent 75%, #f6fafc 0),
      //     linear-gradient(45deg, #f6fafc 25%, transparent 0),
      //     linear-gradient(45deg, transparent 75%, #f6fafc 0);
      //   background-position: 0 0, 10px 10px, 10px 10px, 20px 20px;
      //   background-size: 20px 20px;
      // }

      #feedback {
        position: absolute;
        border: 1px solid red;
      }

      .mouse-position {
        position: absolute;
        bottom: 4px;
        right: 5px;
        color: rgba(0, 0, 0, 0.6);
      }
    }
  }
}
</style>
