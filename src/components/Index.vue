<template>
  <div ref="wrapper" class="NScroll-wrapper">
    <div ref="pull" class="pullDown-loading">
      <img src="/images/loading.gif" alt="" />
    </div>
    <div ref="scroll" class="NScroll-content">
      <slot></slot>
      <div class="pullup-loading">
        <template v-if="config._loadState === 'disabled'">
          {{ relOptions.loadEndText }}
        </template>
        <template v-else-if="config._loadState === 'pullingUp'">
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
        <template v-else-if="config._loadState === 'pending'">
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
  threshold: 50,
  loadEndText: "已经全部加载完毕"
};

export default {
  name: "KScroll",
  props: {
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
      config: {
        _distThreshold: 60,
        _distMax: 80,
        _distReload: 50,
        _distResisted: 0,
        _refreshTimeout: 500,
        _loadTimeout: 200,
        _state: "pending",
        _loadState: "pending"
      }
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
    this._pullEl = this.$refs.pull;
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

    _resistanceFunction(t) {
      return Math.min(1, t / 2.5);
    },

    _pullReset() {
      this._pullEl.classList.add("pending");
      this._pullEl.classList.remove("pulling");
      this._pullEl.classList.remove("refreshing");
      this._pullEl.classList.remove("releasing");
      this._pullEl.style.height = "0px";
      this._pullStartY = this._pullMoveY = null;
      this.config._distResisted = 0;
      setTimeout(() => {
        this.config._state = "pending";
      }, this.config._refreshTimeout);
    },

    initPullDownEvent() {
      const { pullDown } = this.relOptions;
      if (!pullDown) {
        return;
      }
      const { wrapper } = this.$refs;
      wrapper.addEventListener("touchstart", this._onTouchStart);
      wrapper.addEventListener("touchmove", this._onTouchMove);
      wrapper.addEventListener("touchend", this._onTouchEnd);
    },

    _onTouchStart(event) {
      if (this.config._state === "pending") {
        if (this._shouldPullToRefresh()) {
          this._pullStartY = this._screenY(event);
        }
        clearTimeout(this._refreshTimer);
      }
    },

    _onTouchMove(event) {
      if (!this._shouldPullToRefresh()) {
        return;
      }
      if (!this._pullStartY) {
        if (this._shouldPullToRefresh()) {
          this._pullStartY = this._screenY(event);
        }
      } else {
        this._pullMoveY = this._screenY(event);
      }

      if (this.config._state === "refreshing") {
        if (this._shouldPullToRefresh() && this._pullStartY < this._pullMoveY) {
          event.preventDefault();
        }
        return;
      }

      if (this.config._state === "pending") {
        this._pullEl.classList.add("pulling");
        this.config._state = "pulling";
      }

      if (this._pullStartY && this._pullMoveY) {
        this._distance = this._pullMoveY - this._pullStartY;
      }
      this._pullEl.style.height = `${this.config._distResisted}px`;
      this.config._distResisted =
        this._resistanceFunction(this._distance / this.config._distThreshold) *
        Math.min(this.config._distMax, this._distance);

      if (
        this.config._state === "pulling" &&
        this.config._distResisted > this.config._distThreshold
      ) {
        this._pullEl.classList.add("releasing");
        this.config._state = "releasing";
      }

      if (
        this.config._state === "releasing" &&
        this.config._distResisted < this.config._distThreshold
      ) {
        this._pullEl.classList.remove("releasing");
        this.config._state = "pulling";
      }
    },

    _onTouchEnd() {
      // wait 1/2 sec before unmounting...
      clearTimeout(this._timeout);
      this._timeout = setTimeout(() => {
        if (this.config._state === "pending") {
          this._pullReset();
        }
      }, 500);

      if (
        this.config._state === "releasing" &&
        this.config._distResisted > this.config._distThreshold
      ) {
        this.config._state = "refreshing";
        this._pullEl.style.height = `${this.config._distReload}px`;
        this._pullEl.classList.add("refreshing");
        this._refreshTimer = setTimeout(() => {
          if (this.config._loadState === "pullingUp") {
            this._pullReset();
            return;
          }
          let resPromise = null;
          if (this.$listeners["refresh"]) {
            resPromise = this.$listeners.refresh();
          }
          if (resPromise && typeof resPromise.then === "function") {
            resPromise
              .catch(() => {
                this.removePullDownEvent();
              })
              .finally(() => {
                this._pullReset();
                this.scrollTo(0);
              });
          } else {
            this._pullReset();
          }
        }, this.config._refreshTimeout);
      } else {
        if (this.config._state === "refreshing") {
          return;
        }
        this._pullReset();
      }
    },

    removePullDownEvent() {
      const { wrapper } = this.$refs;
      wrapper.removeEventListener("touchstart", this._onTouchStart);
      wrapper.removeEventListener("touchmove", this._onTouchMove);
      wrapper.removeEventListener("touchend", this._onTouchEnd);
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
      if (this.config._loadState === "pullingUp") {
        return;
      }
      const { scrollY, innerHeight } = window;
      const { scrollHeight } = document.body;
      if (innerHeight + scrollY >= scrollHeight - 10) {
        this.config._loadState = "pullingUp";
        let resPromise = null;
        if (this.$listeners.load) {
          resPromise = this.$listeners.load();
        }
        if (resPromise && typeof resPromise.then === "function") {
          resPromise
            .then(() => {
              this.config._loadState = "pending";
            })
            .catch(() => {
              this.removePullUpEvent();
              this.config._loadState = "disabled";
            });
        } else {
          this.config._loadState = "pending";
        }
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
.NScroll-wrapper {
  position: relative;
  .pullup-loading {
    position: relative;
    height: 40px;
    display: flex;
    justify-content: center;
    align-items: center;
  }
}
.NScroll-content {
  transition-timing-function: cubic-bezier(0.23, 1, 0.32, 1);
  transition-duration: 0ms;
  position: relative;
  z-index: 2;
  background-color: #fff;
}
.pullDown-loading {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  height: 0;
  box-shadow: inset 0 -3px 5px rgba(0, 0, 0, 0.12);
  pointer-events: none;
  font-size: 0.85em;
  font-weight: bold;
  top: 0;
  transition: height 0.3s, min-height 0.3s;
  overflow: hidden;
  img {
    width: 50px;
  }
  &.pulling {
    transition: none;
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
    animation: spinner-fade 1s linear infinite;
  }
  @for $num from 1 through 12 {
    .loading-spinner:nth-child(#{$num}) {
      animation-delay: #{($num - 1) / 12}s;
      transform: rotate(30deg * ($num - 6)) translateY(-150%);
    }
  }

  @keyframes spinner-fade {
    0% {
      opacity: 0.85;
    }
    50% {
      opacity: 0.25;
    }
    100% {
      opacity: 0.25;
    }
  }
}
</style>
