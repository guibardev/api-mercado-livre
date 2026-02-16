<script setup>
import { computed, ref, watch } from 'vue'

const siteId = ref('MLB')
const sellerId = ref('2278946593')
const cep = ref('01001000')
const limite = ref(20)
const carregando = ref(false)
const erro = ref('')
const anuncios = ref([])
const IDS_TESTE = ['MLB5321650590', 'MLB5321611740', 'MLB5321244626']

const TOKEN_STORAGE_KEY = 'ml_access_token'
const TAX_RATE_STORAGE_KEY = 'ml_tax_rate'
const COSTS_STORAGE_KEY = 'ml_product_costs'

const accessToken = ref(localStorage.getItem(TOKEN_STORAGE_KEY) || '')
const aliquotaImposto = ref(Number(localStorage.getItem(TAX_RATE_STORAGE_KEY) || 12))
const custoProdutosById = ref(JSON.parse(localStorage.getItem(COSTS_STORAGE_KEY) || '{}'))
const API_BASE = import.meta.env.DEV ? '/ml' : 'https://api.mercadolibre.com'

const filtroTipo = ref('todos')
const filtroFreteGratis = ref('todos')
const filtroMargemMinima = ref('')
const chaveOrdenacao = ref('margemContribuicao')
const direcaoOrdenacao = ref('desc')

const mapaTipos = {
  free: 'Gratis',
  bronze: 'Bronze',
  silver: 'Prata',
  gold: 'Ouro',
  gold_pro: 'Classico',
  gold_special: 'Premium',
  gold_premium: 'Premium Plus'
}

watch(accessToken, (novoToken) => {
  const valor = novoToken.trim()
  if (!valor) {
    localStorage.removeItem(TOKEN_STORAGE_KEY)
    return
  }
  localStorage.setItem(TOKEN_STORAGE_KEY, valor)
})

watch(aliquotaImposto, (novaAliquota) => {
  const valor = Number(novaAliquota)
  if (!Number.isFinite(valor) || valor < 0) {
    localStorage.removeItem(TAX_RATE_STORAGE_KEY)
    return
  }
  localStorage.setItem(TAX_RATE_STORAGE_KEY, String(valor))
})

watch(
  custoProdutosById,
  (novoMapa) => {
    localStorage.setItem(COSTS_STORAGE_KEY, JSON.stringify(novoMapa))
  },
  { deep: true }
)

const formatarMoeda = (valor) =>
  typeof valor === 'number'
    ? valor.toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' })
    : 'N/D'

const formatarPercentual = (valor) =>
  typeof valor === 'number' ? `${valor.toFixed(2).replace('.', ',')}%` : 'N/D'

function traduzirTipo(listingTypeId) {
  return mapaTipos[listingTypeId] || listingTypeId || 'N/D'
}

function obterNumero(valor) {
  const convertido = Number(valor)
  return Number.isFinite(convertido) ? convertido : null
}

function obterCustoProduto(itemId) {
  const valor = Number(custoProdutosById.value[itemId])
  return Number.isFinite(valor) ? valor : 0
}

function atualizarCustoProduto(itemId, valorDigitado) {
  const valor = Number(valorDigitado)
  custoProdutosById.value[itemId] = Number.isFinite(valor) && valor >= 0 ? valor : 0
}

function calcularImpostoProduto(anuncio) {
  if (typeof anuncio.valorVenda !== 'number') return null
  return anuncio.valorVenda * (Number(aliquotaImposto.value || 0) / 100)
}

function calcularTotalLiquido(anuncio) {
  if (
    typeof anuncio.valorVenda !== 'number' ||
    typeof anuncio.tarifaVenda !== 'number' ||
    typeof anuncio.envios !== 'number'
  ) {
    return null
  }

  return anuncio.valorVenda - anuncio.tarifaVenda - anuncio.envios
}

function calcularMargemContribuicao(anuncio) {
  const totalLiquido = calcularTotalLiquido(anuncio)
  const impostoProduto = calcularImpostoProduto(anuncio)
  if (typeof totalLiquido !== 'number' || typeof impostoProduto !== 'number') return null

  return totalLiquido - obterCustoProduto(anuncio.id) - impostoProduto
}

function calcularMargemPercentual(anuncio) {
  const margem = calcularMargemContribuicao(anuncio)
  if (typeof margem !== 'number' || typeof anuncio.valorVenda !== 'number' || anuncio.valorVenda === 0) {
    return null
  }
  return (margem / anuncio.valorVenda) * 100
}

function classeMargem(valor) {
  if (typeof valor !== 'number') return ''
  if (valor < 0) return 'margem-negativa'
  if (valor < 15) return 'margem-baixa'
  return 'margem-boa'
}

function trocarOrdenacao(chave) {
  if (chaveOrdenacao.value === chave) {
    direcaoOrdenacao.value = direcaoOrdenacao.value === 'asc' ? 'desc' : 'asc'
    return
  }
  chaveOrdenacao.value = chave
  direcaoOrdenacao.value = 'desc'
}

function indicadorOrdenacao(chave) {
  if (chaveOrdenacao.value !== chave) return '?'
  return direcaoOrdenacao.value === 'asc' ? '?' : '?'
}

function valorOrdenavel(row, chave) {
  if (chave === 'custoProduto') return obterCustoProduto(row.id)
  return row[chave]
}

const linhasCalculadas = computed(() =>
  anuncios.value.map((anuncio) => {
    const totalLiquido = calcularTotalLiquido(anuncio)
    const impostoProduto = calcularImpostoProduto(anuncio)
    const margemContribuicao = calcularMargemContribuicao(anuncio)
    const margemPercentual = calcularMargemPercentual(anuncio)

    return {
      ...anuncio,
      totalLiquido,
      impostoProduto,
      margemContribuicao,
      margemPercentual
    }
  })
)

const linhasFiltradasOrdenadas = computed(() => {
  let linhas = [...linhasCalculadas.value]

  if (filtroTipo.value !== 'todos') {
    linhas = linhas.filter((linha) => linha.tipo === filtroTipo.value)
  }

  if (filtroFreteGratis.value !== 'todos') {
    const esperado = filtroFreteGratis.value === 'sim'
    linhas = linhas.filter((linha) => linha.freteGratis === esperado)
  }

  const margemMinima = Number(filtroMargemMinima.value)
  if (Number.isFinite(margemMinima) && filtroMargemMinima.value !== '') {
    linhas = linhas.filter(
      (linha) => typeof linha.margemContribuicao === 'number' && linha.margemContribuicao >= margemMinima
    )
  }

  const chave = chaveOrdenacao.value
  const direcao = direcaoOrdenacao.value === 'asc' ? 1 : -1

  linhas.sort((a, b) => {
    const av = valorOrdenavel(a, chave)
    const bv = valorOrdenavel(b, chave)

    if (av === null || av === undefined) return 1
    if (bv === null || bv === undefined) return -1

    if (typeof av === 'string' && typeof bv === 'string') {
      return av.localeCompare(bv, 'pt-BR') * direcao
    }

    return (Number(av) - Number(bv)) * direcao
  })

  return linhas
})

const totalMargem = computed(() =>
  linhasFiltradasOrdenadas.value.reduce((soma, linha) => soma + (linha.margemContribuicao || 0), 0)
)

const ticketMedio = computed(() => {
  const validos = linhasFiltradasOrdenadas.value.filter((linha) => typeof linha.valorVenda === 'number')
  if (!validos.length) return null
  const soma = validos.reduce((total, linha) => total + linha.valorVenda, 0)
  return soma / validos.length
})

const percentualFreteGratis = computed(() => {
  const total = linhasFiltradasOrdenadas.value.length
  if (!total) return null
  const gratis = linhasFiltradasOrdenadas.value.filter((linha) => linha.freteGratis).length
  return (gratis / total) * 100
})

async function parseErroResposta(resposta, padrao) {
  try {
    const body = await resposta.json()
    return body?.message || body?.error || padrao
  } catch {
    return padrao
  }
}

async function mlFetch(url) {
  const token = accessToken.value.trim()
  const alvo = import.meta.env.DEV ? `${API_BASE}${url}` : new URL(`${API_BASE}${url}`)

  if (alvo instanceof URL && token) {
    alvo.searchParams.set('access_token', token)
  }

  if (import.meta.env.DEV) {
    const headers = {}
    if (token) headers.Authorization = `Bearer ${token}`
    return fetch(alvo, { headers })
  }

  return fetch(alvo.toString())
}

async function buscarTarifaVenda(item) {
  if (!item.category_id || !item.listing_type_id || typeof item.price !== 'number') {
    return null
  }

  const params = new URLSearchParams({
    price: String(item.price),
    listing_type_id: item.listing_type_id,
    category_id: item.category_id
  })

  const resposta = await mlFetch(`/sites/${siteId.value}/listing_prices?${params.toString()}`)
  if (!resposta.ok) return null

  const dados = await resposta.json()
  const detalhe = Array.isArray(dados) ? dados[0] : dados
  if (!detalhe) return null

  const saleFeeDetails = detalhe.sale_fee_details || {}
  return {
    saleFeeAmount: obterNumero(detalhe.sale_fee_amount),
    fixedFee: obterNumero(saleFeeDetails.fixed_fee),
    grossAmount: obterNumero(saleFeeDetails.gross_amount),
    percentageFee: obterNumero(saleFeeDetails.percentage_fee)
  }
}

async function buscarEnvios(item) {
  const fallbackEnvio =
    typeof item.shipping_cost === 'number' ? item.shipping_cost : item.shipping?.free_shipping ? 0 : null

  if (item.status !== 'active' || Number(item.available_quantity || 0) <= 0) {
    return { envioSelecionado: fallbackEnvio, opcoes: [] }
  }

  if (!cep.value) {
    return { envioSelecionado: fallbackEnvio, opcoes: [] }
  }

  const resposta = await mlFetch(`/items/${item.id}/shipping_options?zip_code=${cep.value}`)
  if (!resposta.ok) {
    return { envioSelecionado: fallbackEnvio, opcoes: [] }
  }

  const dados = await resposta.json()
  const opcoes = Array.isArray(dados.options)
    ? dados.options.map((opcao) => ({
        nome: opcao.name || 'Opcao',
        tipo: opcao.shipping_method_type || '',
        display: opcao.display || '',
        baseCost: obterNumero(opcao.base_cost),
        cost: obterNumero(opcao.cost),
        listCost: obterNumero(opcao.list_cost)
      }))
    : []

  const opcaoSelecionada = opcoes.find((opcao) => opcao.display === 'recommended') || opcoes[0] || null
  const envioSelecionado = opcaoSelecionada?.cost ?? opcaoSelecionada?.listCost ?? fallbackEnvio

  return { envioSelecionado, opcoes }
}

async function buscarItensAutenticado() {
  const respostaItens = await mlFetch(`/items?ids=${IDS_TESTE.join(',')}`)
  if (!respostaItens.ok) {
    const detalhe = await parseErroResposta(respostaItens, 'Nao foi possivel carregar detalhes dos anuncios.')
    throw new Error(detalhe)
  }

  const dadosItens = await respostaItens.json()
  return Array.isArray(dadosItens)
    ? dadosItens.filter((x) => x?.code === 200 && x?.body).map((x) => x.body)
    : []
}

async function carregarAnuncios() {
  if (!sellerId.value.trim()) {
    erro.value = 'Informe o Seller ID.'
    return
  }
  if (!accessToken.value.trim()) {
    erro.value = 'Informe o Access Token.'
    return
  }

  erro.value = ''
  carregando.value = true
  anuncios.value = []

  try {
    const resultados = await buscarItensAutenticado()

    anuncios.value = await Promise.all(
      resultados.map(async (item) => {
        const [tarifaInfo, envioInfo] = await Promise.all([buscarTarifaVenda(item), buscarEnvios(item)])

        return {
          id: item.id,
          titulo: item.title,
          tipo: traduzirTipo(item.listing_type_id),
          freteGratis: Boolean(item.shipping?.free_shipping),
          valorVenda: obterNumero(item.price),
          tarifaVenda: tarifaInfo?.saleFeeAmount ?? null,
          fixedFee: tarifaInfo?.fixedFee ?? null,
          grossAmount: tarifaInfo?.grossAmount ?? null,
          percentageFee: tarifaInfo?.percentageFee ?? null,
          envios: obterNumero(envioInfo.envioSelecionado),
          freteOpcoes: envioInfo.opcoes
        }
      })
    )
  } catch (e) {
    erro.value = e instanceof Error ? e.message : 'Erro inesperado ao buscar anuncios.'
  } finally {
    carregando.value = false
  }
}
</script>

<template>
  <main class="container">
    <h1>Anuncios Mercado Livre - Simulador</h1>

    <form class="filtros" @submit.prevent="carregarAnuncios">
      <label>
        Seller ID
        <input v-model="sellerId" type="text" placeholder="Ex.: 2278946593" />
      </label>
      <label>
        Access Token
        <input v-model="accessToken" type="password" autocomplete="off" placeholder="APP_USR-..." />
      </label>
      <label>
        Site
        <input v-model="siteId" type="text" />
      </label>
      <label>
        CEP (envio)
        <input v-model="cep" type="text" placeholder="Ex.: 01001000" />
      </label>
      <label>
        Limite
        <input v-model.number="limite" type="number" min="1" max="50" />
      </label>
      <label>
        Imposto %
        <input v-model.number="aliquotaImposto" type="number" min="0" step="0.01" />
      </label>
      <button type="submit" :disabled="carregando">
        {{ carregando ? 'Carregando...' : 'Buscar anuncios' }}
      </button>
    </form>

    <section class="kpis" v-if="linhasFiltradasOrdenadas.length">
      <article class="kpi-card">
        <span>Margem total</span>
        <strong>{{ formatarMoeda(totalMargem) }}</strong>
      </article>
      <article class="kpi-card">
        <span>Ticket medio</span>
        <strong>{{ formatarMoeda(ticketMedio) }}</strong>
      </article>
      <article class="kpi-card">
        <span>% frete gratis</span>
        <strong>{{ formatarPercentual(percentualFreteGratis) }}</strong>
      </article>
      <article class="kpi-card">
        <span>Anuncios filtrados</span>
        <strong>{{ linhasFiltradasOrdenadas.length }}</strong>
      </article>
    </section>

    <section class="filtros-rapidos">
      <label>
        Tipo
        <select v-model="filtroTipo">
          <option value="todos">Todos</option>
          <option value="Classico">Classico</option>
          <option value="Premium">Premium</option>
          <option value="Premium Plus">Premium Plus</option>
        </select>
      </label>
      <label>
        Frete gratis
        <select v-model="filtroFreteGratis">
          <option value="todos">Todos</option>
          <option value="sim">Sim</option>
          <option value="nao">Nao</option>
        </select>
      </label>
      <label>
        Margem minima (R$)
        <input v-model="filtroMargemMinima" type="number" step="0.01" min="0" placeholder="Ex.: 20" />
      </label>
    </section>

    <p v-if="erro" class="erro">{{ erro }}</p>

    <div class="table-wrap desktop-only">
      <table>
        <thead>
          <tr>
            <th class="sticky-col">ID</th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('titulo')">Titulo {{ indicadorOrdenacao('titulo') }}</button>
            </th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('tipo')">Tipo {{ indicadorOrdenacao('tipo') }}</button>
            </th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('freteGratis')">Frete gratis {{ indicadorOrdenacao('freteGratis') }}</button>
            </th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('valorVenda')">Valor venda {{ indicadorOrdenacao('valorVenda') }}</button>
            </th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('tarifaVenda')">Tarifa ML {{ indicadorOrdenacao('tarifaVenda') }}</button>
            </th>
            <th>Gross</th>
            <th>Fixed fee</th>
            <th>% fee</th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('envios')">Envios {{ indicadorOrdenacao('envios') }}</button>
            </th>
            <th>Fretes</th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('totalLiquido')">Total liquido {{ indicadorOrdenacao('totalLiquido') }}</button>
            </th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('custoProduto')">Custo produto {{ indicadorOrdenacao('custoProduto') }}</button>
            </th>
            <th>Imposto</th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('margemContribuicao')">Margem {{ indicadorOrdenacao('margemContribuicao') }}</button>
            </th>
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('margemPercentual')">% Margem {{ indicadorOrdenacao('margemPercentual') }}</button>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-if="!carregando && linhasFiltradasOrdenadas.length === 0">
            <td colspan="16">Nenhum anuncio para os filtros aplicados.</td>
          </tr>
          <tr v-for="linha in linhasFiltradasOrdenadas" :key="linha.id">
            <td class="sticky-col">{{ linha.id }}</td>
            <td>{{ linha.titulo }}</td>
            <td>{{ linha.tipo }}</td>
            <td>{{ linha.freteGratis ? 'Sim' : 'Nao' }}</td>
            <td>{{ formatarMoeda(linha.valorVenda) }}</td>
            <td>{{ formatarMoeda(linha.tarifaVenda) }}</td>
            <td>{{ formatarMoeda(linha.grossAmount) }}</td>
            <td>{{ formatarMoeda(linha.fixedFee) }}</td>
            <td>{{ formatarPercentual(linha.percentageFee) }}</td>
            <td>{{ formatarMoeda(linha.envios) }}</td>
            <td>
              <details v-if="linha.freteOpcoes?.length" class="fretes-detalhe">
                <summary>{{ linha.freteOpcoes.length }} opcoes</summary>
                <div class="fretes-lista">
                  <div v-for="(opcao, idx) in linha.freteOpcoes" :key="`${linha.id}-${idx}`">
                    {{ opcao.nome }} / {{ opcao.tipo || 'N/D' }}: {{ formatarMoeda(opcao.cost) }}
                  </div>
                </div>
              </details>
              <span v-else>N/D</span>
            </td>
            <td>{{ formatarMoeda(linha.totalLiquido) }}</td>
            <td>
              <input
                class="input-custo"
                type="number"
                min="0"
                step="0.01"
                :value="obterCustoProduto(linha.id)"
                @input="atualizarCustoProduto(linha.id, $event.target.value)"
              />
            </td>
            <td>{{ formatarMoeda(linha.impostoProduto) }}</td>
            <td :class="classeMargem(linha.margemContribuicao)">{{ formatarMoeda(linha.margemContribuicao) }}</td>
            <td :class="classeMargem(linha.margemContribuicao)">{{ formatarPercentual(linha.margemPercentual) }}</td>
          </tr>
        </tbody>
        <tfoot v-if="linhasFiltradasOrdenadas.length > 0">
          <tr>
            <td colspan="14">Total de margem de contribuicao (filtros atuais)</td>
            <td class="margem-boa">{{ formatarMoeda(totalMargem) }}</td>
            <td></td>
          </tr>
        </tfoot>
      </table>
    </div>

    <div class="mobile-only cards-wrap" v-if="linhasFiltradasOrdenadas.length > 0">
      <article class="mobile-card" v-for="linha in linhasFiltradasOrdenadas" :key="`m-${linha.id}`">
        <h3>{{ linha.titulo }}</h3>
        <p><strong>ID:</strong> {{ linha.id }}</p>
        <p><strong>Tipo:</strong> {{ linha.tipo }}</p>
        <p><strong>Frete gratis:</strong> {{ linha.freteGratis ? 'Sim' : 'Nao' }}</p>
        <p><strong>Valor venda:</strong> {{ formatarMoeda(linha.valorVenda) }}</p>
        <p><strong>Tarifa ML:</strong> {{ formatarMoeda(linha.tarifaVenda) }}</p>
        <p><strong>Envios:</strong> {{ formatarMoeda(linha.envios) }}</p>
        <p><strong>Total liquido:</strong> {{ formatarMoeda(linha.totalLiquido) }}</p>
        <p><strong>Imposto:</strong> {{ formatarMoeda(linha.impostoProduto) }}</p>
        <p :class="classeMargem(linha.margemContribuicao)">
          <strong>Margem:</strong> {{ formatarMoeda(linha.margemContribuicao) }}
          ({{ formatarPercentual(linha.margemPercentual) }})
        </p>
      </article>
    </div>
  </main>
</template>

<style scoped>
.container {
  width: 90vw;
  margin: 0 5vw;
  padding: 1.2rem 0;
}

h1 {
  margin-bottom: 1rem;
  font-size: 1.5rem;
}

.filtros,
.filtros-rapidos {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 0.75rem;
  margin-bottom: 1rem;
  align-items: end;
}

.filtros label,
.filtros-rapidos label {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
  font-size: 0.9rem;
}

.filtros input,
.filtros select,
.filtros-rapidos input,
.filtros-rapidos select,
.filtros button {
  height: 38px;
  border: 1px solid #cfcfcf;
  border-radius: 6px;
  padding: 0 0.65rem;
}

.filtros button {
  background: #156f3a;
  color: #fff;
  font-weight: 700;
  cursor: pointer;
}

.filtros button:disabled {
  opacity: 0.65;
  cursor: not-allowed;
}

.kpis {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(170px, 1fr));
  gap: 0.75rem;
  margin-bottom: 1rem;
}

.kpi-card {
  border: 1px solid #2d2d2d;
  border-radius: 8px;
  padding: 0.75rem;
  background: rgba(25, 30, 34, 0.85);
}

.kpi-card span {
  display: block;
  font-size: 0.8rem;
  color: #b9bec7;
}

.kpi-card strong {
  display: block;
  margin-top: 0.2rem;
  font-size: 1.1rem;
}

.erro {
  margin-bottom: 0.75rem;
  color: #ff7b8c;
}

.table-wrap {
  overflow: auto;
  max-height: 64vh;
  border: 1px solid #3a3a3a;
  border-radius: 8px;
}

table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
  min-width: 2200px;
}

th,
td {
  padding: 0.65rem;
  border-bottom: 1px solid #30343a;
  text-align: left;
  vertical-align: middle;
  background: #121416;
}

thead th {
  position: sticky;
  top: 0;
  z-index: 2;
  background: #dde3ec;
  color: #101827;
  font-weight: 700;
}

.sticky-col {
  position: sticky;
  left: 0;
  z-index: 1;
  background: #161a1f;
}

thead .sticky-col {
  z-index: 3;
  background: #cfd7e4;
  color: #101827;
}

.sort-btn {
  all: unset;
  cursor: pointer;
  font-weight: 700;
}

tfoot td {
  background: #1a2028;
  font-weight: 700;
}

.input-custo {
  width: 115px;
  height: 32px;
  border: 1px solid #cfcfcf;
  border-radius: 6px;
  padding: 0 0.5rem;
}

.fretes-detalhe summary {
  cursor: pointer;
  color: #9fc6ff;
}

.fretes-lista {
  margin-top: 0.4rem;
  min-width: 250px;
  font-size: 0.8rem;
  line-height: 1.35;
}

.margem-boa {
  color: #27c267;
  font-weight: 700;
}

.margem-baixa {
  color: #f7b500;
  font-weight: 700;
}

.margem-negativa {
  color: #ff6d6d;
  font-weight: 700;
}

.mobile-only {
  display: none;
}

@media (max-width: 1280px) {
  .container {
    width: 94vw;
    margin: 0 3vw;
  }

  h1 {
    font-size: 1.2rem;
  }

  .desktop-only {
    display: none;
  }

  .mobile-only {
    display: block;
  }

  .filtros,
  .filtros-rapidos {
    grid-template-columns: 1fr 1fr;
  }

  .filtros button {
    width: 100%;
  }

  .cards-wrap {
    display: grid;
    gap: 0.75rem;
  }

  .mobile-card {
    border: 1px solid #343a43;
    border-radius: 8px;
    background: #121416;
    padding: 0.75rem;
  }

  .mobile-card h3 {
    margin: 0 0 0.5rem;
    font-size: 0.95rem;
    line-height: 1.3;
  }

  .mobile-card p {
    margin: 0.25rem 0;
    font-size: 0.88rem;
  }

  .kpis {
    grid-template-columns: 1fr 1fr;
  }
}

@media (max-width: 720px) {
  .filtros,
  .filtros-rapidos,
  .kpis {
    grid-template-columns: 1fr;
  }
}
</style>
