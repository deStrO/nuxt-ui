<template>
  <div :class="ui.wrapper">
    <div :class="ui.container">
      <input
        :id="inputId"
        v-model="toggle"
        :name="name"
        :required="required"
        :value="value"
        :disabled="disabled || formDisabled"
        :checked="checked"
        :indeterminate="indeterminate"
        type="checkbox"
        :class="inputClass"
        v-bind="attrs"
        @change="onChange"
      >
    </div>
    <div v-if="label || $slots.label" :class="ui.inner">
      <label :for="inputId" :class="ui.label">
        <slot name="label">{{ label }}</slot>
        <span v-if="required" :class="ui.required">*</span>
      </label>
      <p v-if="help" :class="ui.help">
        {{ help }}
      </p>
    </div>
  </div>
</template>

<script lang="ts">
import {computed, toRef, defineComponent, inject, ref} from 'vue'
import type { PropType, Ref } from 'vue'
import { twMerge, twJoin } from 'tailwind-merge'
import { useUI } from '../../composables/useUI'
import { useFormGroup } from '../../composables/useFormGroup'
import { mergeConfig } from '../../utils'
import type { Strategy } from '../../types'
// @ts-expect-error
import appConfig from '#build/app.config'
import { checkbox } from '#ui/ui.config'
import colors from '#ui-colors'
import {FormError} from "../../types";

const config = mergeConfig<typeof checkbox>(appConfig.ui.strategy, appConfig.ui.checkbox, checkbox)

export default defineComponent({
  inheritAttrs: false,
  props: {
    id: {
      type: String,
      default: () => null
    },
    value: {
      type: [String, Number, Boolean, Object],
      default: null
    },
    modelValue: {
      type: [Boolean, Array],
      default: null
    },
    name: {
      type: String,
      default: null
    },
    disabled: {
      type: Boolean,
      default: false
    },
    checked: {
      type: Boolean,
      default: false
    },
    indeterminate: {
      type: Boolean,
      default: false
    },
    help: {
      type: String,
      default: null
    },
    label: {
      type: String,
      default: null
    },
    required: {
      type: Boolean,
      default: false
    },
    color: {
      type: String as PropType<typeof colors[number]>,
      default: () => config.default.color,
      validator (value: string) {
        return appConfig.ui.colors.includes(value)
      }
    },
    inputClass: {
      type: String,
      default: ''
    },
    class: {
      type: [String, Object, Array] as PropType<any>,
      default: () => ''
    },
    ui: {
      type: Object as PropType<Partial<typeof config> & { strategy?: Strategy }>,
      default: () => ({})
    }
  },
  emits: ['update:modelValue', 'change'],
  setup (props, { emit }) {
    const { ui, attrs } = useUI('checkbox', toRef(props, 'ui'), config, toRef(props, 'class'))

    const { emitFormChange, color, name, inputId } = useFormGroup(props)

    const toggle = computed({
      get () {
        return props.modelValue
      },
      set (value) {
        emit('update:modelValue', value)
      }
    })

    const onChange = (event: Event) => {
      emit('change', event)
      emitFormChange()
    }

    const inputClass = computed(() => {
      return twMerge(twJoin(
        ui.value.base,
        ui.value.form,
        ui.value.rounded,
        ui.value.background,
        ui.value.border,
        color.value && ui.value.ring.replaceAll('{color}', color.value),
        color.value && ui.value.color.replaceAll('{color}', color.value)
      ), props.inputClass)
    })

    const formDisabled = inject<Ref<boolean>>('form-disabled', ref(false))
    const isFormDisabled = computed(() => formDisabled.value);

    return {
      // eslint-disable-next-line vue/no-dupe-keys
      ui,
      attrs,
      toggle,
      inputId,
      // eslint-disable-next-line vue/no-dupe-keys
      name,
      // eslint-disable-next-line vue/no-dupe-keys
      inputClass,
      onChange,
      formDisabled: isFormDisabled
    }
  }
})
</script>
