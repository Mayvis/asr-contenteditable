<script setup lang="ts">
import { onMounted, onUnmounted, ref } from "vue"

const text = ref("")
const result = ref("")

const platform = (
  navigator.userAgentData?.platform || navigator.platform
).toLowerCase()

function handleContenteditableInput(e: Event) {
  const target = e.target as HTMLDivElement

  target.childNodes.forEach((childNode, index) => {
    if (childNode.nodeName === "FONT") {
      // browser will save the style, and add font tag on it, so manually manipulate the font tag to text node
      const text = childNode.textContent || ""
      const parentNode = childNode.parentNode

      if (parentNode) {
        parentNode.removeChild(childNode)
        const newTextNode = document.createTextNode(text)
        parentNode.insertBefore(newTextNode, parentNode.childNodes[index])

        const selection = window.getSelection() as Selection
        const cloneRange = selection.getRangeAt(0).cloneRange()

        cloneRange.setStart(newTextNode, 1)

        selection.removeAllRanges()
        selection.addRange(cloneRange)
      }
    }
  })
}

function handleContenteditableKeydown(e: KeyboardEvent) {
  if (e.code === "Enter" && e.isComposing === false) {
    e.preventDefault()
    return
  }

  if (e.code === "Tab") {
    e.preventDefault()
    return
  }

  if (e.isComposing === true) return

  const isController =
    e.code === "ArrowDown" ||
    e.code === "ArrowUp" ||
    e.code === "ArrowLeft" ||
    e.code === "ArrowRight"

  if (isController) return

  const selection = window.getSelection() as Selection
  const cloneRange = selection.getRangeAt(0).cloneRange()

  if (cloneRange.startContainer.nodeName === "#text") {
    const parentNode = cloneRange.startContainer.parentNode

    if (parentNode) {
      if (
        e.key !== "Backspace" &&
        e.key !== "Delete" &&
        e.key !== "Enter" &&
        e.key !== "Tab" &&
        e.ctrlKey === false &&
        e.metaKey === false &&
        (e.altKey === false || (e.key !== "Alt" && e.altKey === true)) &&
        (e.shiftKey === false || (e.key !== "Shift" && e.shiftKey === true))
      ) {
        if (parentNode.nodeName === "SPAN") {
          if (cloneRange.startOffset === 0 && cloneRange.endOffset === 0) {
            cloneRange.setStartBefore(parentNode)

            const node = document.createTextNode("_")
            cloneRange.collapse(false)

            parentNode.parentNode?.insertBefore(node, parentNode)

            cloneRange.setStart(node, 0)

            selection.removeAllRanges()
            selection.addRange(cloneRange)
          } else if (
            cloneRange.startOffset ===
            cloneRange.startContainer.textContent?.length
          ) {
            cloneRange.setEndAfter(parentNode)

            const node = document.createTextNode("_")
            cloneRange.collapse(false)

            cloneRange.insertNode(node)
            cloneRange.setStart(node, 0)

            selection.removeAllRanges()
            selection.addRange(cloneRange)
          }
        }
      }
    }
  } else if (cloneRange.startContainer.nodeName === "DIV") {
    if (
      e.key !== "Backspace" &&
      e.key !== "Delete" &&
      e.key !== "Enter" &&
      e.key !== "Tab" &&
      e.ctrlKey === false &&
      e.metaKey === false &&
      (e.altKey === false || (e.key !== "Alt" && e.altKey === true)) &&
      (e.shiftKey === false || (e.key !== "Shift" && e.shiftKey === true))
    ) {
      if (cloneRange.startOffset === cloneRange.endOffset) {
        const node = document.createTextNode("_")
        cloneRange.collapse(false)

        cloneRange.insertNode(node)

        cloneRange.setStart(node, 0)

        selection.removeAllRanges()
        selection.addRange(cloneRange)
      } else {
        const node = document.createTextNode("_")
        cloneRange.collapse(false)

        cloneRange.setStartBefore(cloneRange.startContainer.parentNode as Node)

        cloneRange.insertNode(node)

        cloneRange.setStart(node, 0)

        selection.removeAllRanges()
        selection.addRange(cloneRange)
      }
    }
  }
}

function handleCreateTag(tag: string) {
  const node = document.createElement("span")
  node.innerText = tag
  node.classList.add(tag === "[TW]" ? "tw" : "en")

  return node
}

function handleInsertTag(tag: string) {
  const selection = window.getSelection() as Selection
  const cloneRange = selection.getRangeAt(0).cloneRange()

  // 代表是用框選的方式，使用此種方式不能貼
  if (!cloneRange.collapsed) return

  if (cloneRange.startContainer.nodeName === "#text") {
    const parentNode = cloneRange.startContainer.parentNode

    if (parentNode) {
      if (parentNode.nodeName === "SPAN") {
        if (cloneRange.startOffset === 0 && cloneRange.endOffset === 0) {
          // <div contenteditable="true"><span>hello</span></div> <- 代表從div及em中間插入
          const node = handleCreateTag(tag)

          parentNode.parentNode?.insertBefore(node, parentNode)

          cloneRange.collapse(false)

          cloneRange.setStartAfter(node)

          selection.removeAllRanges()
          selection.addRange(cloneRange)
        } else if (cloneRange.startOffset === parentNode.textContent?.length) {
          // <span>hello</span> <- 代表從後面插入
          const node = handleCreateTag(tag)

          cloneRange.setEndAfter(parentNode)
          cloneRange.collapse(false)
          cloneRange.insertNode(node)

          cloneRange.setStartAfter(node)

          selection.removeAllRanges()
          selection.addRange(cloneRange)
        }
      } else if (parentNode.nodeName === "DIV") {
        // 代表是 contenteditable，直接就現在的位置進行插入
        const node = handleCreateTag(tag)

        cloneRange.collapse(false)
        cloneRange.insertNode(node)

        cloneRange.setStartAfter(node)

        selection.removeAllRanges()
        selection.addRange(cloneRange)
      }
    }
  } else if (cloneRange.startContainer.nodeName === "DIV") {
    if (cloneRange.startContainer.textContent?.length === 0) {
      // 代表直接在 contenteditable 插入且 contenteditable 沒有任何資料，那就直接插入
      const node = handleCreateTag(tag)

      cloneRange.collapse(false)
      cloneRange.insertNode(node)

      cloneRange.setStartAfter(node)

      selection.removeAllRanges()
      selection.addRange(cloneRange)
    } else {
      // 代表直接在 contenteditable 插入，但 contenteditable 有資料，那就在最後面插入
      const node = handleCreateTag(tag)

      cloneRange.setEndAfter(
        cloneRange.startContainer.childNodes[cloneRange.startOffset - 1]
      )
      cloneRange.collapse(false)
      cloneRange.insertNode(node)

      cloneRange.setStartAfter(node)

      selection.removeAllRanges()
      selection.addRange(cloneRange)
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

  result.value = html.replace(/(<([^>]+)>)/gi, "").replace(/&nbsp;/g, "")
}

function handleClear() {
  result.value = ""
}

onMounted(() => {
  document.addEventListener("keydown", handleRegisterKeydown)

  const data = "he[TW]ll[TW]o wor[EN]ld"
    .replaceAll("[TW]", '<span class="tw">[TW]</span>')
    .replaceAll("[EN]", '<span class="en">[EN]</span>')

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
    @input="handleContenteditableInput"
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

  :deep(span) {
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
