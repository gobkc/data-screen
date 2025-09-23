<template>
  <div class="xml-tree">
    <TreeNode :node="rootNode" :level="0" />
  </div>
</template>

<script lang="ts">
import { defineComponent, computed, type PropType } from "vue";
import TreeNode from "./TreeNode.vue";

export interface XmlNode {
  name: string;
  attributes?: Record<string, string>;
  children?: XmlNode[];
  value?: string;
}

export type JsonLike = Record<string, unknown>;

export default defineComponent({
  name: "XmlTree",
  components: { TreeNode },
  props: {
    data: {
      type: [Object, String] as PropType<JsonLike | string>,
      required: true,
    },
  },
  setup(props) {
    // 解析 XML 字符串为 XmlNode
    const parseXmlString = (xmlStr: string): XmlNode => {
      const parser = new DOMParser();
      const xmlDoc = parser.parseFromString(xmlStr, "application/xml");

      const parseElement = (el: Element): XmlNode => {
        const children: XmlNode[] = [];
        el.childNodes.forEach((child) => {
          if (child.nodeType === Node.ELEMENT_NODE) {
            children.push(parseElement(child as Element));
          } else if (child.nodeType === Node.TEXT_NODE) {
            const text = child.textContent?.trim();
            if (text) children.push({ name: "#text", value: text });
          }
        });

        const attributes: Record<string, string> = {};
        Array.from(el.attributes).forEach((attr) => {
          attributes[attr.name] = attr.value;
        });

        return { name: el.nodeName, attributes, children };
      };

      return parseElement(xmlDoc.documentElement);
    };

    // 解析 JSON 对象为 XmlNode
    const parseJsonObject = (obj: JsonLike): XmlNode => {
      const children: XmlNode[] = [];
      for (const key in obj) {
        if (Object.prototype.hasOwnProperty.call(obj, key)) {
          const val = obj[key];
          if (typeof val === "string" && val.trim().startsWith("<")) {
            // 保留 key 作为父节点
            const childXml = parseXmlString(val);
            children.push({ name: key, children: [childXml] });
          } else if (typeof val === "object" && val !== null) {
            children.push({
              name: key,
              children: [parseJsonObject(val as JsonLike)],
            });
          } else {
            children.push({ name: key, value: String(val) });
          }
        }
      }
      return { name: "root", children };
    };

    const rootNode = computed<XmlNode>(() => {
      if (typeof props.data === "string") return parseXmlString(props.data);
      if (typeof props.data === "object" && props.data !== null)
        return parseJsonObject(props.data as JsonLike);
      return { name: "root" };
    });

    return { rootNode };
  },
});
</script>

<style scoped>
.xml-tree {
  font-family: monospace;
  background-color: var(--ds-theme-bg);
  color: var(--ds-text);
  padding: 10px;
  border-radius: 6px;
  user-select: text; /* 可选：允许复制 */
}
</style>
