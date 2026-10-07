<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// URL do Web App do Google Sheets (Substitua pela sua URL real)
const GOOGLE_SCRIPT_URL = 'COLE_AQUI_A_SUA_URL_DO_WEB_APP_DO_GOOGLE_SHEETS'

// Controlo de Navegação ('home', 'lista', 'form', 'sucesso')
const telaAtual = ref('home')
const termoBusca = ref('')

// Dados do Cronograma integrados com o localStorage
const defaultTrainings = [
  { id: "1", date: "01/10 (Qui)", dayNum: "01", time: "10h", title: "Integra + Recepção", role: "Recepção", instructor: "Jorge Emanoel", roomLink: "https://meet.google.com/abc-defg-hij", status: "Agendado" },
  { id: "3", date: "05/10 (Seg)", dayNum: "05", time: "10h", title: "Acolhimento para Pacientes de Risco e Prioritários", role: "Enfermagem", instructor: "Fabiane Praxedes", roomLink: "https://meet.google.com/xyz-uvwx-rst", status: "Agendado" },
  { id: "10", date: "07/10 (Qua)", dayNum: "07", time: "10h", title: "Primeiros Socorros", role: "Enfermagem", instructor: "Fabiane Praxedes", roomLink: "https://teams.microsoft.com/l/meetup-join/example", status: "Agendado" },
  { id: "11", date: "07/10 (Qua)", dayNum: "07", time: "10h", title: "Módulo 2 - Higienização e Desinfecção de Áreas e Equipamentos", role: "ASG", instructor: "Hotelaria", roomLink: "", status: "Agendado" },
  { id: "12", date: "07/10 (Qua)", dayNum: "07", time: "15h", title: "Eventos Adversos", role: "Enfermagem", instructor: "Fabiane Praxedes", roomLink: "", status: "Agendado" }
]

const listaTreinamentos = ref(
  JSON.parse(localStorage.getItem('outubro_2026_trainings_v3')) || defaultTrainings
)

// Dados do Formulário de Frequência
const treinamentoSelecionado = ref(null)
const nomeAluno = ref('')
const matricula = ref('')
const unidade = ref('')
const cargo = ref('')
const telefone = ref('')
const cpf = ref('')
const email = ref('')

const etapaFormulario = ref(1)
const aCarregar = ref(false)

const unidadesDisponiveis = ['Clínica Central', 'Unidade Matriz', 'Unidade Zona Sul', 'Unidade Norte']
const cargosDisponiveis = ['Assistente Administrativo', 'Técnico de Enfermagem', 'Enfermeira', 'Analista de Qualidade', 'Médico', 'Auxiliar de Produção']

// Temporizador para atualizar a verificação de tempo a cada minuto
const tempoAtual = ref(new Date())
let timerInterval = null

onMounted(() => {
  timerInterval = setInterval(() => {
    tempoAtual.value = new Date()
  }, 60000)
})

onUnmounted(() => {
  clearInterval(timerInterval)
})

// Função que valida se o treinamento está a menos de 15 minutos de iniciar ou a decorrer
const verificarLiberacaoTurma = (treino) => {
  if (treino.status === 'Realizado') return { liberado: true, motivo: 'Realizado' }
  if (treino.status === 'Cancelado') return { liberado: false, motivo: 'Cancelado' }

  try {
    const partesData = treino.date.split('/')
    const dia = parseInt(partesData[0])
    const mes = parseInt(partesData[1]) || 10
    const ano = 2026

    const horaMatch = treino.time.match(/\d+/)
    const hora = horaMatch ? parseInt(horaMatch[0]) : 8

    const dataTreino = new Date(ano, mes - 1, dia, hora, 0, 0)
    const agora = new Date()

    const quinzeMinutos = 15 * 60 * 1000
    const diferenca = dataTreino.getTime() - agora.getTime()

    if (diferenca <= quinzeMinutos && diferenca > -120 * 60 * 1000) {
      return { liberado: true, motivo: 'Liberado' }
    } else if (diferenca > quinzeMinutos) {
      return { liberado: false, motivo: 'Disponível apenas 15 min antes' }
    } else {
      return { liberado: false, motivo: 'Treinamento encerrado' }
    }
  } catch (e) {
    return { liberado: true, motivo: 'Liberado por segurança' }
  }
}

// Filtragem restrita aos treinamentos do dia atual (07/10/2026) e com status Agendado
const treinamentosFiltrados = computed(() => {
  // Data atual simulada do sistema (07/10/2026)
  const diaAtualStr = "07" 

  return listaTreinamentos.value.filter(t => {
    // Valida se é do dia atual e se está agendado
    const éDoDia = t.date.includes(diaAtualStr)
    const estaAgendado = (t.status || 'Agendado') === 'Agendado'

    // Filtra também pelo termo de busca digitado, se houver
    const matchBusca = !termoBusca.value || 
      t.title.toLowerCase().includes(termoBusca.value.toLowerCase()) ||
      t.instructor.toLowerCase().includes(termoBusca.value.toLowerCase())

    return éDoDia && estaAgendado && matchBusca
  })
})

const selecionarTreinamento = (treino) => {
  const statusLiberacao = verificarLiberacaoTurma(treino)
  if (!statusLiberacao.liberado) {
    alert(`Turma indisponível no momento: ${statusLiberacao.motivo}. O acesso é liberado apenas 15 minutos antes do início!`)
    return
  }

  treinamentoSelecionado.value = treino
  telaAtual.value = 'form'
  etapaFormulario.value = 1
}

const avancarEtapa = () => {
  if (!nomeAluno.value || !matricula.value || !unidade.value || !cargo.value) {
    alert('Por favor, preencha todos os campos obrigatórios desta etapa.')
    return
  }
  etapaFormulario.value = 2
}

const registrarFrequencia = async () => {
  if (!telefone.value || !cpf.value || !email.value) {
    alert('Por favor, preencha os campos de contato e documentos.')
    return
  }

  aCarregar.value = true

  const dadosPresenca = {
    dataRegistro: new Date().toLocaleDateString('pt-BR'),
    horaRegistro: new Date().toLocaleTimeString('pt-BR'),
    treinamento: treinamentoSelecionado.value.title,
    dataTreinamento: treinamentoSelecionado.value.date,
    instrutor: treinamentoSelecionado.value.instructor,
    nome: nomeAluno.value,
    matricula: matricula.value,
    unidade: unidade.value,
    cargo: cargo.value,
    telefone: telefone.value,
    cpf: cpf.value,
    email: email.value
  }

  try {
    if (GOOGLE_SCRIPT_URL.includes('COLE_AQUI')) {
      await new Promise(resolve => setTimeout(resolve, 1000))
      console.warn('Aviso: URL do Google Sheets não configurada. Simulando envio bem-sucedido.')
    } else {
      await fetch(GOOGLE_SCRIPT_URL, {
        method: 'POST',
        mode: 'no-cors',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(dadosPresenca)
      })
    }
    telaAtual.value = 'sucesso'
  } catch (error) {
    console.error('Erro ao enviar dados:', error)
    alert('Ocorreu um erro ao registar a frequência.')
  } finally {
    aCarregar.value = false
  }
}

const reiniciarFluxo = () => {
  nomeAluno.value = ''
  matricula.value = ''
  unidade.value = ''
  cargo.value = ''
  telefone.value = ''
  cpf.value = ''
  email.value = ''
  treinamentoSelecionado.value = null
  etapaFormulario.value = 1
  telaAtual.value = 'home'
}
</script>

<template>
  <div class="app-container">
    
    <!-- TELA 1: INICIAL -->
    <div v-if="telaAtual === 'home'" class="screen">
      <header class="hero-header">
        <div class="logo-box-pure">
          <div class="logo-icon-circle">
            <div class="people-dots"><span></span><span></span><span></span></div>
            <div class="orbit-arc"></div>
          </div>
          <div class="logo-text-group">
            <span class="brand-top">Melhoria Contínua</span>
            <span class="brand-main">Clínicas Médicas</span>
          </div>
        </div>
      </header>

      <div class="hero-card">
        <div class="icon-main-wrapper">
          <svg viewBox="0 0 24 24" width="36" height="36" stroke="#004d80" stroke-width="2" fill="none"><path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"></path><circle cx="9" cy="7" r="4"></circle><path d="M23 21v-2a4 4 0 0 0-3-3.87"></path><path d="M16 3.13a4 4 0 0 1 0 7.75"></path></svg>
        </div>
        <h2>Controle de Frequência</h2>
        <p>Acesse os treinamentos de hoje. O registo de presença é liberado 15 minutos antes de cada sessão.</p>
        <button @click="telaAtual = 'lista'" class="btn-primary">
          <svg viewBox="0 0 24 24" width="18" height="18" stroke="currentColor" stroke-width="2.5" fill="none"><polygon points="5 3 19 12 5 21 5 3"></polygon></svg>
          Acessar Treinamentos do Dia
        </button>
      </div>

      <div class="hero-footer-banner">
        <span>Capacitação hoje, resultados amanhã!</span>
      </div>
    </div>

    <!-- TELA 2: LISTA DE TREINAMENTOS DO DIA -->
    <div v-else-if="telaAtual === 'lista'" class="screen">
      <header class="top-bar">
        <button @click="telaAtual = 'home'" class="btn-back">
          <svg viewBox="0 0 24 24" width="20" height="20" stroke="currentColor" stroke-width="2.5" fill="none"><line x1="19" y1="12" x2="5" y2="12"></line><polyline points="12 19 5 12 12 5"></polyline></svg>
        </button>
        <div class="logo-box-pure mini">
          <div class="logo-icon-circle mini"><div class="people-dots"><span></span><span></span><span></span></div></div>
          <div class="logo-text-group">
            <span class="brand-top">Melhoria Contínua</span>
            <span class="brand-main">Clínicas Médicas</span>
          </div>
        </div>
      </header>

      <div class="content-body">
        <div class="search-box">
          <svg viewBox="0 0 24 24" width="18" height="18" stroke="#a0aec0" stroke-width="2" fill="none"><circle cx="11" cy="11" r="8"></circle><line x1="21" y1="21" x2="16.65" y2="16.65"></line></svg>
          <input type="text" v-model="termoBusca" placeholder="Filtrar treinamentos de hoje..." />
        </div>

        <div v-if="treinamentosFiltrados.length === 0" class="empty-state">
          <p>Nenhum treinamento agendado para hoje.</p>
        </div>

        <div class="training-list">
          <div v-for="treino in treinamentosFiltrados" :key="treino.id" 
               @click="selecionarTreinamento(treino)" 
               class="training-item-card"
               :class="{ 'disabled-card': !verificarLiberacaoTurma(treino).liberado }">
            
            <div class="training-icon">
              <svg viewBox="0 0 24 24" width="20" height="20" stroke="#004d80" stroke-width="2" fill="none"><path d="M2 3h6a4 4 0 0 1 4 4v14a3 3 0 0 0-3-3H2z"></path><path d="M22 3h-6a4 4 0 0 0-4 4v14a3 3 0 0 1 3-3h7z"></path></svg>
            </div>
            
            <div class="training-info">
              <h3>{{ treino.title }}</h3>
              <div class="training-meta">
                <span>📅 {{ treino.date }}</span>
                <span>⏰ {{ treino.time }}</span>
                <span class="instructor-tag">👤 {{ treino.instructor }}</span>
              </div>
              <div class="status-badge-container">
                <span v-if="verificarLiberacaoTurma(treino).liberado" class="badge-liberado">
                  🟢 Turma Liberada
                </span>
                <span v-else class="badge-bloqueado">
                  🔒 {{ verificarLiberacaoTurma(treino).motivo }}
                </span>
              </div>
            </div>
            
            <svg class="arrow-right" viewBox="0 0 24 24" width="16" height="16" stroke="#cbd5e0" stroke-width="2.5" fill="none"><polyline points="9 18 15 12 9 6"></polyline></svg>
          </div>
        </div>
      </div>
    </div>

    <!-- TELA 3 & 4: FORMULÁRIO DE PRESENÇA -->
    <div v-else-if="telaAtual === 'form'" class="screen">
      <header class="top-bar">
        <button @click="etapaFormulario === 2 ? etapaFormulario = 1 : telaAtual = 'lista'" class="btn-back">
          <svg viewBox="0 0 24 24" width="20" height="20" stroke="currentColor" stroke-width="2.5" fill="none"><line x1="19" y1="12" x2="5" y2="12"></line><polyline points="12 19 5 12 12 5"></polyline></svg>
        </button>
        <div class="logo-box-pure mini">
          <div class="logo-icon-circle mini"><div class="people-dots"><span></span><span></span><span></span></div></div>
          <div class="logo-text-group">
            <span class="brand-top">Melhoria Contínua</span>
            <span class="brand-main">Clínicas Médicas</span>
          </div>
        </div>
      </header>

      <div class="content-body">
        <div class="selected-badge-card">
          <div class="badge-icon"><svg viewBox="0 0 24 24" width="18" height="18" stroke="#004d80" stroke-width="2" fill="none"><path d="M2 3h6a4 4 0 0 1 4 4v14a3 3 0 0 0-3-3H2z"></path><path d="M22 3h-6a4 4 0 0 0-4 4v14a3 3 0 0 1 3-3h7z"></path></svg></div>
          <div>
            <strong>{{ treinamentoSelecionado.title }}</strong>
            <p>{{ treinamentoSelecionado.date }} • {{ treinamentoSelecionado.time }} • Inst: {{ treinamentoSelecionado.instructor }}</p>
          </div>
        </div>

        <!-- ETAPA 1 -->
        <div v-if="etapaFormulario === 1">
          <p class="form-instruction">Preencha seus dados para registrar sua presença no treinamento.</p>
          
          <div class="input-group">
            <label>Nome completo *</label>
            <div class="input-with-icon">
              <svg viewBox="0 0 24 24" width="16" height="16" stroke="#718096" stroke-width="2" fill="none"><path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path><circle cx="12" cy="7" r="4"></circle></svg>
              <input type="text" v-model="nomeAluno" placeholder="Digite seu nome completo" />
            </div>
          </div>

          <div class="input-group">
            <label>Matrícula *</label>
            <div class="input-with-icon">
              <svg viewBox="0 0 24 24" width="16" height="16" stroke="#718096" stroke-width="2" fill="none"><rect x="3" y="4" width="18" height="16" rx="2"></rect><line x1="7" y1="8" x2="17" y2="8"></line><line x1="7" y1="12" x2="12" y2="12"></line></svg>
              <input type="text" v-model="matricula" placeholder="Digite sua matrícula" />
            </div>
          </div>

          <div class="input-group">
            <label>Unidade *</label>
            <div class="input-with-icon">
              <svg viewBox="0 0 24 24" width="16" height="16" stroke="#718096" stroke-width="2" fill="none"><path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z"></path><circle cx="12" cy="10" r="3"></circle></svg>
              <select v-model="unidade">
                <option disabled value="">Selecione a unidade</option>
                <option v-for="u in unidadesDisponiveis" :key="u" :value="u">{{ u }}</option>
              </select>
            </div>
          </div>

          <div class="input-group">
            <label>Cargo *</label>
            <div class="input-with-icon">
              <svg viewBox="0 0 24 24" width="16" height="16" stroke="#718096" stroke-width="2" fill="none"><rect x="2" y="7" width="20" height="14" rx="2" ry="2"></rect><path d="M16 21V5a2 2 0 0 0-2-2h-4a2 2 0 0 0-2 2v16"></path></svg>
              <select v-model="cargo">
                <option disabled value="">Selecione o cargo</option>
                <option v-for="c in cargosDisponiveis" :key="c" :value="c">{{ c }}</option>
              </select>
            </div>
          </div>

          <button @click="avancarEtapa" class="btn-primary">Continuar →</button>
        </div>

        <!-- ETAPA 2 -->
        <div v-else>
          <div class="input-group">
            <label>Telefone *</label>
            <div class="input-with-icon">
              <svg viewBox="0 0 24 24" width="16" height="16" stroke="#718096" stroke-width="2" fill="none"><path d="M22 16.92v3a2 2 0 0 1-2.18 2 19.79 19.79 0 0 1-8.63-3.07 19.5 19.5 0 0 1-6-6 19.79 19.79 0 0 1-3.07-8.67A2 2 0 0 1 4.11 2h3a2 2 0 0 1 2 1.72 12.84 12.84 0 0 0 .7 2.81 2 2 0 0 1-.45 2.11L8.09 9.91a16 16 0 0 0 6 6l1.27-1.27a2 2 0 0 1 2.11-.45 12.84 12.84 0 0 0 2.81.7A2 2 0 0 1 22 16.92z"></path></svg>
              <input type="text" v-model="telefone" placeholder="(00) 00000-0000" />
            </div>
          </div>

          <div class="input-group">
            <label>CPF *</label>
            <div class="input-with-icon">
              <svg viewBox="0 0 24 24" width="16" height="16" stroke="#718096" stroke-width="2" fill="none"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path><polyline points="14 2 14 8 20 8"></polyline><line x1="16" y1="13" x2="8" y2="13"></line><line x1="16" y1="17" x2="8" y2="17"></line><polyline points="10 9 9 9 8 9"></polyline></svg>
              <input type="text" v-model="cpf" placeholder="Digite seu CPF" />
            </div>
          </div>

          <div class="input-group">
            <label>E-mail *</label>
            <div class="input-with-icon">
              <svg viewBox="0 0 24 24" width="16" height="16" stroke="#718096" stroke-width="2" fill="none"><path d="M4 4h16c1.1 0 2 .9 2 2v12c0 1.1-.9 2-2 2H4c-1.1 0-2-.9-2-2V6c0-1.1.9-2 2-2z"></path><polyline points="22,6 12,13 2,6"></polyline></svg>
              <input type="email" v-model="email" placeholder="Digite seu e-mail" />
            </div>
          </div>

          <div class="security-notice">
            <svg viewBox="0 0 24 24" width="16" height="16" stroke="#004d80" stroke-width="2" fill="none"><rect x="3" y="11" width="18" height="11" rx="2" ry="2"></rect><path d="M7 11V7a5 5 0 0 1 10 0v4"></path></svg>
            <span>Seus dados estão seguros e serão utilizados apenas para controle de presença nos treinamentos.</span>
          </div>

          <button @click="registrarFrequencia" class="btn-primary" :disabled="aCarregar">
            {{ aCarregar ? 'Registrando...' : 'Finalizar →' }}
          </button>
        </div>
      </div>
    </div>

    <!-- TELA 5: SUCESSO -->
    <div v-else-if="telaAtual === 'sucesso'" class="screen success-screen">
      <div class="success-card-box">
        <div class="success-circle-icon">
          <svg viewBox="0 0 24 24" width="40" height="40" stroke="#22c55e" stroke-width="3" fill="none"><polyline points="20 6 9 17 4 12"></polyline></svg>
        </div>
        <h2>Presença registrada!</h2>
        <p class="success-subtitle">Obrigado por participar do treinamento <strong>{{ treinamentoSelecionado.title }}</strong>.</p>
        
        <div class="receipt-box">
          <div class="receipt-row">
            <span>Treinamento</span>
            <strong>{{ treinamentoSelecionado.title }}</strong>
          </div>
          <div class="receipt-row">
            <span>Data / Hora</span>
            <strong>{{ treinamentoSelecionado.date }} • {{ treinamentoSelecionado.time }}</strong>
          </div>
          <div class="receipt-row">
            <span>Instrutor</span>
            <strong>{{ treinamentoSelecionado.instructor }}</strong>
          </div>
        </div>

        <button @click="reiniciarFluxo" class="btn-primary">Voltar para a lista</button>
      </div>
    </div>

  </div>
</template>

<style scoped>
.app-container {
  min-height: 100vh;
  background-color: #0f172a;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 1rem;
}

.screen {
  width: 100%;
  max-width: 420px;
  min-height: 720px;
  background: #f8fafc;
  border-radius: 24px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
}

.logo-box-pure {
  background: white;
  padding: 8px 16px;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  display: flex;
  align-items: center;
  gap: 10px;
}

.logo-box-pure.mini {
  padding: 4px 10px;
  box-shadow: none;
  background: transparent;
}

.logo-icon-circle {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.logo-icon-circle.mini {
  width: 28px;
  height: 28px;
}

.people-dots {
  display: flex;
  gap: 3px;
  z-index: 2;
  margin-bottom: -4px;
}

.people-dots span {
  width: 6px;
  height: 6px;
  background-color: #003366;
  border-radius: 50%;
}

.orbit-arc {
  position: absolute;
  width: 100%;
  height: 100%;
  border: 3px solid transparent;
  border-top-color: #004d80;
  border-right-color: #f97316;
  border-radius: 50%;
  transform: rotate(-45deg);
}

.logo-text-group {
  display: flex;
  flex-direction: column;
}

.brand-top {
  font-size: 0.85rem;
  font-weight: 700;
  color: #003366;
  line-height: 1.1;
}

.brand-main {
  font-size: 1.15rem;
  font-weight: 800;
  color: #f97316;
  line-height: 1.1;
}

.logo-box-pure.mini .brand-top {
  font-size: 0.65rem;
  color: #ffffff;
}

.logo-box-pure.mini .brand-main {
  font-size: 0.85rem;
  color: #fed7aa;
}

.hero-header {
  background: linear-gradient(135deg, #004d80 0%, #002b45 100%);
  padding: 2rem 1.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
}

.top-bar {
  background: #004d80;
  padding: 0.75rem 1rem;
  display: flex;
  align-items: center;
  gap: 1rem;
}

.btn-back {
  background: rgba(255,255,255,0.1);
  border: none;
  color: white;
  width: 36px;
  height: 36px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.hero-card {
  background: white;
  margin: 1.5rem;
  padding: 2rem 1.5rem;
  border-radius: 16px;
  text-align: center;
  box-shadow: 0 4px 12px rgba(0,0,0,0.05);
}

.icon-main-wrapper {
  width: 64px;
  height: 64px;
  background: #e0f2fe;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 1rem auto;
}

.hero-card h2 {
  color: #0f172a;
  font-size: 1.25rem;
  margin-bottom: 0.5rem;
}

.hero-card p {
  color: #64748b;
  font-size: 0.9rem;
  line-height: 1.4;
  margin-bottom: 1.5rem;
}

.hero-footer-banner {
  text-align: center;
  color: #64748b;
  font-size: 0.8rem;
  font-weight: 500;
  margin-top: auto;
  margin-bottom: 1.5rem;
}

.content-body {
  padding: 1.25rem;
  flex: 1;
  overflow-y: auto;
}

.search-box {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  display: flex;
  align-items: center;
  padding: 0.65rem 1rem;
  gap: 10px;
  margin-bottom: 1rem;
}

.search-box input {
  border: none;
  outline: none;
  width: 100%;
  font-size: 0.9rem;
  background: transparent;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.9rem;
}

.training-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.training-item-card {
  background: white;
  padding: 1rem;
  border-radius: 12px;
  display: flex;
  align-items: center;
  gap: 12px;
  border: 1px solid #e2e8f0;
  cursor: pointer;
  transition: all 0.2s;
}

.training-item-card:hover {
  border-color: #004d80;
  box-shadow: 0 4px 12px rgba(0,77,128,0.08);
}

.training-item-card.disabled-card {
  opacity: 0.55;
  background-color: #f1f5f9;
  border-color: #cbd5e1;
}

.training-icon {
  background: #f0f9ff;
  padding: 10px;
  border-radius: 10px;
}

.training-info {
  flex: 1;
}

.training-info h3 {
  font-size: 0.95rem;
  color: #1e293b;
  margin: 0 0 4px 0;
}

.training-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  font-size: 0.75rem;
  color: #64748b;
  margin-bottom: 6px;
}

.status-badge-container {
  display: flex;
  align-items: center;
}

.badge-liberado {
  font-size: 0.7rem;
  background: #dcfce7;
  color: #166534;
  padding: 2px 8px;
  border-radius: 6px;
  font-weight: 700;
}

.badge-bloqueado {
  font-size: 0.7rem;
  background: #fee2e2;
  color: #991b1b;
  padding: 2px 8px;
  border-radius: 6px;
  font-weight: 700;
}

.arrow-right {
  color: #cbd5e1;
}

.selected-badge-card {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 0.75rem 1rem;
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 1.25rem;
}

.badge-icon {
  background: #f0f9ff;
  padding: 8px;
  border-radius: 8px;
}

.selected-badge-card strong {
  font-size: 0.9rem;
  color: #0f172a;
  display: block;
}

.selected-badge-card p {
  font-size: 0.75rem;
  color: #64748b;
  margin: 2px 0 0 0;
}

.form-instruction {
  font-size: 0.85rem;
  color: #64748b;
  margin-bottom: 1rem;
}

.input-group {
  margin-bottom: 1rem;
  text-align: left;
}

.input-group label {
  display: block;
  font-size: 0.8rem;
  font-weight: 600;
  color: #334155;
  margin-bottom: 0.3rem;
}

.input-with-icon {
  background: white;
  border: 1px solid #cbd5e0;
  border-radius: 8px;
  display: flex;
  align-items: center;
  padding: 0 0.75rem;
  gap: 10px;
}

.input-with-icon input, .input-with-icon select {
  border: none;
  outline: none;
  width: 100%;
  padding: 0.7rem 0;
  font-size: 0.9rem;
  background: transparent;
}

.security-notice {
  background: #f1f5f9;
  padding: 0.75rem;
  border-radius: 8px;
  display: flex;
  gap: 8px;
  font-size: 0.75rem;
  color: #475569;
  align-items: flex-start;
  margin-bottom: 1.25rem;
}

.btn-primary {
  width: 100%;
  background-color: #004d80;
  color: white;
  border: none;
  padding: 0.85rem;
  font-size: 0.95rem;
  font-weight: 700;
  border-radius: 10px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  box-shadow: 0 4px 12px rgba(0, 77, 128, 0.2);
  transition: background 0.2s;
  margin-top: 1rem;
}

.btn-primary:hover {
  background-color: #003a60;
}

.success-screen {
  justify-content: center;
  align-items: center;
  padding: 1.5rem;
}

.success-card-box {
  background: white;
  width: 100%;
  padding: 2rem 1.5rem;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 10px 25px rgba(0,0,0,0.05);
}

.success-circle-icon {
  width: 72px;
  height: 72px;
  background: #dcfce7;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 1.25rem auto;
}

.success-card-box h2 {
  color: #0f172a;
  font-size: 1.35rem;
  margin-bottom: 0.5rem;
}

.success-subtitle {
  color: #64748b;
  font-size: 0.85rem;
  margin-bottom: 1.5rem;
  line-height: 1.4;
}

.receipt-box {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 1rem;
  text-align: left;
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 1.5rem;
}

.receipt-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.85rem;
}

.receipt-row span {
  color: #64748b;
}

.receipt-row strong {
  color: #1e293b;
}
</style>