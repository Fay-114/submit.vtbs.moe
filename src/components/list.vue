<template>
<div class="index" ref="container">
  <div class="indexContainer" :style="{ height: fileList.length * height + 'px' }">
    <vtb v-for="({i=0, file},n) in renderedList" :key="`vtb_${n}`" :i="i" :file="file" :n="n"></vtb>
  </div>
</div>
</template>

<script>
import { mapState, mapActions } from 'vuex'

import vtb, { height, eventEmitter } from './list/vtb'

const RENDER_LENGTH = 100 * 2

export default {
  data() {
    return {
      height,
      renderTop: 0,
      lastIntersect: 0,
      renderedList: Array(RENDER_LENGTH).fill({}),
      observer: undefined
    }
  },
  async mounted() {
    await this.loadFileList()
    this.observer = new IntersectionObserver(entries => entries.forEach(({ isIntersecting, target }) => {
      if (isIntersecting) {
        const i = Number(target.getAttribute('i'))
        const n = target.getAttribute('n')
        eventEmitter.emit(n)
        this.lastIntersect = i
      }
    }), { root: this.$refs.container, thresholds: [0] })
    await this.$nextTick()
    const vtbs = document.getElementsByClassName('vtb')
    Array(vtbs.length).fill().map((_, i) => vtbs[i]).map(vtb => this.observer.observe(vtb))
  },
  methods: {
    render(start, length) {
      Array(length).fill()
        .map((_, i) => i + start)
        .map(i => {
          const domIndex = i % RENDER_LENGTH
          const file = this.fileList[i]
          this.renderedList[domIndex] = { i, file }
        })
      this.renderedList = [...this.renderedList]
    },
    ...mapActions(['loadFileList'])
  },
  watch: {
    async fileList() {
      this.render(this.renderTop, RENDER_LENGTH)
      const top = Math.max(0, this.lastIntersect)
      await this.$nextTick()
      Array(10).fill().map((_, i) => i + top - 5).map(String).forEach(n => eventEmitter.emit(n))
    },
    renderTop(newVal, oldVal) {
      if (newVal > oldVal) {
        this.render(oldVal + RENDER_LENGTH, newVal - oldVal)
      } else if (oldVal > newVal) {
        this.render(newVal, oldVal - newVal)
      }
    },
    lastIntersect() {
      const { lastIntersect } = this
      const margin = lastIntersect - this.renderTop
      if (margin > RENDER_LENGTH / 2 || margin < RENDER_LENGTH / 2) {
        this.renderTop = Math.min(Math.max(lastIntersect - RENDER_LENGTH / 2, 0), this.fileList.length - RENDER_LENGTH)
      }
    }
  },
  beforeDestroy() {
    this.observer.disconnect()
  },
  components: {
    vtb
  },
  computed: mapState(['fileList'])
}
</script>

<style scoped>
.index {
  height: calc(100vh - 50px);
  overflow-y: auto;
  overflow-x: hidden;
}

.indexContainer {
  position: relative;
}
</style>