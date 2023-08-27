<template>
  <div class="main-container">
    <el-container>
      <el-aside>
        <span>在线用户数量:{{ onlineCount }}</span>
        <ul
          v-infinite-scroll="load"
          class="infinite-list"
          s
          style="overflow: auto"
        >
          <!-- 循环在线用户列表 -->
          <li
            @click="selectUserHandle(user.userId)"
            v-for="user in onlineUser"
            :key="user.value"
            class="infinite-list-item"
          >
            {{ user.userName }}
          </li>
        </ul>
      </el-aside>
      <el-container>
        <el-header>
          <h2>请输入当前的用户名：</h2>
          <el-input
            v-model="username"
            placeholder="Please input"
            @keyup.enter="finishUserName"
          />
        </el-header>
        <el-main>
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
        </el-main>
        <el-footer>Footer</el-footer>
      </el-container>
    </el-container>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, reactive } from 'vue'
import { ElMessage } from 'element-plus'
import { v4 as uuidv4 } from 'uuid'

const userId = uuidv4()

let isSelectUser = ref<boolean>(false)

const selectUserHandle = (userId: string) => {
  isSelectUser.value = true
  console.log(userId)
}

const count = ref(0)
const load = () => {
  count.value += 2
}

// eslint-disable-next-line @typescript-eslint/ban-ts-comment
//@ts-expect-error
import * as Stomp from 'stompjs'

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
const onlineUser = ref<any[]>([])
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
  //在线的所有用户
  stompClient?.subscribe('/online/onlineOrOfflineUserList', (message: any) => {
    onlineUser.value = []
    let onlineUserList = JSON.parse(message.body)
    onlineUserList.forEach((item: any) => {
      onlineUser.value.push(item)
    })
  })
  //加入或者离开的用户
  stompClient?.subscribe('/online/onlineOrOfflineUser', (message: any) => {
    let onlineUserMessage = JSON.parse(message.body)
    onlineCount.value = onlineUserMessage.currentOnlineUserCount
    //有人加入或者离开了聊天室
    if (onlineUserMessage.userType) {
      ElMessage({
        message: onlineUserMessage.userName + '加入了聊天室',
        type: 'success',
      })
    } else {
      ElMessage({
        message: onlineUserMessage.userName + '离开了聊天室',
        type: 'success',
      })
    }
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
<style scoped>
.main-container {
  margin: 10px;
}
.el-input {
  margin: 10px;
}
.el-aside {
  width: 300px;
  height: 850px;
  text-align: center;
}
.infinite-list {
  height: 800px;
  padding: 0;
  margin: 0;
  list-style: none;
}
.infinite-list .infinite-list-item {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 50px;
  background: var(--el-color-primary-light-9);
  margin: 10px;
  color: var(--el-color-primary);
}
.infinite-list .infinite-list-item + .list-item {
  margin-top: 10px;
}
</style>
