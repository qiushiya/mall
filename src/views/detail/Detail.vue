<template>
  <div id="detail">
    <detail-nav-bar class="detail-nav" @titleClick="titleClick" ref="nav"/>
    <scroll class="content" ref="scroll"
            @scroll="contentScroll" :probe-type="3">
      <!-- 属性：topImages 传入值：top-images -->
      <detail-swiper :top-images="topImages"/>
      <detail-base-info :goods="goods"/>
      <detail-shop-info :shop="shop"/>
      <detail-goods-info :detail-info="detailInfo" @imageLoad="imageLoad"/>
      <detail-param-info :param-info="paramInfo" ref="params"/>
      <detail-comment-info :comment-info="commentInfo" ref="comment"/>
      <goods-list :goods="recommends" ref="recommend"/>
    </scroll>
    <detail-bottom-bar @addCart="addToCart"/>
    <back-top @click.native="backClick" v-show="isShowBackTop"/>
<!--    <toast :message="message" :show="show"/>-->
  </div>
</template>

<script>
import DetailNavBar from "./childComps/DetailNavBar";
import DetailSwiper from "./childComps/DetailSwiper";
import DetailBaseInfo from "./childComps/DetailBaseInfo";
import DetailShopInfo from "./childComps/DetailShopInfo";
import DetailGoodsInfo from "./childComps/DetailGoodsInfo";
import DetailParamInfo from "./childComps/DetailParamInfo";
import DetailCommentInfo from "./childComps/DetailCommentInfo";
import DetailBottomBar from "./childComps/DetailBottomBar";

import Scroll from "components/common/scroll/Scroll";
import GoodsList from "components/content/goods/GoodsList";
// import Toast from "components/common/toast/Toast";

import {getDetail, Goods, Shop, GoodsParam, getRecommend} from 'network/detail';
import {debounce} from "common/utils";
import {itemListenerMixin, backTopMixin} from "common/mixin";

import {mapActions} from 'vuex'

export default {
  name: "Detail",
  components: {
    DetailNavBar,
    DetailSwiper,
    DetailBaseInfo,
    DetailShopInfo,
    DetailGoodsInfo,
    DetailParamInfo,
    DetailCommentInfo,
    DetailBottomBar,
    Scroll,
    GoodsList,
    // Toast
  },
  mixins: [itemListenerMixin, backTopMixin],
  data() {
    return {
      iid: null,
      topImages: [],
      goods: {},
      shop: {},
      detailInfo: {},
      paramInfo: {},
      commentInfo: {},
      recommends: [],
      // itemImgListener: null
      themeTopY: [],
      currentIndex: 0,
      // message: '',
      // show: false
    }
  },
  created() {
    // 1. 保存传入的iid
    this.iid = this.$route.params.iid;

    // 2. 根据iid请求详情数据
    getDetail(this.iid).then(res => {
      // console.log(res);
      // 1. 获取顶部的图片轮播数据
      const data = res.result
      this.topImages = data.itemInfo.topImages;

      // 2. 获取商品信息
      this.goods = new Goods(data.itemInfo, data.columns, data.shopInfo.services)

      // 3. 创建店铺信息的对象
      this.shop = new Shop(data.shopInfo)

      // 4. 保存商品的详细数据
      this.detailInfo = data.detailInfo;

      // 5. 获取参数的信息
      this.paramInfo = new GoodsParam(data.itemParams.info, data.itemParams.rule)

      // 6. 取出评论的信息
      if (data.rate.cRate !== 0) {
        this.commentInfo = data.rate.list[0];
      }

      // 第一次获取，值不对
      // 值不对的原因：this.$refs.params.$el压根没有渲染
      // this.themeTopY = []
      //
      // this.themeTopY.push(0)
      // this.themeTopY.push(this.$refs.params.$el.offsetTop)
      // this.themeTopY.push(this.$refs.comment.$el.offsetTop)
      // this.themeTopY.push(this.$refs.recommend.$el.offsetTop)
      //
      // console.log(this.themeTopY);

      // $nextTick等前面东西渲染完成后，再执行里面的东西
      // this.$nextTick(() => {
      //   // 2. 第二次获取：值不对
      //   // 值不对的原因，图片没有计算在内
      //   // 根据最新的数据，对应的DOM是已经被渲染出来
      //   // 但是图片依然是没有加载完(目前获取到offsetTop不包含其中的图片)
      //   // offsetTop值不对的时候，都是因为图片的问题
      //   this.themeTopY = []
      //
      //   this.themeTopY.push(0)
      //   this.themeTopY.push(this.$refs.params.$el.offsetTop)
      //   this.themeTopY.push(this.$refs.comment.$el.offsetTop)
      //   this.themeTopY.push(this.$refs.recommend.$el.offsetTop)
      //
      //   console.log(this.themeTopY);
      // })
    })

    // 3. 请求推荐数据
    getRecommend().then(res => {
      // console.log(res);
      this.recommends = res.data.list
    })
  },
  mounted() {
    // console.log('mounted');
  },
  destroyed() {
    this.$bus.$off('itemImageLoad', this.itemImgListener)
  },
  methods: {
    ...mapActions(['addCart']),
    imageLoad() {
      this.$refs.scroll.refresh()

      this.themeTopY = []

      this.themeTopY.push(0)
      this.themeTopY.push(this.$refs.params.$el.offsetTop)
      this.themeTopY.push(this.$refs.comment.$el.offsetTop)
      this.themeTopY.push(this.$refs.recommend.$el.offsetTop)
      this.themeTopY.push(Number.MAX_VALUE)

      console.log(this.themeTopY);
    },
    titleClick(index) {
      // console.log(index);
      this.$refs.scroll.scrollTo(0, -this.themeTopY[index], 200)
    },
    contentScroll(position) {
      // 1. 获取y值
      const positionY = -position.y

      // 2. positionY与主题中的值进行对比
      // [0, 3221, 4066, 4356]
      // console.log(Number.MAX_VALUE);

      // positionY 在 0 和 3221 之间，index = 0
      // positionY 在 =3221 和 4066 之间，index = 1
      // positionY 在 4066 和 4356 之间，index = 2
      // positionY 在 4356 和 非常大的值之间，index = 3

      // positionY 大于等于 4356，index = 3
      let length = this.themeTopY.length;
      for (let i = 0; i < length - 1; i++) {
        // console.log(i); // i是字符串类型
        // if (positionY > this.themeTopY[i] && positionY < this.themeTopY[i+1]) {
        //   console.log(i);
        // }

        if (this.currentIndex !== i && (positionY >= this.themeTopY[i] && positionY < this.themeTopY[i+1])) {
          this.currentIndex = i;
          // console.log(this.currentIndex);
          this.$refs.nav.currentIndex = this.currentIndex;
        }

        // if (this.currentIndex !== i && (i < length - 1 && positionY >= this.themeTopY[i] && positionY < this.themeTopY[i+1]) || (i === length - 1 && positionY >= this.themeTopY[i])) {
        //   this.currentIndex = i;
        //   console.log(this.currentIndex);
        //   this.$refs.nav.currentIndex = this.currentIndex;
        // }
      }

      // 3. 是否显示回到顶部
      this.isShowBackTop = (-position.y) > 1000
    },
    addToCart() {
      // 1. 获取购物车需要展示的信息
      const product = {}
      product.image = this.topImages[0];
      product.title = this.goods.title;
      product.desc = this.goods.desc;
      product.price = this.goods.realPrice;
      product.iid = this.iid;

      // 2. 将商品添加到购物车里(1. Promise  2. mapActions)
      // this.$store.cartList.push(product)
      // this.$store.commit('addCart', product)

      this.addCart(product).then(res => {
        // this.show = true;
        // this.message = res;
        //
        // setTimeout(() => {
        //   this.show = false;
        //   this.message = '';
        // }, 1500)

        // console.log(this.$toast);
        this.$toast.show(res)
      })

      // this.$store.dispatch('addCart', product).then(res => {
      //   console.log(res);
      // })
    }
  }
}
</script>

<style scoped>
  #detail {
    position: relative;
    z-index: 9;
    background-color: #ffffff;
    height: 100vh;
  }

  .detail-nav {
    position: relative;
    z-index: 9;
    background-color: #ffffff;
  }

  .content {
    height: calc(100% - 44px - 58px);
    overflow: hidden;
  }
</style>
