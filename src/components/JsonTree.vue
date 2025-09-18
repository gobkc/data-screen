<template>
  <span>
    <span v-if="label" class="node-key">
      "{{ label }}"<span class="colon">:</span>
    </span>

    <template v-if="isObject(data) || isArray(data)">
      <span class="bracket collapsible" :class="{ expanded }" @click="toggle">
        {{
          expanded ? (isArray(data) ? "[" : "{") : isArray(data) ? "[…]" : "{…}"
        }}
      </span>

      <ul v-if="expanded" class="children">
        <li v-for="(entry, i) in entries" :key="entry.k">
          <JsonTree
            :data="entry.v"
            :label="!isArray(data) ? entry.k : undefined"
            :showComma="i < entries.length - 1"
          />
        </li>
      </ul>

      <span class="bracket">
        {{ isArray(data) ? "]" : "}" }}<span v-if="showComma">,</span>
      </span>
    </template>

    <template v-else>
      <span class="node-value">
        {{ formatValue(data) }}<span v-if="showComma">,</span>
      </span>
    </template>
  </span>
</template>

<script lang="ts">
import { defineComponent, ref, computed } from "vue";

/**
 * 定义 JSON 类型
 */
type JsonPrimitive = string | number | boolean | null;
export type JsonValue =
  | JsonPrimitive
  | JsonValue[]
  | { [key: string]: JsonValue };

interface Entry {
  k: string;
  v: JsonValue;
}

export default defineComponent({
  name: "JsonTree",
  props: {
    data: { type: null as unknown as () => JsonValue, required: true },
    label: String,
    showComma: { type: Boolean, default: false },
  },
  setup(props) {
    const expanded = ref(true);

    /** 判断是否是对象 */
    const isObject = (v: JsonValue): v is { [key: string]: JsonValue } =>
      v !== null && typeof v === "object" && !Array.isArray(v);

    /** 判断是否是数组 */
    const isArray = Array.isArray as (v: JsonValue) => v is JsonValue[];

    /** 生成 entries，方便 v-for 渲染 */
    const entries = computed<Entry[]>(() => {
      if (isArray(props.data)) {
        return props.data.map((v, i) => ({ k: String(i), v }));
      }
      if (isObject(props.data)) {
        return Object.entries(props.data).map(([k, v]) => ({ k, v }));
      }
      return [];
    });

    /** 折叠/展开 */
    const toggle = () => {
      expanded.value = !expanded.value;
    };

    /** 格式化基本值 */
    const formatValue = (v: JsonValue): string => {
      if (v === null) return "null";
      if (typeof v === "string") return `"${v}"`;
      return String(v);
    };

    return { expanded, isObject, isArray, entries, toggle, formatValue };
  },
});
</script>

<style scoped>
.json-tree {
  list-style: none;
  margin: 0;
  padding-left: 20px;
  font-family: monospace;
  font-size: 13px;
  color: var(--ds-text-dark);
  background: var(--ds-theme-bg);
  user-select: text;
}
.json-tree li::marker,
.children li::marker {
  content: "";
}

.node-key {
  color: var(--ds-theme-secondary);
  font-weight: 600;
}
.colon {
  color: var(--ds-text-light);
  margin-left: 2px;
}
.node-value {
  color: var(--ds-theme-primary);
  margin-left: 4px;
}

.bracket {
  cursor: pointer;
  color: var(--ds-text);
}

.bracket.collapsible::before {
  content: "";
  display: inline-block;
  width: 12px;
  height: 12px;
  margin-right: 2px;

  /* 用 mask 绘制空心箭头 */
  -webkit-mask: no-repeat center / contain;
  -webkit-mask-image: url("data:image/svg+xml;utf8,\
  <svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 12 12'>\
    <polyline points='4,2 8,6 4,10' fill='none' stroke='black' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'/>\
  </svg>");
  mask: no-repeat center / contain;
  mask-image: url("data:image/svg+xml;utf8,\
  <svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 12 12'>\
    <polyline points='4,2 8,6 4,10' fill='none' stroke='black' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'/>\
  </svg>");

  background-color: var(--ds-text-lighter); /* 继承文本颜色 */
  transition: transform 0.2s;
}
.bracket.expanded.collapsible::before {
  transform: rotate(90deg);
  color: var(--ds-text-light);
}

.children {
  margin: 2px 0 2px 14px;
  padding-left: 12px;
}

.children li {
  position: relative;
}

.children li .bracket.collapsible::before {
  position: absolute;
  left: -15px;
  top: 5px;
}

.json-item {
  display: inline;
}
.comma {
  color: var(--ds-text-light);
  margin-left: 0;
}
</style>
