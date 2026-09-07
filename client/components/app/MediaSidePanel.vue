<template>
  <div id="media-side-panel" :class="{ collapsed: isCollapsed }" :style="{ flexBasis: isCollapsed ? '0px' : width + 'px' }">
    <div id="media-side-panel-toggle" :style="{ right: isCollapsed ? '0px' : width + 'px' }" @click="toggleCollapsed">
      <span>{{ isCollapsed ? '&#8249;' : '&#8250;' }}</span>
    </div>
    <div id="media-side-panel-divider" @mousedown="startDrag"></div>
    <div id="media-side-panel-inner" v-show="!isCollapsed">
      <div class="p-3 border-b border-white border-opacity-10">
        <form class="flex gap-1.5" @submit.prevent="loadFromInput">
          <input v-model="linkInput" type="text" placeholder="Paste a YouTube link" autocomplete="off" class="flex-1 min-w-0 bg-bg text-white border border-white border-opacity-10 rounded px-2 py-1.5 text-sm" />
          <button type="submit" class="bg-success text-black rounded px-3 text-sm font-semibold">Load</button>
        </form>
        <p class="text-xs text-gray-400 mt-1.5 leading-tight">Paste a YouTube video or playlist link.</p>
      </div>
      <div class="flex-1 overflow-y-auto p-3">
        <iframe v-if="embedUrl" :key="embedUrl" :src="embedUrl" loading="lazy" class="w-full border-0 rounded-xl" :style="iframeStyle" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
        <p v-else class="text-gray-400 text-sm text-center mt-8">No YouTube link loaded yet.</p>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      linkInput: '',
      embedUrl: null,
      isCollapsed: false,
      width: 320,
      dragging: false
    }
  },
  computed: {
    iframeStyle() {
      return { aspectRatio: '16 / 9' }
    }
  },
  methods: {
    linkKey() {
      return 'mediaSidePanel.link.youtube'
    },
    toYouTubeEmbedUrl(raw) {
      try {
        const url = new URL(raw.trim())
        const host = url.hostname.replace(/^www\./, '').replace(/^music\./, '')
        if (host === 'youtu.be') {
          const id = url.pathname.split('/').filter(Boolean)[0]
          return id ? `https://www.youtube.com/embed/${id}` : null
        }
        if (host !== 'youtube.com') return null
        if (url.pathname.indexOf('/embed/') === 0) {
          return `https://www.youtube.com${url.pathname}${url.search}`
        }
        const v = url.searchParams.get('v')
        const list = url.searchParams.get('list')
        if (url.pathname === '/playlist' && list) {
          return `https://www.youtube.com/embed/videoseries?list=${list}`
        }
        if (v) {
          return `https://www.youtube.com/embed/${v}${list ? '?list=' + list : ''}`
        }
        if (list) {
          return `https://www.youtube.com/embed/videoseries?list=${list}`
        }
        return null
      } catch (e) {
        return null
      }
    },
    getSavedLink() {
      try {
        return localStorage.getItem(this.linkKey())
      } catch (e) {
        return null
      }
    },
    loadFromInput() {
      const raw = this.linkInput
      if (!raw || !raw.trim()) return
      const embedUrl = this.toYouTubeEmbedUrl(raw)
      if (!embedUrl) {
        this.$toast.error('Could not read a playable link from that YouTube URL.')
        return
      }
      try {
        localStorage.setItem(this.linkKey(), raw.trim())
      } catch (e) {}
      this.embedUrl = embedUrl
    },
    toggleCollapsed() {
      this.isCollapsed = !this.isCollapsed
      try {
        localStorage.setItem('mediaSidePanel.collapsed', this.isCollapsed ? '1' : '0')
      } catch (e) {}
    },
    startDrag(e) {
      e.preventDefault()
      this.dragging = true
      document.body.classList.add('media-side-panel-resizing')
      window.addEventListener('mousemove', this.onDrag)
      window.addEventListener('mouseup', this.stopDrag)
    },
    onDrag(e) {
      if (!this.dragging || this.isCollapsed) return
      let newWidth = window.innerWidth - e.clientX
      if (newWidth < 240) newWidth = 240
      if (newWidth > window.innerWidth * 0.92) newWidth = window.innerWidth * 0.92
      this.width = newWidth
    },
    stopDrag() {
      if (this.dragging) {
        try {
          localStorage.setItem('mediaSidePanel.width', this.width)
        } catch (e) {}
      }
      this.dragging = false
      document.body.classList.remove('media-side-panel-resizing')
      window.removeEventListener('mousemove', this.onDrag)
      window.removeEventListener('mouseup', this.stopDrag)
    }
  },
  mounted() {
    try {
      const savedWidth = localStorage.getItem('mediaSidePanel.width')
      if (savedWidth) this.width = parseInt(savedWidth, 10)
      this.isCollapsed = localStorage.getItem('mediaSidePanel.collapsed') === '1'
    } catch (e) {}
    const saved = this.getSavedLink()
    this.linkInput = saved || ''
    this.embedUrl = saved ? this.toYouTubeEmbedUrl(saved) : null
  },
  beforeDestroy() {
    window.removeEventListener('mousemove', this.onDrag)
    window.removeEventListener('mouseup', this.stopDrag)
  }
}
</script>

<style>
#media-side-panel {
  position: relative;
  flex: 0 0 auto;
  min-width: 0;
  max-width: 92vw;
  display: flex;
  background: #1b1e24;
  border-left: 1px solid rgba(255, 255, 255, 0.1);
  overflow: hidden;
  transition: flex-basis 0.15s ease;
  height: 100%;
}
#media-side-panel.collapsed {
  border-left: 0;
}
body.media-side-panel-resizing #media-side-panel {
  transition: none;
}
body.media-side-panel-resizing iframe {
  pointer-events: none;
}
#media-side-panel-inner {
  display: flex;
  flex-direction: column;
  width: 100%;
  min-width: 240px;
}
#media-side-panel-divider {
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 6px;
  cursor: col-resize;
  z-index: 2;
}
#media-side-panel-divider:hover {
  background: rgba(29, 185, 84, 0.4);
}
#media-side-panel-toggle {
  position: fixed;
  top: 50%;
  transform: translate(50%, -50%);
  width: 22px;
  height: 56px;
  background: #1b1e24;
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 6px;
  color: #9aa0aa;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 12px;
  z-index: 40;
  user-select: none;
}
#media-side-panel-toggle:hover {
  color: white;
  border-color: #1db954;
}
</style>
