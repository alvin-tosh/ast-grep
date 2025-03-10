<script setup lang="ts">
import { ref, watchEffect } from 'vue'
import Monaco from './Monaco.vue'
import TreeSitter from 'web-tree-sitter'
import init, {find_nodes, setup_parser} from 'ast-grep-wasm'
import SelectLang from './SelectLang.vue'
import Tabs from './Tabs.vue'

async function initializeTreeSitter() {
  await TreeSitter.init()
  let entrypoint = globalThis as any
  entrypoint.Parser = TreeSitter
  entrypoint.Language = TreeSitter.Language
}

await initializeTreeSitter()
await init()

let source = ref(
`/* All console.log() call will be highlighted!*/

function tryAstGrep() {
  console.log('Hello World')
}

const multiLineExpression =
  console
   .log('Also matched!')

if (true) {
  const notThis = 'console.log("not me")'
}`
)

enum Mode {
  Patch = 'Patch',
  Config = 'Config',
}
let query = ref('console.log($MATCH)')
let config = ref(`
# Configure Rule in YAML
rule:
  any:
    - pattern: if (false) { $$$ }
    - pattern: if (true) { $$$ }
constraints:
  # META_VAR: pattern
`.trim())
let lang = ref('javascript')
let langLoaded = ref(false)
let mode = ref(Mode.Patch)

const matchedHighlights = ref([])
const parserPaths: Record<string, string> = {
  javascript: 'tree-sitter-javascript.wasm',
  typescript: 'tree-sitter-typescript.wasm',
}

async function parseYAML(src: string) {
  const yaml = await import('js-yaml')
  return yaml.load(src)
}

async function doFind() {
  if (mode.value === Mode.Patch) {
    return find_nodes(
      source.value,
      {rule: {pattern: query.value}},
    )
  } else {
    const src = source.value
    const val = config.value;
    const json = await parseYAML(val)
    return find_nodes(
      src,
      json,
    )
  }
}

watchEffect(async () => {
  langLoaded.value = false
  await setup_parser(parserPaths[lang.value])
  langLoaded.value = true
})

watchEffect(async () => {
  try {
    if (!langLoaded.value) {
      return () => {}
    }
    matchedHighlights.value = JSON.parse(await doFind())
  } catch (e) {
    matchedHighlights.value = []
  }
  return () => {}
})


const modeText = {
  [Mode.Patch]: 'Pattern Code',
  [Mode.Config]: 'YAML Rule',
}
let placeholder = ref('code')

</script>

<template>
  <main class="playground">
    <div class="half">
      <Tabs v-model="placeholder" :modeText="{code: 'Test Code'}">
      <template #code>
        <Monaco v-model="source" :language="lang" :highlights="matchedHighlights"/>
      </template>
      <template #addon>
        <p class="match-result">
          <span  v-if="matchedHighlights.length > 0">
            Found {{ matchedHighlights.length }} match(es).
          </span>
          <span v-else>No match found.</span>
        </p>
      </template>
      </Tabs>
    </div>
    <div class="half">
      <Tabs v-model="mode" :modeText="modeText">
        <template #[Mode.Patch]>
          <Monaco :language="lang" v-model="query"/>
        </template>
        <template #[Mode.Config]>
          <Monaco language="yaml" v-model="config"/>
        </template>
        <template #addon>
          <SelectLang v-model="lang"/>
        </template>
      </Tabs>
    </div>
  </main>
</template>

<style scoped>
.playground {
  display: flex;
  flex-wrap: wrap;
  flex: 1 0 auto;
  align-items: stretch;
}
.half {
  flex: 1 0 30%;
  display: flex;
  flex-direction: column;
  filter: drop-shadow(0 0 8px #00000010);
}
.half:first-child {
  margin-right: 10px;
}
.half:focus-within {
  filter: drop-shadow(0 0 16px #00000020);
}
</style>
