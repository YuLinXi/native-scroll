<template>
  <div>
    <scroll ref="scroll" @refresh="handleRefresh" @load="handleLoad">
      <div v-for="item in dataSource" class="item" :key="item">{{ item }}</div>
    </scroll>
    <div @click="handleGoTop" class="toTop">回到顶部</div>
  </div>
</template>

<script>
import Scroll from "../package/Index";

let index = 1;

export default {
  name: "App",
  data() {
    return {
      dataSource: []
    };
  },
  components: {
    Scroll
  },
  mounted() {
    this.dataSource = this.getDataSource();
  },
  methods: {
    handleRefresh() {
      return new Promise(resolve => {
        setTimeout(() => {
          index = 1;
          this.dataSource = this.getDataSource();
          console.log("refresh");
          resolve();
        }, 1000);
      });
    },
    handleLoad() {
      return new Promise(resolve => {
        setTimeout(() => {
          this.dataSource = this.dataSource.concat(this.getDataSource());
          console.log("load");
          resolve();
        }, 2000);
      });
    },

    getDataSource() {
      const result = index + 20;
      const data = [];
      for (index; index < result; index++) {
        data.push(index);
      }
      return data;
    },

    handleGoTop() {
      const { scroll } = this.$refs;
      scroll.scrollTo(0, 100);
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
.toTop {
  color: #000;
  background-color: aqua;
  position: fixed;
  right: 20px;
  bottom: 50px;
}
</style>
