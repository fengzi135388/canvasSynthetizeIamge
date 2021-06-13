<template>
  <div ref="pageBoxRef" style="overflow:hidden" @click="hidePictureOpt">
    <!-- <div class="canvas_box"> -->
      <!-- <canvas id="canvas" ref="canvasImg" :width="pageWidth" height="350"></canvas> -->
      <!-- 手机壳模板图片 -->
      <!-- <img ref="phoneShellTemplate" class="phone_shell_template" :width="imgWidth" :height="imgHeight" :src="phoneImg"/> -->
      <!-- 缓存选中制作图片 -->
      <!-- <img ref="selectImg" class="select_img" :width="selectImgWidth" :src="selectImgs"/> -->
    <!-- </div> -->
    <div class="phone_box">
      <img ref="phoneShellTemplate" @click="phoneImgClick" class="phone_shell_template"  :width="imgWidth" :height="imgHeight" :src="phoneImg"/>
      <!-- 覆盖图片 -->
      <div :class="coverBoxClass" :style="showPictureOpt?'':'overflow:hidden'">
        <div class="img_box" v-show="showImgBox" :style="imgBoxStyle" >
          <img
            @load="setCurImgWH"
            :style="showPictureOpt?'opacity:0.7':''"
            v-finger:pinch="pinchHandler"
            v-finger:rotate="rotateHandler"
            ref="curImg"
            class="cur_img"
            @touchstart="getStartPoint"
            @touchmove="imgBoxTouchMove"
            :width="curImgWidth"
            :height="curImgHeight"
            :src="selectImgs"
            />
          <img class="del_icon opt_icon" v-show="showPictureOpt" @click="delImg" src="@/assets/images/del_icon.png" alt="">
          <img class="rotate_icon opt_icon" v-show="showPictureOpt" src="@/assets/images/rotate_icon.png" alt="">
          <img class="scale_icon opt_icon" v-show="showPictureOpt" src="@/assets/images/scale_icon.png" alt="">
        </div>
      </div>
      <!-- 覆盖文本 -->
      <div class="cover_text" v-if="showCoverText">
        <div class="text_con_box" :style="coverTextStyle">{{showTextBox?tempTextCon:textCon}}</div>
      </div>
      <div class="add_text_btn" v-if="showCoverText&&step!==3&&!textCon&&!tempTextCon">
        <van-button size="small" type="info"  @click="showTextEdit">添加文字</van-button>
      </div>
      <div class="text_icon_box"
        :style="textIconStyle"
        v-finger:pinch="pinchTextHandler"
        v-finger:rotate="rotateTextHandler"
        @touchstart="getStartPoint"
        @touchmove="textBoxTouchMove"
        v-if="showCoverText&&step!==3&&(textCon||tempTextCon)"
        v-show="!showPictureOpt"
        >
          <img class="text_close_icon" @click="delText" src="@/assets/images/text_close_icon.png" alt="">
          <img class="text_edit_icon" @click="showTextEdit" src="@/assets/images/text_edit_icon.png" alt="">
          <img class="text_scale_icon"  src="@/assets/images/text_scale_icon.png" alt="">
          <img class="text_dir_icon" @click="changeArray" src="@/assets/images/text_dir_icon.png" alt="">
        </div>
    </div>
    <!-- 选择图片 start -->
    <template v-if="step==1">
      <div class="select_img_box">
        <div class="upload_button" @click="popupState = true">
          <van-icon name="photograph" size="25" />
          <span class="text">上传图片</span>
        </div>
        <div v-for="(item, ind) in 3" :key="item">
          <van-image class="img_list" width="80" height="140" src="https://www.hz2030.com/userPicures/TOB20200806-RQMY6ZKL.jpeg" @click="selectImg(item,ind)"/>
        </div>
      </div>
      <van-popup overlay-class="popup" v-model="popupState" position="bottom">
        <van-uploader capture="camera" :after-read="afterRead">
          <p>拍照</p>
        </van-uploader>
        <van-uploader :after-read="afterRead">
          <p>从手机相册选择</p>
        </van-uploader>
        <p @click="popupState = false">取消</p>
      </van-popup>
    </template>
    <!-- 选择图片 end -->
    <!-- 颜色选择器进度条 start -->
    <div class="color_font_select" v-if="step==2">
      <Slider v-model="colors" :showDetails="false" />
      <div class="font_slider">
        <span v-for="item in 10" :key="item">文字{{item}}</span>
      </div>
    </div>
    <!-- 颜色选择器进度条 end -->
    <div class="bottom-btn">
      <van-button size="large" round type="info" v-if="step==1"  @click="next">下一步</van-button>
      <van-button size="large" round type="info" v-if="step==2"  @click="next">完成</van-button>
    </div>
    <!-- 文本输入框 -->
    <div class="text-box-modal" v-if="showTextBox" @click="textInpBlur">
      <div class="text-box">
        <input class="textInp" ref="textInp" v-model.trim="tempTextCon" type="text">
        <van-button size="small" round type="primary"  @click="saveText">保存</van-button>
      </div>
    </div>
  </div>
</template>

<script>
import Vue from 'vue'
import Canvas from '@/utils/canvasEL'
import Slider from '@/components/vue-color/Slider.vue'
import { Image as VanImage, Uploader, Icon, Button, Popup } from 'vant'
Vue.use(VanImage)
Vue.use(Uploader)
Vue.use(Icon)
Vue.use(Button)
Vue.use(Popup)
export default {
  name: 'selectPhoto',
  components: {
    Slider
  },
  data () {
    return {
      // 手机壳模型宽度、高度、手机型号图片
      imgWidth: 180,
      imgHeight: 320,
      phoneImg: 'https://oss.hz2030.com/2020/10/17/3b4df02f853d413fb62e326b68dc0c18.png',
      // 页面宽度
      pageWidth: document.body.clientWidth,
      // 用户选择中图片、图片宽度
      selectImgs: '',
      selectImgWidth: 180,
      // 弹窗状态
      popupState: false,
      step: 1, // 分为1，2，3
      colors: '#ffffff',
      // 是否展示图片操作按钮
      showPictureOpt: false,
      // 当前选中图片的宽高
      curImgWidth: 0,
      curImgHeight: 0,
      // 旋转角度和放大倍数
      rotate: 0,
      scale: 1,
      pointX: 0, // 拖拽 点击时的X坐标
      pointY: 0, // 拖拽 点击时的Y坐标
      imgBoxLeft: 0, // 覆盖图片盒子偏移量
      imgBoxTop: 0,
      curImgIndex: -1,
      // 是否展示图片盒子
      showImgBox: true,
      // 是否展示输入框
      showTextBox: false,
      // 图片上文本内容
      textCon: '我晚上等你到了',
      tempTextCon: '',
      // 文本是否竖向排列
      isColumn: false,
      textBoxLeft: 0, // 覆盖文字盒子偏移量
      textBoxTop: 0,
      textBoxRotate: 0, // 覆盖文字盒子旋转角度
      textBoxScale: 1, // 覆盖文字盒子缩放倍数
      showCoverText: false // 是否展示文字盒子
    }
  },
  computed: {
    // 覆盖图片顶层盒子样式
    coverBoxClass () {
      return this.showPictureOpt ? 'cover_box zIndex15' : 'cover_box'
    },
    // 覆盖图片外层盒子样式
    imgBoxStyle () {
      return `left:calc(50% ${this.imgBoxLeft > 0 ? '- ' + this.imgBoxLeft : '+ ' + (-this.imgBoxLeft)}px);top:calc(50% ${this.imgBoxTop > 0 ? '- ' + this.imgBoxTop : '+ ' + (-this.imgBoxTop)}px);transform:translate(-50%,-50%) rotate(${this.rotate}deg) scale(${this.scale});`
    },
    // 覆盖文字盒子样式
    coverTextStyle () {
      return `color: ${this.colors};width:${this.isColumn ? '28px' : '140px'};left:calc(50% ${this.textBoxLeft > 0 ? '- ' + this.textBoxLeft : '+ ' + (-this.textBoxLeft)}px);top:calc(50% ${this.textBoxTop > 0 ? '- ' + this.textBoxTop : '+ ' + (-this.textBoxTop)}px);transform:translate(-50%,-50%) rotate(${this.textBoxRotate}deg) scale(${this.textBoxScale});`
    },
    // 文字图标盒子样式
    textIconStyle () {
      return `left:calc(50% ${this.textBoxLeft > 0 ? '- ' + this.textBoxLeft : '+ ' + (-this.textBoxLeft)}px);top:calc(50% ${this.textBoxTop > 0 ? '- ' + this.textBoxTop : '+ ' + (-this.textBoxTop)}px);transform:translate(-50%,-50%) rotate(${this.textBoxRotate}deg);`
    }
  },
  created () {
  },
  mounted () {
  },
  methods: {
    // 点击手机模型图
    phoneImgClick () {
      if (this.selectImgs) {
        this.showPictureOpt = true
      }
    },
    afterRead (file) {
      console.log(file)
      this.showImgBox = true
      this.showPictureOpt = true
      this.selectImgs = file.content
    },
    // 选择图片
    selectImg (item, ind) {
      if (this.curImgIndex === ind) {
        this.curImgIndex = -1
        this.selectImgs = ''
        this.showPictureOpt = false
        this.showImgBox = false
        this.step = 1
        this.rotate = 0
        this.scale = 1
        this.imgBoxTop = 0
        this.imgBoxLeft = 0
        return
      }
      this.showImgBox = true
      this.selectImgs = 'https://www.hz2030.com/userPicures/TOB20200806-RQMY6ZKL.jpeg'
      this.curImgIndex = ind
      this.$nextTick(() => {
        // this.drawCanvas()
      })
    },
    delImg () {
      this.curImgIndex = -1
      this.selectImgs = ''
      this.showPictureOpt = false
      this.showImgBox = false
      this.step = 1
      this.showTextBox = false
      this.rotate = 0
      this.scale = 1
      this.imgBoxTop = 0
      this.imgBoxLeft = 0
      this.showCoverText = false
    },
    drawCanvas () {
      const that = this
      console.log('this.$refs.canvasImg', that.$refs.canvasImg)
      // 读取手机壳模板图片参数
      const phoneBgObj = that.$refs.phoneShellTemplate
      // console.log('phoneBgObj====', that.$refs.phoneShellTemplate)
      // 模板距离视图左边距离、顶部距离，原始宽度，原始高度
      // const templateLeft = phoneBgObj.getBoundingClientRect().left
      // const templateTop = phoneBgObj.getBoundingClientRect().top
      const templateWidth = phoneBgObj.naturalWidth
      const templateHeight = phoneBgObj.naturalHeight
      // 读取制作图片参数
      const selectImgObj = that.$refs.selectImg
      // console.log('selectImgObj===', selectImgObj)
      // 制作图片原始宽度、原始高度
      const selectImgWidth = selectImgObj.naturalWidth
      const selectImgHeight = selectImgObj.naturalHeight
      // 计算制作图片宽高
      // 获取模板图片的实际宽度
      const templateScrollWidth = phoneBgObj.scrollWidth
      const templateScrollHeight = templateHeight / (templateWidth / templateScrollWidth)
      const scale = (selectImgWidth / templateScrollWidth) > (selectImgHeight / templateScrollHeight) ? (selectImgHeight / templateScrollHeight) : (selectImgWidth / templateScrollWidth)
      const adImgWidth = parseInt(selectImgWidth / scale)
      that.selectImgWidth = adImgWidth
      // console.log('that.selectImgWidth===', that.selectImgWidth)
      const canvas = new Canvas.Element()
      console.log('Canvas====', canvas)
    },
    // 下一步
    next () {
      console.log('下一步')
      if (!this.selectImgs || this.curImgWidth === 0 || this.curImgHeight === 0) {
        return
      }
      this.step = this.step + 1
      if (this.step === 2) {
        this.showPictureOpt = false
        this.showCoverText = true
        this.showTextBox = true
        this.tempTextCon = this.textCon
        this.$nextTick(() => {
          this.$refs.textInp.focus()
        })
      }
    },
    // 隐藏图片的操作按钮
    hidePictureOpt (evt) {
      if (evt.target === this.$refs.pageBoxRef) {
        this.showPictureOpt = false
      }
    },
    // 设置选中图片宽高
    setCurImgWH () {
      if (this.$refs.curImg.naturalWidth < this.$refs.curImg.naturalHeight) {
        this.curImgHeight = 306
        this.curImgWidth = 'auto'
      } else {
        this.curImgWidth = 160
        this.curImgHeight = 'auto'
      }
      // if (this.$refs.curImg.naturalWidth < this.$refs.curImg.naturalHeight) {
      //   this.curImgWidth = 160
      //   this.curImgHeight = 'auto'
      // } else {
      //   this.curImgHeight = 300
      //   this.curImgWidth = 'auto'
      // }
      this.showPictureOpt = true
    },
    // 缩放
    pinchHandler (evt) {
      evt.preventDefault()
      if (evt.zoom > 1) {
        this.scale = this.scale + 0.05
      } else {
        if (this.scale < 0.45) {
          this.scale = 0.4
        } else {
          this.scale = this.scale - 0.05
        }
      }
    },
    // 旋转
    rotateHandler (evt) {
      evt.preventDefault()
      this.rotate = evt.angle < 0 ? this.rotate - 1 : this.rotate + 1
    },
    // 图片拖拽
    getStartPoint (evt) {
      this.pointX = evt.touches[0].clientX
      this.pointY = evt.touches[0].clientY
    },
    imgBoxTouchMove (evt) {
      evt.preventDefault()
      let mPointX = evt.touches[0].clientX
      let mPointY = evt.touches[0].clientY
      let left = this.imgBoxLeft + this.pointX - mPointX
      let top = this.imgBoxTop + this.pointY - mPointY
      if (left > this.$refs.curImg.offsetWidth - 30 || left < 30 - this.$refs.curImg.offsetWidth || top > this.$refs.curImg.offsetHeight - 30 || top < 30 - this.$refs.curImg.offsetHeight) {
        return
      }
      this.imgBoxLeft = left
      this.imgBoxTop = top
      this.pointX = mPointX
      this.pointY = mPointY
    },
    // 覆盖文字盒子缩放
    pinchTextHandler (evt) {
      evt.preventDefault()
      if (evt.zoom > 1) {
        this.textBoxScale = this.textBoxScale + 0.05
      } else {
        if (this.textBoxScale < 0.45) {
          this.textBoxScale = 0.4
        } else {
          this.textBoxScale = this.textBoxScale - 0.05
        }
      }
    },
    // 覆盖文字盒子旋转
    rotateTextHandler (evt) {
      evt.preventDefault()
      this.textBoxRotate = evt.angle < 0 ? this.textBoxRotate - 1 : this.textBoxRotate + 1
    },
    // 文字盒子拖拽
    textBoxTouchMove (evt) {
      evt.preventDefault()
      let mPointX = evt.touches[0].clientX
      let mPointY = evt.touches[0].clientY
      let left = this.textBoxLeft + this.pointX - mPointX
      let top = this.textBoxTop + this.pointY - mPointY
      if (left > 60 || left < -60 || top > 200 || top < -200) {
        return
      }
      this.textBoxLeft = left
      this.textBoxTop = top
      this.pointX = mPointX
      this.pointY = mPointY
    },
    // 删除文字
    delText () {
      this.colors = '#ffffff'
      this.textCon = ''
      this.textBoxLeft = 0
      this.textBoxTop = 0
      this.textBoxRotate = 0
      this.textBoxScale = 1
    },
    // 编辑文本
    showTextEdit () {
      this.showTextBox = true
      this.tempTextCon = this.textCon
      this.$nextTick(() => {
        this.$refs.textInp.focus()
      })
    },
    // 切换文本横竖排列
    changeArray () {
      this.isColumn = !this.isColumn
    },
    textInpBlur (e) {
      this.showTextBox = false
      this.tempTextCon = ''
    },
    // 保存覆盖文字
    saveText (e) {
      this.textCon = this.tempTextCon
    }
  }
}
</script>

<style scoped>
  .phone_box{
    width: 180px;
    height: 320px;
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 20px auto 30px;
  }
  .canvas_box {
    width: 180px;
    height: 320px;
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 20px auto 30px;
    /* overflow: hidden; */
  }
  /* 手机壳样式 */
  .phone_shell_template {
    position: absolute;
    margin: auto;
    z-index: 10;
  }
  /* 缓存选择图片 */
  .select_img {
    position: absolute;
    top: -9999px;
    left: -9999px;
    height: auto;
  }
  /* 图片列表 */
  .select_img_box {
    display: flex;
    flex-direction: row;
    align-items: flex-start;
    justify-content: flex-start;
    padding: 0 10px;
  }
  .img_list{
    margin-right: 10px;
  }
  .van-image__img{
    border-radius: 8px;
  }
  /* 上传图片 */
  .upload_button{
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    width: 80px;
    height: 140px;
    background: rgba(0, 0, 0, 0.8);
    border-radius: 8px;
    color: #fff;
    margin-right: 10px;
  }
  .text{
    font-size: 12px;
    margin-top: 5px;
  }
  .van-popup--bottom {
    height: '30%';
    display: flex;
    flex-direction: column;
    justify-content: center;
  }
  .van-popup--bottom p{
    margin: 0;
    height: 60px;
    line-height: 60px;
    text-align: center;
    color: #1b1b1b;
    border-top: 1px solid #ebedf0;
  }
  /deep/ .van-uploader__wrapper{
    display: block !important;
  }
  .font_slider{
    display: flex;
    flex-direction: row;
    flex-flow: row;
    overflow-x: auto;
  }
  .font_slider span{
    flex:none;
    padding: 0 30px;
  }
  .bottom-btn{
    width: 90%;
    margin: 30px auto 0;
  }
  /* 覆盖图片盒子 */
  .cover_box{
    position: absolute;
    width: 180px;
    height: 320px;
    z-index: 5;
  }
  .zIndex15{
    z-index: 15;
  }
  /* 操作图标 */
  .opt_icon{
    position: absolute;
    width: 24px;
    height: 24px;
    z-index: 17;
  }
  .del_icon{
    top: -5px;
    left: -5px;
  }
  .scale_icon{
    bottom: -5px;
    left: -5px;
  }
  .rotate_icon{
    top: -5px;
    right: -5px;
  }
  .text_close_icon{
    position: absolute;
    width: 26px;
    height: 26px;
    top: -8px;
    left: -8px;
  }
  .text_edit_icon{
    position: absolute;
    width: 24px;
    height: 24px;
    right: -7px;
    top: -7px;
  }
  .text_scale_icon{
    position: absolute;
    width: 22px;
    height: 22px;
    right: -5px;
    bottom: -5px;
  }
  .text_dir_icon{
    position: absolute;
    width: 30px;
    height: 30px;
    left: -10px;
    bottom: -10px;
  }
  .text_icon_box{
    position: absolute;
    border: 1px solid #fff;
    width: 110px;
    height: 110px;
    /* top: 50%;
    left: 50%;
    transform: translate(-50%, -50%); */
    z-index: 12;
  }
  .img_box{
    position: absolute;
    /* transform: translate(-50%,-50%); */
  }
  .cover_text{
    position: absolute;
    width: 180px;
    height: 320px;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 8;
    font-size: 24px;
    font-weight: bold;
    overflow: hidden;
  }
  .text_con_box{
    position: absolute;
    max-height: 260px;
    max-width: 140px;
    text-align: center;
    overflow: hidden;
  }
  .text-box-modal{
    position: fixed;
    width: 100%;
    height: 100vh;
    top: 0;
    left: 0;
    z-index: 99;
  }
  .text-box{
    position: fixed;
    width: 100%;
    bottom: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    background: #ffffff;
    padding: 14px 0;
  }
  .textInp{
    width: 60%;
    height: 32px;
    margin-right: 10px;
    border: 1px solid #333333;
    border-radius: 16px;
    padding: 0 8px;
  }
  .add_text_btn{
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 12;
  }
</style>
