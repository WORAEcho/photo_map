<script setup>
import HelloWorld from './components/HelloWorld.vue'
import TheWelcome from './components/TheWelcome.vue'
import AmapStatic from './components/AmapStatic.vue'
</script>


<template>

  <!-- <header> -->

    <!-- <img alt="Vue logo" class="logo" src="./assets/logo.svg" width="125" height="125" />

    <div class="wrapper">
      <HelloWorld msg="You did it!" />
    </div> -->
  <!-- </header> -->

  <!-- <main> -->

    <!-- <TheWelcome /> -->
  <!-- </main> -->
  <div>
    <!-- 网页标题栏 -->
    <header style="width: 100%; height: 64px; background: #fff; box-shadow: 0 2px 8px rgba(0,0,0,0.04); display: flex; align-items: center; justify-content: center; position: fixed; top: 0; left: 0; z-index: 100;">
      <img src="/src/assets/logo.svg" alt="Logo" style="height: 40px; display: block;" />
    </header>
    <div style="height: 64px;"></div> <!-- 占位，避免内容被标题栏遮挡 -->
    <!-- 其余内容 -->
    <router-view />
  </div>
  <AmapStatic />

</template>

<style scoped>
header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
}
</style>
