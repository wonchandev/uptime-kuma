<template>
    <div v-if="serviceCards.length > 0">
        <div class="section-label">{{ $t("metricPerService") }}</div>

        <div class="service-grid">
            <div v-for="card in serviceCards" :key="card.id" class="service-card">
                <div class="card-head">
                    <span class="status-dot" :class="card.statusClass"></span>
                    <span class="card-name">{{ card.name }}</span>
                </div>

                <div class="card-value">
                    <template v-if="card.lastPing !== null">
                        {{ card.lastPing }}<span class="card-unit">ms</span>
                    </template>
                    <template v-else>–</template>
                </div>

                <div
                    :ref="'chart' + card.id"
                    class="card-chart"
                    @mousemove="onChartMove($event, card)"
                    @mouseleave="hover = null"
                >
                    <svg
                        v-if="card.points.length > 1"
                        class="chart-svg"
                        :viewBox="`0 0 ${chartW} ${chartH}`"
                        preserveAspectRatio="none"
                        aria-hidden="true"
                    >
                        <defs>
                            <linearGradient :id="'cardGrad' + card.id" x1="0" y1="0" x2="0" y2="1">
                                <stop offset="0%" stop-color="currentcolor" stop-opacity="0.2" />
                                <stop offset="100%" stop-color="currentcolor" stop-opacity="0" />
                            </linearGradient>
                        </defs>
                        <path :d="card.areaPath" :fill="`url(#cardGrad${card.id})`" stroke="none" />
                        <path :d="card.linePath" class="chart-line" />
                    </svg>
                    <div v-else class="chart-empty">
                        <span class="chart-empty-dot"></span>
                        {{ $t("metricChartEmpty") }}
                    </div>

                    <!-- Live pulse on the most recent point -->
                    <div v-if="card.points.length > 1" class="chart-last-dot" :style="lastPointStyle(card)">
                        <span class="pulse"></span>
                    </div>

                    <!-- Hover crosshair -->
                    <template v-if="hover && hover.id === card.id">
                        <div class="chart-crosshair" :style="{ left: hover.xPct + '%' }"></div>
                        <div class="chart-dot" :style="{ left: hover.xPct + '%', top: hover.yPct + '%' }"></div>
                        <div class="chart-tooltip" :style="{ left: hover.clampedPct + '%' }">
                            <span class="tooltip-time">{{ hover.timeLabel }}</span>
                            <span class="tooltip-value">{{ hover.value }}ms</span>
                        </div>
                    </template>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import dayjs from "dayjs";
import { DOWN, MAINTENANCE, PENDING, UP } from "../util.ts";

const CHART_WIDTH = 280;
const CHART_HEIGHT = 64;
const CHART_PAD_Y = 8;
const CHART_BUCKETS = 30;

export default {
    data() {
        return {
            chartW: CHART_WIDTH,
            chartH: CHART_HEIGHT,
            hover: null,
        };
    },

    computed: {
        /**
         * One card per public monitor, in page order — every service gets
         * its own response-time series (never merged).
         * @returns {object[]} Card data with chart paths and latest ping
         */
        serviceCards() {
            const cards = [];

            for (const group of this.$root.publicGroupList || []) {
                for (const monitor of group.monitorList || []) {
                    const beats = this.$root.heartbeatList[monitor.id] || [];
                    const points = this.buildSeries(beats);
                    const lastBeat = beats[beats.length - 1];

                    cards.push({
                        id: monitor.id,
                        name: monitor.name,
                        points,
                        linePath: this.smoothPath(points),
                        areaPath: this.areaPath(points),
                        lastPing: typeof lastBeat?.ping === "number" ? lastBeat.ping : null,
                        statusClass: this.statusClass(lastBeat),
                    });
                }
            }

            return cards;
        },
    },

    methods: {
        /**
         * Map a heartbeat status to a dot class
         * @param {object|undefined} beat Last heartbeat of the monitor
         * @returns {string} CSS class name
         */
        statusClass(beat) {
            switch (Number(beat?.status)) {
                case UP:
                    return "dot-up";
                case DOWN:
                    return "dot-down";
                case PENDING:
                    return "dot-pending";
                case MAINTENANCE:
                    return "dot-maintenance";
                default:
                    return "dot-unknown";
            }
        },

        /**
         * Bucket one monitor's heartbeats into chart coordinates.
         * Prefers the last hour; falls back to the whole heartbeat window
         * when there are not enough recent samples yet.
         * @param {object[]} beats Heartbeat list of a monitor
         * @returns {Array<{ x: number, y: number, value: number, t: number }>} Points
         */
        buildSeries(beats) {
            const cutoff = dayjs.utc().subtract(1, "hour").valueOf();
            const pings = [];

            for (const beat of beats) {
                if (beat && Number(beat.status) === UP && typeof beat.ping === "number") {
                    pings.push({
                        t: dayjs.utc(beat.time).valueOf(),
                        ping: beat.ping,
                    });
                }
            }

            pings.sort((a, b) => a.t - b.t);

            const lastHour = pings.filter((p) => p.t >= cutoff);
            const source = lastHour.length >= 5 ? lastHour : pings;

            if (source.length < 2) {
                return [];
            }

            const minT = source[0].t;
            const maxT = source[source.length - 1].t;
            const span = Math.max(1, maxT - minT);

            const buckets = [];
            for (let i = 0; i < CHART_BUCKETS; i++) {
                buckets.push({ sum: 0, tSum: 0, count: 0 });
            }
            for (const p of source) {
                const idx = Math.min(CHART_BUCKETS - 1, Math.floor(((p.t - minT) / span) * CHART_BUCKETS));
                buckets[idx].sum += p.ping;
                buckets[idx].tSum += p.t;
                buckets[idx].count++;
            }

            const series = [];
            buckets.forEach((b, i) => {
                if (b.count > 0) {
                    series.push({
                        bucket: i,
                        value: b.sum / b.count,
                        t: b.tSum / b.count,
                    });
                }
            });

            if (series.length < 2) {
                return [];
            }

            let min = Infinity;
            let max = -Infinity;
            for (const s of series) {
                min = Math.min(min, s.value);
                max = Math.max(max, s.value);
            }

            // Flat data would otherwise collapse onto the baseline — pad the
            // domain so the line sits in the middle of the chart
            if (max - min < Math.max(2, max * 0.05)) {
                const pad = Math.max(1, max * 0.1);
                min -= pad;
                max += pad;
            }

            const range = Math.max(1, max - min);
            const usableH = CHART_HEIGHT - CHART_PAD_Y * 2;
            const firstBucket = series[0].bucket;
            const lastBucket = series[series.length - 1].bucket;
            const bucketSpan = Math.max(1, lastBucket - firstBucket);

            return series.map((s) => ({
                x: +(((s.bucket - firstBucket) / bucketSpan) * CHART_WIDTH).toFixed(2),
                y: +(CHART_PAD_Y + usableH - ((s.value - min) / range) * usableH).toFixed(2),
                value: Math.round(s.value),
                t: s.t,
            }));
        },

        /**
         * Build a smooth cubic bezier path (Catmull-Rom) through points.
         * @param {Array<{ x: number, y: number }>} pts Points to connect
         * @returns {string} SVG path definition
         */
        smoothPath(pts) {
            if (pts.length < 2) {
                return "";
            }

            let d = `M ${pts[0].x} ${pts[0].y}`;
            for (let i = 0; i < pts.length - 1; i++) {
                const p0 = pts[i - 1] || pts[i];
                const p1 = pts[i];
                const p2 = pts[i + 1];
                const p3 = pts[i + 2] || p2;

                const cp1x = +(p1.x + (p2.x - p0.x) / 6).toFixed(2);
                const cp1y = +(p1.y + (p2.y - p0.y) / 6).toFixed(2);
                const cp2x = +(p2.x - (p3.x - p1.x) / 6).toFixed(2);
                const cp2y = +(p2.y - (p3.y - p1.y) / 6).toFixed(2);

                d += ` C ${cp1x} ${cp1y}, ${cp2x} ${cp2y}, ${p2.x} ${p2.y}`;
            }
            return d;
        },

        /**
         * Close a line path down to the baseline for the gradient fill.
         * @param {Array<{ x: number, y: number }>} pts Points to connect
         * @returns {string} SVG path definition
         */
        areaPath(pts) {
            if (pts.length < 2) {
                return "";
            }
            const line = this.smoothPath(pts);
            return `${line} L ${pts[pts.length - 1].x} ${CHART_HEIGHT} L ${pts[0].x} ${CHART_HEIGHT} Z`;
        },

        /**
         * Position of the most recent point, as percentages of the chart box.
         * @param {object} card Card whose chart is rendered
         * @returns {object} Style object
         */
        lastPointStyle(card) {
            const last = card.points[card.points.length - 1];
            if (!last) {
                return {};
            }
            return {
                left: (last.x / CHART_WIDTH) * 100 + "%",
                top: (last.y / CHART_HEIGHT) * 100 + "%",
            };
        },

        /**
         * Track the nearest chart point under the cursor for a card.
         * @param {MouseEvent} event Mouse event
         * @param {object} card Card being hovered
         * @returns {void}
         */
        onChartMove(event, card) {
            if (card.points.length === 0) {
                this.hover = null;
                return;
            }

            const el = this.$refs["chart" + card.id];
            const chartEl = Array.isArray(el) ? el[0] : el;
            if (!chartEl) {
                return;
            }

            const rect = chartEl.getBoundingClientRect();
            const x = ((event.clientX - rect.left) / rect.width) * CHART_WIDTH;

            let nearest = card.points[0];
            for (const p of card.points) {
                if (Math.abs(p.x - x) < Math.abs(nearest.x - x)) {
                    nearest = p;
                }
            }

            const xPct = (nearest.x / CHART_WIDTH) * 100;

            this.hover = {
                id: card.id,
                value: nearest.value,
                timeLabel: dayjs(nearest.t).format("HH:mm"),
                xPct,
                yPct: (nearest.y / CHART_HEIGHT) * 100,
                clampedPct: Math.max(18, Math.min(82, xPct)),
            };
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.section-label {
    font-size: 16px;
    font-weight: 700;
    letter-spacing: -0.2px;
    color: #4e5968;
    margin-bottom: 14px;
    padding-left: 4px;

    .dark & {
        color: #c3c9d1;
    }
}

.service-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 16px;

    @media (max-width: 550px) {
        grid-template-columns: 1fr;
    }
}

.service-card {
    background-color: #fff;
    border-radius: 18px;
    padding: 16px 18px 14px;
    box-shadow: 0 1px 2px rgba(23, 32, 42, 0.04), 0 8px 24px rgba(23, 32, 42, 0.05);
    color: #017531;
    transition: transform 0.2s $easing-in, box-shadow 0.2s $easing-in;

    &:hover {
        transform: translateY(-1px);
        box-shadow: 0 2px 4px rgba(23, 32, 42, 0.04), 0 10px 28px rgba(23, 32, 42, 0.08);
    }

    .dark & {
        background-color: #1e1e24;
        box-shadow: none;
        color: #12a35b;
    }
}

.card-head {
    display: flex;
    align-items: center;
    gap: 8px;
    min-width: 0;
}

.status-dot {
    flex-shrink: 0;
    width: 8px;
    height: 8px;
    border-radius: 50%;

    &.dot-up {
        background-color: #017531;
        box-shadow: 0 0 0 3px rgba(1, 117, 49, 0.14);

        .dark & {
            background-color: #12a35b;
            box-shadow: 0 0 0 3px rgba(18, 163, 91, 0.18);
        }
    }

    &.dot-down {
        background-color: $danger;
        box-shadow: 0 0 0 3px rgba(220, 53, 69, 0.15);
    }

    &.dot-pending {
        background-color: $warning;
        box-shadow: 0 0 0 3px rgba(248, 163, 6, 0.15);
    }

    &.dot-maintenance {
        background-color: $maintenance;
        box-shadow: 0 0 0 3px rgba(23, 71, 245, 0.15);
    }

    &.dot-unknown {
        background-color: #adb5bd;
        box-shadow: 0 0 0 3px rgba(173, 181, 189, 0.15);
    }
}

.card-name {
    font-size: 13.5px;
    font-weight: 600;
    color: #7d8896;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    .dark & {
        color: #d8dce1;
    }
}

.card-value {
    margin-top: 6px;
    font-size: 28px;
    font-weight: 800;
    line-height: 1.1;
    letter-spacing: -0.5px;
    color: #191f28;
    font-variant-numeric: tabular-nums;

    .dark & {
        color: #fff;
    }
}

.card-unit {
    font-size: 16px;
    font-weight: 700;
    margin-left: 2px;
    color: #6b7684;

    .dark & {
        color: #8b95a1;
    }
}

.card-chart {
    position: relative;
    margin-top: 10px;
    height: 64px;
    cursor: crosshair;
}

.chart-svg {
    display: block;
    width: 100%;
    height: 64px;
    overflow: visible;
}

.chart-line {
    fill: none;
    stroke: currentcolor;
    stroke-width: 2;
    stroke-linecap: round;
    stroke-linejoin: round;
    vector-effect: non-scaling-stroke;
}

.chart-last-dot {
    position: absolute;
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background-color: currentcolor;
    transform: translate(-50%, -50%);
    pointer-events: none;

    .pulse {
        position: absolute;
        inset: -5px;
        border-radius: 50%;
        background-color: currentcolor;
        opacity: 0.3;
        animation: metric-pulse 2s $easing-out infinite;
    }
}

@keyframes metric-pulse {
    0% {
        transform: scale(0.5);
        opacity: 0.35;
    }

    70% {
        transform: scale(1.4);
        opacity: 0;
    }

    100% {
        transform: scale(1.4);
        opacity: 0;
    }
}

.chart-crosshair {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 1px;
    background-color: rgba(139, 149, 161, 0.4);
    transform: translateX(-50%);
    pointer-events: none;
}

.chart-dot {
    position: absolute;
    width: 9px;
    height: 9px;
    border-radius: 50%;
    background-color: currentcolor;
    border: 2px solid #fff;
    transform: translate(-50%, -50%);
    pointer-events: none;

    .dark & {
        border-color: #1e1e24;
    }
}

.chart-tooltip {
    position: absolute;
    top: -32px;
    transform: translateX(-50%);
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 11.5px;
    padding: 4px 9px;
    border-radius: 7px;
    background-color: #191f28;
    pointer-events: none;
    white-space: nowrap;
    z-index: 5;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.18);

    .tooltip-time {
        color: #8b95a1;
        font-variant-numeric: tabular-nums;
    }

    .tooltip-value {
        color: #fff;
        font-weight: 700;
        font-variant-numeric: tabular-nums;
    }

    .dark & {
        background-color: #fff;

        .tooltip-time {
            color: #6b7684;
        }

        .tooltip-value {
            color: #191f28;
        }
    }
}

.chart-empty {
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    font-size: 12px;
    color: #8b95a1;
    border-radius: 12px;
    background-color: rgba(139, 149, 161, 0.06);
}

.chart-empty-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background-color: currentcolor;
    opacity: 0.7;
    animation: metric-pulse 1.6s $easing-out infinite;
}
</style>
