<template>
  <div ref="container" class="vis-container" :style="{ width: widthStyle, height: heightStyle }">
    <div class="vis-root" ref="vis"/>
  </div>
</template>

<script>
import updateNode from './updateNode'
import vis from 'vis'
import { items as theme } from '@/theme'

import controllerImg from '@/assets/network/controller.svg'
import hostImg from '@/assets/network/host.svg'
import portImg from '@/assets/network/port.svg'
import switchImg from '@/assets/network/switch.svg'

export default {
  name: 'VisCanvas',
  data: () => ({
    width: null,
    height: null
  }),
  computed: {
    data () {
      return this.$store.state.data
    },
    widthStyle () {
      return this.width == null
        ? undefined
        : `${this.width}px`
    },
    heightStyle () {
      return this.height == null
        ? undefined
        : `${this.height}px`
    }
  },
  methods: {
    toBlob (scale) {
      return new Promise(resolve => {
        scale = scale || 2

        const beforeDrawingHandler = ctx => {
          const { x, y } = this.net.view.targetTranslation
          const scale = this.net.view.targetScale
          ctx.fillStyle = '#fff'
          ctx.fillRect(
            -x / scale - 1,
            -y / scale - 1,
            ctx.canvas.width / scale + 2,
            ctx.canvas.height / scale + 2
          )
        }
        const afterDrawingHandler = async (ctx) => {
          this.net.off('beforeDrawing', beforeDrawingHandler)
          this.net.off('afterDrawing', afterDrawingHandler)

          const blob = await new Promise(resolve => {
            ctx.canvas.toBlob(resolve, 'image/png')
          })

          this.width = null
          this.height = null

          resolve(blob)
        }
        const resizeHandler = () => {
          this.net.off('resize', resizeHandler)

          this.net.fit({ animation: false })
          this.net.moveTo({ scale, animation: false })

          this.net.on('beforeDrawing', beforeDrawingHandler)
          this.net.on('afterDrawing', afterDrawingHandler)
          this.net.redraw()
        }

        const coords = this.nodes.get().reduce((acc, { x, y }) => {
          if (x < acc.sX) {
            acc.sX = x
          } else if (x > acc.eX) {
            acc.eX = x
          }
          if (y < acc.sY) {
            acc.sY = y
          } else if (y > acc.eY) {
            acc.eY = y
          }

          return acc
        }, { sX: 0, eX: 0, sY: 0, eY: 0 })
        this.net.on('resize', resizeHandler)
        this.width = (coords.eX - coords.sX) * scale + 200 * scale
        this.height = (coords.eY - coords.sY) * scale + 200 * scale
      })
    },
    isEdge (type) {
      return type === 'link' || type === 'association'
    },
    buildGroupColor (primary, bg, alwaysBorder) {
      bg = bg || 'rgba(0, 0, 0, 0)'
      return {
        background: bg,
        border: primary,
        highlight: {
          background: bg,
          border: primary
        },
        hover: {
          background: bg,
          border: primary
        }
      }
    }
  },
  mounted () {
    const container = this.$refs.container
    const options = {
      physics: {
        enabled: false
      },
      nodes: {
        borderWidth: 0.0001,
        borderWidthSelected: 2,
        shapeProperties: {
          borderRadius: 6,
          useBorderWithImage: true
        }
      },
      edges: {
        smooth: false
      },
      interaction: {
        hover: true
      },
      manipulation: {
        enabled: false
      },
      groups: {
        controller: {
          shape: 'image',
          color: this.buildGroupColor(theme.controller),
          size: 25,
          image: controllerImg
        },
        dummy: {
          shape: 'box',
          color: this.buildGroupColor(theme.dummy, '#fff', true),
          font: { color: theme.dummy },
          borderWidth: 1
        },
        host: {
          shape: 'image',
          color: this.buildGroupColor(theme.host),
          size: 25,
          image: hostImg
        },
        port: {
          shape: 'image',
          color: this.buildGroupColor(theme.port),
          size: 10,
          image: portImg
        },
        switch: {
          shape: 'image',
          color: this.buildGroupColor(theme.switch),
          size: 25,
          image: switchImg
        }
      }
    }

    // Preprocess items
    const items = Object.keys(this.data.items)
      .map(id => {
        const node = JSON.parse(JSON.stringify(this.data.items[id]))
        node.id = id
        return node
      })

    // Create an array with nodes
    const nodes = new vis.DataSet(
      items
        .filter(({ type }) => !this.isEdge(type))
        .map(item => updateNode({
          id: item.id,
          group: item.type,
          x: item.x,
          y: item.y
        }, item))
    )

    // Create an array with edges
    const edges = new vis.DataSet(
      items
        .filter(({ type }) => this.isEdge(type))
        .map(item => updateNode({
          id: item.id,
          from: item.from,
          to: item.to
        }, item))
    )

    // Create the network
    const net = new vis.Network(this.$refs.vis, { nodes, edges }, options)

    this.net = net
    this.nodes = nodes
    this.edges = edges
    this.$emit('ready', { container, net, nodes, edges })
  }
}
</script>

<style scoped>
.vis-container {position: relative; width: 100%; height: 100%;}
.vis-container > * {position: absolute; top: 0px; right: 0px; bottom: 0px; left: 0px;}
</style>