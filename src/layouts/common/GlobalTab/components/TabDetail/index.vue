<template>
  <div ref="tabRef" class="h-full" :class="[isChromeMode ? 'flex items-end' : 'flex-y-center']">
    <component
      :is="activeComponent"
      v-for="(item, index) in tab.tabs"
      :key="item.fullPath"
      :is-active="tab.activeTab === item.fullPath"
      :primary-color="theme.themeColor"
      :closable="item.name !== tab.homeTab.name"
      :dark-mode="theme.darkMode"
      :class="{ '!mr-0': isChromeMode && index === tab.tabs.length - 1, 'mr-10px': !isChromeMode }"
      @click="tab.handleClickTab(item.fullPath)"
      @close="tab.removeTab(item.fullPath)"
      @contextmenu="handleContextMenu($event, item.fullPath)"
    >
      <Icon v-if="item.meta.icon" :icon="item.meta.icon" class="inline-block align-text-bottom mr-4px text-16px" />
      {{ item.meta.title }}
    </component>
  </div>
  <context-menu
    v-model:visible="dropdown.visible"
    :current-path="dropdown.currentPath"
    :x="dropdown.x"
    :y="dropdown.y"
  />
</template>

<script setup lang="ts">
import { ref, reactive, computed, nextTick, watch } from 'vue';
import { useEventListener } from '@vueuse/core';
import { ChromeTab, ButtonTab } from '@soybeanjs/vue-admin-tab';
import { Icon } from '@iconify/vue';
import { useThemeStore, useTabStore } from '@/store';
import { setTabRoutes } from '@/utils';
import { ContextMenu } from './components';

interface Emits {
  (e: 'scroll', clientX: number): void;
}

const emit = defineEmits<Emits>();

const theme = useThemeStore();
const tab = useTabStore();

const isChromeMode = computed(() => theme.tab.mode === 'chrome');
const activeComponent = computed(() => (isChromeMode.value ? ChromeTab : ButtonTab));

// 获取当前激活的tab的clientX
const tabRef = ref<HTMLElement>();
async function getActiveTabClientX() {
  await nextTick();
  if (tabRef.value && tabRef.value.children.length && tabRef.value.children[tab.activeTabIndex]) {
    const activeTabElement = tabRef.value.children[tab.activeTabIndex];
    const { x, width } = activeTabElement.getBoundingClientRect();
    const clientX = x + width / 2;
    setTimeout(() => {
      emit('scroll', clientX);
    }, 50);
  }
}

const dropdown = reactive({
  visible: false,
  x: 0,
  y: 0,
  currentPath: ''
});
function showDropdown() {
  dropdown.visible = true;
}
function hideDropdown() {
  dropdown.visible = false;
}
function setDropdown(x: number, y: number, currentPath: string) {
  Object.assign(dropdown, { x, y, currentPath });
}

/** 点击右键菜单 */
async function handleContextMenu(e: MouseEvent, fullPath: string) {
  e.preventDefault();
  const { clientX, clientY } = e;
  hideDropdown();
  setDropdown(clientX, clientY, fullPath);
  await nextTick();
  showDropdown();
}

watch(
  () => tab.activeTabIndex,
  () => {
    getActiveTabClientX();
  },
  {
    immediate: true
  }
);

/** 页面离开时缓存多页签数据 */
useEventListener(window, 'beforeunload', () => {
  setTabRoutes(tab.tabs);
});
</script>

<style scoped></style>
