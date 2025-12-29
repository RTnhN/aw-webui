<template lang="pug">
  div
    div#visualization

    div.small.text-muted.my-2(v-if="bucketsFromEither.length != 1")
      i Buckets with no events in the queried range will be hidden.

    div(v-if="editingEvent")
      EventEditor(:event="editingEvent" :bucket_id="editingEventBucket")
</template>

<style lang="scss">
div#visualization {
  margin-top: 0.5em;
  margin-bottom: 0.5em;
  overflow: visible;
}

.vis-timeline {
  overflow: visible;
}

.timeline-timeline {
  font-family: sans-serif !important;

  .timeline-panel {
    box-sizing: border-box;
  }

  .timeline-item {
    border-radius: 2px;
  }
}
</style>

<script lang="ts">
import _ from 'lodash';
import moment from 'moment';
import Color from 'color';
import { buildTooltip } from '../util/tooltip.js';
import { getCategoryColorFromEvent, getTitleAttr } from '../util/color';
import { getSwimlane } from '../util/swimlane.js';
import { IEvent } from '../util/interfaces';
import { useSettingsStore } from '~/stores/settings';

import { Timeline } from 'vis-timeline/esnext';
import 'vis-timeline/styles/vis-timeline-graph2d.css';
import EventEditor from '~/components/EventEditor.vue';

let isAlertWarningShown = false;

const mapZoomKeySetting = (value: 'ctrlKey' | 'altKey' | 'metaKey' | 'none' | string | undefined) => {
  if (value === 'none' || value === '') {
    return '';
  }
  if (value === 'ctrl' || value === 'alt' || value === 'meta') {
    return value + 'Key';
  }
  return value || '';
};

interface IChartDataItem {
  bucketId: string;
  title: string;
  tooltip: string;
  start: Date;
  end: Date;
  color: string;
  event: IEvent;
  swimlane: string;
}

type EventWithId = IEvent & { id?: number };

type UndoEntry =
  | {
      type: 'cut';
      bucketId: string;
      rangeStart: string;
      rangeEnd: string;
      originalEvent: EventWithId;
      newEventSignature: {
        timestamp: Date;
        duration: number;
        data: Record<string, unknown>;
        id?: number;
      };
    }
  | {
      type: 'glue';
      bucketId: string;
      rangeStart: string;
      rangeEnd: string;
      firstOriginal: EventWithId;
      secondOriginal: EventWithId;
    }
  | {
      type: 'grow' | 'shrink';
      bucketId: string;
      rangeStart: string;
      rangeEnd: string;
      originalEvent: EventWithId;
    }
  | {
      type: 'clone';
      bucketId: string;
      rangeStart: string;
      rangeEnd: string;
      originalEvent: EventWithId;
    }
  | {
      type: 'swap';
      bucketId: string;
      rangeStart: string;
      rangeEnd: string;
      firstOriginal: EventWithId;
      secondOriginal: EventWithId;
    };
export default {
  components: {
    EventEditor,
  },
  props: {
    buckets: { type: Array },
    events: { type: Array },
    showRowLabels: { type: Boolean },
    queriedInterval: { type: Array },
    showQueriedInterval: { type: Boolean },
    swimlane: { type: String },
    updateTimelineWindow: { type: Boolean },
    tool: { type: String, default: null },
  },
  data() {
    const settingsStore = useSettingsStore();
    return {
      timeline: null,
      filterShortEvents: true,
      items: [],
      groups: [],
      options: {
        zoomMin: 1000 * 60, // 10min in milliseconds
        zoomMax: 1000 * 60 * 60 * 24 * 31 * 3, // about three months in milliseconds
        zoomKey: mapZoomKeySetting(settingsStore.timelineZoomKey),
        stack: false,
        tooltip: {
          followMouse: true,
          overflowMethod: 'flip',
          delay: 0,
        },
      },
      editingEvent: null,
      editingEventBucket: null,

      pendingSelection: null,
      updateHasRun: false,

      undoStack: [] as UndoEntry[],
      undoLimit: 20,
      settingsStore,
    };
  },
  computed: {
    zoomKeySetting(): string {
      return mapZoomKeySetting(this.settingsStore.timelineZoomKey);
    },
    bucketsFromEither() {
      if (this.buckets) {
        return this.buckets;
      } else if (this.events) {
        // If buckets not passed, check if events have been passed and generate a bucket from those events
        return [
          {
            id: 'events',
            type: 'search',
            events: this.events,
          },
        ];
      } else {
        console.error('No buckets or events passed to timeline');
        return [];
      }
    },
    chartData(): IChartDataItem[] {
      const data: IChartDataItem[] = [];
      _.each(this.bucketsFromEither, bucket => {
        if (bucket.events === undefined) {
          return;
        }
        let events = bucket.events;
        // Filter out events shorter than 1 second (notably including 0-duration events)
        // TODO: Use flooding instead, preferably with some additional method of removing/simplifying short events for even greater performance
        if (this.filterShortEvents) {
          events = _.filter(events, e => e.duration > 1);
          console.log(`Filtered ${bucket.events.length - events.length} events`);
        }
        events.sort((a, b) => a.timestamp.valueOf() - b.timestamp.valueOf());
        _.each(events, e => {
          data.push({
            bucketId: bucket.id,
            title: getTitleAttr(bucket, e),
            tooltip: buildTooltip(bucket, e),
            start: new Date(e.timestamp),
            end: new Date(moment(e.timestamp).add(e.duration, 'seconds').valueOf()),
            color: getCategoryColorFromEvent(bucket, e),
            event: e,
            swimlane: getSwimlane(bucket, getCategoryColorFromEvent(bucket, e), this.swimlane, e),
          });
        });
      });
      return data;
    },
  },
  watch: {
    zoomKeySetting(newVal) {
      this.options.zoomKey = newVal;
      if (this.timeline) {
        this.timeline.setOptions(this.options);
      }
    },
    tool(newVal) {
      if (!['glue', 'grow', 'shrink', 'clone', 'swap'].includes(newVal || '')) {
        this.pendingSelection = null;
      }
    },
    buckets() {
      // For some reason, an object is passed here, after which the correct array arrives
      if (this.buckets.length === undefined) {
        //console.log("I told you so!")
        return;
      }

      this.update();
    },
    events() {
      if (this.events.length === undefined) {
        return;
      }

      this.update();
    },
  },
  mounted() {
    this.$nextTick(() => {
      const el = this.$el.querySelector('#visualization');
      this.timeline = new Timeline(el, [], [], this.options);
      this.timeline.on('select', properties => {
        // Sends both 'press' and 'tap' events, only one should trigger
        if (properties.event.type == 'tap') {
          this.onSelect(properties);
        }
      });

      this.ensureUpdate();
      this.emitUndoAvailability();
    });
  },
  methods: {
    openEditor: function () {
      this.$bvModal.show('edit-modal-' + this.editingEvent.id);
    },
    onSelect: async function (properties) {
      if (properties.items.length == 0) {
        return;
      } else if (properties.items.length !== 1) {
        alert('selected multiple items: ' + JSON.stringify(properties.items));
        return;
      }

      const selection = this.getSelectionContext(properties.items[0]);
      if (!selection) return;

      const { event, bucketId } = selection;

      if (this.tool === 'cut') {
        await this.handleCut(selection, properties);
        this.clearSelection();
        return;
      }
      if (this.tool === 'glue') {
        await this.handleGlue(selection);
        this.clearSelection();
        return;
      }
      if (this.tool === 'clone') {
        await this.handleClone(selection);
        this.clearSelection();
        return;
      }
      if (this.tool === 'swap') {
        await this.handleSwap(selection);
        this.clearSelection();
        return;
      }
      if (this.tool === 'grow') {
        await this.handleGrow(selection);
        this.clearSelection();
        return;
      }
      if (this.tool === 'shrink') {
        await this.handleShrink(selection);
        this.clearSelection();
        return;
      }

      // We retrieve the full event to ensure if's not cut-off by the query range
      // See: https://github.com/ActivityWatch/aw-webui/pull/320#issuecomment-1056921587
      this.editingEvent = await this.$aw.getEvent(bucketId, event.id);
      this.editingEventBucket = bucketId;

      this.$nextTick(() => {
        console.log('Editing event', event, ', in bucket', bucketId);
        this.openEditor();
      });
      if (!isAlertWarningShown) {
        alert(
          "Note: Changes won't be reflected in the timeline until the page is refreshed. This will be improved in a future version."
        );
        isAlertWarningShown = true;
      }
      this.clearSelection();
    },
    clearSelection() {
      if (this.timeline) {
        this.timeline.setSelection([]);
      }
    },
    getSelectionContext(itemIndex: number) {
      const chartItem = this.chartData[itemIndex];
      const item = this.items[itemIndex];
      if (!chartItem || !item) return null;
      return {
        event: chartItem.event,
        bucketId: item.group,
        itemIndex,
      };
    },
    getClickTime(properties) {
      if (!this.timeline || !properties || !properties.event) return null;
      try {
        const eventProps = this.timeline.getEventProperties(properties.event);
        if (eventProps && eventProps.time) {
          return moment(eventProps.time);
        }
      } catch (err) {
        console.warn('Unable to determine click time from timeline', err);
      }
      return null;
    },
    async handleCut(selection, properties) {
      const { event, bucketId } = selection;
      const fullEvent = await this.$aw.getEvent(bucketId, event.id);
      const start = moment(fullEvent.timestamp);
      const end = start.clone().add(fullEvent.duration, 'seconds');

      const clickedTime = this.getClickTime(properties);
      let cutMoment =
        clickedTime && clickedTime.isBetween(start, end, undefined, '[]')
          ? clickedTime
          : start.clone().add(fullEvent.duration / 2, 'seconds');
      if (cutMoment.isSameOrBefore(start)) cutMoment = start.clone().add(1, 'seconds');
      if (cutMoment.isSameOrAfter(end)) cutMoment = end.clone().subtract(1, 'seconds');

      if (!cutMoment.isBetween(start, end)) {
        alert('Unable to determine cut position inside the event.');
        return;
      }

      const firstDuration = cutMoment.diff(start, 'seconds', true);
      const secondDuration = end.diff(cutMoment, 'seconds', true);
      if (firstDuration <= 0 || secondDuration <= 0) {
        alert('Cut time must be within the event duration.');
        return;
      }

      const updatedEvent = _.cloneDeep(fullEvent);
      updatedEvent.duration = firstDuration;

      const { id, ...rest } = _.cloneDeep(fullEvent);
      const newEvent = {
        ...rest,
        timestamp: cutMoment.toDate(),
        duration: secondDuration,
      };

      try {
        await this.$aw.replaceEvent(bucketId, updatedEvent);
        await this.$aw.insertEvent(bucketId, newEvent);
        await this.refreshBucketSegment(bucketId, start, end);

        const createdEvent = this.findEventMatch(
          bucketId,
          newEvent.timestamp,
          newEvent.duration,
          newEvent.data
        );
        this.pushUndoEntry({
          type: 'cut',
          bucketId,
          rangeStart: start.toISOString(),
          rangeEnd: end.toISOString(),
          originalEvent: _.cloneDeep(fullEvent),
          newEventSignature: {
            timestamp: newEvent.timestamp,
            duration: newEvent.duration,
            data: _.cloneDeep(newEvent.data),
            id: createdEvent ? createdEvent.id : undefined,
          },
        });
      } catch (err) {
        console.error('Failed to cut event', err);
        alert('Failed to cut event. Please try again.');
      }
    },
    async refreshBucketSegment(bucketId, start, end) {
      const bucket = _.find(this.bucketsFromEither, b => b.id === bucketId);
      if (!bucket || !bucket.events) {
        this.update();
        return;
      }

      try {
        const freshEvents = await this.$aw.getEvents(bucketId, {
          starttime: start.toISOString(),
          endtime: end.toISOString(),
        });
        const outsideEvents = bucket.events.filter(evt => {
          const evtStart = moment(evt.timestamp);
          const evtEnd = evtStart.clone().add(evt.duration, 'seconds');
          return evtEnd.isSameOrBefore(start) || evtStart.isSameOrAfter(end);
        });

        bucket.events = [...outsideEvents, ...freshEvents].sort((a, b) => {
          return new Date(a.timestamp).valueOf() - new Date(b.timestamp).valueOf();
        });
      } catch (err) {
        console.warn('Failed to refresh bucket after cut', err);
      }

      this.update();
    },
    async handleGlue(selection) {
      if (!this.pendingSelection) {
        this.pendingSelection = selection;
        return;
      }

      const first = this.pendingSelection;
      const second = selection;

      if (first.bucketId !== second.bucketId) {
        alert('Events must be in the same bucket to glue.');
        this.pendingSelection = null;
        return;
      }
      if (first.event.id === second.event.id) {
        this.pendingSelection = null;
        return;
      }

      // Ensure order: first should start before second
      const firstEventFull = await this.$aw.getEvent(first.bucketId, first.event.id);
      const secondEventFull = await this.$aw.getEvent(second.bucketId, second.event.id);
      const firstStart = moment(firstEventFull.timestamp);
      const firstEnd = firstStart.clone().add(firstEventFull.duration, 'seconds');
      const secondStart = moment(secondEventFull.timestamp);
      const secondEnd = secondStart.clone().add(secondEventFull.duration, 'seconds');

      // Check consecutiveness (allow tiny overlap/gap tolerance)
      const diffSeconds = secondStart.diff(firstEnd, 'seconds', true);
      const tolerance = 0.5;
      if (diffSeconds < -tolerance || diffSeconds > tolerance) {
        alert(
          'Events must be consecutive (no significant gap or overlap) to glue. You might need to use the "grow" tool before "glue".'
        );
        this.pendingSelection = null;
        return;
      }

      // Glue: extend first to end of second, delete second
      const updatedFirst = _.cloneDeep(firstEventFull);
      updatedFirst.duration = secondEnd.diff(firstStart, 'seconds', true);

      try {
        await this.$aw.replaceEvent(first.bucketId, updatedFirst);
        await this.$aw.deleteEvent(second.bucketId, secondEventFull.id);
        await this.refreshBucketSegment(first.bucketId, firstStart, secondEnd);

        this.pushUndoEntry({
          type: 'glue',
          bucketId: first.bucketId,
          rangeStart: firstStart.toISOString(),
          rangeEnd: secondEnd.toISOString(),
          firstOriginal: _.cloneDeep(firstEventFull),
          secondOriginal: _.cloneDeep(secondEventFull),
        });
      } catch (err) {
        console.error('Failed to glue events', err);
        alert('Failed to glue events. Please try again.');
      } finally {
        this.pendingSelection = null;
      }
    },
    async handleClone(selection) {
      if (!this.pendingSelection) {
        this.pendingSelection = selection;
        return;
      }

      const source = this.pendingSelection;
      const target = selection;

      if (source.bucketId !== target.bucketId) {
        alert('Events must be in the same bucket to clone.');
        this.pendingSelection = null;
        return;
      }
      if (source.event.id === target.event.id) {
        this.pendingSelection = null;
        return;
      }

      const sourceEventFull = await this.$aw.getEvent(source.bucketId, source.event.id);
      const targetEventFull = await this.$aw.getEvent(target.bucketId, target.event.id);
      const updatedTarget = _.cloneDeep(targetEventFull);
      updatedTarget.data = _.cloneDeep(sourceEventFull.data || {});

      try {
        await this.$aw.replaceEvent(target.bucketId, updatedTarget);
        const targetStart = moment(targetEventFull.timestamp);
        const targetEnd = targetStart.clone().add(targetEventFull.duration, 'seconds');
        await this.refreshBucketSegment(target.bucketId, targetStart, targetEnd);

        this.pushUndoEntry({
          type: 'clone',
          bucketId: target.bucketId,
          rangeStart: targetStart.toISOString(),
          rangeEnd: targetEnd.toISOString(),
          originalEvent: _.cloneDeep(targetEventFull),
        });
      } catch (err) {
        console.error('Failed to clone event data', err);
        alert('Failed to clone event data. Please try again.');
      } finally {
        this.pendingSelection = null;
      }
    },
    async handleSwap(selection) {
      if (!this.pendingSelection) {
        this.pendingSelection = selection;
        return;
      }

      const first = this.pendingSelection;
      const second = selection;

      if (first.bucketId !== second.bucketId) {
        alert('Events must be in the same bucket to swap.');
        this.pendingSelection = null;
        return;
      }
      if (first.event.id === second.event.id) {
        this.pendingSelection = null;
        return;
      }

      const firstEventFull = await this.$aw.getEvent(first.bucketId, first.event.id);
      const secondEventFull = await this.$aw.getEvent(second.bucketId, second.event.id);
      const firstStart = moment(firstEventFull.timestamp);
      const firstEnd = firstStart.clone().add(firstEventFull.duration, 'seconds');
      const secondStart = moment(secondEventFull.timestamp);
      const secondEnd = secondStart.clone().add(secondEventFull.duration, 'seconds');

      const rangeStart = moment.min(firstStart, secondStart);
      const rangeEnd = moment.max(firstEnd, secondEnd);

      const updatedFirst = _.cloneDeep(firstEventFull);
      // Swap only the data (which includes title) so the labels move between tiles without the
      // visual positions cancelling out. Timestamps and durations stay with their original events.
      updatedFirst.data = _.cloneDeep(secondEventFull.data);

      const updatedSecond = _.cloneDeep(secondEventFull);
      updatedSecond.data = _.cloneDeep(firstEventFull.data);

      try {
        await this.$aw.replaceEvent(first.bucketId, updatedFirst);
        await this.$aw.replaceEvent(second.bucketId, updatedSecond);
        await this.refreshBucketSegment(first.bucketId, rangeStart, rangeEnd);

        this.pushUndoEntry({
          type: 'swap',
          bucketId: first.bucketId,
          rangeStart: rangeStart.toISOString(),
          rangeEnd: rangeEnd.toISOString(),
          firstOriginal: _.cloneDeep(firstEventFull),
          secondOriginal: _.cloneDeep(secondEventFull),
        });
      } catch (err) {
        console.error('Failed to swap events', err);
        alert('Failed to swap events. Please try again.');
        try {
          await this.$aw.replaceEvent(first.bucketId, firstEventFull);
          await this.$aw.replaceEvent(second.bucketId, secondEventFull);
          await this.refreshBucketSegment(first.bucketId, rangeStart, rangeEnd);
        } catch (rollbackErr) {
          console.warn('Failed to rollback swap attempt', rollbackErr);
        }
      } finally {
        this.pendingSelection = null;
      }
    },
    async handleGrow(selection) {
      if (!this.pendingSelection) {
        this.pendingSelection = selection;
        return;
      }

      const first = this.pendingSelection;
      const second = selection;

      if (first.bucketId !== second.bucketId) {
        alert('Events must be in the same bucket to grow.');
        this.pendingSelection = null;
        return;
      }
      if (first.event.id === second.event.id) {
        this.pendingSelection = null;
        return;
      }

      const firstEventFull = await this.$aw.getEvent(first.bucketId, first.event.id);
      const secondEventFull = await this.$aw.getEvent(second.bucketId, second.event.id);
      const firstStart = moment(firstEventFull.timestamp);
      const firstEnd = firstStart.clone().add(firstEventFull.duration, 'seconds');
      const secondStart = moment(secondEventFull.timestamp);

      const diffSeconds = secondStart.diff(firstEnd, 'seconds', true);
      if (diffSeconds < 0) {
        alert('Second event must start after the first event to grow.');
        this.pendingSelection = null;
        return;
      }

      const updatedFirst = _.cloneDeep(firstEventFull);
      updatedFirst.duration = secondStart.diff(firstStart, 'seconds', true);

      try {
        await this.$aw.replaceEvent(first.bucketId, updatedFirst);
        await this.refreshBucketSegment(first.bucketId, firstStart, secondStart);

        this.pushUndoEntry({
          type: 'grow',
          bucketId: first.bucketId,
          rangeStart: firstStart.toISOString(),
          rangeEnd: moment.max(firstEnd, secondStart).toISOString(),
          originalEvent: _.cloneDeep(firstEventFull),
        });
      } catch (err) {
        console.error('Failed to grow event', err);
        alert('Failed to grow event. Please try again.');
      } finally {
        this.pendingSelection = null;
      }
    },
    async handleShrink(selection) {
      if (!this.pendingSelection) {
        this.pendingSelection = selection;
        return;
      }

      const first = this.pendingSelection;
      const second = selection;

      if (first.bucketId !== second.bucketId) {
        alert('Events must be in the same bucket to shrink.');
        this.pendingSelection = null;
        return;
      }
      if (first.event.id === second.event.id) {
        this.pendingSelection = null;
        return;
      }

      const firstEventFull = await this.$aw.getEvent(first.bucketId, first.event.id);
      const secondEventFull = await this.$aw.getEvent(second.bucketId, second.event.id);
      const firstStart = moment(firstEventFull.timestamp);
      const secondStart = moment(secondEventFull.timestamp);

      if (secondStart.isBefore(firstStart)) {
        alert('Second event must start after the first event to shrink.');
        this.pendingSelection = null;
        return;
      }

      const newDuration = secondStart.diff(firstStart, 'seconds', true);
      if (newDuration <= 0) {
        alert('Cannot shrink: resulting duration would be zero or negative.');
        this.pendingSelection = null;
        return;
      }

      const updatedFirst = _.cloneDeep(firstEventFull);
      updatedFirst.duration = newDuration;

      try {
        await this.$aw.replaceEvent(first.bucketId, updatedFirst);
        await this.refreshBucketSegment(first.bucketId, firstStart, secondStart);

        this.pushUndoEntry({
          type: 'shrink',
          bucketId: first.bucketId,
          rangeStart: firstStart.toISOString(),
          rangeEnd: moment
            .max(firstStart.clone().add(newDuration, 'seconds'), secondStart)
            .toISOString(),
          originalEvent: _.cloneDeep(firstEventFull),
        });
      } catch (err) {
        console.error('Failed to shrink event', err);
        alert('Failed to shrink event. Please try again.');
      } finally {
        this.pendingSelection = null;
      }
    },
    getBucketEvents(bucketId: string) {
      const bucket = _.find(this.bucketsFromEither, b => b.id === bucketId);
      return bucket && bucket.events ? bucket.events : [];
    },
    findEventMatch(bucketId: string, timestamp: Date, duration: number, data: any) {
      const targetTs = moment(timestamp).valueOf();
      const toleranceMs = 500;
      const durationTolerance = 0.001;
      const events = this.getBucketEvents(bucketId);
      return _.find(events, e => {
        const ts = moment(e.timestamp).valueOf();
        const closeInTime = Math.abs(ts - targetTs) <= toleranceMs;
        const closeInDuration = Math.abs(e.duration - duration) <= durationTolerance;
        return closeInTime && closeInDuration && _.isEqual(e.data, data);
      });
    },
    pushUndoEntry(entry: UndoEntry) {
      this.undoStack.push(entry);
      if (this.undoStack.length > this.undoLimit) {
        this.undoStack.shift();
      }
      this.emitUndoAvailability();
    },
    emitUndoAvailability() {
      this.$emit('undo-available', this.undoStack.length > 0);
    },
    async triggerUndo() {
      if (!this.undoStack.length) return;

      const entry = this.undoStack.pop();
      try {
        await this.applyUndo(entry);
      } catch (err) {
        console.error('Failed to undo last action', err);
        alert('Failed to undo last change. Please try again.');
        this.undoStack.push(entry);
      } finally {
        this.emitUndoAvailability();
      }
    },
    async applyUndo(entry?: UndoEntry) {
      if (!entry) return;
      if (entry.type === 'cut') {
        await this.undoCut(entry);
      } else if (entry.type === 'glue') {
        await this.undoGlue(entry);
      } else if (entry.type === 'grow' || entry.type === 'shrink') {
        await this.undoResize(entry);
      } else if (entry.type === 'clone') {
        await this.undoClone(entry);
      } else if (entry.type === 'swap') {
        await this.undoSwap(entry);
      }
    },
    async undoCut(entry: Extract<UndoEntry, { type: 'cut' }>) {
      try {
        await this.$aw.replaceEvent(entry.bucketId, entry.originalEvent);
        if (entry.newEventSignature && entry.newEventSignature.id !== undefined) {
          await this.$aw.deleteEvent(entry.bucketId, entry.newEventSignature.id);
        } else if (entry.newEventSignature) {
          const matching = this.findEventMatch(
            entry.bucketId,
            entry.newEventSignature.timestamp,
            entry.newEventSignature.duration,
            entry.newEventSignature.data
          );
          if (matching && matching.id !== undefined) {
            await this.$aw.deleteEvent(entry.bucketId, matching.id);
          }
        }
      } catch (err) {
        console.error('Failed to undo cut', err);
        throw err;
      }

      await this.refreshBucketSegment(
        entry.bucketId,
        moment(entry.rangeStart),
        moment(entry.rangeEnd)
      );
    },
    async undoClone(entry: Extract<UndoEntry, { type: 'clone' }>) {
      try {
        await this.$aw.replaceEvent(entry.bucketId, entry.originalEvent);
      } catch (err) {
        console.error('Failed to undo clone', err);
        throw err;
      }

      await this.refreshBucketSegment(
        entry.bucketId,
        moment(entry.rangeStart),
        moment(entry.rangeEnd)
      );
    },
    async undoGlue(entry: Extract<UndoEntry, { type: 'glue' }>) {
      try {
        await this.$aw.replaceEvent(entry.bucketId, entry.firstOriginal);
        const { id, ...secondNoId } = entry.secondOriginal as EventWithId;
        await this.$aw.insertEvent(entry.bucketId, secondNoId as IEvent);
      } catch (err) {
        console.error('Failed to undo glue', err);
        throw err;
      }

      await this.refreshBucketSegment(
        entry.bucketId,
        moment(entry.rangeStart),
        moment(entry.rangeEnd)
      );
    },
    async undoResize(entry: Extract<UndoEntry, { type: 'grow' | 'shrink' }>) {
      try {
        await this.$aw.replaceEvent(entry.bucketId, entry.originalEvent);
      } catch (err) {
        console.error('Failed to undo resize', err);
        throw err;
      }

      await this.refreshBucketSegment(
        entry.bucketId,
        moment(entry.rangeStart),
        moment(entry.rangeEnd)
      );
    },
    async undoSwap(entry: Extract<UndoEntry, { type: 'swap' }>) {
      try {
        await this.$aw.replaceEvent(entry.bucketId, entry.firstOriginal);
        await this.$aw.replaceEvent(entry.bucketId, entry.secondOriginal);
      } catch (err) {
        console.error('Failed to undo swap', err);
        throw err;
      }

      await this.refreshBucketSegment(
        entry.bucketId,
        moment(entry.rangeStart),
        moment(entry.rangeEnd)
      );
    },
    ensureUpdate() {
      // Will only run update() if data available and never ran before
      if (!this.updateHasRun) {
        this.update();
      }
    },
    update() {
      // Used by unsureUpdate to check if ran
      this.updateHasRun = true;

      // Build groups
      const buckets = this.bucketsFromEither;
      let groups = _.map(buckets, bucket => {
        // If bucket id is not set, then if only one bucket is given, assume result of a search/query and set a constant placeholder one.
        // Otherwise, log a warning.
        if (bucket.id === undefined) {
          if (buckets.length === 1) {
            bucket.id = 'events';
          } else {
            console.warn(
              'Bucket id is not set, but there are multiple buckets. This is not supported.'
            );
          }
        }
        return { id: bucket.id, content: this.showRowLabels ? bucket.id : '' };
      });

      // Build items
      const items = _.map(this.chartData, (item, i) => {
        const bgColor = item.color;
        const borderColor = Color(bgColor).darken(0.3);
        return {
          id: String(i),
          group: item.bucketId,
          content: item.title,
          title: item.tooltip,
          start: moment(item.start),
          end: moment(item.end),
          style: `background-color: ${bgColor}; border-color: ${borderColor}`,
          subgroup: item.swimlane,
        };
      });

      if (groups.length > 0 && items.length > 0) {
        if (this.queriedInterval && this.showQueriedInterval) {
          const duration = this.queriedInterval[1].diff(this.queriedInterval[0], 'seconds');
          groups.push({ id: String(groups.length), content: 'queried interval' });
          items.push({
            id: String(items.length + 1),
            group: groups.length - 1,
            title: buildTooltip(
              { type: 'test' },
              {
                timestamp: this.queriedInterval[0],
                duration: duration,
                data: { title: 'test' },
              }
            ),
            content: 'query',
            start: this.queriedInterval[0],
            end: this.queriedInterval[1],
            style: 'background-color: #aaa; height: 10px',
            subgroup: ``,
          });
        }

        if (this.updateTimelineWindow) {
          const start =
            (this.queriedInterval && this.queriedInterval[0]) ||
            _.min(_.map(items, item => item.start));
          const end =
            (this.queriedInterval && this.queriedInterval[1]) ||
            _.max(_.map(items, item => item.end));
          this.options.min = start;
          this.options.max = end;
          this.timeline.setOptions(this.options);
          this.timeline.setWindow(start, end);
        }

        // Hide buckets with no events in the queried range
        const count = _.countBy(items, i => i.group);
        groups = _.filter(groups, g => {
          return count[g.id] && count[g.id] > 0;
        });
        this.timeline.setData({ groups: groups, items: items });

        this.items = items;
        this.groups = groups;
      } else {
        // update the timeline range
        this.options.min = this.queriedInterval[0];
        this.options.max = this.queriedInterval[1];
        this.timeline.setOptions(this.options);
        this.timeline.setWindow(this.queriedInterval[0], this.queriedInterval[1]);

        // clear the data
        this.timeline.setData({ groups: [], items: [] });
        this.items = [];
        this.groups = [];
      }
    },
  },
};
</script>
