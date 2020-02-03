<template>
  <div>
    Inputting English Texts:
    <br>
    <textarea
      id
      v-model="inputtingText"
      name
      cols="30"
      rows="10"
    />
    <br>Generated Words Info:
    <br>
    <div class="generated-words-content">
      <textarea
        id
        v-model="generatedWordsInfo"
        name
        cols="30"
        rows="10"
      />
      <textarea
        id
        v-model="generatedWordsTextWithCount"
        name
        cols="30"
        rows="10"
      />
    </div>
    <br>Known words (
    <span class="known-word-import">
      <label class="import-input-label">
        <input
          class="file-input"
          type="file"
          name="upload"
          @change="handleFileInputChange"
        >
        导入
      </label>
    </span>
    <span
      class="known-word-export"
      @click="handleClickExportKnownWords"
    >导出</span>
    )
    <br>
    <textarea
      id
      v-model="knownWordsText"
      name
      cols="30"
      rows="10"
    />

    <br>Diff words with known words:
    <div class="diff-words-wrapper">
      <div>
        Diffing words:
        <br>
        <textarea
          id
          v-model="differingWordsText"
          name
          cols="30"
          rows="10"
        />
      </div>
      <div>
        Result words
        (
        <span
          class="diffed-word-export"
          @click="handleClickExportDiffedWords"
        >导出</span>
        )
        <br>
        <textarea
          id
          v-model="diffedResult"
          name
          cols="30"
          rows="10"
        />
      </div>
    </div>
  </div>
</template>
<script>
import * as download from '@/assets/js/download.js'
import { getFormattedDateString } from '@/utils/time'
import localStore from '@/localStore/localStore'
export default {
  name: 'App',
  props: {},
  data () {
    return {
      inputtingText: '',
      generatedWordsInfo: '',
      generatedWordsTextWithCount: '',

      knownWordsText: '',
      differingWordsText: '',
      diffedResult: ''
    }
  },
  computed: {
    knownWordsMap () {
      const knownWords = this.knownWordsText.split('\n')
      const res = {}
      for (const item of knownWords) {
        res[item.trim()] = 0
      }
      return res
    }
  },
  watch: {
    inputtingText (val) {
      const [outputString, outputStringWithCount] = this.resolveTextToGenerateWordsByFrequency(val)
      this.generatedWordsInfo = outputString
      this.generatedWordsTextWithCount = outputStringWithCount
    },
    differingWordsText () {
      let newWordsStr = ''
      const differingWords = this.differingWordsText.split('\n')
      for (let word of differingWords) {
        word = word.trim()
        if (this.knownWordsMap[word] === undefined) {
          newWordsStr = `${newWordsStr}${word}\n`
        }
      }
      this.diffedResult = newWordsStr
    },
    knownWordsText () {
      const { knownWordsText } = this
      localStore.setStore({
        knownWordsText
      })
    }
  },
  mounted () {
    // # local store
    const local = localStore.getStore()
    if (local && local.knownWordsText != null) {
      this.knownWordsText = local.knownWordsText
    }
  },
  methods: {
    resolveTextToGenerateWordsByFrequency (sourceText) {
      /** @type {{ [wordName: string]: number }} */
      const map = {}
      let outputString = ''
      let outputStringWithCount = ''
      let wordName = ''
      function dealWithWord (wordName) {
        if (map[wordName] === undefined) {
          map[wordName] = 1
        } else {
          map[wordName] = map[wordName] + 1
        }
      }
      for (let i = 0; i < sourceText.length; i++) {
        const char = sourceText[i]
        if (/[a-zA-Z\-]/.test(char)) {
          wordName = `${wordName}${char.toLowerCase()}`
        } else {
          wordName.trim() !== '' && dealWithWord(wordName)
          wordName = ''
        }
      }
      wordName.trim() !== '' && dealWithWord(wordName)

      /**
        # output string
        ```
        foo 10
        bar 10
        zoo 5
        ```
        */
      // # arr example: [ undefined,...., ['zoo'], ..., [ 'foo', 'bar' ] ]
      /** @type {string[][]} */
      const arr = []
      for (const wordName in map) {
        const index = map[wordName] - 1
        arr[index] =
          arr[index] === undefined ? [wordName] : [...arr[index], wordName]
      }
      for (let i = arr.length - 1; i >= 0; i--) {
        if (arr[i] === undefined) {
          continue
        }
        for (const wordName of arr[i]) {
          outputString = `${outputString}${wordName}\n`
          outputStringWithCount = `${outputStringWithCount}${wordName}  ${i + 1}\n`
        }
      }
      return [outputString, outputStringWithCount]
    },

    // # known words
    handleFileInputChange (event) {
      try {
        const reader = new FileReader()
        const fileInput = event.target
        reader.onload = event => {
          // Reset value so that uploading file which has
          // the same name next time still triggers change event
          fileInput.value = ''

          const { result: text } = event.target

          // Here deals with text
          this.knownWordsText = text
        }
        reader.readAsText(event.target.files[0])
      } catch (e) {}
    },
    handleClickExportKnownWords () {
      const defaultName = `known-words-${getFormattedDateString()}`
      const name = window.prompt('File Name', defaultName)
      name != null && download(this.knownWordsText, `${name}.txt`)
    },
    handleClickExportDiffedWords () {
      const defaultName = `result-words-${getFormattedDateString()}`
      const name = window.prompt('File Name', defaultName)
      name != null && download(this.diffedResult, `${name}.txt`)
    }
  }
}
</script>
<style scoped>
textarea {
  resize: none;
}
.known-word-import,
.known-word-export,
.diffed-word-export {
  color: blue;
  cursor: pointer;
}

.diff-words-wrapper {
  display: flex;
}
/* # generated words */
.generated-words-content {
  display: flex;
}

/* # known words */
.known-word-import .import-input-label {
  cursor: pointer;
}
.known-word-import .import-input-label input {
  display: none;
}
</style>
