<template>
  <div ref="wrapper" class="KScroll-wrapper">
    <div
      :style="{
        height: (pullDownY / 2 - 5 < 0 ? 0 : pullDownY / 2 - 5) + 'px'
      }"
      class="pullDown-loading"
    >
      <img src="/images/loading.gif" alt="" />
    </div>
    <div
      ref="scroll"
      :style="{
        transform: `translate(0px, ${pullDownY}px) scale(1) translateZ(0px)`
      }"
      class="KScroll-content"
    >
      <slot></slot>
      <div class="pullup-loading">
        <template v-if="noMoreData">
          {{ noMoreDataText }}
        </template>
        <template v-else-if="pullingUp">
          <span class="loading-spinners">
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
            <i class="loading-spinner"></i>
          </span>
        </template>
        <template v-else>
          加载完成
        </template>
      </div>
    </div>
  </div>
</template>

<script>
import { debounce } from "lodash";

const DEFAULT_OPTIONS = {
  pullUp: true,
  pullDown: true,
  threshold: 50
};

export default {
  name: "KScroll",
  props: {
    noMoreDataText: {
      type: String,
      default: "已经全部加载完毕"
    },

    options: {
      type: Object,
      default: () => ({})
    },

    scrollEvents: {
      type: Array,
      default: () => []
    }
  },
  data() {
    return {
      // 是否正处于
      pullingUp: false,
      pullingDown: false,

      // 没有更多数据了
      noMoreData: false,

      pullDownY: 0
    };
  },
  computed: {
    relOptions() {
      return Object.assign({}, DEFAULT_OPTIONS, this.options);
    }
  },
  mounted() {
    this.initPullUpEvent();
    this.initPullDownEvent();
    this.initScrollNativeEvent();
  },
  beforeDestroy() {
    this.removePullDownEvent();
    this.removePullUpEvent();
    this.removeScrollNativeEvent();
    if (this._scrollTimer) {
      clearInterval(this._scrollTimer);
      this._scrollTimer = null;
    }
  },
  methods: {
    _screenY(event) {
      return event.touches[0].screenY;
    },

    _shouldPullToRefresh() {
      return !window.scrollY;
    },

    forceUpdate(notData) {
      if (this.pullingUp) {
        this.pullingUp = false;
        if (notData) {
          this.noMoreData = true;
          this.removePullUpEvent();
        }
      } else if (this.pullingDown) {
        this.scrollTo(0);
        this.pullDownY = 0;
        this.pullingDown = false;
        document.documentElement.style.overflow = "visible";
      }
    },

    initPullDownEvent() {
      const { pullDown } = this.relOptions;
      if (!pullDown) {
        return;
      }
      const { wrapper, scroll } = this.$refs;
      wrapper.addEventListener("touchstart", event => {
        if (this._shouldPullToRefresh()) {
          this._pullStartY = this._screenY(event);
        }
      });
      wrapper.addEventListener("touchmove", e => {});
      wrapper.addEventListener("touchend", () => {});
    },

    removePullDownEvent() {
      const { wrapper } = this.$refs;
      wrapper.removeEventListener("touchstart", () => {});
      wrapper.removeEventListener("touchmove", () => {});
      wrapper.removeEventListener("touchend", () => {});
    },

    initPullUpEvent() {
      const { pullUp } = this.relOptions;
      if (!pullUp) {
        return;
      }
      window.addEventListener("scroll", this._pullUpHandler);
    },

    removePullUpEvent() {
      window.removeEventListener("scroll", this._pullUpHandler);
    },

    initScrollNativeEvent() {
      // 派发scroll相关事件
      if (this.scrollEvents.includes("scroll-end")) {
        this._scrollHandler = debounce(() => {
          this.$emit("scroll-end", { x: 0, y: window.scrollY });
        }, 300);
        window.addEventListener("scroll", this._scrollHandler);
      }
    },

    removeScrollNativeEvent() {
      if (this._scrollHandler) {
        window.removeEventListener("scroll", this._scrollHandler);
        this._scrollHandler = null;
      }
    },

    _pullUpHandler() {
      if (this.pullingUp) {
        return;
      }
      const { scrollY, innerHeight } = window;
      const { scrollHeight } = document.body;
      if (innerHeight + scrollY >= scrollHeight - 10) {
        this.pullingUp = true;
        this.$emit("pulling-up");
      }
    },

    scrollTo(y, transition = 0) {
      // transition 单位ms
      let { scrollY } = window;
      if (y === scrollY) {
        return;
      }
      if (transition) {
        let pre = 0;
        if (y > scrollY) {
          pre = ((y - scrollY) / transition) * 10;
        } else {
          pre = ((scrollY - y) / transition) * 10;
        }
        this._scrollTimer = setInterval(() => {
          if (y > scrollY) {
            // 向下滚动
            scrollY += pre;
            scrollY = Math.min(y, scrollY);
          } else {
            // 向上滚动
            scrollY -= pre;
            scrollY = Math.max(y, scrollY);
          }
          window.scrollTo(0, scrollY);
          if (scrollY === y) {
            clearInterval(this._scrollTimer);
            this._scrollTimer = null;
          }
        }, 10);
      } else {
        window.scrollTo(0, y);
      }
    }
  }
};
</script>

<style scoped lang="scss">
.KScroll-wrapper {
  position: relative;
  .pullup-loading {
    position: relative;
    height: 40px;
    display: flex;
    justify-content: center;
    align-items: center;
  }
}
.KScroll-content {
  transition-timing-function: cubic-bezier(0.23, 1, 0.32, 1);
  transition-duration: 0ms;
  position: relative;
  z-index: 2;
  background-color: #fff;
}
.pullDown-loading {
  width: 100%;
  text-align: center;
  height: 0;
  img {
    width: 50px;
  }
}
.loading-spinners {
  position: relative;
  display: block;
  width: 24px;
  height: 24px;
  .loading-spinner {
    position: absolute;
    left: 44.5%;
    top: 37%;
    width: 2px;
    height: 25%;
    border-radius: 50%/20%;
    opacity: 0.25;
    background-color: currentColor;
    -webkit-animation: spinner-fade 1s linear infinite;
    animation: spinner-fade 1s linear infinite;
  }
  @for $num from 1 through 12 {
    .loading-spinner:nth-child(#{$num}) {
      animation-delay: #{($num - 1) / 12}s;
      transform: rotate(30deg * ($num - 6)) translateY(-150%);
    }
  }
}
</style>
