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

// 背景模糊状态 - 支持两个层次: 'none', 'light', 'heavy'
const blurLevel = ref<'none' | 'light' | 'heavy'>('none')

// 监听焦点事件 - 检测交互元素（轻度模糊）
const handleFocusIn = (e: FocusEvent) => {
  // 如果当前处于重度模糊状态（文件详情弹窗打开），不响应
  if (blurLevel.value === 'heavy') {
    return
  }
  
  const target = e.target as HTMLElement
  // 检查是否是目标交互元素
  if (
    target.tagName === 'INPUT' ||
    target.tagName === 'TEXTAREA' ||
    target.tagName === 'SELECT' ||
    target.closest('[data-file-upload]') ||
    target.closest('.file-upload-area')
  ) {
    blurLevel.value = 'light'
  }
}

const handleFocusOut = () => {
  // 如果当前处于重度模糊状态（文件详情弹窗打开），不响应
  if (blurLevel.value === 'heavy') {
    return
  }
  blurLevel.value = 'none'
}

// 监听点击事件 - 检测文件上传区域点击（轻度模糊）
const handleClick = (e: MouseEvent) => {
  // 如果当前处于重度模糊状态（文件详情弹窗打开），不响应
  if (blurLevel.value === 'heavy') {
    return
  }
  
  const target = e.target as HTMLElement
  if (
    target.closest('[data-file-upload]') ||
    target.closest('.file-upload-area')
  ) {
    blurLevel.value = 'light'
    setTimeout(() => {
      blurLevel.value = 'none'
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
provide('setBlurLevel', (level: 'none' | 'light' | 'heavy') => {
  blurLevel.value = level
})
</script>

<template>
  <div :class="['app-container', isDarkMode ? 'dark' : 'light']">
    <!-- 背景图片 -->
    <div 
      class="background-image"
      :class="{ 
        'blur-light': blurLevel === 'light', 
        'blur-heavy': blurLevel === 'heavy',
        'dark-mode': isDarkMode 
      }"
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

    <!-- 页脚 -->
    <footer class="footer-container">
      <div class="footer-content">
        <p class="footer-text">
          © 2025
          <a
            href="https://0d000721.xin/"
            target="_blank"
            rel="noopener noreferrer"
            class="footer-link"
          >
            LeiSureLyYrsc
          </a>
          | Based on
          <a
            href="https://github.com/vastsa/FileCodeBox"
            target="_blank"
            rel="noopener noreferrer"
            class="footer-link"
          >
            FileCodeBox
          </a>
          | 
          <a
            href="https://www.gnu.org/licenses/lgpl-3.0.html"
            target="_blank"
            rel="noopener noreferrer"
            class="footer-link"
          >
            LGPL-3.0
          </a>
          | Made with ❤️
        </p>
      </div>
    </footer>

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

/* 轻度模糊 - 用于输入框等交互元素 */
.background-image.blur-light {
  transform: scale(1.08);
  filter: blur(8px);
}

/* 重度模糊 - 用于模态框等强调场景 */
.background-image.blur-heavy {
  transform: scale(1.15);
  filter: blur(12px);
}

/* 暗色模式 + 轻度模糊 */
.background-image.dark-mode.blur-light {
  filter: brightness(0.6) blur(8px);
}

/* 暗色模式 + 重度模糊 */
.background-image.dark-mode.blur-heavy {
  filter: brightness(0.6) blur(12px);
}

.router-view-wrapper {
  position: relative;
  width: 100%;
  min-height: 100vh;
  overflow: hidden;
  z-index: 10;
}

/* 页脚样式 */
.footer-container {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 20;
  padding: 1rem;
  backdrop-filter: blur(10px);
  background: rgba(255, 255, 255, 0.1);
  border-top: 1px solid rgba(255, 255, 255, 0.2);
  transition: all 0.3s ease;
}

.dark .footer-container {
  background: rgba(0, 0, 0, 0.3);
  border-top: 1px solid rgba(255, 255, 255, 0.1);
}

.footer-content {
  max-width: 1200px;
  margin: 0 auto;
  text-align: center;
}

.footer-text {
  margin: 0;
  font-size: 0.875rem;
  font-family: 'MiSans Bold', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  font-weight: 700;
  color: rgba(255, 255, 255, 0.9);
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
}

.dark .footer-text {
  color: rgba(255, 255, 255, 0.8);
}

.footer-link {
  color: rgba(255, 255, 255, 0.95);
  text-decoration: none;
  font-weight: 600;
  transition: all 0.3s ease;
  padding: 0 0.25rem;
  border-bottom: 2px solid transparent;
}

.footer-link:hover {
  color: #818cf8;
  border-bottom-color: #818cf8;
}

.dark .footer-link {
  color: rgba(255, 255, 255, 0.9);
}

.dark .footer-link:hover {
  color: #a5b4fc;
  border-bottom-color: #a5b4fc;
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
