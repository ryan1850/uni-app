<template>
    <view class="image-cropper" @touchmove.stop="_preventTouchMove">
      <view class="main" @touchend="_cutTouchEnd" @touchstart="_cutTouchStart" @touchmove="_cutTouchMove" @tap="_click">
        <view class="content">
          <view :class="['content_top', 'bg_gray', !flag_bright && 'bg_black']" :style="{height: localCutTop + 'px', transitionProperty: cut_animation ? '' : 'background'}"></view>
          <view class="content_middle" :style="{height: height + 'px'}">
            <view :class="['content_middle_left', 'bg_gray', !flag_bright && 'bg_black']" :style="{width: cut_left + 'px', transitionProperty: cut_animation ? '' : 'background'}"></view>
            <view class="content_middle_middle" :style="{width: width + 'px', height: height + 'px', transitionDuration: '.3s', transitionProperty: cut_animation ? '' : 'background'}">
              <view class="border border-top-left"></view>
              <view class="border border-top-right"></view>
              <view class="border border-right-top"></view>
              <view class="border border-right-bottom"></view>
              <view class="border border-bottom-right"></view>
              <view class="border border-bottom-left"></view>
              <view class="border border-left-bottom"></view>
              <view class="border border-left-top"></view>
            </view>
            <view :class="['content_middle_right', 'bg_gray', !flag_bright && 'bg_black']" :style="{transitionProperty: cut_animation ? '' : 'background'}"></view>
          </view>
          <view :class="['content_bottom', 'bg_gray', !flag_bright && 'bg_black']" :style="{transitionProperty: cut_animation ? '' : 'background'}"></view>
        </view>
        <image 
          @load="imageLoad"
          @touchstart="_start"
          @touchmove="_move"
          @touchend="_end"
          :style="{
            width: img_width ? img_width + 'px' : 'auto',
            height: img_height ? img_height + 'px' : 'auto',
            transform: `translate3d(${img_left-img_width/2}px,${img_top-img_height/2}px,0) scale(${scale}) rotate(${angle}deg)`,
            transitionDuration: cut_animation ? '.4s' : '0s'
          }"
          class="img"
          :src="imgSrc"
        />
      </view>
      <canvas 
        :canvas-id="el"
        :disable-scroll="true"
        :style="{
          width: canvas_width * export_scale + 'px',
          height: canvas_height * export_scale + 'px',
        //   left: localCanvasLeft + 'px',
        //   top: localCutTop + 'px'
        }"
        class="image-cropper-canvas"
      />
      <button @click="getImg" >保存</button>
    </view>
  </template>
  
  <script>
  export default {
    name: 'ImageCropper',
    props: {
      imgSrc: {
        type: String,
        default: ''
      },
    //   height: {
    //     type: Number,
    //     default: 200
    //   },
    //   width: {
    //     type: Number,
    //     default: 200  
    //   },
      min_width: {
        type: Number,
        default: 100
      },
      min_height: {
        type: Number,
        default: 100
      },
      max_width: {
        type: Number,
        default: 300
      },
      max_height: {
        type: Number,
        default: 300
      },
      disable_width: {
        type: Boolean,
        default: false
      },
      disable_height: {
        type: Boolean,
        default: false
      },
      disable_ratio: {
        type: Boolean,
        default: false
      },
      export_scale: {
        type: Number,
        default: 3
      },
      quality: {
        type: Number,
        default: 1
      },
      cut_top: {
        type: Number,
        default: null
      },
    //   cut_left: {
    //     type: Number,
    //     default: null
    //   },
      canvas_top: {
        type: Number,
        default: null
      },
      canvas_left: {
        type: Number,
        default: null
      },
      img_width: {
        type: Number,
        default: null
      },
      img_height: {
        type: Number,
        default: null
      },
    //   scale: {
    //     type: Number,
    //     default: 1
    //   },
    //   angle: {
    //     type: Number,
    //     default: 0
    //   },
      min_scale: {
        type: Number,
        default: 0.5
      },
      max_scale: {
        type: Number,
        default: 2
      },
      disable_rotate: {
        type: Boolean,
        default: true
      },
      limit_move: {
        type: Boolean,
        default: false
      }
    },
    data() {
      return {
        el: 'image-cropper',
        info: uni.getSystemInfoSync(),
        MOVE_THROTTLE: null,
        MOVE_THROTTLE_FLAG: true,
        INIT_IMGWIDTH: 0,
        INIT_IMGHEIGHT: 0,
        TIME_BG: null,
        TIME_CUT_CENTER: null,
        touch_img_relative: [{
          x: 0,
          y: 0
        }],
        flag_cut_touch: false,
        hypotenuse_length: 0,
        flag_img_endtouch: false,
        flag_bright: true,
        canvas_overflow: true,
        canvas_width: 200,
        canvas_height: 200,
        origin_x: 0.5,
        origin_y: 0.5,
        cut_animation: false,
        img_top: uni.getSystemInfoSync().windowHeight / 2,
        img_left: uni.getSystemInfoSync().windowWidth / 2,
        ctx: null,
        imageObject: null,
        localCutTop: this.localCutTop,
        localCanvasTop: this.canvas_top,
        localCanvasLeft: this.canvas_left,
        width: 200,
        height: 200,
        cut_left: null,
        scale: 1,
        angle: 0,
      }
    },
    watch: {
      width(value) {
        if (value < this.min_width) {
          this.width = this.min_width
        }
        this._computeCutSize()
      },
      height(value) {
        if (value < this.min_height) {
          this.height = this.min_height
        }
        this._computeCutSize()
      },
      angle(value) {
        this._moveStop()
        if (this.limit_move) {
          if (this.angle % 90) {
            this.angle = Math.round(this.angle / 90) * 90
            return
          }
        }
      },
      cut_animation(value) {
        clearTimeout(this._cut_animation_time)
        if (value) {
          this._cut_animation_time = setTimeout(() => {
            this.cut_animation = false
          }, 300)
        }
      },
      limit_move(value) {
        if (value) {
          if (this.angle % 90) {
            this.angle = Math.round(this.angle / 90) * 90
          }
          this._imgMarginDetectionScale()
          !this.canvas_overflow && this._draw()
        }
      },
      localCutTop() {
        this._canvasDetectionPosition()
      },
      canvas_top(value) {
        this.localCanvasTop = value
      },
      localCanvasLeft() {
        this._canvasDetectionPosition()
      },
      canvas_left(value) {
        this.localCanvasLeft = value
      },
      imgSrc() {
        this.pushImg()
      },
      localCutTop() {
        this._cutDetectionPosition()
        if (this.limit_move) {
          !this.canvas_overflow && this._draw()
        }
      },
      cut_top: {
      handler(newVal) {
        this.localCutTop = newVal
      },
      immediate: true
    },
      cut_left() {
        this._cutDetectionPosition()
        if (this.limit_move) {
          !this.canvas_overflow && this._draw()
        }
      }
    },
    created() {
      this.info = uni.getSystemInfoSync()
      this.INIT_IMGWIDTH = this.img_width
      this.INIT_IMGHEIGHT = this.img_height
      this.canvas_height = this.height
      this.canvas_width = this.width
      this._initCanvas()
      this.imgSrc && (this.imgSrc = this.imgSrc)
      this._initImageSize()
      this._computeCutSize()
      this._cutDetectionPosition()
      this._canvasDetectionPosition()
      this.$emit('load', {
        cropper: this
      })
    },
    methods: {
      // 原有的所有methods方法保持不变
      upload() {
        uni.chooseImage({
          count: 1,
          sizeType: ['original', 'compressed'],
          sourceType: ['album', 'camera'],
          success: (res) => {
            const tempFilePaths = res.tempFilePaths[0]
            this.pushImg(tempFilePaths)
            uni.showLoading({
              title: '加载中...'
            })
          }
        })
      },
      // ... 其他methods保持不变
    
      getImg(getCallback) {
      this._draw(() => {
        uni.canvasToTempFilePath({
          width: this.width * this.export_scale,
          height: Math.round(this.height * this.export_scale),
          destWidth: this.width * this.export_scale,
          destHeight: Math.round(this.height) * this.export_scale,
          fileType: 'png',
          quality: this.quality,
          canvasId: this.el,
          success: (res) => {
            uni.saveImageToPhotosAlbum({
              filePath: res.tempFilePath,
            })
            // getCallback({
            //   url: res.tempFilePath,
            //   width: this.width * this.export_scale,
            //   height: this.height * this.export_scale
            // })
          }
        }, this)
      })
    },

    setTransform(transform) {
      if (!transform) return
      if (!this.disable_rotate) {
        this.angle = transform.angle ? this.angle + transform.angle : this.angle
      }
      
      let scale = this.scale
      if (transform.scale) {
        scale = this.scale + transform.scale
        scale = scale <= this.min_scale ? this.min_scale : scale
        scale = scale >= this.max_scale ? this.max_scale : scale
      }
      this.scale = scale

      if (transform.cutX) {
        this.cut_left = this.cut_left + transform.cutX
      }
      if (transform.cutY) {
        this.localCutTop = this.localCutTop + transform.cutY
      }

      this.img_top = transform.y ? this.img_top + transform.y : this.img_top
      this.img_left = transform.x ? this.img_left + transform.x : this.img_left
      
      this._imgMarginDetectionScale()
      this._moveDuring()
      
      !this.canvas_overflow && this._draw()
      this._moveStop()
    },

    setCutXY(x, y) {
      this.localCutTop = y
      this.cut_left = x
    },

    setCutSize(w, h) {
      this.width = w
      this.height = h
      this._computeCutSize()
    },

    setCutCenter() {
      const localCutTop = (this.info.windowHeight - this.height) * 0.5
      const cut_left = (this.info.windowWidth - this.width) * 0.5
      
      this.img_top = this.img_top - this.localCutTop + localCutTop
      this.localCutTop = localCutTop
      this.img_left = this.img_left - this.cut_left + cut_left
      this.cut_left = cut_left
    },

    _setCutCenter() {
      this.localCutTop = (this.info.windowHeight - this.height) * 0.5
      this.cut_left = (this.info.windowWidth - this.width) * 0.5
    },

    setWidth(width) {
      this.width = width
      this._computeCutSize()
    },

    setHeight(height) {
      this.height = height
      this._computeCutSize()
    },

    setDisableRotate(value) {
      this.disable_rotate = value
    },

    setLimitMove(value) {
      this.cut_animation = true
      this.limit_move = !!value
    },

    imgReset() {
      this.scale = 1
      this.angle = 0
      this.img_top = uni.getSystemInfoSync().windowHeight / 2
      this.img_left = uni.getSystemInfoSync().windowWidth / 2
    },

    pushImg(src) {
      if (src) {
        this.imgSrc = src
        return
      }

      if (!this.imgSrc) return

      uni.getImageInfo({
        src: this.imgSrc,
        success: (res) => {
          this.imageObject = res
          if (this.imgSrc.search(/tmp/) == -1) {
            this.imgSrc = res.path
          }
          this._imgComputeSize()
          if (this.limit_move) {
            this._imgMarginDetectionScale()
          }
          this._draw()
        },
        fail: () => {
          this.imgSrc = ''
        }
      })
    },

    imageLoad(e) {
      setTimeout(() => {
        this.$emit('imageload', this.imageObject)
      }, 1000)
    },

    setScale(scale) {
      if (!scale) return
      this.scale = scale
      !this.canvas_overflow && this._draw()
    },

    setAngle(angle) {
      if (!angle) return
      this.cut_animation = true
      this.angle = angle
      this._imgMarginDetectionScale()
      !this.canvas_overflow && this._draw()
    },

    _initCanvas() {
      if (!this.ctx) {
        this.ctx = uni.createCanvasContext(this.el, this)
      }
    },

    _initImageSize() {
      if (this.INIT_IMGWIDTH && typeof this.INIT_IMGWIDTH == "string" && this.INIT_IMGWIDTH.indexOf("%") != -1) {
        const width = this.INIT_IMGWIDTH.replace("%", "")
        this.INIT_IMGWIDTH = this.img_width = this.info.windowWidth / 100 * width
      }
      if (this.INIT_IMGHEIGHT && typeof this.INIT_IMGHEIGHT == "string" && this.INIT_IMGHEIGHT.indexOf("%") != -1) {
        const height = this.img_height.replace("%", "")
        this.INIT_IMGHEIGHT = this.img_height = this.info.windowHeight / 100 * height
      }
    },

    _cutDetectionPosition() {
      const _cutDetectionPositionTop = () => {
        if (this.localCutTop < 0) {
          this.localCutTop = 0
        }
        if (this.localCutTop > this.info.windowHeight - this.height) {
          this.localCutTop = this.info.windowHeight - this.height
        }
      }
      
      const _cutDetectionPositionLeft = () => {
        if (this.cut_left < 0) {
          this.cut_left = 0
        }
        if (this.cut_left > this.info.windowWidth - this.width) {
          this.cut_left = this.info.windowWidth - this.width
        }
      }

      if (this.localCutTop == null && this.cut_left == null) {
        this._setCutCenter()
      } else if (this.localCutTop != null && this.cut_left != null) {
        _cutDetectionPositionTop()
        _cutDetectionPositionLeft()
      } else if (this.localCutTop != null && this.cut_left == null) {
        _cutDetectionPositionTop()
        this.cut_left = (this.info.windowWidth - this.width) / 2
      } else if (this.localCutTop == null && this.cut_left != null) {
        _cutDetectionPositionLeft()
        this.localCutTop = (this.info.windowHeight - this.height) / 2
      }
    },

    _canvasDetectionPosition() {
      if (this.localCutTop == null && this.localCanvasLeft == null) {
        this.canvas_overflow = false
        this.localCutTop = -5000
        this.localCanvasLeft = -5000
      } else if (this.localCutTop != null && this.localCanvasLeft != null) {
        if (this.localCutTop < -this.height || this.localCutTop > this.info.windowHeight) {
          this.canvas_overflow = true
        } else {
          this.canvas_overflow = false
        }
      } else if (this.localCutTop != null && this.localCanvasLeft == null) {
        this.localCanvasLeft = 0
      } else if (this.localCutTop == null && this.localCanvasLeft != null) {
        this.localCutTop = 0
        if (this.localCanvasLeft < -this.width || this.localCanvasLeft > this.info.windowWidth) {
          this.canvas_overflow = true
        } else {
          this.canvas_overflow = false
        }
      }
    },

    _imgMarginDetectionPosition(scale) {
      if (!this.limit_move) return
      
      let left = this.img_left
      let top = this.img_top
      scale = scale || this.scale
      let img_width = this.img_width
      let img_height = this.img_height
      
      if (this.angle / 90 % 2) {
        img_width = this.img_height
        img_height = this.img_width
      }
      
      left = this.cut_left + img_width * scale / 2 >= left ? left : this.cut_left + img_width * scale / 2
      left = this.cut_left + this.width - img_width * scale / 2 <= left ? left : this.cut_left + this.width - img_width * scale / 2
      top = this.localCutTop + img_height * scale / 2 >= top ? top : this.localCutTop + img_height * scale / 2
      top = this.localCutTop + this.height - img_height * scale / 2 <= top ? top : this.localCutTop + this.height - img_height * scale / 2
      
      this.img_left = left
      this.img_top = top
      this.scale = scale
    },

    _imgMarginDetectionScale() {
      if (!this.limit_move) return
      
      let scale = this.scale
      let img_width = this.img_width
      let img_height = this.img_height
      
      if (this.angle / 90 % 2) {
        img_width = this.img_height
        img_height = this.img_width
      }
      
      if (img_width * scale < this.width) {
        scale = this.width / img_width
      }
      if (img_height * scale < this.height) {
        scale = Math.max(scale, this.height / img_height)
      }
      
      this._imgMarginDetectionPosition(scale)
    },

    _imgComputeSize() {
      let img_width = this.img_width
      let img_height = this.img_height
      
      if (!this.INIT_IMGHEIGHT && !this.INIT_IMGWIDTH) {
        img_width = this.imageObject.width
        img_height = this.imageObject.height
        
        if (img_width / img_height > this.width / this.height) {
          img_height = this.height
          img_width = this.imageObject.width / this.imageObject.height * img_height
        } else {
          img_width = this.width
          img_height = this.imageObject.height / this.imageObject.width * img_width
        }
      } else if (this.INIT_IMGHEIGHT && !this.INIT_IMGWIDTH) {
        img_width = this.imageObject.width / this.imageObject.height * this.INIT_IMGHEIGHT
      } else if (!this.INIT_IMGHEIGHT && this.INIT_IMGWIDTH) {
        img_height = this.imageObject.height / this.imageObject.width * this.INIT_IMGWIDTH
      }
      
      this.img_width = img_width
      this.img_height = img_height
    },

    _computeCutSize() {
      if (this.width > this.info.windowWidth) {
        this.width = this.info.windowWidth
      } else if (this.width + this.cut_left > this.info.windowWidth) {
        this.cut_left = this.info.windowWidth - this.cut_left
      }
      
      if (this.height > this.info.windowHeight) {
        this.height = this.info.windowHeight
      } else if (this.height + this.localCutTop > this.info.windowHeight) {
        this.localCutTop = this.info.windowHeight - this.localCutTop
      }
      
      !this.canvas_overflow && this._draw()
    },

    _start(event) {
        console.log(`1111`, this)
      this.flag_img_endtouch = false
      if (event.touches.length == 1) {
        console.log('2222', this.touch_img_relative)
        this.touch_img_relative[0] = {
          x: (event.touches[0].clientX - this.img_left),
          y: (event.touches[0].clientY - this.img_top)
        }
      } else {
        const width = Math.abs(event.touches[0].clientX - event.touches[1].clientX)
        const height = Math.abs(event.touches[0].clientY - event.touches[1].clientY)
        
        this.touch_img_relative = [{
          x: (event.touches[0].clientX - this.img_left),
          y: (event.touches[0].clientY - this.img_top)
        }, {
          x: (event.touches[1].clientX - this.img_left),
          y: (event.touches[1].clientY - this.img_top)
        }]
        
        this.hypotenuse_length = Math.sqrt(Math.pow(width, 2) + Math.pow(height, 2))
      }
      
      !this.canvas_overflow && this._draw()
    },

    _move_throttle() {
      if (this.info.platform == 'android') {
        clearTimeout(this.MOVE_THROTTLE)
        this.MOVE_THROTTLE = setTimeout(() => {
          this.MOVE_THROTTLE_FLAG = true
        }, 1000 / 40)
        return this.MOVE_THROTTLE_FLAG
      } else {
        this.MOVE_THROTTLE_FLAG = true
      }
    },
        
    _move(event) {
    if (this.flag_img_endtouch || !this.MOVE_THROTTLE_FLAG) return
    this.MOVE_THROTTLE_FLAG = false
    this._move_throttle()
    this._moveDuring()
    console.log(this.touch_img_relative)

    if (event.touches.length == 1) {
      // 单指拖动
      let left = (event.touches[0].clientX - this.touch_img_relative[0].x)
      let top = (event.touches[0].clientY - this.touch_img_relative[0].y)
      
      // 图像边缘检测,防止截取到空白
      this.img_left = left
      this.img_top = top
      this._imgMarginDetectionPosition()
      
      this.img_left = this.img_left
      this.img_top = this.img_top
    } else {
      // 双指放大
      let width = Math.abs(event.touches[0].clientX - event.touches[1].clientX)
      let height = Math.abs(event.touches[0].clientY - event.touches[1].clientY)
      let hypotenuse = Math.sqrt(Math.pow(width, 2) + Math.pow(height, 2))
      let scale = this.scale * (hypotenuse / this.hypotenuse_length)
      let current_deg = 0
      
      scale = scale <= this.min_scale ? this.min_scale : scale
      scale = scale >= this.max_scale ? this.max_scale : scale
      
      this.scale = scale
      this._imgMarginDetectionScale()
      
      // 双指旋转(如果没禁用旋转)
      let touch_img_relative = [{
        x: (event.touches[0].clientX - this.img_left),
        y: (event.touches[0].clientY - this.img_top)
      }, {
        x: (event.touches[1].clientX - this.img_left),
        y: (event.touches[1].clientY - this.img_top)
      }]
      
      if (!this.disable_rotate) {
        let first_atan = 180 / Math.PI * Math.atan2(touch_img_relative[0].y, touch_img_relative[0].x)
        let first_atan_old = 180 / Math.PI * Math.atan2(this.touch_img_relative[0].y, this.touch_img_relative[0].x)
        let second_atan = 180 / Math.PI * Math.atan2(touch_img_relative[1].y, touch_img_relative[1].x)
        let second_atan_old = 180 / Math.PI * Math.atan2(this.touch_img_relative[1].y, this.touch_img_relative[1].x)
        
        let first_deg = first_atan - first_atan_old
        let second_deg = second_atan - second_atan_old
        
        if (first_deg != 0) {
          current_deg = first_deg
        } else if (second_deg != 0) {
          current_deg = second_deg
        }
      }
      
      this.touch_img_relative = touch_img_relative
      this.hypotenuse_length = Math.sqrt(Math.pow(width, 2) + Math.pow(height, 2))
      
      this.angle = this.angle + current_deg
      this.scale = this.scale
    }
    
    !this.canvas_overflow && this._draw()
  },

  _end(event) {
    this.flag_img_endtouch = true
    this._moveStop()
  },

  _click(event) {
    if (!this.imgSrc) {
      this.upload()
      return
    }
    
    this._draw(() => {
      let x = event.detail ? event.detail.x : event.touches[0].clientX
      let y = event.detail ? event.detail.y : event.touches[0].clientY
      
      if ((x >= this.cut_left && x <= (this.cut_left + this.width)) && 
          (y >= this.localCutTop && y <= (this.localCutTop + this.height))) {
        uni.canvasToTempFilePath({
          width: this.width * this.export_scale,
          height: Math.round(this.height * this.export_scale),
          destWidth: this.width * this.export_scale,
          destHeight: Math.round(this.height) * this.export_scale,
          fileType: 'png',
          quality: this.quality,
          canvasId: this.el,
          success: (res) => {
            this.$emit('tapcut', {
              url: res.tempFilePath,
              width: this.width * this.export_scale,
              height: this.height * this.export_scale
            })
          }
        }, this)
      }
    })
  },

  _draw(callback) {
    if (!this.imgSrc) return
    
    let draw = () => {
      // 图片实际大小
      let img_width = this.img_width * this.scale * this.export_scale
      let img_height = this.img_height * this.scale * this.export_scale
      
      // canvas和图片的相对距离
      let xpos = this.img_left - this.cut_left
      let ypos = this.img_top - this.localCutTop
      
      // 旋转画布
      this.ctx.translate(xpos * this.export_scale, ypos * this.export_scale)
      this.ctx.rotate(this.angle * Math.PI / 180)
      this.ctx.drawImage(this.imgSrc, -img_width / 2, -img_height / 2, img_width, img_height)
      this.ctx.draw(false, () => {
        callback && callback()
      })
    }
    
    if (this.ctx.width != this.width || this.ctx.height != this.height) {
      this.canvas_height = this.height
      this.canvas_width = this.width
      
      setTimeout(() => {
        draw()
      }, 40)
    } else {
      draw()
    }
  },
  _cutTouchMove(e) {
    if (this.flag_cut_touch && this.MOVE_THROTTLE_FLAG) {
      if (this.disable_ratio && (this.disable_width || this.disable_height)) return
      
      this.MOVE_THROTTLE_FLAG = false
      this._move_throttle()
      
      let width = this.width
      let height = this.height
      let localCutTop = this.localCutTop
      let cut_left = this.cut_left
      
      const size_correct = () => {
        width = width <= this.max_width ? width >= this.min_width ? width : this.min_width : this.max_width
        height = height <= this.max_height ? height >= this.min_height ? height : this.min_height : this.max_height
      }
      
      const size_inspect = () => {
        if ((width > this.max_width || width < this.min_width || height > this.max_height || height < this.min_height) && this.disable_ratio) {
          size_correct()
          return false
        } else {
          size_correct()
          return true
        }
      }

      height = this.CUT_START.height + ((this.CUT_START.corner > 1 && this.CUT_START.corner < 4 ? 1 : -1) * (this.CUT_START.y - e.touches[0].clientY))
      
      switch (this.CUT_START.corner) {
        case 1:
          width = this.CUT_START.width + this.CUT_START.x - e.touches[0].clientX
          if (this.disable_ratio) {
            height = width / (this.width / this.height)
          }
          if (!size_inspect()) return
          cut_left = this.CUT_START.cut_left - (width - this.CUT_START.width)
          break
          
        case 2:
          width = this.CUT_START.width + this.CUT_START.x - e.touches[0].clientX
          if (this.disable_ratio) {
            height = width / (this.width / this.height)
          }
          if (!size_inspect()) return
          localCutTop = this.CUT_START.localCutTop - (height - this.CUT_START.height)
          cut_left = this.CUT_START.cut_left - (width - this.CUT_START.width)
          break
          
        case 3:
          width = this.CUT_START.width - this.CUT_START.x + e.touches[0].clientX
          if (this.disable_ratio) {
            height = width / (this.width / this.height)
          }
          if (!size_inspect()) return
          localCutTop = this.CUT_START.localCutTop - (height - this.CUT_START.height)
          break
          
        case 4:
          width = this.CUT_START.width - this.CUT_START.x + e.touches[0].clientX
          if (this.disable_ratio) {
            height = width / (this.width / this.height)
          }
          if (!size_inspect()) return
          break
      }

      if (!this.disable_width && !this.disable_height) {
        this.width = width
        this.cut_left = cut_left
        this.height = height
        this.localCutTop = localCutTop
      } else if (!this.disable_width) {
        this.width = width
        this.cut_left = cut_left
      } else if (!this.disable_height) {
        this.height = height
        this.localCutTop = localCutTop
      }
      
      this._imgMarginDetectionScale()
    }
  },

  _cutTouchStart(e) {
    const currentX = e.touches[0].clientX
    const currentY = e.touches[0].clientY
    
    const cutbox_top4 = this.localCutTop + this.height - 30
    const cutbox_bottom4 = this.localCutTop + this.height + 20
    const cutbox_left4 = this.cut_left + this.width - 30
    const cutbox_right4 = this.cut_left + this.width + 30

    const cutbox_top3 = this.localCutTop - 30
    const cutbox_bottom3 = this.localCutTop + 30
    const cutbox_left3 = this.cut_left + this.width - 30
    const cutbox_right3 = this.cut_left + this.width + 30

    const cutbox_top2 = this.localCutTop - 30
    const cutbox_bottom2 = this.localCutTop + 30
    const cutbox_left2 = this.cut_left - 30
    const cutbox_right2 = this.cut_left + 30

    const cutbox_top1 = this.localCutTop + this.height - 30
    const cutbox_bottom1 = this.localCutTop + this.height + 30
    const cutbox_left1 = this.cut_left - 30
    const cutbox_right1 = this.cut_left + 30

    if (currentX > cutbox_left4 && currentX < cutbox_right4 && currentY > cutbox_top4 && currentY < cutbox_bottom4) {
      this._moveDuring()
      this.flag_cut_touch = true
      this.flag_img_endtouch = true
      this.CUT_START = {
        width: this.width,
        height: this.height,
        x: currentX,
        y: currentY,
        corner: 4
      }
    } else if (currentX > cutbox_left3 && currentX < cutbox_right3 && currentY > cutbox_top3 && currentY < cutbox_bottom3) {
      this._moveDuring()
      this.flag_cut_touch = true
      this.flag_img_endtouch = true
      this.CUT_START = {
        width: this.width,
        height: this.height,
        x: currentX,
        y: currentY,
        localCutTop: this.localCutTop,
        cut_left: this.cut_left,
        corner: 3
      }
    } else if (currentX > cutbox_left2 && currentX < cutbox_right2 && currentY > cutbox_top2 && currentY < cutbox_bottom2) {
      this._moveDuring()
      this.flag_cut_touch = true
      this.flag_img_endtouch = true
      this.CUT_START = {
        width: this.width,
        height: this.height,
        localCutTop: this.localCutTop,
        cut_left: this.cut_left,
        x: currentX,
        y: currentY,
        corner: 2
      }
    } else if (currentX > cutbox_left1 && currentX < cutbox_right1 && currentY > cutbox_top1 && currentY < cutbox_bottom1) {
      this._moveDuring()
      this.flag_cut_touch = true
      this.flag_img_endtouch = true
      this.CUT_START = {
        width: this.width,
        height: this.height,
        localCutTop: this.localCutTop,
        cut_left: this.cut_left,
        x: currentX,
        y: currentY,
        corner: 1
      }
    }
  },

  _cutTouchEnd() {
    this._moveStop()
    this.flag_cut_touch = false
  },

  _moveStop() {
    clearTimeout(this.TIME_CUT_CENTER)
    this.TIME_CUT_CENTER = setTimeout(() => {
      if (!this.cut_animation) {
        this.cut_animation = true
      }
      this.setCutCenter()
    }, 1000)

    clearTimeout(this.TIME_BG)
    this.TIME_BG = setTimeout(() => {
      if (this.flag_bright) {
        this.flag_bright = false
      }
    }, 2000)
  },

  _moveDuring() {
    clearTimeout(this.TIME_CUT_CENTER)
    clearTimeout(this.TIME_BG)
    
    if (!this.flag_bright) {
      this.flag_bright = true
    }
  },
  _preventTouchMove() {
    // 阻止页面滚动
  },
    
    }
  }
  </script>
  
  <style>
  .image-cropper {
    background: rgba(14, 13, 13, .8);
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    z-index: 1;
  }
  
  .image-cropper .main {
    position: absolute;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
  }
  
  .image-cropper .content {
    z-index: 9;
    position: absolute;
    width: 100vw;
    height: 100vh;
    display: flex;
    flex-direction: column;
    pointer-events: none;
  }
  
  .image-cropper .bg_black {
    background: rgba(0, 0, 0, 0.8) !important;
  }
  
  .image-cropper .bg_gray {
    background: rgba(0, 0, 0, 0.45);
    transition-duration: .35s;
  }
  
  .image-cropper .content>.content_top {
    pointer-events: none;
  }
  
  .image-cropper .content>.content_middle {
    display: flex;
    height: 200px;
    width: 100%;
  }
  
  .image-cropper .content_middle_middle {
    width: 200px;
    box-sizing: border-box;
    position: relative;
    transition-duration: .3s;
  }
  
  .image-cropper .content_middle_right {
    flex: auto;
  }
  
  .image-cropper .content>.content_bottom {
    flex: auto;
  }
  
  .image-cropper .img {
    z-index: 2;
    top: 0;
    left: 0;
    position: absolute;
    border: none;
    width: 100%;
    backface-visibility: hidden;
    transform-origin: center;
  }
  
  .image-cropper .image-cropper-canvas {
    position: fixed;
    background: white;
    width: 150px;
    height: 150px;
    z-index: 10;
    top: -200%;
    pointer-events: none;
    visibility: hidden;
  }
  
  .image-cropper .border {
    background: white;
    pointer-events: auto;
    position: absolute;
  }
  
  .image-cropper .border-top-left {
    left: -2.5px;
    top: -2.5px;
    height: 2.5px;
    width: 33rpx;
  }
  
  .image-cropper .border-top-right {
    right: -2.5px;
    top: -2.5px;
    height: 2.5px;
    width: 33rpx;
  }
  
  .image-cropper .border-right-top {
    top: -1px;
    width: 2.5px;
    height: 30rpx;
    right: -2.5px;
  }
  
  .image-cropper .border-right-bottom {
    width: 2.5px;
    height: 30rpx;
    right: -2.5px;
    bottom: -1px;
  }
  
  .image-cropper .border-bottom-left {
    height: 2.5px;
    width: 33rpx;
    bottom: -2.5px;
    left: -2.5px;
  }
  
  .image-cropper .border-bottom-right {
    height: 2.5px;
    width: 33rpx;
    bottom: -2.5px;
    right: -2.5px;
  }
  
  .image-cropper .border-left-top {
    top: -1px;
    width: 2.5px;
    height: 30rpx;
    left: -2.5px;
  }
  
  .image-cropper .border-left-bottom {
    width: 2.5px;
    height: 30rpx;
    left: -2.5px;
    bottom: -1px;
  }
  </style>