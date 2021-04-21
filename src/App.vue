<template>
  <div id="app">
    <Simulator :animation="animation" :environments="environments" />
  </div>
</template>

<script>
import Simulator from "./components/Simulator.vue";

export default {
  components: { Simulator },
  data() {
    return {
      animation: JSON.parse(this.$route.query.commands || "[]"),
      environments: []
    };
  }
};
</script>

