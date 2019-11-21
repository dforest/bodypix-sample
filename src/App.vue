<template>
  <div id="app">
    <BodyPix/>
  </div>
</template>

<script>
import BodyPix from './components/BodyPix.vue'

export default {
  name: 'app',
  components: {
    BodyPix
  }
}
</script>

<style>
body {
  margin: 0;
  width: 100%;
}

#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  width: 100%;
}
</style>
