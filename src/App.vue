<script setup>
import { ref, computed } from 'vue'

// ==========================================
// CONSTANTES Y REGLAS EXACTAS
// ==========================================

const ENVIRONMENTS = [
  { value: 2, name: 'Espacio abierto' },
  { value: 3, name: 'Oficina estándar' },
  { value: 4, name: 'Entorno denso' }
]

const MATERIALS = {
  vidrio: {
    key: 'vidrio',
    name: 'Vidrio estándar',
    alpha: 0.15,
    pattern: 'glass'
  },
  tabiqueria: {
    key: 'tabiqueria',
    name: 'Tabiquería / Yeso',
    alpha: 0.25,
    pattern: 'drywall'
  },
  ladrillo: {
    key: 'ladrillo',
    name: 'Ladrillo estructural',
    alpha: 0.45,
    pattern: 'brick'
  },
  hormigon: {
    key: 'hormigon',
    name: 'Hormigón Armado',
    alpha: 0.75,
    pattern: 'concrete'
  }
}

// ==========================================
// ESTADO REACTIVO
// ==========================================

const activeTab = ref('distancia') // 'distancia' o 'obstaculos'
const chartScale = ref('dbm') // 'dbm' o 'mw'

// Valores base requeridos (Fijos)
const P0_base = -40 // Potencia base a 1 metro o inicial
const d0 = 1 // Distancia de referencia en metros

// Parámetros Modelo de Distancia
const distance = ref(15) // slider 1 a 50
const selectedEnvN = ref(2) // selector n

// Parámetros Modelo de Obstáculos
const wallsCount = ref(2) // slider 0 a 5
const selectedMaterialKey = ref('ladrillo')

// ==========================================
// CÁLCULOS EXACTOS (MODELOS)
// ==========================================

// Modelo 1: Pérdida por Distancia
// Pr(d) = Pr(d0) - 10 * n * log10(d / d0)
const rxPowerDistance = computed(() => {
  return P0_base - 10 * selectedEnvN.value * Math.log10(distance.value / d0)
})

const rxPowerDistanceMw = computed(() => {
  return Math.pow(10, rxPowerDistance.value / 10)
})

const derivativeDistanceDbm = computed(() => {
  return -(10 * selectedEnvN.value) / (Math.log(10) * distance.value)
})

const derivativeDistanceMw = computed(() => {
  return -(selectedEnvN.value * rxPowerDistanceMw.value) / distance.value
})

// Modelo 2: Impacto de Materiales
const currentMaterial = computed(() => MATERIALS[selectedMaterialKey.value])

// P(x) = P0 * e^(-alpha * x)
// Interpretado físicamente aplicando el decaimiento exponencial a la energía base
const rxPowerObstacles = computed(() => {
  const lossDb = 10 * currentMaterial.value.alpha * wallsCount.value
  return P0_base - lossDb
})

const rxPowerObstaclesMw = computed(() => {
  return Math.pow(10, rxPowerObstacles.value / 10)
})

// La derivada exacta de P_dBm(x) respecto a x es: d(P_dBm)/dx = -10 * alpha
const derivativeObstaclesDbm = computed(() => {
  return -10 * currentMaterial.value.alpha
})

// La derivada exacta de P_mW(x) respecto a x es: d(P_mW)/dx = -alpha * ln(10) * P_mW(x)
const derivativeObstaclesMw = computed(() => {
  return -currentMaterial.value.alpha * Math.log(10) * rxPowerObstaclesMw.value
})

// ==========================================
// ESTADO DE LA CONEXIÓN (NUEVOS UMBRALES)
// ==========================================

const currentRxPowerDbm = computed(() => {
  return activeTab.value === 'distancia' ? rxPowerDistance.value : rxPowerObstacles.value
})

const currentRxPowerMw = computed(() => {
  return activeTab.value === 'distancia' ? rxPowerDistanceMw.value : rxPowerObstaclesMw.value
})

const connectionStatus = computed(() => {
  const p = currentRxPowerDbm.value
  
  // Excelente si >= -65 dBm
  if (p >= -65) {
    return {
      label: 'Excelente',
      colorText: 'text-emerald-400',
      colorBg: 'bg-emerald-500/10',
      colorBorder: 'border-emerald-500/20',
      colorDot: 'bg-emerald-400',
      desc: 'Resultado >= -65 dBm'
    }
  } 
  // Inestable si está entre -66 y -79 dBm (usamos >= -79.99 para atrapar decimales)
  else if (p >= -79.99) {
    return {
      label: 'Inestable',
      colorText: 'text-amber-400',
      colorBg: 'bg-amber-500/10',
      colorBorder: 'border-amber-500/20',
      colorDot: 'bg-amber-400',
      desc: 'Resultado entre -66 y -79 dBm'
    }
  } 
  // Zona Muerta si cae bajo -80 dBm
  else {
    return {
      label: 'Zona Muerta',
      colorText: 'text-rose-400',
      colorBg: 'bg-rose-500/10',
      colorBorder: 'border-rose-500/20',
      colorDot: 'bg-rose-500',
      desc: 'Resultado < -80 dBm'
    }
  }
})

// ==========================================
// LÓGICA DE DIBUJO SVG
// ==========================================

const receiverX = computed(() => {
  if (activeTab.value === 'distancia') {
    return 150 + ((distance.value - 1) / 49) * 390
  } else {
    return 520
  }
})

const physicalWalls = computed(() => {
  if (activeTab.value !== 'obstaculos' || wallsCount.value === 0) return []
  const walls = []
  const count = wallsCount.value
  const spaceWidth = 340
  const startX = 140
  for (let i = 0; i < count; i++) {
    const x = startX + (i + 0.5) * (spaceWidth / count)
    walls.push({ x: x - 12, width: 24, height: 140, y: 30, index: i + 1 })
  }
  return walls
})

const waveFronts = computed(() => {
  const waves = []
  for (let r = 60; r <= 480; r += 60) {
    let powerAtWave = 0
    let isPastDevice = false
    let opacity = 0.85
    
    if (activeTab.value === 'distancia') {
      const distMeters = 1 + ((r - 90) / 390) * 49
      const validDist = Math.max(1, distMeters)
      powerAtWave = P0_base - 10 * selectedEnvN.value * Math.log10(validDist / d0)
      
      if (60 + r > receiverX.value) {
        isPastDevice = true
        opacity = 0.15
      } else {
        opacity = Math.max(0.2, 0.95 - (validDist / 60))
      }
    } else {
      const waveX = 60 + r
      let crossed = 0
      for (let i = 0; i < wallsCount.value; i++) {
        const wallX = 140 + (i + 0.5) * (340 / wallsCount.value)
        if (wallX < waveX) crossed++
      }
      powerAtWave = P0_base - (10 * currentMaterial.value.alpha * crossed)
      
      if (waveX > receiverX.value) {
        isPastDevice = true
        opacity = 0.1
      } else {
        opacity = Math.max(0.08, 0.9 * Math.exp(-0.45 * crossed))
      }
    }
    
    let color = 'stroke-emerald-400'
    if (powerAtWave < -65 && powerAtWave >= -80) {
      color = 'stroke-amber-400'
    } else if (powerAtWave < -80) {
      color = 'stroke-rose-500/60'
    }
    
    waves.push({
      path: `M ${60 + r * 0.5},${100 - r * 0.866} A ${r},${r} 0 0,1 ${60 + r * 0.5},${100 + r * 0.866}`,
      color,
      opacity,
      isPastDevice
    })
  }
  return waves
})

const mapX = (val, minVal, maxVal) => 50 + ((val - minVal) / (maxVal - minVal)) * 430

const mapY = (val) => {
  if (chartScale.value === 'dbm') {
    const maxDbm = -30
    const minDbm = -110
    const valClipped = Math.max(minDbm, Math.min(maxDbm, val))
    return 15 + ((maxDbm - valClipped) / (maxDbm - minDbm)) * 190
  } else {
    const maxMw = Math.pow(10, -30 / 10) // Escala visual mejorada
    const valClipped = Math.max(0, Math.min(maxMw, val))
    return 15 + (1 - (valClipped / maxMw)) * 190
  }
}

const chartPath = computed(() => {
  const points = []
  if (activeTab.value === 'distancia') {
    for (let i = 0; i <= 60; i++) {
      const d = 1 + (i / 60) * 49
      const pDbm = P0_base - 10 * selectedEnvN.value * Math.log10(d / d0)
      const x = mapX(d, 1, 50)
      const y = chartScale.value === 'dbm' ? mapY(pDbm) : mapY(Math.pow(10, pDbm / 10))
      points.push(`${x.toFixed(1)},${y.toFixed(1)}`)
    }
  } else {
    for (let i = 0; i <= 50; i++) {
      const n = (i / 50) * 5
      const pDbm = P0_base - (10 * currentMaterial.value.alpha * n)
      const x = mapX(n, 0, 5)
      const y = chartScale.value === 'dbm' ? mapY(pDbm) : mapY(Math.pow(10, pDbm / 10))
      points.push(`${x.toFixed(1)},${y.toFixed(1)}`)
    }
  }
  return `M ${points.join(' L ')}`
})

const activeDotCoords = computed(() => {
  if (activeTab.value === 'distancia') {
    return {
      x: mapX(distance.value, 1, 50),
      y: chartScale.value === 'dbm' ? mapY(rxPowerDistance.value) : mapY(rxPowerDistanceMw.value)
    }
  } else {
    return {
      x: mapX(wallsCount.value, 0, 5),
      y: chartScale.value === 'dbm' ? mapY(rxPowerObstacles.value) : mapY(rxPowerObstaclesMw.value)
    }
  }
})

const tangentLine = computed(() => {
  const coords = activeDotCoords.value
  const currentVal = activeTab.value === 'distancia' ? distance.value : wallsCount.value
  const maxVal = activeTab.value === 'distancia' ? 50 : 5
  const minVal = activeTab.value === 'distancia' ? 1 : 0
  const delta = maxVal * 0.01
  const val1 = Math.max(minVal, currentVal - delta)
  const val2 = Math.min(maxVal, currentVal + delta)
  
  let p1Dbm, p2Dbm
  if (activeTab.value === 'distancia') {
    p1Dbm = P0_base - 10 * selectedEnvN.value * Math.log10(val1 / d0)
    p2Dbm = P0_base - 10 * selectedEnvN.value * Math.log10(val2 / d0)
  } else {
    p1Dbm = P0_base - (10 * currentMaterial.value.alpha * val1)
    p2Dbm = P0_base - (10 * currentMaterial.value.alpha * val2)
  }
  
  const y1 = chartScale.value === 'dbm' ? mapY(p1Dbm) : mapY(Math.pow(10, p1Dbm / 10))
  const y2 = chartScale.value === 'dbm' ? mapY(p2Dbm) : mapY(Math.pow(10, p2Dbm / 10))
  const x1 = mapX(val1, minVal, maxVal)
  const x2 = mapX(val2, minVal, maxVal)
  
  const slope = (y2 - y1) / ((x2 - x1) || 1)
  const dx = 45 / Math.sqrt(1 + slope * slope)
  
  return {
    x1: coords.x - dx, y1: coords.y - (slope * dx),
    x2: coords.x + dx, y2: coords.y + (slope * dx)
  }
})

const yGridLines = computed(() => {
  if (chartScale.value === 'dbm') {
    return [-30, -50, -70, -90, -110].map(val => ({ y: mapY(val), label: `${val}` }))
  } else {
    const maxMw = Math.pow(10, -30 / 10)
    return [maxMw, maxMw*0.75, maxMw*0.5, maxMw*0.25, 0].map(val => ({ 
      y: mapY(val), 
      label: val === 0 ? '0' : `${(val * 1000000).toFixed(0)} nW` 
    }))
  }
})

const xGridLines = computed(() => {
  if (activeTab.value === 'distancia') {
    return [1, 10, 20, 30, 40, 50].map(val => ({ x: mapX(val, 1, 50), label: `${val}m` }))
  } else {
    return [0, 1, 2, 3, 4, 5].map(val => ({ x: mapX(val, 0, 5), label: `${val}` }))
  }
})
</script>

<template>
  <div class="min-h-screen bg-slate-950 text-slate-200 flex flex-col font-sans selection:bg-slate-700">
    <!-- DEFINICIONES SVG PARA TEXTURAS -->
    <svg class="absolute w-0 h-0 pointer-events-none">
      <defs>
        <pattern id="pat-concrete" width="10" height="10" patternUnits="userSpaceOnUse">
          <rect width="10" height="10" fill="#475569" />
          <circle cx="2" cy="2" r="1" fill="#334155" />
          <line x1="0" y1="10" x2="10" y2="0" stroke="#334155" stroke-width="0.5" />
        </pattern>
        <pattern id="pat-brick" width="20" height="10" patternUnits="userSpaceOnUse">
          <rect width="20" height="10" fill="#b91c1c" />
          <rect x="0" y="0" width="18" height="4" fill="#991b1b" />
          <rect x="10" y="5" width="18" height="4" fill="#991b1b" />
        </pattern>
        <pattern id="pat-drywall" width="10" height="10" patternUnits="userSpaceOnUse">
          <rect width="10" height="10" fill="#fef3c7" />
          <circle cx="3" cy="3" r="0.5" fill="#d97706" opacity="0.3" />
        </pattern>
        <pattern id="pat-glass" width="30" height="30" patternUnits="userSpaceOnUse">
          <rect width="30" height="30" fill="#0891b2" fill-opacity="0.25" />
          <line x1="0" y1="30" x2="30" y2="0" stroke="#22d3ee" stroke-opacity="0.4" />
        </pattern>
      </defs>
    </svg>

    <!-- HEADER -->
    <header class="border-b border-slate-900 bg-slate-950/80 backdrop-blur-md sticky top-0 z-50 p-4 text-center md:text-left">
      <h1 class="text-2xl font-extrabold text-slate-100">Simulador de Propagación Inalámbrica</h1>
      <p class="text-slate-400 text-sm">Laboratorio de Pérdidas de Señal - Matemáticas Exactas</p>
    </header>

    <main class="flex-grow max-w-7xl w-full mx-auto px-4 py-6 flex flex-col gap-6">
      
      <!-- CANVAS VISUAL (SVG) -->
      <section class="bg-slate-900/40 border border-slate-900 rounded-3xl p-5 md:p-6 backdrop-blur-xl relative shadow-xl">
        <h2 class="text-sm font-semibold uppercase tracking-wider text-slate-300 mb-4">Representación Física Interactiva</h2>
        <div class="relative bg-slate-950/80 border border-slate-900 rounded-2xl p-2 overflow-x-auto">
          <svg viewBox="0 0 600 200" width="100%" class="min-w-[550px] h-auto select-none">
            <!-- Cuadrícula -->
            <pattern id="canvas-grid" width="20" height="20" patternUnits="userSpaceOnUse">
              <path d="M 20 0 L 0 0 0 20" fill="none" stroke="rgba(51, 65, 85, 0.2)" />
            </pattern>
            <rect width="600" height="200" fill="url(#canvas-grid)" rx="8" />

            <line x1="60" y1="100" :x2="receiverX" y2="100" stroke="rgba(255,255,255,0.1)" stroke-dasharray="4,4" />

            <!-- Ondas -->
            <g>
              <path v-for="(wave, idx) in waveFronts" :key="idx" :d="wave.path" fill="none"
                    :class="[wave.color, !wave.isPastDevice ? 'animate-wave' : '']"
                    :stroke-opacity="wave.opacity" stroke-width="2.5"
                    :style="{ animationDuration: '2.5s', animationDelay: `${idx * 0.35}s` }" />
            </g>

            <!-- Muros -->
            <g v-if="activeTab === 'obstaculos'">
              <rect v-for="wall in physicalWalls" :key="wall.index" :x="wall.x" :y="wall.y" :width="wall.width" :height="wall.height"
                    rx="3" :fill="`url(#pat-${currentMaterial.pattern})`" stroke="rgba(0,0,0,0.3)" />
            </g>

            <!-- Router (Fijo a -40 dBm base) -->
            <g transform="translate(40, 80)">
              <rect x="5" y="24" width="30" height="12" rx="3" fill="#0f172a" stroke="#64748b" />
              <circle cx="12" cy="7" r="2" fill="#64748b" /><line x1="12" y1="24" x2="12" y2="8" stroke="#64748b" />
              <circle cx="28" cy="7" r="2" fill="#64748b" /><line x1="28" y1="24" x2="28" y2="8" stroke="#64748b" />
              <text x="20" y="48" fill="#94a3b8" font-size="8" font-family="monospace">P0 = {{ P0_base }} dBm</text>
            </g>

            <!-- Dispositivo -->
            <g :transform="`translate(${receiverX - 20}, 80)`" class="transition-transform duration-300">
              <rect x="7" y="10" width="26" height="17" rx="2" fill="#0f172a" stroke="#6366f1" />
              <path d="M 2,27 L 38,27 L 35,32 L 5,32 Z" fill="#312e81" stroke="#6366f1" />
              <text x="20" y="48" fill="#6366f1" font-size="8" font-family="monospace">Receptor</text>
            </g>
          </svg>
        </div>
      </section>

      <!-- CONTENEDOR PRINCIPAL -->
      <div class="grid grid-cols-1 lg:grid-cols-12 gap-6">
        
        <!-- CONTROLES -->
        <div class="lg:col-span-5 space-y-6">
          <div class="bg-slate-900/40 border border-slate-900 rounded-3xl p-6 backdrop-blur-xl">
            <div class="flex bg-slate-950 p-1 rounded-xl mb-6">
              <button @click="activeTab = 'distancia'" class="flex-1 py-2 text-sm font-semibold rounded-lg"
                :class="activeTab === 'distancia' ? 'bg-slate-800 text-white' : 'text-slate-400'">Distancia</button>
              <button @click="activeTab = 'obstaculos'" class="flex-1 py-2 text-sm font-semibold rounded-lg"
                :class="activeTab === 'obstaculos' ? 'bg-slate-800 text-white' : 'text-slate-400'">Obstáculos</button>
            </div>

            <template v-if="activeTab === 'distancia'">
              <div class="space-y-4">
                <div>
                  <label class="flex justify-between text-sm text-slate-300 mb-2">
                    Distancia (d): <span class="font-mono text-white">{{ distance }} m</span>
                  </label>
                  <input type="range" min="1" max="50" step="1" v-model.number="distance" class="w-full accent-slate-400" />
                </div>
                <div>
                  <label class="block text-sm text-slate-300 mb-2">Entorno (n):</label>
                  <select v-model.number="selectedEnvN" class="w-full bg-slate-950 border border-slate-800 p-3 rounded-xl text-white">
                    <option v-for="env in ENVIRONMENTS" :key="env.value" :value="env.value">{{ env.name }} (n={{ env.value }})</option>
                  </select>
                </div>
              </div>
            </template>
            <template v-else>
              <div class="space-y-4">
                <div>
                  <label class="flex justify-between text-sm text-slate-300 mb-2">
                    Cantidad obstáculos (x): <span class="font-mono text-white">{{ wallsCount }}</span>
                  </label>
                  <input type="range" min="0" max="5" step="1" v-model.number="wallsCount" class="w-full accent-slate-400" />
                </div>
                <div>
                  <label class="block text-sm text-slate-300 mb-2">Material (&alpha;):</label>
                  <select v-model="selectedMaterialKey" class="w-full bg-slate-950 border border-slate-800 p-3 rounded-xl text-white">
                    <option v-for="mat in MATERIALS" :key="mat.key" :value="mat.key">{{ mat.name }} (&alpha;={{ mat.alpha }})</option>
                  </select>
                </div>
              </div>
            </template>
          </div>

          <!-- FÓRMULA EN TIEMPO REAL -->
          <div class="bg-slate-900/40 border border-slate-900 rounded-3xl p-6 backdrop-blur-xl">
            <h3 class="text-xs uppercase tracking-widest text-slate-400 mb-4">Cálculo Matemático Reactivo</h3>
            <div class="bg-slate-950 p-4 rounded-xl font-mono text-sm border border-slate-800 shadow-inner">
              <template v-if="activeTab === 'distancia'">
                <div class="text-slate-400 mb-2">Pr(d) = Pr(d0) - 10 &middot; n &middot; log10(d / d0)</div>
                <div class="text-sky-300 mb-2">Pr({{ distance }}) = {{ P0_base }} - 10 &middot; {{ selectedEnvN }} &middot; log10({{ distance }} / 1)</div>
                <div class="text-emerald-400 font-bold border-t border-slate-800/50 pt-2">Pr({{ distance }}) = {{ rxPowerDistance.toFixed(2) }} dBm</div>
              </template>
              <template v-else>
                <!-- Paso 1: Potencia Lineal Base -->
                <div class="text-[11px] text-slate-500 mb-1 font-semibold uppercase tracking-wider">1. Linealización de Potencia Base (dBm a mW):</div>
                <div class="text-sky-300 mb-2 text-xs pl-2 border-l border-slate-800">
                  P<sub>0,mW</sub> = 10<sup>P<sub>0</sub>/10</sup> = 10<sup>-40/10</sup> = 0.0001 mW
                </div>
                <!-- Paso 2: Pérdida por Obstáculos (Base 10) -->
                <div class="text-[11px] text-slate-500 mb-1 font-semibold uppercase tracking-wider">2. Decaimiento Exponencial en Escala Lineal (Base 10):</div>
                <div class="text-sky-300 mb-2 text-xs pl-2 border-l border-slate-800">
                  P<sub>mW</sub>({{ wallsCount }}) = P<sub>0,mW</sub> &middot; 10<sup>-&alpha; &middot; x</sup><br>
                  P<sub>mW</sub>({{ wallsCount }}) = 0.0001 &middot; 10<sup>-{{ currentMaterial.alpha }} &middot; {{ wallsCount }}</sup> = {{ rxPowerObstaclesMw.toFixed(8) }} mW
                </div>
                <!-- Paso 3: Conversión a dBm -->
                <div class="text-[11px] text-slate-500 mb-1 font-semibold uppercase tracking-wider">3. Conversión Logarítmica (mW a dBm):</div>
                <div class="text-sky-300 mb-2 text-xs pl-2 border-l border-slate-800">
                  P<sub>dBm</sub>({{ wallsCount }}) = 10 &middot; log<sub>10</sub>(P<sub>mW</sub>({{ wallsCount }}))<br>
                  P<sub>dBm</sub>({{ wallsCount }}) = 10 &middot; log<sub>10</sub>({{ rxPowerObstaclesMw.toFixed(8) }})
                </div>
                <!-- Resultado Final -->
                <div class="text-emerald-400 font-bold border-t border-slate-800/50 pt-2 text-xs">
                  P<sub>dBm</sub>({{ wallsCount }}) = {{ rxPowerObstacles.toFixed(2) }} dBm
                </div>
              </template>
            </div>
          </div>
        </div>

        <!-- GRÁFICOS Y RESULTADOS -->
        <div class="lg:col-span-7 space-y-6">
          <div class="bg-slate-900/40 border border-slate-900 rounded-3xl p-6 backdrop-blur-xl grid grid-cols-1 md:grid-cols-2 gap-4">
            
            <div class="flex flex-col items-center justify-center p-6 rounded-2xl border" :class="[connectionStatus.colorBg, connectionStatus.colorBorder]">
              <div class="text-[10px] uppercase tracking-widest mb-2" :class="connectionStatus.colorText">Potencia Actual</div>
              <div class="text-4xl font-mono font-bold" :class="connectionStatus.colorText">{{ currentRxPowerDbm.toFixed(2) }} <span class="text-lg">dBm</span></div>
              <div class="mt-4 px-3 py-1 rounded-full text-xs font-bold uppercase bg-slate-950/40" :class="connectionStatus.colorText">{{ connectionStatus.label }}</div>
            </div>

            <div class="bg-slate-950 p-6 rounded-2xl border border-slate-800 flex flex-col justify-center">
              <div class="text-[10px] uppercase tracking-widest text-slate-500 mb-2">Reglas de Estado Implementadas</div>
              <ul class="text-xs space-y-2 text-slate-300 font-mono">
                <li class="flex items-center gap-2"><span class="w-2 h-2 rounded-full bg-emerald-400"></span> Excelente &ge; -65 dBm</li>
                <li class="flex items-center gap-2"><span class="w-2 h-2 rounded-full bg-amber-400"></span> Inestable entre -66 y -79 dBm</li>
                <li class="flex items-center gap-2"><span class="w-2 h-2 rounded-full bg-rose-500"></span> Zona Muerta &lt; -80 dBm</li>
              </ul>
            </div>
          </div>

          <!-- GRÁFICO SVG -->
          <div class="bg-slate-900/40 border border-slate-900 rounded-3xl p-6 backdrop-blur-xl">
            <div class="flex justify-between items-center mb-4">
              <h3 class="text-sm font-bold text-slate-300">Curva de Atenuación Teórica</h3>
              <div class="flex gap-1 bg-slate-950 p-1 rounded-lg">
                <button @click="chartScale = 'dbm'" class="px-2 py-1 text-[10px] rounded" :class="chartScale === 'dbm' ? 'bg-slate-800 text-white' : 'text-slate-500'">dBm</button>
                <button @click="chartScale = 'mw'" class="px-2 py-1 text-[10px] rounded" :class="chartScale === 'mw' ? 'bg-slate-800 text-white' : 'text-slate-500'">mW</button>
              </div>
            </div>

            <div class="bg-slate-950 rounded-xl border border-slate-800 p-2">
              <svg viewBox="0 0 500 240" class="w-full h-auto select-none font-mono">
                <g v-for="line in yGridLines" :key="line.y">
                  <line x1="50" :y1="line.y" x2="480" :y2="line.y" stroke="#1e293b" stroke-dasharray="2,2" />
                  <text x="45" :y="line.y + 3" fill="#64748b" font-size="8" text-anchor="end">{{ line.label }}</text>
                </g>
                <g v-for="line in xGridLines" :key="line.x">
                  <line :x1="line.x" y1="15" :x2="line.x" y2="205" stroke="#1e293b" stroke-dasharray="2,2" />
                  <text :x="line.x" y="220" fill="#64748b" font-size="8" text-anchor="middle">{{ line.label }}</text>
                </g>
                <path :d="chartPath" fill="none" class="stroke-slate-300" stroke-width="2" />
                <line :x1="tangentLine.x1" :y1="tangentLine.y1" :x2="tangentLine.x2" :y2="tangentLine.y2" stroke="#f43f5e" stroke-width="2" class="animate-pulse" />
                <circle :cx="activeDotCoords.x" :cy="activeDotCoords.y" r="5" class="fill-white" />
              </svg>
            </div>
            
            <div class="mt-4 p-4 bg-slate-950 rounded-xl border border-slate-800 text-xs text-slate-400 font-mono">
              La recta roja muestra la derivada en el punto de operación:<br>
              <span class="text-white">
                {{ activeTab === 'distancia' ? derivativeDistanceDbm.toFixed(4) + ' dBm/m' : derivativeObstaclesDbm.toFixed(4) + ' dBm/muro' }}
              </span>
            </div>
          </div>
        </div>
      </div>
      
      <!-- GLOSARIO (RESTAURADO) -->
      <section class="bg-slate-900/40 border border-slate-900 rounded-3xl p-6 backdrop-blur-xl shadow-xl">
        <h2 class="text-lg font-bold text-slate-200 mb-5">Glosario de Conceptos Integrados</h2>
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 text-sm text-slate-400">
          <div class="space-y-2">
            <h3 class="text-slate-100 font-bold">1. Fórmula Logarítmica de Espacio Libre (Cisco)</h3>
            <p>Utilizada en el estándar industrial para estimar la atenuación geométrica pura. El factor `n` representa los rebotes y dispersión. En nuestro caso, la potencia base se asume fijada matemáticamente en `Pr(d0) = -40 dBm` a 1 metro de distancia.</p>
          </div>
          <div class="space-y-2">
            <h3 class="text-slate-100 font-bold">2. Decaimiento Exponencial de Euler</h3>
            <p>Representa la absorción del medio sólido. Un muro reduce físicamente la energía a una fracción `e^(-alpha)`. Aplicado sobre una señal base ya débil (-40 dBm), observamos una atenuación escalonada severa visualizada correctamente mediante la escala lineal/logarítmica conjugada.</p>
          </div>
        </div>
      </section>

    </main>
  </div>
</template>

<style>
@keyframes wave-propagate {
  0% { stroke-dashoffset: 0; opacity: 0.15; }
  50% { opacity: 0.9; }
  100% { stroke-dashoffset: -30; opacity: 0.1; }
}
.animate-wave {
  stroke-dasharray: 6, 8;
  animation: wave-propagate 3s infinite linear;
}
</style>
