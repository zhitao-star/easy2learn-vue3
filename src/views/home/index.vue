<template>
  <div class="common-layout">
    <el-container>
      <el-aside>
        <span>在线用户</span>
        <ul
          v-infinite-scroll="load"
          class="infinite-list"
          style="overflow: auto"
        >
          <li v-for="i in count" :key="i" class="infinite-list-item">
            {{ i }}
          </li>
        </ul>
      </el-aside>
      <el-container>
        <el-header>Header</el-header>
        <el-main>Main</el-main>
        <el-footer>Footer</el-footer>
      </el-container>
    </el-container>
  </div>
</template>
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
const load = () => {
  count.value += 2
}
</script>
<style lang="scss">
.el-aside {
  width: 300px;
  height: 800px;
  background-color: red;
  text-align: center;
}
.infinite-list {
  height: 300px;
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
