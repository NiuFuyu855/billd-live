<template>
  <div class="about-wrap">
    <h2>billd-live目前实现</h2>
    <h3>1. 原生webrtc一对多直播（DONE）</h3>
    <h3>2. srs-webrtc一对多直播（DONE）</h3>
    <h3>3. 原生webrtc多对多直播（DONE）</h3>

    <h1>微信交流群 & 我的微信</h1>
    <img
      src="@/assets/img/wechat-group.webp"
      alt=""
      class="wechat-group"
    />
    <img
      src="@/assets/img/my-wechat.webp"
      alt=""
      class="my-wechat"
    />
  </div>
</template>

<script lang="ts" setup></script>

<style lang="scss" scoped>
.about-wrap {
  text-align: center;
  .wechat-group {
    height: 500px;
  }
  .my-wechat {
    height: 500px;
  }
}
</style>
