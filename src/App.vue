<template>
  <div class="container">
    <h1>🔬 Controle da Keithley 2611B</h1>
    <!-- Indicador de modo (agora automático) -->
    <div class="mode-indicator" :class="{'mock': isMockMode, 'real': !isMockMode}">
      <span v-if="isMockMode">🎭 MODO SIMULAÇÃO ATIVO</span>
      <span v-else>🔌 MODO REAL - Keithley Conectada</span>
    </div>
    
    <!-- Status da conexão -->
    <div class="status-bar">
      <div class="status-indicator" :class="{'connected': statusConectado, 'disconnected': !statusConectado}">
        {{ statusConectado ? '✅ Conectado' : '❌ Desconectado' }}
      </div>
      <button @click="testarConexao" class="btn-small" :disabled="conectando">
        {{ conectando ? '⏳ Testando...' : 'Testar Conexão' }}
      </button>
    </div>

    <!-- Configurações do Mock (visível apenas em modo mock) -->
    <div v-if="isMockMode" class="card mock-config">
      <h3>🎮 Configurações da Simulação</h3>
      
      <div class="form-group">
        <label>Modo de Simulação:</label>
        <select v-model="mockConfig.modo" @change="atualizarMockConfig">
          <option value="realista">Realista (comportamento de componentes)</option>
          <option value="random">Aleatório</option>
          <option value="sine">Onda Senoidal</option>
          <option value="linear">Linear</option>
          <option value="constante">Constante</option>
          <option value="exponencial">Exponencial</option>
        </select>
      </div>
      
      <div v-if="mockConfig.modo === 'sine'" class="form-group">
        <label>Amplitude:</label>
        <input type="number" v-model.number="mockConfig.amplitude" step="0.1" @change="atualizarMockConfig">
      </div>
      
      <div v-if="mockConfig.modo === 'sine'" class="form-group">
        <label>Frequência (Hz):</label>
        <input type="number" v-model.number="mockConfig.frequencia" step="0.1" @change="atualizarMockConfig">
      </div>
      
      <div v-if="mockConfig.modo === 'random'" class="form-group">
        <label>Valor Mínimo:</label>
        <input type="number" v-model.number="mockConfig.min" step="0.1" @change="atualizarMockConfig">
      </div>
      
      <div v-if="mockConfig.modo === 'random'" class="form-group">
        <label>Valor Máximo:</label>
        <input type="number" v-model.number="mockConfig.max" step="0.1" @change="atualizarMockConfig">
      </div>
      
      <div class="form-group">
        <label>Nível de Ruído (%):</label>
        <input type="range" v-model.number="mockConfig.ruido" min="0" max="0.2" step="0.001">
        <span>{{ (mockConfig.ruido * 100).toFixed(1) }}%</span>
      </div>
      
      <div class="form-group">
        <label>Delay Artificial (s):</label>
        <input type="number" v-model.number="mockConfig.delay_artificial" min="0" max="2" step="0.1">
        <small>Simula tempo de medição real</small>
      </div>
      
      <button @click="resetMockCounter" class="btn-small">Resetar Contador</button>
    </div>

    <!-- Abas para alternar entre modos de medição -->
    <div class="tabs">
      <button 
        :class="{'tab-active': modoOperacao === 'simples'}" 
        @click="modoOperacao = 'simples'">
        📊 Medição Simples
      </button>
      <button 
        :class="{'tab-active': modoOperacao === 'iv'}" 
        @click="modoOperacao = 'iv'">
        ⚡ Medição I/V Simultânea
      </button>
      <button 
        v-if="resultadosIV.length > 0"
        :class="{'tab-active': modoOperacao === 'grafico'}" 
        @click="mostrarGrafico">
        📈 Gráfico I x V
      </button>
    </div>

    <!-- Modo Simples -->
    <div v-if="modoOperacao === 'simples'" class="card">
      <h2>⚙️ Parâmetros da Medição</h2>
      
      <div class="form-group">
        <label>Modo de Medição:</label>
        <select v-model="parametros.modo_medicao">
          <option value="tensao">Tensão (V)</option>
          <option value="corrente">Corrente (A)</option>
        </select>
      </div>

      <div class="form-group">
        <label>Número de Leituras:</label>
        <input type="number" v-model.number="parametros.num_leituras" min="1" max="100">
      </div>

      <div class="form-group">
        <label>Delay entre leituras (s):</label>
        <input type="number" v-model.number="parametros.delay_entre_leituras" min="0.1" max="10" step="0.1">
      </div>

      <div class="form-group">
        <label>Tensão aplicada (V, opcional):</label>
        <input type="number" v-model.number="parametros.nivel_tensao_aplicada" step="0.1">
      </div>

      <button @click="iniciarMedicao" :disabled="medindo" class="btn-primary">
        {{ medindo ? '⏳ Medindo...' : '▶️ Iniciar Medição' }}
      </button>
    </div>

    <!-- Modo I/V Simultâneo -->
    <div v-if="modoOperacao === 'iv'" class="card">
      <h2>⚡ Medição Simultânea (Corrente e Tensão)</h2>
      
      <div class="form-group">
        <label>Modo de Operação:</label>
        <select v-model="parametrosIV.modo">
          <option value="tensao_fixa">Aplicar Tensão Fixa, Medir Corrente</option>
          <option value="corrente_fixa">Aplicar Corrente Fixa, Medir Tensão</option>
        </select>
      </div>

      <div class="form-group">
        <label>{{ parametrosIV.modo === 'tensao_fixa' ? 'Tensão a Aplicar (V):' : 'Corrente a Aplicar (A):' }}</label>
        <input 
          type="number" 
          v-model.number="parametrosIV.valor_fixo" 
          :step="parametrosIV.modo === 'tensao_fixa' ? 0.1 : 0.0001"
        >
        <small v-if="parametrosIV.modo === 'tensao_fixa'" class="hint">Valor típico: 1V a 200V</small>
        <small v-else class="hint">Valor típico: 0.001A a 1.5A</small>
      </div>

      <div class="form-group">
        <label>Número de Leituras:</label>
        <input type="number" v-model.number="parametrosIV.num_leituras" min="1" max="100">
      </div>

      <div class="form-group">
        <label>Delay entre leituras (s):</label>
        <input type="number" v-model.number="parametrosIV.delay_entre_leituras" min="0.1" max="10" step="0.1">
      </div>

      <button @click="iniciarMedicaoIV" :disabled="medindoIV" class="btn-primary">
        {{ medindoIV ? '⏳ Medindo...' : '▶️ Iniciar Medição I/V' }}
      </button>
    </div>

    <!-- Modo Gráfico I x V -->
<div v-if="modoOperacao === 'grafico' && resultadosIV.length > 0" class="card" key="grafico-card">
  <h2>📈 Curva Característica I x V</h2>
  
  <div class="chart-controls">
    <div class="chart-type-selector">
      <button @click="atualizarTipoGrafico('iv')" :class="{'active': tipoGrafico === 'iv'}" class="btn-small">
        📊 I x V (Corrente vs Tensão)
      </button>
      <button @click="atualizarTipoGrafico('vi')" :class="{'active': tipoGrafico === 'vi'}" class="btn-small">
        📈 V x I (Tensão vs Corrente)
      </button>
      <button @click="atualizarTipoGrafico('pv')" :class="{'active': tipoGrafico === 'pv'}" class="btn-small">
        ⚡ P x V (Potência vs Tensão)
      </button>
    </div>
    
    <div class="chart-view-options">
      <label>
        <input type="checkbox" v-model="mostrarPontos" @change="atualizarPontos"> Mostrar Pontos
      </label>
      <label v-if="tipoGrafico === 'iv'">
        <input type="checkbox" v-model="escalaLog" @change="atualizarEscalaLog"> Escala Logarítmica (Corrente)
      </label>
    </div>
  </div>
  
  <!-- Container fixo para o canvas -->
  <div class="chart-container" ref="chartContainer">
    <canvas ref="chartCanvas"></canvas>
  </div>
  
  <div class="chart-stats">
    <div class="stat-item">
      <span class="stat-label">Ponto de máxima potência:</span>
      <span class="stat-value">{{ pontoMaxPotencia.vMPP.toFixed(3) }} V @ {{ pontoMaxPotencia.iMPP.toExponential(3) }} A ({{ pontoMaxPotencia.pMPP.toExponential(3) }} W)</span>
    </div>
    <div class="stat-item">
      <span class="stat-label">Resistência estimada (região linear):</span>
      <span class="stat-value">{{ resistenciaEstimada.toExponential(3) }} Ω</span>
    </div>
  </div>
  
  <div class="chart-actions">
    <button @click="exportarGrafico" class="btn-secondary">📸 Exportar Gráfico (PNG)</button>
    <button @click="voltarParaTabela" class="btn-secondary">📊 Voltar para Tabela</button>
  </div>
</div>

    <!-- Resultados Modo Simples -->
    <div class="card" v-if="modoOperacao === 'simples' && resultados.length > 0">
      <h2>📊 Resultados</h2>
      
      <div class="results-summary">
        <p><strong>Status:</strong> {{ statusMensagem }}</p>
        <p><strong>Média:</strong> {{ mediaResultados.toFixed(6) }} {{ unidade }}</p>
        <p><strong>Mínimo:</strong> {{ minimoResultados.toFixed(6) }} {{ unidade }}</p>
        <p><strong>Máximo:</strong> {{ maximoResultados.toFixed(6) }} {{ unidade }}</p>
        <p v-if="isMockMode" class="mock-note"><strong>🎭 Modo simulação:</strong> {{ mockConfig.modo }}</p>
      </div>

      <table class="results-table">
        <thead>
          <tr><th>Medição #</th><th>Valor ({{ unidade }})</th></tr>
        </thead>
        <tbody>
          <tr v-for="(valor, index) in resultados" :key="index">
            <td>{{ index + 1 }}</td>
            <td>{{ valor.toFixed(6) }}</td>
          </tr>
        </tbody>
      </table>

      <button @click="exportarCSV" class="btn-secondary">📥 Exportar CSV</button>
    </div>

    <!-- Resultados Modo IV -->
    <div class="card" v-if="modoOperacao === 'iv' && resultadosIV.length > 0">
      <h2>📊 Resultados I/V</h2>
      
      <div class="results-summary">
        <p><strong>Status:</strong> {{ statusMensagemIV }}</p>
        <p><strong>Corrente Média:</strong> {{ mediaCorrente.toFixed(6) }} A</p>
        <p><strong>Tensão Média:</strong> {{ mediaTensao.toFixed(6) }} V</p>
        <p><strong>Potência Média:</strong> {{ mediaPotencia.toFixed(6) }} W</p>
        <p v-if="isMockMode" class="mock-note"><strong>🎭 Modo simulação:</strong> {{ mockConfig.modo }}</p>
      </div>

      <table class="results-table">
        <thead>
          <tr><th>#</th><th>Corrente (A)</th><th>Tensão (V)</th><th>Potência (W)</th></tr>
        </thead>
        <tbody>
          <tr v-for="(med, index) in resultadosIV" :key="index">
            <td>{{ index + 1 }}</td>
            <td>{{ med.corrente.toFixed(6) }}</td>
            <td>{{ med.tensao.toFixed(6) }}</td>
            <td>{{ med.potencia.toFixed(6) }}</td>
          </tr>
        </tbody>
      </table>

      <div class="action-buttons">
        <button @click="mostrarGrafico" class="btn-primary">📈 Ver Gráfico I x V</button>
        <button @click="exportarCSV_IV" class="btn-secondary">📥 Exportar CSV (I/V)</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue'
import { Chart, registerables } from 'chart.js'

// Registra todos os componentes do Chart.js
Chart.register(...registerables)

// ============================================
// DETECÇÃO AUTOMÁTICA DO MODO (MOCK ou REAL)
// ============================================

const detectMode = async () => {
  const currentUrl = window.location.origin
  const realBaseUrl = 'http://127.0.0.1:8001/api'
  const mockBaseUrl = 'http://127.0.0.1:8000/api'

  const probe = async (url) => {
    try {
      const response = await fetch(url, { method: 'GET' })
      return response.ok
    } catch {
      return false
    }
  }

  if (currentUrl.includes(':8000')) {
    return { isMock: true, baseUrl: currentUrl + '/api' }
  }

  if (currentUrl.includes(':8001')) {
    return { isMock: false, baseUrl: currentUrl + '/api' }
  }

  if (await probe(`${realBaseUrl}/health`)) {
    return { isMock: false, baseUrl: realBaseUrl }
  }

  if (await probe(`${mockBaseUrl}/health`)) {
    return { isMock: true, baseUrl: mockBaseUrl }
  }

  return { isMock: true, baseUrl: mockBaseUrl }
}

const isMockMode = ref(true)
const API_BASE_URL = ref('http://127.0.0.1:8000/api')
const conectando = ref(false)
const statusConectado = ref(false)

// Modo de operação
const modoOperacao = ref('simples')
const tipoGrafico = ref('iv')
const mostrarPontos = ref(true)
const escalaLog = ref(false)
const chartInstance = ref(null)
const chartCanvas = ref(null)

// Parâmetros
const parametros = ref({
  num_leituras: 5,
  delay_entre_leituras: 0.5,
  modo_medicao: 'tensao',
  nivel_tensao_aplicada: 0.0
})

const parametrosIV = ref({
  num_leituras: 5,
  delay_entre_leituras: 0.5,
  modo: 'tensao_fixa',
  valor_fixo: 1.0
})

// Resultados
const resultados = ref([])
const medindo = ref(false)
const statusMensagem = ref('')

const resultadosIV = ref([])
const medindoIV = ref(false)
const statusMensagemIV = ref('')

// Mock config
const mockConfig = ref({
  modo: 'realista',
  ruido: 0.001,
  delay_artificial: 0.1,
  amplitude: 1.0,
  frequencia: 0.5,
  min: 0,
  max: 10
})

// ============================================
// PROPRIEDADES COMPUTADAS
// ============================================

const pontoMaxPotencia = computed(() => {
  if (resultadosIV.value.length === 0) {
    return { vMPP: 0, iMPP: 0, pMPP: 0 }
  }
  
  let maxP = 0
  let vAtMaxP = 0
  let iAtMaxP = 0
  
  for (const med of resultadosIV.value) {
    if (med.potencia > maxP) {
      maxP = med.potencia
      vAtMaxP = med.tensao
      iAtMaxP = med.corrente
    }
  }
  
  return { vMPP: vAtMaxP, iMPP: iAtMaxP, pMPP: maxP }
})

const resistenciaEstimada = computed(() => {
  if (resultadosIV.value.length < 2) return 0
  
  const pontos = resultadosIV.value.filter(m => m.tensao > 0.1 && m.corrente > 1e-6)
  if (pontos.length < 2) return 0
  
  const n = pontos.length
  const sumV = pontos.reduce((s, p) => s + p.tensao, 0)
  const sumI = pontos.reduce((s, p) => s + p.corrente, 0)
  const sumVI = pontos.reduce((s, p) => s + p.tensao * p.corrente, 0)
  const sumV2 = pontos.reduce((s, p) => s + p.tensao * p.tensao, 0)
  
  const denominator = n * sumV2 - sumV * sumV
  if (denominator === 0) return 0
  
  const invR = (n * sumVI - sumV * sumI) / denominator
  return 1 / invR
})

const unidade = computed(() => {
  return parametros.value.modo_medicao === 'tensao' ? 'V' : 'A'
})

const mediaResultados = computed(() => {
  if (resultados.value.length === 0) return 0
  const soma = resultados.value.reduce((a, b) => a + b, 0)
  return soma / resultados.value.length
})

const minimoResultados = computed(() => {
  if (resultados.value.length === 0) return 0
  return Math.min(...resultados.value)
})

const maximoResultados = computed(() => {
  if (resultados.value.length === 0) return 0
  return Math.max(...resultados.value)
})

const mediaCorrente = computed(() => {
  if (resultadosIV.value.length === 0) return 0
  const soma = resultadosIV.value.reduce((a, b) => a + b.corrente, 0)
  return soma / resultadosIV.value.length
})

const mediaTensao = computed(() => {
  if (resultadosIV.value.length === 0) return 0
  const soma = resultadosIV.value.reduce((a, b) => a + b.tensao, 0)
  return soma / resultadosIV.value.length
})

const mediaPotencia = computed(() => {
  if (resultadosIV.value.length === 0) return 0
  const soma = resultadosIV.value.reduce((a, b) => a + b.potencia, 0)
  return soma / resultadosIV.value.length
})

// ============================================
// FUNÇÕES DO GRÁFICO
// ============================================

const chartContainer = ref(null)
let isCreatingChart = false

const criarGrafico = () => {
  // Previne criação múltipla simultânea
  if (isCreatingChart) return
  if (!chartCanvas.value || resultadosIV.value.length === 0) return
  
  isCreatingChart = true
  
  // Destroi instância anterior se existir
  if (chartInstance.value) {
    chartInstance.value.destroy()
    chartInstance.value = null
  }
  
  // Força o container a ter tamanho fixo antes de criar o canvas
  if (chartContainer.value) {
    chartContainer.value.style.width = '100%'
    chartContainer.value.style.height = '450px'
  }
  
  // Prepara os dados
  let dadosX = []
  let dadosY = []
  let labelX = ''
  let labelY = ''
  let titulo = ''
  let cor = ''
  
  if (tipoGrafico.value === 'iv') {
    dadosX = resultadosIV.value.map(m => m.tensao)
    dadosY = resultadosIV.value.map(m => escalaLog.value ? Math.max(Math.abs(m.corrente), 1e-12) : m.corrente)
    labelX = 'Tensão (V)'
    labelY = escalaLog.value ? 'Corrente (A) - Escala Log' : 'Corrente (A)'
    titulo = 'Curva I x V'
    cor = 'rgba(54, 162, 235, 0.7)'
  } else if (tipoGrafico.value === 'vi') {
    dadosX = resultadosIV.value.map(m => m.corrente)
    dadosY = resultadosIV.value.map(m => m.tensao)
    labelX = 'Corrente (A)'
    labelY = 'Tensão (V)'
    titulo = 'Curva V x I'
    cor = 'rgba(255, 99, 132, 0.7)'
  } else {
    dadosX = resultadosIV.value.map(m => m.tensao)
    dadosY = resultadosIV.value.map(m => m.potencia)
    labelX = 'Tensão (V)'
    labelY = 'Potência (W)'
    titulo = 'Potência vs Tensão'
    cor = 'rgba(75, 192, 192, 0.7)'
  }
  
  // Configuração da escala Y
  let escalaY = {}
  if (tipoGrafico.value === 'iv' && escalaLog.value) {
    escalaY = {
      type: 'logarithmic',
      title: { display: true, text: labelY, font: { size: 14, weight: 'bold' } },
      ticks: { 
        callback: (value) => value.toExponential(2),
        stepSize: 1
      },
      grid: { color: 'rgba(0,0,0,0.1)' }
    }
  } else {
    escalaY = {
      type: 'linear',
      title: { display: true, text: labelY, font: { size: 14, weight: 'bold' } },
      grid: { color: 'rgba(0,0,0,0.1)' }
    }
  }
  
  // Cria o gráfico - SEM setTimeout e SEM resize manual
  const canvas = chartCanvas.value
  const ctx = canvas.getContext('2d')
  
  chartInstance.value = new Chart(ctx, {
    type: 'scatter',
    data: {
      datasets: [{
        label: titulo,
        data: dadosX.map((x, i) => ({ x: x, y: dadosY[i] })),
        borderColor: cor,
        backgroundColor: cor.replace('0.7', '0.1'),
        borderWidth: 2,
        pointRadius: mostrarPontos.value ? 5 : 0,
        pointHoverRadius: 7,
        pointBackgroundColor: cor,
        pointBorderColor: '#fff',
        showLine: true,
        tension: 0.1,
        fill: false,
        order: 1
      }]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      plugins: {
        legend: { 
          position: 'top',
          labels: { font: { size: 12 } }
        },
        tooltip: {
          mode: 'index',
          intersect: false,
          callbacks: {
            label: function(context) {
              const point = context.raw
              if (tipoGrafico.value === 'iv') {
                return [`Tensão: ${point.x.toFixed(4)} V`, `Corrente: ${point.y.toExponential(6)} A`]
              } else if (tipoGrafico.value === 'vi') {
                return [`Corrente: ${point.x.toExponential(6)} A`, `Tensão: ${point.y.toFixed(4)} V`]
              } else {
                return [`Tensão: ${point.x.toFixed(4)} V`, `Potência: ${point.y.toExponential(6)} W`]
              }
            }
          }
        }
      },
      scales: {
        x: {
          title: { display: true, text: labelX, font: { size: 14, weight: 'bold' } },
          ticks: { 
            callback: (value) => value.toFixed(3),
            autoSkip: true,
            maxTicksLimit: 10
          },
          grid: { color: 'rgba(0,0,0,0.1)' }
        },
        y: escalaY
      },
      elements: {
        line: {
          borderJoin: 'round'
        }
      },
      layout: {
        padding: {
          top: 10,
          bottom: 10,
          left: 10,
          right: 10
        }
      }
    }
  })
  
  isCreatingChart = false
}

const atualizarTipoGrafico = (tipo) => {
  if (tipoGrafico.value === tipo) return // Evita recriação desnecessária
  tipoGrafico.value = tipo
  nextTick(() => {
    criarGrafico()
  })
}

const atualizarPontos = () => {
  if (chartInstance.value) {
    chartInstance.value.data.datasets[0].pointRadius = mostrarPontos.value ? 5 : 0
    chartInstance.value.update('none') // Use 'none' para evitar animação desnecessária
  }
}

const atualizarEscalaLog = () => {
  // Recria o gráfico com nova escala
  nextTick(() => {
    criarGrafico()
  })
}

const mostrarGrafico = () => {
  modoOperacao.value = 'grafico'
  // Aguarda o DOM ser completamente renderizado
  nextTick(() => {
    // Pequeno delay para garantir que o container está pronto
    setTimeout(() => {
      criarGrafico()
    }, 50)
  })
}

const voltarParaTabela = () => {
  modoOperacao.value = 'iv'
}

const exportarGrafico = () => {
  if (!chartCanvas.value) return
  
  const link = document.createElement('a')
  const tipoTexto = tipoGrafico.value === 'iv' ? 'IV' : (tipoGrafico.value === 'vi' ? 'VI' : 'PV')
  link.download = `grafico_${tipoTexto}_${new Date().toISOString().slice(0, 19)}.png`
  link.href = chartCanvas.value.toDataURL()
  link.click()
}

// ============================================
// FUNÇÕES AUXILIARES
// ============================================

const apiRequest = async (url, options = {}) => {
  const response = await fetch(url, {
    headers: { 'Content-Type': 'application/json', ...options.headers },
    ...options
  })
  
  if (!response.ok) {
    const error = await response.json().catch(() => ({ mensagem: 'Erro na requisição' }))
    throw new Error(error.mensagem || `HTTP ${response.status}`)
  }
  
  return response.json()
}

// ============================================
// MÉTODOS - CONEXÃO
// ============================================
const testarConexao = async () => {
  conectando.value = true
  try {
    const data = await apiRequest(`${API_BASE_URL.value}/testar_conexao`)
    if (data.status === 'conectado') {
      statusConectado.value = true
      console.log('Conectado a:', data.instrumento)
    }
  } catch (error) {
    statusConectado.value = false
    console.error('Falha na conexão:', error.message)
  } finally {
    conectando.value = false
  }
}

// ============================================
// MÉTODOS - MOCK CONFIG
// ============================================
const atualizarMockConfig = async () => {
  if (!isMockMode.value) return
  
  try {
    const payload = {
      modo: mockConfig.value.modo,
      ruido: mockConfig.value.ruido,
      delay_artificial: mockConfig.value.delay_artificial
    }
    
    if (mockConfig.value.modo === 'sine') {
      payload.amplitude = mockConfig.value.amplitude
      payload.frequencia = mockConfig.value.frequencia
    }
    
    if (mockConfig.value.modo === 'random') {
      payload.min = mockConfig.value.min
      payload.max = mockConfig.value.max
    }
    
    await apiRequest(`${API_BASE_URL.value}/mock/config`, {
      method: 'POST',
      body: JSON.stringify(payload)
    })
    console.log('Mock configurado:', mockConfig.value.modo)
  } catch (error) {
    console.error('Erro ao configurar mock:', error)
  }
}

const resetMockCounter = async () => {
  if (!isMockMode.value) return
  try {
    await apiRequest(`${API_BASE_URL.value}/mock/reset`, { method: 'POST' })
    console.log('Contador do mock resetado')
  } catch (error) {
    console.error('Erro ao resetar:', error)
  }
}

// ============================================
// MÉTODOS - MEDIÇÃO SIMPLES
// ============================================
const iniciarMedicao = async () => {
  medindo.value = true
  resultados.value = []
  statusMensagem.value = ''

  try {
    const data = await apiRequest(`${API_BASE_URL.value}/medir`, {
      method: 'POST',
      body: JSON.stringify(parametros.value)
    })
    
    if (data.status === 'sucesso') {
      resultados.value = data.medicoes
      statusMensagem.value = data.mensagem || 'Medição concluída com sucesso!'
    }
  } catch (error) {
    statusMensagem.value = error.message || 'Erro na comunicação com o backend'
    alert(`Erro: ${statusMensagem.value}`)
  } finally {
    medindo.value = false
  }
}

const exportarCSV = () => {
  let csvContent = `Medição,Valor (${unidade.value})\n`
  resultados.value.forEach((valor, index) => {
    csvContent += `${index + 1},${valor.toFixed(6)}\n`
  })
  
  csvContent += `\nMédia,${mediaResultados.value.toFixed(6)}\n`
  csvContent += `Mínimo,${minimoResultados.value.toFixed(6)}\n`
  csvContent += `Máximo,${maximoResultados.value.toFixed(6)}\n`
  
  if (isMockMode.value) {
    csvContent += `\nModo de simulação,${mockConfig.value.modo}\n`
  }
  
  const blob = new Blob([csvContent], { type: 'text/csv' })
  const url = window.URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `medicao_keithley_${new Date().toISOString().slice(0, 19)}.csv`
  a.click()
  window.URL.revokeObjectURL(url)
}

// ============================================
// MÉTODOS - MEDIÇÃO I/V
// ============================================
const iniciarMedicaoIV = async () => {
  medindoIV.value = true
  resultadosIV.value = []
  statusMensagemIV.value = ''

  try {
    const data = await apiRequest(`${API_BASE_URL.value}/medir_iv`, {
      method: 'POST',
      body: JSON.stringify(parametrosIV.value)
    })
    
    if (data.status === 'sucesso') {
      resultadosIV.value = data.medicoes
      statusMensagemIV.value = data.mensagem || 'Medição IV concluída com sucesso!'
    }
  } catch (error) {
    statusMensagemIV.value = error.message || 'Erro na medição IV'
    alert(`Erro: ${statusMensagemIV.value}`)
  } finally {
    medindoIV.value = false
  }
}

const exportarCSV_IV = () => {
  let csvContent = 'Medição,Corrente (A),Tensão (V),Potência (W)\n'
  resultadosIV.value.forEach((med, index) => {
    csvContent += `${index + 1},${med.corrente.toFixed(6)},${med.tensao.toFixed(6)},${med.potencia.toFixed(6)}\n`
  })
  
  csvContent += `\nMédias,,,\n,${mediaCorrente.value.toFixed(6)},${mediaTensao.value.toFixed(6)},${mediaPotencia.value.toFixed(6)}\n`
  csvContent += `\nPonto de máxima potência,,,\n,${pontoMaxPotencia.value.iMPP.toExponential(3)},${pontoMaxPotencia.value.vMPP.toFixed(3)},${pontoMaxPotencia.value.pMPP.toExponential(3)}\n`
  csvContent += `\nResistência estimada,,,\n,${resistenciaEstimada.value.toExponential(3)},,Ω\n`
  
  if (isMockMode.value) {
    csvContent += `\nModo de simulação,${mockConfig.value.modo}\n`
  }
  
  const blob = new Blob([csvContent], { type: 'text/csv' })
  const url = window.URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `medicao_IV_${new Date().toISOString().slice(0, 19)}.csv`
  a.click()
  window.URL.revokeObjectURL(url)
}

// ============================================
// LIFECYCLE
// ============================================
const initMode = async () => {
  const detected = await detectMode()
  isMockMode.value = detected.isMock
  API_BASE_URL.value = detected.baseUrl
}

onMounted(async () => {
  await initMode()
  console.log(`🖥️ UI iniciada no modo: ${isMockMode.value ? 'MOCK' : 'REAL'}`)
  console.log(`📍 API URL: ${API_BASE_URL.value}`)
  await testarConexao()
})
</script>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

h1 {
  color: #2c3e50;
  border-bottom: 3px solid #42b983;
  padding-bottom: 10px;
}

.mode-indicator {
  text-align: center;
  padding: 10px;
  border-radius: 8px;
  font-weight: bold;
  margin: 15px 0;
}

.mode-indicator.mock {
  background: #fff3cd;
  color: #856404;
  border: 1px solid #ffeeba;
}

.mode-indicator.real {
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.status-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #f8f9fa;
  padding: 10px 15px;
  border-radius: 8px;
  margin: 20px 0;
}

.status-indicator {
  font-weight: bold;
  padding: 5px 15px;
  border-radius: 20px;
}

.status-indicator.connected {
  background: #d4edda;
  color: #155724;
}

.status-indicator.disconnected {
  background: #f8d7da;
  color: #721c24;
}

.tabs {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.tabs button {
  padding: 10px 20px;
  background: #e0e0e0;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.3s;
}

.tabs button:hover {
  background: #d0d0d0;
}

.tabs button.tab-active {
  background: #42b983;
  color: white;
}

.card {
  background: white;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.mock-config {
  border-left: 4px solid #ffc107;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
  color: #555;
}

.form-group input, .form-group select {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

.form-group input[type="range"] {
  width: calc(100% - 60px);
  display: inline-block;
  margin-right: 10px;
}

.form-group span {
  display: inline-block;
  width: 50px;
  text-align: center;
}

.hint {
  font-size: 12px;
  color: #666;
  display: block;
  margin-top: 5px;
}

.btn-primary {
  background: #42b983;
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  width: 100%;
}

.btn-primary:hover:not(:disabled) {
  background: #359268;
}

.btn-primary:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.btn-secondary {
  background: #6c757d;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  margin-top: 15px;
}

.btn-small {
  background: #007bff;
  color: white;
  border: none;
  padding: 5px 10px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
}

.btn-small:hover:not(:disabled) {
  background: #0056b3;
}

.results-summary {
  background: #e8f4f8;
  padding: 15px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.results-summary p {
  margin: 5px 0;
}

.mock-note {
  color: #856404;
  font-size: 12px;
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px solid #ddd;
}

.results-table {
  width: 100%;
  border-collapse: collapse;
}

.results-table th, .results-table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: center;
}

.results-table th {
  background: #f2f2f2;
  font-weight: bold;
}

.action-buttons {
  display: flex;
  gap: 10px;
  margin-top: 15px;
}

.action-buttons .btn-primary,
.action-buttons .btn-secondary {
  flex: 1;
  margin-top: 0;
}

.chart-controls {
  margin-bottom: 20px;
}

.chart-type-selector {
  display: flex;
  gap: 10px;
  margin-bottom: 15px;
  flex-wrap: wrap;
}

.chart-type-selector .btn-small {
  background: #e0e0e0;
  color: #333;
}

.chart-type-selector .btn-small.active {
  background: #42b983;
  color: white;
}

.chart-view-options {
  display: flex;
  gap: 20px;
  margin-bottom: 15px;
  padding: 10px;
  background: #f8f9fa;
  border-radius: 8px;
}

.chart-view-options label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  font-weight: normal;
}


.chart-stats {
  background: #e8f4f8;
  padding: 15px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.stat-item {
  margin: 5px 0;
  font-size: 14px;
}

.stat-label {
  font-weight: bold;
  color: #2c3e50;
}

.stat-value {
  color: #42b983;
  font-family: monospace;
  font-size: 14px;
}

.chart-actions {
  display: flex;
  gap: 10px;
  justify-content: center;
}

.chart-actions .btn-secondary {
  margin-top: 0;
}

@media (max-width: 768px) {
  .container {
    padding: 10px;
  }
  
  .tabs {
    flex-direction: column;
  }
  
  .action-buttons {
    flex-direction: column;
  }
  
  .chart-type-selector {
    flex-direction: column;
  }
  
  .chart-view-options {
    flex-direction: column;
    gap: 10px;
  }
  
  .chart-canvas {
    height: 300px;
  }
  
  .results-table {
    font-size: 12px;
  }
  
  .results-table th, .results-table td {
    padding: 4px;
  }
}

/* Wrapper com altura fixa para o canvas - IMPLEMENTAÇÃO CORRIGIDA */
.chart-wrapper {
  position: relative;
  width: 100%;
  height: 450px;
  margin-bottom: 20px;
}

.chart-wrapper canvas {
  width: 100% !important;
  height: 100% !important;
  display: block;
}

/* Garante que o canvas tenha fundo branco */
canvas {
  background: white;
  border-radius: 8px;
}

/* Responsivo para mobile */
@media (max-width: 768px) {
  .chart-wrapper {
    height: 300px;
  }
}

/* Container fixo para o canvas */
.chart-container {
  position: relative;
  width: 100%;
  height: 450px;
  margin-bottom: 20px;
  background: white;
  border-radius: 8px;
  overflow: hidden;
}

.chart-container canvas {
  width: 100% !important;
  height: 100% !important;
  display: block;
}

/* Remove qualquer estilo que possa estar causando crescimento */
canvas {
  max-width: 100%;
  background: white;
}

/* Responsivo para mobile */
@media (max-width: 768px) {
  .chart-container {
    height: 300px;
  }
}
</style>