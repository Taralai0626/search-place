<template>
  <h1 style="color: #8ab388;"> Search Location</h1>
  <SearchLocation msg="Welcome to Your Vue.js App"/>
</template>

<script>
import SearchLocation from './components/SearchLocation.vue';

export default {
  name: 'App',
  components: {
    SearchLocation
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
