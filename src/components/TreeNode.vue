<template>
  <div class="tree-node" @copy="handleCopy">
    <!-- 当前节点 -->
    <div :style="{ paddingLeft: `${level * 16}px` }" class="node-label">
      <!-- 折叠按钮 -->
      <span v-if="showCollapseButton" class="collapse-toggle" @click="toggle">
        {{ collapsed ? "+" : "-" }}
      </span>

      <!-- 单行显示无子节点 -->
      <template
        v-if="!hasChildren && node.name !== 'root' && node.name !== '#text'"
      >
        <span class="tag">
          &lt;{{ node.name }}&gt;{{ node.value ?? "" }}&lt;/{{ node.name }}&gt;
        </span>
      </template>

      <!-- 普通标签 -->
      <template v-else-if="node.name !== 'root' && node.name !== '#text'">
        <span class="tag">&lt;{{ node.name }}&gt;</span>
      </template>

      <!-- 文本节点 -->
      <template v-else-if="node.name === '#text'">
        <span class="value">{{ node.value }}</span>
      </template>
    </div>

    <!-- 子节点 -->
    <div v-if="hasChildren && !collapsed">
      <TreeNode
        v-for="(child, index) in node.children"
        :key="index"
        :node="child"
        :level="level + 1"
      />
    </div>

    <!-- 关闭标签 -->
    <div
      v-if="
        hasChildren &&
        node.name !== 'root' &&
        node.name !== '#text' &&
        !collapsed
      "
      :style="{ paddingLeft: `${level * 16}px` }"
      class="tag"
    >
      &lt;/{{ node.name }}&gt;
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, computed, type PropType } from "vue";
import type { XmlNode } from "./XmlTree.vue";

export default defineComponent({
  name: "TreeNode",
  props: {
    node: {
      type: Object as PropType<XmlNode>,
      required: true,
    },
    level: {
      type: Number,
      default: 0,
    },
  },
  setup(props) {
    const collapsed = ref(false);

    const toggle = (): void => {
      collapsed.value = !collapsed.value;
    };

    const hasChildren = computed(() => {
      return !!(props.node.children && props.node.children.length > 0);
    });

    const showCollapseButton = computed(() => {
      if (props.node.name === "root" && props.node.children?.length === 1) {
        return false;
      }
      return hasChildren.value;
    });

    /**
     * 递归生成带缩进的 XML 文本
     */
    const generateXmlText = (
      node: XmlNode,
      indent: number = 0,
      isRoot: boolean = false,
    ): string => {
      const pad = "  ".repeat(indent);

      if (node.name === "#text") {
        return pad + (node.value ?? "");
      }

      const children = node.children ?? [];

      // 顶层只有一个节点时，不生成 root
      if (isRoot && node.name === "root" && children.length === 1) {
        return generateXmlText(children[0], indent, true);
      }

      if (children.length === 0) {
        return `${pad}<${node.name}>${node.value ?? ""}</${node.name}>`;
      }

      const innerXml = children
        .map((child) => generateXmlText(child, indent + 1))
        .join("\n");

      return `${pad}<${node.name}>\n${innerXml}\n${pad}</${node.name}>`;
    };

    /**
     * 监听原生 copy 事件
     */
    const handleCopy = (event: ClipboardEvent): void => {
      if (!event.clipboardData) return;

      const xmlText = generateXmlText(props.node, 0, true);

      event.clipboardData.setData("text/plain", xmlText);
      event.preventDefault();
    };

    return { collapsed, toggle, hasChildren, handleCopy, showCollapseButton };
  },
});
</script>

<style scoped>
.tree-node {
  font-family: monospace;
  line-height: 1.5;
  user-select: text; /* 允许复制文本 */
}

.node-label {
  display: block;
}

.tag {
  color: var(--ds-theme-primary);
}

.value {
  color: var(--ds-text-light);
}

.collapse-toggle {
  cursor: pointer;
  color: var(--ds-theme-secondary);
  user-select: none;
  width: 16px;
  display: inline-block;
  text-align: center;
  font-size: 16px;
  line-height: 1.5;
}
</style>
