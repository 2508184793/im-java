<template>
  <div class="main">
    <div class="left">
      <div class="title">
        <el-row>
          <el-col :span="20">
            <div class="text">聊天</div>
          </el-col>
          <el-col :span="4" class="add">
            <!--            <i class="iconfont icon-v-add" title="拉群聊天"></i>-->
          </el-col>
        </el-row>
      </div>
      <el-scrollbar class="list" v-if="show">
        <chat-item
          v-for="(chat, index) in store.chats"
          :key="chat.id"
          :index="index"
          :id="chat.id"
          :img="chat.avatar"
          :username="chat.name"
          :unreadCount="chat.unreadCount"
          :text="chat.lastMessage"
          :showDel="true"
          :active="index === store.index"
          @del="delChat"
          @click="showChat(chat)"
        ></chat-item>
      </el-scrollbar>
    </div>
    <div class="right">
      <Top></Top>
      <chat-message v-if="chat"></chat-message>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useChatStore } from "@/store/chatStore";
import Chat from "@/mode/Chat";
import ChatMessage from "@/views/chat/ChatMessage.vue";
import Top from "@/components/Top.vue";
import ChatItem from "@/components/ChatItem.vue";
import { computed, nextTick, onMounted, ref } from "vue";
import ChatUtils from "@/utils/ChatUtils";
const chatStore = useChatStore();
const store = useChatStore();
const show = ref<boolean>(false);
const chat = computed((): Chat => {
  return chatStore.chats[chatStore.index];
});
const showChat = (chat: Chat) => {
  store.openChat(chat);
  nextTick(() => {
    ChatUtils.imageLoad("message-box");
  });
};
const delChat = (id: string) => {
  store.delChat(id);
};
onMounted(() => {
  show.value = true;
});
</script>

<style lang="scss" scoped></style>
