<template>
  <div id="app">
    <router-view></router-view>
  </div>
</template>

<script>
export default {
  name: 'App'
}
</script>

<style>
body, html {
  margin: 0;
  padding: 0;
  font-family: Arial, sans-serif;
}
#app {
  width: 100%;
  height: 100vh;
}
</style>