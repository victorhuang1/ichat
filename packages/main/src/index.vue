<script>
import {
  ref,
  getCurrentInstance,
  onMounted,
  onUpdated,
  h,
  Fragment,
  renderList,
  provide,
  watch,
  nextTick,
} from "vue";
import ChatContent from "../../chat";
import ChatTabs from "./chat-tabs";

// 默认值
const default_mine = {
  id: "10001",
  username: "jule-meteor",
  status: "online",
  sign: "与其感慨路难行,不如马上出发！",
  avatar: "",
};

// 默认配置
const default_config = {
  //简约模式
  brief: true,
  // 标签是否可以删除
  tabRemove: false,
  // 窗口是否可以越界
  moveOut: true,
};

export default {
  name: "IChat",
  props: {
    // 配置项
    config: {
      type: Object,
      default: () => default_config,
    },
    // 当前用户
    mine: {
      type: Object,
      default: () => default_mine,
    },
    // 对话
    chats: {
      type: Array,
      default: () => [],
    },
  },
  components: {
      ChatTabs,
      ChatContent,
    },
  setup(props, ctx) {
    // 会话框组件集合
    const panes = ref([]);
    //  当前选中的会话框，以 index 索引为key
    const selected = ref("0");
    //  会话框状态 键值对
    const paneStateMap = {};

    const chatDisplay = ref(true);
    // 具体实例
    const instance = getCurrentInstance();
    let { config, mine, chats } = props;

    // 过滤出 chat 对话框 filterPaneComponents
    const filterPaneComponents = (vnode = [], componentInstances = []) => {
    Array.from(vnode || []).forEach((node) => {
      let type = node.type;
      type = type.name || type;
      if (type === "chat-content" && node.component) {
        componentInstances.push(node.component);
      } else {
        filterPaneComponents(node.children, componentInstances);
      }
    });
    return componentInstances;
  };
  // 计算会话框实例 calcPaneInstances
  const calcPaneInstances = () => {
  //  因为没有使用 slot 所以不用 slots.default() 作为判断依据
  if (instance.subTree.children) {
    const children = instance.subTree.children;
    if (!children) return;
    const paneVNodes = filterPaneComponents(children).map((item) => {
      return paneStateMap[item.uid];
    });
    // 通过 pane 的uid 判断是否有变动，避免死循环
    const panesChanged = !(
      paneVNodes.length === panes.value.length &&
      paneVNodes.every((item, index) => item.uid === panes.value[index].uid)
    );
    if (panesChanged) {
      panes.value = paneVNodes;
      selected.value = "0";
    }
  } else if (panes.value.length !== 0) {
    panes.value = [];
  }
};


    // 初始化设置 聊天窗口的位置
    const initWindowPosition = () => {
      /**
       * 1、
       *  setup 时期比 created 早，这里的 this 并且vue2 获取的 上下文关系
       *  但是我们可以通过 getCurrentInstance  获取具体实例
       */
      const el = instance.refs.chat;
      if (el) {
        // 等候页面全部加载完在计算坐标
        nextTick(() => {
          let win = document.body;
          el.style.left = (win.clientWidth - 10 - el.clientWidth) / 2 + "px";
          el.style.top = (win.clientHeight - 70 - el.clientHeight) / 2 + "px";
        });
      }
    };
    // 处理窗口拖拽
    const handleDragWindow = (e) => {
      const el = instance.refs.chat;
      let X = e.clientX - el.offsetLeft;
      let Y = e.clientY - el.offsetTop;
      let fixed = false;
      document.onmousemove = function (e) {
        // 阻止冒泡事件
        e.preventDefault();
        let l = e.clientX - X;
        let t = e.clientY - Y;
        // 是否允许窗口越界
        if (!config.moveOut) {
          let win = document.body;
          let stX = fixed ? 0 : win.scrollLeft,
            stY = fixed ? 0 : win.scrollTop;
          let setRig = win.clientWidth - el.clientWidth + stX,
            setBot = win.clientHeight - el.clientHeight + stY;
          l < stX && (l = stX);
          l > setRig && (l = setRig);
          t < stY && (t = stY);
          t > setBot && (t = setBot);
        }
        el.style.left = `${l}px`;
        el.style.top = `${t}px`;
      };
      document.onmouseup = function () {
        document.onmousemove = null;
        document.onmouseup = null;
      };
    };

    // 页面加载后调用
    onMounted(() => {
      calcPaneInstances();
      initWindowPosition();
    });

    // 页面数据变动后调用
    onUpdated(() => {
      calcPaneInstances();
    });
    // 提供子组件 inject 获取
    provide("rootChat", {
      config,
      chatDisplay,
      selected,
    });
    provide("updatePaneStateCallback", (pane) => {
      paneStateMap[pane.uid] = pane;
    });
    return {
      panes,
      selected,
      chatDisplay,
      handleDragWindow,
      paneStateMap,
    };
  },
  methods: {
    // 处理会话框显示
    handleChatDisplay() {
      this.chatDisplay = !this.chatDisplay;
    },
    // 处理会话标签单击事件
    handleTabClick(pane) {
      this.selected = pane.index;
    },
    // 处理会话标签删除事件
    handleTabRemove(pane) {
      console.log(pane);
    },
  },
  render() {
    let {
      mine,
      chats,
      chatDisplay,
      panes,
      handleChatDisplay,
      handleTabClick,
      handleTabRemove,
      handleDragWindow,
    } = this;
    const el_chat_panes = renderList(chats, (chat) => {
      // 构建会话窗口
      return h(ChatContent, {
        chat,
      });
    });
    // 会话标签
    const el_chat_tabs = h(ChatTabs, {
      panes,
      onTabClick: handleTabClick,
      onTabRemove: handleTabRemove,
      onChatDisplay: handleChatDisplay,
      onDragWindow: handleDragWindow,
    });
    // 主体内容
    // 主体内容
    const el_chat_content = h(
      "div",
      {
        class: ["im-layer-tabs", "im-layer-content"],
        ref: "ChatContent",
      },
      [el_chat_tabs, el_chat_panes]
    );
    // 窗口设定烂
    const el_chat_win_setting = h(
      "span",
      {
        class: "im-box-setwin",
      },
      [
        h("i", {
          class: "i-icon-top",
          title: "置顶",
          onClick: (ev) => {},
        }),
        h("i", {
          class: "i-icon-minus",
          title: "最小化",
          onClick: (ev) => {},
        }),
        h("i", {
          class: "i-icon-maxus",
          title: "最大化",
          onClick: (ev) => {},
        }),
        h("i", {
          class: "i-icon-close",
          title: "关闭",
          onClick: (ev) => {},
        }),
      ]
    );
    // 宽度和高度设置
    const el_chat_win_resize = h("span", { class: "im-icon-resize" });
    // 入口
    const el_chat_main = h(
      "div",
      {
        class: [
          "im-layer  layer-anim im-box im-chat",
          chatDisplay ? "chat-show" : "",
        ],
        ref: "chat",
        style: { "z-index": 1002, display: "inline" },
      },
      [
        h("div", {
          class: "im-layer-title",
          style: "cursor: move;",
          onMousedown: (ev) => {
            handleDragWindow(ev);
          },
        }),
        el_chat_content,
        el_chat_win_setting,
        el_chat_win_resize,
      ]
    );
    return h("div", [el_chat_main]);
  },
};
</script>

<style>
.chat-show {
  -webkit-box-shadow: 1px 1px 10px rgba(0, 0, 0, 0.3);
  box-shadow: 1px 1px 10px rgba(0, 0, 0, 0.3);
  min-width: 500px;
  width: 800px;
}

.chat-show.alone {
  width: 620px;
}
</style>