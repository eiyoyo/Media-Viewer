<template>
  <div class="index">
    <header>Medio Viewer</header>
    <div class="display-area">
      <div class="show-time">
        <div class="color-box"></div>
        <!-- 媒体列表 -->
        <div class="media-list">
          <img v-for="(item,index) in MediaData" :key="index" :src="item" alt="img" @click="showViewer(index)">
        </div>
        <div class="color-box"></div>
        <div class="color-box"></div>
        <!-- 媒体查看组件 -->
        <media-viewer
          v-model="viewerVisible"
          :medioData="MediaData"
          :initialIndex="initialIndex"
        >
        </media-viewer>
      </div>
    </div>
  </div>
</template>

<script>
// @ is an alias to /src
import Viewer from '@/components/Viewer.vue'

export default {
  name: 'index',
  components: {
    'media-viewer': Viewer
  },
  data () {
    return {
      viewerVisible: false,
      initialIndex: 0,
      MediaData: []
    }
  },
  created () {
    this.MediaData = [
      'http://images.ehaofang.com/1901242009-d526ecf3-5e94-4bc7-99df-33c5d253d7d1.jpg',
      'http://images.ehaofang.com/1901242009-8b07535b-6860-42d8-a660-a5120d97c16a.jpg',
      'http://images.ehaofang.com/1901242009-c3ecddb0-98ef-46cd-9393-c1149019105d.jpg',
      'http://images.ehaofang.com/1901242009-a9e047bb-2980-444b-a4fe-60918efc09a3.jpg',
      'http://images.ehaofang.com/1901242009-5bf69f83-1ea1-4048-bae7-b4a11b330e34.jpg',
      'http://images.ehaofang.com/1901242009-2e26597a-fd03-4bee-9601-6e914ad26591.jpg',
      'http://images.ehaofang.com/1901242009-f6a2f59f-fe58-423a-be0c-306a4e459d09.jpg',
      'http://images.ehaofang.com/1901242009-56cead4a-3442-4362-b72d-aaaa6a8cea5e.jpg',
      'http://images.ehaofang.com/1901242009-5c55fa1a-e494-4ff2-8a74-01441e796c0a.jpg',
      'http://images.ehaofang.com/1901242009-fe7ebe9d-9277-48c5-9729-b7fb6c217568.jpg',
      'http://images.ehaofang.com/1901242009-d526ecf3-5e94-4bc7-99df-33c5d253d7d1.jpg',
      'http://images.ehaofang.com/1901242009-8b07535b-6860-42d8-a660-a5120d97c16a.jpg',
      require('../assets/images/pic.jpg')
    ]
  },
  methods: {
    // open Media-Viewer
    showViewer (index) {
      this.initialIndex = index
      this.viewerVisible = true
    }
  }
}
</script>

<style scoped>
  header {
    position: fixed;
    top: 0;
    width: 100%;
    height: 46px;
    line-height: 46px;
    font-size: 18px;
    font-weight: 700;
    text-align: center;
    background-color: #fff;
    box-shadow: 0 5px 5px #f3f3f3;
  }
  /* 媒体列表 */
  .media-list {
    padding: 0 16px;
  }
  .media-list img {
    width: 30%;
    height: 70px;
    border-radius: 2%;
  }
  .media-list img:not(:nth-child(3n)) {
    margin-right: 5%;
  }
  .display-area {
    height: calc(100vh - 46px);
  }
  .color-box {
    width: 100%;
    height: 400px;
  }
</style>
