<template>
  <KeepAlive exclude="BarUpdatePanel">
    <Suspense>
      <component
        :is="panel"
        v-bind="panelProps"
        v-on="panelEvents"
      />

      <template #fallback>
        Loading bookmark bars...
      </template>
    </Suspense>
  </KeepAlive>
</template>

<script>
import { computed, ref, shallowRef } from 'vue'
import BarsList from '@/components/BarsList.vue'
import BarUpdatePanel from '@/components/BarUpdatePanel.vue'
import EmojiPicker from '@/components/EmojiPicker.vue'
import { useBookmarkBars } from '@/composables/useBookmarks'
import { useBrowserStorage } from '@/composables/useBrowserStorage'
import { updatePopupIcon } from '@/composables/usePopupIcon'

export default {
  name: 'Popup',
  components: { BarsList, BarUpdatePanel },
  async setup () {
    const { bars: bookmarkBars, currentBar, createBar, deleteBar } = await useBookmarkBars()
    const { useBrowserStorageKey } = useBrowserStorage()
    const excludedBookmarkIds = await useBrowserStorageKey('pinnedBookmarks')
    updatePopupIcon(computed(() => currentBar.value.icon))

    const panel = shallowRef(BarsList)
    const panelProps = ref({})
    const panelEvents = ref({})

    function editing (bar) {
      panelProps.value = {
        title: bar.title,
        icon: bar.icon,
        bookmarks: bar.bookmarks,
        isActive: bar.id === currentBar.value.id,
      }
      panelEvents.value = {
        'update:title': (newTitle) => { bar.title = newTitle },
        icon: () => emoji(bar),
        submit: listing,
        cancel: listing,
        remove: () => { console.log('removing', bar.id) },
        pin: (bookmarkId) => {
          if (excludedBookmarkIds.value?.some(excluded => excluded === bookmarkId)) {
            excludedBookmarkIds.value = excludedBookmarkIds.value.filter(excluded => excluded !== bookmarkId)
          } else {
            excludedBookmarkIds.value = [...new Set([...(excludedBookmarkIds.value || []), bookmarkId])]
          }
        },
      }
      panel.value = BarUpdatePanel
    }

    function listing () {
      panelProps.value = {
        bookmarkBars,
        currentBarId: computed(() => currentBar.value.id),
      }
      panelEvents.value = {
        create: createBar,
        remove: deleteBar,
        switch: switchBar,
        edit: editing,
      }
      panel.value = BarsList
    }

    function emoji (bar) {
      panelProps.value = {
        modelValue: bar.icon,
      }
      panelEvents.value = {
        'update:modelValue': (newIcon) => {
          console.log('new icon', newIcon)
          bar.icon = newIcon
          editing(bar)
        },
      }
      panel.value = EmojiPicker
    }

    function switchBar (bar) {
      currentBar.value = bar
    }

    listing()

    return {
      panel,
      panelProps,
      panelEvents,
      editing,
      listing,
      emoji,
      currentBar,
      bookmarkBars,
      createBar,
      deleteBar,
      switchBar,
    }
  },
}
</script>

<style>
body {
  margin: 0;
  background-color: #444444;
}

body #app {
  min-width: 240px;
}

svg {
  fill: white;
}

.row {
  display: flex;
  justify-content: flex-end;
  height: 32px;
}

.flex {
  flex: 1 0 auto;
}

input {
  padding-left: 10px;
}
button, input {
  background-color: #222;
  color: white;
  border: 1px solid #444;
  outline: none;
}
button:disabled {
  color: #555;
}
button:disabled svg {
  fill: #555;
}
button:enabled {
  cursor: pointer;
}
button:hover:enabled {
  background-color: #333;
}

button.btn-icon {
  display: grid;
  place-content: center;
  padding: 0 5px;
}
</style>