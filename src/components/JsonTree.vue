<template>
  <ul class="json-tree">
    <li>
      <!-- 展示键 -->
      <span v-if="label" class="node-key">
        "{{ label }}"<span class="colon">:</span>
      </span>

      <!-- 如果是对象或数组 -->
      <template v-if="isObject(data) || isArray(data)">
        <span
          class="bracket"
          :class="{ collapsible: true, expanded }"
          @click.stop="toggle"
        >
          {{
            expanded
              ? isArray(data)
                ? "["
                : "{"
              : isArray(data)
                ? "[…]"
                : "{…}"
          }}
        </span>

        <!-- 展开时渲染子节点 -->
        <ul v-if="expanded" class="children">
          <li v-for="(entry, i) in entries" :key="entry.k">
            <JsonTree
              :data="entry.v"
              :label="!isArray(data) ? entry.k : undefined"
              :isRoot="false"
            />
            <span v-if="i < entries.length - 1" class="comma">,</span>
          </li>
        </ul>

        <!-- 收尾括号 -->
        <span class="bracket">{{ isArray(data) ? "]" : "}" }}</span>
      </template>

      <!-- 如果是基本类型 -->
      <template v-else>
        <span class="node-value">{{ formatValue(data) }}</span>
      </template>
    </li>
  </ul>
</template>

<script lang="ts">
import {
  defineComponent,
  ref,
  computed,
  onMounted,
  onBeforeUnmount,
} from "vue";
import type { PropType } from "vue";

type JsonPrimitive = string | number | boolean | null;
export type JsonValue = JsonPrimitive | JsonObject | JsonArray;
export type JsonObject = { [key: string]: JsonValue };
export type JsonArray = JsonValue[];

export default defineComponent({
  name: "JsonTree",
  props: {
    data: { type: null as unknown as PropType<JsonValue>, required: true },
    label: { type: String, required: false },
    isRoot: { type: Boolean, default: true },
  },
  setup(props) {
    const expanded = ref<boolean>(props.isRoot === true);

    const isObject = (v: JsonValue): v is JsonObject =>
      typeof v === "object" && v !== null && !Array.isArray(v);
    const isArray = (v: JsonValue): v is JsonArray => Array.isArray(v);

    const entries = computed(() => {
      const v = props.data;
      if (isArray(v)) return v.map((item, i) => ({ k: String(i), v: item }));
      if (isObject(v))
        return Object.entries(v).map(([k, val]) => ({ k, v: val }));
      return [] as { k: string; v: JsonValue }[];
    });

    function toggle() {
      if (isObject(props.data) || isArray(props.data)) {
        expanded.value = !expanded.value;
      }
    }

    function formatValue(v: JsonValue) {
      if (v === null) return "null";
      if (typeof v === "string") return `"${v}"`;
      return String(v);
    }

    /** ✅ 拦截复制，输出标准 JSON */
    function handleCopy(e: ClipboardEvent) {
      if (props.isRoot) {
        e.preventDefault();
        const json = JSON.stringify(props.data, null, 2);
        e.clipboardData?.setData("text/plain", json);
      }
    }

    onMounted(() => {
      if (props.isRoot) document.addEventListener("copy", handleCopy);
    });
    onBeforeUnmount(() => {
      if (props.isRoot) document.removeEventListener("copy", handleCopy);
    });

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

.json-tree > li {
  position: relative;
}

.json-tree li::marker {
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
.comma {
  color: var(--ds-text-light);
}
.bracket {
  cursor: pointer;
  color: var(--ds-text);
}

.bracket.collapsible::before {
  content: "▶";
  display: inline-block;
  margin-right: 2px;
  color: var(--ds-text-lighter);
  font-size: 12px;
  transition: transform 0.2s;
}
.bracket.expanded.collapsible::before {
  transform: rotate(90deg);
  color: var(--ds-text-light);
}

.children {
  margin: 2px 0 2px 14px;
  padding-left: 12px;
  border-left: 1px dashed var(--ds-border-color);
}
</style>
