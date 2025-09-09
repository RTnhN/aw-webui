<template lang="pug">
svg
</template>

<style scoped lang="scss">
@import '../style/globals';

svg {
  width: 100%;
  height: 40pt;
  border: 1px solid $lightBorderColor;
  border-radius: 0.5em;
}
</style>

<script lang="ts">
// NOTE: This is just a Vue.js component wrapper for periodusage.js
//       Code should generally go in the framework-independent file.

import { PropType } from 'vue';
import { IEvent } from '~/util/interfaces';
import periodusage from './periodusage';

export default {
  name: 'aw-periodusage',
  props: {
    periodusage_arr: {
      type: Array as PropType<IEvent[][] | null>,
      default: null,
    },
  },
  watch: {
    periodusage_arr: {
      handler() {
        if (!this.periodusage_arr) {
          periodusage.set_status(this.$el, 'Loading...');
        } else {
          periodusage.update(this.$el, this.periodusage_arr, this.onPeriodClicked);
        }
      },
      immediate: true,
    },
  },
  mounted: function () {
    periodusage.create(this.$el);
    periodusage.set_status(this.$el, 'Loading...');
  },
  methods: {
    onPeriodClicked: function (period) {
      this.$emit('update', period);
    },
  },
};
</script>
