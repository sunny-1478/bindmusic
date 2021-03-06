<template>
  <div class="player" v-show="playlist.length > 0">
    <transition
      name="normal"
      @enter="enter"
      @after-enter="afterEnter"
      @leave="leave"
      @after-leave="afterLeave"
    >
      <!-- 基础播放器 -->
      <div class="normal-player" v-show="fullScreen">
        <!-- 背景图 -->
        <div class="background">
          <img width="100%" height="100%" :src="currentSong.image" />
        </div>
        <div class="top">
          <!-- 顶部左上角返回按钮 -->
          <div class="back" @click="back">
            <i class="icon-back"></i>
          </div>
          <!-- 顶部歌曲名称 -->
          <h1 class="title" v-html="currentSong.name"></h1>
          <!-- 歌手名称 -->
          <h2 class="subtitle" v-html="currentSong.singer"></h2>
        </div>
        <!-- 中间唱片 -->
        <div class="middle" @touchstart.prevent="middleTouchStart" 
        @touchmove.prevent="middleTouchMove"
        @touchend.prevent="middleTouchEnd">
          <div class="middle-l" ref="middleL">
            <div class="cd-wrapper" ref="cdWrapper">
              <div class="cd" :class="cdCls">
                <img :class="cdCls" class="image" :src="currentSong.image" />
              </div>
            </div>
          </div>
          <!-- 歌词 -->
          <scroll class="middle-r" ref="lyricList" :data="currentLyric&&currentLyric.lines">
            <div class="lyric-wrapper">
              <div v-if="currentLyric">
                <p ref="lyricLine" class="text"
                :class="{'current':currentLineNum===index}"
                 v-for="(line,index) in currentLyric.lines" :key="index" >{{line.txt}}</p>
              </div>
            </div>
          </scroll>
        </div>
        <!-- 底部操作区 -->
        <div class="bottom">
          <!-- 歌词切换按钮 -->
          <div class="dot-wrapper" >
            <span class="dot" :class="{'active':currentShow==='cd'}"></span>
            <span class="dot" :class="{'active':currentShow==='lyric'}"></span>
          </div>
          <!-- 进度条 -->
          <div class="progress-wrapper">
            <!-- 当前播放时长 -->
            <span class="time time-l">{{ format(currentTime) }}</span>
            <div class="progress-bar-wrapper">
              <!-- 进度条 -->
              <!-- 传入percent -->
              <progress-bar
                :percent="percent"
                @percentChange="onProgressBarChange"
              ></progress-bar>
            </div>
            <!-- 总播放时长 -->
            <span class="time time-r">{{ format(currentSong.duration) }}</span>
          </div>
          <div class="operators">
            <!-- changeMode改变歌曲播放模式 -->
            <div class="icon i-left" @click="changeMode">
              <i :class="iconMode"></i>
            </div>
            <div class="icon i-left" :class="disableCls">
              <!-- 添加点击事件上一首 -->
              <i @click="prev" class="icon-prev"></i>
            </div>
            <div class="icon i-center" :class="disableCls">
              <i @click="togglePlaying" :class="playIcon"></i>
            </div>
            <div class="icon i-right" :class="disableCls">
              <!-- 添加点击事件下一首 -->
              <i @click="next" class="icon-next"></i>
            </div>
            <div class="icon i-right">
              <i class="icon icon-not-favorite"></i>
            </div>
          </div>
        </div>
      </div>
    </transition>
    <!-- 隐藏播放器时底部的小播放器 -->
    <transition name="mini">
      <div class="mini-player" v-show="!fullScreen" @click="open">
        <!-- 小型唱片图片 -->
        <div class="icon">
          <div class="imgWrapper" ref="miniWrapper">
            <img
              :class="cdCls"
              width="40"
              height="40"
              :src="currentSong.image"
            />
          </div>
        </div>
        <!-- 歌手名称和歌曲名称 -->
        <div class="text">
          <h2 class="name" v-html="currentSong.name"></h2>
          <p class="desc" v-html="currentSong.singer"></p>
        </div>
        <!-- 播放按钮 -->
        <div class="control">
          <progress-circle :radius="radius" :percent="percent">
            <!-- .stop阻止冒泡 -->
            <i
              @click.stop="togglePlaying"
              class="icon-mini"
              :class="miniIcon"
            ></i>
          </progress-circle>
        </div>
        <!-- 点开歌曲列表按钮 -->
        <div class="control">
          <i class="icon-playlist"></i>
        </div>
      </div>
    </transition>
    <!-- 绑定歌曲路径 -->
    <!-- 歌曲从加载到播放 有一个@canplay 歌曲播放时有一个timeupdate事件 -->
    <audio
      ref="audio"
      :src="currentSong.url"
      @canplay="ready"
      @error="error"
      @timeupdate="updateTime"
      @ended="end"
    ></audio>
  </div>
</template>

<script>
import { mapGetters, mapMutations } from "vuex";
import { prefixStyle } from "common/js/dom";
import animations from "create-keyframe-animation";
import ProgressCircle from "base/progress-circle/progress-circle";
// 引入进度条
import ProgressBar from "base/progress-bar/progress-bar";
// 引入播放模式
import { playMode } from 'common/js/config'
import {shuffle} from 'common/js/util'
import Lyric from 'lyric-parser'
import Scroll from 'base/scroll/scroll'


const transform = prefixStyle("transform");
const transitionDuration = prefixStyle('transitionDuration')

export default {
  data() {
    return {
      // 当歌曲ready时 才能点击下一首播放
      songReady: false,
      // 进度条当前时间
      currentTime: 0,
      radius:32,
      currentLyric:null,
      // 当前歌词所在行
      currentLineNum:0,
      // cd和歌词默认选中为cd
      currentShow:'cd'
    };
  },
  computed: {
    cdCls() {
      return this.playing ? "play" : "play pause";
    },
    playIcon() {
      return this.playing ? "icon-pause" : "icon-play";
    },
    miniIcon() {
      return this.playing ? "icon-pause-mini" : "icon-play-mini";
    },
    // 当不能点击时 给按钮一个disabled属性
    disableCls() {
      return this.songReady ? "" : "disable";
    },
    // 计算进度条百分比：歌曲播放比例等于当前播放的时间除以歌曲总时长
    percent() {
      return this.currentTime / this.currentSong.duration;
    },
    // 播放模式 从config.js中拿到playMode
    iconMode(){
      return this.mode=== playMode.sequence ? 'icon-sequence' : this.mode === playMode.loop ? 'icon-loop' :'icon-random'
    },
    // 导入mapGetter
    ...mapGetters([
      // 控制显示隐藏
      "fullScreen",
      // 控制播放器的渲染
      "playlist",
      //当前歌曲信息
      "currentSong",
      // 歌曲播放状态
      "playing",
      // 映射getter中的currentIndex
      "currentIndex",
      // 通过this.mode访问到当前的播放模式
      "mode",
      'sequenceList'
    ]),
  },
  created(){
    this.touch={}
  },
  methods: {
    // 点击左上角按钮使播放界面隐藏 下方mini播放器显示
    back() {
      this.setFullScreen(false);
    },
    // 播放界面隐藏时 点击mini播放器时使播放器显示
    open() {
      this.setFullScreen(true);
    },
    // @enter="enter"@after-enter="afterEnter"@leave="leave"@after-leave="afterLeave" 四个钩子函数定义
    enter(el, done) {
      const { x, y, scale } = this._getPosAndScale();

      let animation = {
        0: {
          transform: `translate3d(${x}px,${y}px,0) scale(${scale})`,
        },
        60: {
          transform: `translate3d(0,0,0) scale(1.1)`,
        },
        100: {
          transform: `translate3d(0,0,0) scale(1)`,
        },
      };

      animations.registerAnimation({
        name: "move",
        animation,
        presets: {
          duration: 400,
          easing: "linear",
        },
      });

      animations.runAnimation(this.$refs.cdWrapper, "move", done);
    },
    afterEnter() {
      animations.unregisterAnimation("move");
      this.$refs.cdWrapper.style.animation = "";
    },
    leave(el, done) {
      this.$refs.cdWrapper.style.transition = "all 0.4s";
      const { x, y, scale } = this._getPosAndScale();
      this.$refs.cdWrapper.style[
        transform
      ] = `translate3d(${x}px,${y}px,0) scale(${scale})`;
      const timer = setTimeout(done, 400);
      this.$refs.cdWrapper.addEventListener("transitionend", () => {
        clearTimeout(timer);
        done();
      });
    },
    afterLeave() {
      this.$refs.cdWrapper.style.transition = "";
      this.$refs.cdWrapper.style[transform] = "";
    },
    togglePlaying() {
      if (!this.songReady) {
        return;
      }
      this.setPlayingState(!this.playing);
      if(this.currentLyric){
        this.currentLyric.togglePlay()
      }
    },
    // 歌曲播放完时 播放下一首
    end(){
      // 如果是单曲循环模式 从头播放当前歌曲
      if(this.mode===playMode.loop){
        this.loop()
      }else{
      this.next()
      }
    },
    // 循环播放模式下 如果调用循环播放 改变当前歌曲播放时间为0
    loop(){
        this.$refs.audio.currentTime=0
        this.$refs.audio.play()
        if(this.currentLyric){
          this.currentLyric.seek(0)
        }
    },
    // 下一首
    next() {
      if (!this.songReady) {
        return;
      }
      let index = this.currentIndex + 1;
      // 当前歌曲为最后一首时 返回第一首播放
      if (index == this.playlist.length) {
        index = 0;
      }
      // 调用setCurrentIndex来修改index
      this.setCurrentIndex(index);
      // 如果播放状态为暂停 点击下一首时 打开歌曲播发
      if (!this.playing) {
        this.togglePlaying();
      }
      this.songReady = false;
    },
    // 上一首
    prev() {
      // 如果歌曲没有准备好 return
      if (!this.songReady) {
        return;
      }
      let index = this.currentIndex - 1;
      if (index == -1) {
        index = this.playlist.length - 1;
      }
      this.setCurrentIndex(index);
      if (!this.playing) {
        this.togglePlaying();
      }
      this.songReady = false;
    },
    ready() {
      this.songReady = true;
    },
    paused() {
        this.setPlayingState(false)
        if (this.currentLyric) {
          this.currentLyric.stop()
        }
      },
    // 当歌曲加载失败 出发error
    error() {
      this.songReady = true;
    },
    // 定义updateTime,参数是一个event对象 target对应audio标签
    updateTime(e) {
      // 把audio标签的currentTime赋值给data里边的currentTime
      this.currentTime = e.target.currentTime;
    },
    // 改变歌曲播放模式方法
    changeMode(){
      const mode=(this.mode+1)%3;
      this.setPlayMode(mode)
      let list=null
      if(mode===playMode.random){
        list=shuffle(this.sequenceList)
      }else{
        list=this.sequenceList
      }
      this.resetCurrentIndex(list)
      this.setPlayList(list)
      // console.log(currentSong)
    },
    // 切换播放模式当前播放歌曲不发生改变
    resetCurrentIndex(list){
      let index=list.findIndex((item)=>{
        return item.id===this.currentSong.id
      })
      this.setCurrentIndex(index)
    },
    // 时间戳函数
    format(interval) {
      interval = interval | 0; //一个正数向下取整
      const minute = (interval / 60) | 0;
      const second = this._pad(interval % 60);
      // 返回分和秒
      return `${minute}:${second}`;
    },
    // 获取歌词
    getLyric(){
      this.currentSong.getLyric().then((lyric)=>{
        this.currentLyric=new Lyric(lyric,this.handleLyric)
        if(this.playing){
          this.currentLyric.play()
        }
      })
    },
    // 当歌曲每一行发生改变时 触发回调函数 歌词高亮显示
    handleLyric({lineNum,txt}){
      this.currentLineNum=lineNum
      // 当歌词显示到大于第五行时 开始滚动
      if(lineNum>5){
        let lineEl=this.$refs.lyricLine[lineNum-5]
        this.$refs.lyricList.scrollToElement(lineEl,1000)
      }else{
        this.$refs.lyricList.scrollTo(0,0,1000)
      }
    },
    middleTouchStart(e){
      this.touch.initiated=true
      const touch=e.touches[0]
      this.touch.startX=touch.pageX
      this.touch.startY=touch.pageY
    },
    middleTouchMove(e){
      if(!this.touch.initiated){
        return
      }
      const touch=e.touches[0]
      const deltaX=touch.pageX-this.touch.startX
      const deltaY=touch.pageY-this.touch.startY
      if(Math.abs(deltaY)>Math.abs(deltaX)){
        return
      }
      const left=this.currentShow==='cd'?0:-window.innerWidth
      const offsetWidth=Math.min(0, Math.max(-window.innerWidth, left + deltaX))
        this.touch.percent = Math.abs(offsetWidth / window.innerWidth)
      this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px,0,0)`
        this.$refs.lyricList.$el.style[transitionDuration] = 0
        this.$refs.middleL.style.opacity = 1 - this.touch.percent
        this.$refs.middleL.style[transitionDuration] = 0
    },
    middleTouchEnd(e){
      let offsetWidth
      let opacity
      if (this.currentShow==='cd') {
        if(this.touch.percent>0.1){
          offsetWidth=-window.innerWidth
          this.currentShow='lyric'
            opacity = 0

        }else{
          offsetWidth=0
            opacity = 1

        }
      }else{
        if(this.touch.percent<0.9){
          offsetWidth=0
          this.currentShow='cd'
            opacity = 1

        }else{
          offsetWidth=-window.innerWidth
            opacity = 0
        }
      }
      const time = 300
      this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px,0,0)`
      this.$refs.lyricList.$el.style[transitionDuration] = `${time}ms`
      this.$refs.middleL.style.opacity =opacity
        this.$refs.middleL.style[transitionDuration] =  `${time}ms`
    },
    // 数字为一位时，用0补位为两位 n为字符串长度
    _pad(num, n = 2) {
      // 获取number字符串长度
      let len = num.toString().length;
      // 当字符串长度小于两位时  用0补位
      while (len < n) {
        num = "0" + num;
        len++;
      }
      return num;
    },
    // 监听进度条变化 通过audio实例改变播放时间
    onProgressBarChange(percent) {
      const currentTime=this.currentSong.duration * percent;
      this.$refs.audio.currentTime = currentTime
      // 如果当前播放状态为暂停 改变播放状态为播放
      if (!this.playing) {
        this.togglePlaying();
      }
      if(this.currentLyric){
        this.currentLyric.seek(currentTime*1000)
      }
    },
    //   定义一个函数获取小唱片初始位置
    _getPosAndScale() {
      // 目标宽度
      const targetWidth = 40;
      const paddingLeft = 40;
      const paddingBottom = 30;
      const paddingTop = 80;
      const width = window.innerWidth * 0.8;
      // 初始缩放比例
      const scale = targetWidth / width;
      const x = -(window.innerWidth / 2 - paddingLeft);
      const y = window.innerHeight - paddingTop - width / 2 - paddingBottom;
      return {
        x,
        y,
        scale,
      };
    },
    /**
     * 计算内层Image的transform，并同步到外层容器
     * @param wrapper
     * @param inner
     */
    syncWrapperTransform(wrapper, inner) {
      if (!this.$refs[wrapper]) {
        return;
      }
      let imageWrapper = this.$refs[wrapper];
      let image = this.$refs[inner];
      let wTransform = getComputedStyle(imageWrapper)[transform];
      let iTransform = getComputedStyle(image)[transform];
      imageWrapper.style[transform] =
        wTransform === "none" ? iTransform : iTransform.concat(" ", wTransform);
    },
    ...mapMutations({
      // 映射setFullScreen
      setFullScreen: "SET_FULL_SCREEN",
      setPlayingState: "SET_PLAYING_STATE",
      // 映射store中mutation-tyes中的SET_CURRENT_INDEX
      setCurrentIndex: "SET_CURRENT_INDEX",
      // 映射播放模式
      setPlayMode:'SET_PLAY_MODE',
      setPlayList:'SET_PLAYLIST'
    }),
  },
  // 监听currentSong的变化
  watch: {
    currentSong(newSong,oldSong) {
      if(newSong.id===oldSong.id){
        return
      }
      if(this.currentLyric){
        this.currentLyric.stop()
      }
      this.$nextTick(() => {
        this.$refs.audio.play();
        this.getLyric()
      });
    },
    playing(newPlaying) {
        if (!this.songReady) {
          return
        }
        const audio = this.$refs.audio
        this.$nextTick(() => {
          newPlaying ? audio.play() : audio.pause()
        })
        if (!newPlaying) {
          if (this.fullScreen) {
            this.syncWrapperTransform('imageWrapper', 'image')
          } else {
            this.syncWrapperTransform('miniWrapper', 'miniImage')
          }
        }
      },
  },
  // 注册组件
  components: {
    ProgressBar,
    ProgressCircle,
    Scroll
  },
};
</script>

</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
  @import "~common/stylus/variable"
  @import "~common/stylus/mixin"

  .player
    .normal-player
      position: fixed
      left: 0
      right: 0
      top: 0
      bottom: 0
      z-index: 150
      background: $color-background
      .background
        position: absolute
        left: 0
        top: 0
        width: 100%
        height: 100%
        z-index: -1
        opacity: 0.6
        filter: blur(20px)
      .top
        position: relative
        margin-bottom: 25px
        .back
          position absolute
          top: 0
          left: 6px
          z-index: 50
          .icon-back
            display: block
            padding: 9px
            font-size: $font-size-large-x
            color: $color-theme
            transform: rotate(-90deg)
        .title
          width: 70%
          margin: 0 auto
          line-height: 40px
          text-align: center
          no-wrap()
          font-size: $font-size-large
          color: $color-text
        .subtitle
          line-height: 20px
          text-align: center
          font-size: $font-size-medium
          color: $color-text
      .middle
        position: fixed
        width: 100%
        top: 80px
        bottom: 170px
        white-space: nowrap
        font-size: 0
        .middle-l
          display: inline-block
          vertical-align: top
          position: relative
          width: 100%
          height: 0
          padding-top: 80%
          .cd-wrapper
            position: absolute
            left: 10%
            top: 0
            width: 80%
            box-sizing: border-box
            height: 100%
            .cd
              width: 100%
              height: 100%
              border-radius: 50%
              .image
                position: absolute
                left: 0
                top: 0
                width: 100%
                height: 100%
                box-sizing: border-box
                border-radius: 50%
                border: 10px solid rgba(255, 255, 255, 0.1)
                &.play
                  animation: rotate 20s linear infinite
                &.pause
                  animation-play-state:paused
          .playing-lyric-wrapper
            width: 80%
            margin: 30px auto 0 auto
            overflow: hidden
            text-align: center
            .playing-lyric
              height: 20px
              line-height: 20px
              font-size: $font-size-medium
              color: $color-text-l
        .middle-r
          display: inline-block
          vertical-align: top
          width: 100%
          height: 100%
          overflow: hidden
          .lyric-wrapper
            width: 80%
            margin: 0 auto
            overflow: hidden
            text-align: center
            .text
              line-height: 32px
              color: $color-text-l
              font-size: $font-size-medium
              &.current
                color: $color-text
            .pure-music
              padding-top: 50%
              line-height: 32px
              color: $color-text-l
              font-size: $font-size-medium
      .bottom
        position: absolute
        bottom: 50px
        width: 100%
        .dot-wrapper
          text-align: center
          font-size: 0
          .dot
            display: inline-block
            vertical-align: middle
            margin: 0 4px
            width: 8px
            height: 8px
            border-radius: 50%
            background: $color-text-l
            &.active
              width: 20px
              border-radius: 5px
              background: $color-text-ll
        .progress-wrapper
          display: flex
          align-items: center
          width: 80%
          margin: 0px auto
          padding: 10px 0
          .time
            color: $color-text
            font-size: $font-size-small
            flex: 0 0 30px
            line-height: 30px
            width: 30px
            &.time-l
              text-align: left
            &.time-r
              text-align: right
          .progress-bar-wrapper
            flex: 1
        .operators
          display: flex
          align-items: center
          .icon
            flex: 1
            color: $color-theme
            &.disable
              color: $color-theme-d
            i
              font-size: 30px
          .i-left
            text-align: right
          .i-center
            padding: 0 20px
            text-align: center
            i
              font-size: 40px
          .i-right
            text-align: left
          .icon-favorite
            color: $color-sub-theme
      &.normal-enter-active, &.normal-leave-active
        transition: all 0.4s
        .top, .bottom
          transition: all 0.4s cubic-bezier(0.86, 0.18, 0.82, 1.32)
      &.normal-enter, &.normal-leave-to
        opacity: 0
        .top
          transform: translate3d(0, -100px, 0)
        .bottom
          transform: translate3d(0, 100px, 0)
    .mini-player
      display: flex
      align-items: center
      position: fixed
      left: 0
      bottom: 0
      z-index: 180
      width: 100%
      height: 60px
      background: $color-highlight-background
      &.mini-enter-active, &.mini-leave-active
        transition: all 0.4s
      &.mini-enter, &.mini-leave-to
        opacity: 0
      .icon
        flex: 0 0 40px
        width: 40px
        height: 40px
        padding: 0 10px 0 20px
        .imgWrapper
          height: 100%
          width: 100%
          img
            border-radius: 50%
            &.play
              animation: rotate 10s linear infinite
            &.pause
              animation-play-state: paused
      .text
        display: flex
        flex-direction: column
        justify-content: center
        flex: 1
        line-height: 20px
        overflow: hidden
        .name
          margin-bottom: 2px
          no-wrap()
          font-size: $font-size-medium
          color: $color-text
        .desc
          no-wrap()
          font-size: $font-size-small
          color: $color-text-d
      .control
        flex: 0 0 30px
        width: 30px
        padding: 0 10px
        .icon-play-mini, .icon-pause-mini, .icon-playlist
          font-size: 30px
          color: $color-theme
        .icon-mini
          font-size: 32px
          position: absolute
          left: 0
          top: 0

  @keyframes rotate
    0%
      transform: rotate(0)
    100%
      transform: rotate(360deg)
</style>
