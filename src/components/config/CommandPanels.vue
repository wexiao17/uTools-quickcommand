<template>
  <div
    class="command-panel-container absolute-right"
    :style="{
      bottom: footerBarHeight,
      left: tabBarWidth,
    }"
  >
    <div class="current-tag-indicator q-pa-xs">
      {{ currentTag }}
    </div>
    
    <q-scroll-area
      ref="scrollArea"
      class="full-height full-width"
      :thumb-style="{
        background: 'grey',
        width: '6px',
        opacity: 0.5,
      }"
      @wheel="handleScrollWheel"
    >
      <draggable
        :model-value="currentTagQuickCommands"
        @update:model-value="handleCommandsReorder"
        :component-data="{
          type: 'div',
          class: 'row wrap justify-start items-start command-items',
        }"
        item-key="features.code"
        handle=".q-card"
        :disabled="currentTag === '默认' || currentTag === '搜索结果'"
      >
        <template #item="{ element: commandInfo }">
          <div
            :key="commandInfo.features.code"
            :style="{
              width: cardStyleSheet[$root.profile.commandCardStyle].width,
            }"
            class="relative-position q-pa-sm command-item"
          >
            <CommandCard
              :commandInfo="commandInfo"
              :isCommandActivated="
                activatedQuickCommandFeatureCodes.includes(
                  commandInfo.features.code
                )
              "
              :cardStyle="cardStyleSheet[$root.profile.commandCardStyle]"
              @commandChanged="$emit('command-changed', $event)"
            ></CommandCard>
          </div>
        </template>
      </draggable>
    </q-scroll-area>
  </div>
</template>

<script>
import CommandCard from "components/CommandCard.vue";
import draggable from "vuedraggable";
import { useCommandManager } from "js/commandManager.js";
import { dbManager } from "js/utools.js";

export default {
  name: "CommandPanels",
  components: {
    CommandCard,
    draggable,
  },
  data() {
    return {
      commandManager: useCommandManager(),
      cardStyleSheet: {
        mini: {
          width: "20%",
          code: 1,
        },
        dense: {
          width: "33%",
          code: 2,
        },
        normal: {
          width: "50%",
          code: 3,
        },
        large: {
          width: "100%",
          code: 4,
        },
      },
      lastWheelTime: 0,
      initialTabBarWidth: null, // 存储初始标签栏宽度
      lastCommandCardStyle: null, // 记录上一次的卡片样式
      maxTabBarWidth: null, // 存储最大标签栏宽度
    };
  },
  props: {
    footerBarHeight: {
      type: String,
      required: true,
    },
    tabBarWidth: {
      type: String,
      required: true,
    },
  },
  mounted() {
    // 确保组件尺寸正确
    this.adjustContainerSize();
    window.addEventListener('resize', this.adjustContainerSize);
    
    // 存储初始标签栏宽度
    this.initialTabBarWidth = this.tabBarWidth;
    this.maxTabBarWidth = this.tabBarWidth;
    
    // 初始化时检查当前视图类型
    this.lastCommandCardStyle = this.$root.profile.commandCardStyle;
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.adjustContainerSize);
  },
  computed: {
    currentTag: {
      get() {
        return this.commandManager.state.currentTag;
      },
      set(value) {
        this.commandManager.state.currentTag = value;
      },
    },
    allQuickCommandTags() {
      return this.commandManager.state.allQuickCommandTags;
    },
    activatedQuickCommandFeatureCodes() {
      return this.commandManager.state.activatedQuickCommandFeatureCodes;
    },
    commandSearchKeyword() {
      return this.commandManager.state.commandSearchKeyword;
    },
    allQuickCommands() {
      return this.commandManager.state.allQuickCommands;
    },
    currentTagQuickCommands() {
      let commands = Object.values(
        window.lodashM.cloneDeep(this.allQuickCommands)
      );

      // 根据 order 排序并确保没有空格子
      const sortCommands = (cmds) => {
        // 先按照激活状态分组
        const activatedCmds = cmds.filter(cmd => 
          this.activatedQuickCommandFeatureCodes.includes(cmd.features.code)
        );
        const inactiveCmds = cmds.filter(cmd => 
          !this.activatedQuickCommandFeatureCodes.includes(cmd.features.code)
        );
        
        // 分别排序
        const sortByOrder = (cmdList) => {
          return cmdList.sort((a, b) => {
            const orderA = a.order ?? Number.MAX_SAFE_INTEGER;
            const orderB = b.order ?? Number.MAX_SAFE_INTEGER;
            return orderA - orderB;
          });
        };
        
        // 先显示已激活的，再显示未激活的
        return [...sortByOrder(activatedCmds), ...sortByOrder(inactiveCmds)];
      };

      switch (this.currentTag) {
        case "未分类":
          return sortCommands(
            commands.filter((cmd) => !cmd.tags || cmd.tags.length === 0)
          );
        case "搜索结果":
          if (this.commandSearchKeyword?.length < 2) return [];
          let searchResult = [];
          commands.forEach((cmd) => {
            // 拼音搜索
            let explain = cmd.features.explain;
            let matchedWordPositions = window.pinyinMatch.match(
              explain,
              this.commandSearchKeyword
            );
            if (!matchedWordPositions) return;
            let matchedWords = explain.slice(
              matchedWordPositions[0],
              matchedWordPositions[1] + 1
            );
            // 高亮
            cmd.features.explain = explain.replace(
              matchedWords,
              `<strong style="color:#ed6237">${matchedWords}</strong>`
            );
            searchResult.push(cmd);
          });
          // 搜索结果也按照激活状态排序
          return sortCommands(searchResult);
        case "默认":
          return sortCommands(
            commands.filter((cmd) => cmd.tags?.includes(this.currentTag))
          );
        default:
          return sortCommands(
            commands.filter((cmd) => cmd.tags?.includes(this.currentTag))
          );
      }
    },
  },
  methods: {
    handleCommandsReorder(commands) {
      // 更新当前tag下的命令顺序
      const tagCommands = {};
      commands.forEach((command, index) => {
        tagCommands[command.features.code] = {
          ...command,
          order: index, // 添加排序信息
        };
      });

      // 更新存储
      this.commandManager.state.allQuickCommands = {
        ...this.allQuickCommands,
        ...tagCommands,
      };

      // 只保存被修改的命令
      this.saveCurrentTagOrderedCommand(tagCommands);
    },
    saveCurrentTagOrderedCommand(tagCommands) {
      // 只保存被修改的命令
      Object.entries(tagCommands).forEach(([code, command]) => {
        if (!this.commandManager.isDefaultCommand(code)) {
          dbManager.putDB(window.lodashM.cloneDeep(command), "qc_" + code);
        }
      });
    },
    // 新增方法：调整容器尺寸
    adjustContainerSize() {
      const container = this.$el;
      if (!container) return;
      
      // 设置正确的高度
      container.style.height = `calc(100% - ${this.footerBarHeight})`;
      
      // 使用固定宽度或初始宽度，确保分类缩进时界面不变小
      if (this.$root.profile.commandCardStyle === 'mini' && 
          this.initialTabBarWidth && 
          this.tabBarWidth !== this.initialTabBarWidth) {
        // 如果标签栏宽度变小了（缩进），使用初始宽度计算
        container.style.width = `calc(100% - ${this.initialTabBarWidth})`;
        // 调整左侧位置，确保右侧对齐
        container.style.left = this.initialTabBarWidth;
      } else {
        // 正常情况下使用当前标签栏宽度
        container.style.width = `calc(100% - ${this.tabBarWidth})`;
        container.style.left = this.tabBarWidth;
      }
    },
    // 新的滚轮处理方法
    handleScrollWheel(event) {
      // 获取滚动区域
      const scrollArea = this.$refs.scrollArea;
      if (!scrollArea) return;
      
      // 获取滚动信息
      const scrollInfo = scrollArea.getScroll();
      const isAtBottom = scrollInfo.verticalPosition >= scrollInfo.verticalSize - scrollInfo.verticalContainerSize - 5; // 5px误差容忍
      const hasScrollbar = scrollInfo.verticalSize > scrollInfo.verticalContainerSize;
      
      // 如果没有滚动条或者已经滚动到底部，则切换分类
      if (!hasScrollbar || isAtBottom && event.deltaY > 0) {
        this.switchCategory(event.deltaY > 0);
        event.preventDefault(); // 阻止默认滚动
      } else if (isAtBottom && event.deltaY < 0) {
        // 如果在底部向上滚动，正常滚动
        // 不阻止默认行为
      } else if (scrollInfo.verticalPosition <= 0 && event.deltaY < 0) {
        // 如果在顶部向上滚动，切换到上一个分类
        this.switchCategory(false);
        event.preventDefault(); // 阻止默认滚动
      }
      // 其他情况正常滚动
    },
    // 切换分类的方法
    switchCategory(isNext) {
      const now = Date.now();
      if (now - this.lastWheelTime < 200) {
        // 防止切换过快
        return;
      }
      this.lastWheelTime = now;
      
      const tags = this.allQuickCommandTags;
      if (tags.length <= 1) return;
      
      const currentIndex = tags.indexOf(this.currentTag);
      let newIndex;
      
      if (isNext) {
        // 下一个分类
        newIndex = (currentIndex + 1) % tags.length;
      } else {
        // 上一个分类
        newIndex = (currentIndex - 1 + tags.length) % tags.length;
      }
      
      // 切换标签
      this.currentTag = tags[newIndex];
      
      // 切换后重置滚动位置
      this.$nextTick(() => {
        if (this.$refs.scrollArea) {
          this.$refs.scrollArea.setScrollPosition('vertical', 0);
        }
      });
    },
  },
  watch: {
    // 监听标签栏宽度变化
    tabBarWidth: {
      handler(newWidth) {
        // 记录最大宽度
        if (!this.maxTabBarWidth || parseFloat(newWidth) > parseFloat(this.maxTabBarWidth)) {
          this.maxTabBarWidth = newWidth;
        }
        
        // 当标签栏宽度变化时重新调整容器尺寸
        this.$nextTick(() => {
          this.adjustContainerSize();
        });
      },
      immediate: true
    },
    // 监听卡片样式变化
    '$root.profile.commandCardStyle': {
      handler(newStyle) {
        // 如果切换到五列视图(mini)，使用最大宽度作为基准
        if (newStyle === 'mini') {
          // 使用记录的最大宽度
          this.initialTabBarWidth = this.maxTabBarWidth || this.tabBarWidth;
        }
        
        this.lastCommandCardStyle = newStyle;
        
        // 立即调整容器尺寸
        this.$nextTick(() => {
          this.adjustContainerSize();
        });
      },
      immediate: true
    }
  },
  emits: ["command-changed"],
};
</script>

<style scoped>
.command-panel-container {
  position: absolute;
  top: 0;
  right: 0;
  overflow: hidden;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  /* 移除内联样式绑定，改为在方法中动态设置 */
}

.current-tag-indicator {
  position: absolute;
  top: 10px;
  right: 10px;
  background-color: rgba(0, 0, 0, 0.6);
  color: white;
  border-radius: 4px;
  padding: 4px 8px;
  font-size: 14px;
  z-index: 10;
  transition: opacity 0.3s;
  opacity: 0.7;
}

.current-tag-indicator:hover {
  opacity: 1;
}

.command-items {
  padding: 8px;
  width: 100%;
  box-sizing: border-box;
}

.command-item {
  transition: all 0.5s;
  box-sizing: border-box;
  padding: 4px;
}

.full-height {
  height: 100%;
}

.full-width {
  width: 100%;
}
</style>
