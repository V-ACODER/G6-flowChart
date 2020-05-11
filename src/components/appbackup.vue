<template>
  <div id="app">
    <!-- <demo1 /> -->
    <demo2 />
  </div>
</template>

<script>
// import demo1 from "./components/demo1";
import demo2 from "./components/demo2";

export default {
  name: "App",
  components: {
    // demo1,
    demo2
  }
};
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
