<script setup>
import { ref, computed } from 'vue'

const presets = [
  { label: 'π',    val: 3.14159 },
  { label: '0.1',  val: 0.1 },
  { label: '1/3',  val: 1/3 },
  { label: '1000', val: 1000 },
  { label: '-2.5', val: -2.5 },
]

const inputValue = ref(3.14159)

function toFp16(value) {
  const buf = new ArrayBuffer(4)
  new Float32Array(buf)[0] = value
  const u32 = new Uint32Array(buf)[0]
  const sign  = (u32 >>> 31) & 1
  const exp32 = (u32 >>> 23) & 0xFF
  const frac32 = u32 & 0x7FFFFF

  if (exp32 === 0xFF)
    return (sign << 15) | (0x1F << 10) | (frac32 ? 0x200 : 0)

  const exp16 = exp32 - 127 + 15
  if (exp16 >= 31) return (sign << 15) | (0x1F << 10)
  if (exp16 <= 0) {
    if (exp16 < -10) return sign << 15
    return (sign << 15) | ((frac32 | 0x800000) >> (1 - exp16 + 13))
  }
  return (sign << 15) | (exp16 << 10) | (frac32 >> 13)
}

function fromFp16(bits) {
  const sign = (bits >>> 15) & 1
  const exp  = (bits >>> 10) & 0x1F
  const frac = bits & 0x3FF
  if (exp === 0x1F) return frac ? NaN : (sign ? -Infinity : Infinity)
  if (exp === 0)    return (sign ? -1 : 1) * 2 ** -14 * (frac / 1024)
  return (sign ? -1 : 1) * 2 ** (exp - 15) * (1 + frac / 1024)
}

const bits     = computed(() => toFp16(Number(inputValue.value)))
const signBit  = computed(() => (bits.value >>> 15) & 1)
const expBits  = computed(() => Array.from({ length: 5  }, (_, i) => ((bits.value >>> 10) >>> (4 - i)) & 1))
const fracBits = computed(() => Array.from({ length: 10 }, (_, i) => (bits.value >>> (9 - i)) & 1))
const expRaw   = computed(() => (bits.value >>> 10) & 0x1F)
const fracRaw  = computed(() => bits.value & 0x3FF)
const decoded  = computed(() => fromFp16(bits.value))

const decodedStr = computed(() => {
  const v = decoded.value
  if (isNaN(v))       return 'NaN'
  if (!isFinite(v))   return v > 0 ? '∞' : '-∞'
  return v.toPrecision(7)
})

const delta = computed(() => {
  const orig = Number(inputValue.value)
  const dec  = decoded.value
  if (!isFinite(orig) || !isFinite(dec)) return null
  const d = Math.abs(orig - dec)
  return d < 1e-15 ? null : d.toExponential(3)
})

const biasedExp  = computed(() => expRaw.value === 0 ? -14 : expRaw.value - 15)
const leadingBit = computed(() => expRaw.value === 0 ? '0' : '1')
</script>

<template>
  <div class="font-mono select-none text-sm">

    <!-- Input + presets -->
    <div class="flex items-center gap-3 mb-4 font-sans">
      <span class="text-gray-500 text-xs">Value</span>
      <input v-model.number="inputValue" type="number" step="any"
        class="border border-gray-300 rounded px-2 py-0.5 w-28 font-mono text-sm" />
      <div class="flex gap-1.5">
        <button v-for="p in presets" :key="p.label" @click="inputValue = p.val"
          class="px-2 py-0.5 text-xs border border-gray-300 rounded hover:bg-gray-100 cursor-pointer font-mono">
          {{ p.label }}
        </button>
      </div>
    </div>

    <!-- Bit cells -->
    <div class="flex items-center gap-0.5 mb-1">
      <div class="w-8 h-9 flex items-center justify-center rounded text-white font-bold bg-red-400 text-base">
        {{ signBit }}
      </div>
      <div class="w-2" />
      <div v-for="(b, i) in expBits" :key="'e'+i"
        class="w-8 h-9 flex items-center justify-center rounded font-bold text-base"
        :class="b ? 'bg-blue-500 text-white' : 'bg-blue-100 text-blue-300'">
        {{ b }}
      </div>
      <div class="w-2" />
      <div v-for="(b, i) in fracBits" :key="'f'+i"
        class="w-8 h-9 flex items-center justify-center rounded font-bold text-base"
        :class="b ? 'bg-emerald-500 text-white' : 'bg-emerald-100 text-emerald-300'">
        {{ b }}
      </div>
    </div>

    <!-- Bit index labels -->
    <div class="flex items-center gap-0.5 text-xs text-gray-400 mb-4">
      <div class="w-8 text-center">S</div>
      <div class="w-2" />
      <div v-for="i in 5"  :key="'el'+i" class="w-8 text-center">E{{ 5 - i }}</div>
      <div class="w-2" />
      <div v-for="i in 10" :key="'fl'+i" class="w-8 text-center">M{{ 10 - i }}</div>
    </div>

    <!-- Legend -->
    <div class="flex gap-6 text-xs font-sans text-gray-600 mb-4">
      <span>
        <span class="inline-block w-2.5 h-2.5 bg-red-400 rounded-sm mr-1 align-middle" />
        Sign
      </span>
      <span>
        <span class="inline-block w-2.5 h-2.5 bg-blue-400 rounded-sm mr-1 align-middle" />
        Exponent — 5 bits, bias&nbsp;15 &nbsp;→&nbsp; {{ expRaw }} − 15 = {{ biasedExp }}
      </span>
      <span>
        <span class="inline-block w-2.5 h-2.5 bg-emerald-400 rounded-sm mr-1 align-middle" />
        Mantissa — 10 bits &nbsp;=&nbsp; {{ fracRaw }}
      </span>
    </div>

    <!-- Decoded formula + result -->
    <div class="bg-gray-50 rounded px-3 py-2 font-sans text-sm">
      <span class="text-gray-500">
        (−1)<sup>{{ signBit }}</sup>
        &thinsp;×&thinsp; 2<sup>{{ biasedExp }}</sup>
        &thinsp;×&thinsp; ({{ leadingBit }} + {{ fracRaw }}/1024)
        &thinsp;=&thinsp;
      </span>
      <span class="font-bold text-blue-700 text-base">{{ decodedStr }}</span>
      <span v-if="delta" class="text-orange-500 ml-4 text-xs">
        precision loss: Δ = {{ delta }}
      </span>
    </div>

  </div>
</template>
