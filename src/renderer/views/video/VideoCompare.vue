<template>
  <div class="video-center" flex="dir:top box:first">
    <Toolbar></Toolbar>
    <Content ref="content" class="content-container"></Content>
  </div>
</template>

<script>
import Toolbar from './Toolbar'
import Content from './Content'

export default {
  name: 'video-compare',
  components: {
    Toolbar,
    Content
  }
}
</script>
<style lang="scss" scoped>
.video-center {
  height: 100%;
  .content-container {
    background: #f4f4f5;
  }
}
</style>
