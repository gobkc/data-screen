<template>
  <div class="app-container">
    <!--topbar-->
    <header class="top-bar">
      <img
        alt="DS Logo"
        class="logo"
        src="@/assets/logo.svg"
        width="30"
        height="30"
      />
    </header>

    <div class="main-container" ref="mainContainer">
      <!-- left -->
      <div class="sidebar left" :style="{ width: leftWidth + 'px' }">
        <select class="dark-select">
          <option>选项一</option>
          <option>选项二</option>
          <option>选项三</option>
          <option>选项四</option>
        </select>

        <button class="ds-btn">
          <svg viewBox="0 0 24 24">
            <path
              d="M15.5 14h-.79l-.28-.27a6.471 6.471 0 001.48-5.34C15.36 5.61 12.28 3 8.5 3S1.64 5.61 1.64 9.39 4.72 15.78 8.5 15.78a6.471 6.471 0 005.34-1.48l.27.28v.79l4.25 4.25c.39.39 1.02.39 1.41 0s.39-1.02 0-1.41L15.5 14zm-7 0C6.01 14 4 11.99 4 9.5S6.01 5 8.5 5 13 7.01 13 9.5 10.99 14 8.5 14z"
            />
          </svg>
          搜索
        </button>

        <button class="ds-btn">
          <svg viewBox="0 0 24 24">
            <path d="M10 18h4v-2h-4v2zm-7-7v2h18v-2H3zm3-7v2h12V4H6z" />
          </svg>
          筛选
        </button>

        <div style="display: flex; gap: 4px">
          <button class="ds-btn ds-btn-small">
            <svg viewBox="0 0 24 24">
              <path d="M15.41 7.41L14 6l-6 6 6 6 1.41-1.41L10.83 12z" />
            </svg>
          </button>
          <button class="ds-btn ds-btn-small">
            <svg viewBox="0 0 24 24">
              <path d="M8.59 16.59L13.17 12 8.59 7.41 10 6l6 6-6 6z" />
            </svg>
          </button>
        </div>

        <div class="ds-pagination">
          <!-- 第一页 -->
          <button title="第一页">
            <svg viewBox="0 0 24 24">
              <path d="M18 6H16L10 12l6 6h2V6zM8 6H6v12h2V6z" />
            </svg>
          </button>

          <!-- 上一页 -->
          <button title="上一页">
            <svg viewBox="0 0 24 24">
              <path d="M15.41 16.59L10.83 12l4.58-4.59L14 6l-6 6 6 6z" />
            </svg>
          </button>

          <!-- 页码输入 -->
          <input type="number" min="1" value="1" title="输入页码" />

          <!-- 下一页 -->
          <button title="下一页">
            <svg viewBox="0 0 24 24">
              <path d="M8.59 16.59L13.17 12 8.59 7.41 10 6l6 6-6 6z" />
            </svg>
          </button>

          <!-- 最后一页 -->
          <button title="最后一页">
            <svg viewBox="0 0 24 24">
              <path d="M6 6h2l6 6-6 6H6V6zM16 6h2v12h-2V6z" />
            </svg>
          </button>

          <!-- 每页数量选择 -->
          <select
            title="选择每页数量"
            style="
              background-color: var(--ds-row-highlight);
              color: var(--ds-text-dark);
              border-radius: 4px;
              border: 1px solid var(--ds-border-color);
              padding: 4px 6px;
            "
          >
            <option value="10">10 / 页</option>
            <option value="20">20 / 页</option>
            <option value="50">50 / 页</option>
            <option value="100">100 / 页</option>
          </select>
        </div>

        <button class="ds-icon-btn" title="添加">
          <svg viewBox="0 0 24 24">
            <path d="M19 11h-6V5h-2v6H5v2h6v6h2v-6h6z" />
          </svg>
        </button>

        <button class="ds-icon-btn" title="刷新">
          <svg viewBox="0 0 24 24">
            <path
              d="M17.65 6.35A7.95 7.95 0 0012 4V1l-5 5 5 5V6c2.76 0 5.26 1.12 7.07 2.93l1.41-1.41zM6.35 17.65A7.95 7.95 0 0012 20v3l5-5-5-5v3c-2.76 0-5.26-1.12-7.07-2.93L6.35 17.65z"
            />
          </svg>
        </button>

        <button class="ds-icon-btn" title="删除">
          <svg viewBox="0 0 24 24">
            <path
              d="M6 19c0 1.1.9 2 2 2h8c1.1 0 2-.9 2-2V7H6v12zM19 4h-3.5l-1-1h-5l-1 1H5v2h14V4z"
            />
          </svg>
        </button>

        <button class="ds-icon-btn" title="编辑">
          <svg viewBox="0 0 24 24">
            <path
              d="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04a1.003 1.003 0 0 0 0-1.41l-2.34-2.34a1.003 1.003 0 0 0-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z"
            />
          </svg>
        </button>

        <ul class="ds-tree">
          <li class="expanded">
            <div class="node">
              <span class="toggle-btn">
                <svg viewBox="0 0 24 24">
                  <path d="M7 10l5 5 5-5H7z" />
                </svg>
              </span>
              根节点
            </div>
            <ul>
              <li>
                <div class="node">
                  <span class="toggle-btn">
                    <svg viewBox="0 0 24 24">
                      <path d="M10 17l5-5-5-5v10z" />
                    </svg>
                  </span>
                  子节点 1
                </div>
              </li>
              <li class="expanded">
                <div class="node">
                  <span class="toggle-btn">
                    <svg viewBox="0 0 24 24">
                      <path d="M7 10l5 5 5-5H7z" />
                    </svg>
                  </span>
                  子节点 2
                </div>
                <ul>
                  <li>
                    <div class="node">孙子节点 2-1</div>
                  </li>
                  <li>
                    <div class="node">孙子节点 2-2</div>
                  </li>
                </ul>
              </li>
            </ul>
          </li>
        </ul>
      </div>

      <!-- splitter -->
      <div class="splitter" @mousedown="startDrag"></div>

      <!-- right -->
      <div class="sidebar right">
        <h2>Right Panel</h2>
        <XmlTree :data="xmlData" />

        <JsonTree :data="jsonData" />
        <CodeEditor v-model="code" width="100%" height="100%" />
        <button @click="addRow">新增空行</button>
        <SimpleGridTable
          v-model="items"
          @update="onUpdate"
          v-on:onSort="sortColumn"
          @onSelectRow="handleSelectRow"
          @onDeleteRow="handleDeleteRow"
          @onUnDelete="handleUnDeleteRow"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
// import { onMounted, onBeforeUnmount } from "vue";
import JsonTree from "./components/JsonTree.vue";
import type { JsonValue } from "./components/JsonTree.vue";
import SimpleGridTable from "./components/SimpleGridTable.vue";
import CodeEditor from "./components/CodeEditor.vue";
import XmlTree from "./components/XmlTree.vue";

import { ref } from "vue";

const code = ref<string>(
  'console.log("Hello World");\nfunction test() {\n  return 42;\n}',
);

const jsonData: JsonValue = {
  id: 1,
  name: "Test Company",
  active: true,
  employees: [
    {
      id: 101,
      name: "Alice",
      role: "Developer",
      aaa: "Developer",
      bbb: `12321312`,
      ccc: `fdsafsdafa`,
      ddd: `dfsafds`,
      eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee: `dfsfdsfas`,
    },
    { id: 102, name: "Bob", role: "Designer" },
  ],
  address: { city: "Singapore", postal: 123456 },
} satisfies JsonValue;

const items = ref([
  {
    id: 101,
    name: "Alice",
    role: "Developer",
    aaa: "Developer",
    bbb: `12321312`,
    ccc: `fdsafsdafa`,
    ddd: `dfsafds`,
    eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee: `dfsfdsfas`,
  } as Record<string, unknown>,
  { id: 2, name: "Test Company2", active: false } as Record<string, unknown>,
  { id: 2, name: "Test Company2", active: false } as Record<string, unknown>,
]);

interface UpdatePayload {
  row: number;
  key: string;
  value: unknown;
}

const onUpdate = (payload: UpdatePayload) => {
  console.log("Cell updated:", payload, items);
};

function handleSelectRow(rowData: Record<string, unknown>) {
  console.log("选中的行数据:", rowData);
}
function handleDeleteRow(rowData: Record<string, unknown>[]) {
  console.log("delete rows:", rowData);
}
function handleUnDeleteRow(rowData: Record<string, unknown>[]) {
  console.log("undelete rows:", rowData);
}
function sortColumn(key: string, sort: string) {
  console.log("sord:", key, sort);
}
// 新增一行空数据（所有 key 清空）
function addRow() {
  if (items.value.length > 0) {
    const keys = Object.keys(items.value[0]);
    const empty: Record<string, unknown> = {};
    keys.forEach((k) => (empty[k] = ""));
    items.value.push(empty);
  }
}

const leftWidth = ref(300);
let isDragging = false;
let startX = 0;
let startWidth = 0;

const startDrag = (e: MouseEvent) => {
  isDragging = true;
  startX = e.clientX;
  startWidth = leftWidth.value;
  document.addEventListener("mousemove", onDrag);
  document.addEventListener("mouseup", stopDrag);
};

const onDrag = (e: MouseEvent) => {
  if (!isDragging) return;
  const delta = e.clientX - startX;
  leftWidth.value = Math.max(100, startWidth + delta); // 最小宽度 100px
};

const stopDrag = () => {
  isDragging = false;
  document.removeEventListener("mousemove", onDrag);
  document.removeEventListener("mouseup", stopDrag);
};

const xmlData = {
  aaa: {
    ccc: true,
    ddd: `<note>
            <to>张三</to>
            <from>李四</from>
            <heading>提醒</heading>
            <body>别忘了下午三点的会议！</body>
          </note>`,
  },
};
</script>

<style scoped></style>
