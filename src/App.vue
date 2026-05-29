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

      <button @click="exportarCSV_IV" class="btn-secondary">📥 Exportar CSV (I/V)</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

// ============================================
// DETECÇÃO AUTOMÁTICA DO MODO (MOCK ou REAL)
// ============================================

// Detecta qual backend está rodando baseado na URL atual e em endpoints disponíveis
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

  // Se não estiver usando uma porta conhecida, tenta detectar automaticamente pela saúde da API.
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

// Modo de operação ('simples' ou 'iv')
const modoOperacao = ref('simples')

// Parâmetros para medição simples
const parametros = ref({
  num_leituras: 5,
  delay_entre_leituras: 0.5,
  modo_medicao: 'tensao',
  nivel_tensao_aplicada: 0.0
})

// Parâmetros para medição I/V
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

// Configuração do Mock (só usada se isMockMode for true)
const mockConfig = ref({
  modo: 'realista',
  ruido: 0.001,
  delay_artificial: 0.1,
  amplitude: 1.0,
  frequencia: 0.5,
  min: 0,
  max: 10
})

// Função auxiliar para requisições fetch
const apiRequest = async (url, options = {}) => {
  const response = await fetch(url, {
    headers: {
      'Content-Type': 'application/json',
      ...options.headers
    },
    ...options
  })
  
  if (!response.ok) {
    const error = await response.json().catch(() => ({ mensagem: 'Erro na requisição' }))
    throw new Error(error.mensagem || `HTTP ${response.status}`)
  }
  
  return response.json()
}

// ============================================
// COMPUTED PROPERTIES
// ============================================
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
// MÉTODOS - MOCK CONFIG (só chamado se for mock)
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
    await apiRequest(`${API_BASE_URL.value}/mock/reset`, {
      method: 'POST'
    })
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
    console.log('🔵 Iniciando medição simples...')
    console.log('📤 Parâmetros:', parametros.value)
    
    const data = await apiRequest(`${API_BASE_URL.value}/medir`, {
      method: 'POST',
      body: JSON.stringify(parametros.value)
    })
    
    if (data.status === 'sucesso') {
      resultados.value = data.medicoes
      statusMensagem.value = data.mensagem || 'Medição concluída com sucesso!'
      console.log('✅ Medição concluída:', resultados.value)
    }
  } catch (error) {
    console.error('❌ Erro:', error)
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
    console.log('🔵 Iniciando medição I/V...')
    console.log('📤 Parâmetros:', parametrosIV.value)
    
    const data = await apiRequest(`${API_BASE_URL.value}/medir_iv`, {
      method: 'POST',
      body: JSON.stringify(parametrosIV.value)
    })
    
    if (data.status === 'sucesso') {
      resultadosIV.value = data.medicoes
      statusMensagemIV.value = data.mensagem || 'Medição IV concluída com sucesso!'
      console.log('✅ Medição IV concluída:', resultadosIV.value.length, 'leituras')
    }
  } catch (error) {
    console.error('❌ Erro:', error)
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

/* Mode indicator */
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

/* Status bar */
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

/* Tabs */
.tabs {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
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

/* Cards */
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

/* Form groups */
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

/* Buttons */
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

/* Results */
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

/* Responsive */
@media (max-width: 768px) {
  .container {
    padding: 10px;
  }
  
  .tabs {
    flex-direction: column;
  }
  
  .results-table {
    font-size: 12px;
  }
  
  .results-table th, .results-table td {
    padding: 4px;
  }
}
</style>