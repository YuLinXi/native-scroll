<template>
  <div>
    <scroll @refresh="handleRefresh" @load="handleLoad">
      <div class="item" v-for="item in 100">{{ item }}</div>
    </scroll>
  </div>
</template>

<script>
import Scroll from "./components/Index";
export default {
  name: "App",
  components: {
    Scroll
  },
  methods: {
    handleRefresh() {
      return new Promise((resolve, reject) => {
        setTimeout(() => {
          reject(new Error("111"));
        }, 1000);
      });
    },
    handleLoad() {
      return new Promise((resolve, reject) => {
        setTimeout(() => {
          reject(new Error("222"));
        }, 2000);
      });
    }
  }
};
</script>

<style lang="scss">
.item {
  border: 1px solid blue;
  height: 50px;
  &:first-child {
    border-top: 0;
  }
}
</style>
