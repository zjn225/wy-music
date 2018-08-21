<template>
  <div class="singer" ref="singer">
    <list-view :data="this.singers" @select="selectSinger" ref="list"></list-view>
    <!-- 注意，这个router-view是渲染子路由的. -->
    <router-view></router-view>
  </div>
</template>

<script>
import Singer from "common/js/singer";
import ListView from "base/listview/listview";
import { getSingers } from "src/api/singer";
import { playlistMixin } from "common/js/mixin";
import { mapMutations } from "vuex";

const pinyin = require("pinyin");
const HOT_NAME = "热门";
const HOT_SINGERS_NUM = 10;

export default {
  mixins: [playlistMixin],
  data() {
    return {
      singers: []
    };
  },
  created() {
    this._getSingers();
  },
  methods: {
    handlePlaylist(playlist) {
      const bottom = playlist.length > 0 ? "60px" : "";
      this.$refs.singer.style.bottom = bottom;
      this.$refs.list.refresh();
    },
    selectSinger(singer) {
      this.$router.push({
        path: `/singer/${singer.id}`
      });
      this.setSinger(singer);
    },

    // 获取歌手数据
    _getSingers() {
      getSingers().then(res => {
        let s = res.data.artists;
        s.map(item => {
          let py = pinyin(item.name[0], {
            style: pinyin.STYLE_FIRST_LETTER
          });
          item.initial = py[0][0].toUpperCase(); //把initial属性改为拼音首字母小写
        });
        this.singers = this._normalizeSinger(s); //格式化歌手数据
      });
    },

    //  格式化歌手数据,按字母排列
    _normalizeSinger(list) {
      // 热门歌单
      let map = {
        hot: {
          title: HOT_NAME, //title就是侧栏显示的东西
          items: []
        }
      };
      /*获得map  热门+A-Z*/
      list.forEach((item, index) => {
        // 获取的数据前十个是热门歌手
        if (index < HOT_SINGERS_NUM) {
          map.hot.items.push(
            new Singer({
              id: item.id,
              name: item.name,
              avatar: item.img1v1Url,
              aliaName: item.alias.join(" / ")
            })
          );
        }
        // 获取姓氏字母
        const key = item.initial;
        //创建map里面的key对象，因为一开始只有hot对象，有很多key
        if (!map[key]) {
          //这里的map[key]就是A-Z包含的歌手集合
          map[key] = {
            title: key, //title就是侧栏显示的东西，这里为A-Z
            items: []
          };
        }
        map[key].items.push(
          new Singer({
            id: item.id,
            name: item.name,
            avatar: item.img1v1Url,
            aliaName: item.alias[0]
          })
        );
      });
      // 进一步格式化，将非热门加入ret数组，热门加入hot数组
      let hot = [];
      let ret = [];
      for (const key in map) {
        let val = map[key];
        if (val.title.match(/[A-Z]/)) {
          ret.push(val);
        } else if (val.title === HOT_NAME) {
          hot.push(val);
        }
      }
      // 非热门数组按照字母字典序排序
      ret.sort((a, b) => {
        return a.title.charCodeAt(0) - b.title.charCodeAt(0);
      });
      return hot.concat(ret);
    },
    ...mapMutations({
      setSinger: "SET_SINGER"
    })
  },
  mounted() {
    this._getSingers();
  },
  components: {
    ListView
  }
};
</script>

<style lang="scss" scoped>
.singer {
  position: fixed;
  top: 88px;
  bottom: 0;
  width: 100%;
}
</style>
