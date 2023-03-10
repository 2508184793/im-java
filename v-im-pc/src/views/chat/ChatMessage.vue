<template>
  <div class="im-chat" v-if="chat">
    <div class="im-chat-top">
      <div>
        <span>{{ chat.name }}</span>
        <span v-if="chat.type === ChatType.GROUP">（{{ count }}人）</span>
      </div>
      <a href="javascript:;" @click="openModal" class="pull-right menu">
        <i class="iconfont icon-v-xinxi" />
      </a>
    </div>
    <div class="im-chat-main">
      <div
        :style="{ left: menu.x + 'px', top: menu.y + 'px' }"
        v-if="menu.show"
        class="right-menu"
      >
        <ul>
          <li>右键菜单</li>
        </ul>
      </div>
      <div class="im-chat-main-left">
        <div class="im-chat-main-box messages" id="message-box">
          <ul>
            <li
              v-for="item in messageList"
              :key="item.id"
              :class="{ 'im-chat-mine': item.mine }"
            >
              <div class="im-chat-user">
                <Avatar :img="userMap.get(item.fromId)?.avatar" />
                <div class="message-info right" v-if="item.mine">
                  <i>
                    <Time :time="item.timestamp" />
                  </i>
                  <span>{{ userMap.get(item.fromId)?.name }}</span>
                </div>
                <div class="message-info" v-if="!item.mine">
                  <span>{{ userMap.get(item.fromId)?.name }}</span>
                  <i>
                    <Time :time="item.timestamp" />
                  </i>
                </div>
              </div>
              <div
                class="im-chat-text"
                v-if="item.messageType === MessageType.text"
              >
                <pre v-html="ChatUtils.transform(item.content)"></pre>
              </div>
              <div
                class="im-chat-text"
                v-if="item.messageType === MessageType.image"
                style="width: 50%"
              >
                <img
                  :src="item.extend.url"
                  style="width: 100%"
                  v-on:click="openImageProxy($event)"
                />
              </div>
              <div
                class="im-chat-text"
                v-if="item.messageType === MessageType.file"
                style="width: 50%"
              >
                <a
                  class="file-box"
                  :title="item.extend.fileName"
                  :href="item.extend.url"
                  v-on:click="openImageProxy($event)"
                >
                  <div class="file-icon">
                    <i class="iconfont icon-v-xiazai"></i>
                  </div>
                  <div class="file-text">
                    <div class="file-name">{{ item.extend.fileName }}</div>
                  </div>
                </a>
              </div>
            </li>
          </ul>
        </div>
        <div class="im-chat-footer">
          <div class="im-chat-tool">
            <i class="iconfont icon-v-smile" @click="showFace = !showFace"></i>
            <upload
              :is-img="true"
              :fileTypes="['png', 'jpg', 'jpeg', 'gif']"
              @uploadSuccess="uploadSuccess"
            >
              <i class="iconfont icon-v-tupian"></i>
            </upload>
            <upload
              :is-img="false"
              :fileTypes="['doc', 'docx', 'xls', 'xlsx', 'pdf', 'zip', 'rar']"
              @uploadSuccess="uploadSuccess"
            >
              <i class="iconfont icon-v-24gl-folderHeart"></i>
            </upload>
            <faces
              v-if="showFace"
              @click="showFace = false"
              class="faces-box"
              @insertFace="insertFace"
            ></faces>
            <el-button
              class="history-message-btn"
              size="small"
              @click="history()"
              >聊天记录
            </el-button>
          </div>
          <textarea
            v-model="messageContent"
            class="textarea"
            @keyup.enter="mineSend()"
          ></textarea>
          <div class="im-chat-send">
            <el-button type="primary" @click="mineSend()" v-if="isGroupUser"
              >发送</el-button
            >
            <el-button type="primary" v-if="!isGroupUser" disabled
              >不在群内
            </el-button>
          </div>
        </div>
      </div>
      <div class="im-chat-users" v-if="chat.type === ChatType.GROUP">
        <el-scrollbar class="chat-user-list">
          <div
            class="user"
            v-for="item in users"
            :key="item.id"
            @click="showUser(item)"
          >
            <avatar :img="item.avatar"></avatar>
            {{ item.name }}
          </div>
        </el-scrollbar>
      </div>
    </div>
  </div>
  <user-modal
    v-if="chat && ChatType.FRIEND === chat.type"
    :show="show"
    :showSend="false"
    :userId="chat.id"
    @close="show = false"
  ></user-modal>
  <group-modal
    v-if="chat && ChatType.GROUP === chat.type"
    :show="show"
    :showSend="false"
    :group-id="chat.id"
    @close="show = false"
  ></group-modal>
  <user-modal
    v-if="chatUser"
    :show="showChatUser"
    :showSend="true"
    :userId="chatUser.id"
    @close="chatUser = false"
  ></user-modal>
  <el-drawer v-model="showHistory" title="聊天记录" :with-header="false" size="60%">
    <history-message
      :from-id="userStore.user.id"
      :chat-users="userMap"
      :chat-id="chat.id"
      :type="chat.type"
      :showHistory="showHistory"
    ></history-message>
  </el-drawer>
</template>

<script setup lang="ts">
import { useChatStore } from "@/store/chatStore.ts";
import { useUserStore } from "@/store/userStore.ts";
import {
  computed,
  getCurrentInstance,
  nextTick,
  onMounted,
  ref,
  watch,
} from "vue";
import Avatar from "@/components/Avatar.vue";
import Time from "@/components/Time.vue";
import Faces from "@/components/Faces.vue";
import UserModal from "@/components/UserModal.vue";
import GroupModal from "@/components/GroupModal.vue";
import HistoryMessage from "@/components/HistoryMessage.vue";
import Chat from "@/mode/Chat";
import Message from "@/mode/Message";
import ChatUtils from "@/utils/ChatUtils.ts";
import ChatType from "@/config/ChatType.ts";
import Upload from "@/views/chat/Upload.vue";
import GroupApi from "@/api/GroupApi";
import User from "@/mode/User";
import MessageApi from "@/api/MessageApi";
import Receipt from "@/mode/Receipt";
import MessageType from "@/utils/MessageType";
import UserApi from "@/api/UserApi";
import { storeToRefs } from "pinia";
import { logout } from "@/api/Login";
// eslint-disable-next-line @typescript-eslint/ban-ts-comment
// @ts-ignore
const { proxy } = getCurrentInstance();
const chatStore = useChatStore();
const userStore = useUserStore();

const menu = ref({
  x: 100,
  y: 100,
  show: false,
});

//是否展示表情
const showFace = ref(false);
//展示用户信息
const show = ref(false);
//群人数
const count = ref(0);
//用户
const users = ref<Array<User>>();
//是否展示群用户
const showChatUser = ref(false);
//是否展示聊天记录
const showHistory = ref(false);
//群单一用户，给user-modal  showChatUser 用
const chatUser = ref<User>();

//是否是群成员，如果是群聊天，并且已经被踢出群，不能再有发送按钮
const isGroupUser = ref(true);
//当前聊天
const chat = computed((): Chat => {
  return chatStore.chats[chatStore.index];
});
const user = userStore.getUser();
if (user) {
  userStore.storeUser(user.id, {
    name: user.name,
    avatar: user.avatar,
  });
}
if (!user) {
  logout().then(() => {
    proxy.$ws.close();
    location.href = "/#/";
  });
}
//聊天对象,
const { userMap } = storeToRefs(userStore);



if (chat.value && chat.value.type === ChatType.FRIEND) {
  UserApi.getUser(chat.value.id).then((res) => {
    const chatUser = res.data;
    userStore.storeUser(chatUser.id, {
      name: chatUser.name,
      avatar: chatUser.avatar,
    });
  });
}

watch(
  chat,
  (n) => {
    isGroupUser.value = true;
    if (n && chat.value.type === ChatType.GROUP) {
      GroupApi.users(n.id).then((res) => {
        res.data.forEach((item: User) => {
          userStore.storeUser(item.id, {
            name: item.name,
            avatar: item.avatar,
          });
        });
        users.value = res.data;
        count.value = res.data.length;
        isGroupUser.value =
          users.value?.find((item) => item.id === user?.id) !== undefined;
      });
    }
    if (n && user) {
      //第一次加载，从数据库中取100条，有序插入到聊天记录里
      MessageApi.list(n.id, user.id, n.type, 10).then((res) => {
        //读取消息
        readMessage();
        res.data.forEach((item: Message) => {
          chatStore.addToMessageList(item, n.id);
          nextTick(() => {
            ChatUtils.imageLoad("message-box");
          });
        });
      });
    }
  },
  {
    immediate: true,
    deep: true,
  }
);

const openModal = () => {
  show.value = true;
};

const messageList = computed((): Array<Message> => {
  return chatStore.chatMessage.get(chat.value.id) ?? new Array<Message>();
});

/**
 * 用户点击消息列表某一个人
 */
const readMessage = () => {
  if (user) {
    let receipt: Receipt = {
      chatId: chat.value.id,
      userId: user.id,
      timestamp: new Date().getTime(),
      type: chat.value.type,
    };
    proxy.$ws.sendRead(receipt);
  }
};

// 附件和图片点击展开
const openImageProxy = (event: any) => {
  event.preventDefault();
  const target = event.currentTarget;
  if (target.nodeName === "IMG") {
    proxy.$winControl.default.openURL(event.target.src);
  } else if (target.className === "file-box" || target.nodeName === "A") {
    proxy.$winControl.default.openURL(target.href);
  }
};

const messageContent = ref("");
const mineSend = (): void => {
  if (messageContent.value && messageContent.value.trim() !== "") {
    if (user) {
      let msg: Message = {
        messageType: MessageType.text,
        chatId: chat.value.id,
        fromId: user.id,
        avatar: user.avatar,
        name: user.name,
        mine: true,
        content: messageContent.value,
        timestamp: new Date().getTime(),
        type: chat.value.type,
      };
      proxy.$ws.sendMessage(msg);
      messageContent.value = "";
      nextTick(() => {
        ChatUtils.imageLoad("message-box");
      });
    }
  }
};

//添加表情
const insertFace = (item: string) => {
  messageContent.value = messageContent.value + "face" + item;
  showFace.value = false;
};

//上传成功
const uploadSuccess = (extend: string, messageType: string) => {
  if (user) {
    let msg: Message = {
      messageType: messageType,
      chatId: chat.value.id,
      fromId: user.id,
      avatar: user.avatar,
      name: user.name,
      mine: true,
      content: "",
      timestamp: new Date().getTime(),
      type: chat.value.type,
      extend: extend,
    };
    proxy.$ws.sendMessage(msg);
  }
};

//历史聊天记录
const history = () => {
  showHistory.value = !showHistory.value;
};

const showUser = (user: User) => {
  showChatUser.value = true;
  chatUser.value = user;
};

onMounted(() => {
  nextTick(() => {
    ChatUtils.imageLoad("message-box");
  });
});
</script>

<style lang="scss">
.right-menu {
  position: absolute;
  z-index: 999;
  background-color: #69cbe9;
  display: inline-block;
}

.im-chat {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.im-chat-top {
  border-bottom: 1px solid #cccccc;
  color: $color-default;
  padding: 0 0 0.2rem 1rem;
  font-size: 1.6rem;
  font-weight: bold;
  height: 40px;
  display: flex;
  justify-content: space-between;
  align-items: center;

  .menu {
    color: $color-default;
    display: inline-block;
    padding: 0 10px;
  }
}

.user-model {
  .user-model-img {
    padding: 15px;
    text-align: center;

    img {
      border-radius: 50%;
    }
  }

  .user-model-item {
    display: flex;
    padding: 5px 0;

    label {
      flex: 2;
      font-weight: bold;
      text-align: right;
    }

    span {
      flex: 3;
    }
  }
}

.im-chat-main {
  flex: 1;
  display: flex;
  flex-direction: row;
  height: calc(100% - 40px);

  .im-chat-main-left {
    flex: 4;
    display: flex;
    flex-direction: column;

    .im-chat-main-box {
      flex: 1;
      padding: 1rem 1rem 0 1rem;
      overflow-x: hidden;
      overflow-y: auto;
    }
  }

  .message-img {
    max-width: 20rem;
  }

  .im-chat-users {
    width: 180px;
    border-left: 1px solid #cccccc;

    .chat-user-list {
      list-style: none;
      margin: 0;

      .user {
        cursor: pointer;
        padding: 5px 2px;
        position: relative;
        display: flex;
        align-items: center;

        &:hover {
          background-color: #eeeeee;

          &:after {
            content: "...";
            position: absolute;
            right: 10px;
            font-weight: bold;
          }
        }

        & > .im-chat-avatar {
          width: 3.2rem;
          height: 3.2rem;
          display: inline-block;
          vertical-align: middle;

          & > img {
            width: 100%;
            height: 100%;
          }
        }
      }
    }
  }

  .messages {
    width: 100%;
    height: calc(100% - 3rem);
    overflow-y: scroll;

    ul {
      width: 100%;

      li {
        position: relative;
        font-size: 0;
        margin-bottom: 10px;
        padding-left: 60px;
        min-height: 68px;

        .im-chat-text {
          position: relative;
          line-height: 22px;
          margin-top: 25px;
          padding: 1rem;
          background-color: #e2e2e2;
          border-radius: 3px;
          color: #333;
          word-break: break-all;
          display: inline-block;
          vertical-align: top;
          font-size: 14px;

          &:after {
            content: "";
            position: absolute;
            left: -10px;
            top: 13px;
            width: 0;
            height: 0;
            border-style: solid dashed dashed;
            border-color: #e2e2e2 transparent transparent;
            overflow: hidden;
            border-width: 10px;
          }

          pre {
            width: 100%;
            white-space: pre-wrap;
            word-break: break-all;

            img {
              display: block;
            }
          }
        }
      }
    }

    .im-chat-user {
      width: 4rem;
      height: 4rem;
      position: absolute;
      display: inline-block;
      vertical-align: top;
      font-size: 14px;
      left: 3px;
      right: auto;

      .message-info {
        position: absolute;
        left: 60px;
        top: -2px;
        width: 500px;
        line-height: 24px;
        font-size: 12px;
        white-space: nowrap;
        color: #999;
        text-align: left;
        font-style: normal;

        i {
          font-style: normal;
          padding-left: 15px;
        }
      }

      .right {
        right: 0;
        text-align: right;
        left: auto;

        i {
          padding-right: 15px;
        }
      }

      img {
        width: 4rem;
        height: 4rem;
      }
    }

    .im-chat-mine {
      text-align: right;
      padding-left: 0;
      padding-right: 60px;

      .im-chat-text {
        margin-left: 10px;
        text-align: left;
        background-color: $color-message-bg;
        color: #fff;
        display: inline-block;
        vertical-align: top;
        font-size: 14px;

        &:after {
          left: auto;
          right: -10px;
          border-top-color: $color-message-bg;
        }
      }

      .im-chat-user {
        left: auto;
        right: 3px;

        cite {
          left: auto;
          right: 60px;
          text-align: right;

          i {
            padding-left: 0;
            padding-right: 15px;
          }
        }

        .message-info {
          right: 60px !important;
          display: inline-block;
        }
      }
    }
  }
}

.im-chat-footer {
  border-top: 1px solid $color-gray;
  height: 15rem;
  display: flex;
  flex-direction: column;

  .im-chat-tool {
    padding: 0.5rem 1rem;
    height: 3.4rem;
    position: relative;

    i {
      font-size: 2rem;
      cursor: pointer;
      margin: 1rem;
    }

    .faces-box {
      position: absolute;
      bottom: 3.8rem;
    }

    .ivu-upload {
      display: inline-block;
    }

    .history-message-btn {
      float: right;
    }
  }

  textarea {
    border: 0;
    padding: 0.5rem;
    width: 100%;
    flex: 1;
    resize: none;
    background-color: $color-box-bg !important;

    &:focus {
      border: 0;
    }
  }

  .im-chat-send {
    height: 4rem;
    text-align: right;
    padding: 0 1rem 1rem 0;
  }
}

.ivu-scroll-wrapper {
  margin: 0 !important;
}

.ivu-scroll-container {
  padding: 15px 15px 5px;
  overflow-y: visible !important;
}

/* 重新覆盖iview 里面的 model 小于768px 时候 宽度变100% 的问题 */
@media (max-width: 768px) {
  .user-model {
    .ivu-modal {
      width: 30rem !important;
      margin: 0 auto;
    }
  }
}

.history-message {
  width: 80%;
  height: calc(100% - 30px);
}

.page {
  position: fixed;
  bottom: 0;
  width: 100%;
  margin: 0.5rem;
}

.ivu-drawer-body {
  padding: 0 !important;

  .messages {
    height: calc(100% - 3rem);
  }
}

.model-footer {
  text-align: right;
  margin: 10px;
}

.file-box {
  width: 100%;
  display: flex;
  background-color: #efefef;
  color: #666666;

  .file-icon {
    background-color: #cccccc;
    padding: 10px;
    flex: 1;
    flex-shrink: 0;

    .iconfont {
      line-height: normal;
      font-size: 4rem;
    }
  }

  .file-text {
    width: 0;
    padding: 10px;
    flex: 5;
    display: flex;
    align-items: center;
    flex-shrink: 0;
    overflow: hidden;

    .file-name {
      -webkit-line-clamp: 2;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
      overflow-wrap: break-word;
      word-break: break-all;
    }
  }
}
</style>
