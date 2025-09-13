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

import periodusage from './periodusage';

export default {
  name: 'aw-periodusage',
  props: {
    periodusage_arr: {
      type: Array,
    },
    loading: {
      type: Boolean,
      default: false,
    },
  },
  watch: {
    periodusage_arr: function () {
      if (this.loading || !this.periodusage_arr || this.periodusage_arr.length === 0) {
        periodusage.set_status(this.$el, 'Loading...');
      } else {
        periodusage.update(this.$el, this.periodusage_arr, this.onPeriodClicked);
      }
    },
    loading: function (val) {
      if (val) {
        periodusage.set_status(this.$el, 'Loading...');
      } else if (this.periodusage_arr && this.periodusage_arr.length > 0) {
        periodusage.update(this.$el, this.periodusage_arr, this.onPeriodClicked);
      }
    },
  },
  mounted: function () {
    periodusage.create(this.$el);
    if (this.loading) {
      periodusage.set_status(this.$el, 'Loading...');
    }
  },
  methods: {
    onPeriodClicked: function (period) {
      this.$emit('update', period);
    },
  },
};
</script>
