<template>
  <div class="content" @click="hideAll">
    <van-nav-bar
      :title="$route.params.conversationID"
      left-arrow
      @click-left="$router.go(-1)"
    />
    <div class="message_list" ref="msgRef">
      <van-list
        v-model:loading="data.loading"
        :finished="data.finished"
        finished-text="没有更多了"
        direction="up"
        @load="onLoad"
      >
        <MsgItem
          v-for="item in msgList"
          :key="item.msgId"
          :message="item"
          :isSelf="item.fromUid != $route.params.uid"
          :msgLastRead="data.chatInfo?.msgLastRead"
          @preview="preview(item)"
          @revoke="revoke"
          @resend="sendRef.resend(item)"
          @playSound="playSound"
          @stopSound="stopSound"
        ></MsgItem>
      </van-list>
    </div>
    <MsgFooter
      :isHideMore="data.isHideMore"
      :isHideEmoji="data.isHideEmoji"
      :uid="$route.params.uid"
      ref="sendRef"
      @scrollB="scrollBottom"
      @showMore="showMore"
      @showEmoji="showEmoji"
      @hide="hideAll"
    />
  </div>
  <van-image-preview
    v-model:show="data.show"
    :images="data.images"
    :startPosition="data.imageListIndex"
  >
  </van-image-preview>
</template>

<script>
import {
  computed,
  onMounted,
  onDeactivated,
  getCurrentInstance,
  ref,
  watch,
  nextTick,
  reactive,
} from "vue";
import { useStore } from "vuex";
import { useRoute } from "vue-router";
import MsgItem from "@/components/MsgItem.vue";
import MsgFooter from "@/components/MsgFooter.vue";
export default {
  components: {
    MsgItem,
    MsgFooter,
  },
  setup(props, context) {
    const store = useStore();
    const ctx = getCurrentInstance().appContext.config.globalProperties;
    const msgRef = ref("msgList");
    const sendRef = ref("sendRef");
    const route = useRoute();
    const msgList = computed(() => store.state.msgList);
    const curConversationID = computed(() => store.state.curConversationID);
    if (curConversationID.value === null) {
      store.commit("changeChat", route.params.conversationID);
    }
    let curAudio = null;
    let soundOptions = null;
    const data = reactive({
      loading: false,
      finished: false,
      isHideMore: true,
      isHideEmoji: true,
      show: false,
      imageListIndex: 0,
      images: [],
      chatInfo: null,
    });
    let imgMsgIdObj = {};

    watch(
      () => curConversationID.value,
      (count, prevCount) => {
        if (count && count !== prevCount) {
          initMessage();
        }
      }
    );
    watch(
      () => msgList.value.length,
      (count, prevCount) => {
        data.images = [];
        if (count > 0) {
          store.state.msgList.forEach((item) => {
            if (item.type === ctx.$IM.TYPES.MSG_TYPE.Img) {
              data.images.push(item.url);
              imgMsgIdObj[item.msgId] = data.images.length - 1;
            }
          });
        }
        if (prevCount > 0 && count > prevCount) {
          let el = msgRef.value;
          if (el) {
            let scrollTop = el.scrollTop;
            let clientHeight = el.clientHeight;
            let heigth = el.scrollHeight;
            if (scrollTop > 0 && heigth - 100 < clientHeight + scrollTop) {
              scrollBottom();
            }
          }
        }
      }
    );

    // 接收消息
    function received(options) {
      console.log("接收到消息", options);
      let curConversationID = route.params.conversationID;
      let newMsg = options.data[0];
      if (newMsg.conversationID === curConversationID) {
        store.commit("updateMsgs", options.data);
        let uid = route.params.uid;
        if (newMsg.fromUid == uid || options.data.length > 1) {
          ctx.$msim.setMessageRead({
            conversationID: newMsg.conversationID,
          });
        }
      }
    }
    // 撤回消息
    function revoked(options) {
      console.log("接收到撤回消息", options);
      let curConversationID = route.params.conversationID;
      let newMsg = options.data[0];
      if (newMsg.conversationID === curConversationID) {
        store.commit("revokeMsgs", options.data);
      }
    }

    onMounted(() => {
      ctx.$msim
        .getConversationProvider({
          conversationID: route.params.conversationID,
        })
        .then((res) => {
          data.chatInfo = res.data;
          console.log(res, 1);
        });
      ctx.$msim.on(ctx.$IM.EVENT.MESSAGE_RECEIVED, received);
      ctx.$msim.on(ctx.$IM.EVENT.MESSAGE_REVOKED, revoked);
    });

    onDeactivated(() => {
      if (soundOptions) {
        stopSound();
      }
      ctx.$msim.off(ctx.$IM.EVENT.MESSAGE_RECEIVED);
      ctx.$msim.off(ctx.$IM.EVENT.MESSAGE_REVOKED);
    });

    function preview(item) {
      if (item.type !== ctx.$IM.TYPES.MSG_TYPE.Img) return;
      data.imageListIndex = imgMsgIdObj[item.msgId];
      data.show = true;
    }

    function initMessage(msgId) {
      ctx.$msim
        .getMessageList({
          conversationID: route.params.conversationID,
          msgEnd: msgId,
        })
        .then((res) => {
          let msgs = res.data.messages;
          if (msgs && msgs.length > 0) {
            store.commit("addMsgs", msgs);
            if (!msgId) {
              scrollBottom();
            }
          } else {
            data.finished = true;
          }
          data.loading = false;
        })
        .catch((err) => {
          data.loading = false;
          return ctx.$toast(err?.msg || err);
        });
    }

    function revoke(msgObj) {
      ctx.$msim
        .revokeMessage({
          conversationID: msgObj.conversationID,
          msgId: msgObj.msgId,
        })
        .then((res) => {
          msgObj.type = 31;
          return ctx.$toast.success("撤回成功");
        })
        .catch((err) => {
          return ctx.$toast(err?.msg || err);
        });
    }

    function scrollBottom() {
      nextTick(() => {
        if (msgRef.value) {
          msgRef.value.scrollTop = msgRef.value.scrollHeight;
        }
      });
    }
    function onLoad() {
      let msgId;
      if (msgList.value && msgList.value.length > 0) {
        msgId = msgList.value[0].msgId;
      }
      initMessage(msgId);
    }
    function hideAll() {
      if (!data.isHideMore) data.isHideMore = true;
      if (!data.isHideEmoji) data.isHideEmoji = true;
    }
    function showMore() {
      if (data.isHideMore) data.isHideMore = false;
      if (!data.isHideEmoji) data.isHideEmoji = true;
    }
    function showEmoji() {
      if (data.isHideEmoji) data.isHideEmoji = false;
      if (!data.isHideMore) data.isHideMore = true;
    }
    function playSound(options) {
      if (soundOptions) {
        stopSound();
      }
      soundOptions = options;
      curAudio = document.createElement("audio");
      curAudio.src = options.message.url;
      options.play();
      const promise = curAudio.play();
      curAudio.addEventListener("ended", () => {
        stopSound();
      });
      if (promise) {
        promise.catch(() => {
          stopSound();
        });
      }
    }
    function stopSound() {
      curAudio.pause();
      soundOptions.stop();
      curAudio = null;
      soundOptions = null;
    }
    return {
      data,
      msgRef,
      sendRef,
      showMore,
      showEmoji,
      hideAll,
      revoke,
      onLoad,
      preview,
      playSound,
      stopSound,
      scrollBottom,
      msgList: msgList,
    };
  },
};
</script>

<style scoped>
.content {
  display: flex;
  flex: 1;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
  position: relative;
  background-color: #eee;
}
.message_list {
  width: 100%;
  box-sizing: border-box;
  overflow: auto;
  padding: 0 10px;
  height: calc(100% - 50px);
}
</style>