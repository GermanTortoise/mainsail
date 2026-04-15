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
                                <span class="text-h6 white--text">{{ $t('Freehand.KeyHints') }}</span>
                            </div>
                        </div>
                    </v-card-text>
                </v-card>
            </v-col>
            <v-col cols="12" md="3">
                <v-card class="mb-4">
                    <v-card-title>{{ $t('Freehand.Controls') }}</v-card-title>
                    <v-card-text>
                        <v-btn
                            color="primary"
                            block
                            :disabled="running || !klippyIsConnected"
                            class="mb-2"
                            @click="start">
                            {{ $t('Freehand.Start') }}
                        </v-btn>
                        <v-btn color="error" block :disabled="!running" class="mb-4" @click="stop">
                            {{ $t('Freehand.Stop') }}
                        </v-btn>
                        <v-divider class="mb-4" />
                        <div class="text-subtitle-2 mb-2">{{ $t('Freehand.Record') }}</div>
                        <v-switch
                            v-model="recording"
                            :label="$t('Freehand.Recording')"
                            :disabled="running"
                            dense
                            hide-details
                            class="mt-0 mb-2" />
                        <v-text-field
                            v-model="recordFilename"
                            :label="$t('Freehand.Filename')"
                            :disabled="running"
                            dense
                            outlined
                            hide-details
                            suffix=".gcode"
                            class="mb-4" />
                        <v-divider class="my-4" />
                        <div class="text-subtitle-2 mb-2">{{ $t('Freehand.Status') }}</div>
                        <div class="d-flex justify-space-between">
                            <span>X</span>
                            <span>{{ livePosition[0].toFixed(2) }}</span>
                        </div>
                        <div class="d-flex justify-space-between">
                            <span>Y</span>
                            <span>{{ livePosition[1].toFixed(2) }}</span>
                        </div>
                        <div class="d-flex justify-space-between">
                            <span>Z</span>
                            <span>{{ livePosition[2].toFixed(3) }}</span>
                        </div>
                        <div class="d-flex justify-space-between">
                            <span>{{ $t('Freehand.CurrentLayer') }}</span>
                            <span>{{ currentLayerIndex }}</span>
                        </div>
                        <div class="d-flex justify-space-between">
                            <span>{{ $t('Freehand.Extruding') }}</span>
                            <v-icon small :color="extrudeActive ? 'success' : 'grey'">
                                {{ extrudeActive ? mdiCircle : mdiCircleOutline }}
                            </v-icon>
                        </div>
                        <div class="d-flex justify-space-between mt-1">
                            <span>{{ $t('Freehand.Status') }}</span>
                            <span :class="running ? 'success--text' : ''">
                                {{ running ? $t('Freehand.Running') : $t('Freehand.NotRunning') }}
                            </span>
                        </div>
                    </v-card-text>
                </v-card>
                <v-card>
                    <v-card-title>{{ $t('Freehand.Settings') }}</v-card-title>
                    <v-card-text>
                        <number-input
                            :label="$t('Freehand.Speed')"
                            param="speed"
                            :target="freehandSpeed"
                            :min="1"
                            :max="null"
                            :dec="0"
                            :step="10"
                            :has-spinner="true"
                            unit="mm/s"
                            class="mb-3"
                            @submit="onFreehandSettingChange" />
                        <number-input
                            :label="$t('Freehand.Acceleration')"
                            param="acceleration"
                            :target="freehandAcceleration"
                            :min="1"
                            :max="null"
                            :dec="0"
                            :step="100"
                            :has-spinner="true"
                            unit="mm/s²"
                            class="mb-3"
                            @submit="onFreehandSettingChange" />
                        <number-input
                            :label="$t('Freehand.ExtruderTemp')"
                            param="extruderTemp"
                            :target="freehandExtruderTemp"
                            :min="0"
                            :max="500"
                            :dec="0"
                            :step="5"
                            :has-spinner="true"
                            unit="°C"
                            class="mb-3"
                            @submit="onFreehandSettingChange" />
                        <number-input
                            :label="$t('Freehand.BedTemp')"
                            param="bedTemp"
                            :target="freehandBedTemp"
                            :min="0"
                            :max="150"
                            :dec="0"
                            :step="5"
                            :has-spinner="true"
                            unit="°C"
                            class="mb-3"
                            @submit="onFreehandSettingChange" />
                        <number-input
                            :label="$t('Freehand.LayerHeight')"
                            param="layerHeight"
                            :target="freehandLayerHeight"
                            :min="0.05"
                            :max="null"
                            :dec="2"
                            :step="0.05"
                            :has-spinner="true"
                            unit="mm"
                            class="mb-3"
                            @submit="onFreehandSettingChange" />
                        <number-input
                            :label="$t('Freehand.LineWidth')"
                            param="lineWidth"
                            :target="freehandLineWidth"
                            :min="0.1"
                            :max="null"
                            :dec="2"
                            :step="0.05"
                            :has-spinner="true"
                            unit="mm"
                            class="mb-3"
                            @submit="onFreehandSettingChange" />
                        <number-input
                            :label="$t('Freehand.Framerate')"
                            param="framerate"
                            :target="freehandFramerate"
                            :min="1"
                            :max="120"
                            :dec="0"
                            :step="5"
                            :has-spinner="true"
                            unit="fps"
                            class="mb-3"
                            @submit="onFreehandSettingChange" />
                        <v-row dense>
                            <v-col cols="6">
                                <number-input
                                    :label="$t('Freehand.XMin')"
                                    param="xMin"
                                    :target="freehandXMin"
                                    :min="0"
                                    :max="null"
                                    :dec="0"
                                    :has-spinner="true"
                                    class="mb-3"
                                    @submit="onFreehandSettingChange" />
                            </v-col>
                            <v-col cols="6">
                                <number-input
                                    :label="$t('Freehand.XMax')"
                                    param="xMax"
                                    :target="freehandXMax"
                                    :min="0"
                                    :max="null"
                                    :dec="0"
                                    :has-spinner="true"
                                    class="mb-3"
                                    @submit="onFreehandSettingChange" />
                            </v-col>
                        </v-row>
                        <v-row dense>
                            <v-col cols="6">
                                <number-input
                                    :label="$t('Freehand.YMin')"
                                    param="yMin"
                                    :target="freehandYMin"
                                    :min="0"
                                    :max="null"
                                    :dec="0"
                                    :has-spinner="true"
                                    class="mb-3"
                                    @submit="onFreehandSettingChange" />
                            </v-col>
                            <v-col cols="6">
                                <number-input
                                    :label="$t('Freehand.YMax')"
                                    param="yMax"
                                    :target="freehandYMax"
                                    :min="0"
                                    :max="null"
                                    :dec="0"
                                    :has-spinner="true"
                                    class="mb-3"
                                    @submit="onFreehandSettingChange" />
                            </v-col>
                        </v-row>
                        <v-row dense>
                            <v-col cols="6">
                                <number-input
                                    :label="$t('Freehand.ZMin')"
                                    param="zMin"
                                    :target="freehandZMin"
                                    :min="0"
                                    :max="null"
                                    :dec="0"
                                    :has-spinner="true"
                                    class="mb-3"
                                    @submit="onFreehandSettingChange" />
                            </v-col>
                            <v-col cols="6">
                                <number-input
                                    :label="$t('Freehand.ZMax')"
                                    param="zMax"
                                    :target="freehandZMax"
                                    :min="0"
                                    :max="null"
                                    :dec="0"
                                    :has-spinner="true"
                                    class="mb-3"
                                    @submit="onFreehandSettingChange" />
                            </v-col>
                        </v-row>
                        <v-switch
                            v-model="freehandToggleShift"
                            :label="$t('Freehand.ShiftToggle')"
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
import NumberInput from '@/components/inputs/NumberInput.vue'
import { mdiCircle, mdiCircleOutline } from '@mdi/js'
import { GuiFreehandState } from '@/store/gui/types'

interface LayerSegment {
    x1: number
    y1: number
    x2: number
    y2: number
}

// const STARTUP_GCODE = ['G90', 'M83', 'G28'].join('\n')

// const SHUTDOWN_GCODE = ['M104 S0', 'M140 S0', 'M107', 'M84'].join('\n')

const SECONDS_IN_MINUTE = 60

/** Number of frame ticks between G-code buffer flushes */
const FLUSH_INTERVAL = 3

@Component({
    components: { NumberInput },
})
export default class PageFreehand extends Mixins(BaseMixin) {
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
    gcodeBuffer: string[] = []
    tickCount = 0
    recording = false
    recordFilename = 'drawing'
    recordedGcode: string[] = []

    // --- Store-backed settings ---

    get freehandConfig(): GuiFreehandState {
        return this.$store.state.gui.freehand
    }

    get freehandSpeed(): number {
        return this.freehandConfig.speed
    }

    get freehandAcceleration(): number {
        return this.freehandConfig.acceleration
    }

    get freehandExtruderTemp(): number {
        return this.freehandConfig.extruderTemp
    }

    get freehandBedTemp(): number {
        return this.freehandConfig.bedTemp
    }

    get freehandLayerHeight(): number {
        return this.freehandConfig.layerHeight
    }

    get freehandLineWidth(): number {
        return this.freehandConfig.lineWidth
    }

    get freehandFramerate(): number {
        return this.freehandConfig.framerate
    }

    get freehandXMin(): number {
        return this.freehandConfig.xMin
    }

    get freehandXMax(): number {
        return this.freehandConfig.xMax
    }

    get freehandYMin(): number {
        return this.freehandConfig.yMin
    }

    get freehandYMax(): number {
        return this.freehandConfig.yMax
    }

    get freehandZMin(): number {
        return this.freehandConfig.zMin
    }

    get freehandZMax(): number {
        return this.freehandConfig.zMax
    }

    get freehandToggleShift(): boolean {
        return this.freehandConfig.toggleShift
    }
    set freehandToggleShift(val: boolean) {
        this.$store.dispatch('gui/saveSetting', { name: 'freehand.toggleShift', value: val })
    }

    onFreehandSettingChange(payload: { name: string; value: number }) {
        this.$store.dispatch('gui/saveSetting', { name: `freehand.${payload.name}`, value: payload.value })
    }

    get currentLayerIndex(): number {
        const idx = Math.round((this.posZ - this.freehandLayerHeight) / this.freehandLayerHeight)
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
        this.renderScene()
    }

    worldToScreen(x: number, y: number): [number, number] {
        const canvas = this.canvasRef
        if (!canvas) return [0, 0]
        const w = Math.max(1, this.freehandXMax - this.freehandXMin)
        const h = Math.max(1, this.freehandYMax - this.freehandYMin)
        const sx = ((x - this.freehandXMin) / w) * canvas.width
        const sy = canvas.height - ((y - this.freehandYMin) / h) * canvas.height
        return [sx, sy]
    }

    renderScene() {
        const canvas = this.canvasRef
        if (!canvas) return
        const ctx = canvas.getContext('2d')
        if (!ctx) return

        ctx.fillStyle = '#101216'
        ctx.fillRect(0, 0, canvas.width, canvas.height)

        const curIdx = this.activeLayerIndex
        const layerFade = this.freehandConfig.layerFade
        const linePx = Math.max(1, this.freehandConfig.linePx)

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
        if (!this.freehandToggleShift) return shiftDown

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
        const step = this.freehandSpeed / this.freehandFramerate
        vx = (vx / mag) * step
        vy = (vy / mag) * step

        this.posX = this.incrementBounded(this.posX, vx, this.freehandXMin, this.freehandXMax)
        this.posY = this.incrementBounded(this.posY, vy, this.freehandYMin, this.freehandYMax)

        const moveDist = this.distance(prevX, prevY, this.posX, this.posY)
        if (moveDist <= 1e-9) return { x: this.posX, y: this.posY, e: null }

        // Slic3r extrusion formula: cross-section area * distance
        const h = this.freehandLayerHeight
        const w = this.freehandLineWidth
        const eLength = moveDist * (Math.PI * (h / 2) ** 2 + (w - h) * h)
        return { x: this.posX, y: this.posY, e: eLength }
    }

    // --- G-code helpers ---

    sendGcode(script: string) {
        if (this.recording) {
            this.recordedGcode.push(script)
        }
        this.$store.dispatch('server/addEvent', { message: script, type: 'command' })
        this.$socket.emit('printer.gcode.script', { script })
    }

    bufferGcode(line: string) {
        this.gcodeBuffer.push(line)
    }

    flushGcodeBuffer() {
        if (this.gcodeBuffer.length === 0) return
        this.sendGcode(this.gcodeBuffer.join('\n'))
        this.gcodeBuffer = []
    }

    buildG1(x: number | null, y: number | null, z: number | null, e: number | null): string {
        let cmd = 'G1'
        if (x !== null) cmd += ` X${x.toFixed(3)}`
        if (y !== null) cmd += ` Y${y.toFixed(3)}`
        if (z !== null) cmd += ` Z${z.toFixed(3)}`
        if (e !== null) cmd += ` E${e.toFixed(5)}`
        cmd += ` F${this.freehandSpeed * 60}`
        return cmd
    }

    // --- Start / Stop ---

    start() {
        if (this.running || !this.klippyIsConnected) return

        this.posX = (this.freehandXMax + this.freehandXMin) / 2
        this.posY = (this.freehandYMax + this.freehandYMin) / 2
        this.posZ = this.freehandLayerHeight
        this.pressedKeys.clear()
        this.spaceWasPressed = false
        this.shiftLatched = false
        this.shiftDownLast = false
        this.printedLayers = [[]]
        this.activeLayerIndex = 0
        this.extrudeActive = false
        this.gcodeBuffer = []
        this.tickCount = 0

        if (this.recording) {
            this.recordedGcode = []
        }

        // Send startup sequence
        this.sendGcode(`G90\nM83`)
        this.sendGcode(`M204 S${this.freehandAcceleration} T${this.freehandAcceleration}`)
        // this.sendGcode(`M140 S${this.freehandBedTemp}`)
        // this.sendGcode(`M104 S${this.freehandExtruderTemp}`)
        this.sendGcode(`G28`)
        // this.sendGcode(`M190 S${this.freehandBedTemp}`)
        // this.sendGcode(`SETUP_KAMP_MESHING DISPLAY_PARAMETERS=1\nBED_MESH_CLEAR\nBED_MESH_CALIBRATE`)
        // this.sendGcode(`M109 S${this.freehandExtruderTemp}`)
        this.sendGcode(`G1 F${this.freehandSpeed * SECONDS_IN_MINUTE}`)
        this.sendGcode(this.buildG1(this.posX, this.posY, null, null))
        this.sendGcode(this.buildG1(null, null, this.posZ, null))

        this.running = true
        this.frameInterval = setInterval(() => this.tick(), 1000 / this.freehandFramerate)

        // Focus canvas container for keyboard input
        this.$nextTick(() => this.canvasContainer?.focus())
    }

    stop() {
        if (!this.running) return
        this.flushGcodeBuffer()
        this.sendGcode(SHUTDOWN_GCODE)

        if (this.recording && this.recordedGcode.length > 0) {
            this.saveRecording()
        }

        this.stopInternal()
    }

    stopInternal() {
        this.running = false
        this.extrudeActive = false
        this.gcodeBuffer = []
        this.tickCount = 0
        if (this.frameInterval !== null) {
            clearInterval(this.frameInterval)
            this.frameInterval = null
        }
        this.pressedKeys.clear()
    }

    // --- Recording ---

    buildHeader(): string {
        const now = new Date()
        const date = now.toISOString().slice(0, 10)
        const time = now.toTimeString().slice(0, 8)
        const totalLayers = this.activeLayerIndex + 1
        const maxZ = this.posZ

        return [
            '; HEADER_BLOCK_START',
            `; generated by Freehand on ${date} at ${time}`,
            `; total layer number: ${totalLayers}`,
            `; layer_height: ${this.freehandLayerHeight}`,
            `; line_width: ${this.freehandLineWidth}`,
            `; max_z_height: ${maxZ.toFixed(2)}`,
            '; HEADER_BLOCK_END',
        ].join('\n')
    }

    async saveRecording() {
        const header = this.buildHeader()
        const content = header + '\n' + this.recordedGcode.join('\n') + '\n'
        const safeName = this.recordFilename.replace(/[^a-zA-Z0-9_-]/g, '_') || 'drawing'
        const filename = `${safeName}.gcode`
        const file = new File([content], filename, { type: 'text/plain' })

        await this.$store.dispatch('files/uploadFile', { file, path: '', root: 'gcodes' })
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

        this.tickCount++

        // Space → layer up (edge-triggered)
        if (this.pressedKeys.has(' ')) {
            if (!this.spaceWasPressed) {
                this.spaceWasPressed = true
                this.posZ = this.incrementBounded(this.posZ, this.freehandLayerHeight, this.freehandZMin, this.freehandZMax)
                this.activeLayerIndex = this.currentLayerIndex
                while (this.printedLayers.length <= this.activeLayerIndex) {
                    this.printedLayers.push([])
                }
                this.flushGcodeBuffer()
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
            this.bufferGcode(gcode)

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

        // Flush buffer every FLUSH_INTERVAL ticks or when idle
        if (this.tickCount % FLUSH_INTERVAL === 0 || !hasMoved) {
            this.flushGcodeBuffer()
        }

        this.renderScene()
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
