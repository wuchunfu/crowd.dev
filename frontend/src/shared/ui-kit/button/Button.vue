<template>
  <button
    :type="props.nativeType"
    class="c-btn"
    :class="[
      `c-btn--${props.type}`,
      `c-btn--${props.size}`,
      { 'is-loading': props.loading },
      { 'c-btn--icon': props.iconOnly },
    ]"
    :disabled="props.disabled"
  >
    <i v-if="props.loading" class="ri-loader-4-line c-btn__loader text-base" />
    <span v-if="props.loading && props.loadingText && !props.iconOnly">{{ props.loadingText }}</span>
    <slot v-else />
  </button>
</template>

<script setup lang="ts">
import { ButtonType } from '@/shared/ui-kit/button/types/ButtonType';
import { ButtonSize } from '@/shared/ui-kit/button/types/ButtonSize';
import { ButtonNativeType } from '@/shared/ui-kit/button/types/ButtonNativeType';

const props = withDefaults(defineProps<{
  type?: ButtonType,
  size?: ButtonSize,
  iconOnly?: boolean;
  nativeType?: ButtonNativeType;
  loading?: boolean;
  loadingText?: string;
  disabled?: boolean;
}>(), {
  type: 'primary',
  size: 'medium',
  iconOnly: false,
  nativeType: 'button',
  loading: false,
  loadingText: undefined,
  disabled: false,
});
</script>

<script lang="ts">
export default {
  name: 'CrButton',
};
</script>
