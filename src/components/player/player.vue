<template>
  <div class="player" v-show="playlist.length > 0">
        <!--展开收起的动画-->
    <transition name="normal" >
      <div class="normal-player" v-show="fullScreen" @touchstart.once="firstPlay">
        <div class="background">
          <!-- <transition name="filterR">
          <div class="filterR" v-show="currentShow === 'lyric'"></div>
          </transition> -->
          <div class="filter"></div>
          <img :src="currentSong.image" width="100%" height="100%">
        </div>
        <div class="top">
          <div class="back" @click="back">
            <i class="fa fa-angle-down"></i>
          </div>
          <h1 class="title" v-html="currentSong.name"></h1>
          <h2 class="subtitle" v-html="currentSong.singer"></h2>
        </div>
        <div class="middle" @click="changeMiddle">
          <!-- cd页面 -->
          <transition name="middleL">
            <div class="middle-l" v-show="currentShow === 'cd'">
              <div class="cd-wrapper">
                <div class="cd" :class="cdCls" >
                  <img :src="currentSong.image" class="image">
                </div>
              </div>
            </div>
          </transition>
          <!--歌词部分-->
          <transition name="middleR">
            <scroll class="middle-r" ref="lyricList" v-show="currentShow === 'lyric'" :data="currentLyric && currentLyric.lines">
              <div class="lyric-wrapper">
                <div class="currentLyric" v-if="currentLyric">
                  <p ref="lyricLine" v-for="(line, index) in currentLyric.lines" :key="line.key"
                  class="text" :class="{'current': currentLineNum === index}">
                    {{line.txt}}
                  </p>
                </div>
                  <p class="no-lyric" v-if="currentLyric === null">{{upDatecurrentLyric}}</p>
              </div>
            </scroll>
          </transition>
        </div>
        <div class="bottom">
            <!--进度条-->
          <div class="progress-wrapper">
            <span class="time time-l">{{format(currentTime)}}</span>
            <div class="progress-bar-wrapper">
              <!-- 横向进度条 -->
              <progress-bar :percent="percent" @percentChangeEnd="percentChangeEnd" @percentChange="percentChange"></progress-bar>
            </div>
            <span class="time time-r">{{format(duration)}}</span>
          </div>
            <!--操作，如下一首上一首-->
          <div class="operators">
            <div class="icon i-left" >
              <i class="iconfont mode" :class="iconMode" @click="changeMode"></i>
            </div>
            <div class="icon i-left" >
              <i class="iconfont icon-prev" @click="prev"></i>
            </div>
            <div class="icon i-center" >
              <i class="fa" @click="togglePlaying" :class="playIcon"></i>
            </div>
            <div class="icon i-right" >
              <i class="iconfont icon-test" @click="next"></i>
            </div>
            <div class="icon i-right">
              <i class="iconfont"  @click="toggleFavorite(currentSong)" :class="getFavoriteIcon(currentSong)"></i>
            </div>
          </div>
        </div>
      </div>
    </transition>
    <!-- 迷你栏，在fullScreen这个state值为false时显示 -->
    <transition name="mini">
      <div class="mini-player" v-show="!fullScreen" @click.stop="open">
        <div class="icon">
          <img :class="cdCls"  :src="currentSong.image" width="40" height="40">
        </div>
        <div class="text">
          <h2 class="name" v-html="currentSong.name"></h2>
          <div class="desc" v-html="currentSong.singer"></div>
        </div>
        <div class="control" @click.stop="togglePlaying">
          <!-- 圆形进度条 -->
          <progress-circle :radius="radius" :percent="percent">
            <!-- <i class="icon-mini" :class="miniIcon" ></i> -->
            <i class="fa" :class="miniIcon"></i>
          </progress-circle>
        </div>
        <!-- 显示播放列表 -->
        <div class="control" @click.stop="showPlaylist">
          <i class="iconfont icon-caidan1"></i>
        </div>
      </div>
    </transition>
    <!-- 播放列表,不用传值，vuex就行了 -->
    <playlist @stopMusic="stopMusic" ref="playlist"></playlist>
    <!-- 播放音乐相关 -->
    <audio id="music-audio" ref="audio" @ended="end" autoplay @canplay="ready" @error="error" @timeupdate="updateTime"></audio>
  </div>
</template>

<script>
import ProgressCircle from "base/progress-circle/progress-circle";
import ProgressBar from "base/progress-bar/progress-bar";
import Lyric from "lyric-parser"; //歌曲解析插件
import Scroll from "base/scroll/scroll";
import Playlist from "cpnts/playlist/playlist";
import { mapGetters, mapMutations, mapActions } from "vuex";
import { getSong, getLyric } from "api/song";
import { playMode } from "common/js/config";
import { shuffle } from "common/js/utl";

export default {
  data() {
    return {
      // id: '',
      url: "",
      songReady: false,
      currentTime: 0,
      duration: 0,
      percent: 0,
      radius: 32,
      currentLyric: null,
      currentLineNum: 0,
      currentShow: "cd",
      noLyric: false
    };
  },
  created() {
    this.move = false;
  },
  computed: {
    // icon切换
    iconMode() {
      if (this.mode === playMode.sequence) {
        return "icon-next";
      } else if (this.mode === playMode.loop) {
        return "icon-loop";
      } else {
        return "icon-random";
      }
    },
    // 添加了play类时，会触发animation的rotate属性
    cdCls() {
      return this.playing ? "play" : "play pause";
    },
    miniIcon() {
      return this.playing ? "fa-pause" : "fa-play";
    },
    playIcon() {
      return this.playing ? "fa-pause" : "fa-play";
    },
    upDatecurrentLyric() {
      if (this.noLyric) {
        return "暂无歌词";
      }
      if (!this.noLyric) {
        return "歌词加载中";
      }
    },
    ...mapGetters([
      "playlist",
      "fullScreen",
      "currentSong",
      "playing",
      "currentIndex",
      "mode",
      "sequenceList",
      "favoriteList"
    ])
  },
  watch: {
    // 当前歌曲更换时的操作
    currentSong(newVal, oldVal) {
      if (!newVal.id) {
        return;
      }
      if (newVal.id === oldVal.id) {
        return;
      }
      // this.$refs.audio.pause()
      this.$refs.audio.currentTime = 0;
      this._getSong(newVal.id);
    },
    // url更换时的操作，给audio的src属性赋值，同时触发audio的play方法播放
    url(newUrl) {
      this._getLyric(this.currentSong.id);
      this.$refs.audio.src = newUrl;
      let play = setInterval(() => {
        if (this.songReady) {
          this.$refs.audio.play();
          clearInterval(play);
        }
        console.log("play");
      }, 20);
      let stop = setInterval(() => {
        this.duration = this.$refs.audio.duration;
        if (this.duration) {
          clearInterval(stop);
        }
      }, 150);
      this.setPlayingState(true);
    },
    //进度条相关，获取当前播放的百分比
    currentTime() {
      this.percent = this.currentTime / this.duration;
    }
  },
  methods: {
    firstPlay() {
      console.log("firstPlay");
      this.$refs.audio.play();
    },
    stopMusic() {
      // 删除最后一首的时候暂停音乐
      // this.$refs.audio.pause()
      console.log("删除最后一首的时候暂停音乐");
    },
    // 点击显示播放列表
    showPlaylist() {
      this.$refs.playlist.show();
    },
    // 切换cd和歌词页
    changeMiddle() {
      if (this.currentShow === "cd") {
        this.currentShow = "lyric";
      } else {
        this.currentShow = "cd";
      }
    },
    // 添加收藏相关。
    getFavoriteIcon(song) {
      if (this.isFavorite(song)) {
        return "icon-like";
      }
      return "icon-dislike";
    },
    toggleFavorite(song) {
      if (this.isFavorite(song)) {
        this.deleteFavoriteList(song);
      } else {
        this.saveFavoriteList(song); //提交action，然后存储到localstorage中
      }
    },
    isFavorite(song) {
      const index = this.favoriteList.findIndex(item => {
        return item.id === song.id;
      });
      return index > -1;
    },
    // 改变循环模式
    changeMode() {
      const mode = (this.mode + 1) % 3;
      this.setPlayMode(mode);
      let list = null;
      if (mode === playMode.random) {
        list = shuffle(this.sequenceList);
      } else {
        list = this.sequenceList;
      }
      this._resetCurrentIndex(list);
      this.setPlaylist(list);
    },
    _resetCurrentIndex(list) {
      let index = list.findIndex(item => {
        return item.id === this.currentSong.id;
      });
      this.setCurrentIndex(index);
    },
    // 进度条被拖动时
    percentChange(percent) {
      this.move = true;
      const currentTime = this.duration * percent;
      this.currentTime = currentTime;
      if (this.currentLyric) {
        this.currentLyric.seek(currentTime * 1000);
      }
    },
    // 进度条拖动结束时
    percentChangeEnd(percent) {
      this.move = false;
      const currentTime = this.duration * percent;
      this.$refs.audio.currentTime = currentTime;
      if (!this.playing) {
        this.$refs.audio.play();
        this.setPlayingState(true);
      }
      if (this.currentLyric) {
        this.currentLyric.seek(currentTime * 1000);
      }
    },
    // 监听当前播放位置，获取到事件的currentTime属性，赋值到data属性
    updateTime(e) {
      if (this.move) {
        return;
      }
      this.currentTime = e.target.currentTime;
    },
    // 格式化时间
    format(interval) {
      interval = interval | 0;
      let minute = (interval / 60) | 0;
      let second = interval % 60;
      if (second < 10) {
        second = "0" + second;
      }
      return minute + ":" + second;
    },
    // 结束时触发的事件，如果当前循环模式是单曲循环，那么触发loop方法，否则下一曲
    end() {
      if (this.mode === playMode.loop) {
        this.loop();
      } else {
        this.next();
      }
    },
    // 继续循环本首
    loop() {
      this.$refs.audio.currentTime = 0;
      this.$refs.audio.play();
      if (this.currentLyric) {
        this.currentLyric.seek();
      }
    },
    error() {
      this.songReady = false;
    },
    ready() {
      this.songReady = true;
      this.savePlayHistory(this.currentSong);
    },
    // 下一首歌曲
    next() {
      if (!this.songReady) {
        return;
      }
      if (this.playlist.length === 1) {
        this.loop();
        return;
      } else {
        let index = this.currentIndex + 1;
        if (index === this.playlist.length) {
          index = 0;
        }
        this.setCurrentIndex(index);
        /*暂停时切歌会播放*/
        if (!this.playing) {
          this.togglePlaying();
        }
      }
      this.songReady = false;
    },
    // 上一首歌曲
    prev() {
      if (!this.songReady) {
        return;
      }
      this.songReady = false;
      let index = this.currentIndex - 1;
      if (index === -1) {
        index = this.playlist.length - 1;
      }
      this.setCurrentIndex(index);
      if (!this.playing) {
        this.togglePlaying();
      }
      this.songReady = false;
    },
    // 离开播放页
    back() {
      this.setFullScreen(false);
      this.currentShow = "cd";
    },
    // 打开播放页
    open() {
      this.setFullScreen(true);
    },
    // 切换播放状态
    togglePlaying() {
      const audio = this.$refs.audio;
      this.setPlayingState(!this.playing); //播放状态取反
      this.playing ? audio.play() : audio.pause();
      if (this.currentLyric) {
        this.currentLyric.togglePlay();
      }
    },
    // 获取歌曲url
    _getSong(id) {
      getSong(id).then(res => {
        this.url = res.data.data[0].url;
      });
    },
    // 获取歌词
    _getLyric(id) {
      if (this.currentLyric) {
        this.currentLyric.stop();
        this.currentLyric = null;
      }
      this.noLyric = false;
      getLyric(id)
        .then(res => {
          this.currentLyric = new Lyric(res.data.lrc.lyric, this.handleLyric);
          if (this.playing) {
            this.currentLyric.play();
            // 歌词重载以后 高亮行设置为 0
            this.currentLineNum = 0;
            this.$refs.lyricList.scrollTo(0, 0, 1000);
          }
        })
        .catch(() => {
          this.currentLyric = null;
          this.noLyric = true;
          this.currentLineNum = 0;
        });
    },
    /*这个回调函数是歌词每一行发生改变时*/
    handleLyric({ lineNum, txt }) {
      this.currentLineNum = lineNum;
      if (lineNum > 5) {
        //保持歌词在屏幕中间
        let lineEl = this.$refs.lyricLine[lineNum - 5];
        this.$refs.lyricList.scrollToElement(lineEl, 1000);
      } else {
        //刚开始的5行之内滚动到顶部
        this.$refs.lyricList.scrollTo(0, 0, 1000);
      }
    },
    ...mapMutations({
      setFullScreen: "SET_FULL_SCREEN",
      setPlayingState: "SET_PLAYING_STATE",
      setCurrentIndex: "SET_CURRENT_INDEX",
      setPlayMode: "SET_PLAY_MODE",
      setPlaylist: "SET_PLAYLIST"
    }),
    ...mapActions(["saveFavoriteList", "deleteFavoriteList", "savePlayHistory"])
  },
  components: {
    ProgressBar,
    ProgressCircle,
    Scroll,
    Playlist
  }
};
</script>

<style lang="scss" scoped>
@import "~common/scss/variable";
@import "~common/scss/mixin";
.player {
  .normal-player {
    position: fixed;
    left: 0;
    right: 0;
    top: 0;
    bottom: 0;
    z-index: 150;
    background: $color-background;
    .background {
      position: absolute;
      left: -50%;
      top: -50%;
      width: 300%;
      height: 300%;
      z-index: -1;
      opacity: 0.6;
      filter: blur(30px);
      .filter {
        position: absolute;
        width: 100%;
        height: 100%;
        background: black;
        opacity: 0.6;
      }
      // .filterR {
      //   position: absolute;
      //   width: 100%;
      //   height: 100%;
      //   background: black;
      //   opacity: 0.4;
      //   &.filterR-enter-active,
      //   &.filterR-leave-active {
      //     transition: all 0.3s;
      //   }
      //   &.filterR-leave-to,
      //   &.filterR-enter {
      //     opacity: 0;
      //   }
      //   &.filterR-leave {
      //     opacity: 0.4;
      //   }
      // }
    }
    .top {
      position: relative;
      margin-bottom: 25px;
      .back {
        position: absolute;
        top: 0;
        left: 6px;
        z-index: 50;
        .fa-angle-down {
          display: block;
          padding: 5px 9px;
          font-size: 35px;
          color: $color-theme-l;
        }
      }
      .title {
        width: 70%;
        margin: 0 auto;
        padding-top: 10px;
        line-height: 20px;
        text-align: center;
        @include no-wrap();
        font-size: $font-size-large;
        font-weight: bold;
        color: $color-text-l;
      }
      .subtitle {
        width: 70%;
        margin: 0 auto;
        line-height: 20px;
        text-align: center;
        @include no-wrap();
        font-size: $font-size-small-x;
        color: $color-text-l;
      }
    }
    // 主页，cd页+歌词页
    .middle {
      display: flex;
      align-items: center;
      position: fixed;
      width: 100%;
      top: 80px;
      bottom: 170px;
      white-space: nowrap;
      font-size: 0;
      // cd页
      .middle-l {
        display: inline-block;
        vertical-align: top;
        position: relative;
        width: 100%;
        height: 0;
        padding-top: 80%;
        &.middleL-enter-active,
        &.middleL-leave-active {
          transition: all 0.3s;
        }
        &.middleL-enter,
        &.middleL-leave-to {
          opacity: 0;
        }
        .cd-wrapper {
          position: absolute;
          left: 10%;
          top: 0;
          width: 80%;
          height: 100%;
          .cd {
            width: 100%;
            height: 100%;
            box-sizing: border-box;
            border: 15px solid rgba(255, 255, 255, 0.1);
            border-radius: 50%;
            &.play {
              animation: rotate 20s linear infinite;
            }
            &.pause {
              animation-play-state: paused;
            }
            .image {
              position: absolute;
              left: 0;
              top: 0;
              width: 100%;
              height: 100%;
              border-radius: 50%;
            }
          }
        }
      }
      // 歌曲栏
      .middle-r {
        position: absolute;
        top: 0;
        vertical-align: top;
        width: 100%;
        height: 100%;
        overflow: hidden;
        &.middleR-enter-active,
        &.middleR-leave-active {
          transition: all 0.2s;
        }
        &.middleR-enter,
        &.middleR-leave-to {
          opacity: 0;
        }
        .lyric-wrapper {
          width: 80%;
          margin: 0 auto;
          overflow: hidden;
          text-align: center;
          .text {
            line-height: 40px;
            color: $color-text-ggg;
            font-size: $font-size-medium;
            &.current {
              color: #fff;
            }
          }
          .no-lyric {
            line-height: 40px;
            margin-top: 60%;
            color: $color-text-ggg;
            font-size: $font-size-medium;
          }
        }
      }
    }
    // 底部播放暂停栏
    .bottom {
      position: absolute;
      bottom: 50px;
      width: 100%;
      .progress-wrapper {
        display: flex;
        align-items: center;
        width: 80%;
        margin: 0px auto;
        padding: 10px 0;
        .time {
          color: $color-text-l;
          font-size: $font-size-small;
          flex: 0 0 30px;
          line-height: 30px;
          width: 30px;
          &.time-l {
            text-align: left;
          }
          &.time-r {
            text-align: right;
            color: $color-text-gg;
          }
        }
        .progress-bar-wrapper {
          flex: 1;
        }
      }
      .operators {
        display: flex;
        align-items: center;
        .icon {
          flex: 1;
          color: $color-theme-l;
          &.disable {
            color: $color-theme;
          }
          i {
            font-size: 30px;
          }
          .mode {
            font-size: 25px;
          }
          &.i-left {
            text-align: right;
          }
          &.i-center {
            padding: 0 20px;
            text-align: center;
            i {
              font-size: 30px;
            }
          }
          &.i-right {
            text-align: left;
          }
          .icon-like {
            color: $color-sub-theme;
          }
        }
      }
    }
    &.normal-enter-active,
    &.normal-leave-active {
      transition: all 0.4s;
      .top,
      .bottom {
        transition: all 0.4s cubic-bezier(0.86, 0.18, 0.82, 1.32);
      }
    }
    &.normal-enter,
    &.normal-leave-to {
      opacity: 0;
    }
  }
  // 迷你播放器
  .mini-player {
    display: flex;
    align-items: center;
    position: fixed;
    left: 0;
    bottom: 0;
    z-index: 180;
    width: 100%;
    height: 60px;
    background: rgba(255, 255, 255, 0.85);
    &.mini-enter-active,
    &.mini-leave-active {
      transition: all 0.4s;
    }
    &.mini-enter,
    &.mini-leave-to {
      opacity: 0;
    }
    .icon {
      flex: 0 0 40px;
      width: 40px;
      padding: 0 10px 0 20px;
      img {
        border-radius: 50%;
        // 旋转特效
        &.play {
          animation: rotate 10s linear infinite;
        }
        &.pause {
          animation-play-state: paused;
        }
      }
    }
    .text {
      display: flex;
      flex-direction: column;
      justify-content: center;
      flex: 1;
      overflow: hidden;
      .name {
        margin-bottom: 2px;
        line-height: 14px;
        @include no-wrap();
        font-size: $font-size-medium;
        color: $color-text;
      }
      .desc {
        @include no-wrap();
        font-size: $font-size-small;
        color: $color-text;
      }
    }
    // 播放列表icon
    .control {
      flex: 0 0 30px;
      width: 30px;
      padding: 0 10px;
      .icon-play-mini,
      .icon-pause-mini,
      .icon-playlist,
      .iconfont {
        font-size: 30px;
        color: $color-theme-d;
      }
      .iconfont {
        position: relative;
        left: -5px;
        top: -3px;
      }
      .fa-play {
        color: $color-theme-d;
        font-size: 14px;
        position: absolute;
        left: 12px;
        top: 8.5px;
      }
      .fa-pause {
        color: $color-theme-d;
        font-size: 12px;
        position: absolute;
        left: 11px;
        top: 10px;
      }
    }
  }
}
@keyframes rotate {
  0% {
    transform: rotate(0);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
