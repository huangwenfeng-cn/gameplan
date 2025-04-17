<template>
  <Dialog v-model="show" :options="{ size: '5xl' }">
    <template #body>
      <div class="flex" :style="{ height: 'calc(100vh - 8rem)' }">
        <div class="flex w-52 shrink-0 flex-col bg-surface-menu-bar p-2">
          <h1 class="px-2 pt-2 text-lg font-semibold">设置</h1>
          <div class="mt-3">
            <button
              class="flex h-7 w-full items-center gap-2 rounded px-2 py-1"
              :class="[
                activeTab?.label == tab.label
                  ? 'bg-surface-white shadow-sm'
                  : 'hover:bg-surface-gray-2',
              ]"
              v-for="tab in tabs"
              :key="tab.label"
              @click="activeTab = tab"
            >
              <component :is="tab.icon" class="h-4 w-4 text-ink-gray-7" />
              <span class="text-base text-ink-gray-8">
                {{ tab.label }}
              </span>
            </button>
          </div>
        </div>
        <div class="flex flex-1 flex-col px-16 pt-10">
          <Tabs v-model="activeTab" className="min-h-0 overflow-y-auto h-[calc(100vh-7rem)]">
            <Tab id="members" label="成员">
              <TeamMembers />
            </Tab>
            <Tab id="invite" label="邀请">
              <InvitePeople />
            </Tab>
            <Tab id="archived-teams" label="已归档团队">
              <ArchivedTeams />
            </Tab>
            <Tab id="settings" label="设置">
              <TeamSettings />
            </Tab>
          </Tabs>
        </div>
      </div>
    </template>
  </Dialog>
</template>
<script>
import { markRaw, ref } from 'vue'
import { Dialog, Tabs, Tab } from 'frappe-ui'
import Members from './Members.vue'
import ArchivedTeams from './ArchivedTeams.vue'
import InvitePeople from './InvitePeople.vue'
import SettingsTabDialog from './SettingsTab.vue'
import LucideUsers from '~icons/lucide/users'
import LucideUsersPlus from '~icons/lucide/user-plus'
import LucideFolderMinus from '~icons/lucide/folder-minus'
import LucideSettings from '~icons/lucide/settings'

let tabs = [
  {
    label: '成员',
    icon: LucideUsers,
    component: markRaw(Members),
  },
  {
    label: '邀请',
    icon: LucideUsersPlus,
    component: markRaw(InvitePeople),
  },
  {
    label: '已归档团队',
    icon: LucideFolderMinus,
    component: markRaw(ArchivedTeams),
  },
  {
    label: '设置',
    icon: LucideSettings,
    component: markRaw(SettingsTabDialog),
  },
]

let show = ref(false)
let activeTab = ref(null)

export function showSettingsDialog(defaultTab = null) {
  show.value = true
  if (defaultTab) {
    activeTab.value = tabs.find((tab) => tab.label == defaultTab)
  } else {
    activeTab.value = tabs[0]
  }
}

export default {
  name: 'SettingsDialog',
  components: {
    Dialog,
    Tabs,
    Tab,
  },
  setup() {
    return { tabs, show, activeTab }
  },
}
</script>
