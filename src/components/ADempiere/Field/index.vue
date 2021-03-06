<template>
  <!--
    this v-show is to indicate that if the field is not shown,
    therefore you should not leave the column size spacing of your
    <el-col></el-col> container-->
  <el-col
    v-if="!inTable"
    v-show="isDisplayed"
    key="is-panel-template"
    :xs="sizeFieldResponsive.xs"
    :sm="sizeFieldResponsive.sm"
    :md="sizeFieldResponsive.md"
    :lg="sizeFieldResponsive.lg"
    :xl="sizeFieldResponsive.xl"
    :class="classField"
  >
    <el-form-item
      :required="isMandatory()"
    >
      <template slot="label">
        <operator-comparison
          v-if="isAdvancedQuery && isDisplayed"
          key="is-field-operator-comparison"
          :field-attributes="fieldAttributes"
          :field-value="field.value"
        />
        <context-info
          v-else-if="isContextInfo"
          key="is-field-context-info"
          :field-attributes="fieldAttributes"
          :field-value="field.value"
        />
        <span v-else key="is-field-name">
          {{ isFieldOnly() }}
        </span>

        <document-status
          v-if="isDocuemntStatus"
          :field="fieldAttributes"
        />

        <translated
          v-if="field.isTranslated && !isAdvancedQuery"
          :field-attributes="fieldAttributes"
          :record-uuid="field.recordUuid"
        />
        <calculator
          v-if="!isAdvancedQuery && isNumeric"
          :field-attributes="fieldAttributes"
          :field-value="field.value"
        />
      </template>
      <component
        :is="componentRender"
        :ref="field.columnName"
        :metadata="fieldAttributes"
        :value-model="recordDataFields"
      />
    </el-form-item>
  </el-col>
  <component
    :is="componentRender"
    v-else
    key="is-table-template"
    :class="classField"
    :metadata="fieldAttributes"
    :value-model="recordDataFields"
    :in-table="true"
  />
</template>

<script>
import contextInfo from '@/components/ADempiere/Field/popover/contextInfo'
import documentStatus from '@/components/ADempiere/Field/popover/documentStatus'
import operatorComparison from '@/components/ADempiere/Field/popover/operatorComparison'
import translated from '@/components/ADempiere/Field/popover/translated'
import calculator from '@/components/ADempiere/Field/popover/calculator'
import { FIELD_ONLY } from '@/components/ADempiere/Field/references'
import { DEFAULT_SIZE } from '@/components/ADempiere/Field/fieldSize'
import { fieldIsDisplayed } from '@/utils/ADempiere/dictionaryUtils'
import { showMessage } from '@/utils/ADempiere/notification'

/**
 * This is the base component for linking the components according to the
 * reference (or type of visualization) of each field
 */
export default {
  name: 'FieldDefinition',
  components: {
    contextInfo,
    documentStatus,
    operatorComparison,
    translated,
    calculator
  },
  props: {
    parentUuid: {
      type: String,
      default: ''
    },
    containerUuid: {
      type: String,
      default: ''
    },
    panelType: {
      type: String,
      default: 'window'
    },
    // receives the property that is an object with all the attributes
    metadataField: {
      type: Object,
      default: () => ({})
    },
    recordDataFields: {
      type: [Number, String, Boolean, Array, Object, Date],
      default: undefined
    },
    inGroup: {
      type: Boolean,
      default: false
    },
    inTable: {
      type: Boolean,
      default: false
    },
    isAdvancedQuery: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      field: {}
    }
  },
  computed: {
    // load the component that is indicated in the attributes of received property
    componentRender() {
      if (this.isSelectCreated) {
        return () => import(`@/components/ADempiere/Field/FieldSelectMultiple`)
      }
      return () => import(`@/components/ADempiere/Field/${this.field.componentPath}`)
    },
    fieldAttributes() {
      return {
        ...this.field,
        panelType: this.panelType,
        inTable: this.inTable,
        isAdvancedQuery: this.isAdvancedQuery,
        // DOM properties
        required: this.isMandatory(),
        readonly: this.isReadOnly(),
        displayed: this.isDisplayed,
        disabled: !this.field.isActive,
        isSelectCreated: this.isSelectCreated
      }
    },
    isDisplayed() {
      if (this.isAdvancedQuery) {
        return this.field.isShowedFromUser
      }
      return fieldIsDisplayed(this.field) && (this.isMandatory() || this.field.isShowedFromUser || this.inTable)
    },
    isSelectCreated() {
      return this.isAdvancedQuery &&
        !['FieldBinary', 'FieldDate', 'FieldSelect', 'FieldYesNo'].includes(this.field.componentPath) &&
        ['IN', 'NOT_IN'].includes(this.field.operator)
    },
    getWidth() {
      return this.$store.getters.getWidthLayout
    },
    classField() {
      if (this.inTable) {
        return 'in-table'
      }
      return ''
    },
    getterIsShowedRecordNavigation() {
      if (this.panelType === 'window') {
        return this.$store.getters.getIsShowedRecordNavigation(this.parentUuid)
      }
      return false
    },
    sizeFieldResponsive() {
      if (!this.isDisplayed) {
        return DEFAULT_SIZE
      }

      const sizeField = this.field.sizeFieldFromType.size
      const newSizes = {}

      // in table set max width, used by browser result and tab children of window
      if (this.inTable) {
        newSizes.xs = 24
        newSizes.sm = 24
        newSizes.md = 24
        newSizes.lg = 24
        newSizes.xl = 24
        return newSizes
      }
      if (this.isAdvancedQuery) {
        newSizes.xs = 24
        newSizes.sm = 24
        newSizes.md = 12
        newSizes.lg = 12
        newSizes.xl = 12
        return newSizes
      }

      if (this.panelType === 'window') {
        if (this.field.componentPath === 'FieldTextLong') {
          return sizeField
        }
        // two columns if is mobile or desktop and show record navigation
        if (this.getWidth <= 768 || (this.getWidth >= 768 && this.getterIsShowedRecordNavigation)) {
          newSizes.xs = 12
          newSizes.sm = 12
          newSizes.md = 12
          newSizes.lg = 12
          newSizes.xl = 12
          return newSizes
        } else if (this.inGroup && this.getWidth >= 992) {
          newSizes.xs = sizeField.xs
          newSizes.sm = sizeField.sm * 2
          if (this.getWidth <= 1199) {
            newSizes.md = sizeField.md
          } else {
            newSizes.md = sizeField.md * 2
          }
          if (this.field.groupAssigned !== '') {
            newSizes.lg = sizeField.lg * 2
            newSizes.xl = sizeField.xl * 2
          } else {
            newSizes.lg = sizeField.lg
            newSizes.xl = sizeField.xl
          }
          return newSizes
        }
        return sizeField
      }
      return sizeField
    },
    getterContextProcessing() {
      const processing = this.$store.getters.getContextProcessing(this.parentUuid)
      if (processing === true || processing === 'Y') {
        return true
      }
      return false
    },
    getterContextProcessed() {
      const processed = this.$store.getters.getContextProcessed(this.parentUuid)
      if (processed === true || processed === 'Y') {
        return true
      }
      return false
    },
    processOrderUuid() {
      return this.$store.getters.getOrders
    },
    isDocuemntStatus() {
      if (this.panelType === 'window') {
        if (this.field.columnName === 'DocStatus' && !this.isEmptyValue(this.processOrderUuid)) {
          return true
        }
      }
      return false
    },
    isContextInfo() {
      if (!this.isAdvancedQuery) {
        return (this.field.contextInfo && this.field.contextInfo.isActive) || this.field.reference.windowsList.length
      }
      return false
    },
    isNumeric() {
      return this.field.componentPath === 'FieldNumber'
    }
  },
  watch: {
    metadataField(value) {
      this.field = value
    }
  },
  created() {
    // assined field with prop
    this.field = this.metadataField
  },
  methods: {
    showMessage,
    isReadOnly() {
      if (this.isAdvancedQuery) {
        if (['NULL', 'NOT_NULL'].includes(this.field.operator)) {
          return true
        }
        return false
      }

      if (!this.field.isActive) {
        return true
      }

      const isUpdateableAllFields = this.field.isReadOnly || this.field.isReadOnlyFromLogic

      if (this.panelType === 'window') {
        if (this.field.isAlwaysUpdateable) {
          return false
        }
        if (this.getterContextProcessing) {
          return true
        }
        if (this.getterContextProcessed) {
          return true
        }

        // TODO: Evaluate record uuid without route.action
        // edit mode is diferent to create new
        let isWithRecord = this.field.recordUuid !== 'create-new'
        if (this.inTable) {
          isWithRecord = !this.isEmptyValue(this.field.recordUuid)
        }

        return (!this.field.isUpdateable && isWithRecord) || (isUpdateableAllFields || this.field.isReadOnlyFromForm)
      } else if (this.panelType === 'browser') {
        if (this.inTable) {
          // browser result
          return this.field.isReadOnly
        }
        // query criteria
        return this.field.isReadOnlyFromLogic
      }
      // other type of panels (process/report)
      return isUpdateableAllFields
    },
    isMandatory() {
      if (this.isAdvancedQuery) {
        return false
      }
      return this.field.isMandatory || this.field.isMandatoryFromLogic
    },
    isFieldOnly() {
      if (this.inTable || this.field.isFieldOnly || this.verifyIsFieldOnly()) {
        return undefined
      }
      return this.field.name
    },
    /**
     * TODO: Evaluate the current field with the only fields contained in the
     * constant FIELD_ONLY
     * @return {boolean}
     */
    verifyIsFieldOnly() {
      const field = FIELD_ONLY.find(itemField => {
        if (this.field.displayType === itemField.id) {
          return true
        }
      })
      return Boolean(field)
    },
    focusField() {
      if (this.isDisplayed && this.isMandatory() && !this.isReadOnly()) {
        this.$refs[this.field.columnName].activeFocus()
      }
    }
  }
}
</script>

<style lang="scss">
  .custom-tittle-popover {
    font-size: 14px;
    font-weight: bold;
    float: left;
  }
  /**
   * Separation between elements (item) of the form
   */
  .el-form-item {
    margin-bottom: 10px !important;
    margin-left: 10px;
    margin-right: 10px;
  }
  /**
   * Reduce the spacing between the form element and its label
   */
  .el-form--label-top .el-form-item__label {
    padding-bottom: 0px !important;
  }

  .in-table {
    margin-bottom: 0px !important;
    margin-left: 0px;
    margin-right: 0px;
  }

  /* Global Styles */
  .el-textarea__inner:not(.in-table) {
    min-height: 36px !important;
    /*
    height: 36px auto !important;
    max-height: 54.2333px !important;
    */
  }

  /**
   * Reduce the spacing between the form element and its label
   */
  .el-form--label-top .el-form-item__label {
    padding-bottom: 0px !important;
  }
  .pre-formatted {
    white-space: pre;
  }
</style>
