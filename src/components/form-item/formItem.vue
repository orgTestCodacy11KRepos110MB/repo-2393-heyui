<template>
  <div :class="formItemCls" :prop="prop" :validable="validable">
    <label v-if="showLabel" :style="labelStyleCls" class="h-form-item-label">
      <i v-if="icon" :class="{ [icon]: true, 'h-form-item-label-icon': true }"></i>
      <span v-if="!$slots.label">{{ label }}</span
      ><slot v-else :label="label" name="label"></slot>
    </label>
    <div class="h-form-item-content" :style="contentStyleCls">
      <div class="h-form-item-wrap">
        <slot></slot>
      </div>
      <div v-if="!errorMessage.valid" class="h-form-item-error">
        <span v-if="errorMessage.type == 'base'" class="h-form-item-error-label">{{ label }}</span
        ><span class="h-form-item-error-message">{{ errorMessage.message }}</span
        ><slot name="error" :type="errorMessage.type"></slot>
      </div>
    </div>
  </div>
</template>
<script>
import utils from 'heyui/utils/utils';

const prefixCls = 'h-form-item';
export default {
  name: 'HFormItem',
  inject: ['validField', 'removeProp', 'requireds', 'setConfig', 'updateProp', 'updateErrorMessage', 'labelWidth', 'params'],
  props: {
    label: String,
    prop: String,
    icon: String,
    required: {
      type: Boolean,
      default: false
    },
    readonly: {
      type: Boolean,
      default: false
    },
    showLabel: {
      type: Boolean,
      default: true
    },
    single: {
      type: Boolean,
      default: false
    },
    validable: {
      type: Boolean,
      default: true
    },
    noPadding: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      validResult: null,
      errorMessage: { valid: true }
    };
  },
  computed: {
    configRequired() {
      if (!this.prop) return false;
      const ruleKey = this.prop.replace(/\[\w+\]/, '[]');
      return this.requireds.indexOf(this.prop) > -1 || this.requireds.indexOf(ruleKey) > -1;
    },
    initLabelWidth() {
      let mode = this.params.mode;
      let hasWidth = !(mode == 'block' || mode == 'inline') || (this.single && mode == 'twocolumn');
      let width = hasWidth ? this.labelWidth || false : false;
      return width ? `${width}px` : 'auto';
    },
    formItemCls() {
      return {
        [`${prefixCls}`]: true,
        [`${prefixCls}-single`]: this.single,
        [`${prefixCls}-readonly`]: !!this.readonly,
        [`${prefixCls}-required`]: this.required || this.configRequired,
        [`${prefixCls}-valid-error`]: !this.errorMessage.valid,
        [`${prefixCls}-no-padding`]: !!this.noPadding
      };
    },
    labelCls() {
      return {
        [`${prefixCls}-label`]: true
      };
    },
    labelStyleCls() {
      let param = {};
      if (this.params.mode != 'block') {
        param.width = this.initLabelWidth;
      } else {
        param.width = '100%';
      }
      return param;
    },
    contentStyleCls() {
      let param = {};
      if (this.params.mode == 'block' || !this.showLabel) {
        param['margin-left'] = '0px';
      } else {
        param['margin-left'] = this.initLabelWidth;
      }
      return param;
    }
  },
  watch: {
    prop(prop, oldProp) {
      if (this.prop) {
        this.errorMessage = this.updateProp(prop, oldProp);
      }
    },
    label() {
      if (this.prop) {
        this.errorMessage = this.updateErrorMessage(this.prop, this.label);
      }
    },
    required() {
      this.setConfig(this.prop, { required: this.required });
    }
  },
  beforeUnmount() {
    if (this.prop && this.required) {
      this.removeProp(this.prop);
    }
  },
  mounted() {
    if (this.prop) {
      if (this.required) {
        this.setConfig(this.prop, { required: this.required });
      }
      this.errorMessage = this.updateErrorMessage(this.prop, this.label);
    }
  },
  methods: {
    reset() {
      console.warn('[HeyUI WARNING] FormItem Component:  FormItem.reset() will be decapitated, please use method FormItem.resetValid()');
      this.errorMessage.valid = true;
    },
    resetValid() {
      this.errorMessage.valid = true;
    },
    trigger() {
      let prop = this.prop;
      if (!this.validable || utils.isNull(prop)) {
        return;
      }
      this.validField(prop);
    }
  }
};
</script>
