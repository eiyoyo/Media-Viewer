<template>
  <section id="viewer">
    <div style=" position: fixed; z-index: 9999; color: #fff;">{{debugInfo}}</div>
    <!-- viewer -->
    <div class="viewer__model" v-show="value" @click="closeViewer" @touchmove.prevent>
      <div class="viewer-container">
        <div class="viewer-wrapper" ref="medioList">
          <div class="viewer-slider" v-for="(item,index) in medioData" :key="index">
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
import { setInterval, clearInterval } from 'timers'
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
      debugInfo: 0,
      // 根节点 用于控制背景颜色
      rootNode: null,
      // 当前View组件的DOM
      swiper: null,
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
      isOver: { // 当图片被放大时，其是否超出了屏幕的上下或左右边框，超出了才可以上下或左右移动
        vertical: false,
        level: false,
        verticalValue: 0,
        levelValue: 0
      },
      // 下滑关闭操作的时间，当下滑操作的时间在1000ms内完成，则关闭组件，如果超过1000ms，则回弹至初始位置
      operationTimedOut: false,
      Timer: null,
      // 记录滑动的起始和移动中的位置的clientX
      StartCoordinates: null, // 滑动开始的时候的坐标
      StartCoordinatesX: 0, // 滑动开始的时候的clientX
      StartCoordinatesTime: 0, // 滑动开始的时间（时间戳）
      MovingCoordinates: null, // 换页滑动中的坐标
      MovingCoordinatesX: 0, // 换页滑动中的clientX
      // 在touchend事件中的定时器
      timeOut: null,
      // 单个手指操作时手指相关信息
      singleFingerPosition: {
        start: {},
        end: {},
        moveing: {},
        displacement: {
          x: 0,
          y: 0
        }
      },
      // 两个手指操作时手指相关信息
      doubleFingerPosition: {
        one: null,
        two: null
      },
      // 浏览器底部高度
      browserToolbarHeight: 0,
      // 单击事件计时器
      clickTimer: null,
      // 滑动事件计时器
      sliderTimer: null,
      slideTime: 0,
      currentScale: 0,
      endScale: 0,
      temporaryMoveData: {
        lastScale: 0
      }
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
    // 是否在往第一个view的左边或者最后一个view右边滑动
    isOverMax () {
      return this.temporaryDisplacement < 0 || this.temporaryDisplacement > ((this.clientWidth + 20) * (this.medioData.length - 1))
    },
    // 是否允许左右滑动换页
    canPaging () {
      // 单个手指  &&  图片未放大  && 没在下滑关闭的过程中 && 未在缩放的过程中
      let controlCondition = this.isSingleTouch && !this.iscloseingDown
      return controlCondition
    },
    // 是否允许下滑关闭MEDIAVIEWER
    canSwipeDownToClose () {
      // 单个手指  && 已经在下滑过程中
      let controlCondition = this.isSingleTouch || this.iscloseingDown
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
    // 单击关闭MEDIAVIEWER
    closeViewer () {
      this.$emit('input', false)
    },
    // 初始化MEDIAVIEWER
    setInitialViewer () {
      let initialDisplacement = (this.clientWidth + 20) * this.initialIndex // 设置初始的偏移量
      this.displacement = initialDisplacement
      this.$refs.medioList.style.transform = `translate3d(-${initialDisplacement}px, 0px, 0px) scale(1)`
    },
    // 切换view
    addLitenerForSlideshow (Views) {
      Views.addEventListener('touchstart', (e) => {
        if (this.canPaging) {
          // 正在切换页面
          this.pageTurning = true
          // 记录切换的开始位置
          this.StartCoordinates = e.touches[0]
          this.StartCoordinatesX = e.touches[0].clientX
          this.StartCoordinatesTime = (new Date()).getTime()
        }
      }, { passive: true })

      Views.addEventListener('touchmove', (e) => {
        this.MovingCoordinates = e.touches[0]
        // 计算切换过程中的方向
        let direction = this.getDirection(this.StartCoordinates, e.touches[0])

        // 是否允许翻页
        let canTurnPage = this.canPaging && (direction === 'left' || direction === 'right')
        if (canTurnPage) {
          this.MovingCoordinatesX = e.touches[0].clientX
        }

        // 能否下滑关闭Viewer
        let canDownToClose = this.canSwipeDownToClose && direction === 'down'
        if (canDownToClose) {
          this.iscloseingDown = true
          this.downSlideToClose(Views)
        }
      }, { passive: true })

      Views.addEventListener('touchend', (e) => {
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
          // 滑动时间是否小于300ms
            let operationTimedOut = ((new Date()).getTime() - this.StartCoordinatesTime) < 300
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
      }, { passive: true })
    },
    // 计算滑动的距离
    computeLength (length) {
      let maxLength = (this.clientWidth + 20) * (this.medioData.length - 1)
      if (length > 0 && length < maxLength) {
        return length
      } else if (length > 0 && length > maxLength) {
        return Math.round(maxLength + this.dampingFunction(length - maxLength))
      } else if (length < 0) {
        return -Math.round(this.dampingFunction(-length))
      }
    },
    // 添加双指缩放事件
    initialEventListener () {
      let imgNodeList = document.querySelectorAll('.viewer-slider')
      Array.prototype.forEach.call(imgNodeList, (element) => {
        if (element.children[0].tagName === 'IMG') { // 给轮播组件中的img添加缩放事件
          this.addLitenerForSlideshow(element)
        }
      })
    },
    // 设置图片的双指缩放功能，以及放大后的单指滑动查看功能
    setImageMagnification (dom) {
      dom.addEventListener('touchstart', (e) => {
        if (e.touches.length >= 2) { // 双指操作
          this.isNormalSized = false
          this.isSingleTouch = false
          this.doubleFingerPosition.one = e.touches[0] // 得到第一个手指坐标
          this.doubleFingerPosition.two = e.touches[1] // 得到第二个手指坐标
        } else { // 单指操作
          // 记录滑动开始时的位置
          this.singleFingerPosition.start = e.touches[0]
          // 关闭动画延迟
          dom.style.transitionDuration = '0ms'
          // 开启滑动记时
          this.sliderTimer = setInterval(() => {
            this.slideTime += 50
          }, 50)
        };
      }, false)
      dom.addEventListener('touchmove', (e) => {
        if (e.touches.length >= 2) { // 双指操作
          // 得到第二组两个点
          let now = e.touches
          // 得到缩放比例，getDistance是勾股定理的一个方法
          let scale = this.getDistance(now[0], now[1]) / this.getDistance(this.doubleFingerPosition.one, this.doubleFingerPosition.two)
          // 缩放比小数处理
          scale = Math.round(scale * 100) / 100
          // 滑动的过程中缩小
          if (this.temporaryMoveData.lastScale > scale) {
            // 滑动 缩小
            let shrunkenScale = this.temporaryMoveData.lastScale - scale
            shrunkenScale = Math.round(shrunkenScale * 100) / 100 // 控制别让他缩放的太快了
            this.endScale = this.endScale - shrunkenScale
            dom.style.transform = `translate3d(0px,0px,0px) scale(${this.endScale})`
          } else {
          // 滑动的过程中放大
            if (this.currentScale !== 0) {
              this.endScale = Math.round((this.currentScale + (scale - 1)) * 100) / 100
              dom.style.transform = `translate3d(0px,0px,0px) scale(${this.endScale})`
            } else {
              this.endScale = scale
              dom.style.transform = `translate3d(0px,0px,0px) scale(${this.endScale})`
            }
          }
          this.temporaryMoveData.lastScale = scale
        } else {
          // 记录本次滑动的位置
          this.singleFingerPosition.end = e.touches[0]
          // 获取滑动方向
          let direction = this.getDirection(this.singleFingerPosition.start, this.singleFingerPosition.end)

          if (this.isMagnifying(dom) && !this.iscloseingDown) { // 当图片放大时提供滑动功能
            this.zoomInSlide(dom.children[0])
          } else if (!this.pageTurning && this.isSingleTouch) { // 当图片未放大时提供下滑动关闭
            if (direction === 'down' || this.iscloseingDown) {
              this.downSlideToClose(dom)
              this.iscloseingDown = true
            }
          }
        }
      }, false)
      dom.addEventListener('touchend', (e) => {
        if (!this.isSingleTouch && this.timeOut === null) { // 双指操作时
          // 开启轮播功能
          this.timeOut = setTimeout(() => {
            this.timeOut = null
            this.isSingleTouch = true
            dom.style.transitionDuration = '0ms'
          }, 300)
          dom.transitionDuration = '300ms'
          if (this.endScale > 3) {
            // 当缩放比大于3时，设置回弹效果，回弹为scale为3
            this.endScale = 3
            dom.style.transform = 'translate3d(0px,0px,0px) scale(3)'
          } else if (this.endScale < 1) {
            // 当缩放比小于1时，设置回弹效果，回弹为scale为1
            this.endScale = 1
            this.isNormalSized = true
            dom.style.transform = 'translate3d(0px,0px,0px) scale(1)'
          }
          if (this.isMagnifying(dom)) { // 如果图片被放大了，则提供回弹至边界的功能
            this.springbackToBorder(dom)
          }
          // 记录双指缩放操作结束时的scale
          this.currentScale = this.endScale
          // 清除滑动参数的临时数据
          this.temporaryMoveData.lastScale = 0
          // 判断放大后的图片是否超出上下左右的边界及其距离
          this.isOverTheEdge(dom.children[0])
        } else { // 单指操作时
          if (!this.iscloseingDown) { // 如果图片不是在下滑时关闭的过程中时
            let lastX = dom.style.transform.match(/\(([^)]*)\)/)[1].match(/[^,*px]+/g)[0]
            let lastY = dom.style.transform.match(/\(([^)]*)\)/)[1].match(/[^,*px]+/g)[1]
            // 如果是快速滑动，则添加一个缓慢减速的过程
            clearInterval(this.sliderTimer)
            this.sliderTimer = null
            // if (this.slideTime < 300) {
            //   dom.style.transitionDuration = '300ms'
            //   let x = this.singleFingerPosition.start.pageX - this.singleFingerPosition.end.pageX
            //   let y = this.singleFingerPosition.start.pageY - this.singleFingerPosition.end.pageY
            //   let X = dom.style.transform.match(/(?:translate3d\()(-?\d+\.?\d{0,})(?:px)/)[1]
            //   let Y = dom.style.transform.match(/(?:\s)(-?\d+\.?\d{0,})(?:px,\s)/)[1]
            //   let scale = dom.style.transform.match(/(?:scale\()(\d\.?\d{0,})(?:\))/)[1]
            //   dom.style.transform = `translate3d(${+X - x}px,${+Y - y}px,0px) scale(${scale})`
            //   setTimeout(() => {
            //     this.springbackToBorder(dom)
            //   }, 300)
            // } else {
            this.springbackToBorder(dom)
            // }
            // // 如果图片被放大了，则提供回弹至边界的功能
            // if (this.isMagnifying(dom)) {

            // }
            this.slideTime = 0
            this.singleFingerPosition.displacement.x = +lastX
            this.singleFingerPosition.displacement.y = +lastY
          } else if (this.iscloseingDown) { // 如果图片是在下滑时关闭的过程中时
            // 如果下滑的时间超过1000ms,则会弹至初始位置，反之则关闭组件
            if (this.operationTimedOut) { // 回弹至初始位置
              dom.style.transitionDuration = '300ms'
              dom.style.transform = 'translate3d(0px,0px,0px) scale(1)'
              this.rootNode.style.backgroundColor = `rgba(0, 0, 0, 1)`
            } else {
              let moving = this.getInitialPosition(dom.children[0]) // 获取返回位置的偏移量
              this.rootNode.style.transitionDuration = '500ms'
              dom.children[0].style.transitionProperty = 'all'
              dom.children[0].style.transitionDuration = '500ms'
              dom.children[0].style.width = 'calc((100vw - 32px) * 0.3)'
              dom.children[0].style.height = '70px'
              dom.children[0].style.transform = moving
              dom.children[0].style.opacity = '0.3'
              this.rootNode.style.background = `rgba(0, 0, 0, 0)`
              setTimeout(() => {
                this.viewerVisible = false
                // 重置一些变量
                this.rootNode.style.background = `rgba(0, 0, 0, 1)`
                dom.children[0].style = ''
                dom.children[0].parentNode.style = ''
              }, 500)
            }
            this.iscloseingDown = false
            // 清除定时器
            clearTimeout(this.Timer)
            this.Timer = null
          }
        }
      }, false)
    },
    // 当图片放大时提供滑动功能
    zoomInSlide (node) {
      // 获取上次滑动的坐标偏移值
      let { x, y } = { ...this.singleFingerPosition.displacement }
      let Xmoving = x
      let Ymoving = y
      if (this.isOver.vertical) { // 当放大后，只有图片上下部分被遮挡时才产生上下滑动
        Ymoving = y + this.singleFingerPosition.end.pageY - this.singleFingerPosition.start.pageY
        if (Math.abs(Ymoving) > this.isOver.verticalValue) {
          // 滑动到边界后产生阻尼曲线
          let dir = Ymoving / Math.abs(Ymoving)
          Ymoving = (this.isOver.verticalValue + this.dampingFunction(Math.abs(Ymoving) - this.isOver.verticalValue)) * dir
        }
      }
      if (this.isOver.level) { // 当放大后，只有图片左右部分被遮挡时才产生上下滑动
        Xmoving = x + this.singleFingerPosition.end.pageX - this.singleFingerPosition.start.pageX
        if (Math.abs(Xmoving) > this.isOver.levelValue) {
          // 滑动到边界后产生阻尼曲线
          let dir = Xmoving / Math.abs(Xmoving)
          Xmoving = (this.isOver.levelValue + this.dampingFunction(Math.abs(Xmoving) - this.isOver.levelValue)) * dir
        }
      }
      let scale = node.parentNode.style.transform.match(/(?:scale\()(\d\.?\d{0,})(?:\))/)[0]
      node.parentNode.style.transform = `translate3d(${Xmoving}px, ${Ymoving}px, 0px) ${scale}`
    },
    // 回弹至边界的功能
    springbackToBorder (node) {
      let boundingClientRect = node.children[0].getBoundingClientRect() // 获取node下的img距离边界的距离
      let bottom = this.clientHeight - boundingClientRect.bottom
      let top = boundingClientRect.top
      let left = boundingClientRect.left
      let right = this.clientWidth - boundingClientRect.right
      let lastX = 0
      let lastY = 0

      // 获取回弹前坐标
      if (node.style.transform !== '') {
        lastX = node.style.transform.match(/(?:translate3d\()(-?\d+\.?\d{0,})(?:px)/)[1]
        lastY = node.style.transform.match(/(?:\s)(-?\d+\.?\d{0,})(?:px,\s)/)[1]
      }
      let leftDistance = left < 0 ? 0 : left
      let rightDistance = right < 0 ? 0 : right
      let topDistance = top < 0 ? 0 : top
      let bottomDistance = bottom < 0 ? 0 : bottom
      // 开启回弹动画
      node.style.transitionDuration = '350ms'
      // 当图片被放大时，其是否超出了屏幕的上下边框，超出了才可以上下移动,此处是为了清除其细微的偏移量
      if (!this.isOver.vertical) {
        bottomDistance = (bottom - top) / 2
        topDistance = 0
      } else if (!this.isOver.level) {
        leftDistance = (left - right) / 2
        rightDistance = 0
      }
      let scale = node.style.transform.match(/(?:scale\()(\d\.?\d{0,})(?:\))/)[1]
      node.style.transform = `translate3d(${+lastX + rightDistance - leftDistance}px,${+lastY + bottomDistance - topDistance}px,0px) scale(${Math.round(scale * 100) / 100})`
    },
    // 判断图片放大后是否上、下、左、右超过边界
    isOverTheEdge (node) {
      let boundingClientRect = node.getBoundingClientRect()
      let bottom = this.clientHeight - boundingClientRect.bottom
      let top = boundingClientRect.top
      let left = boundingClientRect.left
      let right = this.clientWidth - boundingClientRect.right
      if (bottom > 0 && top > 0) { // 当放大后，图片上下部分未被遮挡时，不产生上下滑动
        this.isOver.vertical = false
        this.isOver.verticalValue = 0
      } else {
        this.isOver.vertical = true
        this.isOver.verticalValue = Math.abs(bottom)
      }
      if (left > 0 && right > 0) { // 当放大后，图片上下部分未被遮挡时，不产生上下滑动
        this.isOver.level = false
        this.isOver.levelValue = 0
      } else {
        this.isOver.level = true
        this.isOver.levelValue = Math.abs(left)
      }
    },
    // 单指下滑关闭MEDIAVIEWER功能
    downSlideToClose (node) {
      // 手指横向移动的距离
      let Xmoving = this.MovingCoordinates.pageX - this.StartCoordinates.pageX
      // 手指纵向移动的距离
      let Ymoving = this.MovingCoordinates.pageY - this.StartCoordinates.pageY
      // slider的宽、高
      let nodeHeight = node.offsetHeight
      let nodeWidth = node.offsetWidth
      // 清除动画延时
      node.style.transitionDuration = `0ms`
      // 计算缩放比例
      let zoomOutRatio = 1 - (Ymoving / nodeHeight)
      // slider纵向移动的缩放量
      let Yoffset = Math.abs(this.MovingCoordinates.pageY - nodeHeight / 2) * (1 - zoomOutRatio)
      // slider横向移动的所缩放量
      let Xoffset = Math.abs(this.MovingCoordinates.pageX - nodeWidth / 2) * (1 - zoomOutRatio)
      console.log('runing')
      if (zoomOutRatio > 0.3) { // 最大缩小到0.3
        if (zoomOutRatio > 0.97) { // 去误差
          zoomOutRatio = 1
          node.style.transform = `translate3d(${Xmoving}px,${Ymoving}px,0px) scale(1)`
        } else {
          node.style.transform = `translate3d(${Xmoving - Xoffset}px,${Ymoving - Yoffset}px,0px) scale(${zoomOutRatio})`
        }
        this.rootNode.style.backgroundColor = `rgba(0, 0, 0, ${Math.round(zoomOutRatio * 100) / 100})`
      } else {
        // 重新计算缩放量
        Xoffset = Math.abs(this.MovingCoordinates.pageX - nodeWidth / 2) * 0.7
        Yoffset = Math.abs(this.MovingCoordinates.pageY - nodeHeight / 2) * 0.7
        node.style.transform = `translate3d(${Xmoving - Xoffset}px,${Ymoving - Yoffset}px,0px) scale(0.3)`
      }
    },
    // 辅助方法
    getDirection (start, end) { // 判断滑动方向
      let x = start.pageX - end.pageX
      let y = start.pageY - end.pageY
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
    getDistance (one, two) { // 获取移动的距离
      let x = one.pageX - two.pageX
      let y = one.pageY - two.pageY
      return Math.sqrt((x * x) + (y * y))
    },
    getCenter (one, two) { // 获取移动的中心
      let x = (one.pageX + two.pageX) / 2
      let y = (one.pageY - two.pageY) / 2
      let center = x + 'px ' + y + 'px'
      return center
    },
    getPercentile (value) { // 截取两位小数
      return Math.round(value * 100) / 100
    },
    isMagnifying (node) { // 图片是否被放大
      if (node.style.transform === '' ? true : node.style.transform.match(/scale\(([^)]*)\)/)[0] === 'scale(1)') {
        return false
      } else {
        return true
      }
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
  -webkit-transform: translate3d(0,0,0) scale(1);
  transform: translate3d(0,0,0) scale(1);
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
  max-width: 100%;
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
