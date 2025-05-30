<template>
  <div class="flex h-full flex-1" v-if="$resources.task.doc">
    <div class="w-full flex-1">
      <div class="relative p-6">
        <div class="absolute right-0 top-0 p-6" v-show="$resources.task.setValueDebounced.loading">
          <LoadingText v-if="!$resources.task.setValueDebounced.error" text="保存中..." />
          <ErrorMessage :message="$resources.task.setValueDebounced.error" />
        </div>
        <div class="mb-2 flex items-center justify-between space-x-2">
          <input
            type="text"
            placeholder="标题"
            class="-ml-0.5 w-full rounded-sm border-none p-0.5 text-2xl bg-surface-white font-semibold text-ink-gray-9 focus:outline-none focus:ring-2 focus:ring-outline-gray-3"
            @change="
              $resources.task.setValueDebounced.submit({
                title: $event.target.value,
              })
            "
            v-model="$resources.task.doc.title"
            v-focus
          />
          <Dropdown
            :options="[
              {
                label: '删除',
                onClick: () => {
                  $dialog({
                    title: '删除任务',
                    message: '您确定要删除这个任务吗？',
                    actions: [
                      {
                        label: '删除',
                        theme: 'red',
                        variant: 'solid',
                        onClick(close) {
                          return $resources.task.delete.submit(null, {
                            onSuccess() {
                              close()
                              $router.back()
                            },
                          })
                        },
                      },
                    ],
                  })
                },
              },
            ]"
          >
            <Button variant="ghost">
              <template #icon><LucideMoreHorizontal class="h-4 w-4" /></template>
            </Button>
          </Dropdown>
        </div>
        <TextEditor
          ref="description"
          editor-class="prose-sm max-w-none focus-within:ring-2 focus-within:ring-outline-gray-3 rounded-sm p-0.5 -ml-0.5 min-h-[4rem]"
          placeholder="描述"
          :content="$resources.task.doc.description"
          :bubbleMenu="true"
          :floatingMenu="true"
          @blur="
            !$refs.description.editor.isEmpty
              ? $resources.task.setValueDebounced.submit({
                  description: $refs.description.editor.getHTML(),
                })
              : null
          "
        />
        <div class="mt-8 flex flex-wrap items-center gap-2 sm:hidden">
          <Autocomplete
            placeholder="分配给用户"
            :options="assignableUsers"
            v-model="$resources.task.doc.assigned_to"
            @update:modelValue="changeAssignee"
          />
          <DatePicker
            v-model="$resources.task.doc.due_date"
            variant="subtle"
            placeholder="截止日期"
            :disabled="false"
            @update:modelValue="
              $resources.task.setValue.submit({
                due_date: $event,
              })
            "
          />
          <Dropdown :options="statusOptions">
            <Button>
              <template #prefix>
                <TaskStatusIcon :status="$resources.task.doc.status" />
              </template>
              {{ statusTranslation[$resources.task.doc.status] || $resources.task.doc.status || '设置状态' }}
            </Button>
          </Dropdown>
          <Dropdown :options="priorityOptions">
            <Button>
              <template v-if="$resources.task.doc.priority" #prefix>
                <TaskPriorityIcon :priority="$resources.task.doc.priority" />
              </template>
              {{ priorityTranslation[$resources.task.doc.priority] || $resources.task.doc.priority || '设置优先级' }}
            </Button>
          </Dropdown>
          <Autocomplete
            placeholder="选择项目"
            :options="projectOptions"
            v-model="$resources.task.doc.project"
            @update:modelValue="changeProject"
          />
        </div>
        <CommentsList class="mt-8" doctype="GP Task" :name="taskId" />
      </div>
    </div>
    <div class="hidden w-[20rem] shrink-0 border-l sm:block">
      <div class="grid grid-cols-2 items-center gap-y-6 p-6 text-base text-ink-gray-7">
        <div>分配给</div>
        <div>
          <Autocomplete
            placeholder="分配给用户"
            :options="assignableUsers"
            v-model="$resources.task.doc.assigned_to"
            @update:modelValue="changeAssignee"
          />
        </div>
        <div>截止日期</div>
        <div>
          <DatePicker
            v-model="$resources.task.doc.due_date"
            variant="subtle"
            placeholder="截止日期"
            :disabled="false"
            @update:modelValue="
              $resources.task.setValue.submit({
                due_date: $event,
              })
            "
          />
        </div>
        <div>项目</div>
        <div>
          <Autocomplete
            placeholder="选择项目"
            :options="projectOptions"
            v-model="$resources.task.doc.project"
            @update:modelValue="changeProject"
          />
        </div>
        <div>状态</div>
        <div>
          <Dropdown :options="statusOptions">
            <Button>
              <template #prefix>
                <TaskStatusIcon :status="$resources.task.doc.status" />
              </template>
              {{ statusTranslation[$resources.task.doc.status] || $resources.task.doc.status || '设置状态' }}
            </Button>
          </Dropdown>
        </div>
        <div>优先级</div>
        <div>
          <Dropdown :options="priorityOptions">
            <Button>
              <template v-if="$resources.task.doc.priority" #prefix>
                <TaskPriorityIcon :priority="$resources.task.doc.priority" />
              </template>
              {{ priorityTranslation[$resources.task.doc.priority] || $resources.task.doc.priority || '设置优先级' }}
            </Button>
          </Dropdown>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import { h, ref, computed } from 'vue'
import { useRouter } from 'vue-router'
import TextEditor from '@/components/TextEditor.vue'
import ReadmeEditor from '@/components/ReadmeEditor.vue'
import CommentsArea from '@/components/CommentsArea.vue'
import { focus } from '@/directives'
import { Autocomplete, Dropdown, LoadingText, DatePicker } from 'frappe-ui'
import CommentsList from '@/components/CommentsList.vue'
import TaskStatusIcon from '@/components/icons/TaskStatusIcon.vue'
import TaskPriorityIcon from '@/components/icons/TaskPriorityIcon.vue'
import { activeUsers } from '@/data/users'
import { activeTeams } from '@/data/teams'
import { getTeamProjects } from '@/data/projects'

// 任务状态翻译映射表
const statusTranslation = {
  'Backlog': '待办池',
  'Todo': '待处理',
  'In Progress': '进行中',
  'Done': '已完成',
  'Canceled': '已取消'
}

// 任务优先级翻译映射表
const priorityTranslation = {
  'Low': '低',
  'Medium': '中',
  'High': '高'
}

export default {
  name: 'TaskDetail',
  props: ['taskId'],
  directives: { focus },
  resources: {
    task() {
      return {
        type: 'document',
        doctype: 'GP Task',
        name: this.taskId,
        whitelistedMethods: {
          trackVisit: 'track_visit',
        },
        setValue: {
          onError(e) {
            let message = e.messages ? e.messages.join('\n') : e.message
            this.$toast({
              title: '任务更新错误',
              text: message,
              icon: 'alert-circle',
              iconClasses: 'text-ink-red-4',
            })
          },
        },
        onSuccess(doc) {
          if (
            ['ProjectTaskDetail', 'Task'].includes(this.$route.name) &&
            Number(this.$route.params.taskId) === doc.name
          ) {
            this.$resources.task.trackVisit.submit()
          }
        },
      }
    },
  },
  methods: {
    changeAssignee(option) {
      this.$resources.task.setValue.submit({ assigned_to: option?.value || '' })
    },
    changeProject(option) {
      this.$resources.task.setValue.submit(
        {
          project: option?.value || '',
        },
        {
          onSuccess() {
            this.updateRoute()
          },
        },
      )
    },
    updateRoute() {
      let task = this.$resources.task.doc
      if (task) {
        this.$router.replace({
          name: task.project ? 'ProjectTaskDetail' : 'Task',
          params: {
            taskId: task.name,
            teamId: task.team,
            projectId: task.project,
          },
        })
      }
    },
  },
  computed: {
    assignableUsers() {
      return activeUsers.value.map((user) => ({
        label: user.full_name,
        value: user.name,
      }))
    },
    statusOptions() {
      return ['Backlog', 'Todo', 'In Progress', 'Done', 'Canceled'].map((status) => {
        return {
          icon: () => h(TaskStatusIcon, { status }),
          label: statusTranslation[status] || status,
          onClick: () => this.$resources.task.setValue.submit({ status }),
        }
      })
    },
    priorityOptions() {
      return ['Low', 'Medium', 'High'].map((priority) => {
        return {
          icon: () => h(TaskPriorityIcon, { priority }),
          label: priorityTranslation[priority] || priority,
          onClick: () => this.$resources.task.setValue.submit({ priority }),
        }
      })
    },
    projectOptions() {
      return activeTeams.value.map((team) => ({
        group: team.title,
        items: getTeamProjects(team.name).map((project) => ({
          label: project.title,
          value: project.name.toString(),
        })),
      }))
    },
  },
  components: {
    ReadmeEditor,
    TextEditor,
    CommentsArea,
    Autocomplete,
    Dropdown,
    CommentsList,
    TaskStatusIcon,
    LoadingText,
    TaskPriorityIcon,
    DatePicker,
  },
}
</script>
