<script setup lang="ts">
import '@docsearch/css'
import docsearch from '@docsearch/js'
import { useRoute, useRouter, useData } from 'vitepress'
import { getCurrentInstance, onMounted, watch } from 'vue'
import type { DefaultTheme } from '../config'
import type { DocSearchHit } from '@docsearch/react/dist/esm/types'

const props = defineProps<{
  options: DefaultTheme.AlgoliaSearchOptions
  multilang?: boolean
}>()

const vm = getCurrentInstance()
const route = useRoute()
const router = useRouter()

watch(
  () => props.options,
  (value) => {
    update(value)
  }
)

onMounted(() => {
  initialize(props.options)
})

function isSpecialClick(event: MouseEvent) {
  return (
    event.button === 1 ||
    event.altKey ||
    event.ctrlKey ||
    event.metaKey ||
    event.shiftKey
  )
}

function getRelativePath(absoluteUrl: string) {
  const { pathname, hash } = new URL(absoluteUrl)

  return pathname + hash
}

function update(options: any) {
  if (vm && vm.vnode.el) {
    vm.vnode.el.innerHTML =
      '<div class="algolia-search-box" id="docsearch"></div>'
    initialize(options)
  }
}

const { lang } = useData()

// if the user has multiple locales, the search results should be filtered
// based on the language
const facetFilters: string[] = props.multilang
  ? ['lang:' + lang.value]
  : []

if (props.options.searchParameters?.facetFilters) {
  facetFilters.push(...props.options.searchParameters.facetFilters)
}

watch(
  lang,
  (newLang, oldLang) => {
    const index = facetFilters.findIndex(
      (filter) => filter === 'lang:' + oldLang
    )
    if (index > -1) {
      facetFilters.splice(index, 1, 'lang:' + newLang)
    }
  }
)

function initialize(userOptions: any) {
  docsearch(
    Object.assign({}, userOptions, {
      container: '#docsearch',

      searchParameters: Object.assign({}, userOptions.searchParameters, {
        // pass a custom lang facetFilter to allow multiple language search
        // https://github.com/algolia/docsearch-configs/pull/3942
        facetFilters
      }),

      navigator: {
        navigate: ({ itemUrl }: { itemUrl: string }) => {
          const { pathname: hitPathname } = new URL(
            window.location.origin + itemUrl
          )

          // Router doesn't handle same-page navigation so we use the native
          // browser location API for anchor navigation
          if (route.path === hitPathname) {
            window.location.assign(window.location.origin + itemUrl)
          } else {
            router.go(itemUrl)
          }
        }
      },

      transformItems: (items: DocSearchHit[]) => {
        return items.map((item) => {
          return Object.assign({}, item, {
            url: getRelativePath(item.url)
          })
        })
      },

      hitComponent: ({
        hit,
        children
      }: {
        hit: DocSearchHit
        children: any
      }) => {
        const relativeHit = hit.url.startsWith('http')
          ? getRelativePath(hit.url as string)
          : hit.url

        return {
          type: 'a',
          ref: undefined,
          constructor: undefined,
          key: undefined,
          props: {
            href: hit.url,
            onClick: (event: MouseEvent) => {
              if (isSpecialClick(event)) {
                return
              }

              // we rely on the native link scrolling when user is already on
              // the right anchor because Router doesn't support duplicated
              // history entries
              if (route.path === relativeHit) {
                return
              }

              // if the hits goes to another page, we prevent the native link
              // behavior to leverage the Router loading feature
              if (route.path !== relativeHit) {
                event.preventDefault()
              }

              router.go(relativeHit)
            },
            children
          },
          __v: null
        }
      }
    })
  )
}
</script>

<template>
  <div class="algolia-search-box" id="docsearch" />
</template>

<style>
.algolia-search-box {
  padding-top: 1px;
}

@media (min-width: 720px) {
  .algolia-search-box {
    padding-left: 8px;
  }
}

@media (min-width: 751px) {
  .algolia-search-box {
    min-width: 176.3px; /* avoid layout shift */
  }

  .algolia-search-box .DocSearch-Button-Placeholder {
    padding-left: 8px;
    font-size: 0.9rem;
    font-weight: 500;
  }
}

.DocSearch {
  --docsearch-primary-color: var(--c-brand);
  --docsearch-highlight-color: var(--docsearch-primary-color);
  --docsearch-searchbox-shadow: inset 0 0 0 2px var(--docsearch-primary-color);
  --docsearch-text-color: var(--c-text-light);
  --docsearch-muted-color: var(--c-text-lighter);
  --docsearch-searchbox-background: #f2f2f2;
}
</style>
