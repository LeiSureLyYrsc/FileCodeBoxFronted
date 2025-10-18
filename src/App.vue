<script setup lang="ts">
import { ref, provide, onMounted, onUnmounted } from 'vue'
import { RouterView } from 'vue-router'
import ThemeToggle from './components/common/ThemeToggle.vue'
import LanguageSwitcher from './components/common/LanguageSwitcher.vue'
import { useRouter, useRoute } from 'vue-router'
import AlertComponent from '@/components/common/AlertComponent.vue'
import { useAlertStore } from '@/stores/alertStore'
import { ConfigService } from '@/services'
import type { ApiResponse, ConfigState } from '@/types'
import { useTheme } from '@/composables/useTheme'

const isLoading = ref(false)
const router = useRouter()
const route = useRoute()
const alertStore = useAlertStore()

// 使用主题 composable
const { isDarkMode, toggleTheme, initTheme } = useTheme()

// 背景模糊状态
const isBackgroundBlurred = ref(false)

// 监听焦点事件 - 检测交互元素
const handleFocusIn = (e: FocusEvent) => {
  const target = e.target as HTMLElement
  // 检查是否是目标交互元素
  if (
    target.tagName === 'INPUT' ||
    target.tagName === 'TEXTAREA' ||
    target.tagName === 'SELECT' ||
    target.closest('[data-file-upload]') ||
    target.closest('.file-upload-area')
  ) {
    isBackgroundBlurred.value = true
  }
}

const handleFocusOut = () => {
  isBackgroundBlurred.value = false
}

// 监听点击事件 - 检测文件上传区域点击
const handleClick = (e: MouseEvent) => {
  const target = e.target as HTMLElement
  if (
    target.closest('[data-file-upload]') ||
    target.closest('.file-upload-area')
  ) {
    isBackgroundBlurred.value = true
    setTimeout(() => {
      isBackgroundBlurred.value = false
    }, 3000)
  }
}

// 清理函数
let cleanupThemeListener: (() => void) | null = null

onMounted(() => {
  // 初始化主题并设置监听器
  cleanupThemeListener = initTheme()
  ConfigService.getUserConfig().then((res: ApiResponse<ConfigState>) => {
    if (res.code === 200 && res.detail) {
      localStorage.setItem('config', JSON.stringify(res.detail))
      if (
        res.detail.notify_title &&
        res.detail.notify_content &&
        localStorage.getItem('notify') !== res.detail.notify_title + res.detail.notify_content
      ) {
        localStorage.setItem('notify', res.detail.notify_title + res.detail.notify_content)
        alertStore.showAlert(res.detail.notify_title + ': ' + res.detail.notify_content, 'success')
      }
    }
  })

  // 添加交互事件监听
  document.addEventListener('focusin', handleFocusIn)
  document.addEventListener('focusout', handleFocusOut)
  document.addEventListener('click', handleClick)
})

onUnmounted(() => {
  // 清理主题监听器
  if (cleanupThemeListener) {
    cleanupThemeListener()
  }

  // 移除交互事件监听
  document.removeEventListener('focusin', handleFocusIn)
  document.removeEventListener('focusout', handleFocusOut)
  document.removeEventListener('click', handleClick)
})

router.beforeEach((to, from, next) => {
  isLoading.value = true
  next()
})

router.afterEach(() => {
  setTimeout(() => {
    isLoading.value = false
  }, 200) // 添加一个小延迟，以确保组件已加载
})

provide('isDarkMode', isDarkMode)
provide('toggleTheme', toggleTheme)
provide('isLoading', isLoading)
</script>

<template>
  <div :class="['app-container', isDarkMode ? 'dark' : 'light']">
    <!-- 背景图片 -->
    <div 
      class="background-image"
      :class="{ 'blurred': isBackgroundBlurred, 'dark-mode': isDarkMode }"
    ></div>

    <div class="fixed top-4 right-4 z-50 flex items-center space-x-3">
      <LanguageSwitcher />
      <ThemeToggle v-model="isDarkMode" />
    </div>
    <div class="router-view-wrapper">
      <RouterView v-slot="{ Component }">
        <transition name="fade" mode="out-in">
          <component :is="Component" :key="route.fullPath" />
        </transition>
      </RouterView>
    </div>

    <AlertComponent />
  </div>
</template>

<style>
.app-container {
  position: relative;
  min-height: 100vh;
  width: 100%;
  overflow-x: hidden;
  overflow-y: hidden;
}

/* 背景图片 */
.background-image {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: url('/assets/background.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  z-index: 0;
  transition: transform 0.5s ease, filter 0.5s ease;
  will-change: transform, filter;
}

/* 暗色模式下背景变暗 */
.background-image.dark-mode {
  filter: brightness(0.6);
}

.background-image.blurred {
  transform: scale(1.1);
  filter: blur(10px);
}

/* 暗色模式 + 模糊状态 */
.background-image.dark-mode.blurred {
  filter: brightness(0.6) blur(10px);
}

.router-view-wrapper {
  position: relative;
  width: 100%;
  min-height: 100vh;
  overflow: hidden;
  z-index: 10;
}

/* 过渡动画 */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
