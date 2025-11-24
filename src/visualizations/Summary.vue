<template lang="pug">
div
  div.aw-summary-wrapper
    div.aw-summary-container(ref="container")
    div.aw-summary-actions(v-if="categorizeMode && limitedFields.length" :style="{ height: overlayHeight + 'px' }")
      div.aw-summary-action-row(v-for="(entry, idx) in limitedFields" :key="idx" v-if="isUncategorized(entry)" :style="rowStyle(idx)")
        div.aw-summary-action-buttons
          b-button.mr-2.btn-xs(variant="success" @click="startCreateRule(entry)")
            | New rule
          b-button.btn-xs(variant="warning" @click="startAppendRule(entry)")
            | Append rule
  // Use visibility to make sure elements don't skip when data finishes loading
  div(:style='{"visibility": visible_more ? "visible" : "hidden"}')
    b-button.mt-1.mr-2(v-if="fields && (limit_ < fields.length)", size="sm", variant="outline-secondary", @click="limit_ += 5")
      icon(name="angle-double-down")
      | Show more
    b-button.mt-1(v-if="limit_ != limit" size="sm", variant="outline-secondary", @click="limit_ = limit")
      icon(name="angle-double-up")

  div(v-if="create.categoryId !== null")
    CategoryEditModal(:categoryId="create.categoryId",
                      @ok="createRuleOk()"
                      @hidden="createRuleCancel()")

  b-modal(:id="appendModalId" title="Append rule" @ok="handleAppendOk" :ok-disabled="!validAppend")
    b-form(ref="form" @submit.stop.prevent="handleAppendSubmit")
      b-form-group(label="Category"
                   label-for="append-category"
                   invalid-feedback="Category is required"
                   :state="validAppendCategory"
                   required)
        b-form-select#append-category(v-model="append.category")
          b-form-select-option(v-for="cat in allCategoriesSelect" :value="cat.value" :key="cat.id") {{ cat.text }}
      b-form-group(label="Pattern")
        b-form-input(v-model="append.pattern" readonly)
</template>

<style scoped lang="scss">
.aw-summary-container > svg {
  border: 1px solid #999;
  border-radius: 0.5em;
}

.aw-summary-wrapper {
  position: relative;
}

.aw-summary-actions {
  position: absolute;
  inset: 0;
  pointer-events: none;
  z-index: 2;
}

.aw-summary-action-row {
  position: absolute;
  left: 0;
  width: 100%;
  display: flex;
  justify-content: flex-end;
  align-items: end;
  height: 46px;
  padding-right: 0.25rem;
  pointer-events: none;
}

.aw-summary-action-buttons {
  pointer-events: auto;
}

.btn-xs {
  font-size: 0.65rem;
  padding: 0.1rem 0.35rem;
  line-height: 1.2;
}
</style>

<script lang="ts">
// NOTE: This is just a Vue.js component wrapper for summary.ts
//       Code should generally go in the framework-independent file.

import _ from 'lodash';
import { mapState } from 'pinia';
import summary, { SUMMARY_ENTRY_HEIGHT, SUMMARY_BAR_GAP } from './summary';
import 'vue-awesome/icons/angle-double-down';
import 'vue-awesome/icons/angle-double-up';
import { useCategoryStore } from '~/stores/categories';
import CategoryEditModal from '~/components/CategoryEditModal.vue';

export default {
  name: 'aw-summary',
  components: { CategoryEditModal },
  props: {
    fields: Array,
    namefunc: Function,
    hoverfunc: {
      type: Function,
      default: null, // If not set we will default to namefunc
    },
    colorfunc: Function,
    linkfunc: {
      type: Function,
      default: () => null,
    },
    limit: {
      type: Number,
      default: 5,
    },
    with_limit: {
      type: Boolean,
      default: false,
    },
    categorizeMode: {
      type: Boolean,
      default: false,
    },
  },
  data: function () {
    return {
      limit_: this.limit,
      categoryStore: useCategoryStore(),
      append: {
        pattern: '',
        category: [],
      },
      create: {
        categoryId: null,
      },
      appendModalId: 'appendRule-' + Math.random().toString(36).slice(2),
    };
  },
  computed: {
    visible_more() {
      return this.fields && this.fields.length > 0 && this.with_limit;
    },
    limitedFields() {
      return this.fields ? this.fields.slice(0, this.limit_) : [];
    },
    overlayHeight() {
      if (!this.limitedFields.length) return 0;
      return this.limitedFields.length * SUMMARY_ENTRY_HEIGHT - SUMMARY_BAR_GAP;
    },
    ...mapState(useCategoryStore, ['allCategoriesSelect']),
    validAppendCategory() {
      return this.append.category.length > 0;
    },
    validAppend() {
      return this.validAppendCategory && this.append.pattern !== '';
    },
  },
  watch: {
    fields: function () {
      this.update();
    },
    limit_: function () {
      this.update();
    },
  },
  mounted: function () {
    if (this.categoryStore.classes.length === 0) {
      this.categoryStore.load();
    }
    const el = this.$refs.container as HTMLElement;
    summary.create(el);
    this.update();
  },
  methods: {
    update: function () {
      const el = this.$refs.container as HTMLElement;
      if (this.fields) {
        summary.updateSummedEvents(
          el,
          this.fields.slice(0, this.limit_),
          this.namefunc,
          this.hoverfunc,
          this.colorfunc,
          this.linkfunc
        );
      } else {
        summary.set_status(el, 'Loading...');
      }
    },
    rowStyle(idx: number) {
      return { top: `${idx * SUMMARY_ENTRY_HEIGHT}px` };
    },
    isUncategorized(entry: any) {
      const cat = entry && entry.data ? entry.data.$category : null;
      if (!cat || (Array.isArray(cat) && cat.length === 0)) return true;
      if (Array.isArray(cat)) return cat[0] === 'Uncategorized';
      return cat === 'Uncategorized';
    },
    entryName(entry: any) {
      return this.namefunc(entry);
    },
    startCreateRule(entry: any) {
      const pattern = _.escapeRegExp(this.entryName(entry));
      this.categoryStore.addClass({
        name: [this.entryName(entry)],
        rule: { type: 'regex', regex: pattern },
      });

      const lastId = _.max(_.map(this.categoryStore.classes, 'id'));
      this.create.categoryId = lastId;
    },
    async createRuleOk() {
      await this.categoryStore.save();
      this.create.categoryId = null;
    },
    async createRuleCancel() {
      this.create.categoryId = null;
      this.categoryStore.load();
    },
    startAppendRule(entry: any) {
      this.append.pattern = _.escapeRegExp(this.entryName(entry));
      // Select first option by default if none set
      if (!this.append.category.length && this.allCategoriesSelect.length > 0) {
        this.append.category = this.allCategoriesSelect[0].value;
      }
      this.$bvModal.show(this.appendModalId);
    },
    async appendRuleOk() {
      const cat = this.categoryStore.get_category(this.append.category);
      this.categoryStore.appendClassRule(cat.id, this.append.pattern);
      await this.categoryStore.save();
      this.append = { pattern: '', category: [] };
    },
    handleAppendOk(bvModalEvent) {
      bvModalEvent.preventDefault();
      this.handleAppendSubmit(bvModalEvent);
    },
    handleAppendSubmit(e) {
      if (!this.validAppend) {
        e.preventDefault();
        return;
      }

      this.$nextTick(() => {
        this.$bvModal.hide(this.appendModalId);
      });

      this.appendRuleOk();
    },
  },
};
</script>
