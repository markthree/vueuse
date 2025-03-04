<script setup lang="ts">
import { computed, ref, toRef } from 'vue'
import Fuse from 'fuse.js'
import { useUrlSearchParams } from '@vueuse/core'
import { categories, functions } from '../../../../meta/function-indexes'

const coreCategories = categories.filter(i => !i.startsWith('@'))
const addonCategories = categories.filter(i => i.startsWith('@'))
const sortMethods = ['category', 'name', 'updated']

const query = useUrlSearchParams('hash-params', { removeFalsyValues: true })
const search = toRef(query, 'search')
const category = toRef(query, 'category')
const hasComponent = toRef(query, 'component')
const sortMethod = toRef(query, 'sort') as Ref<'category' | 'name' | 'updated'>

const showCategory = computed(() => !search.value && (!sortMethod.value || sortMethod.value === 'category'))

const items = computed(() => {
  let fn = functions.filter(i => !i.internal)
  if (hasComponent.value)
    fn = fn.filter(i => i.component || i.directive)
  if (!category.value)
    return fn
  return fn.filter(item => item.category === category.value)
})
const fuse = computed(() => new Fuse(items.value, {
  keys: ['name', 'description'],
}))
const result = computed(() => {
  if (search.value) {
    return fuse.value.search(search.value).map(i => i.item)
  }
  else {
    const fns = [...items.value]
    if (sortMethod.value === 'updated')
      fns.sort((a, b) => (b.lastUpdated) || 0 - (a.lastUpdated || 0))
    else if (sortMethod.value === 'name')
      fns.sort((a, b) => a.name.localeCompare(b.name))
    else
      fns.sort((a, b) => categories.indexOf(a.category) - categories.indexOf(b.category))
    return fns
  }
})

const hasFilters = computed(() => Boolean(search.value || category.value || hasComponent.value || sortMethod.value))

function resetFilters() {
  sortMethod.value = null
  category.value = null
  hasComponent.value = null
  search.value = null
}

function toggleCategory(cate: string) {
  category.value = category.value === cate ? null : cate
}

function toggleSort(method: string) {
  sortMethod.value = method
}
</script>

<template>
  <div class="grid grid-cols-[80px,auto] gap-y-2">
    <div opacity="80" text="sm">
      Core
    </div>
    <div flex="~ wrap" gap="2" m="b-2">
      <button
        v-for="cate of coreCategories"
        :key="cate"
        class="select-button"
        :class="{ active: category === cate }"
        @click="toggleCategory(cate)"
      >
        {{ cate }}
      </button>
    </div>
    <div opacity="80" text="sm">
      Add-ons
    </div>
    <div flex="~ wrap" gap="2" m="b-2">
      <button
        v-for="cate of addonCategories"
        :key="cate"
        class="select-button"
        :class="{ active: category === cate }"
        @click="toggleCategory(cate)"
      >
        {{ cate.slice(1) }}
      </button>
    </div>
    <div opacity="80" text="sm">
      Sort by
    </div>
    <div flex="~ wrap" gap="2" m="b-2">
      <button v-if="search" class="select-button active">
        Search
      </button>
      <button
        v-for="method of sortMethods"
        :key="method"
        class="select-button capitalize"
        :class="{
          active: method === (sortMethod || 'category'),
          disabled: search
        }"
        @click="toggleSort(method)"
      >
        {{ method }}
      </button>
    </div>
    <div opacity="80" text="sm">
      Filters
    </div>
    <div flex="~">
      <label class="checkbox">
        <input v-model="hasComponent" type="checkbox">
        <span>Has Component</span>
      </label>
    </div>
  </div>
  <div h="1px" bg="$vt-c-divider-light" m="t-4" />
  <div flex="~" class="children:my-auto" p="2">
    <carbon-search m="r-2" opacity="50" />
    <input v-model="search" class="w-full" type="text" role="search" placeholder="Search...">
  </div>
  <div h="1px" bg="$vt-c-divider-light" m="b-4" />
  <div flex="~ col" gap="2" class="relative" p="t-5">
    <div v-if="hasFilters" class="transition mb-2 opacity-60 absolute -top-3 right-0 z-10">
      <button class="select-button flex gap-1 items-center !px-2 !py-1" @click="resetFilters()">
        <CarbonFilterRemove />
        Clear Filters
      </button>
    </div>
    <template v-for="(fn, idx) of result" :key="fn.name">
      <h3
        v-if="showCategory && fn.category !== result[idx - 1]?.category"
        opacity="60"
        text="!16px"
        tracking="!wide"
        m="!0"
        p="y-2"
      >
        {{ fn.category }}
      </h3>
      <FunctionBadge :fn="fn" />
    </template>
  </div>
  <div h="1px" bg="$vt-c-divider-light" m="t-4" />
</template>

<style scoped lang="postcss">
input {
  --tw-ring-offset-width: 1px !important;
  --tw-ring-color: #8885 !important;
  --tw-ring-offset-color: transparent !important;
}

.checkbox {
  @apply inline-flex items-center my-auto cursor-pointer select-none;

  input {
    appearance: none;
    padding: 0;
    -webkit-print-color-adjust: exact;
    color-adjust: exact;
    display: inline-block;
    vertical-align: middle;
    background-origin: border-box;
    user-select: none;
    flex-shrink: 0;
    height: 1rem;
    width: 1rem;
    @apply bg-gray-400/30;
    @apply rounded-md h-4 w-4 select-none;
  }

  input:checked {
    background-image: url("data:image/svg+xml,%3csvg viewBox='0 0 16 16' fill='white' xmlns='http://www.w3.org/2000/svg'%3e%3cpath d='M12.207 4.793a1 1 0 010 1.414l-5 5a1 1 0 01-1.414 0l-2-2a1 1 0 011.414-1.414L6.5 9.086l4.293-4.293a1 1 0 011.414 0z'/%3e%3c/svg%3e");
  }

  span {
    @apply ml-1.5 text-13px opacity-70;
  }
}

.select-button {
  @apply rounded text-sm px-2 py-0.5 bg-gray-400/5 hover:bg-gray-400/10;
}
.select-button.active:not(.disabled) {
  @apply text-primary bg-primary/5;
}
.select-button.disabled {
  @apply opacity-50 pointer-events-none;
}
</style>
