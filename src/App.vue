<template>
  <div id="app">
    <div v-if="ready">
      <div v-if="error" id="error_msg">An error was met when parsing the commands,
        please check whether the url is correct.
      </div>
      <Simulator v-if="!error" :commands="commands" :config="config"/>
      <div v-if="!error">Rotate Camera: Click and Drag</div>
      <div v-if="!error">Move Camera: Press Ctrl + Click and Drag</div>
      <div v-if="!error">Zoom in/out: Scroll</div>
    </div>
    <div v-else>
      Loading...
    </div>
  </div>
</template>

<script>
import Simulator from "./components/Simulator.vue";

export default {
  components: {Simulator},
  data() {
    return {
      config: [],
      commands: [],
      error: null,
      ready: false
    };
  },
  mounted() {
    const compressed_str = this.$route.query.data;
    if (compressed_str) {
      const pako = require('pako');
      const compressed = Buffer(compressed_str, "base64");
      try {
        const data = JSON.parse(pako.inflate(compressed, {to: "string"}));
        this.config = data.config;
        this.commands = data.commands;
      } catch (err) {
        this.error = err;
      }
    }
    this.ready = true;
  }
};
</script>

<style scoped>
#error_msg {
  color: red;
  margin: 50px;
}
</style>
