<template>
  <ul class="tab">
    <router-link tag="li" class="tab-item" to="/recommend">
      <span>推荐</span>
    </router-link>
    <router-link tag="li" class="tab-item" to="/singer">
      <span>歌手</span>
    </router-link>
    <router-link tag="li" class="tab-item" to="/rank">
      <span>排行</span>
    </router-link>
    <router-link tag="li" class="tab-item" to="/search">
      <span>搜索</span>
    </router-link>
  </ul>
</template>

<script>
  export default {}
</script>

<style scoped lang="scss">
  @import '../common/sass/index';
  .tab {
    display: flex;
    justify-content: space-around;
    padding-bottom: px2rem(8);
    color: $color-text-l;
    .tab-item {
      padding: px2rem(8) px2rem(16);
      &.router-link-active {
        color: $color-theme;
        border-bottom: 2px $color-theme solid;
      }
    }
  }
</style>
