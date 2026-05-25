<!-- eslint-disable vue/no-v-html -->
<template>
  <div
    v-if="reply?.message"
    class="flex items-center justify-around gap-2 px-3 pt-2 sm:px-10"
  >
    <div
      class="mb-1 ml-13 flex-1 cursor-pointer rounded border-0 border-l-4 border-green-500 bg-surface-gray-2 p-2 text-base text-ink-gray-5"
      :class="reply.type == 'Incoming' ? 'border-green-500' : 'border-blue-400'"
    >
      <div
        class="mb-1 text-sm font-bold"
        :class="
          reply.type == 'Incoming' ? 'text-ink-green-2' : 'text-ink-blue-link'
        "
      >
        {{ reply.from_name || __('You') }}
      </div>
      <div
        class="max-h-12 overflow-hidden"
        v-html="sanitizeHTML(reply.message)"
      />
    </div>

    <Button variant="ghost" icon="x" @click="reply = {}" />
  </div>
<div class="flex items-end gap-2 px-4 py-2.5 w-full" v-bind="$attrs">

  <div class="flex flex-1 items-center rounded-[8px] border border-gray-300 bg-white px-3 py-2.5 focus-within:border-green-400 focus-within:ring-1 focus-within:ring-green-400">

    <!-- Emoji -->
    <IconPicker
      v-slot="{ togglePopover }"
      v-model="emoji"
      @update:modelValue="
        () => {
          content += emoji
          $refs.textareaRef.focus()
          capture('whatsapp_emoji_added')
        }
      "
    >
      <SmileIcon
        class="mr-2 flex size-5 shrink-0 cursor-pointer text-ink-gray-4"
        @click="togglePopover"
      />
    </IconPicker>

    <!-- Textarea -->
    <textarea
      ref="textareaRef"
      v-model="content"
      rows="1"
      :placeholder="__('Type a message')"
      class="max-h-24 flex-1 resize-none overflow-y-auto border-0 bg-transparent text-sm text-gray-800 placeholder-gray-400 focus:outline-none focus:ring-0"
      style="height: 24px; overflow-y: hidden; line-height: 20px; padding: 0; margin: 0;"
      @input="autoResize"
    />

    <!-- Attachment -->
    <FileUploader @success="(file) => uploadFile(file)">
      <template #default="{ openFileSelector }">
        <Dropdown :options="uploadOptions(openFileSelector)">
          <FeatherIcon
            name="plus"
            class="ml-2 size-5 shrink-0 cursor-pointer text-ink-gray-5"
          />
        </Dropdown>
      </template>
    </FileUploader>

  </div>

  <!-- Send Button -->
  <button
    :disabled="!content?.trim()"
    class="flex h-12 w-12 shrink-0 self-center items-center justify-center rounded-full bg-green-500 text-white transition hover:bg-green-600 disabled:cursor-not-allowed disabled:opacity-40"
    type="button"
    @click="sendWhatsAppMessage"
  >
    <svg
      xmlns="http://www.w3.org/2000/svg"
      class="h-6 w-6 translate-x-px"
      viewBox="0 0 24 24"
      fill="currentColor"
    >
      <path d="M3.4 20.4L20.85 12 3.4 3.6v6.53L15.87 12 3.4 13.87v6.53z" />
    </svg>
  </button>

</div>
</template>

<script setup>
import IconPicker from '@/components/IconPicker.vue'
import SmileIcon from '@/components/Icons/SmileIcon.vue'
import { sanitizeHTML } from '@/utils'
import { useTelemetry } from 'frappe-ui/frappe'
import {
  createResource,
  Textarea,
  FileUploader,
  Dropdown,
  toast,
} from 'frappe-ui'
import { ref, nextTick, watch } from 'vue'

const props = defineProps({
  doctype: { type: String, default: '' },
})

const doc = defineModel({ type: Object, default: () => ({}) })
const whatsapp = defineModel('whatsapp', { type: Object, default: () => ({}) })
const reply = defineModel('reply', { type: Object, default: () => ({}) })

const { capture } = useTelemetry()

const rows = ref(1)
const textareaRef = ref(null)
const emoji = ref('')

const content = ref('')
const placeholder = ref(__('Type your message here...'))
const fileType = ref('')

function show() {
  nextTick(() => textareaRef.value?.focus())
}

function uploadFile(file) {
  whatsapp.value.attach = file.file_url
  whatsapp.value.content_type = fileType.value
  sendWhatsAppMessage()
  capture('whatsapp_upload_file')
}

const autoResize = (e) => {

  const el = e.target

  el.style.height = '24px'
  const newH = Math.min(el.scrollHeight, 96)
  el.style.height = `${newH}px`
  el.style.overflowY = el.scrollHeight > 96 ? 'auto' : 'hidden'
}

function sendTextMessage(event) {
  if (event.shiftKey) return
  sendWhatsAppMessage()
  textareaRef.value.el?.blur()
  content.value = ''
  capture('whatsapp_send_message')
}

async function sendWhatsAppMessage() {

  if (!content.value?.trim()) return

  let args = {
    reference_doctype: props.doctype,
    reference_name: doc.value.name,
    message: content.value,
    to: doc.value.mobile_no,
    attach: whatsapp.value.attach || '',
    reply_to: reply.value?.name || '',
    content_type: whatsapp.value.content_type,
  }

  try {

    await createResource({
      url: 'crm.api.whatsapp.create_whatsapp_message',
      params: args,
    }).submit()

    content.value = ''
    fileType.value = ''
    whatsapp.value.attach = ''
    whatsapp.value.content_type = 'text'
    reply.value = {}

    whatsapp.value.reload()

    nextTick(() => {
      if (textareaRef.value) {
        textareaRef.value.style.height = '24px'
        textareaRef.value.style.overflowY = 'hidden'
      }
    })

  } catch (error) {

    toast.error(
      error.messages?.[0] ||
      __('Failed to send WhatsApp message')
    )
  }
}

function uploadOptions(openFileSelector) {
  return [
    {
      label: __('Upload Document'),
      icon: 'file',
      onClick: () => {
        fileType.value = 'document'
        openFileSelector()
      },
    },
    {
      label: __('Upload Image'),
      icon: 'image',
      onClick: () => {
        fileType.value = 'image'
        openFileSelector('image/*')
      },
    },
    {
      label: __('Upload Video'),
      icon: 'video',
      onClick: () => {
        fileType.value = 'video'
        openFileSelector('video/*')
      },
    },
  ]
}

watch(reply, (value) => {
  if (value?.message) {
    show()
  }
})

defineExpose({ show })
</script>