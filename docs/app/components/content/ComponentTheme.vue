<script setup lang="ts">
import json5 from 'json5'
import { camelCase } from 'scule'
import { hash } from 'ohash'
import * as theme from '#build/ui'
import * as themePro from '#build/ui-pro'

const props = defineProps<{
  pro?: boolean
  prose?: boolean
  slug?: string
  extra?: string[]
}>()

const route = useRoute()
const { framework } = useSharedData()

const name = camelCase(props.slug ?? route.params.slug?.[route.params.slug.length - 1] ?? '')

const strippedCompoundVariants = ref(false)

const computedTheme = computed(() => props.pro ? props.prose ? themePro.prose : themePro : theme)

const strippedTheme = computed(() => {
  const strippedTheme = {
    ...(computedTheme.value as any)[name]
  }

  if (strippedTheme?.compoundVariants) {
    strippedTheme.compoundVariants = strippedTheme.compoundVariants.filter((compoundVariant: any) => {
      if (compoundVariant.color) {
        if (!['primary', 'neutral'].includes(compoundVariant.color)) {
          strippedCompoundVariants.value = true

          return false
        }
      }

      if (compoundVariant.highlightColor) {
        if (!['primary', 'neutral'].includes(compoundVariant.highlightColor)) {
          strippedCompoundVariants.value = true

          return false
        }
      }

      if (compoundVariant.loadingColor) {
        if (!['primary', 'neutral'].includes(compoundVariant.loadingColor)) {
          strippedCompoundVariants.value = true

          return false
        }
      }

      return true
    })
  }

  return strippedTheme
})

const component = computed(() => {
  const baseKey = props.pro ? 'uiPro' : 'ui'

  const content = props.prose
    ? { prose: { [name]: strippedTheme.value } }
    : { [name]: strippedTheme.value }

  if (props.extra?.length) {
    props.extra.forEach((extra) => {
      const target = props.prose ? content.prose! : content
      target[extra as keyof typeof target] = computedTheme.value[extra as keyof typeof computedTheme.value]
    })
  }

  return {
    [baseKey]: content
  }
})

const { data: ast } = await useAsyncData(`component-theme-${name}-${hash({ props })}`, async () => {
  const md = `
::code-collapse{class="nuxt-only"}

\`\`\`ts [app.config.ts]
export default defineAppConfig(${json5.stringify(component.value, null, 2).replace(/,([ |\t\n]+[}|\])])/g, '$1')})
\`\`\`\

::

::code-collapse{class="vue-only"}

\`\`\`ts [vite.config.ts]
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import ui from '@nuxt/ui/vite'

export default defineConfig({
  plugins: [
    vue(),
    ui(${json5.stringify(component.value, null, 2).replace(/,([ |\t\n]+[}|\])])/g, '$1')
      .split('\n')
      .map((line, i) => i === 0 ? line : `    ${line}`)
      .join('\n')})
  ]
})
\`\`\`

::

${strippedCompoundVariants.value
  ? `
::callout{icon="i-simple-icons-github" to="https://github.com/nuxt/ui/blob/v3/src/theme/${name}.ts"}
Some colors in \`compoundVariants\` are omitted for readability. Check out the source code on GitHub.
::`
  : ''}
`

  return parseMarkdown(md)
}, { watch: [framework] })
</script>

<template>
  <MDCRenderer v-if="ast" :body="ast.body" :data="ast.data" />
</template>
