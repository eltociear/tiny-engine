<template>
  <div v-show="menuState.show" ref="menuDom" class="context-menu" :style="menuState.position">
    <ul class="menu-item">
      <li
        v-for="(item, index) in menus"
        :key="index"
        :class="item.items ? 'li-item' : ''"
        @click="doOperation(item)"
        @mouseover="current = item"
      >
        <div>
          <span>{{ item.name }}</span>
          <span v-if="item.items"><icon-right></icon-right></span>
        </div>
        <ul v-if="item.items && current === item" class="sub-menu menu-item">
          <template v-for="(subItem, subIndex) in item.items" :key="subIndex">
            <li v-if="subItem?.check?.() !== false" @click.stop="doOperation(subItem)">
              {{ subItem.name }}
            </li>
          </template>
        </ul>
      </li>
    </ul>
    <save-new-block :boxVisibility="boxVisibility" fromCanvas @close="close"></save-new-block>
  </div>
</template>

<script lang="jsx">
import { ref, reactive, nextTick } from 'vue'
import { getConfigure, getController, getCurrent, copyNode, removeNodeById } from './container'
import { useLayout, useModal, useCanvas } from '@opentiny/tiny-engine-controller'
import { SaveNewBlock } from '@opentiny/tiny-engine-common'
import { iconRight } from '@opentiny/vue-icon'

const menuState = reactive({
  position: null,
  show: false,
  menus: []
})

const current = ref(null)
const menuDom = ref(null)

export const closeMenu = () => {
  menuState.show = false
  current.value = null
}

export const openMenu = (offset, event) => {
  const { x, y } = offset
  const { getScale } = useLayout()
  menuState.position = {
    // 位置处于画布右侧边缘时需要调整显示方向 TODO
    left: event.clientX * getScale() + x + 2 + 'px',
    top: event.clientY * getScale() + y + 'px'
  }
  menuState.show = sessionStorage.getItem('pageInfo') ? true : false

  nextTick(() => {
    if (menuDom.value) {
      const { bottom, height, top } = menuDom.value.getBoundingClientRect()
      if (bottom > document.body?.clientHeight) {
        menuState.position.top = top - height + 'px'
      }
    }
  })
}

export default {
  components: {
    SaveNewBlock,
    IconRight: iconRight()
  },
  setup(props, { emit }) {
    const menus = ref([
      { name: '修改属性', code: 'config' },
      {
        name: '插入',
        items: [
          { name: '向前', code: 'insert', value: 'top' },
          {
            name: '中间',
            code: 'insert',
            value: 'in',
            check() {
              const { componentName } = getCurrent()?.schema || {}
              return getConfigure(componentName)?.isContainer
            }
          },
          { name: '向后', code: 'insert', value: 'bottom' }
        ]
      },
      {
        name: '添加父级',
        items: [
          { name: '文字提示', code: 'wrap', value: 'TinyTooltip' },
          { name: '弹出框', code: 'wrap', value: 'TinyPopover' }
        ]
      },
      { name: '删除', code: 'del' },
      { name: '复制', code: 'copy' },
      { name: '绑定事件', code: 'bindEvent' }
    ])

    const boxVisibility = ref(false)

    // 计算上下文菜单位置，右键时显示，否则关闭

    const operations = {
      del() {
        removeNodeById(getCurrent().schema.id)
      },
      copy() {
        copyNode(getCurrent().schema.id)
      },
      config() {
        useLayout().activeSetting('props')
      },
      bindEvent() {
        useLayout().activeSetting('event')
      },
      insert({ value }) {
        emit('insert', value)
      },
      wrap({ value, name }) {
        const componentName = value || name
        const { schema, parent } = getCurrent()
        const index = parent.children.indexOf(schema)
        const wrapSchema = {
          componentName,
          id: null,
          props: {},
          children: [schema]
        }

        parent.children.splice(index, 1, wrapSchema)

        getController().addHistory()
      },
      createBlock() {
        if (useCanvas().isSaved()) {
          boxVisibility.value = true
        } else {
          useModal().message({
            message: '请先保存当前页面',
            status: 'error'
          })
        }
      }
    }

    const close = () => {
      boxVisibility.value = false
    }
    const doOperation = (item) => {
      if (item?.code) {
        operations[item.code](item)
        closeMenu()
      }
    }

    return {
      menuState,
      menus,
      doOperation,
      boxVisibility,
      close,
      current,
      menuDom
    }
  }
}
</script>

<style lang="less" scoped>
.context-menu {
  position: fixed;
  z-index: 10;
}
.menu-item {
  width: 140px;
  line-height: 20px;
  border-radius: 6px;
  padding: 8px 0;
  background-color: var(--ti-lowcode-canvas-menu-bg);
  box-shadow: 0 1px 15px 0 rgb(0 0 0 / 20%);
  display: flex;
  flex-direction: column;
  .li-item {
    border-bottom: 1px solid var(--ti-lowcode-canvas-menu-border-color);
  }
  li {
    & > div {
      display: flex;
      width: 100%;
      justify-content: space-between;
    }
    font-size: 12px;
    color: var(--ti-lowcode-toolbar-breadcrumb-color);
    padding: 6px 15px;
    &:hover {
      color: var(--ti-lowcode-toolbar-icon-color);
      background: var(--ti-lowcode-canvas-menu-hover-color);
    }
    position: relative;
  }
  &.sub-menu {
    width: 100px;
    position: absolute;
    right: -100px;
    top: -2px;
  }
}
</style>
