<template>
  <div class="map" />
</template>

<script>
import {
  Group,
  Mesh,
  NearestFilter,
  PerspectiveCamera,
  PlaneBufferGeometry,
  RepeatWrapping,
  Scene,
  ShaderMaterial,
  Vector3,
  WebGLRenderer
} from 'three'
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer'
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass'
import { mapState } from 'vuex'
import MapControls from '@/components/map/MapControls'
import Highlight from '@/components/map/Highlight'
import fragmentShader from '@/components/map/mapFragmentShader'
import textFragmentShader from '@/components/map/mapTextFragmentShader'
import ShatteringPass from '@/components/map/ShatteringPass'
import TextureManager from '@/components/map/TextureManager'
import { clamp01 } from '@/utils'

export default {
  name: 'Map',
  props: {
    transitions: {
      type: Boolean
    }
  },
  data () {
    return {
      transitionValue: 0,
      transitionDirection: 0,
      textHoverProgress: 0,
      textActiveProgress: 0
    }
  },
  computed: {
    ...mapState(['activeLocation']),
    activeEvent () {
      return this.transitions ? this.$store.state.activeEvent : null
    }
  },
  watch: {
    activeEvent (event, oldEvent) {
      this.onEventChanged(event, oldEvent)
    }
  },
  mounted () {
    this.renderer = new WebGLRenderer({ antialias: false, alpha: true })
    this.renderer.setPixelRatio(window.devicePixelRatio)
    this.renderer.setSize(window.innerWidth, window.innerHeight)
    this.renderer.sortObjects = false

    this.textureManager = new TextureManager(this.renderer)

    this.composer = new EffectComposer(this.renderer)

    this.loadTextures()
      .then(this.setupScene)
      .then(() => {
        this.$el.appendChild(this.renderer.domElement)

        this.update()

        this.$emit('ready')

        this.onEventChanged(this.activeEvent, null)
      })
  },
  destroyed () {
    this.renderer.dispose()
    cancelAnimationFrame(this.latestAnimationFrame)
  },
  methods: {
    loadTextures () {
      const textures = {
        map_bg: {},
        map: { hqAvailable: true },
        shadesmar_map_bg: {},
        transition: {},
        text_pattern: {},
        map_text: { hqAvailable: true },
        shadesmar_map_text: { hqAvailable: true }
      }

      return this.textureManager.load(textures)
    },
    async setupScene (textures) {
      this.camera = new PerspectiveCamera(
        60,
        window.innerWidth / window.innerHeight,
        0.01,
        1e3
      )
      this.camera.position.set(30, -10, 40)

      this.controls = new MapControls(this.camera, this.renderer.domElement)
      this.controls.addEventListener('click', ({ position }) => {
        console.log(position.x + 512, 256 - position.y)
        if (this.transitionValue === 0) {
          this.textActiveProgress = 1
          this.$store.commit('selectLocation', this.queryHover(position.x, position.y))
        }
      })

      this.highlights = new Group()

      const geo = new PlaneBufferGeometry(2, 2, 1, 1)
      this.mapMaterial = new ShaderMaterial({
        // language=GLSL
        vertexShader: `
          varying vec2 vUv;

          void main() {
            vUv = uv;

            gl_Position = projectionMatrix * modelViewMatrix * vec4(position * vec3(512, 256, 1.0), 1.0);
          }
        `,
        fragmentShader,
        uniforms: {
          BgTexture: { value: textures.map_bg },
          OutlineTexture: { value: textures.map },
          ShadesmarBgTexture: { value: textures.shadesmar_map_bg },
          TransitionTexture: { value: textures.transition },
          Transition: { value: this.transitionValue }
        }
      })

      const textPattern = textures.text_pattern
      textPattern.wrapS = RepeatWrapping
      textPattern.wrapT = RepeatWrapping
      textPattern.magFilter = NearestFilter

      const textMaterial = new ShaderMaterial({
        // language=GLSL
        vertexShader: `
          varying vec2 vUv;

          void main() {
            vUv = uv;

            gl_Position = projectionMatrix * modelViewMatrix * vec4(position * vec3(512, 256, 1.0), 1.0);
          }
        `,
        fragmentShader: textFragmentShader,
        uniforms: {
          Texture: { value: textures.map_text },
          ShadesmarTexture: { value: textures.shadesmar_map_text },
          PatternTexture: { value: textPattern },
          TransitionTexture: { value: textures.transition },
          Transition: { value: this.transitionValue },
          HoveredItem: { value: 0 },
          ActiveItem: { value: 0 },
          HoverProgress: { value: 0 },
          ActiveProgress: { value: 0 }
        },
        transparent: true,
        depthTest: false
      })
      this.plane = new Mesh(geo, this.mapMaterial)
      this.plane.frustumCulled = false

      this.textPlane = new Mesh(geo, textMaterial)
      this.textPlane.position.z = 1
      this.textPlane.frustumCulled = false

      this.scene = new Scene()
      this.scene.add(this.plane, this.textPlane, this.highlights)

      this.composer.addPass(new RenderPass(this.scene, this.camera))
      this.shatteringPass = new ShatteringPass()
      this.composer.addPass(this.shatteringPass)

      this.hoverTexture = await this.textureManager.loadData('hover_text', false, 'b')
    },
    onEventChanged (event, oldEvent) {
      this.highlights.children.forEach(h => h.leave())
      this.transitionDirection = event !== null && event.shadesmar ? 1 : -2

      if (event !== null && event.specialEffect === 'shattering') {
        this.shatteringPass.enter()
      } else if (oldEvent !== null && oldEvent.specialEffect === 'shattering') {
        this.shatteringPass.leave()
      }

      if (event === null) {
        return
      }

      const newPosition = event.coordinates
      const target = new Vector3(newPosition.x - 512, 256 - newPosition.y, 0)
      this.highlights.add(new Highlight(target.x, target.y, event.specialEffect === 'shattering' ? 2 : undefined))
      this.controls.transitionTo(target, newPosition.zoom !== undefined ? newPosition.zoom : 0.7)
    },
    update (timestamp) {
      this.resizeCanvasToDisplaySize()
      this.transitionValue += this.transitionDirection * 0.01

      if (this.transitionValue <= 0 || this.transitionValue >= 1) {
        this.transitionValue = clamp01(this.transitionValue)
        this.transitionDirection = 0
      }

      this.highlights.children.forEach(h => h.update(this.camera, timestamp))

      this.controls.update()

      this.mapMaterial.uniforms.Transition.value = this.transitionValue
      this.textPlane.material.uniforms.Transition.value = this.transitionValue

      if (this.transitionValue === 0) {
        this.updateTextHighlights()
      }

      this.textPlane.material.uniforms.HoveredItem.value = this.lastHoveredItem !== null ? this.lastHoveredItem : 0
      this.textPlane.material.uniforms.HoverProgress.value = this.textHoverProgress

      this.textPlane.material.uniforms.ActiveItem.value = this.lastActiveLocation !== null ? this.lastActiveLocation : 0
      this.textPlane.material.uniforms.ActiveProgress.value = this.textActiveProgress

      this.composer.render()
      this.latestAnimationFrame = requestAnimationFrame(this.update)
    },
    updateTextHighlights () {
      const hoveredItem = this.queryHover(this.controls.textHoverPosition.x, this.controls.textHoverPosition.y)

      this.textHoverProgress = clamp01(this.textHoverProgress + (hoveredItem !== null ? 0.1 : -0.1))

      if (hoveredItem !== this.lastHoveredItem && hoveredItem !== null) {
        this.textHoverProgress = 0
        this.lastHoveredItem = hoveredItem
      } else if (hoveredItem !== this.lastHoveredItem && hoveredItem === null && this.textHoverProgress === 0) {
        this.lastHoveredItem = null
      }

      if (hoveredItem !== null) {
        document.body.style.cursor = 'pointer'
      } else {
        document.body.style.cursor = 'initial'
      }

      this.textActiveProgress = clamp01(this.textActiveProgress + (this.activeLocation !== null ? 0.0 : -0.1))

      if (this.activeLocation !== this.lastActiveLocation && this.activeLocation !== null) {
        this.textActiveProgress = 1
        this.lastActiveLocation = this.activeLocation
      } else if (this.activeLocation !== this.lastActiveLocation && this.activeLocation === null && this.textActiveProgress === 0) {
        this.lastActiveLocation = null
      }
    },
    queryHover (x, y) {
      if (this.hoverTexture === undefined) {
        return null
      }

      const hoverX = Math.trunc((x + 512) * this.hoverTexture.width / 1024)
      const hoverY = Math.trunc((256 - y) * this.hoverTexture.height / 512)

      const hoveredItem = this.hoverTexture.data[hoverY * this.hoverTexture.width + hoverX]
      if (hoveredItem === undefined || hoveredItem <= 0) {
        return null
      }

      return hoveredItem
    },
    resizeCanvasToDisplaySize () {
      const canvas = this.renderer.domElement
      // look up the size the canvas is being displayed
      const width = canvas.clientWidth
      const height = canvas.clientHeight
      // adjust displayBuffer size to match
      if (canvas.width !== width || canvas.height !== height) {
        // you must pass false here or three.js sadly fights the browser
        this.renderer.setSize(width, height, false)
        this.camera.aspect = width / height
        this.camera.updateProjectionMatrix()
        // update any render target sizes here
      }
    }
  }
}
</script>

<style lang="scss">
.map {
  display: flex;
  align-items: stretch;
  justify-content: stretch;
  position: relative;
  height: 100%;
  flex: 1;
  min-height: 0;

  canvas {
    width: 100% !important;
    height: 100% !important;
    z-index: 0;
    position: relative;

    &:focus {
      outline: none;
    }
  }

  &:after {
    position: absolute;
    content: '';
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    pointer-events: none;
    box-shadow: inset 0 0 60px rgba(0, 0, 0, 0.6);
    z-index: 10;
  }
}
</style>