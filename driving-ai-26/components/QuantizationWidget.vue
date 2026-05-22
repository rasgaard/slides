<script setup>
import { ref, computed } from 'vue'

// 10 representative weight values
const BASE = [0.23, -0.41, 0.87, -0.12, 0.55, -0.73, 0.19, -0.35, 0.62, -0.28]
const OUTLIER_VAL = 47.3

// Fixed SVG view range — axis never moves, quantization grid changes
const VIEW_MIN = -1.5
const VIEW_MAX =  1.5
const W = 530, H = 94, PAD = 24
const AXIS_W = W - 2 * PAD
const toX     = v => PAD + ((v - VIEW_MIN) / (VIEW_MAX - VIEW_MIN)) * AXIS_W
const ZERO_X  = toX(0)  // constant

const showOutlier = ref(false)
const values  = computed(() => showOutlier.value ? [...BASE, OUTLIER_VAL] : BASE)
const maxAbs  = computed(() => Math.max(...values.value.map(v => Math.abs(v))))
const scale   = computed(() => maxAbs.value / 127)

const quantize   = x => Math.max(-128, Math.min(127, Math.round(x / scale.value)))
const dequantize = q => q * scale.value

const rows = computed(() => values.value.map(x => ({
  x,
  q:         quantize(x),
  xhat:      dequantize(quantize(x)),
  err:       Math.abs(x - dequantize(quantize(x))),
  isOutlier: showOutlier.value && x === OUTLIER_VAL,
})))

// INT8 dequantized positions that fall within the fixed view range
const visibleTicks = computed(() => {
  const ts = []
  for (let q = -128; q <= 127; q++) {
    const v = dequantize(q)
    if (v >= VIEW_MIN && v <= VIEW_MAX) ts.push(toX(v))
  }
  return ts
})

// Base value dots — react to scale changes (dequantized positions move)
const baseDots = computed(() => BASE.map(x => {
  const fX  = toX(x)
  const qX  = Math.max(PAD, Math.min(W - PAD, toX(dequantize(quantize(x)))))
  const err = Math.abs(x - dequantize(quantize(x)))
  return { fX, qX, err }
}))

const baseRowsOnly  = computed(() => rows.value.filter(r => !r.isOutlier))
const maxErr        = computed(() => Math.max(...baseRowsOnly.value.map(r => r.err)))
const usedLevels    = computed(() => new Set(baseRowsOnly.value.map(r => r.q)).size)
</script>

<template>
  <div class="font-sans select-none text-sm">

    <!-- Toggle + scale readout -->
    <div class="flex items-center gap-3 mb-3">
      <span class="text-xs text-gray-400">weights:</span>
      <button
        @click="showOutlier = false"
        :class="['px-3 py-0.5 rounded text-xs border cursor-pointer transition-colors',
                 !showOutlier ? 'bg-blue-500 text-white border-blue-500'
                              : 'bg-white text-gray-600 border-gray-300 hover:bg-gray-50']">
        Normal
      </button>
      <button
        @click="showOutlier = true"
        :class="['px-3 py-0.5 rounded text-xs border cursor-pointer transition-colors',
                 showOutlier ? 'bg-orange-500 text-white border-orange-500'
                             : 'bg-white text-gray-600 border-gray-300 hover:bg-gray-50']">
        ⚠ With outlier ({{ OUTLIER_VAL }})
      </button>
      <div class="ml-auto text-xs font-mono text-gray-500">
        scale = <span class="text-gray-800">{{ scale.toFixed(5) }}</span>
      </div>
    </div>

    <!-- Fixed-axis number line — axis stays, INT8 grid changes -->
    <svg :width="W" :height="H" class="block overflow-visible">
      <!-- INT8 quantization levels (ticks within view) -->
      <line v-for="(tx, i) in visibleTicks" :key="i"
        :x1="tx" y1="34" :x2="tx" y2="56"
        stroke="#bfdbfe" stroke-width="1.2" />

      <!-- Axis -->
      <line :x1="PAD" y1="44" :x2="W - PAD" y2="44" stroke="#cbd5e1" stroke-width="1.5" />

      <!-- Zero marker -->
      <line :x1="ZERO_X" y1="26" :x2="ZERO_X" y2="64"
        stroke="#94a3b8" stroke-width="1.5" stroke-dasharray="3,2" />
      <text :x="ZERO_X" :y="22" text-anchor="middle" font-size="9" fill="#94a3b8">0</text>

      <!-- L-shaped connectors: FP16 → quantized (orange, hidden when error ≈ 0) -->
      <path v-for="(d, i) in baseDots" :key="'c'+i"
        :d="`M ${d.fX} 30 L ${d.fX} 44 L ${d.qX} 44 L ${d.qX} 62`"
        fill="none"
        stroke="#f97316"
        stroke-width="1.5"
        stroke-linejoin="round"
        :opacity="d.err > 0.002 ? 0.65 : 0" />

      <!-- Blue dots: original FP16 values (fixed) -->
      <circle v-for="(d, i) in baseDots" :key="'f'+i"
        :cx="d.fX" cy="26" r="4.5"
        fill="#3b82f6" stroke="white" stroke-width="1.5" />

      <!-- Green dots: dequantized positions (move with scale) -->
      <circle v-for="(d, i) in baseDots" :key="'q'+i"
        :cx="d.qX" cy="66" r="3.5"
        fill="#10b981" stroke="white" stroke-width="1.5" opacity="0.9" />

      <!-- Fixed axis labels -->
      <text :x="PAD" y="86" text-anchor="middle" font-size="9" fill="#94a3b8" font-family="monospace">
        {{ VIEW_MIN }}
      </text>
      <text :x="W - PAD" y="86" text-anchor="middle" font-size="9" fill="#94a3b8" font-family="monospace">
        {{ VIEW_MAX }}
      </text>

      <!-- Outlier off-screen annotation -->
      <text v-if="showOutlier" :x="W - PAD - 4" y="20" text-anchor="end" font-size="9" fill="#ef4444">
        outlier {{ OUTLIER_VAL }} lies far outside →
      </text>
    </svg>

    <!-- Legend + INT8 usage summary -->
    <div class="flex items-center gap-5 text-xs text-gray-500 mt-1 mb-3">
      <span class="flex items-center gap-1">
        <svg width="10" height="10"><circle cx="5" cy="5" r="4.5" fill="#3b82f6"/></svg>
        original FP16
      </span>
      <span class="flex items-center gap-1">
        <svg width="10" height="10"><circle cx="5" cy="5" r="3.5" fill="#10b981"/></svg>
        dequantized
      </span>
      <span class="flex items-center gap-1">
        <svg width="16" height="10"><path d="M 1 3 L 1 6 L 15 6 L 15 9" fill="none" stroke="#f97316" stroke-width="1.5"/></svg>
        quantization error
      </span>
      <span class="flex items-center gap-1">
        <svg width="14" height="8"><line x1="0" y1="4" x2="14" y2="4" stroke="#bfdbfe" stroke-width="2.5"/></svg>
        INT8 levels
      </span>
      <span class="ml-auto font-semibold" :class="showOutlier ? 'text-orange-600' : 'text-gray-500'">
        {{ visibleTicks.length }}/256 INT8 levels in view
        <template v-if="showOutlier">— only {{ usedLevels }} used for base weights!</template>
      </span>
    </div>

    <!-- Formula box + scrollable table -->
    <div class="flex gap-4">

      <!-- Formula -->
      <div class="bg-gray-50 rounded px-3 py-2 font-mono text-xs flex-none w-48">
        <div class="text-gray-400 font-sans text-xs mb-1.5">Symmetric INT8</div>
        <div class="mb-0.5 text-gray-700">s = max|x| / 127</div>
        <div class="mb-0.5 text-blue-600">q = ⌊ x / s ⌉</div>
        <div class="mb-2 text-emerald-600">x̂ = q × s</div>
        <div class="border-t border-gray-200 pt-1.5 font-sans text-gray-500 space-y-0.5">
          <div>scale:
            <span class="font-mono text-gray-800 ml-1">{{ scale.toFixed(5) }}</span>
          </div>
          <div>max Δ:
            <span class="font-mono ml-1"
              :class="showOutlier && maxErr > 0.01 ? 'text-orange-600 font-bold' : 'text-gray-800'">
              {{ maxErr.toFixed(5) }}
            </span>
          </div>
        </div>
      </div>

      <!-- Table -->
      <div class="flex-1 overflow-y-auto" style="max-height: 148px">
        <table class="text-xs font-mono w-full">
          <thead class="sticky top-0 bg-white">
            <tr class="text-gray-400 border-b border-gray-100">
              <th class="text-left pr-2 pb-1 font-normal">FP16 x</th>
              <th class="text-right pr-2 pb-1 font-normal">INT8 q</th>
              <th class="text-right pr-2 pb-1 font-normal">x̂ (deq.)</th>
              <th class="text-right pb-1 font-normal">Δ</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(r, i) in rows" :key="i"
              :class="r.isOutlier
                ? 'text-red-500'
                : r.err > 0.05 ? 'text-orange-600'
                : 'text-gray-700'">
              <td class="pr-2 py-px">{{ r.x.toFixed(3) }}</td>
              <td class="text-right pr-2 py-px">{{ r.q }}</td>
              <td class="text-right pr-2 py-px">{{ r.xhat.toFixed(4) }}</td>
              <td class="text-right py-px" :class="r.err > 0.1 ? 'font-bold' : ''">
                {{ r.err.toFixed(4) }}
              </td>
            </tr>
          </tbody>
        </table>
      </div>

    </div>
  </div>
</template>
