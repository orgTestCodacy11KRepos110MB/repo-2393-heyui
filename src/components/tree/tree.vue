<template>
  <div :class="treeCls">
    <Search v-if="filterable" v-model="searchValue" block @search="searchTree"></Search>
    <ul class="h-tree-body">
      <treeItem
        v-for="tree of treeDatas"
        :key="tree.key"
        :data="tree"
        :param="param"
        :multiple="multiple"
        :status="status"
        :choose-mode="chooseMode"
        :toggle-on-select="toggleOnSelect"
        :select-on-click="selectOnClick"
        :level="0"
        @trigger="trigger"
      ></treeItem>
    </ul>
    <Loading :loading="globalloading"></Loading>
  </div>
</template>
<script>
import config from 'heyui/utils/config';
import utils from 'heyui/utils/utils';
import treeItem from './treeitem';
import Search from 'heyui/components/search';

const prefix = 'h-tree';

const updateChildStatus = (data, column, value) => {
  if (data.children) {
    for (let child of data.children) {
      child.status[column] = value;
      updateChildStatus(child, column, value);
    }
  }
};

const updateParentStatus = (objs, data, column, value) => {
  let parent = objs[data.parentKey];
  if (!utils.isNull(data.parentKey) && parent) {
    parent.status[column] = value;
    updateParentStatus(objs, parent, column, value);
  }
};

const updateModeAllChildChooseStatus = data => {
  if (data.children) {
    let isIndeterminateStatus = false;
    let isChoose = !!data.children.length;
    for (let child of data.children) {
      updateModeAllChildChooseStatus(child);
      if (child.status.choose || child.status.indeterminate) {
        isIndeterminateStatus = true;
      }
      if (!child.status.choose) {
        isChoose = false;
      }
    }
    if (!data.status.choose) {
      if (isChoose) {
        data.status.choose = true;
      } else if (isIndeterminateStatus) {
        data.status.indeterminate = true;
      }
    }
  }
};

const getChooseNode = (data, options) => {
  if (data.status.choose) {
    options.push(data.value);
  } else {
    for (let child of data.children) {
      getChooseNode(child, options);
    }
  }
  return options;
};

export default {
  name: 'HTree',
  components: {
    treeItem,
    Search
  },
  props: {
    option: Object,
    multiple: {
      type: Boolean,
      default: false
    },
    filterable: {
      type: Boolean,
      default: false
    },
    chooseMode: {
      type: String,
      default: 'all' // independent, some, all
    },
    modelValue: [Number, String, Array, Object],
    config: String,
    toggleOnSelect: {
      type: Boolean,
      default: true
    },
    selectOnClick: {
      type: Boolean,
      default: false
    },
    className: {
      type: String,
      default: 'h-tree-theme-item-selected'
    }
  },
  emits: ['open', 'select', 'choose', 'update:modelValue', 'change', 'loadDataSuccess'],
  data() {
    return {
      updateFromInput: false,
      globalloading: false,
      loading: true,
      status: {
        selected: null
      },
      treeDatas: [],
      treeObj: {},
      searchValue: null
    };
  },
  computed: {
    param() {
      if (this.config) {
        return utils.extend({}, config.getOption('tree.default'), config.getOption(`tree.configs.${this.config}`), this.option);
      } else {
        return utils.extend({}, config.getOption('tree.default'), this.option);
      }
    },
    treeCls() {
      return {
        [prefix]: true,
        [`${prefix}-multiple`]: this.multiple,
        [`${prefix}-single`]: !this.multiple,
        [this.className]: !!this.className
      };
    }
  },
  watch: {
    modelValue() {
      if (this.updateFromInput) {
        this.updateFromInput = false;
        return;
      }
      this.parse();
    },
    'option.datas': function() {
      this.initTreeDatas();
    }
  },
  mounted() {
    this.initTreeDatas();
    this.parse();
  },
  methods: {
    parse() {
      if (this.multiple) {
        this.updateChoose(this.modelValue, false);
      } else {
        this.updateSelect(this.modelValue, false);
      }
    },
    updateTreeItem(key, value) {
      let item = this.treeObj[key];
      if (item) {
        for (let v of Object.keys(value)) {
          item.value[v] = value[v];
          if (v == this.param.titleName) {
            item.title = value[v];
          }
        }
      }
      this.$forceUpdate();
    },
    appendTreeItem(key, value) {
      let parent = this.treeObj[key];
      let obj = this.initTreeNode(value, key);
      if (parent) {
        parent.children.push(obj);
      } else {
        this.treeDatas.push(obj);
      }
      this.treeObj[obj.key] = obj;
    },
    removeTreeItem(key) {
      let item = this.treeObj[key];
      if (item) {
        let index = this.treeDatas.indexOf(item);
        if (index > -1) {
          this.treeDatas.splice(index, 1);
        } else if (item.parentKey && this.treeObj[item.parentKey]) {
          let parent = this.treeObj[item.parentKey];
          if (parent.children.indexOf(item) > -1) {
            parent.children.splice(parent.children.indexOf(item), 1);
          }
        }
        delete this.treeObj[key];
      }
    },
    searchTree(value) {
      this.searchValue = value;
      if (!utils.isNull(this.searchValue) && this.searchValue !== '') {
        let searchValue = this.searchValue.toLowerCase();
        for (let key of Object.keys(this.treeObj)) {
          let tree = this.treeObj[key];
          tree.status.hide = (tree.html || tree.title || '').toLowerCase().indexOf(searchValue) == -1;
        }
        this.expandAll();
      } else {
        for (let key of Object.keys(this.treeObj)) {
          let tree = this.treeObj[key];
          tree.status.hide = false;
        }
      }
    },
    trigger({ type, data, update }) {
      if (update) Object.assign(data.status, update);
      if (type == 'toggleTreeEvent') {
        data.status.opened = !data.status.opened;
        this.$emit('open', data.value);
      } else if (type == 'loadDataEvent') {
        if (utils.isFunction(this.param.getDatas)) {
          data.status.loading = true;
          this.param.getDatas.call(
            this.param,
            data.value,
            result => {
              data.children = this.initTreeModeData(result, true);
              data.status.isWait = false;
              data.status.loading = false;
              data.status.opened = true;
            },
            () => {
              data.status.loading = false;
            }
          );
        }
      } else if (type == 'selectEvent') {
        if (!this.multiple) {
          this.status.selected = data.key;
          this.$emit('select', data.value);
          this.setvalue();
        }
      } else if (type == 'chooseEvent') {
        let choose = data.status.choose;
        if (this.chooseMode != 'independent') {
          updateChildStatus(data, 'choose', choose);
        }
        this.$emit('choose', this.getChoose());
        if (this.multiple) this.setvalue();
      }
    },
    initTreeDatas() {
      let datas = [];
      if (utils.isArray(this.param.datas)) {
        datas = this.param.datas;
      } else if (utils.isFunction(this.param.datas)) {
        datas = this.param.datas.apply(this.param);
      }
      if (utils.isFunction(this.param.getTotalDatas) || utils.isFunction(this.param.getDatas)) {
        datas = [];
        this.globalloading = true;
        let loadData = this.param.getTotalDatas || this.param.getDatas;
        let param = [
          result => {
            this.treeDatas = this.initDatas(utils.copy(result));
            this.parse();
            this.globalloading = false;
            this.$emit('loadDataSuccess');
          },
          () => {
            this.globalloading = false;
          }
        ];
        if (this.param.getDatas) {
          param.unshift(null);
        }
        loadData.apply(this.param, param);
      }
      this.treeDatas = this.initDatas(datas);
      this.parse();
    },
    initDatas(datas) {
      let list = (datas = utils.copy(datas));
      if (this.param.dataMode == 'list' && datas.length > 0) {
        list = utils.generateTree(datas, this.param);
      }
      let isWait = utils.isFunction(this.param.getDatas);
      return this.initTreeModeData(list, isWait);
    },
    initTreeModeData(list, isWait, parentKey) {
      let datas = [];
      for (let data of list) {
        let obj = this.initTreeNode(data, parentKey, isWait);
        let children = data[this.param.childrenName] || [];
        obj.children = this.initTreeModeData(children, isWait, obj.key);
        this.treeObj[obj.key] = obj;
        datas.push(obj);
      }
      return datas;
    },
    initTreeNode(data, parentKey, isWait = false) {
      return {
        key: data[this.param.keyName],
        title: data[this.param.titleName],
        value: data,
        parentKey,
        icon: data.treeIcon,
        status: {
          hide: false,
          opened: false,
          loading: false,
          checkable: data.checkable !== false,
          isWait,
          selected: false,
          indeterminate: false,
          choose: false,
          disabled: !!data.disabled
        },
        children: []
      };
    },
    refresh() {
      this.initTreeDatas();
    },
    expandAll() {
      for (let tree of Object.keys(this.treeObj)) {
        this.treeObj[tree].status.opened = true;
      }
    },
    foldAll() {
      for (let tree of Object.keys(this.treeObj)) {
        this.treeObj[tree].status.opened = false;
      }
    },
    expand(ids) {
      for (let key of Object.keys(this.treeObj)) {
        let tree = this.treeObj[key];
        tree.status.opened = ids.indexOf(tree.key) > -1;
      }
    },
    chooseAll() {
      for (let key in this.treeObj) {
        this.treeObj[key].status.choose = true;
      }
      this.setvalue();
    },
    updateSelect(key, updateValue = true) {
      if (key === null) {
        this.status.selected = null;
      } else {
        let option = this.treeObj[key];
        if (option) {
          this.status.selected = key;
          updateParentStatus(this.treeObj, option, 'opened', true);
        }
      }
      if (updateValue) {
        this.setvalue();
      }
    },
    getSelect() {
      if (utils.isNull(this.status.selected)) {
        return null;
      }
      let option = this.treeObj[this.status.selected];
      return option.value;
    },
    updateChoose(choose, updateValue = true) {
      if (!this.multiple) return;
      choose = choose || [];
      for (let key of Object.keys(this.treeObj)) {
        let tree = this.treeObj[key];
        tree.status.choose = false;
        tree.status.indeterminate = false;
        tree.status.opened = false;
      }
      for (let key of choose) {
        let tree = this.treeObj[key];
        if (tree) {
          tree.status.choose = choose.indexOf(tree.key) != -1;
          if (tree.status.choose) {
            tree.status.opened = true;
            updateParentStatus(this.treeObj, tree, 'opened', true);
            if (this.chooseMode == 'all') {
              updateChildStatus(tree, 'choose', true);
            }
          }
        }
      }
      if (this.chooseMode == 'all') {
        for (let tree of this.treeDatas) {
          updateModeAllChildChooseStatus(tree);
        }
      }
      if (updateValue) {
        this.setvalue();
      }
    },
    setvalue() {
      let value = null;
      if (this.multiple) {
        let choose = this.getChoose();
        let keys = choose.map(item => item[this.param.keyName]);
        value = keys;
      } else {
        let select = this.getSelect();
        value = select ? select[this.param.keyName] : null;
      }
      this.updateFromInput = true;
      this.$emit('update:modelValue', value);
      this.$emit('change', value);
      let event = document.createEvent('CustomEvent');
      event.initCustomEvent('setvalue', true, true, value);
      this.$el.dispatchEvent(event);
    },
    getFullChoose() {
      let options = [];
      for (let key of Object.keys(this.treeObj)) {
        let tree = this.treeObj[key];
        if (tree.status.choose) {
          options.push(tree.value);
        }
      }
      return options;
    },
    getChoose() {
      if (this.chooseMode == 'some') {
        return this.getFullChoose();
      } else if (this.chooseMode == 'independent') {
        return this.getFullChoose();
      } else {
        let options = [];
        for (let data of this.treeDatas) {
          options = getChooseNode(data, options);
        }
        return options;
      }
    }
  }
};
</script>
