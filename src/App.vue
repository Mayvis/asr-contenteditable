<script setup lang="ts">
import { onMounted, onUnmounted, ref } from "vue"

const text = ref("")
const result = ref("")

const platform = (
  navigator.userAgentData?.platform || navigator.platform
).toLowerCase()

function handleContenteditableKeydown(e: KeyboardEvent) {
  const isController =
    e.code === "ArrowDown" ||
    e.code === "ArrowUp" ||
    e.code === "ArrowLeft" ||
    e.code === "ArrowRight"

  if (isController) return

  const selection = window.getSelection() as Selection
  const cloneRange = selection.getRangeAt(0).cloneRange()

  if (cloneRange.startContainer.isSameNode(cloneRange.endContainer)) {
    const parentNode = cloneRange.startContainer.parentNode

    if (parentNode && parentNode?.nodeName === "EM") {
      if (
        e.key !== "Backspace" &&
        e.key !== "Delete" &&
        e.key !== "Enter" &&
        e.metaKey &&
        e.ctrlKey
      ) {
        const span = document.createElement("span")
        span.innerText = " "

        cloneRange.collapse(false)
        cloneRange.setStartAfter(parentNode)
        cloneRange.insertNode(span)

        cloneRange.setStart(span, 0)

        selection.removeAllRanges()
        selection.addRange(cloneRange)
      }
    }
  }
}

function handleCreateTag(tag: string) {
  const node = document.createElement("em")
  node.innerText = tag
  node.classList.add(tag === "[TW]" ? "tw" : "en")

  return node
}

function handleInsertTag(tag: string) {
  const selection = window.getSelection() as Selection
  const cloneRange = selection.getRangeAt(0).cloneRange()

  // 代表是用框選的方式，使用此種方式不能貼
  if (!cloneRange.collapsed) return

  if (cloneRange.startContainer.isSameNode(cloneRange.endContainer)) {
    if (cloneRange.startContainer.nodeName === "DIV") {
      const isContenteditableDOM = cloneRange.startContainer.isSameNode(
        cloneRange.commonAncestorContainer
      )

      if (isContenteditableDOM) {
        const node = handleCreateTag(tag)
        cloneRange.insertNode(node)
        cloneRange.setStartAfter(node)

        cloneRange.collapse(false)

        selection.removeAllRanges()
        selection.addRange(cloneRange)

        return
      }
    }

    const parentNode = cloneRange.startContainer.parentNode
    const parentElement = cloneRange.startContainer.parentElement

    if (parentNode?.nodeName === "EM") {
      if (
        (cloneRange.startOffset === 0 && cloneRange.endOffset === 0) ||
        (cloneRange.startOffset ===
          cloneRange.startContainer.textContent?.length &&
          cloneRange.endOffset === cloneRange.endContainer.textContent?.length)
      ) {
        const node = handleCreateTag(tag)
        cloneRange.insertNode(node)
        cloneRange.setStartBefore(node)

        cloneRange.collapse(false)

        selection.removeAllRanges()
        selection.addRange(cloneRange)

        return
      }
    } else if (parentNode?.nodeName === "DIV") {
      const isContenteditableDOM =
        parentElement?.classList.contains("contenteditable")

      if (isContenteditableDOM) {
        const node = handleCreateTag(tag)
        cloneRange.insertNode(node)
        cloneRange.setStartAfter(node)

        cloneRange.collapse(false)

        selection.removeAllRanges()
        selection.addRange(cloneRange)

        return
      }
    }
  }
}

function handleRegisterKeydown(e: KeyboardEvent) {
  // metaKey for mac option key
  const modifier = platform.toLowerCase().startsWith("mac")
    ? e.metaKey
    : e.ctrlKey

  if (modifier && e.key === "1") {
    e.preventDefault()
    handleInsertTag("[TW]")
  } else if (modifier && e.key === "2") {
    e.preventDefault()
    handleInsertTag("[EN]")
  }
}

function handleFormat() {
  const html = (document.querySelector(".contenteditable") as HTMLDivElement)
    .innerHTML

  result.value = html
    .replaceAll("[TW]", "")
    .replaceAll("[EN]", "")
    .replaceAll('<em class="tw">', "[TW]")
    .replaceAll('<em class="en">', "[EN]")
    .replaceAll("</em>", "")
    .replace(/(<([^>]+)>)/gi, "")
}

function handleClear() {
  result.value = ""
}

onMounted(() => {
  document.addEventListener("keydown", handleRegisterKeydown)

  const data = "he[TW]ll[TW]o wor[EN]ld"
    .replaceAll("[TW]", '<em class="tw">[TW]</em>')
    .replaceAll("[EN]", '<em class="en">[EN]</em>')

  text.value = data
})

onUnmounted(() => {
  document.removeEventListener("keydown", handleRegisterKeydown)
})
</script>

<template>
  <div
    contenteditable="true"
    class="contenteditable"
    v-html="text"
    @keydown="handleContenteditableKeydown"
  ></div>

  <div class="result">
    <button @click="handleFormat">產出結果</button>
    <button @click="handleClear">清除結果</button>
    <div>{{ result }}</div>
  </div>
</template>

<style scoped lang="scss">
.contenteditable {
  border: 1px solid #ccc;
  padding: 10px;
  min-height: 100px;

  :deep(em) {
    &.tw {
      color: red;
    }

    &.en {
      color: orange;
    }
  }
}

.result {
  margin-top: 20px;
}
</style>