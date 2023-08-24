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
import { v4 as uuidv4 } from 'uuid'

const userId = uuidv4()

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
  if (!isOnlineUser) {
    initializeWebSocket()
  }
}
interface Activities {
  data: messageType[]
}

const activities: Activities = reactive({
  data: [],
})

let isOnlineUser = false

const onConnected = () => {
  //私聊
  stompClient?.subscribe(`/online/receiveMessage/${userId}`, (message: any) => {
    let receiveMessage = JSON.parse(message.body)
    activities.data.push({
      content: receiveMessage.content,
      sender: receiveMessage.userName,
    })
  })
  //在线用户
  stompClient?.subscribe('/online/onlineOrOfflineUser', (message: any) => {
    let onlineUserMessage = JSON.parse(message.body)
    if (onlineUserMessage.userId != userId) {
      //有人加入或者离开了聊天室
      if (onlineUserMessage.userType) {
        onlineCount.value++
        ElMessage({
          message: onlineUserMessage.userName + '加入了聊天室',
          type: 'success',
        })
        //加入下拉选
        onlineUser.value.push({
          label: onlineUserMessage.userName,
          value: onlineUserMessage.sessionId,
        })
      } else {
        onlineCount.value--
        ElMessage({
          message: onlineUserMessage.userName + '离开了聊天室',
          type: 'success',
        })
        //移出下拉选
        onlineUser.value = onlineUser.value.filter(
          (item) => item.value != onlineUserMessage.sessionId,
        )
      }
    }
    //   })
  })
  //公共频道
  stompClient?.subscribe('/topic/public', (message: any) => {
    console.log(message)
  })
  //加入聊天室
  const chatMessage = {
    userName: username.value,
    userId: userId,
    userType: 1,
  }
  stompClient?.send(
    '/app/chat.onlineOrOfflineUser',
    {},
    JSON.stringify(chatMessage),
  )
  isOnlineUser = true
}
const initializeWebSocket = () => {
  const socket = new WebSocket('ws://localhost:9999/chat')
  stompClient = Stomp.over(socket)
  stompClient.connect({}, onConnected)
}

const sendMessage = () => {
  if (newMessage.value.trim() !== '') {
    const chatMessage = {
      content: newMessage.value.trim(),
      userName: username.value,
      userId: userId,
      receiver: selectUser.value,
    }
    console.log(chatMessage)
    stompClient?.send('/app/chat.sendMessage', {}, JSON.stringify(chatMessage))
    newMessage.value = ''
  }
}
</script>
