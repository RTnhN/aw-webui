<template lang="pug">
div
  div.d-sm-flex.justify-content-between
    div
      h5.mb-2.mb-sm-0 Timeline zoom modifier
    div
      b-select(size="sm" :value="zoomKey" @change="zoomKey = $event")
        option(value="ctrlKey") Ctrl
        option(value="altKey") Alt/Option
        option(value="metaKey") Meta/Cmd
        option(value="none") None
  small
    | Choose which modifier key (if any) is required to zoom the timeline with the scroll wheel. Default: None.
</template>
<script lang="ts">
import { useSettingsStore } from '~/stores/settings';

export default {
  name: 'TimelineZoomSettings',
  data() {
    return {
      settingsStore: useSettingsStore(),
    };
  },
  computed: {
    zoomKey: {
      get() {
        return this.settingsStore.timelineZoomKey || 'ctrlKey';
      },
      set(value: 'ctrlKey' | 'altKey' | 'metaKey' | 'none') {
        this.settingsStore.update({ timelineZoomKey: value });
      },
    },
  },
};
</script>
