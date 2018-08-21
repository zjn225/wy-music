<template>
  <scroll
  class="listview"
  ref="listview"
  :listenScroll="listenScroll"
  @scroll="scroll"
  :probeType="probeType"
  >
    <ul>
      <!-- 存放每个单个字母或热门组成的group -->
      <li v-for="group in data" class="list-group" :key="group.id" ref="listGroup">
        <!-- 灰色顶部条，显示姓氏或热门 -->
        <h2 class="list-group-title">{{ group.title }}</h2>
        <ul>
          <!-- 存放每个group里的子项目 -->
          <li @click="selectItem(item)" v-for="item in group.items" class="list-group-item" :key="item.id">
            <img v-lazy="item.avatar" class="avatar">
            <span class="name">{{ item.name }}</span>
          </li>
        </ul>
      </li>
    </ul>
    <!-- 侧栏 -->
    <div class="list-shortcut" @touchstart="onShortcutStart"
    @touchmove.stop.prevent="onShortcutMove">
      <ul>
        <!-- 这里设置了一下自定义属性data-index -->
        <li v-for="(item, index) in shortcutList"
        class="item"
        :data-index="index"  
        :key="item.id"
        :class="{'current': currentIndex === index}"
        >
          {{ item }}
        </li>
      </ul>
    </div>
  </scroll>
</template>

<script>
import Scroll from "base/scroll/scroll";
import { getData } from "common/js/dom";

const ANCHOR_HEIGHT = 20; //每个字母的高度
// const TITLE_HEIGHT = 30

export default {
  props: {
    data: {
      type: Array
    }
  },
  data() {
    return {
      scrollY: -1, ////左右联动的关键变量，滚动条位置
      currentIndex: 0 //左右联动的关键变量，对应着右侧高亮的字母！
    };
  },
  created() {
    this.touch = {};
    //存储点击或滑动侧边栏时的相关数据
    this.listenScroll = true;
    this.listHeight = [];
    this.probeType = 3;
  },
  methods: {
    selectItem(item) {
      this.$emit("select", item);
    },
    refresh() {
      this.$refs.listview.refresh();
    },
    // 得到scrollY值,必须是滑动页面，而不包括侧边栏
    scroll(pos) {
      this.scrollY = pos.y;
    },
    // 点击侧栏字母时或者滑动时，顾名思义start
    onShortcutStart(e) {
      let anchorIndex = getData(e.target, "index"); //取到当前index
      let firstTouch = e.touches[0];
      this.touch.y1 = firstTouch.pageY;
      this.touch.anchorIndex = anchorIndex;
      this._scrollTo(anchorIndex);
    },
    // 滑动侧栏字母时
    onShortcutMove(e) {
      let firshTouch = e.touches[0];
      this.touch.y2 = firshTouch.pageY;
      let delta = ((this.touch.y2 - this.touch.y1) / ANCHOR_HEIGHT) | 0; // |0是向下取整，得到偏移了几个锚点
      let anchorIndex = parseInt(this.touch.anchorIndex) + delta;
      this._scrollTo(anchorIndex);
    },
    // 遍历每个listGroup，得到每个listGroup的高度，组成一个数组
    _calculateHeight() {
      this.listHeight = [];
      const list = this.$refs.listGroup;
      let height = 0;
      this.listHeight.push(height);
      for (let i = 0; i < list.length; i++) {
        let item = list[i];
        height += item.clientHeight;
        this.listHeight.push(height);
      }
    },
    // ※※点击或滑动侧边栏跳转到对应的listGroup
    _scrollTo(index) {
      if (!index) {
        return;
      }
      if (index < 0) {
        index = 0;
      } else if (index > this.listHeight.length - 2) {
        index = this.listHeight.length - 2;
      }
      //手动触发了scroll事件，这样才能和页面滚动事件联动
      this.scrollY = -this.listHeight[index];   //这句删了1、点击字母不会高亮，2、懒加载无效
      this.$refs.listview.scrollToElement(this.$refs.listGroup[index], 0); //scroll.vue里的方法，页面滚动到相应位置
    }
  },
  watch: {
    data() {
      setTimeout(() => {
        this._calculateHeight();
      }, 20);
    },
     // ※※滑动页面时自动对应侧边栏 + 高亮字母
    scrollY(newY) {
      const listHeight = this.listHeight;
      console.log(newY)
      // 滑动到了顶部
      if (newY > 0) {
        this.currentIndex = 0;
        return;
      }
      for (let i = 0; i < listHeight.length - 1; i++) {
        let height1 = listHeight[i];
        let height2 = listHeight[i + 1];
        if (-newY >= height1 && -newY < height2) { //向下滑动newY就是负的，取反才能和hight对比
          this.currentIndex = i;
          return;
        }
      }
      this.currentIndex = listHeight.length - 2;
    }
  },
  computed: {
    // 姓氏字母
    shortcutList() {
      return this.data.map(group => {
        return group.title.substr(0, 1);
      });
    }
    // fixedTitle() {
    //   if (this.scrollY > 0) {
    //     return "";
    //   }
    //   return this.data[this.currentIndex]
    //     ? this.data[this.currentIndex].title
    //     : "";
    // }
  },
  components: {
    Scroll
  }
};
</script>

<style lang="scss" scoped>
@import "~common/scss/variable";
.listview {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden; //滚动到下面时，要隐藏！很重要
  background: $color-background;
  .list-group {
    // padding: 10px 0;
    .list-group-title {
      height: 20px;
      line-height: 20px;
      padding-left: 12px;
      margin-bottom: 10px;
      font-size: $font-size-small;
      color: #fff;
      background: rgba(0, 0, 0, 0.1);
    }
    .list-group-item {
      display: flex;
      align-items: center;
      padding: 5px 0;
      margin: 0 5px;
      border-bottom: 1px solid rgb(228, 228, 228);
      &:last-child {
        border: none;
        margin-bottom: 10px;
      }
      .avatar {
        width: 50px;
        height: 50px;
        border-radius: 3px;
      }
      .name {
        margin-left: 20px;
        color: $color-text;
        font-size: $font-size-medium;
      }
    }
  }
  .list-shortcut {
    position: fixed;
    // z-index: 30;
    right: 3px;
    top: 50%;
    transform: translateY(-47%);
    width: 20px;
    padding: 200px 0;
    border-radius: 3px;
    text-align: center;
    font-family: Helvetica;
    .item {
      padding: 5px 5px;
      // line-height: 1;
      color: $color-text-g;
      font-size: $font-size-small;
      font-weight: bold;
      &.current {
        color: $color-theme;
      }
    }
  }
  // .list-fixed {
  //   position: absolute;
  //   top: 0;
  //   left: 0;
  //   width: 100%;
  //   .fixed-title {
  //     height: 20px;
  //     line-height: 20px;
  //     padding-left: 20px;
  //     font-size: $font-size-small;
  //     color: $color-text-l;
  //     background: rgba(0, 0, 0, 0.1);
  //   }
  // }
  // .loading-container {
  //   position: absolute;
  //   width: 100%;
  //   top: 50%;
  //   transform: translateY(-50%);
  // }
}
</style>
