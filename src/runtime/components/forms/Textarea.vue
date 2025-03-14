<template>
  <div :class="ui.wrapper">
    <textarea
      :id="inputId"
      ref="textarea"
      :value="modelValue"
      :name="name"
      :rows="rows"
      :required="required"
      :disabled="disabled || formDisabled"
      :placeholder="placeholder"
      :class="textareaClass"
      v-bind="attrs"
      @input="onInput"
      @blur="onBlur"
      @change="onChange"
    />

    <slot />
  </div>
</template>

<script lang="ts">
import { ref, computed, toRef, watch, onMounted, nextTick, defineComponent, inject } from 'vue'
import type { PropType, Ref } from 'vue'
import { twMerge, twJoin } from 'tailwind-merge'
import { defu } from 'defu'
import { useUI } from '../../composables/useUI'
import { useFormGroup } from '../../composables/useFormGroup'
import { mergeConfig, looseToNumber } from '../../utils'
import type { TextareaSize, TextareaColor, TextareaVariant, Strategy } from '../../types'
// @ts-expect-error
import appConfig from '#build/app.config'
import { textarea } from '#ui/ui.config'

const config = mergeConfig<typeof textarea>(appConfig.ui.strategy, appConfig.ui.textarea, textarea)

export default defineComponent({
  inheritAttrs: false,
  props: {
    modelValue: {
      type: [String, Number],
      default: ''
    },
    id: {
      type: String,
      default: null
    },
    name: {
      type: String,
      default: null
    },
    placeholder: {
      type: String,
      default: null
    },
    required: {
      type: Boolean,
      default: false
    },
    disabled: {
      type: Boolean,
      default: false
    },
    rows: {
      type: Number,
      default: 3
    },
    autoresize: {
      type: Boolean,
      default: false
    },
    autofocus: {
      type: Boolean,
      default: false
    },
    autofocusDelay: {
      type: Number,
      default: 100
    },
    resize: {
      type: Boolean,
      default: false
    },
    padded: {
      type: Boolean,
      default: true
    },
    size: {
      type: String as PropType<TextareaSize>,
      default: null,
      validator (value: string) {
        return Object.keys(config.size).includes(value)
      }
    },
    color: {
      type: String as PropType<TextareaColor>,
      default: () => config.default.color,
      validator (value: string) {
        return [...appConfig.ui.colors, ...Object.keys(config.color)].includes(value)
      }
    },
    variant: {
      type: String as PropType<TextareaVariant>,
      default: () => config.default.variant,
      validator (value: string) {
        return [
          ...Object.keys(config.variant),
          ...Object.values(config.color).flatMap(value => Object.keys(value))
        ].includes(value)
      }
    },
    textareaClass: {
      type: String,
      default: null
    },
    class: {
      type: [String, Object, Array] as PropType<any>,
      default: () => ''
    },
    ui: {
      type: Object as PropType<Partial<typeof config> & { strategy?: Strategy }>,
      default: () => ({})
    },
    modelModifiers: {
      type: Object as PropType<{ trim?: boolean, lazy?: boolean, number?: boolean }>,
      default: () => ({})
    }
  },
  emits: ['update:modelValue', 'blur'],
  setup (props, { emit }) {
    const { ui, attrs } = useUI('textarea', toRef(props, 'ui'), config, toRef(props, 'class'))

    const { emitFormBlur, emitFormInput, inputId, color, size, name } = useFormGroup(props, config)

    const modelModifiers = ref(defu({}, props.modelModifiers, { trim: false, lazy: false, number: false }))

    const textarea = ref<HTMLTextAreaElement | null>(null)

    const autoFocus = () => {
      if (props.autofocus) {
        textarea.value?.focus()
      }
    }

    const autoResize = () => {
      if (props.autoresize) {
        if (!textarea.value) {
          return
        }

        textarea.value.rows = props.rows

        const styles = window.getComputedStyle(textarea.value)
        const paddingTop = parseInt(styles.paddingTop)
        const paddingBottom = parseInt(styles.paddingBottom)
        const padding = paddingTop + paddingBottom
        const lineHeight = parseInt(styles.lineHeight)
        const { scrollHeight } = textarea.value
        const newRows = (scrollHeight - padding) / lineHeight

        if (newRows > props.rows) {
          textarea.value.rows = newRows
        }
      }
    }

    // Custom function to handle the v-model properties
    const updateInput = (value: string) => {
      if (modelModifiers.value.trim) {
        value = value.trim()
      }

      if (modelModifiers.value.number) {
        value = looseToNumber(value)
      }

      emit('update:modelValue', value)
      emitFormInput()
    }

    const onInput = (event: Event) => {
      autoResize()
      if (!modelModifiers.value.lazy) {
        updateInput((event.target as HTMLInputElement).value)
      }
    }

    const onChange = (event: Event) => {
      const value = (event.target as HTMLInputElement).value

      if (modelModifiers.value.lazy) {
        updateInput(value)
      }

      // Update trimmed input so that it has same behaviour as native input
      if (modelModifiers.value.trim) {
        (event.target as HTMLInputElement).value = value.trim()
      }
    }

    const onBlur = (event: FocusEvent) => {
      emit('blur', event)
      emitFormBlur()
    }

    onMounted(() => {
      setTimeout(() => {
        autoFocus()
      }, props.autofocusDelay)
    })

    watch(() => props.modelValue, () => {
      nextTick(autoResize)
    })

    onMounted(() => {
      setTimeout(() => {
        autoFocus()
        autoResize()
      }, 100)
    })

    const textareaClass = computed(() => {
      const variant = ui.value.color?.[color.value as string]?.[props.variant as string] || ui.value.variant[props.variant]

      return twMerge(twJoin(
        ui.value.base,
        ui.value.form,
        ui.value.rounded,
        ui.value.placeholder,
        ui.value.size[size.value],
        props.padded ? ui.value.padding[size.value] : 'p-0',
        variant?.replaceAll('{color}', color.value),
        !props.resize && 'resize-none'
      ), props.textareaClass)
    })

    const formDisabled = inject<Ref<boolean>>('form-disabled', ref(false))
    const isFormDisabled = computed(() => formDisabled.value);

    return {
      // eslint-disable-next-line vue/no-dupe-keys
      ui,
      attrs,
      // eslint-disable-next-line vue/no-dupe-keys
      name,
      inputId,
      textarea,
      // eslint-disable-next-line vue/no-dupe-keys
      textareaClass,
      formDisabled: isFormDisabled,
      onInput,
      onChange,
      onBlur
    }
  }
})
</script>
