<template>
  <section id="viewer">
    <!-- viewer -->
    <div class="viewer__model" v-show="value" @touchmove.prevent>
      <div class="viewer-container">
        <div class="viewer-wrapper" ref="medioList">
          <div class="viewer-slider" v-for="(item,index) in medioData" :key="index" @dblclick.native="alert('000')">
            <img :src="item" alt="slider-img">
          </div>
        </div>
      </div>
      <div class="viewer-pagination" v-if="medioData.length > 1">
        <span class="viewer-pagination-bullets" v-for="(item,index) in medioData" :key="index" :class="{'viewer-pagination-bullet-active': indexes === index}"></span>
      </div>
    </div>
  </section>
</template>

<script>
// import { setTimeout, clearTimeout } from 'timers'
export default {
  props: {
    value: {
      type: Boolean,
      required: true
    },
    initialIndex: {
      type: Number,
      default: 0
    },
    medioData: {
      type: Array,
      default: () => []
    }
  },
  data () {
    return {
      // 根节点 用于控制背景颜色
      rootNode: null,
      // 当前设备的可视宽高
      clientWidth: 0,
      clientHeight: 0,
      // 位移
      displacement: 0,
      temporaryDisplacement: 0,
      // 主要的控制条件
      iscloseingDown: false, // 是否在下滑关闭的过程中，此时不允许放大图片和换页操作
      isSingleTouch: true, // 是否是单个手指操作
      pageTurning: false, // 是否正在切换页面
      isNormalSized: true, // 图片是否被放大  如果被放大了，单指滑动则不能直接切换页面，反之可以，只在双指缩放功能中可修改
      Timer: 0,
      // 记录滑动的起始和移动中的位置的clientX
      StartCoordinates: null, // 滑动开始的时候的坐标
      StartCoordinatesX: 0, // 滑动开始的时候的clientX
      StartCoordinatesTime: 0, // 滑动开始的时间（时间戳）
      MovingCoordinates: null, // 换页滑动中的坐标
      MovingCoordinatesX: 0, // 换页滑动中的clientX

      // 单击事件计时器
      clickTimer: null,
      currentOperateEle: {
        width: 0,
        height: 0
      },
      noClick: false,
      clickFirstPosition: null
    }
  },
  computed: {
    // 索引
    indexes () {
      return this.displacement / (this.clientWidth + 20)
    },
    // 滑动距离
    slidingLength () {
      if (this.MovingCoordinatesX !== 0) {
        return this.StartCoordinatesX - this.MovingCoordinatesX
      } else {
        return 0
      }
    },
    // 翻页时滑动的最大距离
    maxLength () {
      return (this.clientWidth + 20) * (this.medioData.length - 1)
    },
    // 是否在往第一个view的左边或者最后一个view右边滑动
    isOverMax () {
      return this.temporaryDisplacement < 0 || this.temporaryDisplacement > ((this.clientWidth + 20) * (this.medioData.length - 1))
    },
    // 是否允许左右滑动换页
    canPaging () {
      // 单个手指 && 没在下滑关闭的过程中
      let controlCondition = this.isSingleTouch && !this.iscloseingDown
      return controlCondition
    },
    // 是否允许下滑关闭MEDIAVIEWER
    canSwipeDownToClose () {
      // 单个手指 || 已经在下滑过程中
      let controlCondition = this.isSingleTouch && this.iscloseingDown
      return controlCondition
    }
  },
  watch: {
    // 初始化操作
    value (newValue, oldValue) {
      if (newValue) {
        this.$nextTick(() => {
          // 初始化轮播图
          this.setInitialViewer()
        })
      }
    },
    // 动态计算左右滑动的距离
    slidingLength (newValue, oldValue) {
      if (this.isSingleTouch && !this.iscloseingDown) { // 左右滑动翻页的条件： 单指 且 图片未放大 且 不是在下滑关闭的过程中
        this.pageTurning = true
        let temporaryDisplacement = this.computeLength(this.displacement + newValue)
        this.temporaryDisplacement = temporaryDisplacement
        this.$refs.medioList.style.transform = `translate3d(${-temporaryDisplacement}px, 0px, 0px) scale(1)`
      }
    }
  },
  mounted () {
    // 初始化操作
    this.$nextTick(() => {
      this.rootNode = document.querySelector('.viewer__model')
      // 获取当前设备屏幕宽度
      this.clientWidth = window.innerWidth || document.documentElement.clientWidth || document.body.clientWidth
      this.clientHeight = window.innerHeight || document.documentElement.clientHeight || document.body.clientHeight
      this.rootNode.style.height = this.clientHeight + 'px'
      // 初始化监听事件 View功能初始化，只有一张图片时不能切换，且不展示分页器
      this.initialEventListener()
      // 设置当前屏幕可视区域的高度
      document.documentElement.style.setProperty('--height', this.clientHeight + 'px')
      // 浏览器底部工具栏消失时，重新计算可视区域的高度
      window.onresize = () => {
        this.clientHeight = window.innerHeight || document.documentElement.clientHeight || document.body.clientHeight
        this.rootNode.style.height = this.clientHeight + 'px'
        document.documentElement.style.setProperty('--height', this.clientHeight + 'px')
      }
    })
  },
  methods: {
    // 初始化MEDIAVIEWER的偏移量
    setInitialViewer () {
      let initialDisplacement = (this.clientWidth + 20) * this.initialIndex // 设置初始的偏移量
      this.displacement = initialDisplacement
      this.$refs.medioList.style.transform = `translate3d(-${initialDisplacement}px, 0px, 0px) scale(1)`
    },
    // 初始化MEDIAVIEWER的监听事件
    initialEventListener () {
      let imgNodeList = document.querySelectorAll('.viewer-slider')
      Array.prototype.forEach.call(imgNodeList, (element) => {
        if (element.children[0].tagName === 'IMG') { // 给轮播组件中的img添加缩放事件
          this.addLitenerForSlideshow(element)
        }
      })
    },
    // MEDIAVIEWER注册监听事件
    addLitenerForSlideshow (Views) {
      Views.addEventListener('touchstart', (e) => {
        this.singleTouchAtStart(Views, e)
      }, { passive: true })

      Views.addEventListener('touchmove', (e) => {
        this.singleTouchAtMoving(Views, e)
      }, { passive: true })

      Views.addEventListener('touchend', (e) => {
        this.singleTouchAtEnd(Views, e)
      }, { passive: true })

      Views.addEventListener('click', (e) => {
        clearTimeout(this.clickTimer)
        // 双击事件
        if ((new Date()).getTime() - this.Timer < 300) {
          let { clientX: fx, clientY: fy } = this.clickFirstPosition
          let { clientX: sx, clientY: sy } = e
          let dbclick = (Math.abs(fx - sx) < 10) && (Math.abs(fy - sy) < 10)
          if (dbclick) {
            let hasMagnify = Views.style.transform !== '' && Views.style.transform.match(/(?:scale\()(\d\.?\d{0,})(?:\))/)[1] !== '1'
            if (hasMagnify) {
              Views.style.transform = `translate3d(0px, 0px, 0px) scale(1)`
              Views.style.transitionProperty = 'all'
              Views.style.transitionDuration = '350ms'
            } else {
              Views.style.transform = `translate3d(0px, 0px, 0px) scale(1)`
              Views.style.transitionProperty = 'all'
              Views.style.transitionDuration = '350ms'
              this.enlargePicHandle(Views)
            }
          }
        } else { // 单击事件
          this.Timer = (new Date()).getTime()
          this.clickFirstPosition = e
          this.clickTimer = setTimeout(() => {
            this.closeViewer(Views)
          }, 300)
        }
      }, { passive: false })
    },
    // 执行函数
    singleTouchAtStart (Views, e) {
      // 单指操作
      this.isSingleTouch = true

      // 清除动画延时
      Views.style.transitionDuration = '0ms'

      // 存储当前操作对象信息
      this.currentOperateEle.width = Views.offsetWidth
      this.currentOperateEle.height = Views.offsetHeight

      // 记录切换的开始位置
      this.StartCoordinates = e.touches[0]
      this.StartCoordinatesX = e.touches[0].clientX

      if (this.canPaging) {
        // 正在切换页面
        this.pageTurning = true
        this.StartCoordinatesTime = (new Date()).getTime()
      }
    },
    singleTouchAtMoving (Views, e) {
      // 单指操作
      this.MovingCoordinates = e.touches[0]
      this.noClick = true
      // 计算切换过程中的方向
      let direction = this.getDirection(this.StartCoordinates, e.touches[0])

      // 是否允许翻页,左右滑动可以翻页
      let canTurnPage = this.canPaging && (direction === 'left' || direction === 'right')
      if (canTurnPage) {
        this.MovingCoordinatesX = e.touches[0].clientX
      }
      // 能否下滑关闭Viewer，向下滑动则关闭Viewer
      let canDownToClose = this.canSwipeDownToClose || direction === 'down'
      if (canDownToClose) {
        this.downSlideToClose(Views)
        this.iscloseingDown = true
      }
    },
    singleTouchAtEnd (Views, e) {
      // 当触发的是click事件时
      if (!this.noClick) return
      this.noClick = false
      // 滑动时间是否小于300ms
      let operationTimedOut = ((new Date()).getTime() - this.StartCoordinatesTime) < 300
      if (this.canPaging) {
        // 设置缓动效果，并及时清除
        this.$refs.medioList.style.transitionDuration = '300ms'
        setTimeout(() => {
          this.$refs.medioList.style.transitionDuration = '0ms'
        }, 300)

        // 关闭滑动功能，防止页面抖动
        this.MovingCoordinatesX = 0
        this.StartCoordinatesX = 0

        // 如果在往第一个view的左边或者最后一个view右边滑动，则在滑动结束后直接回弹，否则判断滑动的距离大于屏幕的一半 或 滑动的时间小于300ms，是则换页，否则回弹至原始位置
        if (this.isOverMax) {
          this.$refs.medioList.style.transform = `translate3d(-${this.displacement}px, 0px, 0px) scale(1)`
        } else {
          // 滑动的距离
          let slideLength = this.displacement - this.temporaryDisplacement
          // 如果单指滑动的距离大于屏幕的一半 或 滑动的时间小于300ms，则换页
          if (Math.abs(slideLength) > (this.clientWidth / 2) || operationTimedOut) {
            // 切换到下一节点
            let direction = slideLength > 0 ? 1 : -1 // 滑动方向
            let endLength = this.displacement - ((this.clientWidth + 20) * direction) // 下一节点停留的位置
            this.displacement = endLength // 更新偏移信息
            this.$refs.medioList.style.transform = `translate3d(-${endLength}px, 0px, 0px)` // 切换到下一节点
          } else {
            this.$refs.medioList.style.transform = `translate3d(-${this.displacement}px, 0px, 0px) scale(1)`
          }
        }
      }
      if (this.iscloseingDown) { // 如果图片是在下滑时关闭的过程中时
        // 如果下滑的时间超过1000ms,则会弹至初始位置，反之则关闭组件
        if (this.operationTimedOut) { // 回弹至初始位置
          Views.style.transitionDuration = '300ms'
          Views.style.transform = 'translate3d(0px,0px,0px) scale(1)'
          this.rootNode.style.backgroundColor = `rgba(0, 0, 0, 1)`
        } else {
          this.closeViewer(Views)
        }
        this.iscloseingDown = false
      }
    },
    // ----功能函数
    // 单指下滑关闭MEDIAVIEWER功能
    downSlideToClose (node) {
      // 手指横向移动的距离
      let Xmoving = this.MovingCoordinates.clientX - this.StartCoordinates.clientX
      // 手指纵向移动的距离
      let Ymoving = this.MovingCoordinates.clientY - this.StartCoordinates.clientY
      // slider的宽、高
      let { width: nodeWidth, height: nodeHeight } = this.currentOperateEle
      // 计算缩放比例
      let zoomOutRatio = 1 - (Ymoving / nodeHeight)
      // slider纵向移动的缩放量
      let Yoffset = Math.abs(this.StartCoordinates.clientY - nodeHeight / 2) * (1 - zoomOutRatio)
      // slider横向移动的所缩放量
      let Xoffset = (this.StartCoordinates.clientX - nodeWidth / 2) * (1 - zoomOutRatio)
      if (zoomOutRatio > 0.3) { // 最大缩小到0.3
        if (zoomOutRatio > 0.97) { // 去误差
          zoomOutRatio = 1
          node.style.transform = `translate3d(${Xmoving}px,${Ymoving}px,0px) scale(1)`
        } else {
          node.style.transform = `translate3d(${Xmoving + Xoffset}px,${Ymoving - Yoffset}px,0px) scale(${zoomOutRatio})`
        }
        this.rootNode.style.backgroundColor = `rgba(0, 0, 0, ${Math.round(zoomOutRatio * 100) / 100})`
      } else {
        // 重新计算缩放量
        Xoffset = Math.abs(this.MovingCoordinates.clientX - nodeWidth / 2) * 0.7
        Yoffset = Math.abs(this.MovingCoordinates.clientY - nodeHeight / 2) * 0.7
        node.style.transform = `translate3d(${Xmoving - Xoffset}px,${Ymoving - Yoffset}px,0px) scale(0.3)`
      }
    },
    // 双击放大图片功能
    enlargePicHandle (Views) {
      // 获取View说的子节点（img）的宽高，计算放大2倍后的高度和宽度,如果放大后的高度小于可视窗口的高度，则，重新计算放大倍数
      let oldImg = Views.children[0]
      let scale = 0
      if (oldImg.offsetHeight * 2 < this.clientHeight) {
        scale = this.clientHeight / oldImg.offsetHeight
      } else {
        scale = 2
      }
      let needMoveX = (this.clickFirstPosition.clientX - this.clientWidth / 2) * scale
      let Xdirection = needMoveX > 0 ? -1 : 1
      let canMoveX = this.clientWidth / 2 * (scale - 1)

      let needMoveY = (this.clickFirstPosition.clientY - this.clientHeight / 2) * scale
      let Ydirection = needMoveY > 0 ? -1 : 1
      let canMoveY = oldImg.offsetHeight / 2 * scale - this.clientHeight / 2

      let Xdisplacement = Math.abs(needMoveX) < canMoveX ? -needMoveX : Xdirection * canMoveX
      let Ydisplacement = Math.abs(needMoveY) < canMoveY ? -needMoveY : Ydirection * canMoveY
      Views.style.transform = `translate3d(${Xdisplacement}px, ${Ydisplacement}px, 0px) scale(${scale})`
    },
    // 单击关闭MEDIAVIEWER
    closeViewer (Views) {
      // 设置初始的宽度、高度，这样才能产生transition动效
      let scale = Views.style.transform === '' ? 1 : Views.style.transform.match(/(?:scale\()(\d\.?\d{0,})(?:\))/)[1]
      Views.children[0].style.width = (Views.children[0].innerWidth || Views.children[0].offsetWidth || Views.children[0].clientWidth) * scale + 'px'
      Views.children[0].style.height = (Views.children[0].innerHeight || Views.children[0].offsetHeight || Views.children[0].clientHeight) * scale + 'px'
      let moving = this.getInitialPosition(Views.children[0]) // 获取返回位置的偏移量
      this.rootNode.style.transitionDuration = '350ms'
      Views.children[0].style.transitionProperty = 'all'
      Views.children[0].style.transitionDuration = '350ms'
      Views.children[0].style.transform = moving
      Views.children[0].style.width = 'calc((100vw - 32px) * 0.3)'
      Views.children[0].style.height = '70px'
      this.rootNode.style.background = `rgba(0, 0, 0, 0)`
      setTimeout(() => {
        // 重置相关样式，关闭模态框
        this.rootNode.style.transitionDuration = '0ms'
        this.rootNode.style.background = `rgba(0, 0, 0, 1)`
        Views.children[0].style = ''
        Views.children[0].parentNode.style = ''
        this.$emit('input', false)
      }, 350)
    },
    // 辅助方法
    getDirection (start, end) { // 判断滑动方向
      let x = start.clientX - end.clientX
      let y = start.clientY - end.clientY
      let z = Math.sqrt(x * x + y * y)
      let angle = 0 // 与x轴正方向的夹角
      if (x < 0 && y > 0) {
        angle = Math.asin(y / z) * 180 / Math.PI
      } else if (x > 0 && y > 0) {
        angle = 180 - (Math.asin(y / z) * 180 / Math.PI)
      } else if (x > 0 && y < 0) {
        angle = 180 + Math.abs((Math.asin(y / z) * 180 / Math.PI))
      } else { // x < 0 && y < 0
        angle = 360 + (Math.asin(y / z) * 180 / Math.PI)
      }
      if (angle < 45 || angle === 45 || angle > 315) return 'right'
      if ((angle > 45 && angle < 135) || angle === 135) return 'up'
      if ((angle > 135 && angle < 225) || angle === 225) return 'left'
      if ((angle > 225 && angle < 315) || angle === 315) return 'down'
    },
    getInitialPosition (node) { // 计算下滑结束时，img的轨迹动画
      let endNode = document.querySelector('.media-list').children[this.indexes]
      let endBoundingClientRect = endNode.getBoundingClientRect()
      let boundingClientRect = node.getBoundingClientRect()
      let transform = node.parentNode.style.transform
      let scales = transform === '' ? 1 : transform.match(/(?:scale\()(\d\.?\d{0,})(?:\))/)[1]
      node.parentNode.style.transform = transform.replace(/(?:scale\()(\d\.?\d{0,})(?:\))/, 'scale(1)')
      let fixedDisplacementY = (node.height * (1 - scales) - (node.height - endNode.height)) / 2
      let fixedDisplacementX = (node.width * (1 - scales) - (node.width - endNode.width)) / 2
      let toTop = Math.ceil(endBoundingClientRect.top - boundingClientRect.top)
      let toLeft = Math.ceil(endBoundingClientRect.left - boundingClientRect.left)
      return `translate3d(${toLeft - 1 + fixedDisplacementX}px,${toTop + fixedDisplacementY}px,0px) scale(1)`
    },
    dampingFunction (X) { // 阻尼函数
      const A = 833.100114536909
      const B = -0.98500289542665
      const C = 1775.09591318241
      const D = -0.133922492277198
      let Y = (A - D) / (1 + Math.pow((X / C), B)) + D
      return Y
    },
    computeLength (length) {
      if (length > 0 && length < this.maxLength) {
        return length
      } else if (length > 0 && length > this.maxLength) {
        return Math.round(this.maxLength + this.dampingFunction(length - this.maxLength))
      } else if (length < 0) {
        return -Math.round(this.dampingFunction(-length))
      }
    }
  }
}
</script>
<style scoped>
/* 媒体列表 */
.media-list img {
  width: 30%;
  height: 70px;
  border-radius: 2%;
}
.media-list img:not(:nth-child(3n)) {
  margin-right: 5%;
}
/* MEDIAVIEWER */
.viewer__model {
  position: fixed;
  left: 0;
  top: 0;
  width: 100%;
  height: 100vh;
  background: rgba(0, 0, 0, 1);
  z-index: 9999;
}
.viewer-container {
  height: 100%;
  position: absolute;
  top: 50%;
  left: 0;
  transform: translate(0, -50%);
}
.viewer-wrapper {
  position: relative;
  width: 100%;
  height: 100%;
  z-index: 1;
  display: -webkit-box;
  display: -webkit-flex;
  display: -ms-flexbox;
  display: flex;
  -webkit-transition-property: -webkit-transform;
  transition-property: -webkit-transform;
  -o-transition-property: transform;
  transition-property: transform;
  transition-property: transform,-webkit-transform;
  box-sizing: content-box;
  -webkit-transform: translate3d(0,0,0) scale(1);
  transform: translate3d(0,0,0) scale(1);
}
.viewer-wrapper .viewer-slider {
  -webkit-flex-shrink: 0;
  -ms-flex-negative: 0;
  flex-shrink: 0;
  display: -webkit-box;
  display: -webkit-flex;
  display: -ms-flexbox;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100vw;
  position: relative;
  text-align: center;
  box-sizing: border-box;
}
.viewer-wrapper .viewer-slider:nth-child(n+2) {
  margin: 0 10px;
}
.viewer-wrapper .viewer-slider:first-child {
  margin-right: 10px;
}
.viewer-wrapper .viewer-slider:last-child {
  margin-right: 0px;
}
.viewer-wrapper .viewer-slider img{
  width: 100%;
  max-height: var(--height);
  position: relative;
}
.viewer-pagination {
  position: absolute;
  text-align: center;
  -webkit-transition: .3s opacity;
  -o-transition: .3s opacity;
  transition: .3s opacity;
  -webkit-transform: translate3d(0,0,0) scale(1);
  -o-transform: translate3d(0,0,0) scale(1);
  transform: translate3d(0,0,0) scale(1);
  z-index: 10;
  bottom: 2vh;
  left: 0;
  width: 100%;
}
.viewer-pagination .viewer-pagination-bullets {
  width: 8px;
  height: 8px;
  display: inline-block;
  border-radius: 100%;
  background: rgba(255, 255, 255, .3);
  margin: 0 2px;
}
.viewer-pagination .viewer-pagination-bullets.viewer-pagination-bullet-active {
  background-color: #fff;
}
</style>
