<template>
  <div>
    <div class="group">
      <div class="header">
        <avatar-upload
          v-if="isMaster"
          :avatar="dataForm.avatar"
          @uploadSuccess="uploadSuccess"
        ></avatar-upload>
        <avatar img="dataForm.avatar" v-if="!isMaster" size="large" />
      </div>
      <div class="box">
        <el-form
          :inline="true"
          ref="ruleFormRef"
          :model="dataForm"
          :rules="rules"
        >
          <el-form-item label="群名称:" prop="name">
            <el-input v-model="dataForm.name" :readonly="!isMaster"></el-input>
          </el-form-item>
          <el-form-item label="进群审核:" v-if="isMaster">
            <el-switch v-model="dataForm.needCheck" />
          </el-form-item>
        </el-form>
        <div>
          <el-button @click="updateGroup" v-if="isMaster">保存</el-button>
          <el-button @click="deleteGroup" v-if="isMaster">解散</el-button>
          <el-button @click="openChat">聊天</el-button>
        </div>
      </div>
    </div>
    <el-divider />
    <div class="members">
      <el-row :gutter="20">
        <el-col v-if="isMaster" :span="3" @click="addUser()">
          <div class="add-user">
            <i class="iconfont icon-v-add"></i>
          </div>
        </el-col>
        <el-col
          :span="3"
          v-for="user in users"
          :key="user.id"
          class="user user-box"
          @click="openUser(user.id)"
        >
          <Avatar :img="user.avatar"></Avatar>
          <div>{{ user.name }}</div>
          <div
            class="close"
            @click.stop="deleteUser(user.id)"
            v-if="user.id !== dataForm.master && isMaster"
          >
            <i class="iconfont icon-v-close"></i>
          </div>
        </el-col>
      </el-row>
    </div>
    <user-modal
      :show="show"
      :showSend="true"
      :userId="userId"
      @close="show = false"
    ></user-modal>
    <el-drawer v-model="drawer" title="添加群成员" size="50%" direction="rtl">
      <add-group-user
        v-if="drawer"
        :group="dataForm"
        @reload="loadUser"
      ></add-group-user>
    </el-drawer>
  </div>
</template>

<script setup lang="ts">
import { getCurrentInstance, reactive, ref, toRefs } from "vue";
import { useUserStore } from "@/store/userStore";
import { useChatStore } from "@/store/chatStore";
import { useRoute, useRouter } from "vue-router";
import type { ElForm } from "element-plus";
import { ElMessage, ElMessageBox } from "element-plus";

import Avatar from "@/components/Avatar.vue";
import AddGroupUser from "@/views/group/AddGroupUser.vue";
import AvatarUpload from "@/components/AvatarUpload.vue";
import FetchRequest from "@/api/FetchRequest";
import Auth from "@/api/Auth";
import GroupApi from "@/api/GroupApi";
import User from "@/mode/User";
import UserModal from "@/components/UserModal.vue";
import { useGroupStore } from "@/store/groupStore";
import ChatType from "@/config/ChatType";
import Group from "@/mode/Group";
import Message from "@/mode/Message";
import UserApi from "@/api/UserApi";
// eslint-disable-next-line @typescript-eslint/ban-ts-comment
// @ts-ignore
const { proxy } = getCurrentInstance();

const ruleFormRef = ref<InstanceType<typeof ElForm>>();
const groupStore = useGroupStore();
const router = useRouter();
const store = useUserStore();
const chatStore = useChatStore();
const route = useRoute();

interface VimData {
  host: string;
  headers: any;
  needCheck: boolean;
  data: any;
  users: Array<User>;
  show: boolean;
  userId: string;
  isMaster: boolean;
  drawer: boolean;
}

const dataForm = reactive<Group>({
  id: "",
  name: "",
  needCheck: "",
  avatar: "/img/default-group.png",
  master: "",
});

const rules = reactive({
  name: [
    {
      required: true,
      message: "必填",
      trigger: "blur",
    },
    {
      min: 3,
      max: 10,
      message: "长度介于3-10",
      trigger: "blur",
    },
  ],
});
const vimData = reactive<VimData>({
  host: FetchRequest.getHost(),
  headers: {
    "Access-Control-Allow-Origin": "*",
  },
  data: {
    access_token: Auth.getToken(),
    type: "file",
  },
  needCheck: false,
  users: new Array<User>(),
  show: false,
  userId: "",
  //是否是群管理员
  isMaster: false,
  //是否展示添加用户(抽屉)
  drawer: false,
});
const groupId = route.params.id;
if (typeof groupId === "string") {
  GroupApi.get(groupId)
    .then((res) => {
      dataForm.avatar = res.data.avatar;
      dataForm.id = res.data.id;
      dataForm.name = res.data.name;
      dataForm.master = res.data.master;
      dataForm.needCheck = res.data.needCheck;
      vimData.isMaster = res.data.master === store.user?.id;
      loadUser();
    })
    .catch((err) => {
      ElMessage.error(err.message);
    });
}
//加载群用户
const loadUser = () => {
  vimData.users.splice(0, vimData.users.length);
  if (typeof groupId === "string") {
    GroupApi.users(groupId).then((res) => {
      vimData.drawer = false;
      res.data.forEach((item: User) => {
        vimData.users.push(item);
      });
    });
  }
};

const uploadSuccess = (url: string) => {
  dataForm.avatar = url;
};

const updateGroup = () => {
  ruleFormRef.value?.validate((f) => {
    if (f) {
      GroupApi.update(
        dataForm.id,
        dataForm.name,
        dataForm.avatar,
        dataForm.needCheck ? "1" : "0"
      )
        .then(() => {
          groupStore.reloadGroupList();
          ElMessage.success("保存成功");
        })
        .catch(() => {
          ElMessage.error("保存失败");
        });
    }
  });
};
const openUser = (id: string) => {
  if (id !== "") {
    vimData.show = true;
    vimData.userId = id;
  }
};

const addUser = () => {
  vimData.drawer = true;
};
//从群众删除拥护
const deleteUser = (userId: string) => {
  ElMessageBox.confirm("是否从群中删除此人?", "提示", {
    confirmButtonText: "确定",
    cancelButtonText: "取消",
    type: "warning",
  }).then(() => {
    if (typeof groupId === "string") {
      GroupApi.deleteUser(groupId, userId).then((res) => {
        if (res) {
          loadUser();
          ElMessage.success("删除成功!");
        } else {
          ElMessage.error("删除失败");
        }
      });
    }
  });
};

const deleteGroup = () => {
  ElMessageBox.confirm("是否解散此群?", "提示", {
    confirmButtonText: "确定",
    cancelButtonText: "取消",
    type: "warning",
  }).then(() => {
    if (typeof groupId === "string") {
      GroupApi.delete(groupId).then((res) => {
        if (res) {
          groupStore.reloadGroupList();
          notify(dataForm.master, groupId);
          ElMessage.success("解散成功!");
          router.push("/index/group/new");
        } else {
          ElMessage.error("解散失败");
        }
      });
    }
  });
};

/**
 * 通知用户群已经被删除
 * @param master 群主id
 * @param groupId 群id
 */
const notify = (master: string, groupId: string) => {
  if (master) {
    const content = "群已被解散";
    UserApi.getUser(master).then((res) => {
      let user = res.data;
      let msg: Message = {
        chatId: groupId,
        fromId: master,
        avatar: user.avatar,
        name: user.name,
        mine: true,
        content: content,
        timestamp: new Date().getTime(),
        type: ChatType.GROUP,
      };
      proxy.$ws.sendMessage(msg);
    });
  }
};

/**
 * 打开聊天室
 */
const openChat = () => {
  chatStore.openChat({
    id: dataForm.id,
    name: dataForm.name,
    avatar: dataForm.avatar,
    type: ChatType.GROUP,
    lastMessage: "",
    unreadCount: 0,
    isLoading: false,
    loaded: true,
  });
  router.push("/index/chat");
};

const { users, show, userId, isMaster, drawer } = toRefs(vimData);
</script>

<style scoped lang="scss">
.group {
  padding: 15px;
  display: flex;
}

.header {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

.box {
  flex: 5;
  padding: 15px;
}

.members {
  padding: 0 15px;
  text-align: center;
}

.add-user {
  display: flex;
  align-items: flex-start;
  justify-content: center;
  width: 100%;
  height: 100%;
  font-size: 6rem;
  color: #ccc;

  i {
    font-size: 30px;
    line-height: 40px;
    width: 40px;
    height: 40px;
    text-align: center;
    border: 1px solid #cccccc;
  }
}

.user {
  position: relative;

  .close {
    position: absolute;
    top: -8px;
    right: -8px;
    background-color: #ff0000;
    width: 16px;
    height: 16px;
    color: #ffffff;
    border-radius: 8px;

    .icon-v-close {
      font-size: 12px;
    }
  }
}
.user-box {
  margin-left: 10px;
  margin-right: 10px;
  border: 1px solid #cccccc;
  padding: 10px;
  flex: 0 0 80px;
}
</style>
