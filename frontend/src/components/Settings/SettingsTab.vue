<template>
  <div class="flex min-h-0 flex-col">
    <h2 class="text-xl font-semibold">设置</h2>
    <div class="mt-6 divide-y overflow-y-auto pb-16">
      <a href="/update-password">
        <Button variant="solid"> 更新密码 </Button>
      </a>
    </div>
  </div>
</template>
<script>
import { teams } from '@/data/teams'
import router from '@/router'

export default {
  name: 'SettingsTabDialog',
}
</script>
