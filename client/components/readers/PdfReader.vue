<template>
  <div class="relative w-full h-full flex flex-row overflow-hidden bg-primary">

    <button
      v-if="toc && toc.length"
      @click="showSidebar = !showSidebar"
      class="absolute top-4 left-4 z-50 p-3 bg-gray-800 text-white rounded-full shadow-2xl opacity-80 hover:opacity-100 transition-opacity"
    >
      <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"></path>
      </svg>
    </button>

    <div
      v-show="showSidebar"
      class="w-72 h-full bg-gray-900 border-r border-gray-700 overflow-y-auto flex-shrink-0 absolute md:relative z-40 shadow-2xl transition-all"
    >
      <div class="p-6 pt-20">
        <h2 class="text-white text-xl font-semibold mb-6">Contents</h2>
        <ul class="text-gray-300 text-sm space-y-4">
          <li
            v-for="(item, idx) in toc"
            :key="idx"
            @click="goToPage(item.page)"
            class="cursor-pointer hover:text-white hover:underline transition-colors"
            :style="{ paddingLeft: `${item.depth * 16}px` }"
          >
            {{ item.title }}
          </li>
        </ul>
      </div>
    </div>

    <div
      id="pdf-scroll-container"
      class="flex-1 h-full overflow-y-auto flex flex-col items-center py-8"
      @scroll="handleScroll"
    >
      <div v-if="!numPages" class="text-gray-200 mt-20 text-lg">Loading Document...</div>

      <div
        v-for="i in renderedPages"
        :key="i"
        class="pdf-page-wrapper w-full max-w-4xl mb-12 flex flex-col items-center px-4 md:px-0"
        :data-page="i"
      >
        <div class="w-full bg-white shadow-2xl relative">
          <pdf
            :src="pdfDocInitParams"
            :page="i"
            :rotate="rotate"
            @num-pages="numPagesLoaded"
            @loaded="loadedEvt"
            @error="error"
            class="w-full"
          ></pdf>
        </div>

        <div v-if="i < numPages" class="w-2/3 border-b-2 border-gray-400 border-dashed mt-12 opacity-50"></div>
      </div>
    </div>
  </div>
</template>

<script>
import pdf from '@teckel/vue-pdf'

export default {
  components: {
    pdf
  },
  props: {
    libraryItem: {
      type: Object,
      default: () => {}
    },
    playerOpen: Boolean,
    keepProgress: Boolean,
    fileId: String
  },
  data() {
    return {
      windowWidth: 0,
      windowHeight: 0,
      scale: 1,
      rotate: 0,
      loadedRatio: 0,
      page: 1,
      numPages: 0,
      pdfDocInitParams: null,
      isRefreshing: false,
      renderedPages: 1,
      pagesToLoadAtOnce: 3,
      observer: null,
      toc: [],
      showSidebar: false
    }
  },
  watch: {
    numPages(newVal) {
      if (newVal > 0 && this.renderedPages <= 1) {
        this.renderedPages = Math.min(this.pagesToLoadAtOnce, newVal)
        this.$nextTick(() => {
          if (typeof this.setupIntersectionObserver === 'function') {
            this.setupIntersectionObserver()
          }
        })
      }
    }
  },
  computed: {
    userToken() {
      return this.$store.getters['user/getToken']
    },
    libraryItemId() {
      return this.libraryItem?.id
    },
    fitToPageWidth() {
      return this.pdfHeight * 0.6
    },
    pdfWidth() {
      return this.fitToPageWidth * this.scale
    },
    pdfHeight() {
      if (this.windowHeight < 400 || !this.playerOpen) return this.windowHeight - 120
      return this.windowHeight - 284
    },
    maxScale() {
      return Math.floor((this.windowWidth * 10) / this.fitToPageWidth) / 10
    },
    canGoNext() {
      return this.page < this.numPages
    },
    canGoPrev() {
      return this.page > 1
    },
    canScaleUp() {
      return this.scale < this.maxScale
    },
    canScaleDown() {
      return this.scale > 1
    },
    userMediaProgress() {
      if (!this.libraryItemId) return
      return this.$store.getters['user/getUserMediaProgress'](this.libraryItemId)
    },
    savedPage() {
      if (!this.keepProgress) return 0

      // Validate ebookLocation is a number
      if (!this.userMediaProgress?.ebookLocation || isNaN(this.userMediaProgress.ebookLocation)) return 0
      return Number(this.userMediaProgress.ebookLocation)
    },
    ebookUrl() {
      if (this.fileId) {
        return `/api/items/${this.libraryItemId}/ebook/${this.fileId}`
      }
      return `/api/items/${this.libraryItemId}/ebook`
    }
  },
  methods: {
    zoomIn() {
      this.scale += 0.1
    },
    zoomOut() {
      this.scale -= 0.1
    },
    updateProgress() {
      if (!this.keepProgress) return
      if (!this.numPages) {
        console.error('Num pages not loaded')
        return
      }

      const payload = {
        ebookLocation: this.page,
        ebookProgress: Math.max(0, Math.min(1, (Number(this.page) - 1) / Number(this.numPages)))
      }
      this.$axios.$patch(`/api/me/progress/${this.libraryItemId}`, payload, { progress: false }).catch((error) => {
        console.error('EpubReader.updateProgress failed:', error)
      })
    },
    loadedEvt() {
      if (this.savedPage > 0 && this.savedPage <= this.numPages) {
        this.page = this.savedPage
      }
    },
    progressEvt(progress) {
      this.loadedRatio = progress
    },
    numPagesLoaded(e) {
      if (!e) return
      this.numPages = e
    },
    prev() {
      if (this.page <= 1) return
      this.page--
      this.updateProgress()
    },
    next() {
      if (this.page >= this.numPages) return
      this.page++
      this.updateProgress()
    },
    async refreshToken() {
      if (this.isRefreshing) return
      this.isRefreshing = true
      const newAccessToken = await this.$store.dispatch('user/refreshToken').catch((error) => {
        console.error('Failed to refresh token', error)
        return null
      })
      if (!newAccessToken) {
        // Redirect to login on failed refresh
        this.$router.push('/login')
        return
      }

      // Force Vue to re-render the PDF component by creating a new object
      this.pdfDocInitParams = {
        url: this.ebookUrl,
        httpHeaders: {
          Authorization: `Bearer ${newAccessToken}`
        }
      }
      this.isRefreshing = false
    },
    async error(err) {
      if (err && err.status === 401) {
        console.log('Received 401 error, refreshing token')
        await this.refreshToken()
        return
      }
      console.error(err)
    },
    resize() {
      this.windowWidth = window.innerWidth
      this.windowHeight = window.innerHeight
    },
    init() {
      const params = {
        url: this.ebookUrl,
        httpHeaders: {
          Authorization: `Bearer ${this.userToken}`
        }
      }

      this.pdfDocInitParams = pdf.createLoadingTask(params)

      this.pdfDocInitParams.promise.then(doc => {
        this.extractTOC(doc)
      }).catch(err => console.error("PDF Init Error:", err))
    },
    handleScroll(e) {
      const container = e.target
      const bottomDistance = container.scrollHeight - container.scrollTop - container.clientHeight

      // Load the next batch if we are within 2500px of the bottom
      if (bottomDistance < 2500 && this.renderedPages < this.numPages) {
        this.renderedPages = Math.min(this.renderedPages + this.pagesToLoadAtOnce, this.numPages)
        this.$nextTick(() => {
          this.observeNewPages()
        })
      }
    },
    setupIntersectionObserver() {
      const options = {
        root: document.getElementById('pdf-scroll-container'),
        threshold: 0.4 // Triggers when 40% of the page is visible
      }

      this.observer = new IntersectionObserver((entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            const visiblePage = parseInt(entry.target.getAttribute('data-page'))

            // Sync with AudioBookShelf's native page tracker
            if (visiblePage && visiblePage !== this.page) {
              this.page = visiblePage
            }
          }
        })
      }, options)

      this.observeNewPages()
    },
    observeNewPages() {
      const wrappers = this.$el.querySelectorAll('.pdf-page-wrapper:not(.observed)')
      wrappers.forEach(wrapper => {
        if (this.observer) this.observer.observe(wrapper)
        wrapper.classList.add('observed')
      })
    },
    async extractTOC(doc) {
      try {
        const outline = await doc.getOutline()
        if (!outline) return

        // Recursive function to flatten nested chapters
        const parseItems = async (items, depth = 0) => {
          let result = []
          for (const item of items) {
            let pageIndex = -1
            if (item.dest) {
              try {
                // PDF destinations can be a string reference or an array
                const destArray = typeof item.dest === 'string' ? await doc.getDestination(item.dest) : item.dest
                if (destArray && destArray[0]) {
                  pageIndex = await doc.getPageIndex(destArray[0])
                }
              } catch (e) {
                console.warn('Could not resolve destination for:', item.title)
              }
            }

            if (pageIndex >= 0) {
              result.push({ title: item.title, page: pageIndex + 1, depth })
            }

            // If this chapter has sub-chapterrs, parse them too
            if (item.items && item.items.length > 0) {
              result = result.concat(await parseItems(item.items, depth + 1))
            }
          }
          return result
        }

        this.toc = await parseItems(outline)
      } catch (error) {
        console.error('Failed to parse PDF Outline', error)
      }
    },
    goToPage(pageNum) {
      // If the target page isn't rendered yet, increase renderedPages to force it to draw
      if (this.renderedPages < pageNum) {
        this.renderedPages = Math.min(pageNum + 1, this.numPages)
      }

      this.$nextTick(() => {
        // Track the newly rendered pages
        this.observeNewPages()

        // Find the specific page wrapper and glide to it
        const targetEl = this.$el.querySelector('.pdf-page-wrapper[data-page="${pageNum}"]')
        if (targetEl) {
          targetEl.scrollIntoView({ behavior: 'smooth' })
        }

        // On mobile screens, automatically close the sidebar after clicking
        if (this.windowWidth < 768) {
          this.showSidebar = false
        }
      })
    }
  },
  mounted() {
    this.windowWidth = window.innerWidth
    this.windowHeight = window.innerHeight
    window.addEventListener('resize', this.resize)

    this.init()
  },
  beforeDestroy() {
    if (this.observer) this.observer.disconnect()
    //window.removeEventListener('resize', this.resize)
  }
}
</script>

<style scoped>
#pdf-scroll-container {
  scroll-behavior: smooth;
  -webkit-overflow-scrolling: touch;
}

::v-deep .annotationLayer {
  display: none !important;
}
</style>
