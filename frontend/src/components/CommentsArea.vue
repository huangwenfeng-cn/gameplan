<template>
  <div class="flex flex-col">
    <div
      v-if="$resources.comments.data == null"
      class="flex animate-pulse items-start space-x-3 px-2 py-4 text-base"
    >
      <div class="h-8 w-8 rounded-full bg-surface-gray-3"></div>
      <div>
        <div class="flex h-8 flex-col justify-center">
          <div class="h-2 w-40 bg-surface-gray-3"></div>
        </div>
        <div class="flex flex-col gap-2">
          <div v-for="i in 4">
            <div
              class="h-2 bg-surface-gray-3"
              :style="{ width: `${Math.max(Math.random() * 800, 600)}px` }"
            ></div>
          </div>
        </div>
      </div>
    </div>
    <div :style="{ paddingBottom: `${addCommentHeight + 80}px` }">
      <template v-for="item in timelineItems" :key="item.doctype + item.name">
        <div
          v-if="newMessagesFrom && newMessagesFrom == item.name"
          class="relative my-4"
          role="separator"
        >
          <div class="border-b border-blue-600"></div>
          <span
            class="absolute -top-2 left-1/2 -translate-x-1/2 bg-surface-white px-2 text-sm font-medium text-ink-blue-3"
          >
            新评论
          </span>
        </div>
        <Comment
          :class="{
            'border-t': item.name != newMessagesFrom,
          }"
          v-if="item.doctype == 'GP Comment'"
          :ref="($comment) => setItemRef($comment, item)"
          :comment="item"
          :highlight="highlightedItem == item"
          :readOnlyMode="readOnlyMode"
          :comments="$resources.comments"
        />
        <Activity
          class="border-t py-5"
          v-else-if="item.doctype == 'GP Activity'"
          :activity="item"
        />
        <Poll
          class="border-t"
          v-else-if="item.doctype == 'GP Poll'"
          :ref="($poll) => setItemRef($poll, item)"
          :highlight="highlightedItem == item"
          :poll="item"
          :readOnlyMode="readOnlyMode"
        />
      </template>
    </div>

    <div
      v-if="!readOnlyMode && !disableNewComment"
      class="fixed z-[2] bottom-12 left-0 sm:left-auto px-4 sm:px-0 mb-px mt-2 w-full sm:max-w-3xl bg-surface-white py-3 sm:bottom-[-1px] standalone:bottom-16"
      ref="addComment"
    >
      <button
        class="flex w-full items-center rounded-lg bg-surface-gray-2 px-2 py-2 text-left text-base text-ink-gray-5 hover:bg-surface-gray-3"
        @click="showCommentBox = true"
        v-show="!showCommentBox"
      >
        <UserAvatar class="mr-3" :user="$user().name" size="sm" />
        添加评论
      </button>
      <div
        v-show="showCommentBox"
        class="w-full rounded-lg border bg-surface-white p-4 focus-within:border-outline-gray-3"
        @keydown.ctrl.enter.capture.stop="submitComment"
        @keydown.meta.enter.capture.stop="submitComment"
      >
        <div class="mb-4 flex items-center">
          <UserAvatar :user="$user().name" size="md" />
          <span class="ml-2 text-base font-medium text-ink-gray-9">
            {{ $user().full_name }}
          </span>
          <TabButtons
            class="ml-auto"
            :buttons="[{ label: '评论' }, { label: '投票' }]"
            v-model="newCommentType"
          />
        </div>
        <CommentEditor
          ref="newCommentEditor"
          v-show="newCommentType == 'Comment'"
          :value="newComment"
          @change="onNewCommentChange"
          :submitButtonProps="{
            variant: 'solid',
            onClick: submitComment,
            loading: $resources.comments.insert.loading,
            disabled: commentEmpty,
          }"
          :discardButtonProps="{
            onClick: discardComment,
          }"
          :editable="showCommentBox"
          placeholder="添加评论..."
        />
        <PollEditor
          v-show="newCommentType == 'Poll'"
          v-model:poll="newPoll"
          :submitButtonProps="{
            onClick: submitPoll,
            loading: $resources.polls.insert.loading,
          }"
          :discardButtonProps="{
            onClick: discardPoll,
          }"
        />
        <ErrorMessage :message="$resources.polls.insert.error" />
      </div>
    </div>
  </div>
</template>
<script>
import { nextTick } from 'vue'
import { TabButtons } from 'frappe-ui'
import CommentEditor from '@/components/CommentEditor.vue'
import Comment from './Comment.vue'
import Activity from './Activity.vue'
import PollEditor from './PollEditor.vue'
import Poll from './Poll.vue'
import { getScrollContainer } from '@/utils/scrollContainer'

export default {
  name: 'CommentsArea',
  props: ['doctype', 'name', 'newCommentsFrom', 'readOnlyMode', 'disableNewComment'],
  components: {
    CommentEditor,
    Comment,
    Activity,
    TabButtons,
    PollEditor,
    Poll,
  },
  data() {
    let draftComment = localStorage.getItem(this.draftCommentKey())
    return {
      commentMap: {},
      showCommentBox: false,
      newCommentType: 'Comment',
      newComment: draftComment || '',
      newPoll: {
        title: '',
        multiple_answers: false,
        options: [
          { title: '', idx: 1 },
          { title: '', idx: 2 },
        ],
      },
      newMessagesFrom: this.newCommentsFrom,
      highlightedItem: null,
      addCommentHeight: 0,
    }
  },
  watch: {
    showCommentBox(val) {
      if (val) {
        nextTick(() => {
          this.$refs.newCommentEditor?.editor.commands.focus()
          this.scrollToEnd()
        })
      }
    },
  },
  mounted() {
    if (!this.$refs.newCommentEditor?.editor.isEmpty) {
      this.showCommentBox = true
    }
    this.$socket.on('new_activity', (data) => {
      if (data.reference_doctype == this.doctype && data.reference_name == this.name) {
        this.$resources.activities.reload()
      }
    })
    this.setupMutationObserver()
  },
  beforeUnmount() {
    this.$socket.off('new_activity')
    // cleanup mutation observer
    this.mutationObserver?.disconnect()
    delete this.mutationObserver
  },
  resources: {
    comments() {
      return {
        type: 'list',
        doctype: 'GP Comment',
        cache: ['Comments', this.doctype, this.name],
        fields: [
          'name',
          'content',
          'owner',
          'creation',
          'modified',
          'deleted_at',
          { reactions: ['name', 'user', 'emoji'] },
        ],
        transform(data) {
          for (let d of data) {
            d.doctype = 'GP Comment'
          }
          return data
        },
        filters: {
          reference_doctype: this.doctype,
          reference_name: this.name,
        },
        orderBy: 'creation asc',
        pageLength: 99999,
        auto: true,
        onSuccess() {
          if (this.$route.query.comment) {
            if (this.$route.query.comment == 'first_post') {
              this.$router.replace({ query: {} })
              return
            }
            let comment = this.$resources.comments.getRow(this.$route.query.comment)
            this.scrollToItem(comment)
          } else if (!this.$route.query.fromSearch && this.$resources.comments.data.length > 0) {
            this.scrollToEnd()
          }
        },
      }
    },
    activities() {
      return {
        type: 'list',
        doctype: 'GP Activity',
        fields: ['name', 'user', 'action', 'data', 'creation'],
        filters: {
          reference_doctype: this.doctype,
          reference_name: this.name,
        },
        orderBy: 'creation asc',
        pageLength: 99999,
        auto: true,
        transform(activities) {
          for (let activity of activities) {
            activity.doctype = 'GP Activity'
            activity.data = activity.data ? JSON.parse(activity.data) : null
          }
          return activities
        },
      }
    },
    polls() {
      return {
        type: 'list',
        doctype: 'GP Poll',
        fields: [
          'name',
          'title',
          'anonymous',
          'multiple_answers',
          'creation',
          'owner',
          'stopped_at',
          { options: ['name', 'title', 'idx', 'percentage'] },
          { votes: ['user', 'option'] },
          { reactions: ['name', 'user', 'emoji'] },
        ],
        filters: {
          discussion: this.name,
        },
        orderBy: 'creation asc',
        auto: true,
        pageLength: 99999,
        transform(data) {
          for (let d of data) {
            d.doctype = 'GP Poll'
          }
          return data
        },
        onSuccess() {
          if (this.$route.query.poll) {
            let poll = this.$resources.polls.getRow(this.$route.query.poll)
            this.scrollToItem(poll)
          }
        },
      }
    },
  },
  methods: {
    submitComment() {
      if (this.commentEmpty) {
        return
      }
      this.$resources.comments.setData((data) => {
        data.push({
          owner: this.$user().name,
          content: this.newComment,
          reference_doctype: this.doctype,
          reference_name: this.name,
          loading: true,
          reactions: [],
          creation: this.$dayjs().format('YYYY-MM-DD HH:mm:ss'),
        })
        return data
      })
      this.$resources.comments.insert.submit(
        {
          reference_doctype: this.doctype,
          reference_name: this.name,
          content: this.newComment,
        },
        {
          onError(error) {
            this.$resources.comments.setData((data) => {
              let lastComment = data[data.length - 1]
              lastComment.loading = false
              lastComment.error = error
              return data
            })
            this.$toast({
              title: '添加新评论出错',
              text: error.messages.join(', '),
              position: 'bottom-center',
              icon: 'alert-circle',
              iconClasses: 'text-ink-red-4',
            })
          },
        },
      )
      this.resetCommentState()
    },
    async scrollToItem(item) {
      if (!item) return
      await nextTick()
      if (item.$el) {
        this.highlightedItem = item
        this.scrollToElement(item.$el)
      }
      setTimeout(() => {
        this.highlightedItem = null
        this.$router.replace({ query: {} })
      }, 10000)
    },
    scrollToElement($el) {
      let scrollContainer = getScrollContainer()
      let headerHeight = 64
      let top = $el.offsetTop - scrollContainer.scrollTop - headerHeight
      scrollContainer.scrollBy({ top, left: 0, behavior: 'smooth' })
    },
    scrollToEnd() {
      let scrollContainer = getScrollContainer()
      scrollContainer.scrollTop = scrollContainer.scrollHeight
    },
    discardComment() {
      if (!this.editorObject.isEmpty) {
        this.$dialog({
          title: '放弃评论',
          message: '您确定要放弃您的评论吗？',
          actions: [
            {
              label: '放弃评论',
              onClick: (close) => {
                this.resetCommentState()
                close
              },
              variant: 'solid',
            },
            {
              label: '保留评论',
            },
          ],
        })
      } else {
        this.resetCommentState()
      }
    },
    onNewCommentChange(content) {
      this.newComment = content

      // save draft comment to local storage
      setTimeout(() => {
        // set timeout to move it off main thread
        localStorage.setItem(this.draftCommentKey(), content)
      }, 0)
    },
    resetCommentState() {
      localStorage.removeItem(this.draftCommentKey())
      this.$resetData([
        'newComment',
        'showCommentBox',
        'newCommentType',
        'newPoll',
        'highlightedItem',
      ])
    },
    submitPoll() {
      if (this.doctype !== 'GP Discussion') return
      return this.$resources.polls.insert.submit(
        {
          ...this.newPoll,
          discussion: this.name,
        },
        {
          onSuccess() {
            this.resetCommentState()
          },
        },
      )
    },
    discardPoll() {
      this.resetCommentState()
    },
    draftCommentKey() {
      return `draft-comment-${this.doctype}-${this.name}`
    },
    setItemRef($component, item) {
      if ($component?.$el) {
        item.$el = $component.$el
      }
    },
    setupMutationObserver() {
      let $el = this.$refs.addComment
      if (!$el) return
      this.mutationObserver = new MutationObserver(() => {
        this.addCommentHeight = $el.clientHeight
      })
      this.mutationObserver.observe($el, { childList: true, subtree: true })
    },
  },
  computed: {
    timelineItems() {
      let items = []
      if (this.$resources.comments.data?.length) {
        items = items.concat(this.$resources.comments.data)
      }
      if (this.$resources.activities.data?.length) {
        items = items.concat(this.$resources.activities.data)
      }
      if (this.$resources.polls.data?.length) {
        items = items.concat(this.$resources.polls.data)
      }
      return items.sort((a, b) => {
        return new Date(a.creation) - new Date(b.creation)
      })
    },
    commentEmpty() {
      return !this.newComment || this.newComment === '<p></p>'
    },
    editorObject() {
      return this.$refs.newCommentEditor?.editor
    },
  },
}
</script>
