<template>
  <div>
    <scroll @refresh="handleRefresh" @load="handleLoad">
      <div v-for="item in dataSource" class="item" :key="item">{{ item }}</div>
    </scroll>
  </div>
</template>

<script>
import Scroll from "./components/Index";

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
