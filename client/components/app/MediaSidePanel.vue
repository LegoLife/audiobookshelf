<template>
  <div id="media-side-panel" :class="{ collapsed: isCollapsed }" :style="{ flexBasis: isCollapsed ? '0px' : width + 'px' }">
    <div id="media-side-panel-toggle" :style="{ right: isCollapsed ? '0px' : width + 'px' }" @click="toggleCollapsed">
      <span>{{ isCollapsed ? '&#8249;' : '&#8250;' }}</span>
    </div>
    <div id="media-side-panel-divider" @mousedown="startDrag"></div>
    <div id="media-side-panel-inner" v-show="!isCollapsed">
      <div class="p-3 border-b border-white border-opacity-10">
        <div class="flex gap-1.5 mb-2">
          <button v-for="src in sources" :key="src" type="button" class="flex-1 text-xs font-semibold uppercase tracking-wide rounded py-1.5 border" :class="activeSource === src ? 'text-black border-transparent' : 'text-gray-400 border-white border-opacity-10 bg-primary'" :style="activeSource === src ? { background: src === 'spotify' ? '#1db954' : '#ff0000' } : {}" @click="setActiveSource(src)">
            {{ src }}
          </button>
        </div>
        <form class="flex gap-1.5" @submit.prevent="loadFromInput">
          <input v-model="linkInput" type="text" placeholder="Paste a Spotify or YouTube link" autocomplete="off" class="flex-1 min-w-0 bg-bg text-white border border-white border-opacity-10 rounded px-2 py-1.5 text-sm" />
          <button type="submit" class="bg-success text-black rounded px-3 text-sm font-semibold">Load</button>
        </form>
        <p class="text-xs text-gray-400 mt-1.5 leading-tight">Paste a Spotify playlist/album/track link, or a YouTube video/playlist link.</p>
        <div v-if="activeSource === 'spotify'" class="flex items-center gap-2 mt-2">
          <button type="button" class="text-xs font-semibold text-black rounded px-2.5 py-1" style="background: #1db954" @click="openSpotifyLogin">Log in to Spotify &#8599;</button>
          <button type="button" class="text-xs text-gray-400 underline" @click="reloadEmbed">Refresh player</button>
        </div>
      </div>
      <div class="flex-1 overflow-y-auto p-3">
        <p v-if="activeSource === 'spotify'" class="text-xs text-gray-400 mb-2 leading-tight">Full playlist playback requires being logged into Spotify. Use "Log in to Spotify" above, sign in in the tab that opens, then come back here and hit "Refresh player".</p>
        <iframe v-if="embedUrl" :key="embedReloadKey + embedUrl" :src="embedUrl" loading="lazy" class="w-full border-0 rounded-xl" :style="iframeStyle" :allow="iframeAllow" :allowfullscreen="activeSource === 'youtube'"></iframe>
        <p v-else class="text-gray-400 text-sm text-center mt-8">No {{ activeSource === 'youtube' ? 'YouTube' : 'Spotify' }} link loaded yet.</p>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      sources: ['spotify', 'youtube'],
      activeSource: 'spotify',
      linkInput: '',
      embedUrl: null,
      isCollapsed: false,
      width: 320,
      dragging: false,
      embedReloadKey: 0
    }
  },
  computed: {
    iframeStyle() {
      return this.activeSource === 'youtube' ? { aspectRatio: '16 / 9' } : { height: '600px' }
    },
    iframeAllow() {
      return this.activeSource === 'youtube' ? 'accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share' : 'autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture'
    }
  },
  methods: {
    linkKey(source) {
      return `mediaSidePanel.link.${source}`
    },
    toSpotifyEmbedUrl(raw) {
      try {
        const url = new URL(raw.trim())
        const host = url.hostname.replace(/^open\./, '')
        if (host !== 'spotify.com') return null
        const parts = url.pathname.split('/').filter(Boolean)
        if (parts[0] && parts[0].indexOf('intl-') === 0) parts.shift()
        if (parts.length < 2) return null
        const type = parts[0]
        const id = parts[1].split('?')[0]
        const allowed = ['playlist', 'album', 'track', 'show', 'episode', 'artist']
        if (allowed.indexOf(type) === -1) return null
        return `https://open.spotify.com/embed/${type}/${id}?utm_source=generator`
      } catch (e) {
        return null
      }
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
    detectSource(raw) {
      try {
        const url = new URL(raw.trim())
        const host = url.hostname.replace(/^open\./, '').replace(/^www\./, '').replace(/^music\./, '')
        if (host === 'spotify.com') return 'spotify'
        if (host === 'youtube.com' || host === 'youtu.be') return 'youtube'
        return null
      } catch (e) {
        return null
      }
    },
    toEmbedUrl(source, raw) {
      return source === 'spotify' ? this.toSpotifyEmbedUrl(raw) : this.toYouTubeEmbedUrl(raw)
    },
    setActiveSource(source) {
      this.activeSource = source
      try {
        localStorage.setItem('mediaSidePanel.activeSource', source)
      } catch (e) {}
      const saved = this.getSavedLink(source)
      this.linkInput = saved || ''
      this.embedUrl = saved ? this.toEmbedUrl(source, saved) : null
    },
    getSavedLink(source) {
      try {
        return localStorage.getItem(this.linkKey(source))
      } catch (e) {
        return null
      }
    },
    loadFromInput() {
      const raw = this.linkInput
      if (!raw || !raw.trim()) return
      const source = this.detectSource(raw)
      if (!source) {
        this.$toast.error('That does not look like a Spotify or YouTube link.')
        return
      }
      const embedUrl = this.toEmbedUrl(source, raw)
      if (!embedUrl) {
        this.$toast.error(`Could not read a playable link from that ${source === 'youtube' ? 'YouTube' : 'Spotify'} URL.`)
        return
      }
      try {
        localStorage.setItem(this.linkKey(source), raw.trim())
      } catch (e) {}
      this.activeSource = source
      try {
        localStorage.setItem('mediaSidePanel.activeSource', source)
      } catch (e) {}
      this.embedUrl = embedUrl
    },
    openSpotifyLogin() {
      window.open('https://accounts.spotify.com/login?continue=https://open.spotify.com/', '_blank', 'noopener')
    },
    reloadEmbed() {
      this.embedReloadKey++
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
    const initialSource = (() => {
      try {
        return localStorage.getItem('mediaSidePanel.activeSource') || 'spotify'
      } catch (e) {
        return 'spotify'
      }
    })()
    this.setActiveSource(initialSource)
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
