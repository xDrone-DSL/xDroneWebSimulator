<template>
  <div id="app">
    <div v-if="error" id="error_msg">An error was met when parsing the commands,
      please check whether the url is correct.
    </div>
    <Simulator v-if="!error" :animation="animation" :environments="environments"/>
  </div>
</template>

<script>
import Simulator from "./components/Simulator.vue";

export default {
  components: {Simulator},
  data() {
    return {
      animation: [],
      environments: [],
      error: null
    };
  },
  mounted() {
    const compressed_str = this.$route.query.commands;
    if (compressed_str) {
      const pako = require('pako');
      const compressed = Buffer(compressed_str, "base64");
      try {
        this.animation = JSON.parse(pako.inflate(compressed, {to: "string"}));
      } catch (err) {
        this.error = err;
      }
    }
  }
};
</script>

<style scoped>
#error_msg {
  color: red;
  margin: 50px;
}
</style>
