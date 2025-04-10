<template>
  <div>
    <search-form
      :attrList="typeTableAttrList"
      @expandChange="handleExpandChange"
      @search="handleSearch"
      @searchFormReset="searchFormReset"
      @export="handleExport"
    ></search-form>
    <vxe-table
      ref="xTable"
      :loading="loading"
      resizable
      :data="tableData"
      :max-height="`${windowHeight - windowHeightMinus}px`"
      row-id="_XID"
      size="small"
      :row-config="{ isHover: true }"
      stripe
      class="ops-stripe-table"
    >
      <vxe-column field="created_at" width="159px" :title="$t('cmdb.history.opreateTime')"></vxe-column>
      <vxe-column field="user" width="116px" :title="$t('cmdb.history.user')"></vxe-column>
      <vxe-column field="operate_type" width="135px" :title="$t('operation')">
        <template #header="{ column }">
          <span>{{ column.title }}</span>
          <a-popover trigger="click" placement="bottom">
            <a-icon class="filter" type="filter" theme="filled" />
            <a class="filter-content" slot="content">
              <a-select
                v-model="queryParams.operate_type"
                :placeholder="$t('cmdb.history.filterOperate')"
                show-search
                style="width: 200px"
                :filter-option="filterOption"
                allowClear
              >
                <a-select-option
                  :value="Object.values(choice)[0]"
                  :key="index"
                  v-for="(choice, index) in typeTableAttrList[1].choice_value"
                >
                  {{ Object.keys(choice)[0] }}
                </a-select-option>
              </a-select>
              <a-button type="link" class="filterButton" @click="filterOperate">{{
                $t('cmdb.history.filter')
              }}</a-button>
              <a-button type="link" class="filterResetButton" @click="filterOperateReset">{{ $t('reset') }}</a-button>
            </a>
          </a-popover>
        </template>
        <template #default="{ row }">
          <a-tag color="green" v-if="row.operate_type.includes($t('new'))">
            {{ row.operate_type }}
          </a-tag>
          <a-tag color="orange" v-else-if="row.operate_type.includes($t('update'))">
            {{ row.operate_type }}
          </a-tag>
          <a-tag color="red" v-else>
            {{ row.operate_type }}
          </a-tag>
        </template>
      </vxe-column>
      <vxe-column field="type_id" :title="$t('cmdb.ciType.ciType')" width="150px">
        <template #default="{ row }">
          {{ row.operate_type === $t('cmdb.history.deleteCIType') ? row.change.alias : row.type_id }}
        </template>
      </vxe-column>
      <vxe-column field="changeDescription" :title="$t('desc')">
        <template #default="{ row }">
          <p style="color:rgba(0, 0, 0, 0.65);" v-if="row.changeDescription === $t('cmdb.history.noUpdate')">
            {{ row.changeDescription }}
          </p>
          <template v-else-if="row.operate_type.includes($t('update'))">
            <p :key="index" style="color:#fa8c16;" v-for="(tag, index) in row.changeArr">
              {{ tag }}
            </p>
          </template>
          <p class="more-tag" style="color:#52c41a;" v-else-if="row.operate_type.includes($t('new'))">
            {{ row.changeDescription }}
          </p>
          <p style="color:#f5222d;" v-else-if="row.operate_type.includes($t('delete'))">
            {{ row.changeDescription }}
          </p>
        </template>
      </vxe-column>
    </vxe-table>
    <a-row class="row" type="flex" justify="end">
      <a-col>
        <a-pagination
          size="small"
          v-model="current"
          :page-size-options="pageSizeOptions"
          :total="numfound"
          show-size-changer
          :page-size="pageSize"
          @change="onChange"
          @showSizeChange="onShowSizeChange"
          :show-total="(total) => $t('cmdb.history.totalItems', { total: total })"
        >
        </a-pagination>
      </a-col>
    </a-row>
  </div>
</template>

<script>
import _ from 'lodash'
import { mapState } from 'vuex'
import SearchForm from './searchForm'
import { getCITypesTable, getUsers } from '@/modules/cmdb/api/history'
import { getCITypes } from '@/modules/cmdb/api/CIType'
import { getRelationTypes } from '@/modules/cmdb/api/relationType'
export default {
  name: 'TypeTable',
  components: { SearchForm },
  inject: ['reload'],
  data() {
    return {
      loading: true,
      relationTypeList: null,
      typeList: null,
      userList: [],
      pageSizeOptions: ['50', '100', '200', '500'],
      isExpand: false,
      current: 1,
      pageSize: 50,
      total: 0,
      numfound: 0,
      tableData: [],
      queryParams: {
        page: 1,
        page_size: 50,
        type_id: undefined,
        operate_type: undefined,
      },
      typeTableAttrList: [
        {
          alias: this.$t('cmdb.ciType.ciType'),
          is_choice: true,
          name: 'type_id',
          value_type: '2',
          choice_value: [],
        },
        {
          alias: this.$t('operation'),
          is_choice: true,
          name: 'operate_type',
          value_type: '2',
          choice_value: [
            { [this.$t('cmdb.history.addCIType')]: 0 },
            { [this.$t('cmdb.history.updateCIType')]: 1 },
            { [this.$t('cmdb.history.deleteCIType')]: 2 },
            { [this.$t('cmdb.history.addAttribute')]: 3 },
            { [this.$t('cmdb.history.updateAttribute')]: 4 },
            { [this.$t('cmdb.history.deleteAttribute')]: 5 },
            { [this.$t('cmdb.history.addTrigger')]: 6 },
            { [this.$t('cmdb.history.updateTrigger')]: 7 },
            { [this.$t('cmdb.history.deleteTrigger')]: 8 },
            { [this.$t('cmdb.history.addUniqueConstraint')]: 9 },
            { [this.$t('cmdb.history.updateUniqueConstraint')]: 10 },
            { [this.$t('cmdb.history.deleteUniqueConstraint')]: 11 },
            { [this.$t('cmdb.history.addRelation')]: 12 },
            { [this.$t('cmdb.history.deleteRelation')]: 13 },
            { [this.$t('cmdb.history.addReconciliation')]: 14 },
            { [this.$t('cmdb.history.updateReconciliation')]: 15 },
            { [this.$t('cmdb.history.deleteReconciliation')]: 16 },
          ],
        },
      ],
    }
  },
  async created() {
    await Promise.all([this.getRelationTypes(), this.getTypes(), this.getUserList()])
    await this.getTable(this.queryParams)
  },
  updated() {
    this.$refs.xTable.$el.querySelector('.vxe-table--body-wrapper').scrollTop = 0
  },
  computed: {
    ...mapState(['locale']),
    windowHeight() {
      return this.$store.state.windowHeight
    },
    windowHeightMinus() {
      return this.isExpand ? 396 : 335
    },
    operateTypeMap() {
      return new Map([
        ['0', this.$t('cmdb.history.addCIType')],
        ['1', this.$t('cmdb.history.updateCIType')],
        ['2', this.$t('cmdb.history.deleteCIType')],
        ['3', this.$t('cmdb.history.addAttribute')],
        ['4', this.$t('cmdb.history.updateAttribute')],
        ['5', this.$t('cmdb.history.deleteAttribute')],
        ['6', this.$t('cmdb.history.addTrigger')],
        ['7', this.$t('cmdb.history.updateTrigger')],
        ['8', this.$t('cmdb.history.deleteTrigger')],
        ['9', this.$t('cmdb.history.addUniqueConstraint')],
        ['10', this.$t('cmdb.history.updateUniqueConstraint')],
        ['11', this.$t('cmdb.history.deleteUniqueConstraint')],
        ['12', this.$t('cmdb.history.addRelation')],
        ['13', this.$t('cmdb.history.deleteRelation')],
        ['14', this.$t('cmdb.history.addReconciliation')],
        ['15', this.$t('cmdb.history.updateReconciliation')],
        ['16', this.$t('cmdb.history.deleteReconciliation')],
      ])
    },
  },
  watch: {
    current(val) {
      this.queryParams.page = val
    },
    pageSize(val) {
      this.queryParams.page_size = val
    },
    locale() {
      this.reload()
    },
  },
  methods: {
    async getTable(queryParams) {
      try {
        this.loading = true
        const res = await getCITypesTable(queryParams)
        res.result.forEach((item) => {
          this.handleChangeDescription(item, item.operate_type)
          item.operate_type = this.handleOperateType(item.operate_type)
          item.type_id = this.handleTypeId(item.type_id)
          item.uid = this.handleUID(item.uid)
        })
        this.tableData = res.result
        this.pageSize = res.page_size
        this.current = res.page
        this.numfound = res.numfound
        this.total = res.total
        console.log(this.tableData)
      } finally {
        this.loading = false
      }
    },
    async getTypes() {
      const res = await getCITypes()
      const typesArr = []
      const typesMap = new Map()
      res.ci_types.forEach((item) => {
        const tempObj = {}
        tempObj[item.alias] = item.id
        if (item.alias) {
          typesMap.set(item.id, item.alias)
          typesArr.push(tempObj)
        }
      })
      this.typeList = typesMap
      this.typeTableAttrList[0].choice_value = typesArr
    },
    async getUserList() {
      const res = await getUsers()
      const userListMap = new Map()
      res.forEach((item) => {
        userListMap.set(item.uid, item.nickname)
      })
      this.userList = userListMap
    },
    async getRelationTypes() {
      const res = await getRelationTypes()
      const relationTypeMap = new Map()
      res.forEach((item) => {
        relationTypeMap.set(item.id, item.name)
      })
      this.relationTypeList = relationTypeMap
    },
    onChange(current) {
      this.current = current
      this.getTable(this.queryParams)
    },
    onShowSizeChange(current, size) {
      this.current = 1
      this.pageSize = size
      this.getTable(this.queryParams)
    },
    handleExpandChange(expand) {
      this.isExpand = expand
    },
    handleSearch(queryParams) {
      this.current = 1
      this.queryParams = queryParams
      this.getTable(this.queryParams)
    },
    searchFormReset() {
      this.queryParams = {
        page: 1,
        page_size: 50,
        type_id: undefined,
        operate_type: undefined,
      }
      this.getTable(this.queryParams)
    },
    handleOperateType(operate_type) {
      return this.operateTypeMap.get(operate_type)
    },
    handleTypeId(type_id) {
      return this.typeList.get(type_id) ? this.typeList.get(type_id) : type_id
    },
    handleUID(uid) {
      return this.userList.get(uid)
    },
    handleRelationType(relation_type_id) {
      return this.relationTypeList.get(relation_type_id)
    },
    handleChangeDescription(item, operate_type) {
      switch (operate_type) {
        // add CIType
        case '0': {
          item.changeDescription = this.$t('cmdb.history.addCIType') + ': ' + item.change.alias
          break
        }
        // update CIType
        case '1': {
          item.changeArr = []
          const diffs = this.deepCompare({
            obj1: item?.change?.old,
            obj2: item?.change?.new,
            ignoreKeys: ['updated_at']
          })
          for (const val of diffs) {
            const str = ` [ ${val.path} :  ${val.value1} -> ${val.value2} ] `
            item.changeDescription += str
            item.changeArr.push(str)
          }
          if (!item.changeDescription) item.changeDescription = this.$t('cmdb.history.noModifications')
          break
        }
        // delete CIType
        case '2': {
          item.changeDescription = this.$t('cmdb.history.deleteCIType') + ': ' + `${item.change.alias}`
          break
        }
        // add Attribute
        case '3': {
          item.changeDescription = `${this.$t('cmdb.history.attr')}：${item.change.alias}`
          break
        }
        // update Attribute
        case '4': {
          item.changeArr = []
          const diffs = this.deepCompare({
            obj1: item?.change?.old,
            obj2: item?.change?.new,
            ignoreKeys: ['updated_at']
          })
          for (const val of diffs) {
            const str = ` [ ${val.path} :  ${val.value1} -> ${val.value2} ] `
            item.changeDescription += str
            item.changeArr.push(str)
          }
          if (!item.changeDescription) item.changeDescription = this.$t('cmdb.history.noModifications')
          break
        }
        // delete Attribute
        case '5': {
          item.changeDescription = `${this.$t('delete')}：${item.change.alias}`
          break
        }
        // add trigger
        case '6': {
          item.changeDescription = `${this.$t('cmdb.history.addTrigger')}：${item?.change?.option?.name || ''}`
          break
        }
        // update trigger
        case '7': {
          item.changeArr = []
          const diffs = this.deepCompare({
            obj1: item?.change?.old,
            obj2: item?.change?.new,
            directDeepKeys: ['notifies'],
            ignoreKeys: ['updated_at']
          })
          for (const val of diffs) {
            const str = ` [ ${val.path} :  ${val.value1} -> ${val.value2} ] `
            item.changeDescription += str
            item.changeArr.push(str)
          }
          if (!item.changeDescription) item.changeDescription = this.$t('cmdb.history.noModifications')
          break
        }
        // delete trigger
        case '8': {
          item.changeDescription = `${this.$t('cmdb.history.deleteTrigger')}：${item?.change?.option?.name || ''}`
          break
        }
        // add unique constraint
        case '9': {
          item.changeDescription = `${this.$t('cmdb.history.attrId')}：[${item.change.attr_ids}]`
          break
        }
        // update unique constraint
        case '10': {
          item.changeArr = []
          const oldVal = item.change.old.attr_ids
          const newVal = item.change.new.attr_ids
          const str = `${this.$t('cmdb.history.attrId')}：[${oldVal}] -> [${newVal}]`
          item.changeDescription = str
          item.changeArr.push(str)
          break
        }
        // delete unique constraint
        case '11': {
          item.changeDescription = `${this.$t('cmdb.history.attrId')}：[${item.change.attr_ids}]`
          break
        }
        // add relation
        case '12': {
          item.changeDescription = `${this.$t('new')}：${item.change.parent.alias} -> ${this.handleRelationType(
            item.change.relation_type_id
          )} -> ${item.change.child.alias}`
          break
        }
        // delete relation
        case '13': {
          item.changeDescription = `${this.$t('delete')}：${item.change.parent_id.alias} -> ${this.handleRelationType(
            item.change.relation_type_id
          )} -> ${item.change.child.alias}`
          break
        }
        case '14': {
          item.changeDescription = this.$t('cmdb.history.addReconciliation') + ': ' + item.change.name || item.change.alias
          break
        }
        case '15': {
          item.changeArr = []
          const diffs = this.deepCompare({
            obj1: item?.change?.old,
            obj2: item?.change?.new,
            directDeepKeys: ['notifies'],
            ignoreKeys: ['updated_at']
          })
          for (const val of diffs) {
            const str = ` [ ${val.path} :  ${val.value1} -> ${val.value2} ] `
            item.changeDescription += str
            item.changeArr.push(str)
          }
          if (!item.changeDescription) item.changeDescription = this.$t('cmdb.history.updateReconciliation')
          break
        }
        case '16': {
          item.changeDescription = this.$t('cmdb.history.deleteReconciliation') + ': ' + item.change.name || item.change.alias
          break
        }
      }
    },

    deepCompare({
      obj1,
      obj2,
      directDeepKeys = [],
      ignoreKeys = [],
    }) {
      const diffs = []

      function compare(obj1, obj2, path = '') {
        if (typeof obj1 !== 'object' || typeof obj2 !== 'object' || obj1 === null || obj2 === null) {
          if (obj1 !== obj2) {
            diffs.push({ path, value1: formatValue(obj1), value2: formatValue(obj2) })
          }
          return
        }

        const keys1 = new Set(Object.keys(obj1))
        const keys2 = new Set(Object.keys(obj2))
        const allKeys = new Set([...keys1, ...keys2])

        allKeys.forEach(key => {
          const newPath = path ? `${path}.${key}` : key
          if (!ignoreKeys.includes(key)) {
            if (directDeepKeys.includes(key) && !_.isEqual(obj1[key], obj2[key])) {
              diffs.push({ path: newPath, value1: formatValue(obj1[key]), value2: formatValue(obj2[key]) })
            } else if (!keys1.has(key)) {
              diffs.push({ path: newPath, value1: undefined, value2: formatValue(obj2[key]) })
            } else if (!keys2.has(key)) {
              diffs.push({ path: newPath, value1: formatValue(obj1[key]), value2: undefined })
            } else {
              compare(obj1[key], obj2[key], newPath)
            }
          }
        })
      }

      function formatValue(val) {
        return _.isObject(val) ? JSON.stringify(val) : val
      }

      compare(obj1, obj2)
      return diffs
    },

    filterOperate() {
      this.queryParams.page = 1
      this.queryParams.page_size = 50
      this.getTable(this.queryParams)
    },
    filterOperateReset() {
      this.queryParams.page = 1
      this.queryParams.page_size = 50
      this.queryParams.operate_type = undefined
      this.getTable(this.queryParams)
    },
    filterOption(input, option) {
      return option.componentOptions.children[0].text.indexOf(input) >= 0
    },

    async handleExport(params) {
      const hide = this.$message.loading(this.$t('loading'), 0)
      const res = await getCITypesTable({
        ...params,
        page: this.queryParams.page,
        page_size: this.queryParams.page_size,
      })
      hide()

      res.result.forEach((item) => {
        this.handleChangeDescription(item, item.operate_type)
        item.operate_type = this.handleOperateType(item.operate_type)
        item.type_id = this.handleTypeId(item.type_id)
        item.uid = this.handleUID(item.uid)
        if (item.operate_type.includes(this.$t('update'))) {
          item.changeDescription = item.changeArr.join(';')
        }
      })

      this.$refs.xTable.exportData({
        filename: this.$t('cmdb.history.ciTypeChange'),
        sheetName: 'Sheet1',
        type: 'xlsx',
        types: ['xlsx', 'csv', 'html', 'xml', 'txt'],
        isMerge: true,
        isColgroup: true,
        data: res.result,
      })
    },
  },
}
</script>

<style lang="less" scoped>
.row {
  margin-top: 5px;
}
.filter {
  margin-left: 10px;
  color: #c0c4cc;
  cursor: pointer;
  &:hover {
    color: #606266;
  }
}
.more-tag {
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
}
p {
  margin-bottom: 0;
}

.filter-content {
  display: flex;
  align-items: center;
  column-gap: 8px;
}
</style>
