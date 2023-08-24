<template>
  <div>
    <h2>请输入当前的用户名：</h2>
    <el-input
      v-model="username"
      placeholder="Please input"
      @keyup.enter="finishUserName"
    />
    <h2>当前在线用户：</h2>
    <el-select
      v-model="selectUser"
      class="m-2"
      placeholder="Select"
      size="large"
    >
      <el-option
        v-for="item in onlineUser"
        :key="item.value"
        :label="item.label"
        :value="item.value"
      />
    </el-select>
    <h2>在线用户：{{ onlineCount }}</h2>
    <h2>请输入当前的消息：</h2>
    <el-input v-model="newMessage" placeholder="Please input" />
    <el-button type="primary" @click="sendMessage" style="margin: 20px">
      发送消息
    </el-button>
    <el-timeline>
      <el-timeline-item
        v-for="(activity, index) in activities.data"
        :key="index"
        :timestamp="activity.content"
      >
        {{ activity.sender }}
      </el-timeline-item>
    </el-timeline>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, reactive } from 'vue'
import { ElMessage } from 'element-plus'

// eslint-disable-next-line @typescript-eslint/ban-ts-comment
//@ts-expect-error
import * as Stomp from 'stompjs'
interface selectOption {
  label: string
  value: string
}

interface messageType {
  sender: string
  content: string
}

onMounted(() => {})
const finishUserName = () => {
  joinChat()
}
const username = ref('')
const newMessage = ref('')
const selectUser = ref('')
const onlineUser = ref<selectOption[]>([])
const onlineCount = ref<number>(0)

let stompClient: Stomp.Client | null = null

const joinChat = () => {
  initializeWebSocket()
}
interface Activities {
  data: messageType[]
}

const activities: Activities = reactive({
  data: [],
})

const initializeWebSocket = () => {
  const socket = new WebSocket('ws://localhost:9999/chat')
  stompClient = Stomp.over(socket)
  stompClient.connect({}, () => {
    //私聊
    stompClient?.subscribe(
      `/queue/destination/${username.value}`,
      (message: any) => {
        let receiveMessage = JSON.parse(message.body)
        activities.data.push({
          content: receiveMessage.content,
          sender: receiveMessage.sender,
        })
      },
    )
    //在线用户
    stompClient?.subscribe('/online/count', (message: any) => {
      let onlineUserLit = JSON.parse(message.body)
      let keys = Object.keys(onlineUserLit)
      onlineUser.value = []
      onlineCount.value = 0
      ElMessage({
        message: '有新伙伴加入了聊天室！',
        type: 'success',
      })
      keys.forEach((user) => {
        if (onlineUserLit[user] != username.value) {
          onlineCount.value++
          onlineUser.value.push({
            label: onlineUserLit[user],
            value: onlineUserLit[user],
          })
        }
      })
    })
    //公共频道
    stompClient?.subscribe('/topic/public', (message: any) => {
      console.log(message)
    })
    const chatMessage = {
      content: `${username.value} has joined the chat.`,
      sender: username.value,
    }
    stompClient?.send('/app/chat.addUser', {}, JSON.stringify(chatMessage))
  })
}

const sendMessage = () => {
  if (newMessage.value.trim() !== '') {
    const chatMessage = {
      content: newMessage.value.trim(),
      sender: username.value,
      receiver: selectUser.value,
    }
    console.log(chatMessage)
    stompClient?.send('/app/chat.sendMessage', {}, JSON.stringify(chatMessage))
    newMessage.value = ''
  }
}
</script>
