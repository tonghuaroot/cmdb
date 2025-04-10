<template>
  <a-form :form="form">
    <a-divider style="font-size:14px;margin:14px 0;font-weight:700;">{{ group.name || $t('other') }}</a-divider>
    <a-row :gutter="24" align="top" type="flex">
      <a-col
        :span="12"
        v-for="(attr, attr_idx) in group.attributes.filter(
          (item) =>
            !(
              item.default &&
              item.default.default &&
              typeof item.default.default === 'string' &&
              item.default.default.startsWith('$')
            )
        )"
        :key="attr.name + attr_idx"
      >
        <a-form-item :label="attr.alias || attr.name" :colon="false">
          <CIReferenceAttr
            v-if="attr.is_reference"
            :referenceTypeId="attr.reference_type_id"
            :isList="attr.is_list"
            v-decorator="[
              attr.name,
              {
                rules: [{ required: attr.is_required, message: $t('placeholder2') + `${attr.alias || attr.name}` }],
                initialValue: attr.is_list ? [] : ''
              }
            ]"
          />
          <a-switch
            v-else-if="attr.is_bool"
            v-decorator="[
              attr.name,
              {
                rules: [{ required: false }],
                valuePropName: 'checked',
                initialValue: attr.default ? Boolean(attr.default.default) : false
              }
            ]"
          />
          <a-select
            :style="{ width: '100%' }"
            v-decorator="[
              attr.name,
              {
                rules: [{ required: attr.is_required, message: $t('placeholder2') + `${attr.alias || attr.name}` }],
                initialValue: getChoiceDefault(attr),
              },
            ]"
            :placeholder="$t('placeholder2')"
            v-else-if="attr.is_choice"
            :mode="attr.is_list ? 'multiple' : 'default'"
            showSearch
            allowClear
          >
            <a-select-option
              :value="choice[0]"
              :key="'New_' + attr.name + choice_idx"
              v-for="(choice, choice_idx) in attr.choice_value"
            >
              <span :style="{ ...(choice[1] ? choice[1].style : {}), display: 'inline-flex', alignItems: 'center' }">
                <template v-if="choice[1] && choice[1].icon && choice[1].icon.name">
                  <img
                    v-if="choice[1].icon.id && choice[1].icon.url"
                    :src="`/api/common-setting/v1/file/${choice[1].icon.url}`"
                    :style="{ maxHeight: '13px', maxWidth: '13px', marginRight: '5px' }"
                  />
                  <ops-icon
                    v-else
                    :style="{ color: choice[1].icon.color, marginRight: '5px' }"
                    :type="choice[1].icon.name"
                  />
                </template>
                <a-tooltip placement="topLeft" :title="choice[1] ? choice[1].label || choice[0] : choice[0]">
                  {{ choice[1] ? choice[1].label || choice[0] : choice[0] }}
                </a-tooltip>
              </span>
            </a-select-option>
          </a-select>
          <a-input
            v-else-if="attr.is_list"
            :style="{ width: '100%' }"
            v-decorator="[
              attr.name,
              {
                rules: [{ required: attr.is_required, message: $t('placeholder2') + `${attr.alias || attr.name}` }],
                initialValue: attr.default && attr.default.default ? Array.isArray(attr.default.default) ? attr.default.default.join(',') : attr.default.default : '',
              },
            ]"
          >
          </a-input>
          <a-input-number
            v-decorator="[
              attr.name,
              {
                rules: [{ required: attr.is_required, message: $t('placeholder1') + `${attr.alias || attr.name}` }],
                initialValue: attr.default && attr.default.default !== undefined && attr.default.default !== null ? attr.default.default : null,
              },
            ]"
            style="width: 100%"
            v-else-if="attr.value_type === '0' || attr.value_type === '1'"
          />
          <a-date-picker
            v-decorator="[
              attr.name,
              {
                rules: [{ required: attr.is_required, message: $t('placeholder2') + `${attr.alias || attr.name}` }],
                initialValue: attr.default && attr.default.default ? moment(attr.default.default) : null,
              },
            ]"
            style="width: 100%"
            :format="attr.value_type === '4' ? 'YYYY-MM-DD' : 'YYYY-MM-DD HH:mm:ss'"
            v-else-if="attr.value_type === '4' || attr.value_type === '3'"
            :showTime="attr.value_type === '4' ? false : { format: 'HH:mm:ss' }"
          />
          <a-input
            @focus="(e) => handleFocusInput(e, attr)"
            v-decorator="[
              attr.name,
              {
                validateTrigger: ['submit'],
                rules: [{ required: attr.is_required, message: $t('placeholder1') + `${attr.alias || attr.name}` }],
                initialValue: attr.default && attr.default.default ? JSON.stringify(attr.default.default) : '',
              },
            ]"
            style="width: 100%"
            v-else-if="attr.value_type === '6'"
          />
          <a-input
            @focus="(e) => handleFocusInput(e, attr)"
            v-decorator="[
              attr.name,
              {
                validateTrigger: ['submit'],
                rules: [{ required: attr.is_required, message: $t('placeholder1') + `${attr.alias || attr.name}` }],
                initialValue: attr.default && attr.default.default ? attr.default.default : null,
              },
            ]"
            style="width: 100%"
            v-else
          />
        </a-form-item>
      </a-col>
    </a-row>
    <JsonEditor ref="jsonEditor" @jsonEditorOk="jsonEditorOk" />
  </a-form>
</template>

<script>
import _ from 'lodash'
import moment from 'moment'
import JsonEditor from '../../../components/JsonEditor/jsonEditor.vue'
import CIReferenceAttr from '@/components/ciReferenceAttr/index.vue'

export default {
  name: 'CreateInstanceFormByGroup',
  components: {
    JsonEditor,
    CIReferenceAttr
  },
  props: {
    group: {
      type: Object,
      default: () => {},
    },
    attributeList: {
      type: Array,
      default: () => [],
    },
    ciTypeId: {
      type: [Number, String],
      default: ''
    }
  },
  inject: ['getFieldType'],
  data() {
    return {
      form: this.$form.createForm(this, this.group.id),
      editAttr: null,
    }
  },
  methods: {
    moment,
    handleFocusInput(e, attr) {
      console.log(attr)
      const _tempFind = this.attributeList.find((item) => item.name === attr.name)
      if (_tempFind.value_type === '6') {
        this.editAttr = attr
        e.srcElement.blur()
        const jsonData = this.form.getFieldValue(attr.name)
        this.$refs.jsonEditor.open(null, null, jsonData ? JSON.parse(jsonData) : {})
      } else {
        this.editAttr = null
      }
    },
    getData() {
      let data = null
      this.form.validateFields((err, values) => {
        if (err) {
          data = 'error'
          throw new Error(err)
        } else {
          data = values
        }
      })
      return data
    },
    jsonEditorOk(jsonData) {
      this.form.setFieldsValue({ [this.editAttr.name]: JSON.stringify(jsonData) })
    },

    getChoiceDefault(attr) {
      if (_.isNil(attr?.default?.default)) {
        return attr.is_list ? [] : null
      }

      if (attr.is_list) {
        let defaultValue = []
        if (Array.isArray(attr.default.default)) {
          defaultValue = attr.default.default
        } else {
          defaultValue = String(attr.default.default).split(',')
        }
        if (['0', '1', '11'].includes(attr.value_type)) {
          defaultValue = defaultValue?.map((item) => {
            const numberValue = Number(item)
            return Number.isNaN(numberValue) ? item : numberValue
          })
        }
        return defaultValue
      }

      let defaultValue = attr.default.default
      if (['0', '1', '11'].includes(attr.value_type)) {
        const numberValue = Number(defaultValue)
        defaultValue = Number.isNaN(numberValue) ? attr.default.default : numberValue
      }
      return defaultValue
    }
  },
}
</script>

<style></style>
