<script setup>

import { EventStreamContentType, fetchEventSource } from '@microsoft/fetch-event-source'

definePageMeta({
  // middleware: ["auth"]
})

const runtimeConfig = useRuntimeConfig()
const currentModel = 'gpt-3.5-turbo' // 'text-davinci-003' // gpt-3.5-turbo-0301 //useCurrentModel()
const openaiApiKey = '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'//useApiKey()
const fetchingResponse = ref(false)

let ctrl
const abortFetch = () => {
  if (ctrl) {
    ctrl.abort()
  }
  fetchingResponse.value = false
}

const systemExpecte = ref('你是一个很有帮助的助手')
const moreChat = ref(true)
const requestMessage = reactive(
  [
    {
      role: "system",
      content: systemExpecte
    }
  ]
)


const fetchReply = async (message, hooks) => {


  requestMessage.push(
    {
      role: "user",
      content: message
    }
  )



  ctrl = new AbortController()
  try {

    await fetchEventSource('https://api.openai.com/v1/chat/completions', {
      signal: ctrl.signal,
      method: 'POST',
      headers: {
        'Accept': 'text/event-stream',
        'Content-Type': 'application/json',
        'Authorization': `Bearer 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`
      },
      body: JSON.stringify({
        "model": "gpt-3.5-turbo",
        "messages": requestMessage,
        "stream": true
      }),
      onopen(response) {
        if (response.ok && response.headers.get('content-type') === EventStreamContentType) {
          return;
        }
        throw new Error(`Failed to send message. HTTP ${response.status} - ${response.statusText}`);
      },
      onclose() {
        if (ctrl.signal.aborted === true) {
          return;
        }
        throw new Error(`Failed to send message. Server closed the connection unexpectedly.`);
      },
      onerror(err) {
        throw err;
      },
      onmessage(message) {
        // console.log("onmessage", message)

        // 处理错误事件
        const event = message.event
        if (event === 'error') {
          throw new Error(data.error);
        }

        // 消息为 'DONE' success
        if (message.data === '[DONE]') {
          // 取消请求
          abortFetch()
          return
        }
        // 拼接消息 SSE
        const data = JSON.parse(message.data)
        let rtu = data.choices[0].delta.content
        if (rtu === undefined || rtu === 'undefined') {
          return
        }

        currentConversation.value.messages[currentConversation.value.messages.length - 1].message += data.choices[0].delta.content
        scrollChatWindow()
      },
    })
  } catch (err) {
    console.log(err)
    abortFetch()
    showSnackbar(err.message)
  }
}

const currentConversation = useConversion()

const grab = ref(null)
const scrollChatWindow = () => {
  if (grab.value === null) {
    return;
  }
  grab.value.scrollIntoView({ behavior: 'smooth' })
}

//  
const send = async (message) => {
  fetchingResponse.value = true
  currentConversation.value.messages.push({ id: new Date().getTime(), is_bot: false, message: message })
  currentConversation.value.messages.push({ id: new Date().getTime() + Math.floor((Math.random() * 10) + 1), is_bot: true, message: '' })
  await fetchReply(message, null)

  if (moreChat.value) {
    requestMessage.push(
      {
        role: "assistant",
        content: currentConversation.value.messages[currentConversation.value.messages.length - 1].message
      }
    )
  }
  scrollChatWindow()
}
const stop = () => {
  abortFetch()
}

const snackbar = ref(false)
const snackbarText = ref('')
const showSnackbar = (text) => {
  snackbarText.value = text
  snackbar.value = true
}

</script>

<template>
  <div v-if="currentConversation.messages.length > 0" ref="chatWindow">
    <v-card rounded="0" elevation="0" v-for="(conversation, index) in currentConversation.messages" :key="index"
      :variant="conversation.is_bot ? 'tonal' : 'text'">
      <v-container>
        <v-card-text class="text-caption text-disabled">{{ $t(`roles.${conversation.is_bot ? 'ai' : 'me'}`)
        }}</v-card-text>
        <v-card-text>
          <MsgContent :content="conversation.message" />
        </v-card-text>
      </v-container>
      <v-divider></v-divider>
    </v-card>
    <div ref="grab" class="w-100" style="height: 200px;"></div>
  </div>
  <Welcome v-else />
  <v-footer app class="d-flex flex-column">
    <div class="px-md-16 w-100 d-flex align-center">
      <v-btn v-show="fetchingResponse" icon="close" title="stop" class="mr-3" @click="stop"></v-btn>
      <MsgEditor :send-message="send" :disabled="fetchingResponse" :loading="fetchingResponse" />
    </div>

    <div class="px-4 py-2 text-disabled text-caption font-weight-light text-center w-100">
      © {{ new Date().getFullYear() }} {{ runtimeConfig.public.appName }}
    </div>
  </v-footer>
  <v-snackbar v-model="snackbar" multi-line location="top">
    {{ snackbarText }}

    <template v-slot:actions>
      <v-btn color="red" variant="text" @click="snackbar = false">
        Close
      </v-btn>
    </template>
  </v-snackbar>
</template>

