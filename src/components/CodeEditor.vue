<template>
  <div class="editor-wrapper" :style="{ width, height }">
    <div class="editor-gutter">
      <div v-for="(line, i) in lines" :key="i" class="editor-line-number">
        {{ i + 1 }}
      </div>
    </div>
    <div class="editor-main">
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
    const lineNumbers = computed(() =>
      Array.from({ length: lines.value.length }, (_, i) =>
        (i + 1).toString(),
      ).join("\n"),
    );
    const escapeHtml = (str: string): string =>
      str.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");
    const highlightLine = (line: string): string => {
      if (line === "") return "&nbsp;";
      let html = escapeHtml(line);
      html = html.replace(/(".*?"|'.*?')/g, '<span class="string">$1</span>');
      html = html.replace(/\b(\d+)\b/g, '<span class="number">$1</span>');
      html = html.replace(
        /\b(function|return|const|let|var|if|else|select|from|set|where|insert|update|delete|and|or|left|right|join|on)\b/gi,
        '<span class="keyword">$1</span>',
      );
      return html;
    };
    const syncScroll = (): void => {
      if (!editor.value) return;
    };
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
    const onInput = (): void => {
      if (lines.value.length === 0) {
        lines.value = [""];
      }
      syncLinesFromEditor();
      emit("update:modelValue", lines.value.join("\n"));
      updateSuggestions();
    };
    const onKeyDown = (e: KeyboardEvent): void => {
      if (!editor.value) return;
      if (e.key === "Backspace") {
        const sel = window.getSelection();
        if (sel && sel.rangeCount > 0) {
          const range = sel.getRangeAt(0);
          // 如果有选区就允许删除（保持默认行为）
          if (range.collapsed) {
            const { node, offset } = getTextNodeAndOffset(
              range.startContainer,
              range.startOffset,
            );

            // 找到该 node 所在的直接行容器（editor 的直接子元素 <div>）
            let el: HTMLElement | null = null;
            if (node.nodeType === Node.TEXT_NODE) {
              el = node.parentElement;
            } else if (node instanceof HTMLElement) {
              el = node;
            }
            let parentDiv: HTMLElement | null = null;
            while (el && el !== editor.value) {
              if (el.parentElement === editor.value) {
                parentDiv = el;
                break;
              }
              el = el.parentElement;
            }

            // 如果没有找到父行容器，尝试 fallback 到第一个子节点（若存在）
            if (!parentDiv) {
              parentDiv = editor.value.firstElementChild as HTMLElement | null;
            }

            // 计算该光标在行内的偏移（遍历行内文本节点累计长度）
            let totalBefore = 0;
            if (parentDiv) {
              const tw = document.createTreeWalker(
                parentDiv,
                NodeFilter.SHOW_TEXT,
                null,
              );
              let n: Node | null = null;
              while ((n = tw.nextNode())) {
                if (n === node) {
                  totalBefore += offset;
                  break;
                }
                totalBefore += n.textContent?.length ?? 0;
              }
            } else {
              // editor 没有子行，视为在第一行开头
              totalBefore = 0;
            }

            const lineIndex = parentDiv
              ? Array.from(editor.value.children).indexOf(parentDiv)
              : 0;

            // 如果在第一行且行内偏移为 0，则阻止 Backspace
            if (lineIndex === 0 && totalBefore === 0) {
              e.preventDefault();
              return;
            }
          }
        }
      }
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
          const node: Node | null = range.startContainer;
          let el: HTMLElement | null =
            node && node.nodeType === Node.ELEMENT_NODE
              ? (node as HTMLElement)
              : ((node as Node & { parentElement?: HTMLElement })
                  .parentElement ?? null);
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
        const selectionAfter = window.getSelection();
        if (selectionAfter) selectionAfter.removeAllRanges();
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
        if (selectionAfter) selectionAfter.addRange(newRange);
        syncLinesFromEditor();
        emit("update:modelValue", lines.value.join("\n"));
      }
      if (e.key === " ") {
        hideSuggestions();
        syncLinesFromEditor();
        renderEditor();
        emit("update:modelValue", lines.value.join("\n"));
      }
      if (
        showSuggestions.value &&
        (e.key === "ArrowDown" || e.key === "ArrowUp")
      ) {
        e.preventDefault();
        if (suggestions.value.length === 0) return;
        if (suggestionIndex.value === -1) {
          suggestionIndex.value =
            e.key === "ArrowDown" ? 0 : suggestions.value.length - 1;
        } else {
          if (e.key === "ArrowDown") {
            suggestionIndex.value =
              (suggestionIndex.value + 1) % suggestions.value.length;
          } else if (e.key === "ArrowUp") {
            suggestionIndex.value =
              (suggestionIndex.value - 1 + suggestions.value.length) %
              suggestions.value.length;
          }
        }
      }
      if (showSuggestions.value && e.key === "Enter") {
        if (suggestionIndex.value == -1) {
          suggestionIndex.value = 0;
        }
        e.preventDefault();
        const sel = window.getSelection();
        if (!sel || !sel.rangeCount) return;
        const range = sel.getRangeAt(0);
        const { node, offset } = getTextNodeAndOffset(
          range.startContainer,
          range.startOffset,
        );
        const textBefore = (node.textContent ?? "").slice(0, offset);
        const currentWord = textBefore.split(/\s+/).pop() ?? "";
        if (!currentWord) return;
        const startOffset = offset - currentWord.length;
        range.setStart(node, startOffset);
        range.setEnd(node, offset);
        range.deleteContents();
        const inserted = document.createTextNode(
          suggestions.value[suggestionIndex.value],
        );
        range.insertNode(inserted);
        range.setStartAfter(inserted);
        range.collapse(true);
        sel.removeAllRanges();
        sel.addRange(range);
        hideSuggestions();
        syncLinesFromEditor();
        renderEditor();
        emit("update:modelValue", lines.value.join("\n"));
      }
    };

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
    const renderEditor = (): void => {
      if (!editor.value) return;

      const selection = window.getSelection();
      let caretOffset = 0;

      // 计算光标相对整个编辑器的字符偏移
      if (selection && selection.rangeCount > 0) {
        const range = selection.getRangeAt(0);
        const preCaretRange = range.cloneRange();
        preCaretRange.selectNodeContents(editor.value);
        preCaretRange.setEnd(range.endContainer, range.endOffset);
        caretOffset = preCaretRange.toString().length;
      }

      // 重新渲染
      editor.value.innerHTML = "";
      lines.value.forEach((line) => {
        const div = document.createElement("div");
        div.innerHTML = highlightLine(line);
        editor.value?.appendChild(div);
      });

      // 恢复光标
      if (caretOffset > 0) {
        const walker = document.createTreeWalker(
          editor.value,
          NodeFilter.SHOW_TEXT,
          null,
        );
        let remaining = caretOffset;
        let node: Node | null = null;

        while ((node = walker.nextNode())) {
          const len = node.textContent?.length ?? 0;
          if (remaining <= len) {
            const range = document.createRange();
            range.setStart(node, remaining);
            range.collapse(true);
            selection?.removeAllRanges();
            selection?.addRange(range);
            break;
          }
          remaining -= len;
        }
      }
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
      "VAR",
      "IF",
      "ELSE",
      "JOIN",
      "LEFT JOIN",
      "RIGHT JOIN",
      "INNER JOIN",
      "LET",
      "AND",
      "OR",
      "ON",
    ];
    const showSuggestions = ref(false);
    const suggestions = ref<string[]>([]);
    const suggestionIndex = ref(-1);
    const suggestionPos = ref({ top: 0, left: 0 });
    const getTextNodeAndOffset = (
      container: Node,
      offset: number,
    ): { node: Text; offset: number } => {
      if (container.nodeType === Node.TEXT_NODE) {
        return { node: container as Text, offset };
      }
      const el = container as Node & { childNodes: NodeListOf<ChildNode> };
      if (el.childNodes.length === 0) {
        const t = document.createTextNode("");
        (container as HTMLElement).appendChild(t);
        return { node: t, offset: 0 };
      }
      let target: Node | null = null;
      if (offset > 0) {
        target = el.childNodes[offset - 1];
      } else {
        target = el.childNodes[0];
      }
      if (!target) {
        const t = document.createTextNode("");
        (container as HTMLElement).appendChild(t);
        return { node: t, offset: 0 };
      }
      if (target.nodeType === Node.TEXT_NODE) {
        return {
          node: target as Text,
          offset: (target.textContent ?? "").length,
        };
      }
      let last: Node | null = target;
      while (last && last.nodeType !== Node.TEXT_NODE && last.lastChild) {
        last = last.lastChild;
      }
      if (last && last.nodeType === Node.TEXT_NODE) {
        return { node: last as Text, offset: (last.textContent ?? "").length };
      }
      const t = document.createTextNode("");
      (container as HTMLElement).appendChild(t);
      return { node: t, offset: 0 };
    };
    const updateSuggestions = (): void => {
      const sel = window.getSelection();
      if (!sel || !editor.value || !sel.rangeCount) return;
      const range = sel.getRangeAt(0);
      const { node, offset } = getTextNodeAndOffset(
        range.startContainer,
        range.startOffset,
      );
      const textBefore = (node.textContent ?? "").slice(0, offset);
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
        suggestionIndex.value = -1;
        showSuggestions.value = true;
        const cloned = range.cloneRange();
        const rect = cloned.getBoundingClientRect();
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
      suggestionIndex.value = -1;
    };
    return {
      editor,
      lines,
      lineNumbers,
      onInput,
      onKeyDown,
      onPaste,
      syncScroll,
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

.editor-line-number {
  height: 1.5em;
  line-height: 1.5;
  font-family: monospace;
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
