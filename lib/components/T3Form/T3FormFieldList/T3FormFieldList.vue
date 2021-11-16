<template>
  <div>
    <div
      v-for="field in elements"
      :key="field.identifier"
      :class="[`t3-form-field-${field.identifier}`, getCssClass(field, classes)]"
    >
      <template v-if="field.type === 'Fieldset'">
        <T3FormFieldset :field="field">
          <T3FormFieldList
            ref="nested"
            :elements="field.elements"
            :components="components"
            :classes="classes"
            @change="updateModel"
          />
        </T3FormFieldset>
      </template>
      <template v-else>
        <component
          :is="getComponentField(field)"
          v-model="model[field.name]"
          :field="field"
        />
      </template>
    </div>
  </div>
</template>
<script>
import Vue from 'vue'
import { pascalCase } from 'scule'
import { ValidationProvider, extend } from 'vee-validate'
import extendableComponents from '../../../mixins/component/extendableComponents'
import T3FormFieldset from '../T3FormFieldset/T3FormFieldset.vue'
import T3FormField from '../T3FormField/T3FormField.vue'
import T3FormFieldHidden from '../T3FormFieldHidden/T3FormFieldHidden.vue'
import T3FormFieldSelect from '../T3FormFieldSelect/T3FormFieldSelect.vue'
import * as validators from '../T3FormValidators'

export default Vue.extend({
  name: 'T3FormFieldList',
  components: {
    ValidationProvider,
    T3FormFieldset,
    T3FormField,
    T3FormFieldHidden,
    T3FormFieldSelect,
    T3FormFieldSingleSelect: T3FormFieldSelect
  },
  mixins: [extendableComponents],
  props: {
    elements: {
      type: Array,
      required: true
    },
    components: {
      type: [Object, Boolean],
      default: false
    },
    classes: {
      type: [Object, Boolean],
      default: false
    },
    extendRules: {
      type: Array,
      default: () => []
    },
    extendTypes: {
      type: Object,
      default: () => ({})
    }
  },
  data () {
    return {
      rules: [],
      availableCustomTypes: {
        Honeypot: 'T3FormFieldHidden'
      },
      model: {}
    }
  },
  computed: {
    customTypes () {
      return ({
        ...this.availableCustomTypes,
        ...this.extendTypes
      })
    }
  },
  watch: {
    model: {
      handler (newModel) {
        this.$emit('change', Object.assign(this.model, newModel))
      },
      deep: true
    }
  },
  created () {
    if (this.components) {
      Object.assign(this.$options.components, this.components)
    }

    if (this.elements.length) {
      this.rules = this.getValidationRules(this.elements)
    }

    if (this.rules.length) {
      this.extendValidation(this.rules)
    }

    this.setupModel()
  },
  methods: {
    setupModel () {
      if (this.elements.length) {
        this.model = this.createModel(this.elements)
      }
    },
    updateModel (model) {
      this.$emit('change', Object.assign(this.model, model))
    },
    restoreModel () {
      this.setupModel()
      this.$refs?.nested?.[0].restoreModel()
    },
    createModel (elements) {
      return Object.assign({}, this.createModelByFieldset(elements))
    },
    createModelByFieldset (elements) {
      const fieldset = {}
      elements.forEach((element) => {
        if (element.name) {
          fieldset[element.name] = element.defaultValue || ''
        }
      })
      return fieldset
    },
    /**
     * Get recursively all validation rules and designate uniqe
     * @param {Elements} elements Elements array
     * @returns {Validator[]} Array with unique validation rules
     */
    getValidationRules (elements) {
      let rules = []
      elements.forEach((element) => {
        if (element.elements) {
          rules = rules.concat(this.getValidationRulesByFieldset(element.elements))
          return rules
        }

        rules = rules.concat(this.getValidationRulesByFieldset(elements))
      })

      return [...new Set(rules.map(validator => JSON.stringify(validator)))].map(validator => JSON.parse(validator))
    },

    /**
     * Get all validation rules from one fieldset
     * @param {Elements} elements Elements array
     * @returns {Validator[]} Array with validation rules
     */
    getValidationRulesByFieldset (elements) {
      let rules = []
      elements.forEach((element) => {
        if (element.validators) {
          rules = rules.concat(element.validators)
        }
      })
      return rules
    },

    /**
     * Extend vee-validate with custom validation rules
     * @param {Validator[]} rules Array with validation rules
     */
    extendValidation (rules) {
      rules.forEach((rule) => {
        extend(rule.identifier, {
          /* eslint import/namespace: [2, { allowComputed: true }] */
          ...validators[rule.identifier],
          message: rule.errorMessage
        })
      })
    },
    /**
     * Get component by field type
     * @param {Element} field single field
     * @returns {String} field component name
     */
    getComponentField (field) {
      if (Object.keys(this.components).includes(field.identifier)) {
        return field.identifier
      }
      if (Object.keys(this.customTypes).includes(field.type)) {
        return this.customTypes[field.type]
      }
      if (this.$options.components && this.$options.components[`T3FormField${pascalCase(field.identifier)}`]) {
        return `T3FormField${pascalCase(field.identifier)}`
      }
      if (this.$options.components && this.$options.components[`T3FormField${field.type}`]) {
        return `T3FormField${field.type}`
      }
      return 'T3FormField'
    },
    /**
     * Get css class for single row based on fields
     * @param {Element} field single field
     * @returns {String} css class
     */
    getCssClass (field, classes) {
      if (classes && field) {
        if (Object.keys(classes).includes(field.identifier)) {
          return classes[field.identifier]
        }
      }
    }
  }
})
</script>