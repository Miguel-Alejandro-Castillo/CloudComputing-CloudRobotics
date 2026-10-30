<template>
    <div class="robot-controller">

        <!-- CABECERA -->
        <header class="controller-header">
            <div>
                <h2>Robot 4WD</h2>
                <span>Control manual</span>
            </div>
        </header>

        <!-- INSTRUMENTOS -->
        <div class="instruments">

            <!-- VOLANTE -->
            <section class="instrument-section">

                <div
                    class="wheel"
                    :class="{ active: steeringActive }"
                    :style="{
                        transform: `rotate(${steeringAngle}deg)`
                    }"
                    @pointerdown="startSteering"
                >
                    <div class="wheel-center">
                        <span>4WD</span>
                    </div>

                    <div class="wheel-spoke spoke-top"></div>
                    <div class="wheel-spoke spoke-left"></div>
                    <div class="wheel-spoke spoke-right"></div>
                </div>

                <div class="instrument-value">
                    Dirección: {{ Math.round(steeringAngle) }}°
                </div>

                <div class="instrument-action">
                    <button
                        type="button"
                        class="center-button"
                        @click="centerSteering"
                    >
                        Centrar volante
                    </button>
                </div>

            </section>

            <!-- GAUGE INTERACTIVO -->
            <section class="instrument-section">

                <div
                    class="gauge"
                    :class="{ active: powerGaugeActive }"
                >
                    <svg
                        ref="powerGauge"
                        viewBox="0 0 220 150"
                        role="slider"
                        aria-label="Potencia de los motores"
                        aria-valuemin="10"
                        aria-valuemax="90"
                        :aria-valuenow="speed"
                        tabindex="0"
                        @pointerdown="startPowerGauge"
                        @keydown="handleGaugeKeyboard"
                    >

                        <!-- Fondo -->
                        <path
                            class="gauge-background"
                            d="M 30 115 A 80 80 0 0 1 190 115"
                        />

                        <!-- Progreso -->
                        <path
                            class="gauge-progress"
                            :style="gaugeProgressStyle"
                            d="M 30 115 A 80 80 0 0 1 190 115"
                        />

                        <!-- Marcas -->
                        <line
                            v-for="mark in gaugeMarks"
                            :key="mark.value"
                            class="gauge-mark"
                            :class="{
                                major: mark.value % 20 === 10
                            }"
                            x1="110"
                            y1="31"
                            x2="110"
                            :y2="mark.value % 20 === 10 ? 43 : 38"
                            :style="{
                                transform:
                                    `rotate(${mark.angle}deg)`
                            }"
                        />

                        <!-- Aguja -->
                        <line
                            class="gauge-needle"
                            x1="110"
                            y1="115"
                            x2="110"
                            y2="45"
                            :style="{
                                transform:
                                    `rotate(${needleAngle}deg)`
                            }"
                        />

                        <circle
                            class="gauge-center"
                            cx="110"
                            cy="115"
                            r="10"
                        />

                        <!-- Valor -->
                        <text
                            class="gauge-number"
                            x="110"
                            y="91"
                            text-anchor="middle"
                        >
                            {{ speed }}
                        </text>

                        <text
                            class="gauge-unit"
                            x="110"
                            y="107"
                            text-anchor="middle"
                        >
                            % PWM
                        </text>

                        <!-- Escala -->
                        <text
                            class="gauge-limit"
                            x="21"
                            y="135"
                        >
                            10
                        </text>

                        <text
                            class="gauge-middle"
                            x="110"
                            y="24"
                            text-anchor="middle"
                        >
                            50
                        </text>

                        <text
                            class="gauge-limit"
                            x="184"
                            y="135"
                        >
                            90
                        </text>

                    </svg>
                </div>

                <div class="instrument-value">
                    Potencia: {{ speed }}%
                </div>

                <!-- ÚNICO ESTADO -->
                <div class="instrument-action">
                    <div
                        class="status"
                        :class="statusClass"
                    >
                        {{ statusText }}
                    </div>
                </div>

            </section>

        </div>

        <!-- MARCHA -->
        <div class="drive-controls">

            <div class="direction-buttons">

                <button
                    type="button"
                    class="forward-button"
                    :class="{
                        selected:
                            running && direction === 1
                    }"
                    @click="startForward"
                >
                    ▲ FORWARD
                </button>

                <button
                    type="button"
                    class="reverse-button"
                    :class="{
                        selected:
                            running && direction === -1
                    }"
                    @click="startReverse"
                >
                    ▼ REVERSE
                </button>

            </div>

            <button
                type="button"
                class="stop-button"
                @click="stopRobot"
            >
                ■ STOP
            </button>

        </div>

        <!-- ESTADO DE MOTORES -->
        <div class="motor-state">

            <div class="motor-card">
                <span>Motor izquierdo</span>

                <strong>
                    {{ formatMotorValue(currentCommand.left) }}
                </strong>

                <small>
                    {{ motorDirection(currentCommand.left) }}
                </small>
            </div>

            <div class="motor-card">
                <span>Motor derecho</span>

                <strong>
                    {{ formatMotorValue(currentCommand.right) }}
                </strong>

                <small>
                    {{ motorDirection(currentCommand.right) }}
                </small>
            </div>

        </div>

        <div class="safety-note">
            Cambio de sentido protegido mediante parada previa.
            STOP envía inmediatamente left = 0 y right = 0.
        </div>

    </div>
</template>


<script>
export default {
    data() {
        return {
            /*
             * Estado general
             */
            running: false,

            /*
             *  1 = FORWARD
             * -1 = REVERSE
             *  0 = STOP
             */
            direction: 0,

            /*
             * Potencia PWM seleccionada.
             */
            speed: 40,

            /*
             * Volante.
             */
            steeringAngle: 0,
            steeringActive: false,

            pointerStartX: 0,
            initialAngle: 0,

            maxSteeringAngle: 120,

            /*
             * En el giro máximo, el motor interior
             * puede reducirse hasta un 80%.
             */
            maxSteeringReduction: 0.80,

            /*
             * Gauge.
             */
            powerGaugeActive: false,

            /*
             * Protección al invertir marcha.
             */
            reverseDelayMs: 250,
            reverseTimer: null,

            /*
             * Última orden enviada al L298N.
             */
            currentCommand: {
                left: 0,
                right: 0
            }
        }
    },

    computed: {

        statusText() {
            if (!this.running) {
                return "DETENIDO"
            }

            return this.direction === 1
                ? "FORWARD"
                : "REVERSE"
        },

        statusClass() {
            if (!this.running) {
                return "status-stopped"
            }

            return this.direction === 1
                ? "status-forward"
                : "status-reverse"
        },

        /*
         * 10% -> -90 grados
         * 50% ->   0 grados
         * 90% -> +90 grados
         */
        needleAngle() {
            const percentage =
                (this.clamp(this.speed, 10, 90) - 10) / 80

            return -90 + percentage * 180
        },

        gaugeProgressStyle() {
            const totalLength = 251.3

            const percentage =
                (this.clamp(this.speed, 10, 90) - 10) / 80

            const visibleLength =
                totalLength * percentage

            return {
                strokeDasharray:
                    `${visibleLength} ${totalLength}`
            }
        },

        gaugeMarks() {
            const marks = []

            for (
                let value = 10;
                value <= 90;
                value += 10
            ) {
                const percentage =
                    (value - 10) / 80

                marks.push({
                    value,
                    angle:
                        -90 +
                        percentage * 180
                })
            }

            return marks
        }
    },

    mounted() {

        window.addEventListener(
            "pointermove",
            this.handlePointerMove
        )

        window.addEventListener(
            "pointerup",
            this.handlePointerUp
        )

        window.addEventListener(
            "pointercancel",
            this.handlePointerUp
        )

        /*
         * Estado seguro inicial.
         */
        this.sendCommand(
            0,
            0,
            "initial-stop"
        )
    },

    unmounted() {

        window.removeEventListener(
            "pointermove",
            this.handlePointerMove
        )

        window.removeEventListener(
            "pointerup",
            this.handlePointerUp
        )

        window.removeEventListener(
            "pointercancel",
            this.handlePointerUp
        )

        if (this.reverseTimer) {
            window.clearTimeout(
                this.reverseTimer
            )
        }

        this.sendCommand(
            0,
            0,
            "widget-closed"
        )
    },

    methods: {

        /*
         * =================================================
         * MARCHA
         * =================================================
         */

        startForward() {
            this.changeDirection(
                1,
                "forward"
            )
        },

        startReverse() {
            this.changeDirection(
                -1,
                "reverse"
            )
        },

        changeDirection(
            newDirection,
            action
        ) {

            const directionChanged =
                this.running &&
                this.direction !== 0 &&
                this.direction !== newDirection

            /*
             * Nunca pasar directamente:
             *
             * +PWM -> -PWM
             *
             * Primero STOP.
             */
            if (directionChanged) {

                if (this.reverseTimer) {
                    window.clearTimeout(
                        this.reverseTimer
                    )
                }

                this.running = false
                this.direction = 0

                this.sendCommand(
                    0,
                    0,
                    `${action}-safety-stop`
                )

                this.reverseTimer =
                    window.setTimeout(() => {

                        this.direction =
                            newDirection

                        this.running = true

                        this.reverseTimer = null

                        this.updateMovement(
                            action
                        )

                    }, this.reverseDelayMs)

                return
            }

            this.direction =
                newDirection

            this.running = true

            this.updateMovement(
                action
            )
        },

        stopRobot() {

            if (this.reverseTimer) {

                window.clearTimeout(
                    this.reverseTimer
                )

                this.reverseTimer = null
            }

            this.running = false
            this.direction = 0

            this.sendCommand(
                0,
                0,
                "stop"
            )
        },

        /*
         * =================================================
         * VOLANTE
         * =================================================
         */

        startSteering(event) {

            event.preventDefault()

            this.steeringActive = true

            this.pointerStartX =
                event.clientX

            this.initialAngle =
                this.steeringAngle
        },

        moveSteering(event) {

            if (!this.steeringActive) {
                return
            }

            const displacement =
                event.clientX -
                this.pointerStartX

            const sensitivity = 1.2

            const angle =
                this.initialAngle +
                displacement *
                sensitivity

            this.steeringAngle =
                this.clamp(
                    angle,
                    -this.maxSteeringAngle,
                    this.maxSteeringAngle
                )

            this.updateMovement(
                "steering"
            )
        },

        stopSteering() {
            this.steeringActive = false
        },

        centerSteering() {

            this.steeringAngle = 0

            this.updateMovement(
                "center"
            )
        },

        /*
         * =================================================
         * GAUGE DE POTENCIA
         * =================================================
         */

        startPowerGauge(event) {

            event.preventDefault()

            this.powerGaugeActive = true

            this.updatePowerFromPointer(
                event
            )
        },

        movePowerGauge(event) {

            if (!this.powerGaugeActive) {
                return
            }

            this.updatePowerFromPointer(
                event
            )
        },

        stopPowerGauge() {
            this.powerGaugeActive = false
        },

        updatePowerFromPointer(event) {

            const gauge =
                this.$refs.powerGauge

            if (!gauge) {
                return
            }

            const rectangle =
                gauge.getBoundingClientRect()

            if (
                rectangle.width <= 0 ||
                rectangle.height <= 0
            ) {
                return
            }

            /*
             * Conversión de coordenadas navegador
             * -> viewBox SVG.
             */
            const x =
                (
                    event.clientX -
                    rectangle.left
                ) *
                (
                    220 /
                    rectangle.width
                )

            const y =
                (
                    event.clientY -
                    rectangle.top
                ) *
                (
                    150 /
                    rectangle.height
                )

            const centerX = 110
            const centerY = 115

            let angle =
                Math.atan2(
                    y - centerY,
                    x - centerX
                ) *
                (
                    180 /
                    Math.PI
                )

            /*
             * Solo utilizamos el semicírculo superior.
             */
            if (angle > 0) {
                angle =
                    x < centerX
                        ? -180
                        : 0
            }

            angle =
                this.clamp(
                    angle,
                    -180,
                    0
                )

            const percentage =
                (
                    angle +
                    180
                ) / 180

            const newSpeed =
                10 +
                percentage * 80

            this.speed =
                Math.round(
                    this.clamp(
                        newSpeed,
                        10,
                        90
                    )
                )

            this.updateMovement(
                "power-gauge"
            )
        },

        /*
         * Control accesible mediante teclado.
         */
        handleGaugeKeyboard(event) {

            let newSpeed =
                this.speed

            if (
                event.key === "ArrowRight" ||
                event.key === "ArrowUp"
            ) {

                newSpeed += 1

            } else if (
                event.key === "ArrowLeft" ||
                event.key === "ArrowDown"
            ) {

                newSpeed -= 1

            } else if (
                event.key === "PageUp"
            ) {

                newSpeed += 10

            } else if (
                event.key === "PageDown"
            ) {

                newSpeed -= 10

            } else if (
                event.key === "Home"
            ) {

                newSpeed = 10

            } else if (
                event.key === "End"
            ) {

                newSpeed = 90

            } else {

                return
            }

            event.preventDefault()

            this.speed =
                Math.round(
                    this.clamp(
                        newSpeed,
                        10,
                        90
                    )
                )

            this.updateMovement(
                "power-keyboard"
            )
        },

        /*
         * =================================================
         * EVENTOS DE PUNTERO
         * =================================================
         */

        handlePointerMove(event) {

            this.moveSteering(event)

            this.movePowerGauge(event)
        },

        handlePointerUp() {

            this.stopSteering()

            this.stopPowerGauge()
        },

        /*
         * =================================================
         * CONTROL DIFERENCIAL
         * =================================================
         */

        updateMovement(
            reason = "update"
        ) {

            /*
             * Si estamos detenidos se conserva
             * la potencia elegida, pero no se
             * envía movimiento.
             */
            if (
                !this.running ||
                this.direction === 0
            ) {

                this.currentCommand = {
                    left: 0,
                    right: 0
                }

                return
            }

            const baseSpeed =
                this.clamp(
                    Number(this.speed),
                    10,
                    90
                )

            /*
             * steeringRatio:
             *
             * -1 = izquierda máxima
             *  0 = centro
             * +1 = derecha máxima
             */
            const steeringRatio =
                this.steeringAngle /
                this.maxSteeringAngle

            let leftMagnitude =
                baseSpeed

            let rightMagnitude =
                baseSpeed

            /*
             * Izquierda:
             * reducimos potencia al lado izquierdo.
             */
            if (steeringRatio < 0) {

                leftMagnitude =
                    baseSpeed *
                    (
                        1 -
                        Math.abs(
                            steeringRatio
                        ) *
                        this.maxSteeringReduction
                    )
            }

            /*
             * Derecha:
             * reducimos potencia al lado derecho.
             */
            if (steeringRatio > 0) {

                rightMagnitude =
                    baseSpeed *
                    (
                        1 -
                        steeringRatio *
                        this.maxSteeringReduction
                    )
            }

            const left =
                Math.round(
                    leftMagnitude
                ) *
                this.direction

            const right =
                Math.round(
                    rightMagnitude
                ) *
                this.direction

            this.sendCommand(
                left,
                right,
                reason
            )
        },

        /*
         * =================================================
         * SALIDA HACIA NODE-RED
         * =================================================
         */

        sendCommand(
            left,
            right,
            action
        ) {

            const safeLeft =
                this.clamp(
                    Number(left) || 0,
                    -90,
                    90
                )

            const safeRight =
                this.clamp(
                    Number(right) || 0,
                    -90,
                    90
                )

            this.currentCommand = {
                left: safeLeft,
                right: safeRight
            }

            /*
             * Este payload se conecta directamente
             * al Subflow L298N.
             */
            this.send({

                topic:
                    "robot/manual/control",

                payload: {
                    left: safeLeft,
                    right: safeRight
                },

                control: {
                    source:
                        "flowfuse-wheel",

                    action,

                    running:
                        this.running,

                    direction:
                        this.direction,

                    speed:
                        this.speed,

                    steeringAngle:
                        Math.round(
                            this.steeringAngle
                        )
                }
            })
        },

        /*
         * =================================================
         * REPRESENTACIÓN
         * =================================================
         */

        formatMotorValue(value) {

            const numericValue =
                Number(value) || 0

            if (numericValue > 0) {
                return `+${numericValue}`
            }

            return `${numericValue}`
        },

        motorDirection(value) {

            const numericValue =
                Number(value) || 0

            if (numericValue > 0) {
                return "Adelante"
            }

            if (numericValue < 0) {
                return "Reversa"
            }

            return "Detenido"
        },

        clamp(
            value,
            minimum,
            maximum
        ) {

            return Math.max(
                minimum,
                Math.min(
                    maximum,
                    value
                )
            )
        }
    }
}
</script>


<style>
.robot-controller {
    width: 100%;
    box-sizing: border-box;

    padding: 20px;

    border-radius: 20px;

    color: #edf4ff;

    background:
        radial-gradient(
            circle at top,
            #263950 0%,
            #121c2a 52%,
            #090f18 100%
        );

    box-shadow:
        inset 0 1px 0
            rgba(255, 255, 255, 0.08),

        0 14px 32px
            rgba(0, 0, 0, 0.28);

    user-select: none;
    touch-action: none;
}


/* CABECERA */

.controller-header {
    margin-bottom: 20px;
}

.controller-header h2 {
    margin: 0;

    font-size: 1.4rem;
}

.controller-header span {
    color: #9aacbf;

    font-size: 0.85rem;
}


/* INSTRUMENTOS */

.instruments {
    display: grid;

    grid-template-columns:
        minmax(260px, 1fr)
        minmax(260px, 1fr);

    align-items: end;

    gap: 34px;
}

.instrument-section {
    display: grid;

    grid-template-rows:
        310px
        32px
        44px;

    align-items: center;
    justify-items: center;

    gap: 8px;
}

.instrument-value {
    min-height: 32px;

    display: flex;
    align-items: center;
    justify-content: center;

    color: #aebfd1;

    font-size: 0.92rem;
    font-weight: 600;

    font-variant-numeric:
        tabular-nums;
}

.instrument-action {
    min-height: 44px;

    display: flex;
    align-items: center;
    justify-content: center;
}


/* VOLANTE */

.wheel {
    position: relative;

    width: min(275px, 58vw);

    aspect-ratio: 1;

    box-sizing: border-box;

    border:
        22px solid #1d2632;

    border-radius: 50%;

    cursor: grab;

    transition:
        transform 80ms linear,
        filter 160ms ease;

    box-shadow:
        inset 0 0 0 5px
            #465466,

        inset 0 0 22px
            rgba(0, 0, 0, 0.75),

        0 14px 25px
            rgba(0, 0, 0, 0.45);
}

.wheel.active {
    cursor: grabbing;

    filter:
        brightness(1.15);
}

.wheel-center {
    position: absolute;

    left: 50%;
    top: 50%;

    width: 86px;
    height: 86px;

    transform:
        translate(
            -50%,
            -50%
        );

    display: grid;

    place-items: center;

    border:
        5px solid #111923;

    border-radius: 50%;

    background:
        linear-gradient(
            145deg,
            #526579,
            #18222e
        );

    box-shadow:
        0 5px 12px
            rgba(0, 0, 0, 0.45);

    z-index: 3;
}

.wheel-center span {
    color: #e7eff8;

    font-weight: 900;

    letter-spacing:
        0.08rem;
}

.wheel-spoke {
    position: absolute;

    left: 50%;
    top: 50%;

    width: 18px;
    height: 85px;

    transform-origin:
        50% 0%;

    border-radius:
        10px;

    background:
        linear-gradient(
            90deg,
            #263341,
            #67778a,
            #263341
        );
}

.spoke-top {
    transform:
        translate(-50%, 0)
        rotate(180deg);
}

.spoke-left {
    transform:
        translate(-50%, 0)
        rotate(58deg);
}

.spoke-right {
    transform:
        translate(-50%, 0)
        rotate(-58deg);
}

.center-button {
    height: 38px;

    padding:
        0 16px;

    color: #d9e7f5;

    background:
        #243244;

    border:
        1px solid #4b6076;

    border-radius:
        10px;

    cursor: pointer;
}


/* GAUGE */

.gauge {
    width: 100%;

    max-width: 320px;

    box-sizing: border-box;

    padding: 6px;

    border-radius: 18px;

    cursor: grab;

    transition:
        background 120ms ease,
        box-shadow 120ms ease;
}

.gauge:hover {
    background:
        rgba(
            255,
            255,
            255,
            0.035
        );
}

.gauge.active {
    cursor: grabbing;

    background:
        rgba(
            86,
            200,
            255,
            0.07
        );

    box-shadow:
        0 0 0 2px
            rgba(
                86,
                200,
                255,
                0.35
            );
}

.gauge svg {
    width: 100%;

    overflow: visible;

    touch-action: none;
}

.gauge svg:focus-visible {
    outline:
        3px solid #63b3ff;

    outline-offset:
        3px;

    border-radius:
        12px;
}

.gauge-background,
.gauge-progress {
    fill: none;

    stroke-width: 15;

    stroke-linecap:
        round;
}

.gauge-background {
    stroke: #273645;
}

.gauge-progress {
    stroke: #56c8ff;

    filter:
        drop-shadow(
            0 0 6px
            rgba(
                86,
                200,
                255,
                0.55
            )
        );

    transition:
        stroke-dasharray
        100ms linear;
}

.gauge-mark {
    stroke: #516478;

    stroke-width: 2;

    transform-origin:
        110px 115px;
}

.gauge-mark.major {
    stroke: #8ba1b7;

    stroke-width: 3;
}

.gauge-needle {
    stroke: #ffcf4a;

    stroke-width: 5;

    stroke-linecap:
        round;

    transform-origin:
        110px 115px;

    transition:
        transform
        100ms linear;

    filter:
        drop-shadow(
            0 0 4px
            rgba(
                255,
                207,
                74,
                0.65
            )
        );
}

.gauge-center {
    fill: #edf4ff;

    stroke: #1a2634;

    stroke-width: 4;
}

.gauge-number {
    fill: #edf4ff;

    font-size: 27px;

    font-weight: 900;

    pointer-events: none;
}

.gauge-unit {
    fill: #9aacbf;

    font-size: 10px;

    font-weight: 700;

    pointer-events: none;
}

.gauge-limit,
.gauge-middle {
    fill: #8194a8;

    font-size: 12px;

    font-weight: 700;

    pointer-events: none;
}


/* ESTADO ÚNICO */

.status {
    min-width: 118px;

    box-sizing: border-box;

    padding:
        8px 14px;

    text-align: center;

    border-radius:
        999px;

    font-size:
        0.76rem;

    font-weight: 900;

    letter-spacing:
        0.06rem;
}

.status-forward {
    color: #d8ffe5;

    background:
        #14733b;

    box-shadow:
        0 0 16px
        rgba(
            32,
            214,
            103,
            0.35
        );
}

.status-reverse {
    color: #fff0c8;

    background:
        #9d630d;

    box-shadow:
        0 0 16px
        rgba(
            255,
            174,
            44,
            0.30
        );
}

.status-stopped {
    color: #ffdede;

    background:
        #7c2020;
}


/* CONTROLES DE MARCHA */

.drive-controls {
    display: grid;

    grid-template-columns:
        2fr 1fr;

    gap: 14px;

    margin-top:
        26px;
}

.direction-buttons {
    display: grid;

    grid-template-columns:
        1fr 1fr;

    gap: 12px;
}

.drive-controls button {
    min-height:
        58px;

    border: 0;

    border-radius:
        14px;

    color: white;

    font-size:
        1rem;

    font-weight:
        900;

    letter-spacing:
        0.06rem;

    cursor: pointer;

    transition:
        transform
            100ms ease,

        filter
            100ms ease,

        box-shadow
            100ms ease;
}

.forward-button {
    background:
        linear-gradient(
            145deg,
            #23a84e,
            #0b6d2d
        );
}

.reverse-button {
    background:
        linear-gradient(
            145deg,
            #e69a29,
            #965806
        );
}

.stop-button {
    background:
        linear-gradient(
            145deg,
            #ec3e3e,
            #961313
        );

    box-shadow:
        0 8px 18px
        rgba(
            224,
            47,
            47,
            0.25
        );
}

.direction-buttons
button.selected {
    box-shadow:
        0 0 0 3px
            rgba(
                255,
                255,
                255,
                0.80
            ),

        0 0 20px
            rgba(
                80,
                210,
                255,
                0.55
            );
}

.drive-controls
button:hover {
    filter:
        brightness(1.12);
}

.drive-controls
button:active {
    transform:
        scale(0.97);
}


/* ESTADO DE MOTORES */

.motor-state {
    display: grid;

    grid-template-columns:
        1fr 1fr;

    gap: 12px;

    margin-top:
        18px;
}

.motor-card {
    display: flex;

    flex-direction:
        column;

    align-items:
        center;

    padding:
        11px;

    border:
        1px solid #34475d;

    border-radius:
        12px;

    background:
        rgba(
            255,
            255,
            255,
            0.04
        );
}

.motor-card span,
.motor-card small {
    color: #9fb0c3;

    font-size:
        0.8rem;
}

.motor-card strong {
    margin:
        4px 0;

    font-size:
        1.25rem;
}


/* SEGURIDAD */

.safety-note {
    margin-top:
        14px;

    padding:
        9px;

    text-align:
        center;

    color:
        #aebfd1;

    background:
        rgba(
            255,
            255,
            255,
            0.035
        );

    border-radius:
        10px;

    font-size:
        0.78rem;
}


/* FOCUS */

.center-button:focus-visible,
.drive-controls
button:focus-visible {
    outline:
        3px solid #63b3ff;

    outline-offset:
        3px;
}


/* TABLET */

@media
(max-width: 760px) {

    .instruments {
        grid-template-columns:
            1fr;

        align-items:
            center;

        gap:
            22px;
    }

    .instrument-section {
        grid-template-rows:
            auto
            32px
            44px;
    }

    .gauge {
        max-width:
            310px;
    }
}


/* SMARTPHONE */

@media
(max-width: 480px) {

    .robot-controller {
        padding:
            14px;
    }

    .wheel {
        width:
            min(
                250px,
                72vw
            );

        border-width:
            18px;
    }

    .gauge {
        max-width:
            280px;
    }

    .drive-controls {
        grid-template-columns:
            1fr;
    }

    .direction-buttons {
        grid-template-columns:
            1fr 1fr;
    }

    .motor-state {
        grid-template-columns:
            1fr;
    }
}
</style>