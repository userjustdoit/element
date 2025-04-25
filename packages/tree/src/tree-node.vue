<template>
  <div
    class="el-tree-node"
    @click.stop="handleClick"
    @contextmenu="($event) => this.handleContextMenu($event)"
    v-show="node.visible||isLoadMoreView"
    :class="{
      'is-expanded': expanded,
      'is-current': node.isCurrent,
      'is-hidden': !node.visible,
      'is-focusable': !node.disabled,
      'is-checked': !node.disabled && node.checked,
      'is-select':node.data.select&&!isLoadMoreView,
      'is-fold-area':node.level===(tree.foldLevel+1),
      'is-unselect':!node.data.select
    }"
    role="treeitem"
    tabindex="-1"
    :aria-expanded="expanded"
    :aria-disabled="node.disabled"
    :aria-checked="node.checked"
    :draggable="tree.draggable"
    @dragstart.stop="handleDragStart"
    @dragover.stop="handleDragOver"
    @dragend.stop="handleDragEnd"
    @drop.stop="handleDrop"
    ref="node"
  >
    <node-content :node="node" :slotName="'contentBefore'" v-if="!node.hideNode&&!isLoadMoreView"></node-content>
    <div class="el-tree-node__content"
         v-show="!node.hideNode"
         :style="{'position':'relative','padding-left': ((node.level>tree.foldLevel)?(node.level-tree.foldLevel - 1):(node.level - 1)) * tree.indent + 'px' }">
       <span
        v-if="!node.hideNode"
        :style="isLoadMoreView?{'visibility':'hidden'}:{}"
        @click.stop="handleExpandIconClick"
        :class="[
          { 'is-leaf': node.isLeaf, expanded: !node.isLeaf && expanded },
          'el-tree-node__expand-icon',
          tree.iconClass ? tree.iconClass : 'el-icon-caret-right'
        ]"
      >
      </span>
      <el-checkbox
        v-if="showCheckbox&&!node.hideNode"
        :style="isLoadMoreView?{'visibility':'hidden'}:{}"
        v-model="node.checked"
        :indeterminate="node.indeterminate"
        :disabled="!!node.disabled"
        @click.native.stop
        @change="handleCheckChange"
      >
      </el-checkbox>
      <span
        v-if="node.loading"
        :style="isLoadMoreView?{'visibility':'hidden'}:{}"
        class="el-tree-node__loading-icon el-icon-loading">
      </span>
      <node-content :node="node" :slotName="'contentLeft'" v-if="!node.hideNode" :style="isLoadMoreView?{'visibility':'hidden'}:{}" ></node-content>
<!--      <span ref="contentStartRef"  v-if="childAllCount>1"></span>-->
      <el-button type="info" size="mini" :title="node.parent?node.parent.label:''" @click="$emit('loadMoreClick')" v-if="isLoadMoreView">{{loadMoreText}}</el-button>
      <node-content :node="node" v-else></node-content>
    </div>
    <node-content :node="node" :slotName="'contentAfter'" v-if="!node.hideNode&&!isLoadMoreView"></node-content>
    <el-collapse-transition  v-if="!node.hideNode&&!isLoadMoreView">
      <div
        class="el-tree-node__children"
        :style="{'position':'relative'}"
        ref="borderParentRef"
        v-if="!renderAfterExpand || childNodeRendered"
        v-show="expanded"
        role="group"
        :aria-expanded="expanded"
      >
<!--        <div :style="{'position':'absolute','left':borderLineLeftPx+'px','top':borderLineTopPx+'px','bottom':borderLineBottomPx+'px','border-left': '1px dotted #000000','z-index':1}" v-if="childNodes.length>1&&borderLineLeftPx>=0">
        </div>-->
        <el-tree-node :isLoadMoreView="showLoadMore" :limitSize="limitSize" :initLimitedSize="initLimitedSize" :limitRefresh="limitRefresh" :show-checkbox="showCheckbox" loadMoreText="刷新" @loadMoreClick="refreshClick" :node="childNodes[childNodes.length-1]" v-if="showRefresh">
        </el-tree-node>
        <el-tree-node
          :render-content="renderContent"
          v-for="(child,index) in childNodes"
          :childIndexCurrent="index"
          :childAllCount="childNodes.length"
          :render-after-expand="renderAfterExpand"
          :show-checkbox="showCheckbox"
          :limitSize="limitSize"
          :initLimitedSize="initLimitedSize"
          :limitRefresh="limitRefresh"
          :key="getNodeKey(child)"
          :node="child"
          @node-expand="handleChildNodeExpand">
        </el-tree-node>
        <el-tree-node :isLoadMoreView="showLoadMore" :limitSize="limitSize" :initLimitedSize="initLimitedSize" :limitRefresh="limitRefresh" :show-checkbox="showCheckbox"  @loadMoreClick="loadMoreClick" :node="childNodes[childNodes.length-1]" v-if="showLoadMore">
        </el-tree-node>
      </div>
    </el-collapse-transition>
  </div>
</template>

<script type="text/jsx">
import ElCollapseTransition from 'element-ui/src/transitions/collapse-transition'
import ElCheckbox from 'element-ui/packages/checkbox'
import ElButton from 'element-ui/packages/button'
import emitter from 'element-ui/src/mixins/emitter'
import {getNodeKey} from './model/util'

let $log=console.log

export default {
  name: 'ElTreeNode',

  componentName: 'ElTreeNode',

  mixins: [emitter],

  props: {
    node: {
      default () {
        return {}
      }
    },
    childIndexCurrent:{
      type: Number,
      default: -1
    },
    childAllCount:{
      type: Number,
      default: -1
    },
    limitSize: {
      type: Number,
      default: -1
    },
    initLimitedSize: {
      type: Number,
      default: -1
    },
    limitRefresh: {
      type: Boolean,
      default: false
    },
    isLoadMoreView: {
      type: Boolean,
      default: false
    },
    loadMoreText: {
      type: String,
      default: "加载更多"
    },
    props: {},
    renderContent: Function,
    renderAfterExpand: {
      type: Boolean,
      default: true
    },
    showCheckbox: {
      type: Boolean,
      default: false
    }
  },

  components: {
    ElCollapseTransition,
    ElCheckbox,
    ElButton, /*NodeContent: {
     props: {
     node: {
     required: true
     }
     },
     render(h) {
     const parent = this.$parent;
     const tree = parent.tree;
     const node = this.node;
     const { data, store } = node;
     return (
     parent.renderContent
     ? parent.renderContent.call(parent._renderProxy, h, { _self: tree.$vnode.context, node, data, store })
     : tree.$scopedSlots.default
     ? tree.$scopedSlots.default({ node, data })
     : <span class="el-tree-node__label">{ node.label }</span>
     );
     }
     },*/
    NodeContent: {
      props: {
        node: {
          required: true
        },
        slotName: {
          required: false
        }
      },
      render (h) {
        const parent = this.$parent
        const tree = parent.tree
        const node = this.node
        const isOther = this.slotName
        const slotName = this.slotName ? this.slotName : 'default'
        const {data, store} = node
        return (parent.renderContent ? parent.renderContent.call(parent._renderProxy, h, {
          _self: tree.$vnode.context,
          node,
          data,
          store
        }) : tree.$scopedSlots[slotName] ? tree.$scopedSlots[slotName]({
          node,
          data
        }) : isOther ? '' : <span class="el-tree-node__label">{node.label}</span>)
      }
    }
  },

  data () {
    return {
      tree: null,
      expanded: false,
      childNodeRendered: false,
      oldChecked: null,
      oldIndeterminate: null,
      showCount:-1,
      /*borderLineLeftPx:-1,
      borderLineTopPx:0,
      borderLineBottomPx:0,
      borderLineRightPx:0,*/
    }
  },
  computed: {
    initChildNodes(){
      return this.node.childNodes.length>0
    },
    showRefresh(){
      return this.limitRefresh&&this.initChildNodes&&this.showCount>this.limitSize&&this.enableLoadMore
    },
    enableLoadMore(){
      return this.limitSize>0&&(this.node.childNodes.length>this.limitSize)
    },
    showLoadMore(){
      if(this.showCount<0){
        this.refreshClick()
      }
      return this.initChildNodes&&(this.showCount<this.node.childNodes.length)&&this.enableLoadMore
    },
    childNodes(){
      if(!this.initChildNodes){
        return []
      }
      if(this.showLoadMore){
        return this.node.childNodes.slice(0,this.showCount)
      }
      return this.node.childNodes
    },
    // borderLineLeftPx(){
    //   const parent = this.$refs.node;
    //   const child = this.$refs.expandIcon;
    //   $log(parent)
    //   $log(child)
    //   if (parent && child) {
    //     const parentRect = parent.getBoundingClientRect();
    //     const childRect = child.getBoundingClientRect();
    //     if(childRect.width!==24){
    //       return false
    //     }
    //     $log(JSON.stringify(parentRect))
    //     $log(JSON.stringify(childRect))
    //     // 相对于父元素的位置
    //     const relativeLeft = childRect.left - parentRect.left+childRect.width/2;
    //     return relativeLeft+'px'
    //   }
    //   return false
    // },
  },
  watch: {
    'node.indeterminate' (val) {
      this.handleSelectChange(this.node.checked, val)
    },

    'node.checked' (val) {
      this.handleSelectChange(val, this.node.indeterminate)
    },

    'node.expanded' (val) {
      this.$nextTick(() => this.expanded = val)
      if (val) {
        this.childNodeRendered = true
      }
    },
    // 'limitSize' (val) {
    //   this.refreshClick()
    // },
  },

  methods: {
    // setContentLeftPx(childRect,childIndex){
    //   const parent = this.$refs.borderParentRef;
    //   if(childIndex===0){
    //     if (parent) {
    //       const parentRect = parent.getBoundingClientRect();
    //       // 相对于父元素的位置
    //       const relativeLeft = childRect.left - parentRect.left;
    //       this.borderLineLeftPx=relativeLeft
    //       const relativeTop = childRect.top - parentRect.top;
    //       this.borderLineTopPx=relativeTop
    //     }
    //   }else if(childIndex===this.childNodes.length-1){
    //     if (parent) {
    //       const parentRect = parent.getBoundingClientRect();
    //       const relativeBottom = parentRect.height-(childRect.top - parentRect.top);
    //       this.borderLineBottomPx=relativeBottom
    //     }
    //   }
    // },
    // emitContentLeftPx(){
    //     const child = this.$refs.contentStartRef;
    //     if (child) {
    //       const childRect = child.getBoundingClientRect();
    //       this.$emit('setContentLeftPx',childRect,this.childIndexCurrent)
    //     }
    // },
    loadMoreClick(){
      this.showCount=this.showCount+this.limitSize
    },
    refreshClick(){
      if(this.limitSize>0){
        let showCount = this.limitSize
        if(this.initLimitedSize>0){
          showCount=this.initLimitedSize
        }
        this.showCount=showCount
      }else{
        this.showCount=10000000
      }
    },
    getNodeKey (node) {
      return getNodeKey(this.tree.nodeKey, node.data)
    },

    handleSelectChange (checked, indeterminate) {
      if(this.isLoadMoreView){
        return
      }
      if (this.oldChecked !== checked && this.oldIndeterminate !== indeterminate) {
        this.tree.$emit('check-change', this.node.data, checked, indeterminate)
      }
      this.oldChecked = checked
      this.indeterminate = indeterminate
    },

    handleClick () {
      if(this.isLoadMoreView){
        return
      }
      const store = this.tree.store
      store.setCurrentNode(this.node)
      this.tree.$emit('current-change', store.currentNode ? store.currentNode.data : null, store.currentNode)
      this.tree.currentNode = this
      if (this.tree.expandOnClickNode) {
        this.handleExpandIconClick()
      }
      if (this.tree.checkOnClickNode && !this.node.disabled) {
        this.handleCheckChange(null, {
          target: {checked: !this.node.checked}
        })
      }
      this.tree.$emit('node-click', this.node.data, this.node, this)
    },

    handleContextMenu (event) {
      if(this.isLoadMoreView){
        return
      }
      if (this.tree._events['node-contextmenu'] && this.tree._events['node-contextmenu'].length > 0) {
        event.stopPropagation()
        event.preventDefault()
      }
      this.tree.$emit('node-contextmenu', event, this.node.data, this.node, this)
    },

    handleExpandIconClick () {
      if(this.isLoadMoreView){
        return
      }
      if (this.node.isLeaf) return
      if (this.expanded) {
        this.tree.$emit('node-collapse', this.node.data, this.node, this)
        this.node.collapse()
      } else {
        this.node.expand()
        this.$emit('node-expand', this.node.data, this.node, this)
      }
    },

    handleCheckChange (value, ev) {
      if(this.isLoadMoreView){
        return
      }
      this.node.setChecked(ev.target.checked, !this.tree.checkStrictly)
      this.$nextTick(() => {
        const store = this.tree.store
        this.tree.$emit('check', this.node.data, {
          checkedNodes: store.getCheckedNodes(),
          checkedKeys: store.getCheckedKeys(),
          halfCheckedNodes: store.getHalfCheckedNodes(),
          halfCheckedKeys: store.getHalfCheckedKeys(),
        })
      })
    },

    handleChildNodeExpand (nodeData, node, instance) {
      if(this.isLoadMoreView){
        return
      }
      this.broadcast('ElTreeNode', 'tree-node-expand', node)
      this.tree.$emit('node-expand', nodeData, node, instance)
    },

    handleDragStart (event) {
      if(this.isLoadMoreView){
        return
      }
      if (!this.tree.draggable) return
      this.tree.$emit('tree-node-drag-start', event, this)
    },

    handleDragOver (event) {
      if(this.isLoadMoreView){
        return
      }
      if (!this.tree.draggable) return
      this.tree.$emit('tree-node-drag-over', event, this)
      event.preventDefault()
    },

    handleDrop (event) {
      if(this.isLoadMoreView){
        return
      }
      event.preventDefault()
    },

    handleDragEnd (event) {
      if(this.isLoadMoreView){
        return
      }
      if (!this.tree.draggable) return
      this.tree.$emit('tree-node-drag-end', event, this)
    }
  },

  created () {

    const parent = this.$parent

    if (parent.isTree) {
      this.tree = parent
    } else {
      this.tree = parent.tree
    }

    const tree = this.tree
    if (!tree) {
      console.warn('Can not find node\'s tree.')
    }

    const props = tree.props || {}
    const childrenKey = props['children'] || 'children'

    this.$watch(`node.data.${childrenKey}`, () => {
      this.node.updateChildren()
    })

    if (this.node.expanded) {
      this.expanded = true
      this.childNodeRendered = true
    }

    if (this.tree.accordion) {
      this.$on('tree-node-expand', node => {
        if (this.node !== node) {
          this.node.collapse()
        }
      })
    }
  },
  // updated() {
  //   if(this.childIndexCurrent===0||this.childIndexCurrent===this.childAllCount-1){
  //     this.$nextTick(()=>{
  //       this.emitContentLeftPx()
  //     })
  //   }
  // },
  // mounted() {
  //   if(this.childIndexCurrent===0||this.childIndexCurrent===this.childAllCount-1){
  //     this.$nextTick(()=>{
  //       this.emitContentLeftPx()
  //     })
  //   }
  // }
}
</script>
