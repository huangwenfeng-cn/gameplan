<template>
  <div class="flex min-h-0 flex-col">
    <div class="flex items-center justify-between">
      <div class="flex items-center gap-2">
        <h2 class="text-xl font-semibold leading-none">邀请人员</h2>
      </div>
    </div>
    <div class="mt-4 space-y-4">
      <FormControl
        type="textarea"
        label="通过邮箱邀请"
        placeholder="user1@example.com, user2@example.com, ..."
        @input="emails = $event.target.value"
        :debounce="100"
        :disabled="$resources.inviteByEmail.loading"
      />
      <template v-if="emails">
        <div>
          <FormControl
            label="角色"
            type="select"
            :options="[
              { label: '管理员', value: 'Gameplan Admin' },
              { label: '成员', value: 'Gameplan Member' },
              { label: '访客', value: 'Gameplan Guest' },
            ]"
            v-model="role"
          />
          <p class="mt-2 text-base text-ink-gray-9">{{ description }}</p>
        </div>
        <div v-if="role === 'Gameplan Guest'">
          <label class="text-sm leading-4 text-ink-gray-7">邀请访客到项目</label>
          <div class="mt-1 flex flex-wrap gap-2">
            <Button
              v-for="project in projects"
              :key="project.value"
              @click="projects = projects.filter((p) => p !== project)"
            >
              {{ project.label }}
              <template #suffix><LucideX class="w-4" /></template>
            </Button>
          </div>
          <Autocomplete
            class="mt-2"
            :options="projectOptions"
            placeholder="选择项目"
            v-model="selectedProject"
          />
        </div>
        <ErrorMessage :message="$resources.inviteByEmail.error" />
        <Button
          variant="solid"
          @click="$resources.inviteByEmail.submit({ emails, role })"
          :loading="$resources.inviteByEmail.loading"
        >
          发送邀请
        </Button>
      </template>
    </div>
    <template v-if="$resources.pendingInvitations.data?.length && !emails">
      <div class="mt-4 flex items-center justify-between border-b py-2 text-base text-ink-gray-5">
        <div class="w-4/5">待处理邀请</div>
      </div>
      <ul class="divide-y overflow-auto">
        <li
          class="flex items-center justify-between py-2"
          v-for="user in $resources.pendingInvitations.data"
          :key="user.name"
        >
          <div class="w-4/5 text-base">
            <span class="text-ink-gray-9">
              {{ user.email }}
            </span>
            <span class="text-ink-gray-5"> ({{ user.role.replace('Gameplan ', '') }}) </span>
          </div>
          <div>
            <Tooltip text="删除邀请">
              <Button
                @click="$resources.pendingInvitations.delete.submit(user.name)"
                :loading="
                  $resources.pendingInvitations.delete.loading &&
                  $resources.pendingInvitations.delete.params.name === user.name
                "
                label="删除邀请"
              >
                <template #icon><LucideX class="w-4" /></template>
              </Button>
            </Tooltip>
          </div>
        </li>
      </ul>
    </template>
  </div>
</template>
<script>
import { Autocomplete, Dropdown, Tooltip } from 'frappe-ui'
import { projects, getTeamProjects } from '@/data/projects'
import { activeTeams } from '@/data/teams'

export default {
  name: 'Invite',
  components: { Dropdown, Tooltip, Autocomplete },
  data() {
    return {
      role: 'Gameplan Member',
      emails: '',
      invitedUsers: [],
      projects: [],
      selectedProject: null,
    }
  },
  watch: {
    selectedProject(value) {
      if (!value) return
      if (!this.projects.includes(value)) {
        this.projects.push(value)
      }
      this.selectedProject = null
    },
  },
  resources: {
    inviteByEmail: {
      url: 'gameplan.api.invite_by_email',
      makeParams() {
        return {
          emails: this.emails,
          role: this.role,
          projects: this.projects.map((project) => project.value),
        }
      },
      onSuccess() {
        this.$resetData()
        this.$resources.pendingInvitations.reload()
      },
    },
    pendingInvitations() {
      return {
        type: 'list',
        doctype: 'GP Invitation',
        filters: { status: 'Pending' },
        fields: ['name', 'email', 'role'],
        auto: !this.emails,
      }
    },
  },
  computed: {
    description() {
      return {
        'Gameplan Admin':
          '可以创建新团队和项目，邀请管理员和成员，浏览和创建讨论。',
        'Gameplan Member': '可以创建项目，邀请成员，浏览和创建讨论。',
        'Gameplan Guest': '可以浏览和参与受邀的团队或项目。',
      }[this.role]
    },
    projectOptions() {
      return activeTeams.value.map((team) => ({
        group: team.title,
        items: getTeamProjects(team.name).map((project) => ({
          label: project.title,
          value: project.name,
        })),
      }))
    },
  },
}
</script>
