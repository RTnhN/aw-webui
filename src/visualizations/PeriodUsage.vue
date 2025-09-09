<template lang="pug">
div.periodusage-container.position-relative
  svg(ref="svg", v-show="!loading")
  div.spinner-overlay.d-flex.justify-content-center.align-items-center(v-if="loading")
    icon(name="spinner" pulse)
</template>

<style scoped lang="scss">
@import '../style/globals';

.periodusage-container {
  width: 100%;
  height: 40pt;
  border: 1px solid $lightBorderColor;
  border-radius: 0.5em;
}

svg {
  width: 100%;
  height: 100%;
}

.spinner-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}
</style>

<script lang="ts">
// NOTE: This is just a Vue.js component wrapper for periodusage.js
//       Code should generally go in the framework-independent file.

import periodusage from './periodusage';
import 'vue-awesome/icons/spinner';

export default {
  name: 'aw-periodusage',
  props: {
    periodusage_arr: {
      type: Array,
    },
  },
  data() {
    return {
      loading: true,
    };
  },
  watch: {
    periodusage_arr: {
      handler() {
        if (this.periodusage_arr) {
          periodusage.update(
            this.$refs.svg as SVGElement,
            this.periodusage_arr,
            this.onPeriodClicked
          );
          this.loading = false;
        } else {
          this.loading = true;
        }
      },
      immediate: true,
    },
  },
  mounted() {
    periodusage.create(this.$refs.svg as SVGElement);
  },
  methods: {
    onPeriodClicked(period: string) {
      this.$emit('update', period);
    },
  },
};
</script>
