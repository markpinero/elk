<script setup lang="ts">
// @ts-expect-error missing types
import { DynamicScroller, DynamicScrollerItem } from 'vue-virtual-scroller'
import type { ComponentPublicInstance } from 'vue'

definePageMeta({
  name: 'status',
  key: route => route.path,
  // GoToSocial
  alias: ['/:server?/@:account/statuses/:status'],
})

const route = useRoute()
const id = $(computedEager(() => route.params.status as string))
const main = ref<ComponentPublicInstance | null>(null)

const { data: status, pending, refresh: refreshStatus } = useAsyncData(
  `status:${id}`,
  () => fetchStatus(id),
  { watch: [isHydrated], immediate: isHydrated.value, default: () => shallowRef() },
)
const { client } = $(useMasto())
const { data: context, pending: pendingContext, refresh: refreshContext } = useAsyncData(
  `context:${id}`,
  async () => client.v1.statuses.fetchContext(id),
  { watch: [isHydrated], immediate: isHydrated.value, lazy: true, default: () => shallowRef() },
)

if (pendingContext)
  watchOnce(pendingContext, scrollTo)

if (pending)
  watchOnce(pending, scrollTo)

async function scrollTo() {
  await nextTick()

  const statusElement = unrefElement(main)
  if (!statusElement)
    return

  statusElement.scrollIntoView(true)
}

const publishWidget = ref()
const focusEditor = () => publishWidget.value?.focusEditor?.()

provide('focus-editor', focusEditor)

watch(publishWidget, () => {
  if (window.history.state.focusReply)
    focusEditor()
})

const replyDraft = $computed(() => status.value ? getReplyDraft(status.value) : null)

onReactivated(() => {
  // Silently update data when reentering the page
  // The user will see the previous content first, and any changes will be updated to the UI when the request is completed
  refreshStatus()
  refreshContext()
})
</script>

<template>
  <MainContent back>
    <template v-if="!pending">
      <template v-if="status">
        <div xl:mt-4 mb="50vh" border="b base">
          <template v-if="!pendingContext">
            <StatusCard
              v-for="comment, i of context?.ancestors" :key="comment.id"
              :status="comment" :actions="comment.visibility !== 'direct'" context="account"
              :has-older="true" :newer="context?.ancestors[i - 1]"
            />
          </template>

          <StatusDetails
            ref="main"
            :status="status"
            :newer="context?.ancestors.at(-1)"
            command
            style="scroll-margin-top: 60px"
          />
          <PublishWidget
            v-if="currentUser"
            ref="publishWidget"
            border="y base"
            :draft-key="replyDraft!.key"
            :initial="replyDraft!.draft"
            @published="refreshContext()"
          />

          <template v-if="!pendingContext">
            <DynamicScroller
              v-slot="{ item, index, active }"
              :items="context?.descendants || []"
              :min-item-size="200"
              key-field="id"
              page-mode
            >
              <DynamicScrollerItem :item="item" :active="active">
                <StatusCard
                  :status="item"
                  context="account"
                  :older="context?.descendants[index + 1]"
                  :newer="index > 0 ? context?.descendants[index - 1] : status"
                  :has-newer="index === 0"
                  :main="status"
                />
              </DynamicScrollerItem>
            </DynamicScroller>
          </template>
        </div>
      </template>

      <StatusNotFound v-else :account="route.params.account as string" :status="id" />
    </template>

    <StatusCardSkeleton v-else border="b base" />
    <TimelineSkeleton v-if="pending || pendingContext" />
  </MainContent>
</template>
