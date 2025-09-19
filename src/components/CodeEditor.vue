<template>
  <div class="editor-wrapper" :style="{ width, height }">
    <div class="editor-gutter">
      <pre>{{ lineNumbers }}</pre>
    </div>

    <div class="editor-main">
      <!-- 编辑层 -->
      <div
        ref="editor"
        class="editor-content"
        contenteditable="true"
        spellcheck="false"
        @input="onInput"
        @keydown="onKeyDown"
        @paste="onPaste"
        @scroll="syncScroll"
      ></div>

      <!-- 提示框 -->
      <ul
        v-if="showSuggestions"
        class="suggestion-box"
        :style="{
          top: suggestionPos.top + 'px',
          left: suggestionPos.left + 'px',
        }"
      >
        <li
          v-for="(s, i) in suggestions"
          :key="i"
          :class="{ active: i === suggestionIndex }"
        >
          {{ s }}
        </li>
      </ul>
    </div>
  </div>
</template>

<script lang="ts">
import {
  defineComponent,
  ref,
  computed,
  watch,
  nextTick,
  onMounted,
} from "vue";

export default defineComponent({
  name: "CodeEditor",
  props: {
    modelValue: { type: String, required: true },
    width: { type: String, default: "100%" },
    height: { type: String, default: "100%" },
  },
  emits: ["update:modelValue"],
  setup(props, { emit }) {
    const editor = ref<HTMLDivElement | null>(null);
    const lines = ref<string[]>(props.modelValue.split("\n"));

    /** -------------------- 行号 -------------------- */
    const lineNumbers = computed(() =>
      Array.from({ length: lines.value.length }, (_, i) =>
        (i + 1).toString(),
      ).join("\n"),
    );

    /** -------------------- 高亮 -------------------- */
    const escapeHtml = (str: string): string =>
      str.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");

    const highlightLine = (line: string): string => {
      if (line === "") return "&nbsp;";
      let html = escapeHtml(line);
      // 字符串
      html = html.replace(/(".*?"|'.*?')/g, '<span class="string">$1</span>');
      // 数字
      html = html.replace(/\b(\d+)\b/g, '<span class="number">$1</span>');
      // 关键字（大小写不敏感）
      html = html.replace(
        /\b(function|return|const|let|var|if|else|select|set|where|insert|update|delete)\b/gi,
        '<span class="keyword">$1</span>',
      );
      return html;
    };

    /** -------------------- 滚动同步 -------------------- */
    const syncScroll = (): void => {
      if (!editor.value) return;
    };

    /** -------------------- 行数据同步 -------------------- */
    const syncLinesFromEditor = (): void => {
      if (!editor.value) return;
      const newLines: string[] = [];
      editor.value.childNodes.forEach((node) => {
        if (node.nodeType === Node.ELEMENT_NODE) {
          const div = node as HTMLElement;
          if (div.innerText === "" || div.innerHTML === "<br>") {
            newLines.push("");
          } else {
            newLines.push(div.innerText);
          }
        }
      });
      lines.value = newLines;
    };

    /** -------------------- 输入事件 -------------------- */
    const onInput = (): void => {
      syncLinesFromEditor();
      emit("update:modelValue", lines.value.join("\n"));
      updateSuggestions();
    };

    /** -------------------- Tab 缩进 -------------------- */
    const onKeyDown = (e: KeyboardEvent): void => {
      if (!editor.value) return;

      // Tab 缩进
      if (e.key === "Tab") {
        e.preventDefault();
        const selection = window.getSelection();
        if (!selection || selection.rangeCount === 0) return;
        const range = selection.getRangeAt(0);

        const allDivs = Array.from(
          editor.value.querySelectorAll("div"),
        ) as HTMLDivElement[];
        const editorDivs = allDivs.filter(
          (d) => d.querySelector("div") === null,
        );

        const selectedLines: HTMLDivElement[] = [];
        editorDivs.forEach((div) => {
          const divRange = document.createRange();
          divRange.selectNodeContents(div);
          const intersects =
            range.compareBoundaryPoints(Range.END_TO_START, divRange) < 0 &&
            range.compareBoundaryPoints(Range.START_TO_END, divRange) > 0;
          if (intersects) selectedLines.push(div);
        });

        if (selectedLines.length === 0) {
          let node: Node | null = range.startContainer;
          let el: HTMLElement | null =
            node.nodeType === Node.ELEMENT_NODE
              ? (node as HTMLElement)
              : node.parentElement;
          while (el && el !== editor.value) {
            if (
              el.tagName === "DIV" &&
              el.querySelector("div") === null &&
              editor.value.contains(el)
            ) {
              selectedLines.push(el as HTMLDivElement);
              break;
            }
            el = el.parentElement;
          }
        }

        if (selectedLines.length === 0) return;

        selectedLines.forEach((div) => {
          const text = div.innerText;
          const newText = e.shiftKey
            ? text.startsWith("  ")
              ? text.slice(2)
              : text
            : "  " + text;
          div.textContent = newText;
          div.innerHTML = highlightLine(newText);
        });

        selection.removeAllRanges();
        const newRange = document.createRange();
        const first = selectedLines[0];
        const last = selectedLines[selectedLines.length - 1];
        const startNode = first.firstChild || first;
        const endNode = last.firstChild || last;
        const endOffset =
          endNode.nodeType === Node.TEXT_NODE
            ? (endNode.textContent?.length ?? 0)
            : last.childNodes.length;

        try {
          newRange.setStart(startNode as Node, 0);
          newRange.setEnd(endNode as Node, endOffset);
        } catch {
          newRange.selectNodeContents(last);
          newRange.collapse(false);
        }
        selection.addRange(newRange);

        syncLinesFromEditor();
        emit("update:modelValue", lines.value.join("\n"));
      }

      // 输入空格时隐藏提示
      if (e.key === " ") {
        hideSuggestions();
      }
    };

    /** -------------------- 粘贴 -------------------- */
    const onPaste = (e: ClipboardEvent): void => {
      if (!editor.value) return;
      e.preventDefault();

      const clipboard: DataTransfer | null = e.clipboardData;
      const rawText = clipboard?.getData("text/plain") ?? "";
      if (!rawText) return;

      const linesToInsert = rawText.split(/\r?\n/);
      const htmlParts = linesToInsert.map(
        (ln) => `<div>${highlightLine(ln)}</div>`,
      );
      const htmlToInsert = htmlParts.join("");

      let insertedWithExec = false;
      try {
        insertedWithExec = document.execCommand(
          "insertHTML",
          false,
          htmlToInsert,
        );
      } catch {
        insertedWithExec = false;
      }

      if (!insertedWithExec) {
        const selection = window.getSelection();
        if (!selection || selection.rangeCount === 0) {
          const frag = document.createDocumentFragment();
          linesToInsert.forEach((ln) => {
            const div = document.createElement("div");
            div.innerHTML = highlightLine(ln);
            frag.appendChild(div);
          });
          editor.value.appendChild(frag);
        } else {
          const range = selection.getRangeAt(0);
          range.deleteContents();
          const frag = document.createDocumentFragment();
          linesToInsert.forEach((ln) => {
            const div = document.createElement("div");
            div.innerHTML = highlightLine(ln);
            frag.appendChild(div);
          });
          range.insertNode(frag);
        }
      }

      setTimeout(() => {
        syncLinesFromEditor();
        emit("update:modelValue", lines.value.join("\n"));
      }, 0);
    };

    /** -------------------- 渲染 -------------------- */
    const renderEditor = (): void => {
      if (!editor.value) return;
      editor.value.innerHTML = "";
      lines.value.forEach((line) => {
        const div = document.createElement("div");
        div.innerHTML = highlightLine(line);
        editor.value?.appendChild(div);
      });
    };

    watch(
      () => props.modelValue,
      (newVal) => {
        if (newVal !== lines.value.join("\n")) {
          lines.value = newVal.split("\n");
          nextTick(() => renderEditor());
        }
      },
    );

    onMounted(() => {
      renderEditor();
    });

    /** -------------------- 提示词功能 -------------------- */
    const allKeywords = [
      "SELECT",
      "SET",
      "UPDATE",
      "DELETE",
      "INSERT",
      "WHERE",
      "FROM",
      "FUNCTION",
      "RETURN",
      "CONST",
      "LET",
      "VAR",
      "IF",
      "ELSE",
    ];
    const showSuggestions = ref(false);
    const suggestions = ref<string[]>([]);
    const suggestionIndex = ref(0);
    const suggestionPos = ref({ top: 0, left: 0 });

    const updateSuggestions = (): void => {
      const sel = window.getSelection();
      if (!sel || !editor.value) return;

      const textBefore =
        sel.anchorNode?.textContent?.slice(0, sel.anchorOffset) ?? "";
      const currentWord = textBefore.split(/\s+/).pop() ?? "";

      if (!currentWord) {
        hideSuggestions();
        return;
      }

      const matches = allKeywords.filter((kw) =>
        kw.startsWith(currentWord.toUpperCase()),
      );
      if (matches.length > 0) {
        suggestions.value = matches;
        suggestionIndex.value = 0;
        showSuggestions.value = true;

        const range = sel.getRangeAt(0).cloneRange();
        const rect = range.getBoundingClientRect();
        const editorRect = editor.value.getBoundingClientRect();
        suggestionPos.value = {
          top: rect.bottom - editorRect.top,
          left: rect.left - editorRect.left,
        };
      } else {
        hideSuggestions();
      }
    };

    const hideSuggestions = (): void => {
      showSuggestions.value = false;
      suggestions.value = [];
    };

    return {
      editor,
      lines,
      lineNumbers,
      onInput,
      onKeyDown,
      onPaste,
      syncScroll,
      // suggestions
      showSuggestions,
      suggestions,
      suggestionIndex,
      suggestionPos,
    };
  },
});
</script>

<style>
.editor-wrapper {
  display: flex;
  font-family: monospace;
  background-color: var(--ds-text-editor-bg-color);
  color: var(--ds-text-editor-fg-color);
  border: 1px solid var(--ds-border-color);
  height: 100%;
  overflow: hidden;
  padding: 12px 8px;
}

.editor-gutter {
  padding: 0px 12px 4px 8px;
  user-select: none;
  text-align: right;
  color: var(--ds-text-editor-linenumber-fg-color);
  background-color: var(--ds-text-editor-gutter-bg-color);
}

.editor-gutter pre {
  margin: 0;
}

.editor-main {
  position: relative;
  flex: 1;
  overflow: hidden;
  font-family: monospace;
  font-size: 14px;
  line-height: 1.5;
  padding: 0px;
}

.editor-content {
  position: relative;
  z-index: 1;
  overflow: auto;
  width: 100%;
  height: 100%;
  white-space: pre-wrap;
  word-wrap: break-word;
  outline: none;
  background: transparent;
  caret-color: var(--ds-text-editor-cursor-bg-color);
  font-family: monospace;
  font-size: 14px;
  line-height: 1.5;
  margin: 0;
  padding: 0 8px;
  tab-size: 2;
}

.editor-content .keyword {
  color: var(--ds-text-editor-variable-2-fg-color);
  font-weight: bold;
}
.editor-content .string {
  color: var(--ds-text-editor-string-fg-color);
}
.editor-content .number {
  color: var(--ds-text-editor-number-fg-color);
}

/* 提示框 */
.suggestion-box {
  position: absolute;
  background: var(--ds-text-editor-context-menu-bg-color);
  border: 1px solid var(--ds-border-color);
  list-style: none;
  padding: 4px;
  margin: 0;
  font-size: 12px;
  z-index: 10;
  max-height: 150px;
  overflow-y: auto;
  color: var(--ds-text-editor-context-menu-fg-color);
}
.suggestion-box li {
  padding: 2px 6px;
  cursor: pointer;
}
.suggestion-box li.active {
  background-color: var(--ds-selected-bg);
  color: var(--ds-theme-base);
}
</style>
