<template>
    <div>
        <v-row>
            <v-col cols="12" md="9">
                <v-card>
                    <v-card-text class="pa-0 position-relative">
                        <div
                            ref="canvasContainer"
                            class="draw-canvas-container"
                            tabindex="0"
                            @keydown="onKeyDown"
                            @keyup="onKeyUp">
                            <canvas ref="canvas" />
                            <div v-if="!running" class="draw-overlay d-flex align-center justify-center">
                                <span class="text-h6 white--text">{{ $t('Draw.KeyHints') }}</span>
                            </div>
                        </div>
                    </v-card-text>
                </v-card>
            </v-col>
            <v-col cols="12" md="3">
                <v-card class="mb-4">
                    <v-card-title>{{ $t('Draw.Controls') }}</v-card-title>
                    <v-card-text>
                        <v-btn
                            color="primary"
                            block
                            :disabled="running || !klippyIsConnected"
                            class="mb-2"
                            @click="start">
                            {{ $t('Draw.Start') }}
                        </v-btn>
                        <v-btn color="error" block :disabled="!running" class="mb-4" @click="stop">
                            {{ $t('Draw.Stop') }}
                        </v-btn>
                        <v-divider class="mb-4" />
                        <div class="text-subtitle-2 mb-2">{{ $t('Draw.Status') }}</div>
                        <div class="d-flex justify-space-between">
                            <span>X</span>
                            <span>{{ posX.toFixed(2) }}</span>
                        </div>
                        <div class="d-flex justify-space-between">
                            <span>Y</span>
                            <span>{{ posY.toFixed(2) }}</span>
                        </div>
                        <div class="d-flex justify-space-between">
                            <span>Z</span>
                            <span>{{ posZ.toFixed(3) }}</span>
                        </div>
                        <div class="d-flex justify-space-between">
                            <span>{{ $t('Draw.CurrentLayer') }}</span>
                            <span>{{ currentLayerIndex }}</span>
                        </div>
                        <div class="d-flex justify-space-between">
                            <span>{{ $t('Draw.Extruding') }}</span>
                            <v-icon small :color="extrudeActive ? 'success' : 'grey'">
                                {{ extrudeActive ? mdiCircle : mdiCircleOutline }}
                            </v-icon>
                        </div>
                        <div class="d-flex justify-space-between mt-1">
                            <span>{{ $t('Draw.Status') }}</span>
                            <span :class="running ? 'success--text' : ''">
                                {{ running ? $t('Draw.Running') : $t('Draw.NotRunning') }}
                            </span>
                        </div>
                    </v-card-text>
                </v-card>
                <v-card>
                    <v-card-title>{{ $t('Draw.Settings') }}</v-card-title>
                    <v-card-text>
                        <v-text-field
                            v-model.number="drawSpeed"
                            :label="$t('Draw.Speed')"
                            type="number"
                            dense
                            outlined
                            suffix="mm/s"
                            hide-details
                            class="mb-3" />
                        <v-text-field
                            v-model.number="drawAcceleration"
                            :label="$t('Draw.Acceleration')"
                            type="number"
                            dense
                            outlined
                            suffix="mm/s²"
                            hide-details
                            class="mb-3" />
                        <v-text-field
                            v-model.number="drawLayerHeight"
                            :label="$t('Draw.LayerHeight')"
                            type="number"
                            step="0.05"
                            dense
                            outlined
                            suffix="mm"
                            hide-details
                            class="mb-3" />
                        <v-text-field
                            v-model.number="drawLineWidth"
                            :label="$t('Draw.LineWidth')"
                            type="number"
                            step="0.05"
                            dense
                            outlined
                            suffix="mm"
                            hide-details
                            class="mb-3" />
                        <v-text-field
                            v-model.number="drawFramerate"
                            :label="$t('Draw.Framerate')"
                            type="number"
                            dense
                            outlined
                            suffix="fps"
                            hide-details
                            class="mb-3" />
                        <v-row dense>
                            <v-col cols="6">
                                <v-text-field
                                    v-model.number="drawXMin"
                                    :label="$t('Draw.XMin')"
                                    type="number"
                                    dense
                                    outlined
                                    hide-details
                                    class="mb-3" />
                            </v-col>
                            <v-col cols="6">
                                <v-text-field
                                    v-model.number="drawXMax"
                                    :label="$t('Draw.XMax')"
                                    type="number"
                                    dense
                                    outlined
                                    hide-details
                                    class="mb-3" />
                            </v-col>
                        </v-row>
                        <v-row dense>
                            <v-col cols="6">
                                <v-text-field
                                    v-model.number="drawYMin"
                                    :label="$t('Draw.YMin')"
                                    type="number"
                                    dense
                                    outlined
                                    hide-details
                                    class="mb-3" />
                            </v-col>
                            <v-col cols="6">
                                <v-text-field
                                    v-model.number="drawYMax"
                                    :label="$t('Draw.YMax')"
                                    type="number"
                                    dense
                                    outlined
                                    hide-details
                                    class="mb-3" />
                            </v-col>
                        </v-row>
                        <v-row dense>
                            <v-col cols="6">
                                <v-text-field
                                    v-model.number="drawZMin"
                                    :label="$t('Draw.ZMin')"
                                    type="number"
                                    dense
                                    outlined
                                    hide-details
                                    class="mb-3" />
                            </v-col>
                            <v-col cols="6">
                                <v-text-field
                                    v-model.number="drawZMax"
                                    :label="$t('Draw.ZMax')"
                                    type="number"
                                    dense
                                    outlined
                                    hide-details
                                    class="mb-3" />
                            </v-col>
                        </v-row>
                        <v-switch
                            v-model="drawToggleShift"
                            :label="$t('Draw.ShiftToggle')"
                            dense
                            hide-details
                            class="mt-0" />
                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>
    </div>
</template>

<script lang="ts">
import { Component, Mixins, Ref, Watch } from 'vue-property-decorator'
import { Debounce } from 'vue-debounce-decorator'
import BaseMixin from '@/components/mixins/base'
import { mdiCircle, mdiCircleOutline } from '@mdi/js'
import { GuiDrawState } from '@/store/gui/types'

interface LayerSegment {
    x1: number
    y1: number
    x2: number
    y2: number
}

const STARTUP_GCODE = ['G90', 'M83', 'G28'].join('\n')

const SHUTDOWN_GCODE = ['M104 S0', 'M140 S0', 'M107', 'M84'].join('\n')

@Component
export default class PageDraw extends Mixins(BaseMixin) {
    mdiCircle = mdiCircle
    mdiCircleOutline = mdiCircleOutline

    @Ref('canvas') readonly canvasRef!: HTMLCanvasElement
    @Ref('canvasContainer') readonly canvasContainer!: HTMLElement

    running = false
    posX = 0
    posY = 0
    posZ = 0
    extrudeActive = false
    pressedKeys: Set<string> = new Set()
    spaceWasPressed = false
    shiftLatched = false
    shiftDownLast = false
    frameInterval: ReturnType<typeof setInterval> | null = null
    resizeObserver: ResizeObserver | null = null
    printedLayers: LayerSegment[][] = [[]]
    activeLayerIndex = 0

    // --- Store-backed settings (getter/setter pattern) ---

    get drawConfig(): GuiDrawState {
        return this.$store.state.gui.draw
    }

    get drawSpeed(): number {
        return this.drawConfig.speed
    }
    set drawSpeed(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.speed', value: val })
    }

    get drawAcceleration(): number {
        return this.drawConfig.acceleration
    }
    set drawAcceleration(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.acceleration', value: val })
    }

    get drawLayerHeight(): number {
        return this.drawConfig.layerHeight
    }
    set drawLayerHeight(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.layerHeight', value: val })
    }

    get drawLineWidth(): number {
        return this.drawConfig.lineWidth
    }
    set drawLineWidth(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.lineWidth', value: val })
    }

    get drawFramerate(): number {
        return this.drawConfig.framerate
    }
    set drawFramerate(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.framerate', value: val })
    }

    get drawXMin(): number {
        return this.drawConfig.xMin
    }
    set drawXMin(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.xMin', value: val })
    }

    get drawXMax(): number {
        return this.drawConfig.xMax
    }
    set drawXMax(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.xMax', value: val })
    }

    get drawYMin(): number {
        return this.drawConfig.yMin
    }
    set drawYMin(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.yMin', value: val })
    }

    get drawYMax(): number {
        return this.drawConfig.yMax
    }
    set drawYMax(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.yMax', value: val })
    }

    get drawZMin(): number {
        return this.drawConfig.zMin
    }
    set drawZMin(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.zMin', value: val })
    }

    get drawZMax(): number {
        return this.drawConfig.zMax
    }
    set drawZMax(val: number) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.zMax', value: val })
    }

    get drawToggleShift(): boolean {
        return this.drawConfig.toggleShift
    }
    set drawToggleShift(val: boolean) {
        this.$store.dispatch('gui/saveSetting', { name: 'draw.toggleShift', value: val })
    }

    get currentLayerIndex(): number {
        const idx = Math.round((this.posZ - this.drawLayerHeight) / this.drawLayerHeight)
        return Math.max(0, idx)
    }

    get livePosition(): number[] {
        return this.$store.state.printer?.motion_report?.live_position ?? [0, 0, 0]
    }

    // --- Lifecycle ---

    mounted() {
        window.addEventListener('keydown', this.onKeyDown)
        window.addEventListener('keyup', this.onKeyUp)

        this.resizeObserver = new ResizeObserver(() => this.onResize())
        this.resizeObserver.observe(this.canvasContainer)
        this.resizeCanvas()
    }

    beforeDestroy() {
        this.stopInternal()
        window.removeEventListener('keydown', this.onKeyDown)
        window.removeEventListener('keyup', this.onKeyUp)
        this.resizeObserver?.disconnect()
        this.resizeObserver = null
    }

    @Watch('printer_state')
    onPrinterStateChange(newState: string) {
        if (this.running && (newState === 'error' || newState === 'shutdown')) {
            this.stopInternal()
        }
    }

    // --- Canvas ---

    @Debounce(200)
    onResize() {
        this.resizeCanvas()
    }

    resizeCanvas() {
        if (!this.canvasRef || !this.canvasContainer) return
        const width = this.canvasContainer.clientWidth
        const height = Math.max(400, width)
        this.canvasRef.width = width
        this.canvasRef.height = height
        this.drawScene()
    }

    worldToScreen(x: number, y: number): [number, number] {
        const canvas = this.canvasRef
        if (!canvas) return [0, 0]
        const w = Math.max(1, this.drawXMax - this.drawXMin)
        const h = Math.max(1, this.drawYMax - this.drawYMin)
        const sx = ((x - this.drawXMin) / w) * canvas.width
        const sy = canvas.height - ((y - this.drawYMin) / h) * canvas.height
        return [sx, sy]
    }

    drawScene() {
        const canvas = this.canvasRef
        if (!canvas) return
        const ctx = canvas.getContext('2d')
        if (!ctx) return

        ctx.fillStyle = '#101216'
        ctx.fillRect(0, 0, canvas.width, canvas.height)

        const curIdx = this.activeLayerIndex
        const layerFade = this.drawConfig.layerFade
        const linePx = Math.max(1, this.drawConfig.linePx)

        for (let idx = 0; idx < this.printedLayers.length; idx++) {
            if (idx > curIdx) continue
            const depth = curIdx - idx
            let color: string
            if (depth === 0) {
                color = 'rgb(245, 199, 66)'
            } else {
                const fade = Math.pow(layerFade, depth)
                const c = Math.max(20, Math.min(255, Math.round(175 * fade)))
                color = `rgb(${c}, ${c}, ${c})`
            }

            ctx.strokeStyle = color
            ctx.lineWidth = linePx
            ctx.beginPath()
            for (const seg of this.printedLayers[idx]) {
                const [x1, y1] = this.worldToScreen(seg.x1, seg.y1)
                const [x2, y2] = this.worldToScreen(seg.x2, seg.y2)
                ctx.moveTo(x1, y1)
                ctx.lineTo(x2, y2)
            }
            ctx.stroke()
        }

        // Simulated position (green)
        const [simSx, simSy] = this.worldToScreen(this.posX, this.posY)
        ctx.beginPath()
        ctx.arc(simSx, simSy, 4, 0, Math.PI * 2)
        ctx.fillStyle = '#50dcaa'
        ctx.fill()

        // Live toolhead position (red)
        const [liveSx, liveSy] = this.worldToScreen(this.livePosition[0], this.livePosition[1])
        ctx.beginPath()
        ctx.arc(liveSx, liveSy, 3, 0, Math.PI * 2)
        ctx.fillStyle = '#dc5a5a'
        ctx.fill()
    }

    // --- Input ---

    onKeyDown(e: KeyboardEvent) {
        if (!this.running) return
        const key = e.key.toLowerCase()
        if (['w', 'a', 's', 'd', ' ', 'shift', 'q'].includes(key)) {
            e.preventDefault()
            this.pressedKeys.add(key)
        }
    }

    onKeyUp(e: KeyboardEvent) {
        const key = e.key.toLowerCase()
        this.pressedKeys.delete(key)
    }

    // --- Movement math ---

    incrementBounded(val: number, move: number, min: number, max: number): number {
        return Math.min(max, Math.max(min, val + move))
    }

    distance(x1: number, y1: number, x2: number, y2: number): number {
        return Math.sqrt((x1 - x2) ** 2 + (y1 - y2) ** 2)
    }

    isExtrudeEnabled(): boolean {
        const shiftDown = this.pressedKeys.has('shift')
        if (!this.drawToggleShift) return shiftDown

        if (shiftDown && !this.shiftDownLast) {
            this.shiftLatched = !this.shiftLatched
        }
        this.shiftDownLast = shiftDown
        return this.shiftLatched
    }

    computeLateralMove(): { x: number; y: number; e: number | null } {
        const prevX = this.posX
        const prevY = this.posY
        let vx = 0
        let vy = 0

        if (this.pressedKeys.has('w')) vy += 1
        if (this.pressedKeys.has('s')) vy -= 1
        if (this.pressedKeys.has('a')) vx -= 1
        if (this.pressedKeys.has('d')) vx += 1

        const mag = Math.sqrt(vx * vx + vy * vy)
        if (mag === 0) return { x: this.posX, y: this.posY, e: null }

        // Normalize and scale by speed / framerate
        const step = this.drawSpeed / this.drawFramerate
        vx = (vx / mag) * step
        vy = (vy / mag) * step

        this.posX = this.incrementBounded(this.posX, vx, this.drawXMin, this.drawXMax)
        this.posY = this.incrementBounded(this.posY, vy, this.drawYMin, this.drawYMax)

        const moveDist = this.distance(prevX, prevY, this.posX, this.posY)
        if (moveDist <= 1e-9) return { x: this.posX, y: this.posY, e: null }

        // Slic3r extrusion formula: cross-section area * distance
        const h = this.drawLayerHeight
        const w = this.drawLineWidth
        const eLength = moveDist * (Math.PI * (h / 2) ** 2 + (w - h) * h)
        return { x: this.posX, y: this.posY, e: eLength }
    }

    // --- G-code helpers ---

    sendGcode(script: string) {
        this.$store.dispatch('server/addEvent', { message: script, type: 'command' })
        this.$socket.emit('printer.gcode.script', { script })
    }

    buildG1(x: number | null, y: number | null, z: number | null, e: number | null): string {
        let cmd = 'G1'
        if (x !== null) cmd += ` X${x.toFixed(3)}`
        if (y !== null) cmd += ` Y${y.toFixed(3)}`
        if (z !== null) cmd += ` Z${z.toFixed(3)}`
        if (e !== null) cmd += ` E${e.toFixed(5)}`
        cmd += ` F${this.drawSpeed * 60}`
        return cmd
    }

    // --- Start / Stop ---

    start() {
        if (this.running || !this.klippyIsConnected) return

        this.posX = (this.drawXMax + this.drawXMin) / 2
        this.posY = (this.drawYMax + this.drawYMin) / 2
        this.posZ = this.drawLayerHeight
        this.pressedKeys.clear()
        this.spaceWasPressed = false
        this.shiftLatched = false
        this.shiftDownLast = false
        this.printedLayers = [[]]
        this.activeLayerIndex = 0
        this.extrudeActive = false

        // Send startup sequence
        this.sendGcode(STARTUP_GCODE)
        this.sendGcode(`M204 S${this.drawAcceleration} T${this.drawAcceleration}`)
        this.sendGcode(`G1 F${this.drawSpeed * 60}`)
        this.sendGcode(this.buildG1(this.posX, this.posY, null, null))
        this.sendGcode(this.buildG1(null, null, this.posZ, null))

        this.running = true
        this.frameInterval = setInterval(() => this.tick(), 1000 / this.drawFramerate)

        // Focus canvas container for keyboard input
        this.$nextTick(() => this.canvasContainer?.focus())
    }

    stop() {
        if (!this.running) return
        this.sendGcode(SHUTDOWN_GCODE)
        this.stopInternal()
    }

    stopInternal() {
        this.running = false
        this.extrudeActive = false
        if (this.frameInterval !== null) {
            clearInterval(this.frameInterval)
            this.frameInterval = null
        }
        this.pressedKeys.clear()
    }

    // --- Frame tick ---

    tick() {
        if (!this.running || !this.klippyIsConnected) {
            this.stopInternal()
            return
        }

        // Quit on Q
        if (this.pressedKeys.has('q')) {
            this.stop()
            return
        }

        const extrudeEnabled = this.isExtrudeEnabled()

        // Space → layer up (edge-triggered)
        if (this.pressedKeys.has(' ')) {
            if (!this.spaceWasPressed) {
                this.spaceWasPressed = true
                this.posZ = this.incrementBounded(this.posZ, this.drawLayerHeight, this.drawZMin, this.drawZMax)
                this.sendGcode(this.buildG1(null, null, this.posZ, null))
            }
        } else {
            this.spaceWasPressed = false
        }

        // Lateral movement
        const prevX = this.posX
        const prevY = this.posY
        const move = this.computeLateralMove()
        const hasMoved = move.e !== null && move.e > 0
        const extrudeNow = hasMoved && extrudeEnabled
        this.extrudeActive = extrudeNow

        if (hasMoved) {
            const gcode = extrudeNow
                ? this.buildG1(move.x, move.y, null, move.e)
                : this.buildG1(move.x, move.y, null, null)
            this.sendGcode(gcode)

            // Record extruded segment for canvas
            if (extrudeNow) {
                this.activeLayerIndex = this.currentLayerIndex
                while (this.printedLayers.length <= this.activeLayerIndex) {
                    this.printedLayers.push([])
                }
                this.printedLayers[this.activeLayerIndex].push({
                    x1: prevX,
                    y1: prevY,
                    x2: move.x,
                    y2: move.y,
                })
            }
        }

        this.drawScene()
    }
}
</script>

<style scoped>
.draw-canvas-container {
    position: relative;
    width: 100%;
    outline: none;
}

.draw-canvas-container canvas {
    display: block;
    width: 100%;
}

.draw-overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    pointer-events: none;
}

.position-relative {
    position: relative;
}
</style>
