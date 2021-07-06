<script>
import {
  ref,
  getCurrentInstance,
  computed,
  inject,
  h,
  Fragment,
  renderList,
  provide,
  watch,
} from "vue";

export default {
  name: "chat-content",
  componentName: "ChatContent",
  props: {
    // 会话信息
    chat: {
      type: Object,
      default: () => {},
    },
  },
  setup(props, ctx) {
    const rootChat = inject("rootChat");
    const updatePaneState = inject("updatePaneStateCallback");

    const index = ref(null);
    // 判断是否被 父级选中
    const active = computed(() => {
      return rootChat.selected.value === index.value;
    });
    const instance = getCurrentInstance();
    updatePaneState({
      uid: instance.uid,
      chat: props.chat,
      active,
      index,
    });
    return {
      active,
      index,
    };
  },
  render() {
    let { index, active } = this;
    // 聊天窗口主体
    const el_chat_content = h("div", { class: ["im-chat-main", "page"] }, [
      h("div", {
        class: ["im-pane-item", active ? "display" : ""],
        style: { height: "440px" },
      }),
    ]);
    return el_chat_content;
  },
};
</script>