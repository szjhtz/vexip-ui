<script setup lang="ts">
import { Icon } from '@/components/icon'
import { useFieldStore } from '@/components/form'

import { computed, ref, toRef, watch } from 'vue'

import { useHover, useModifier } from '@vexip-ui/hooks'
import {
  createSizeProp,
  createStateProp,
  emitEvent,
  useIcons,
  useLocale,
  useNameHelper,
  useProps
} from '@vexip-ui/config'
import {
  boundRange,
  debounce,
  isNull,
  minus,
  plus,
  throttle,
  toFixed,
  toNumber
} from '@vexip-ui/utils'
import { numberInputProps } from './props'
import { numberRE } from './symbol'

type InputEventType = 'input' | 'change'

const isEmpty = (value: unknown) => !value && value !== 0
const isNullOrNaN = (value: unknown) => isNull(value) || Number.isNaN(value)

defineOptions({ name: 'NumberInput' })

const {
  idFor,
  state,
  disabled,
  loading,
  size,
  validateField,
  clearField,
  getFieldValue,
  setFieldValue
} = useFieldStore<number>(focus)

const _props = defineProps(numberInputProps)
const props = useProps('numberInput', _props, {
  size: createSizeProp(size),
  state: createStateProp(state),
  locale: null,
  prefix: null,
  prefixColor: '',
  suffix: null,
  suffixColor: '',
  // 格式化后显示
  formatter: {
    default: null,
    isFunc: true
  },
  value: {
    default: () => getFieldValue(null!),
    static: true
  },
  min: -Infinity,
  max: Infinity,
  placeholder: null,
  autofocus: false,
  spellcheck: false,
  autocomplete: false,
  precision: -1,
  readonly: false,
  step: 1,
  ctrlStep: 100,
  shiftStep: 10,
  altStep: 0.1,
  disabled: () => disabled.value,
  controlClass: null,
  debounce: false,
  delay: null,
  clearable: false,
  loading: () => loading.value,
  loadingIcon: null,
  loadingLock: false,
  loadingEffect: null,
  sync: false,
  controlType: 'right',
  emptyType: 'NaN',
  controlAttrs: null,
  name: {
    default: '',
    static: true
  }
})

const emit = defineEmits(['update:value'])

const slots = defineSlots<{
  prefix: () => any,
  suffix: () => any
}>()

const nh = useNameHelper('number-input')
const locale = useLocale('numberInput', toRef(props, 'locale'))
const icons = useIcons()

const focused = ref(false)
const currentValue = ref<string | number>(props.value)
const inputting = ref(false)

const control = ref<HTMLInputElement>()
const { wrapper, isHover } = useHover()

useModifier({
  target: control,
  passive: false,
  onKeyDown: (event, modifier) => {
    emitEvent(props.onKeyDown, event)

    if (modifier.up || modifier.down) {
      event.preventDefault()
      event.stopPropagation()
      changeStep(
        modifier.up ? 'plus' : 'minus',
        event.ctrlKey ? 'ctrl' : event.shiftKey ? 'shift' : event.altKey ? 'alt' : undefined
      )
      modifier.resetAll()
    } else if (modifier.enter) {
      event.preventDefault()
      event.stopPropagation()
      emitChangeEvent('change')
      modifier.resetAll()
    }
  },
  onKeyUp: event => {
    emitEvent(props.onKeyUp, event)

    if (event.key === 'Enter') {
      handleEnter()
    }
  }
})

let lastValue: number

const outOfRange = computed(() => {
  return (
    !isNullOrNaN(currentValue.value) &&
    (toNumber(currentValue.value) > props.max || toNumber(currentValue.value) < props.min)
  )
})
const isReadonly = computed(() => (props.loading && props.loadingLock) || props.readonly)
const className = computed(() => {
  const [display, fade] = (props.controlType || 'right').split('-')

  return [
    nh.b(),
    nh.bs('vars'),
    nh.ns('input-vars'),
    {
      [nh.bm('inherit')]: props.inherit,
      [nh.bm('focused')]: inputting.value,
      [nh.bm('disabled')]: props.disabled,
      [nh.bm('readonly')]: isReadonly.value,
      [nh.bm('loading')]: props.loading,
      [nh.bm(props.size)]: props.size !== 'default',
      [nh.bm(props.state)]: props.state !== 'default',
      [nh.bm(`control-${display}`)]: display !== 'right',
      [nh.bm('control-fade')]: fade,
      [nh.bm('out-of-range')]: outOfRange.value
    }
  ]
})
const hasPrefix = computed(() => {
  return !!(slots.prefix || props.prefix)
})
const hasSuffix = computed(() => {
  return !!(slots.suffix || props.suffix)
})
const preciseNumber = computed(() => {
  return !inputting.value && typeof currentValue.value === 'number' && props.precision >= 0
    ? toFixed(currentValue.value, props.precision)
    : currentValue.value
})
const formattedValue = computed(() => {
  if (typeof preciseNumber.value !== 'number') return preciseNumber.value ?? ''

  return !inputting.value && typeof props.formatter === 'function'
    ? props.formatter(preciseNumber.value as number)
    : preciseNumber.value.toString()
})
const hasValue = computed(() => !!(currentValue.value || currentValue.value === 0))
const showClear = computed(() => {
  return !props.disabled && !isReadonly.value && props.clearable && isHover.value && hasValue.value
})
const inputValue = computed(() => {
  if (Number.isNaN(currentValue.value)) {
    return ''
  }

  return inputting.value ? preciseNumber.value : formattedValue.value
})

watch(
  () => props.value,
  value => {
    if (value !== lastValue) {
      parseValue()
    }
  },
  { immediate: true }
)

function boundValueRange(value: number) {
  return boundRange(value, props.min, props.max)
}

function parseValue() {
  let value = props.value
  value = inputting.value ? value : numberRE.test(String(value)) ? toNumber(value) : getEmptyValue()

  if (props.precision >= 0 && !isNullOrNaN(value)) {
    value = toFixed(boundValueRange(value), props.precision)
  }

  currentValue.value = value
  lastValue = value
}

function focus(options?: FocusOptions) {
  control.value?.focus(options)
}

function handleFocus(event: FocusEvent) {
  focused.value = true
  inputting.value = true
  emitEvent(props.onFocus, event)
}

function handleBlur(event: FocusEvent) {
  focused.value = false

  setTimeout(() => {
    if (!focused.value) {
      inputting.value = false
      emitEvent(props.onBlur, event)
      emitChangeEvent('change')
    }
  }, 120)
}

function plusNumber(event: MouseEvent) {
  !focused.value && focus()
  changeStep(
    'plus',
    event.ctrlKey ? 'ctrl' : event.shiftKey ? 'shift' : event.altKey ? 'alt' : undefined
  )
}

function minusNumber(event: MouseEvent) {
  !focused.value && focus()
  changeStep(
    'minus',
    event.ctrlKey ? 'ctrl' : event.shiftKey ? 'shift' : event.altKey ? 'alt' : undefined
  )
}

function changeStep(type: 'plus' | 'minus', modifier?: 'ctrl' | 'shift' | 'alt') {
  if (props.disabled || isReadonly.value) return

  let value = currentValue.value || 0
  let step!: number

  switch (modifier) {
    case 'ctrl':
      step = props.ctrlStep
      break
    case 'shift':
      step = props.shiftStep
      break
    case 'alt':
      step = props.altStep
      break
    default:
      step = props.step
  }

  const stringValue = value.toString().trim()

  if (stringValue.endsWith('.')) {
    value = toNumber(stringValue.slice(0, -1))
  }

  if (type === 'plus') {
    value = plus(value, step)
  } else {
    value = minus(value, step)
  }

  setValue(value, 'input')
}

function handleChange(event: Event) {
  const type = event.type as InputEventType
  const stringValue = (event.target as HTMLInputElement).value

  let value = stringValue.trim()

  if (type === 'change' && stringValue && !numberRE.test(stringValue)) {
    const floatValue = parseFloat(stringValue)

    if (Number.isNaN(floatValue)) {
      value = ''
    } else {
      value = floatValue.toString()
    }

    ;(event.target as HTMLInputElement).value = value
  }

  inputting.value = type === 'input'

  setValue(value, type)
}

function setValue(value: string | number, type: InputEventType, sync = props.sync) {
  if (type !== 'input') {
    currentValue.value = isEmpty(value) ? getEmptyValue() : toNumber(value)
  } else {
    currentValue.value = value
  }

  emitChangeEvent(type, sync)
}

function getEmptyValue() {
  switch (props.emptyType) {
    case 'undefined':
      return undefined! as number
    case 'null':
      return null! as number
    default:
      return NaN
  }
}

function emitChangeEvent(type: InputEventType, sync = props.sync) {
  const empty = isEmpty(currentValue.value)
  const value = empty ? getEmptyValue() : toNumber(currentValue.value)

  type = type === 'input' ? 'input' : 'change'

  if (type === 'change') {
    let boundValue = empty ? value : boundValueRange(toNumber(value))

    if (props.precision >= 0) {
      boundValue = toFixed(boundValue, props.precision)
    }

    const boundChange = !Object.is(boundValue, value)

    if (!empty) {
      currentValue.value = boundValue
    }

    if (!sync && Object.is(lastValue, boundValue)) {
      !Object.is(props.value, value) && emit('update:value', boundValue)
      return
    }

    lastValue = boundValue

    if (!sync || boundChange) {
      emit('update:value', boundValue)
      setFieldValue(boundValue)
    }

    emitEvent(props.onChange, boundValue)

    if (!sync || boundChange) {
      validateField()
    }
  } else {
    if (sync) {
      emit('update:value', value)
      setFieldValue(value)
    }

    emitEvent(props.onInput, value)

    if (sync) {
      validateField()
    }
  }
}

function handleClear() {
  if (props.disabled || isReadonly.value) return

  setValue(NaN, 'change', false)
  emitEvent(props.onClear)
  clearField()
}

function handleEnter() {
  emitEvent(props.onEnter)
}

function handlePrefixClick(event: MouseEvent) {
  emitEvent(props.onPrefixClick, event)
}

function handleSuffixClick(event: MouseEvent) {
  emitEvent(props.onSuffixClick, event)
}

function handleKeyPress(event: KeyboardEvent) {
  emitEvent(props.onKeyPress, event)
}

const delay = toNumber(props.delay)
const handleInput = props.debounce
  ? debounce(handleChange, delay || 100)
  : throttle(handleChange, delay || 16)

defineExpose({
  idFor,
  focused,
  isHover,
  outOfRange,
  preciseNumber,
  formattedValue,
  isReadonly,
  wrapper,
  input: control,
  focus,
  blur: () => control.value?.blur()
})
</script>

<template>
  <div
    :id="idFor"
    ref="wrapper"
    :class="className"
    @click="control?.focus()"
  >
    <div
      v-if="hasPrefix"
      :class="[nh.be('icon'), nh.be('prefix')]"
      :style="{ color: props.prefixColor }"
      @click="handlePrefixClick"
    >
      <slot name="prefix">
        <Icon :icon="props.prefix"></Icon>
      </slot>
    </div>
    <input
      v-bind="props.controlAttrs"
      ref="control"
      :class="[nh.be('control'), props.controlAttrs?.class, props.controlClass]"
      :value="inputValue"
      type="text"
      :autofocus="props.autofocus"
      :autocomplete="props.autocomplete ? 'on' : 'off'"
      :spellcheck="props.spellcheck"
      :disabled="props.disabled"
      :readonly="isReadonly"
      :placeholder="props.placeholder ?? locale.placeholder"
      :name="props.name || props.controlAttrs?.name"
      role="spinbutton"
      :title="outOfRange ? locale.outOfRange : undefined"
      :aria-valuenow="preciseNumber"
      :aria-valuemin="props.min !== -Infinity ? props.min : undefined"
      :aria-valuemax="props.max !== Infinity ? props.max : undefined"
      @submit.prevent
      @blur="handleBlur"
      @focus="handleFocus"
      @keypress="handleKeyPress"
      @input="handleInput"
      @change="handleChange"
    />
    <div
      v-if="hasSuffix"
      :class="[nh.be('icon'), nh.be('suffix')]"
      :style="{
        color: props.suffixColor,
        opacity: showClear || props.loading ? '0%' : ''
      }"
      @click="handleSuffixClick"
    >
      <slot name="suffix">
        <Icon :icon="props.suffix"></Icon>
      </slot>
    </div>
    <div
      v-else-if="props.clearable || props.loading"
      :class="[nh.be('icon'), nh.bem('icon', 'placeholder'), nh.be('suffix')]"
    ></div>
    <Transition :name="nh.ns('fade')" appear>
      <div v-if="showClear" :class="[nh.be('icon'), nh.be('clear')]" @click.stop="handleClear">
        <Icon v-bind="icons.clear" label="clear"></Icon>
      </div>
      <div v-else-if="props.loading" :class="[nh.be('icon'), nh.be('loading')]">
        <Icon
          v-bind="icons.loading"
          :effect="props.loadingEffect || icons.loading.effect"
          :icon="props.loadingIcon || icons.loading.icon"
          label="loading"
        ></Icon>
      </div>
    </Transition>
    <template v-if="props.controlType !== 'none'">
      <div :class="nh.be('plus')" @click="plusNumber" @mousedown.prevent>
        <Icon v-bind="icons.caretUp" :scale="+(icons.caretUp.scale || 1) * 0.8"></Icon>
      </div>
      <div :class="nh.be('minus')" @click="minusNumber" @mousedown.prevent>
        <Icon v-bind="icons.caretDown" :scale="+(icons.caretDown.scale || 1) * 0.8"></Icon>
      </div>
    </template>
  </div>
</template>
