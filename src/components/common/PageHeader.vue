<template>
  <div class="text-center">
    <div class="flex justify-center mb-8">
      <div
        class="rounded-full from-indigo-500 via-purple-500 to-pink-500 p-1 animate-spin-slow"
      >
          <img 
            src="/assets/logo_small.jpg" 
            :alt="title"
            class="w-15 h-20 object-contain rounded-full" 
          />
      </div>
    </div>
    <h2
      @click="$emit('title-click')"
      class="text-3xl cursor-pointer font-extrabold text-center mb-6"
      :class="[
        isDarkMode
          ? 'text-transparent bg-clip-text bg-gradient-to-r from-indigo-300 via-purple-300 to-pink-300'
          : 'text-indigo-600'
      ]"
    >
      {{ title }}
    </h2>
  </div>
</template>

<script setup lang="ts">
import { inject } from 'vue'

interface Props {
  title: string
}

interface Emits {
  'title-click': []
}

defineProps<Props>()
defineEmits<Emits>()

const isDarkMode = inject('isDarkMode')
</script>

<style scoped>
@keyframes spin-pause {
  /* 0-0.2秒: 快速旋转5圈 (1800度 = 5 * 360度) */
  0% {
    transform: rotate(0deg);
  }
  35% {
    transform: rotate(1800deg);
  }
  /* 0.2-2.2秒: 停止不动 */
  35%, 100% {
    transform: rotate(1800deg);
  }
}

.animate-spin-slow {
  animation: spin-pause 2s linear infinite;
}
</style>