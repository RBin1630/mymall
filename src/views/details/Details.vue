<template>
  <div id="details">
    <details-nav-bar @titleClick="titleClick" class="details-nav" ref="nav"></details-nav-bar>
    <scroll class="content" ref="scroll" @scroll="contentScroll" :probeType="3">
      <details-swiper :topImages="topImages"></details-swiper>

      <details-base-info :detailsData="detailsData"></details-base-info>

      <details-shop-info :shopInfo="shopInfo"></details-shop-info>

      <details-goods-info :detailsInfo="detailsInfo" @imgLoad="imgLoad"></details-goods-info>

      <details-params-info ref="params" :itemParams="itemParams" :rate="rate"></details-params-info>

      <details-comment-info ref="comment" :rate="rate"></details-comment-info>

      <goods-list ref="recommend" class="recommend" :list="recommend"></goods-list>
    </scroll>
    <!-- 返回顶部按钮 -->
    <back-top @click.native="topClick" v-show="isShowTop"></back-top>

    <!-- 底部功能模块 -->
    <details-bottom-bar @addToCart="addToCart"></details-bottom-bar>
  </div>
</template>

<script>
import { mapActions } from 'vuex';

import detailsNavBar from "./childCpn/detailsNavBar";
import detailsSwiper from "./childCpn/detailsSwiper";
import detailsBaseInfo from "./childCpn/detailsBaseInfo";
import detailsShopInfo from "./childCpn/detailsShopInfo";
import detailsGoodsInfo from './childCpn/detailsGoodsInfo';
import detailsParamsInfo from './childCpn/detailsParamsInfo';
import detailsCommentInfo from './childCpn/detailsCommentInfo';
import detailsBottomBar from './childCpn/detailsBottomBar';

import Scroll from "components/common/scroll/Scroll";
import GoodsList from 'components/content/goods/GoodsList';

// import {debounce} from 'common/utils';
import {imgLoadedMixin, backTopMixin} from 'common/mixin';

import { getDetailsData, detailGoods, shopInfo, getRecommend } from "network/details";

export default {
  name: "Details",
  mixins: [imgLoadedMixin, backTopMixin],
  components: {
    Scroll,
    detailsNavBar,
    detailsSwiper,
    detailsBaseInfo,
    detailsShopInfo,
    detailsGoodsInfo,
    detailsParamsInfo,
    detailsCommentInfo,
    GoodsList,
    detailsBottomBar,
  },
  data() {
    return {
      iid: null,
      // 保存轮播图的数据
      topImages: [],
      detailsData: {},
      shopInfo: {},
      // 商品详细数据
      detailsInfo: {},
      // 商品尺码数据
      itemParams: {},
      // 评论数据
      rate: {},
      // 详情页推荐数据
      recommend: [],
      // 保存每个模块的位置信息
      viewOffsetTops: [],
      // 保存导航栏点击后的index值
      currentIndex: 0,
    };
  },
  created() {
    // 保存传入的iid
    this.iid = this.$route.params.iid;

    // 获取详情页数据
    getDetailsData(this.iid).then((res) => {
      console.log(res.result);
      const data = res.result;
      // 保存详情页的轮播图图片
      this.topImages = data.itemInfo.topImages;
      // 把整合好的数据保存到detailsData中
      // console.log(new detailGoods(data.itemInfo, data.columns, data.shopInfo));
      this.detailsData = new detailGoods(
        data.itemInfo,
        data.columns,
        data.shopInfo
      );
      // console.log(this.detailsData);
      this.shopInfo = new shopInfo(data.shopInfo);

      // 保存商品详情数据
      this.detailsInfo = data.detailInfo;
      
      // 保存商品尺码数据
      this.itemParams = data.itemParams;
      // console.log(this.itemParams);

      // 保存评论数据
      this.rate = data.rate;
      // console.log(this.rate);

    });

    // 获取详情页推荐数据
    getRecommend().then(res => {
      // console.log(res.data.list);
      this.recommend = res.data.list;
    })
  },

  destroyed() {
    // 当页面被销毁时 我们让这个事件不再继续监听
    this.$bus.$off('itemImgLoaded', this.imgLoaded);
  },
  methods: {
    ...mapActions({
      cartAdd: 'addToCart',
    }),

    // 详情页商品的图片加载传过来的自定义事件的方法
    imgLoad() {
      this.$refs.scroll.refresh();

      this.viewOffsetTops.push(0);
      this.viewOffsetTops.push(this.$refs.params.$el.offsetTop);
      this.viewOffsetTops.push(this.$refs.comment.$el.offsetTop);
      this.viewOffsetTops.push(this.$refs.recommend.$el.offsetTop);
    },
    titleClick(index) {
      // console.log(index);
      // console.log(this.viewOffsetTops);
      this.$refs.scroll.scrollTo(0, -this.viewOffsetTops[index], 500);
    },
    contentScroll(position) {
      // 保存当前位置的y值
      const positionY = -position.y
      // console.log(positionY);
      // [0, 2897, 3911, 4134]
      for(let i = 0; i < this.viewOffsetTops.length; i++) {
        if ((i < this.viewOffsetTops.length - 1 && positionY >= this.viewOffsetTops[i] && positionY < this.viewOffsetTops[i+1] || (i === this.viewOffsetTops.length - 1 && positionY >= this.viewOffsetTops[i]))) {
          this.currentIndex = i;
          this.$refs.nav.currentIndex = this.currentIndex;
        }
      };
      // 当拉到1000的位置就把isShowTop设置为true
      this.isShowTop = (-position.y) > 1000;
    },
    topClick() {
      // 实现点击回到顶部
      this.$refs.scroll.scrollTo(0, 0, 500);
    },
    addToCart() {
      // 将商品添加到购物车
      // 把购物车需要展示的数据保存到一个对象中
      const cartInfo = {};
      cartInfo.iid = this.iid;
      cartInfo.img = this.topImages[0];
      cartInfo.title = this.detailsData.title;
      cartInfo.desc = this.detailsInfo.desc;
      cartInfo.realPrice = this.detailsData.realPrice;
      // console.log(cartInfo);
      // 分发到actions
      // this.$store.dispatch('addToCart', cartInfo).then(res => {
      //   // actions 返回一个promise实例 告诉我们该操作已经成功
      //   // 操作成功后要显示一个弹窗
      //   console.log(res);
      // })
      this.cartAdd(cartInfo).then(res => {
        // console.log(this.$toast);
        this.$toast.showToast(res);
      })
    }
  },
};
</script>

<style scoped>
#details {
  position: relative;
  height: 100vh;
}
.details-nav {
  position: relative;
  z-index: 99;
  background-color: #fff;
}
.content {
  position: absolute;
  top: 44px;
  left: 0;
  right: 0;
  bottom: 49px;
  z-index: 9;
  background-color: #fff;
}
.recommend {
  background-color: #fff;
}
</style>