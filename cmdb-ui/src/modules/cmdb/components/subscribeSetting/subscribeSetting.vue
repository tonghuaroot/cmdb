<template>
  <CustomDrawer
    :width="600"
    @close="onClose"
    :visible="visible"
    :hasTitle="false"
    :hasFooter="false"
    :maskClosable="false"
    :bodyStyle="{ padding: 0 }"
    wrapClassName="cmdb-subscribe-drawer"
  >
    <a-tabs v-model="activeKey">
      <a-tab-pane key="1">
        <span slot="tab"><ops-icon type="cmdb-ci" />{{ $t('cmdb.menu.ciTable') }}</span>
        <div class="cmdb-subscribe-drawer-container" :style="{ height: `${windowHeight - 60}px` }">
          <div class="cmdb-subscribe-drawer-container-title">
            <span>{{ $t('cmdb.components.subCIType') }}: {{ ciType.alias || ciType.name }}</span>
            <span :style="{ fontWeight: 500, color: instanceSubscribed ? 'green' : 'red' }">{{
              `（${instanceSubscribed ? $t('cmdb.components.already') : $t('cmdb.components.not')}`
            }}{{ $t('cmdb.components.sub') }})</span>
          </div>
          <template>
            <AttributesTransfer
              :dataSource="attrList"
              :targetKeys="selectedAttrList"
              @setTargetKeys="setTargetKeys"
              @changeSingleItem="changeSingleItem"
              :hasFooter="false"
              :fixedList="fixedList"
              @setFixedList="setFixedList"
              :height="windowHeight - 170"
              :showDefaultAttr="true"
            />
            <div class="custom-drawer-bottom-action">
              <a-button @click="subInstanceSubmit" type="primary">{{ $t('cmdb.preference.sub') }}</a-button>
            </div>
          </template>
        </div>
      </a-tab-pane>
      <a-tab-pane key="2" force-render>
        <span slot="tab"><ops-icon type="cmdb-tree" />{{ $t('cmdb.menu.ciTree') }}</span>
        <div class="cmdb-subscribe-drawer-container" :style="{ height: `${windowHeight - 60}px` }">
          <div class="cmdb-subscribe-drawer-container-title">
            <span>{{ $t('cmdb.components.subCIType') }}: {{ ciType.alias || ciType.name }}</span>
            <span :style="{ fontWeight: 500, color: treeSubscribed ? 'green' : 'red' }">{{
              `（${treeSubscribed ? $t('cmdb.components.already') : $t('cmdb.components.not')}`
            }}{{ $t('cmdb.components.sub') }})</span>
          </div>
          <div class="cmdb-subscribe-drawer-tree-header" :style="{ maxHeight: `${(windowHeight - 170) / 3 - 20}px` }">
            <span v-if="!treeViews.length">{{ $t('cmdb.components.selectBelow') }}</span>
            <div
              class="cmdb-subscribe-drawer-tree-header-selected"
              :style="{ marginLeft: `${18 * index}px` }"
              v-for="(item, index) in treeViews"
              :key="item"
            >
              <span
              >{{ getDisplayAttr(item) }}
                <a-icon
                  @click="closeTreeViews(item, index)"
                  class="cmdb-subscribe-drawer-tree-header-selected-close"
                  type="close"
                />
              </span>
            </div>
          </div>
          <div class="cmdb-subscribe-drawer-tree-main" :style="{ maxHeight: `${((windowHeight - 170) * 2) / 3}px` }">
            <div @click="changeTreeViews(attr)" v-for="attr in treeViewAttrList" :key="attr.name">
              <a-checkbox :checked="treeViews.includes(attr.name)" />
              {{ attr.title }}
            </div>
          </div>
          <div class="custom-drawer-bottom-action">
            <a-button @click="subTreeSubmit" type="primary">{{ $t('cmdb.preference.sub') }}</a-button>
          </div>
        </div>
      </a-tab-pane>
    </a-tabs>
  </CustomDrawer>
</template>

<script>
import { mapState } from 'vuex'
import router, { resetRouter } from '@/router'
import store from '@/store'
import {
  subscribeCIType,
  getSubscribeAttributes,
  getSubscribeTreeView,
  subscribeTreeView,
} from '@/modules/cmdb/api/preference'
import { getCITypeAttributesByName } from '@/modules/cmdb/api/CITypeAttr'
import AttributesTransfer from '../attributesTransfer'
import { CI_DEFAULT_ATTR } from '@/modules/cmdb/utils/const.js'

export default {
  name: 'SubscribeSetting',
  components: { AttributesTransfer },
  data() {
    return {
      visible: false,
      activeKey: '1',
      ciType: {},
      instanceSubscribed: false,
      treeSubscribed: false,
      attrList: [],
      selectedAttrList: [],
      treeViews: [],
      fixedList: [],
    }
  },
  computed: {
    ...mapState({
      windowHeight: (state) => state.windowHeight,
    }),
    treeViewAttrList() {
      return this.attrList.filter((item) => ![CI_DEFAULT_ATTR.UPDATE_USER, CI_DEFAULT_ATTR.UPDATE_TIME].includes(item.name))
    }
  },
  methods: {
    open(ciType = {}, activeKey = '1') {
      this.ciType = ciType
      this.activeKey = activeKey
      const updatedByKey = CI_DEFAULT_ATTR.UPDATE_USER
      const updatedAtKey = CI_DEFAULT_ATTR.UPDATE_TIME

      getCITypeAttributesByName(ciType.type_id).then((res) => {
        const attributes = res.attributes.filter((item) => ![updatedByKey, updatedAtKey].includes(item.name))

        ;[updatedByKey, updatedAtKey].map((key) => {
          attributes.push({
            alias: key,
            name: key,
            id: key
          })
        })

        getSubscribeAttributes(ciType.type_id).then((_res) => {
          this.instanceSubscribed = _res.is_subscribed
          const selectedAttrList = _res.attributes.map((item) => item.id.toString())

          const attrList = attributes.map((item) => {
            return {
              key: item.id.toString(),
              title: item.alias || item.name,
              name: item.name,
            }
          })

          this.attrList = attrList
          this.selectedAttrList = selectedAttrList
          this.fixedList = _res.attributes.filter((item) => item.is_fixed).map((item) => item.id.toString())
          this.visible = true
        })
      })
      this.getTreeView(ciType.type_id)
    },
    getTreeView(typeId) {
      const that = this
      this.treeViews = []
      getSubscribeTreeView().then((res) => {
        let hasMatch = false
        res.forEach((item) => {
          if (item.type_id === typeId) {
            hasMatch = true
            const levels = []
            if (item.levels && item.levels.length >= 1) {
              item.levels.forEach((level) => {
                levels.push(level.name)
              })
            }
            if (levels.length > 0) {
              that.treeSubscribed = true
            } else {
              that.treeSubscribed = false
            }
            that.treeViews = levels
          }
        })
        if (!hasMatch) {
          that.treeSubscribed = false
        }
      })
    },
    onClose() {
      if (!this.treeSubscribed) {
        if (Number(this.$route.params.typeId) === Number(this.ciType.type_id)) {
          this.$router.history.push('/cmdb/tree_views')
          this.$emit('reload')
        } else {
          this.$emit('reload')
        }
      } else {
        this.$emit('reload')
      }
      this.visible = false
      this.activeKey = '1'
    },
    subTreeSubmit() {
      subscribeTreeView(this.ciType.type_id, this.treeViews).then((res) => {
        this.$message.success(this.$t('cmdb.components.subSuccess'))
        if (this.treeViews.length > 0) {
          this.treeSubscribed = true
        } else {
          this.treeSubscribed = false
        }
      })
    },
    subInstanceSubmit() {
      const customAttr = []
      const defaultAttr = []
      this.selectedAttrList.map((attr) => {
        if ([CI_DEFAULT_ATTR.UPDATE_USER, CI_DEFAULT_ATTR.UPDATE_TIME].includes(attr)) {
          defaultAttr.push(attr)
        } else {
          customAttr.push(attr)
        }
      })
      const selectedAttrList = [...customAttr, ...defaultAttr]

      subscribeCIType(
        this.ciType.type_id,
        selectedAttrList.map((item) => {
          return [item, !!this.fixedList.includes(item)]
        })
      ).then((res) => {
        this.$message.success(this.$t('cmdb.components.subSuccess'))
        this.resetRoute()
        if (this.selectedAttrList.length > 0) {
          this.instanceSubscribed = true
        } else {
          this.instanceSubscribed = false
        }
      })
    },
    resetRoute() {
      resetRouter()
      const roles = store.getters.roles
      store.dispatch('GenerateRoutes', { roles }, { root: true }).then(() => {
        router.addRoutes(store.getters.appRoutes)
      })
    },
    setTargetKeys(targetKeys) {
      this.selectedAttrList = targetKeys
    },
    changeSingleItem(item) {
      const idx = this.selectedAttrList.findIndex((key) => key === item.key)
      if (idx > -1) {
        this.selectedAttrList.splice(idx, 1)
      } else {
        this.selectedAttrList.push(item.key)
      }
    },
    setFixedList(fixedList) {
      this.fixedList = fixedList
    },
    changeTreeViews(attr) {
      const _idx = this.treeViews.findIndex((item) => item === attr.name)
      if (_idx > -1) {
        this.treeViews.splice(_idx, 1)
      } else {
        this.treeViews.push(attr.name)
      }
    },
    closeTreeViews(item, idx) {
      this.treeViews.splice(idx, 1)
    },
    getDisplayAttr(item) {
      const _find = this.attrList.find((attr) => attr.name === item)
      return _find?.title ?? item
    },
  },
}
</script>

<style lang="less" scoped>

.cmdb-subscribe-drawer {
  .cmdb-subscribe-drawer-container {
    padding: 0 24px;
    .cmdb-subscribe-drawer-container-title {
      margin-bottom: 12px;
    }
    .cmdb-subscribe-drawer-tree-header {
      border-radius: 4px;
      background-color: @primary-color_5;
      color: rgba(0, 0, 0, 0.4);
      padding: 8px 12px;
      margin-bottom: 12px;
      overflow: auto;
      .cmdb-subscribe-drawer-tree-header-selected {
        color: #a5a9bc;
        margin-bottom: 10px;
        > span {
          display: inline-block;
          background-color: #fff;
          border-left: 2px solid @primary-color;
          padding: 3px 12px;
          position: relative;
          white-space: nowrap;
          .cmdb-subscribe-drawer-tree-header-selected-close {
            cursor: pointer;
            font-size: 12px;
            &:hover {
              color: @primary-color;
            }
          }
        }
      }
      .cmdb-subscribe-drawer-tree-header-selected:not(:first-child) {
        > span::after {
          content: '';
          position: absolute;
          background-color: rgba(47, 84, 235, 0.5);
          height: 1px;
          width: 18px;
          left: -18px;
          top: 14px;
        }
      }
      .cmdb-subscribe-drawer-tree-header-selected:not(:last-child) {
        > span::before {
          content: '';
          position: absolute;
          width: 1px;
          height: 25px;
          left: -1px;
          top: 27px;
          background-color: rgba(47, 84, 235, 0.5);
        }
      }
    }
    .cmdb-subscribe-drawer-tree-main {
      background-color: #fff;
      box-shadow: -1px 5px 10px #dee3f9;
      padding: 10px 18px;
      overflow: auto;
      > div {
        height: 30px;
      }
    }
  }
}
</style>

<style lang="less">
.cmdb-subscribe-drawer {
  .ant-tabs-bar {
    background-color: @primary-color_5;
    border-bottom: none;
  }
}
</style>
