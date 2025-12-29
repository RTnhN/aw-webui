<template lang="pug">
div
  h2 Timeline

  input-timeinterval(v-model="daterange", :defaultDuration="timeintervalDefaultDuration", :maxDuration="maxDuration").mb-3

  // blocks
  div.d-inline-block.border.rounded.mr-2
    div.d-inline-block.p-2.mr-2
      | Swimlanes:  
      select(v-model="swimlane")
        option(:value='null') None
        option(value='category') Categories
        option(value='bucketType') Bucket Specific
    details.d-inline-block.bg-light.mr-2.p-2(ref="filterDetails")
      summary
        | Host and Client Filters
        icon(name="filter" v-if="filtersApplied" class="fa-icon")
      div.p-2.bg-light.filter-panel
        // Host filter controls
        .filter-group.mb-2
          label.mb-1.mr-2 Host:
          .btn-group.mb-1
            button.btn.btn-sm.btn-link(@click.prevent="selectAllHosts") All
            button.btn.btn-sm.btn-link(@click.prevent="clearAllHosts") None
          .checkbox-list.mb-3
            div(v-for="host in hosts" :key="host")
              label.d-block
                input(type="checkbox" :value="host" v-model="filter_hostnames" class="mr-2")
                | {{ host }}
        // Client filter controls
        .filter-group
          label.mb-1.mr-2 Client:
          .btn-group.mb-1
            button.btn.btn-sm.btn-link(@click.prevent="selectAllClients") All
            button.btn.btn-sm.btn-link(@click.prevent="clearAllClients") None
          .checkbox-list
            div(v-for="client in clients" :key="client")
              label.d-block
                input(type="checkbox" :value="client" v-model="filter_clients" class="mr-2")
                | {{ client }}
    div.d-inline-block.p-2.mr-2
      | Duration: 
      select(v-model="filter_duration")
        option(:value='null') All
        option(:value='2') 2+ secs
        option(:value='5') 5+ secs
        option(:value='10') 10+ secs 
        option(:value='30') 30+ sec
        option(:value='1 * 60') 1+ mins
        option(:value='2 * 60') 2+ mins
        option(:value='3 * 60') 3+ mins
        option(:value='10 * 60') 10+ mins
        option(:value='30 * 60') 30+ mins
        option(:value='1 * 60 * 60') 1+ hrs
        option(:value='2 * 60 * 60') 2+ hrs
  div.d-inline-block.border.rounded.p-2.mr-2(v-if="num_events !== 0")
    | Events shown: {{ num_events }}
  b-alert.d-inline-block.p-2.mb-0.mt-2(v-if="num_events === 0", variant="warning", show)
    | No events match selected criteria. Timeline is not updated.
  div(style="float: right; color: #999").d-inline-block.pt-3
    | Drag to pan and {{ zoomInstruction }}

  div(style="clear: both")
  div.mt-3.mb-2
    b-button.mr-2(
      size="sm",
      :variant="toolMenuOpen ? 'primary' : 'outline-primary'",
      @click="toggleToolMenu"
    ) Tools
    span.text-muted.small(v-if="activeTool") Active tool: {{ activeTool }}
    div.d-inline-block.ml-2(v-if="toolMenuOpen")
      b-button.mr-2(
        size="sm",
        :variant="undoAvailable ? 'primary' : 'outline-secondary'",
        :disabled="!undoAvailable",
        @click="undoLastChange",
        title="Undo last timeline tool action"
      ) Undo
      b-button.mr-1(size="sm", :variant="activeTool === 'cut' ? 'primary' : 'outline-primary'" @click="selectTool('cut')" title="Split an event at the clicked time") Cut
      b-button.mr-1(size="sm", :variant="activeTool === 'glue' ? 'primary' : 'outline-primary'" @click="selectTool('glue')" title="Merge two consecutive events into one") Glue
      b-button.mr-1(size="sm", :variant="activeTool === 'grow' ? 'primary' : 'outline-primary'" @click="selectTool('grow')" title="Extend first event to start of second to fill a gap") Grow
      b-button.mr-1(size="sm", :variant="activeTool === 'shrink' ? 'primary' : 'outline-primary'" @click="selectTool('shrink')" title="Trim first event to start of second to remove overlap") Shrink
      b-button.mr-1(size="sm", :variant="activeTool === 'clone' ? 'primary' : 'outline-primary'" @click="selectTool('clone')" title="Copy data/title from one event onto another") Clone
      b-button.mr-1(size="sm", :variant="activeTool === 'swap' ? 'primary' : 'outline-primary'" @click="selectTool('swap')" title="Swap time and data between two events") Swap

  div(v-if="buckets !== null")
    div(style="clear: both")
    vis-timeline(
      ref="visTimeline"
      :buckets="buckets"
      :showRowLabels='true'
      :queriedInterval="daterange"
      :swimlane="swimlane"
      :updateTimelineWindow='updateTimelineWindow'
      :tool="activeTool"
      @undo-available="onUndoAvailable"
    )

    aw-devonly(reason="Not ready for production, still experimenting")
      aw-calendar(:buckets="buckets")
  div(v-else)
    h1.aw-loading Loading...
</template>

<script lang="ts">
import _ from 'lodash';
import { useSettingsStore } from '~/stores/settings';
import { useBucketsStore } from '~/stores/buckets';
import 'vue-awesome/icons/filter';

const zoomInstructionText = (zoomKey: string | undefined) => {
  switch (zoomKey) {
    case 'altKey':
      return 'hold Alt/Option while scrolling to zoom';
    case 'metaKey':
      return 'hold Cmd/Meta while scrolling to zoom';
    case '':
    case 'none':
      return 'scroll to zoom';
    case 'ctrlKey':
    default:
      return 'hold Ctrl while scrolling to zoom';
  }
};

export default {
  name: 'Timeline',
  data() {
    const savedHosts = JSON.parse(
      (typeof localStorage !== 'undefined' && localStorage.getItem('timeline_filter_hostnames')) ||
        '[]'
    );
    const savedClients = JSON.parse(
      (typeof localStorage !== 'undefined' && localStorage.getItem('timeline_filter_clients')) ||
        '[]'
    );
    const savedDuration = JSON.parse(
      (typeof localStorage !== 'undefined' && localStorage.getItem('timeline_filter_duration')) ||
        'null'
    );
    return {
      all_buckets: null as any[] | null,
      hosts: null as string[] | null,
      buckets: null as any[] | null,
      clients: null as string[] | null,
      daterange: null as any,
      maxDuration: 31 * 24 * 60 * 60,
      filter_hostnames: savedHosts as string[],
      filter_clients: savedClients as string[],
      firstLoad: true,
      filter_duration: savedDuration as number | null,
      swimlane: null,
      updateTimelineWindow: true,
      toolMenuOpen: false,
      activeTool: null as string | null,
      undoAvailable: false,
    };
  },
  computed: {
    timeintervalDefaultDuration(): number {
      const settingsStore = useSettingsStore();
      return Number(settingsStore.durationDefault);
    },
    zoomInstruction(): string {
      const settingsStore = useSettingsStore();
      return zoomInstructionText(settingsStore.timelineZoomKey);
    },
    num_events(): number {
      return _.sumBy(this.buckets || [], 'events.length');
    },
    filtersApplied(): boolean {
      if (!this.hosts || !this.clients) return false;
      const allHosts = this.filter_hostnames.length === this.hosts.length;
      const allClients = this.filter_clients.length === this.clients.length;
      const durationDefault = this.filter_duration === null;
      return !(allHosts && allClients && durationDefault);
    },
  },
  watch: {
    daterange() {
      this.updateTimelineWindow = true;
      this.getBuckets();
    },
    filter_hostnames() {
      this.updateTimelineWindow = false;
      if (typeof localStorage !== 'undefined') {
        localStorage.setItem('timeline_filter_hostnames', JSON.stringify(this.filter_hostnames));
      }
      this.getBuckets();
    },
    filter_clients() {
      this.updateTimelineWindow = false;
      if (typeof localStorage !== 'undefined') {
        localStorage.setItem('timeline_filter_clients', JSON.stringify(this.filter_clients));
      }
      this.getBuckets();
    },
    filter_duration() {
      this.updateTimelineWindow = false;
      if (typeof localStorage !== 'undefined') {
        localStorage.setItem('timeline_filter_duration', JSON.stringify(this.filter_duration));
      }
      this.getBuckets();
    },
    swimlane() {
      this.updateTimelineWindow = false;
      this.getBuckets();
    },
  },
  mounted() {
    document.addEventListener('click', this.handleClickOutside, true);
  },
  beforeUnmount() {
    document.removeEventListener('click', this.handleClickOutside, true);
  },
  methods: {
    async getBuckets() {
      if (!this.daterange) return;

      this.all_buckets = Object.freeze(
        await useBucketsStore().getBucketsWithEvents({
          start: this.daterange[0].format(),
          end: this.daterange[1].format(),
        })
      );

      this.hosts = [...new Set(this.all_buckets.map(a => a.hostname))];
      this.clients = [...new Set(this.all_buckets.map(a => a.client))];

      // ensure filters are valid for current options
      const validHosts = this.filter_hostnames.filter(h => this.hosts.includes(h));
      if (!_.isEqual(validHosts, this.filter_hostnames)) {
        this.filter_hostnames = validHosts;
      }
      const validClients = this.filter_clients.filter(c => this.clients.includes(c));
      if (!_.isEqual(validClients, this.filter_clients)) {
        this.filter_clients = validClients;
      }

      // default select all only once if no saved filters
      if (this.firstLoad) {
        if (this.filter_hostnames.length === 0) {
          this.filter_hostnames = [...(this.hosts || [])];
        }
        if (this.filter_clients.length === 0) {
          this.filter_clients = [...(this.clients || [])];
        }
        this.firstLoad = false;
      }

      let buckets = this.all_buckets;
      if (this.filter_hostnames.length) {
        buckets = buckets.filter(b => this.filter_hostnames.includes(b.hostname));
      }
      if (this.filter_clients.length) {
        buckets = buckets.filter(b => this.filter_clients.includes(b.client));
      }
      if (this.filter_duration > 0) {
        for (const bucket of buckets) {
          bucket.events = _.filter(bucket.events, e => e.duration >= this.filter_duration);
        }
      }

      this.buckets = buckets;
    },
    selectAllHosts() {
      this.filter_hostnames = [...(this.hosts || [])];
    },
    clearAllHosts() {
      this.filter_hostnames = [];
    },
    selectAllClients() {
      this.filter_clients = [...(this.clients || [])];
    },
    clearAllClients() {
      this.filter_clients = [];
    },
    toggleToolMenu() {
      this.toolMenuOpen = !this.toolMenuOpen;
    },
    selectTool(tool: string) {
      this.activeTool = this.activeTool === tool ? null : tool;
    },
    onUndoAvailable(status: boolean) {
      this.undoAvailable = status;
    },
    async undoLastChange() {
      const visTimeline = this.$refs.visTimeline as any;
      if (visTimeline && typeof visTimeline.triggerUndo === 'function') {
        await visTimeline.triggerUndo();
      }
    },
    handleClickOutside(event: MouseEvent) {
      const details = this.$refs.filterDetails as HTMLElement | undefined;
      if (details && details.hasAttribute('open')) {
        if (!details.contains(event.target as Node)) {
          details.removeAttribute('open');
        }
      }
    },
  },
};
</script>

<style scoped>
details {
  position: relative;
}

details[open] summary ~ * {
  visibility: visible;
  position: absolute;
  border: 1px solid #ddd;
  border-radius: 5px;
  left: 0;
  top: 2.7em;
  background: white;
  z-index: 100;
}

.filter-panel {
  min-width: 250px;
}

.checkbox-list {
  display: flex;
  flex-direction: column;
}
.filter-group {
  margin-bottom: 0.5rem;
}
.btn-group button {
  margin-right: 0.5rem;
  padding: 0;
}
</style>
