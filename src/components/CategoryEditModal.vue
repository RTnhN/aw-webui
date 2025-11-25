<template lang="pug">
// The category edit modal
b-modal(id="edit" ref="edit" title="Edit category" @show="resetModal" @hidden="hidden" @ok="handleOk")
  div.my-1
    b-input-group.my-1(prepend="Name")
      b-form-input(v-model="editing.name")
    b-input-group(prepend="Parent")
      b-select(v-model="editing.parent", :options="allCategories")
    //| ID: {{editing.id}}

  hr
  div.my-1
    b Rule
    b-input-group.my-1(prepend="Type")
      b-select(v-model="editing.rule.type", :options="allRuleTypes")
    div(v-if="editing.rule.type === 'regex'")
      b-input-group.my-1(prepend="Pattern")
        b-form-input(v-model="editing.rule.regex")
    div(v-else-if="editing.rule.type === 'regex_list'")
      div(v-if="editing.rule.regex_list && editing.rule.regex_list.length === 0")
        small.text-muted Add one or more patterns to build the combined regex.
      div(v-for="(item, index) in editing.rule.regex_list" :key="`regex-item-${index}`")
        b-input-group.my-1
          b-form-input(v-model="editing.rule.regex_list[index]" placeholder="Pattern")
          b-input-group-append
            b-button(size="sm" variant="outline-danger" @click="removeRegexListItem(index)")
              | Delete
      b-button(size="sm" variant="outline-primary" @click="addRegexListItem") Add pattern
    div(v-if="editing.rule.type === 'regex' || editing.rule.type === 'regex_list'")
      div.d-flex.mt-1
        div.flex-grow-1
          b-form-checkbox(v-model="editing.rule.ignore_case" switch)
            | Case insensitive
        div.flex-grow-1
          small.text-right
            //div(v-if="valid" style="color: green") Valid
            div(v-if="!validPattern" style="color: red") Invalid pattern
            div(v-if="validPattern && broad_pattern" style="color: orange") Pattern too broad

  hr
  div.my-1
    b Color

    b-form-checkbox(v-model="editing.inherit_color" switch)
      | Inherit parent color
    div.mt-1(v-show="!editing.inherit_color")
      color-picker(v-model="editing.color")

  hr
  div.my-1
    b Productivity score
    b-form-checkbox(v-model="editing.inherit_score" switch)
      | Inherit parent score
    b-input-group.my-1(prepend="Score" v-if="!editing.inherit_score")
      b-form-input(v-model="editing.score")

  hr
  div.my-1
    b-btn(variant="danger", @click="removeClass(categoryId); $refs.edit.hide()")
      icon(name="trash")
      | Remove category
</template>

<script lang="ts">
import _ from 'lodash';
import ColorPicker from '~/components/ColorPicker.vue';
import { useCategoryStore } from '~/stores/categories';
import { mapState } from 'pinia';
import { validateRegex, isRegexBroad } from '~/util/validate';

import 'vue-awesome/icons/trash';

export default {
  name: 'CategoryEditModal',
  components: {
    'color-picker': ColorPicker,
  },
  props: {
    categoryId: { type: Number, required: true },
  },
  data: function () {
    return {
      categoryStore: useCategoryStore(),

      editing: {
        id: 0, // FIXME: Use ID assigned to category in store, in order for saves to be uniquely targeted
        name: null,
        rule: { type: 'none', regex: '', regex_list: [], ignore_case: false },
        parent: [],
        inherit_color: true,
        color: null,
        inherit_score: true,
        score: null,
      },
    };
  },
  computed: {
    ...mapState(useCategoryStore, {
      allCategories: state => [{ value: [], text: 'None' }].concat(state.allCategoriesSelect),
    }),
    allRuleTypes: function () {
      return [
        { value: 'none', text: 'None' },
        { value: 'regex', text: 'Regular Expression' },
        { value: 'regex_list', text: 'Regex list' },
        //{ value: 'glob', text: 'Glob pattern' },
      ];
    },
    valid: function () {
      return this.editing.rule.type === 'none' || this.validPattern;
    },
    validPattern: function () {
      if (this.editing.rule.type === 'regex') {
        return validateRegex(this.ruleRegex);
      }
      if (this.editing.rule.type === 'regex_list') {
        return this.ruleRegex.length > 0 && validateRegex(this.ruleRegex);
      }
      return true;
    },
    broad_pattern: function () {
      return (
        (this.editing.rule.type === 'regex' || this.editing.rule.type === 'regex_list') &&
        isRegexBroad(this.ruleRegex)
      );
    },
    ruleRegex: function () {
      if (this.editing.rule.type === 'regex_list') {
        return this.getRegexFromList();
      }
      return this.editing.rule.regex || '';
    },
  },
  watch: {
    categoryId: function (new_value) {
      if (new_value !== null) {
        this.showModal();
      }
    },
    'editing.rule.type': function (new_value, old_value) {
      if (new_value === 'regex_list') {
        this.ensureRegexListFromRegex();
      } else if (old_value === 'regex_list' && new_value === 'regex') {
        this.editing.rule.regex = this.getRegexFromList();
      }
    },
  },
  mounted: function () {
    if (this.categoryId !== null) {
      this.showModal();
    }
  },
  methods: {
    showModal() {
      this.$refs.edit.show();
    },
    hidden() {
      this.$emit('hidden');
    },
    removeClass() {
      // TODO: Show a confirmation dialog
      // TODO: Remove children as well?
      this.categoryStore.removeClass(this.categoryId);
    },
    checkFormValidity() {
      return this.valid;
    },
    handleOk(event) {
      // Prevent modal from closing
      event.preventDefault();
      // Trigger submit handler
      this.handleSubmit();
      this.$emit('ok');
    },
    handleSubmit() {
      // Exit when the form isn't valid
      if (!this.checkFormValidity()) {
        return;
      }

      // Save the category
      const rule =
        this.editing.rule.type === 'none'
          ? { type: 'none' }
          : this.editing.rule.type === 'regex_list'
          ? {
              type: 'regex',
              regex: this.getRegexFromList(),
              ignore_case: this.editing.rule.ignore_case,
            }
          : this.editing.rule;

      const new_class = {
        id: this.editing.id,
        name: this.editing.parent.concat(this.editing.name),
        rule,
        data: {
          color: this.editing.inherit_color === true ? undefined : this.editing.color,
          score: this.editing.inherit_score === true ? undefined : this.editing.score,
        },
      };
      this.categoryStore.updateClass(new_class);

      // Hide the modal manually
      this.$nextTick(() => {
        this.$refs.edit.hide();
      });
    },
    resetModal() {
      const cat = this.categoryStore.get_category_by_id(this.categoryId);
      const color = cat.data ? cat.data.color : undefined;
      const inherit_color = !color;
      const score = cat.data ? cat.data.score : undefined;
      const inherit_score = !score;
      const rule = _.cloneDeep(cat.rule);
      if (rule.type === 'regex') {
        rule.regex_list = rule.regex ? rule.regex.split('|') : [];
      }
      this.editing = {
        id: cat.id,
        name: cat.subname,
        rule,
        parent: cat.parent ? cat.parent : [],
        color,
        inherit_color,
        score,
        inherit_score,
      };
    },
    ensureRegexListFromRegex() {
      this.ensureRegexListExists();
      const parts =
        this.editing.rule.regex && this.editing.rule.regex.length > 0
          ? this.editing.rule.regex.split('|')
          : [];
      this.editing.rule.regex_list.splice(0, this.editing.rule.regex_list.length, ...parts);
      if (this.editing.rule.regex_list.length === 0) {
        this.editing.rule.regex_list.push('');
      }
    },
    ensureRegexListExists() {
      if (!this.editing.rule.regex_list) {
        this.$set(this.editing.rule, 'regex_list', []);
      }
    },
    addRegexListItem() {
      this.ensureRegexListExists();
      this.editing.rule.regex_list.push('');
    },
    removeRegexListItem(index: number) {
      this.editing.rule.regex_list.splice(index, 1);
      if (this.editing.rule.regex_list.length === 0) {
        this.editing.rule.regex_list.push('');
      }
    },
    getRegexFromList(): string {
      const list = this.editing.rule.regex_list || [];
      return list.filter(r => r !== '').join('|');
    },
  },
};
</script>
