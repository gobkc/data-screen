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
        @scroll="syncScroll"
      ></div>
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

    const escapeHtml = (str: string) =>
      str.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");

    const highlightLine = (line: string) => {
      if (line === "") return "&nbsp;";
      let html = escapeHtml(line);
      html = html.replace(/(".*?"|'.*?')/g, '<span class="string">$1</span>');
      html = html.replace(/\b(\d+)\b/g, '<span class="number">$1</span>');
      html = html.replace(
        /\b(function|return|const|let|var|if|else)\b/g,
        '<span class="keyword">$1</span>',
      );
      return html;
    };

    const syncScroll = () => {
      if (!editor.value) return;
      // No longer need to sync scroll for highlight layer
    };

    const syncLinesFromEditor = () => {
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

    const onInput = () => {
      syncLinesFromEditor();
      emit("update:modelValue", lines.value.join("\n"));
    };
    const onKeyDown = (e: KeyboardEvent) => {
      if (!editor.value) return;
      if (e.key === "Tab" || e.key === "Backspace") {
        e.preventDefault();
        const selection = window.getSelection();
        if (!selection || selection.rangeCount === 0) return;
        const range = selection.getRangeAt(0);
        const editorDivs = Array.from(
          editor.value.children,
        ) as HTMLDivElement[];
        const selectedLines: HTMLDivElement[] = [];

        editorDivs.forEach((div) => {
          const divRange = document.createRange();
          divRange.selectNodeContents(div);
          const intersects =
            range.compareBoundaryPoints(Range.END_TO_START, divRange) < 0 &&
            range.compareBoundaryPoints(Range.START_TO_END, divRange) > 0;
          if (intersects) selectedLines.push(div);
        });

        if (selectedLines.length === 0 && range.startContainer) {
          const parentDiv = range.startContainer.parentElement;
          if (parentDiv && parentDiv.parentElement === editor.value)
            selectedLines.push(parentDiv as HTMLDivElement);
        }

        // Adjust the indent for selected lines
        selectedLines.forEach((div) => {
          const text = div.innerText;
          div.innerText = e.shiftKey
            ? text.startsWith("  ")
              ? text.slice(2) // Shift + Tab will decrease indent
              : text
            : "  " + text; // Tab will increase indent
        });

        // Re-render the highlighted part
        selectedLines.forEach((div) => {
          const highlightedHTML = highlightLine(div.innerText);
          div.innerHTML = highlightedHTML;
        });

        selection.removeAllRanges();
        if (selectedLines.length > 0) {
          const newRange = document.createRange();
          newRange.setStart(selectedLines[0].firstChild || selectedLines[0], 0);

          const lastDiv = selectedLines[selectedLines.length - 1];
          const endOffset = Math.min(
            lastDiv.textContent?.length ?? 0,
            range.endOffset,
          );

          newRange.setEnd(lastDiv.firstChild || lastDiv, endOffset);
          selection.addRange(newRange);
        }

        syncLinesFromEditor();
        emit("update:modelValue", lines.value.join("\n"));
      }
    };

    const renderEditor = () => {
      if (!editor.value) return;
      editor.value.innerHTML = "";
      lines.value.forEach((line) => {
        const div = document.createElement("div");
        // Use innerHTML to insert the highlighted HTML directly
        div.innerHTML = highlightLine(line);
        editor.value?.appendChild(div);
      });
    };

    watch(
      () => props.modelValue,
      (newVal) => {
        if (newVal !== lines.value.join("\n")) {
          lines.value = newVal.split("\n");
          nextTick(() => {
            renderEditor();
          });
        }
      },
    );

    onMounted(() => {
      renderEditor();
    });

    return {
      editor,
      lines,
      lineNumbers,
      onInput,
      onKeyDown,
      syncScroll,
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
</style>
