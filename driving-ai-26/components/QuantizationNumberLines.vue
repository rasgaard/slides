<script setup>
import { ref, onMounted } from 'vue'

const container = ref(null)

// Shared promise so tikzjax is loaded only once across all instances
let tikzjaxReady = null

function loadTikzJax() {
  if (tikzjaxReady) return tikzjaxReady
  tikzjaxReady = new Promise((resolve, reject) => {
    const link = document.createElement('link')
    link.rel = 'stylesheet'
    link.href = 'https://tikzjax.com/v1/fonts.css'
    document.head.appendChild(link)

    const s = document.createElement('script')
    s.src = 'https://tikzjax.com/v1/tikzjax.js'
    s.onload = resolve
    s.onerror = reject
    document.head.appendChild(s)
  })
  return tikzjaxReady
}

// FP32 value 0.55 → x = (0.55+0.87)/1.74*6 = 4.897 ≈ 4.9
// INT8 value  80  → x = (80+128)/255*6    = 4.894 ≈ 4.9  (same position ✓)
const tikzCode = `
\\begin{tikzpicture}[scale=0.95]
  % FP32 number line (continuous, blue)
  \\node[anchor=east, blue, font=\\small\\bfseries] at (-0.3, 0.8) {FP32};
  \\draw[blue!60, line width=1.5pt] (0,0.8) -- (6,0.8);
  \\foreach \\x/\\lbl in {0/{$-$0.87}, 3/{0}, 6/{0.87}} {
    \\draw[gray!70] (\\x,0.55)--(\\x,1.05);
    \\node[above, gray!70, font=\\tiny] at (\\x,1.05) {\\lbl};
  }
  \\fill[blue!80] (4.9,0.8) circle (3pt);
  \\node[above, blue!80, font=\\small\\bfseries] at (4.9,1.15) {0.55};

  % INT8 number line (discrete ticks, teal)
  \\node[anchor=east, teal, font=\\small\\bfseries] at (-0.3,-0.8) {INT8};
  \\draw[teal!60, line width=1.5pt] (0,-0.8) -- (6,-0.8);
  \\foreach \\i in {0,1,...,63} {
    \\draw[teal!40] ({\\i*6/63},-0.94)--({\\i*6/63},-0.66);
  }
  \\foreach \\x/\\lbl in {0/{$-$128}, 3/{0}, 6/{127}} {
    \\draw[gray!70] (\\x,-1.1)--(\\x,-0.5);
    \\node[below, gray!70, font=\\tiny] at (\\x,-1.1) {\\lbl};
  }
  \\fill[teal!80!black] (4.9,-0.8) circle (3pt);
  \\node[below, teal!80!black, font=\\small\\bfseries] at (4.9,-1.15) {80};

  % Mapping arrow
  \\draw[->, orange, thick, dashed] (4.9,0.5) -- (4.9,-0.5)
    node[midway, right, orange, font=\\scriptsize] {$\\lfloor x/s \\rceil$};
\\end{tikzpicture}
`

onMounted(async () => {
  await loadTikzJax()
  const el = document.createElement('script')
  el.type = 'text/tikz'
  el.textContent = tikzCode
  container.value.appendChild(el)
  // tikzjax watches for new <script type="text/tikz"> via MutationObserver
})
</script>

<template>
  <div ref="container" class="[&_svg]:max-w-full [&_svg]:h-auto" />
</template>
