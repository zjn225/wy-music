<template>
<!-- 整体用transition定义过渡动画 -->
<transition name="slide" mode="out-in">  
  <div class="music-list">
    <!-- 顶部 -->
    <div class="header" ref="header">
      <div class="back" @click="back">
        <i class="fa fa-angle-left"></i>
      </div>
      <div class="text">
        <h1 class="title">{{headerTitle}}</h1>  
      </div>
    </div>
    <!-- 整体用scroll包裹进行滚动 -->
    <scroll class="list"
    @scroll="scroll"
    :probe-type="probeType"
    :listen-scroll="listenScroll"
    :data="listDetail"
    ref="list">
      <div class="music-list-wrapper">  
        <!-- 歌单背景图 -->
        <div class="bg-image" :style="bgStyle" ref="bgImage">
          <div class="filter"></div>
          <!-- 歌单名字和一些icon、播放次数 -->
          <div class="text">
            <h2 class="list-title">{{title}}</h2>
            <p class="play-count" v-if="playCount">
              <i class="fa fa-headphones"></i>
              {{playCount}}
            </p>
          </div>
        </div>
        <!-- 列表 -->
        <div class="song-list-wrapper">
          <div class="sequence-play" v-show="listDetail.length" @click="sequence">
            <i class="iconfont icon-bofangicon"></i>
            <span class="text">播放全部</span>
            <span class="count">(共{{listDetail.length}}首)</span>
          </div>
          <song-list @select="selectItem" :songs="listDetail"></song-list>
        </div>
      </div>
    </scroll>
    <!-- loading组件 -->
    <div v-show="!listDetail.length" class="loading-content">
      <loading></loading>
    </div>
  </div>
</transition>
</template>

<script>
import Scroll from "base/scroll/scroll";
import SongList from "base/song-list/song-list";
import Loading from "base/loading/loading";
import { mapGetters, mapActions } from "vuex";
import { getRecommendListDetail } from "api/recommend";
import { ERR_OK } from "common/js/config";
import { createRecommendListSong } from "common/js/song";
import { playlistMixin } from "common/js/mixin";

const RESERVED_HEIGHT = 44;

export default {
  mixins: [playlistMixin],
  data() {
    return {
      listDetail: [],
      scrollY: 0,
      headerTitle: "歌单"
    };
  },
  created() {
    this._getRecommendListDetail(this.musicList.id);
    this.probeType = 3;
    this.listenScroll = true;
  },
  mounted() {
    //背景图片高度
    this.imageHeight = this.$refs.bgImage.clientHeight;  
    //滑过一屏之后的遮罩层的滚动距离，加44是因为滑动的时候顶部始终要留44px的高
    this.minTranslateY = -this.imageHeight + RESERVED_HEIGHT; 
  },
  computed: {
    playCount() {
      if (!this.musicList.playCount) {
        return;
      }
      if (this.musicList.playCount < 1e5) {
        return Math.floor(this.musicList.playCount);
      } else {
        return Math.floor(this.musicList.playCount / 10000) + "万";
      }
    },
    bgStyle() {
      return `background-image: url(${this.musicList.picUrl})`;
    },
    title() {
      return this.musicList.name;
    },
    ...mapGetters(["musicList"])
  },
  methods: {
    // mixin方法
    handlePlaylist(playlist) {
      const bottom = playlist.length > 0 ? "60px" : "";
      this.$refs.list.$el.style.bottom = bottom;
      this.$refs.list.refresh();
    },
    // 获取歌单数据
    _getRecommendListDetail(id) {
      // vuex刷新后数据丢失，返回recommend页
      if (!id) {
        this.$router.push("/recommend");
        return;
      }
      getRecommendListDetail(id).then(res => {
        if (res.status === ERR_OK) {
          this.listDetail = res.data.playlist.tracks.map(item => {
            return createRecommendListSong(item);  //格式化歌单数据
          });
        } else {
          console.error("getRecommendListDetail 获取失败！");
        }
      });
    },
    // 点击列表歌曲
    selectItem(item, index) {
      this.selectPlay({
        list: this.listDetail,
        index: index
      });
    },
    // 监听滚动，pos来自于scroll组件
    scroll(pos) {
      this.scrollY = pos.y;
    },
    back() {
      this.$router.back();
    },
    // 播放全部
    sequence() {
      let list = this.listDetail;
      this.sequencePlay({
        list: list
      });
    },
    ...mapActions(["selectPlay", "sequencePlay"])
  },
  watch: {
    scrollY(newY) {
      const percent = Math.abs(newY / this.imageHeight); //当前滚动高度占背景图片的百分比
      if (newY < this.minTranslateY + 20) {   //滚动到背景图片之下距离时
        this.headerTitle = this.musicList.name;
      } else {
        this.headerTitle = "歌单";
      }
      if (newY < 0) { //只要一向下滚动
        this.$refs.header.style.background = `rgba(212, 68, 57, ${percent})`;
      } else {
        this.$refs.header.style.background = `rgba(212, 68, 57, 0)`;
      }
    }
  },
  components: {
    SongList,
    Scroll,
    Loading
  }
};
</script>

<style lang="scss" scoped>
@import "~common/scss/variable";
@import "~common/scss/mixin";
.slide-enter-active,
.slide-leave-active {
  transition: all 0.2s;
}
.slide-enter,
.slide-leave-to {
  transform: translate3d(50%, 0, 0);
  opacity: 0;
}

.music-list {
  // position: fixed;
  // z-index: 100;
  // top: 0;
  // left: 0;
  // bottom: 0;
  // right: 0;
  background: $color-background;
  // 顶部吸顶栏
  .header {
    position: fixed;
    top: 0;
    width: 100%;
    height: 44px;
    color: #fff;
    z-index: 100;
    .back {
      position: absolute;
      top: 0;
      left: 4px;
      .fa-angle-left {
        padding: 5px 10px;
        font-size: 30px;
      }
    }
    .text {
      position: absolute;
      left: 38px;
      line-height: 44px;
      font-size: $font-size-medium-x;
      @include no-wrap();
    }
  }
  .list {
    position: fixed;
    top: 0;
    bottom: 0;
    width: 100%;
    background: $color-background;
    .music-list-wrapper {
      .bg-image {
        position: relative;
        width: 100%;
        height: 0;
        padding-top: 75%;
        transform-origin: top;
        background-size: cover;
        background-position: 0 30%;
        .filter {
          position: absolute;
          top: 0;
          left: 0;
          width: 100%;
          height: 100%;
          background: black;
          opacity: 0.2;
        }
        .text {
          position: absolute;
          width: 80%;
          height: 40px;
          bottom: 50px;
          left: 15px;
          color: #fff;
          .play-count {
            position: absolute;
            bottom: -16px;
            font-size: $font-size-small;
          }
          .list-title {
            position: absolute;
            bottom: 0;
            font-size: $font-size-medium-x;
            line-height: 18px;
            font-weight: bold;
            letter-spacing: 1px;
          }
        }
      }
      .song-list-wrapper {
        padding: 41px 0 20px 0;
        border-radius: 10px;
        position: relative;
        top: -20px;
        background: $color-background;
        .sequence-play {
          position: absolute;
          // left: 8;
          top: 0px;
          display: flex;
          align-items: center;
          width: 100%;
          height: 40px;
          padding-left: 16px;
          border-bottom: 1px solid rgb(228, 228, 228);
          .iconfont {
            font-size: 18px;
            color: $color-text;
            padding-right: 14px;
          }
          .text {
            font-size: $font-size-medium-x;
          }
          .count {
            font-size: $font-size-medium;
            color: $color-text-g;
          }
        }
      }
      // position: fixed;
      // top: 0;
      // bottom: 0;
      // width: 100%;
      // background: $color-background;
      // height: 100%;
      // z-index: 20;
      // overflow: hidden;
      // padding: 5px 0 20px 0;
    }
  }
  .loading-content {
    position: fixed;
    width: 100%;
    top: 70%;
    transform: translateY(-50%);
  }
}
</style>
