<script setup>
import {computed, onMounted, onUnmounted, reactive, ref, useSlots, watch, watchEffect} from "vue";
import {useI18n} from "vue-i18n";
import {extractNumberFromString, generateRandomID} from "@/utils/helper.js";
import PageHeader from "@/components/workplaces/PageHeader.vue";

const id = generateRandomID();
const {t} = useI18n();
const slots = useSlots();
const model = defineModel({
  type: [String, Number, Array, null],
  default: (props) => props.value ?? props.multiple ? [] : null,
});
const props = defineProps({
  items: {
    type: Array,
    default: []
  },
  label: String,
  searchable: Boolean,
  placeholder: String,
  itemTitle: String,
  itemValue: String,
  value: [String, Number, Array],
  multiple: Boolean,
  required: Boolean,
  icon: {
    type: String,
    default: ''
  },
  disabled: Boolean,
  trackBy: {
    type: String,
    default: (props) => props.itemTitle
  },
});

const activeValues = computed(() => {
  return props.items.filter(el => {
    if (props.multiple && Array.isArray(model.value)) {
      return model.value.includes(el[props.itemValue]);
    } else {
      return model.value === el[props.itemValue]
    }
  }).map(el => {
    return {
      title: el[props.itemTitle],
      value: el[props.itemValue]
    }
  });
});

const appSelectInput = ref(null);

const borderRadius = reactive({
  topLeft: '0',
  topRight: '0',
  bottomLeft: '0',
  bottomRight: '0',
});

const setBorderRadius = () => {
  const appSelectInputStyle = window.getComputedStyle(appSelectInput.value);
  if (appSelectInputStyle) {
    borderRadius.topLeft = appSelectInputStyle.borderTopLeftRadius;
    borderRadius.bottomLeft = appSelectInputStyle.borderBottomLeftRadius;
    borderRadius.topRight = appSelectInputStyle.borderTopRightRadius;
    borderRadius.bottomRight = appSelectInputStyle.borderBottomRightRadius;
  }
}

const dropdownPosition = ref('bottom');

const getBorderRadius = (type) => {
  if (!isOpen.value) return ''
  if ((type === 'input' && dropdownPosition.value === 'top') || (type === 'dropdown' && dropdownPosition.value === 'bottom')) {
    return `0 0 ${borderRadius.bottomRight} ${borderRadius.bottomLeft}`;
  } else if ((type === 'input' && dropdownPosition.value === 'bottom') || (type === 'dropdown' && dropdownPosition.value === 'top')) {
    return `${borderRadius.topLeft} ${borderRadius.topRight} 0 0`;
  }
}

const appSelect = ref(null);
const handleClickOutside = (event) => {
  if (!isOpen.value) return
  if (appSelect.value && !appSelect.value?.contains(event.target)) isOpen.value = false;
};

onMounted(() => {
  setBorderRadius();
  document.addEventListener('click', handleClickOutside);
});

onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside);
});

const appSelectInputClass = computed(() => {
  const className = ['app-select__input'];
  if (props.searchable) className.push('app-select__input--searchable');
  return className;
});

const isOpen = ref(false);

const appSelectClass = computed(() => {
  const className = ['app-select'];
  if (isOpen.value) className.push('app-select--open');
  if (props.icon) className.push('app-select--icon');
  return className;
});

const dropdownClass = computed(() => {
  const className = ['app-select__dropdown'];
  if (!isOpen.value) className.push('app-select__dropdown--hidden');
  if (dropdownPosition.value === 'top') className.push('app-select__dropdown--top');
  return className;
});

const pointerIndex = ref(0);

const optionClass = (value, index) => {
  const className = ['app-select__dropdown-item'];
  if (pointerIndex.value === index) className.push('app-select__dropdown-item--focus');
  if (!!activeValues.value.find(el => el.value === value)) {
    if (props.multiple) className.push('app-select__dropdown-item--multi-active');
    else className.push('app-select__dropdown-item--active');
  }

  return className;
}

const emit = defineEmits(["change"]);

const deleteModelValue = (index) => {
  model.value.splice(index, 1);
}

const clickOption = (option) => {
  const value = option[props.itemValue] ?? null;
  if (!value) return

  if (props.multiple) {
    const index = model.value?.indexOf(value);
    if (index === -1) model.value.push(value);
    else deleteModelValue(index);
  } else model.value = value;
  isOpen.value = false;
  emit("change", option);
}

const appSelectDropdownOptions = ref(null);
const appSelectDropdownWrapper = ref(null);

const scrollToActiveOption = (type) => {
  const firstItem = appSelectDropdownOptions.value[pointerIndex.value];
  const itemHeight = firstItem ? firstItem?.offsetHeight : null;
  if (!itemHeight) return
  const pointerPosition = pointerIndex.value * itemHeight;
  if (type === 'down') {
    let height = window.getComputedStyle(appSelectDropdownWrapper.value).height;
    height = extractNumberFromString(height);
    const visibleElements = height / itemHeight;
    const newDownPosition = pointerPosition - (visibleElements - 1) * itemHeight;
    if (appSelectDropdownWrapper.value.scrollTop <= newDownPosition) appSelectDropdownWrapper.value.scrollTop = newDownPosition;
  } else if ((type === 'up' && appSelectDropdownWrapper.value.scrollTop >= pointerPosition) || type === 'search') {
    appSelectDropdownWrapper.value.scrollTop = pointerPosition;
  }
}

const keyDown = () => {
  if (isOpen.value && pointerIndex.value < props.items.length - 1) {
    pointerIndex.value += 1;
    scrollToActiveOption("down");
  }
}

const keyUp = () => {
  if (isOpen.value && pointerIndex.value > 0) {
    pointerIndex.value -= 1;
    scrollToActiveOption('up');
  }
}

const addOption = () => {
  if (!isOpen.value) return
  const option = props.items[pointerIndex.value];
  clickOption(option);
}

const startSearch = (event) => {
  const value = event.target.value;
  if (isOpen && props.searchable && value.trim().length > 0) {
    const filterOptions = props.items.filter(el => el[props.trackBy].toLowerCase().includes(value.toLowerCase()));
    const activeOption = filterOptions?.length > 0 ? filterOptions[0] : null;
    if (activeOption) {
      setPointerIndex(activeOption[props.itemValue]);
      scrollToActiveOption('search');
    } else pointerIndex.value = 0;
  }
}

const setPointerIndex = (value) => {
  const newIndex = props.items.findIndex(option => option[props.itemValue] === value);
  pointerIndex.value = newIndex && newIndex !== -1 ? newIndex : 0;
}

const appSelectDropdown = ref(null);

const setPosition = () => {
  const dropdownRect = appSelectDropdown.value?.getBoundingClientRect();
  const checkPosition = dropdownPosition.value === 'bottom' ? dropdownRect?.bottom > window.innerHeight : dropdownRect?.top > 80;
  dropdownPosition.value = checkPosition ? 'top' : 'bottom';
}

const inputValue = ref('');

watchEffect(() => {
  if (!props.multiple && activeValues.value.length > 0 && ((props.searchable && !isOpen.value) || !props.searchable)) {
    inputValue.value = activeValues.value[0].title;
  } else {
    inputValue.value = '';
  }
});

const multiSearchInput = ref(null);

watch(isOpen, (newValue) => {

  if (newValue) {
    setPosition();
    const lastActiveValue = activeValues.value[activeValues.value.length - 1] ?? null;
    setPointerIndex(lastActiveValue ? lastActiveValue.value : null);
    scrollToActiveOption('search');

    if(props.searchable){
      if(props.multiple) {
        multiSearchInput.value?.focus();
        multiSearchInput.value.value = '';
      }
      else appSelectInput.value?.focus();
    }
  }
});

watch(() => props.items, () => {
  if (!props.multiple && !activeValues.value.length) model.value = null;
  else {
    model.value = model.value.filter(el => !!activeValues.value.find(item => item.value === el));
  }
});

const changeDropdown = (type = '') => {
  if(type === 'input' && props.searchable && isOpen.value) return
  isOpen.value = !isOpen.value;
}

</script>

<template>
  <div ref="appSelect">
    <label
        v-if="label || slots.label"
        :for="id"
        @click="changeDropdown"
        class="app-select__label"
    >
      <template v-if="label">
        {{ label }}
      </template>
      <slot
          v-else
          name="label"
      />
      <span
          v-if="required"
          class="app-select__label--required"
      >
      *
    </span>
    </label>
    <div :class="appSelectClass">
      <div class="app-select__select">
        <svg
            @click.prevent="changeDropdown"
            v-if="icon"
            :data-src="icon"
            class="app-select__icon"
        />
        <button
            v-if="multiple"
            :id="id"
            ref="appSelectInput"
            :class="appSelectInputClass"
            class="app-select__multiple"
            @click="changeDropdown"
            @keydown.down.prevent="keyDown"
            @keydown.up.prevent="keyUp"
            @keydown.enter.prevent.stop="addOption"
            @keydown.tab.prevent="keyDown"
            @keyup.esc="isOpen = false"
            :style="{'border-radius': getBorderRadius('input')}"
        >
          <span
              v-if="placeholder && !activeValues.length"
              class="app-select__input__placeholder"
          >
            {{ placeholder }}
          </span>
          <template v-else-if="activeValues.length>0">
            <div
                v-for="(item, index) in activeValues"
                :key="item.value"
                class="app-select__multiple-item"
            >
              <span class="app-select__multiple-item__title">
                {{ item.title }}
              </span>
              <button
                  class="app-select__multiple-item__btn"
                  @click.prevent="deleteModelValue(index)"
              >
                <svg
                    class="app-select__multiple-item__btn-icon"
                    data-src="/images/svg/icon-close.svg"
                />
              </button>
            </div>
          </template>
        </button>
        <input
            v-else
            v-model="inputValue"
            ref="appSelectInput"
            type="text"
            :class="appSelectInputClass"
            :id="id"
            :disabled="disabled"
            :readonly="!searchable || !(searchable && isOpen) || !items.length"
            :placeholder="placeholder"
            @keydown.down.prevent="keyDown"
            @keydown.up.prevent="keyUp"
            @keydown.enter.prevent.stop="addOption"
            @keydown.tab.prevent="keyDown"
            @keyup.esc="isOpen = false"
            @click="changeDropdown('input')"
            :style="{'border-radius': getBorderRadius('input')}"
            @input="startSearch"
        >
        <svg
            @click.prevent="changeDropdown"
            data-src="/images/svg/icon-dropdown.svg"
            class="app-select__icon-dropdown"
        />
      </div>
      <div
          ref="appSelectDropdown"
          :class="dropdownClass"
          :style="{'border-radius': getBorderRadius('dropdown')}"
      >
        <input
            v-if="multiple && searchable && dropdownPosition === 'bottom'"
            type="text"
            placeholder="Qidirish"
            @input="startSearch"
            class="app-select__multiple-search"
            ref="multiSearchInput"
        >
        <div
            ref="appSelectDropdownWrapper"
            class="app-select__dropdown-wrapper"
        >
          <div>
            <template v-if="items.length>0">
              <button
                  v-for="(option, index) in items"
                  :key="option[itemValue]"
                  ref="appSelectDropdownOptions"
                  :class="optionClass(option[itemValue], index)"
                  @click="clickOption(option)"
              >
                {{ option[itemTitle] }}
              </button>
            </template>
            <span
                v-else
                ref="appSelectDropdownOptions"
                class="app-select__dropdown--no-information app-select__dropdown-item"
            >
          Ma'lumot yo'q
        </span>
          </div>
        </div>
        <input
            v-if="multiple && searchable && dropdownPosition === 'top'"
            type="text"
            placeholder="Qidirish"
            @input="startSearch"
            class="app-select__multiple-search"
            ref="multiSearchInput"
        >
      </div>
    </div>
  </div>
</template>

<style lang="scss">
.app-select {
  position: relative;
  display: flex;
  flex-direction: column;
  border-radius: 12px;

  &__label {
    margin: 0 4px 6px 7px;
    @include fontSize(18);
    color: #07274b;

    &--required {
      color: $soft-red;
    }
  }

  &__select {
    position: relative;
  }

  &__multiple{
    display: flex;
    flex-wrap: wrap;
    gap: 8px 5px;

    &-search{
      display: flex;
      height: 48px;
      border: 1px solid rgb(225, 229, 234);
      outline: none;
      width: 100%;
      padding: 15px 20px;
    }

    &-item{
      display: flex;
      align-items: center;
      gap: 5px;
      background-color: $dark-blue;
      border-radius: 5px;
      padding: 3px 4px 3px 8px;

      &__title{
        color: #ffffff;
      }

      &__btn{
        padding: 0;
        display: flex;
        align-items: center;
        justify-content: center;

        &-icon {
          width: 17px;
          height: 17px;

          path{
            stroke: #ffffff;
            fill: #ffffff;
          }
        }
      }
    }
  }

  &__input {
    z-index: 2;
    width: 100%;
    padding: 15px 40px 15px 15px;
    border-radius: 12px;
    border: 1px solid rgb(225, 229, 234);
    @include fontSize(20px);
    font-weight: 400;
    line-height: 20px;
    min-height: 45px;
    letter-spacing: -2%;
    transition: 0.6s linear;
    cursor: pointer;

    &:focus {
      outline: none;
    }

    &__placeholder, &::placeholder {
      color: rgb(127, 127, 127);
      font-weight: inherit;
    }
  }

  &--open &__input {
    box-shadow: 0 10px 30px 5px rgba(0, 0, 0, 0.1);
  }

  &--icon &__input {
    padding: 15px 38px 15px 45px;
  }

  &__icon {
    left: 15px;
    max-width: 24px;
    max-height: 24px;

    &-dropdown, & {
      position: absolute;
      top: 50%;
      z-index: 3;
      cursor: pointer;
      transform: translateY(-50%);
    }

    &-dropdown {
      right: 15px;
      cursor: pointer;
      transition: 0.4s ease-in-out;
    }
  }

  &--open &__icon-dropdown {
    transform: rotate(-180deg) translateY(50%);
  }

  &__dropdown {
    position: absolute;
    width: 100%;
    z-index: 5;
    left: 0;
    background-color: #FFFFFF;
    box-shadow: 0 10px 30px -5px rgba(0, 0, 0, 0.1);
    border-radius: 0 0 12px 12px;
    opacity: 1;
    transition: 0.3s ease-in-out;
    transform: translateY(0);
    pointer-events: auto;
    top: calc(100% - 1px);
    overflow: hidden;

    &--top {
      top: unset;
      bottom: calc(100% - 1px);
    }


    &--hidden {
      opacity: 0;
      transform: translateY(-30px);
      pointer-events: none;

      &.app-select__dropdown--top {
        transform: translateY(30px);
      }

      &.app-select__dropdown--bottom {
        transform: translateY(-30px);
      }
    }


    &-wrapper {
      display: flex;
      flex-direction: column;
      overflow: auto;
      width: 100%;
      max-height: 300px;

      &::-webkit-scrollbar-track {
        -webkit-box-shadow: inset 0 0 6px rgba(0, 0, 0, 0.3);
        border-radius: 8px;
        background-color: #f5f5f5;
        margin-left: 15px;
      }

      &::-webkit-scrollbar {
        width: 5px;
        height: 5px;
        border-radius: 8px;
      }

      &::-webkit-scrollbar-thumb {
        border-radius: 8px;
        -webkit-box-shadow: inset 0 0 6px rgba(0, 0, 0, 0.11);
        background-color: $dark-blue;
      }
    }

    &-item {
      cursor: pointer;
      padding: 15px 20px;
      transition: 0.3s ease-in-out;
      font-weight: 400;
      background-color: #FFFFFF;
      min-height: 45px;
      width: 100%;
      display: flex;
      height: 100%;

      &:last-child {
        padding: 10px 20px 20px;
      }

      &--active, &--multi-active {
        color: $dark-blue;
        font-weight: 700;
        background-color: #f3f3f3;
      }

      &:hover {
        &:not(.app-select__dropdown--no-information, .app-select__dropdown-item--multi-active) {
          border-color: $dark-blue !important;
          background-color: $dark-blue !important;
          color: #FFFFFF !important
        }
      }

      &--focus, &:focus {
        &:not(.app-select__dropdown--no-information, .app-select__dropdown-item--multi-active) {
          border-color: #2a519f;
          background-color: #2a519f;
          color: #FFFFFF !important;
        }
      }

      &--multi-active {
        &:hover, &:focus, &.app-select__dropdown-item--focus {
          border-color: #ff6a6a !important;
          background-color: #ff6a6a !important;
          color: #fff !important;
        }
      }
    }

    &--no-information {
      cursor: text;
    }
  }
}
</style>
