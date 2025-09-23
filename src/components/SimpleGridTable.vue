<template>
  <div class="table-wrapper">
    <table class="grid-table" @keydown="onKeyDown">
      <thead>
        <tr>
          <th class="row-number"></th>
          <th
            v-for="(key, kIdx) in headers"
            :key="kIdx"
            @mousedown.prevent="selectColumn(kIdx)"
            class="header-cell"
          >
            {{ key }}
            <span
              class="sort-buttons"
              @click.stop="sortColumn(key)"
              @mousedown.stop
            >
              <svg
                class="sort-arrow up"
                width="12"
                height="12"
                viewBox="0 0 24 24"
                fill="currentColor"
                style="cursor: pointer; margin-left: 2px"
              >
                <path d="M7 14l5-5 5 5H7z" />
              </svg>
              <svg
                class="sort-arrow down"
                width="12"
                height="12"
                viewBox="0 0 24 24"
                fill="currentColor"
                style="cursor: pointer; margin-left: 2px"
              >
                <path d="M7 10l5 5 5-5H7z" />
              </svg>
            </span>
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(row, rIdx) in local" :key="rIdx">
          <td class="row-number" @mousedown.prevent="selectRow(rIdx)">
            {{ rIdx + 1 }}
          </td>
          <td
            v-for="(key, cIdx) in headers"
            :key="cIdx"
            spellcheck="false"
            :class="cellClass(rIdx, key)"
            :contenteditable="true"
            @mousedown="startSelection(rIdx, key, $event)"
            @mouseover="extendSelection(rIdx, key, $event)"
            @mouseup="endSelection"
            @input="onInput($event, rIdx, key)"
            @blur="onCellBlur($event, rIdx, key)"
          >
            {{ row[key] }}
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, computed } from "vue";

const props = withDefaults(
  defineProps<{
    modelValue?: Record<string, unknown>[];
  }>(),
  { modelValue: () => [] as Record<string, unknown>[] },
);

const emit = defineEmits<{
  (e: "update:modelValue", value: Record<string, unknown>[]): void;
  (e: "update", value: { row: number; key: string; value: unknown }): void;
  (e: "onSelectRow", rowData: Record<string, unknown>): void;
  (e: "onDeleteRow", rowData: Record<string, unknown>[]): void;
  (e: "onUnDelete", rowData: Record<string, unknown>[]): void;
  (e: "onSort", column: string, order: "asc" | "desc"): void;
}>();

const local = ref<Record<string, unknown>[]>([...props.modelValue]);
const initialValue = ref<Record<string, unknown>[]>([...props.modelValue]);

watch(
  () => props.modelValue,
  (v) => {
    local.value = [...v];
    if (initialValue.value.length === 0) {
      initialValue.value = [...v]; // 只初始化一次
    }
  },
  { deep: true },
);

const headers = computed(() =>
  local.value.length > 0 ? Object.keys(local.value[0]) : [],
);

function getText(e: Event) {
  return (e.target as HTMLElement)?.innerText ?? "";
}

function onInput(e: Event, row: number, key: string) {
  const txt = getText(e);

  // 如果当前行已删除，不更新 local 的值
  if (isRowDeleted(row)) return;

  const next = local.value.map((r, i) =>
    i === row ? { ...r, [key]: txt } : r,
  );
  local.value = next;
  emit("update:modelValue", next);
  emit("update", { row, key, value: txt });

  // 同样处理 updatedRows
  const original = initialValue.value[row]?.[key];
  if (txt !== original) updatedRows.value.add(row);
  else {
    const rowKeys = Object.keys(next[row]);
    const hasOtherChanges = rowKeys.some(
      (k) => next[row][k] !== initialValue.value[row]?.[k],
    );
    if (!hasOtherChanges) updatedRows.value.delete(row);
  }
}

function onCellBlur(e: FocusEvent, row: number, key: string) {
  if (!isRowDeleted(row)) return;

  const cell = e.target as HTMLElement;
  const original = initialValue.value[row]?.[key] ?? "";
  // 还原单元格内容为原始值
  cell.innerText = String(local.value[row]?.[key] ?? original);
}

function cellKey(r: number, k: string) {
  return `${r}-${k}`;
}

const selection = ref<Set<string>>(new Set());
const isSelecting = ref(false);
const startCell = ref<{ row: number; col: number } | null>(null);
const copyBuffer = ref<string[][]>([]);

function cellClass(r: number, k: string) {
  const classes = [];
  if (selection.value.has(cellKey(r, k))) classes.push("selected");
  if (isRowDeleted(r)) classes.push("deleted-row");
  else if (updatedRows.value.has(r)) classes.push("updated-row");
  return classes.join(" ");
}

function startSelection(r: number, k: string, e: MouseEvent) {
  if (e.button !== 0) return;
  selection.value.clear();
  const col = headers.value.indexOf(k);
  startCell.value = { row: r, col };
  selection.value.add(cellKey(r, k));
  isSelecting.value = true;
  emit("onSelectRow", local.value[r]);
}

function extendSelection(r: number, k: string, e: MouseEvent) {
  e.stopPropagation();
  if (!isSelecting.value || !startCell.value) return;
  const col = headers.value.indexOf(k);
  const minRow = Math.min(startCell.value.row, r);
  const maxRow = Math.max(startCell.value.row, r);
  const minCol = Math.min(startCell.value.col, col);
  const maxCol = Math.max(startCell.value.col, col);
  selection.value.clear();
  for (let rr = minRow; rr <= maxRow; rr++) {
    for (let cc = minCol; cc <= maxCol; cc++) {
      selection.value.add(cellKey(rr, headers.value[cc]));
    }
  }
  emit("onSelectRow", local.value[r]);
}

function endSelection() {
  isSelecting.value = false;
}

function selectRow(r: number) {
  selection.value.clear();
  headers.value.forEach((key) => selection.value.add(cellKey(r, key)));
  emit("onSelectRow", local.value[r]);
}

function selectColumn(cIdx: number) {
  selection.value.clear();
  local.value.forEach((_, r) => {
    selection.value.add(cellKey(r, headers.value[cIdx]));
  });
}

const sortStates = ref<Record<string, "asc" | "desc" | "">>({});

type SortOrder = "asc" | "desc";

function sortColumn(key: string) {
  const current = sortStates.value[key] ?? "";
  const next: "" | SortOrder =
    current === "" ? "asc" : current === "asc" ? "desc" : "";
  sortStates.value[key] = next;
  emit("onSort", key, next as SortOrder);
}

// 记录被标记删除的行
const deletedRows = ref<Set<number>>(new Set());

// 删除边框行类
function isRowDeleted(r: number) {
  return deletedRows.value.has(r);
}

// 标记行删除
function markRowDeleted(rows: number[]) {
  rows.forEach((r) => deletedRows.value.add(r));
}

// 撤销删除
function undeleteRows(rows?: number[]) {
  const toUndelete = rows ?? Array.from(deletedRows.value);
  deletedRows.value = new Set(
    [...deletedRows.value].filter((r) => !toUndelete.includes(r)),
  ); // 这里用新的 Set 替换，确保响应式
  emit(
    "onUnDelete",
    toUndelete.map((r) => local.value[r]),
  );
}

const updatedRows = ref<Set<number>>(new Set());
function unmarkUpdatedRows(rows?: number[]) {
  const toUnmark = rows ?? Array.from(updatedRows.value);
  toUnmark.forEach((r) => updatedRows.value.delete(r));
}

defineExpose({
  undeleteRows,
  unmarkUpdatedRows,
});

// 只在表格内部处理 Ctrl+C / Ctrl+V
function onKeyDown(e: KeyboardEvent) {
  // 删除
  if (e.key === "Delete" && !e.shiftKey) {
    e.preventDefault();
    const rows = Array.from(selection.value).map((ck) =>
      parseInt(ck.split("-")[0]),
    );
    const uniqueRows = [...new Set(rows)];
    markRowDeleted(uniqueRows);
    emit(
      "onDeleteRow",
      uniqueRows
        .map((r) => local.value[r])
        .filter((row): row is Record<string, unknown> => row !== undefined),
    );
  }

  if (e.key === "Delete" && e.shiftKey) {
    e.preventDefault();
    const rows = Array.from(selection.value).map((ck) =>
      parseInt(ck.split("-")[0]),
    );
    const uniqueRows = [...new Set(rows)];
    // 用 undeleteRows 替代 unmarkRowDeleted
    undeleteRows(uniqueRows);
  }

  // 复制
  if (e.ctrlKey && e.key.toLowerCase() === "c") {
    e.preventDefault();
    if (selection.value.size === 0) return;
    const coords = Array.from(selection.value).map((ck) => {
      const [r, k] = ck.split("-");
      return { row: parseInt(r), col: headers.value.indexOf(k), key: k };
    });
    const minRow = Math.min(...coords.map((c) => c.row));
    const maxRow = Math.max(...coords.map((c) => c.row));
    const minCol = Math.min(...coords.map((c) => c.col));
    const maxCol = Math.max(...coords.map((c) => c.col));
    const matrix: string[][] = [];
    for (let r = minRow; r <= maxRow; r++) {
      const rowVals: string[] = [];
      for (let c = minCol; c <= maxCol; c++) {
        rowVals.push(String(local.value[r]?.[headers.value[c]] ?? ""));
      }
      matrix.push(rowVals);
    }
    copyBuffer.value = matrix;
  }

  // 粘贴
  if (e.ctrlKey && e.key.toLowerCase() === "v") {
    e.preventDefault();
    if (copyBuffer.value.length === 0) return;
    const first = selection.value.values().next().value;
    if (!first) return;
    const [rowStr, key] = first.split("-");
    const startRow = parseInt(rowStr);
    const startCol = headers.value.indexOf(key);
    if (startCol < 0) return;
    const next = [...local.value];
    copyBuffer.value.forEach((rowVals, rOffset) => {
      const targetRow = startRow + rOffset;
      if (!next[targetRow]) return;
      rowVals.forEach((val, cOffset) => {
        const targetCol = startCol + cOffset;
        const targetKey = headers.value[targetCol];
        if (targetKey)
          next[targetRow] = { ...next[targetRow], [targetKey]: val };
      });
    });
    local.value = next;
    emit("update:modelValue", next);
  }
}
</script>

<style scoped>
.table-wrapper {
  width: 100%;
  overflow: auto;
}

/* 滚动条整体 */
.table-wrapper::-webkit-scrollbar {
  width: 8px;
  height: 12px;
}

/* 滚动条轨道 */
.table-wrapper::-webkit-scrollbar-track {
  background: var(--ds-text-editor-bg-color);
  border-radius: 4px;
}

/* 滚动条滑块 */
.table-wrapper::-webkit-scrollbar-thumb {
  background: var(--ds-statusbar-bg);
  border-radius: 0px;
}

/* 鼠标悬停时 */
.table-wrapper::-webkit-scrollbar-thumb:hover {
  background: var(--ds-statusbar-bg);
}

.grid-table {
  width: 100%;
  border-collapse: collapse;
  border-spacing: 0;
  background: var(--ds-theme-bg);
  color: var(--ds-text-light);
  font-size: 13px;
  border: none;
}

.grid-table tbody tr:last-child td:focus {
  border-bottom: none;
}

.grid-table th:not(.row-number),
.grid-table td:not(.row-number) {
  border: none;
  padding: 4px 8px;
  text-align: left;
  min-width: 80px;
}

.grid-table th {
  background: var(--ds-text-editor-bg-color);
  color: var(--ds-text-dark);
  font-weight: 500;
  user-select: none;
}

.row-number {
  min-width: 30px;
  width: fit-content;
  white-space: nowrap;
  text-align: center;
  background: var(--ds-text-editor-bg-color) !important;
  color: var(--ds-input-highlight);
  font-weight: 400;
}

.grid-table td {
  box-sizing: border-box !important;
  background: var(--ds-text-editor-bg-color);
  transition:
    background 0.1s ease,
    color 0.1s ease,
    border 0.1s ease;
  cursor: text;
  white-space: nowrap; /* 不换行 */
  overflow: hidden; /* 隐藏超出内容 */
  text-overflow: ellipsis; /* 用省略号显示 */
}

.grid-table td:focus {
  outline: none;
  background: var(--ds-text-editor-bg-color);
  color: var(--ds-text);
}

.deleted-row {
  background-color: var(--ds-row-delete) !important;
}

.updated-row {
  background-color: var(--ds-row-add) !important;
}

.selected {
  background: var(--ds-selected-bg) !important;
  color: var(--ds-text-dark);
}

/* 内容区域整体边框修正 */
/* 先重置行号列，保持无边框 */
/* 冻结第一列（行号列） */
.grid-table td.row-number,
.grid-table th.row-number {
  position: sticky;
  left: 0;
  z-index: 2; /* 确保行号在内容列上方 */
}
.grid-table tbody td.row-number {
  border: none;
}

.grid-table tbody tr:nth-child(odd) td {
  background: var(--ds-theme-bg);
}

/* 最后一列内容区加右边框（矩形闭合） */
.grid-table tbody tr td:not(.row-number):last-child {
  border-right: 1px solid var(--ds-border-color);
}

.header-cell {
  position: relative;
  padding-right: 24px; /* 为箭头留空间 */
}

.header-cell svg.sort-arrow {
  width: 22px; /* 放大箭头 */
  height: 22px;
  position: absolute;
  right: 4px; /* 靠右 */
  cursor: pointer;
  fill: var(--ds-text-lighter);
}

.header-cell svg.sort-arrow.up {
  top: 0px; /* 上箭头在上方 */
}

.header-cell svg.sort-arrow.down {
  bottom: 0px; /* 下箭头在下方 */
}
</style>
