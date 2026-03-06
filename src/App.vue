<script setup>
import { computed, ref, watch } from 'vue'

const siteId = ref('MLB')
const sellerId = ref('2326962319')
const marketplace = ref('ml')
const shopeeShopId = ref('')
const cep = ref('01001000')
const carregando = ref(false)
const erro = ref('')
const anuncios = ref([])

const TOKEN_STORAGE_KEY = 'ml_access_token'
const SHOPEE_TOKEN_STORAGE_KEY = 'shopee_access_token'
const TAX_RATE_STORAGE_KEY = 'ml_tax_rate'
const COSTS_STORAGE_KEY = 'ml_product_costs'
const FREIGHTS_STORAGE_KEY = 'ml_freight_simulation'
const THEME_STORAGE_KEY = 'ui_theme'
const ADS_CACHE_STORAGE_KEY = 'ml_ads_cache_v1'

const accessToken = ref('APP_USR-7222342522058077-030616-3ea319770fa421f3bcc17425dad5632d-2326962319')
const shopeeAccessToken = ref(localStorage.getItem(SHOPEE_TOKEN_STORAGE_KEY) || '')
const aliquotaImposto = ref(Number(localStorage.getItem(TAX_RATE_STORAGE_KEY) || 12))
const custoProdutosById = ref(JSON.parse(localStorage.getItem(COSTS_STORAGE_KEY) || '{}'))
const freteSimuladoById = ref(JSON.parse(localStorage.getItem(FREIGHTS_STORAGE_KEY) || '{}'))
const tema = ref(localStorage.getItem(THEME_STORAGE_KEY) || 'dark')
const API_BASE = import.meta.env.DEV ? '/ml' : 'https://api.mercadolibre.com'

const filtroTipo = ref('todos')
const filtroFreteGratis = ref('todos')
const filtroMargemMinima = ref('')
const chaveOrdenacao = ref('margemContribuicao')
const direcaoOrdenacao = ref('desc')
const ultimaAtualizacaoCache = ref('')

const mapaTipos = {
  free: 'Gratis',
  bronze: 'Bronze',
  silver: 'Prata',
  gold: 'Ouro',
  gold_pro: 'Classico',
  gold_special: 'Premium',
  gold_premium: 'Premium Plus'
}

watch(shopeeAccessToken, (novoToken) => {
  const valor = novoToken.trim()
  if (!valor) {
    localStorage.removeItem(SHOPEE_TOKEN_STORAGE_KEY)
    return
  }
  localStorage.setItem(SHOPEE_TOKEN_STORAGE_KEY, valor)
})

watch(
  tema,
  (novoTema) => {
    localStorage.setItem(THEME_STORAGE_KEY, novoTema)
    if (typeof document !== 'undefined') {
      document.body.classList.toggle('app-light', novoTema === 'light')
      document.body.classList.toggle('app-dark', novoTema === 'dark')
    }
  },
  { immediate: true }
)

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

watch(
  freteSimuladoById,
  (novoMapa) => {
    localStorage.setItem(FREIGHTS_STORAGE_KEY, JSON.stringify(novoMapa))
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

function obterFreteSimulado(itemId) {
  const valor = Number(freteSimuladoById.value[itemId])
  return Number.isFinite(valor) && valor >= 0 ? valor : null
}

function atualizarFreteSimulado(itemId, valorDigitado) {
  const texto = String(valorDigitado || '').trim()
  if (texto === '') {
    delete freteSimuladoById.value[itemId]
    return
  }
  const valor = Number(texto)
  freteSimuladoById.value[itemId] = Number.isFinite(valor) && valor >= 0 ? valor : null
}

function obterFreteUsado(anuncio) {
  const simulado = obterFreteSimulado(anuncio.id)
  if (typeof simulado === 'number') return simulado
  return anuncio.envios
}

function calcularImpostoProduto(anuncio) {
  if (typeof anuncio.valorVenda !== 'number') return null
  return anuncio.valorVenda * (Number(aliquotaImposto.value || 0) / 100)
}

function calcularTotalLiquido(anuncio) {
  const freteUsado = obterFreteUsado(anuncio)
  if (
    typeof anuncio.valorVenda !== 'number' ||
    typeof anuncio.tarifaVenda !== 'number' ||
    typeof freteUsado !== 'number'
  ) {
    return null
  }

  return anuncio.valorVenda - anuncio.tarifaVenda - freteUsado
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
  if (chave === 'freteUsado') return obterFreteUsado(row)
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

// Shopee manual calculator (no API integration required)
const modoShopee = ref('verificar_lucro')
const shopeeCustoProduto = ref(0)
const shopeeCustoOperacional = ref(0)
const shopeeImpostosOutrosPercent = ref(0)
const shopeePrecoVenda = ref(0)
const shopeeMargemDesejadaPercent = ref(20)
const shopeeTaxaComissaoPercent = ref(20)
const shopeeTaxasAdicionaisPercent = ref(0)
const shopeePagamentoPix = ref(false)
const shopeeSubsidioPixPercent = ref(0)
const shopeeCupomFreteGratis = ref(false)
const shopeeValorFreteCupom = ref(0)
const shopeeVaiRodarAds = ref(false)
const shopeeRoas = ref(1)

const shopeeTaxaFixa = computed(() => (Number(shopeePrecoVenda.value || 0) <= 79.99 ? 4 : 0))
const shopeeComissaoValor = computed(
  () => (Number(shopeePrecoVenda.value || 0) * Number(shopeeTaxaComissaoPercent.value || 0)) / 100
)
const shopeeTaxasAdicionaisValor = computed(
  () => (Number(shopeePrecoVenda.value || 0) * Number(shopeeTaxasAdicionaisPercent.value || 0)) / 100
)
const shopeeImpostosValor = computed(
  () => (Number(shopeePrecoVenda.value || 0) * Number(shopeeImpostosOutrosPercent.value || 0)) / 100
)
const shopeeSubsidioPixValor = computed(() =>
  shopeePagamentoPix.value
    ? (Number(shopeePrecoVenda.value || 0) * Number(shopeeSubsidioPixPercent.value || 0)) / 100
    : 0
)
const shopeeCoparticipacaoCupomValor = computed(() =>
  shopeeCupomFreteGratis.value ? Number(shopeeValorFreteCupom.value || 0) * 0.25 : 0
)
const shopeeCustoAdsValor = computed(() =>
  shopeeVaiRodarAds.value && Number(shopeeRoas.value) > 0
    ? Number(shopeePrecoVenda.value || 0) / Number(shopeeRoas.value)
    : 0
)
const shopeeTaxasTotais = computed(
  () =>
    shopeeComissaoValor.value +
    shopeeTaxaFixa.value +
    shopeeTaxasAdicionaisValor.value +
    shopeeImpostosValor.value +
    shopeeCoparticipacaoCupomValor.value +
    shopeeCustoAdsValor.value -
    shopeeSubsidioPixValor.value
)
const shopeeLucroLiquido = computed(
  () =>
    Number(shopeePrecoVenda.value || 0) -
    Number(shopeeCustoProduto.value || 0) -
    Number(shopeeCustoOperacional.value || 0) -
    shopeeTaxasTotais.value
)
const shopeeMargemLiquida = computed(() => {
  const preco = Number(shopeePrecoVenda.value || 0)
  if (!preco) return 0
  return (shopeeLucroLiquido.value / preco) * 100
})
const shopeePrecoSugerido = computed(() => {
  const comissao = Number(shopeeTaxaComissaoPercent.value || 0) / 100
  const taxasAdic = Number(shopeeTaxasAdicionaisPercent.value || 0) / 100
  const impostos = Number(shopeeImpostosOutrosPercent.value || 0) / 100
  const margemMeta = Number(shopeeMargemDesejadaPercent.value || 0) / 100
  const subsidioPix = shopeePagamentoPix.value
    ? Number(shopeeSubsidioPixPercent.value || 0) / 100
    : 0
  const adsPercent =
    shopeeVaiRodarAds.value && Number(shopeeRoas.value) > 0 ? 1 / Number(shopeeRoas.value) : 0

  const baseFixa =
    Number(shopeeCustoProduto.value || 0) +
    Number(shopeeCustoOperacional.value || 0) +
    shopeeCoparticipacaoCupomValor.value

  const percentualTotal = comissao + taxasAdic + impostos + margemMeta + adsPercent - subsidioPix
  if (percentualTotal >= 1) return 0

  // tentativa na faixa com taxa fixa de R$4,00
  let sugestao = (baseFixa + 4) / (1 - percentualTotal)
  if (sugestao > 79.99) {
    // acima de R$79,99 remove taxa fixa
    sugestao = baseFixa / (1 - percentualTotal)
  }
  return sugestao > 0 ? sugestao : 0
})

function alternarTema() {
  tema.value = tema.value === 'dark' ? 'light' : 'dark'
}

function esperar(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

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

async function mlFetchComRetentativa(url, tentativas = 3) {
  for (let tentativa = 0; tentativa < tentativas; tentativa += 1) {
    const resposta = await mlFetch(url)
    if (resposta.status !== 429 && resposta.status < 500) return resposta
    if (tentativa === tentativas - 1) return resposta
    await esperar(600 * (tentativa + 1))
  }
}

function normalizarItemCache(item) {
  return {
    id: item.id,
    title: item.title,
    listing_type_id: item.listing_type_id,
    category_id: item.category_id,
    price: item.price,
    shipping_cost: item.shipping_cost,
    status: item.status,
    available_quantity: item.available_quantity,
    shipping: {
      free_shipping: Boolean(item.shipping?.free_shipping)
    }
  }
}

function mapearItemParaLinha(item) {
  return {
    id: item.id,
    titulo: item.title,
    tipo: traduzirTipo(item.listing_type_id),
    freteGratis: Boolean(item.shipping?.free_shipping),
    valorVenda: obterNumero(item.price),
    tarifaVenda: null,
    fixedFee: null,
    grossAmount: null,
    percentageFee: null,
    envios: obterNumero(item.shipping_cost),
    freteOpcoes: []
  }
}

function obterCacheAnuncios() {
  try {
    const bruto = localStorage.getItem(ADS_CACHE_STORAGE_KEY)
    if (!bruto) return null
    const dados = JSON.parse(bruto)
    const seller = sellerId.value.trim()
    const site = siteId.value.trim()
    if (dados?.sellerId !== seller || dados?.siteId !== site) return null
    if (!Array.isArray(dados?.items)) return null
    ultimaAtualizacaoCache.value = dados.updatedAt || ''
    return dados.items
  } catch {
    return null
  }
}

function salvarCacheAnuncios(items) {
  const payload = {
    sellerId: sellerId.value.trim(),
    siteId: siteId.value.trim(),
    updatedAt: new Date().toISOString(),
    items
  }
  localStorage.setItem(ADS_CACHE_STORAGE_KEY, JSON.stringify(payload))
  ultimaAtualizacaoCache.value = payload.updatedAt
}

function limparCacheAnuncios() {
  localStorage.removeItem(ADS_CACHE_STORAGE_KEY)
  ultimaAtualizacaoCache.value = ''
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

async function buscarIdsAnunciosVendedor() {
  const seller = sellerId.value.trim()
  const limitePagina = 100
  let scrollId = ''
  const ids = []

  while (true) {
    const params = new URLSearchParams({
      search_type: 'scan',
      limit: String(limitePagina)
    })
    if (scrollId) params.set('scroll_id', scrollId)

    const respostaIds = await mlFetchComRetentativa(`/users/${seller}/items/search?${params.toString()}`)

    if (!respostaIds.ok) {
      const detalhe = await parseErroResposta(respostaIds, 'Nao foi possivel carregar os anuncios do vendedor.')
      throw new Error(detalhe)
    }

    const dadosIds = await respostaIds.json()
    if (!dadosIds) break
    const resultadosPagina = Array.isArray(dadosIds?.results) ? dadosIds.results : []
    if (!resultadosPagina.length) break
    ids.push(...resultadosPagina)

    if (!dadosIds.scroll_id || dadosIds.scroll_id === scrollId) break
    scrollId = dadosIds.scroll_id
    await esperar(180)
  }

  return [...new Set(ids)]
}

async function buscarItensAutenticado() {
  const idsAnuncios = await buscarIdsAnunciosVendedor()
  if (!idsAnuncios.length) return []

  const tamanhoLote = 20
  const atributos =
    'id,title,listing_type_id,category_id,price,shipping,shipping_cost,status,available_quantity'
  const itens = []

  for (let i = 0; i < idsAnuncios.length; i += tamanhoLote) {
    const lote = idsAnuncios.slice(i, i + tamanhoLote)
    const respostaItens = await mlFetchComRetentativa(`/items?ids=${lote.join(',')}&attributes=${atributos}`)
    if (!respostaItens.ok) {
      const detalhe = await parseErroResposta(respostaItens, 'Nao foi possivel carregar detalhes dos anuncios.')
      throw new Error(detalhe)
    }
    const dadosItens = await respostaItens.json()
    const itensLote = Array.isArray(dadosItens)
      ? dadosItens.filter((x) => x?.code === 200 && x?.body).map((x) => normalizarItemCache(x.body))
      : []
    itens.push(...itensLote)
    await esperar(140)
  }

  return itens
}

async function carregarAnunciosMercadoLivre(forcarAtualizacao = false) {
  const cache = forcarAtualizacao ? null : obterCacheAnuncios()
  if (cache?.length) {
    return cache.map(mapearItemParaLinha)
  }

  const resultados = await buscarItensAutenticado()
  salvarCacheAnuncios(resultados)
  return resultados.map(mapearItemParaLinha)
}

async function carregarAnunciosShopee() {
  return []
}

async function carregarAnuncios(forcarAtualizacao = false) {
  erro.value = ''
  carregando.value = true
  anuncios.value = []

  try {
    if (marketplace.value === 'ml') {
      if (!sellerId.value.trim()) {
        throw new Error('Informe o Seller ID.')
      }
      if (!accessToken.value.trim()) {
        throw new Error('Informe o Access Token.')
      }
      if (forcarAtualizacao) limparCacheAnuncios()
      anuncios.value = await carregarAnunciosMercadoLivre(forcarAtualizacao)
    } else {
      erro.value = ''
      anuncios.value = await carregarAnunciosShopee()
    }
  } catch (e) {
    erro.value = e instanceof Error ? e.message : 'Erro inesperado ao buscar anuncios.'
  } finally {
    carregando.value = false
  }
}
</script>

<template>
  <main class="container" :class="`theme-${tema}`">
    <div class="topbar">
      <h1>Simulador de Margem - Marketplace</h1>
      <button
        class="theme-btn lamp-toggle"
        :class="{ on: tema === 'light' }"
        type="button"
        :aria-label="tema === 'dark' ? 'Ativar tema claro' : 'Ativar tema escuro'"
        @click="alternarTema"
      >
        <span class="lamp-icon" aria-hidden="true"></span>
      </button>
    </div>

    <form class="filtros" @submit.prevent="carregarAnuncios">
      <label>
        Marketplace
        <select v-model="marketplace">
          <option value="ml">Mercado Livre</option>
          <option value="shopee">Shopee</option>
        </select>
      </label>
      <label v-if="marketplace === 'ml'">
        Seller ID
        <input v-model="sellerId" type="text" readonly />
      </label>
      <label v-else>
        Shop ID
        <input v-model="shopeeShopId" type="text" placeholder="Ex.: 123456789" />
      </label>
      <label>
        {{ marketplace === 'ml' ? 'Access Token ML' : 'Access Token Shopee' }}
        <input
          v-if="marketplace === 'ml'"
          v-model="accessToken"
          type="password"
          autocomplete="off"
          readonly
        />
        <input
          v-else
          v-model="shopeeAccessToken"
          type="password"
          autocomplete="off"
          placeholder="Shopee token"
        />
      </label>
      <label v-if="marketplace === 'ml'">
        Site
        <input v-model="siteId" type="text" />
      </label>
      <label v-if="marketplace === 'ml'">
        CEP (envio)
        <input v-model="cep" type="text" placeholder="Ex.: 01001000" />
      </label>
      <label v-if="marketplace === 'ml'">
        Imposto %
        <input v-model.number="aliquotaImposto" type="number" min="0" step="0.01" />
      </label>
      <button type="submit" :disabled="carregando">
        {{ carregando ? 'Carregando...' : marketplace === 'ml' ? 'Buscar anuncios' : 'Atualizar calculo' }}
      </button>
      <button
        v-if="marketplace === 'ml'"
        type="button"
        :disabled="carregando"
        @click="carregarAnuncios(true)"
      >
        Sincronizar da API
      </button>
      <small v-if="marketplace === 'ml' && ultimaAtualizacaoCache">
        Cache local: {{ new Date(ultimaAtualizacaoCache).toLocaleString('pt-BR') }}
      </small>
    </form>

    <section v-if="marketplace === 'shopee'" class="shopee-grid">
      <article class="shopee-card">
        <h3>Dados do produto</h3>
        <label>
          Custo do produto (R$)
          <input v-model.number="shopeeCustoProduto" type="number" min="0" step="0.01" />
        </label>
        <label>
          Custo operacional / pedido (R$)
          <input v-model.number="shopeeCustoOperacional" type="number" min="0" step="0.01" />
        </label>
        <label>
          Impostos / outros (%)
          <input v-model.number="shopeeImpostosOutrosPercent" type="number" min="0" step="0.01" />
        </label>
        <label>
          Preco de venda (R$)
          <input v-model.number="shopeePrecoVenda" type="number" min="0" step="0.01" />
        </label>
      </article>

      <article class="shopee-card">
        <h3>Taxas</h3>
        <label>
          Comissao Shopee (%)
          <input v-model.number="shopeeTaxaComissaoPercent" type="number" min="0" step="0.01" />
        </label>
        <label>
          Taxas adicionais (%)
          <input v-model.number="shopeeTaxasAdicionaisPercent" type="number" min="0" step="0.01" />
        </label>
        <label>
          Margem desejada (%)
          <input v-model.number="shopeeMargemDesejadaPercent" type="number" min="0" step="0.01" />
        </label>
        <p class="calc-line">Taxa fixa: <strong>{{ formatarMoeda(shopeeTaxaFixa) }}</strong></p>
      </article>

      <article class="shopee-card">
        <h3>Opcoes</h3>
        <label class="opt-line switch-row">
          <span class="opt-text">Pagamento via Pix</span>
          <span class="switch">
            <input v-model="shopeePagamentoPix" class="switch-input" type="checkbox" />
            <span class="switch-slider"></span>
          </span>
        </label>
        <label v-if="shopeePagamentoPix">
          Subsidio Pix (%)
          <input v-model.number="shopeeSubsidioPixPercent" type="number" min="0" step="0.01" />
        </label>
        <label class="opt-line switch-row">
          <span class="opt-text">Cupom frete gratis</span>
          <span class="switch">
            <input v-model="shopeeCupomFreteGratis" class="switch-input" type="checkbox" />
            <span class="switch-slider"></span>
          </span>
        </label>
        <label v-if="shopeeCupomFreteGratis">
          Valor frete do cupom (R$)
          <input v-model.number="shopeeValorFreteCupom" type="number" min="0" step="0.01" />
        </label>
        <label class="opt-line switch-row">
          <span class="opt-text">Vai rodar Shopee Ads</span>
          <span class="switch">
            <input v-model="shopeeVaiRodarAds" class="switch-input" type="checkbox" />
            <span class="switch-slider"></span>
          </span>
        </label>
        <label v-if="shopeeVaiRodarAds">
          ROAS
          <input v-model.number="shopeeRoas" type="number" min="0.01" step="0.01" />
        </label>
      </article>

      <article class="shopee-card shopee-result">
        <h3>Resultado</h3>
        <p>Valor final da venda: <strong>{{ formatarMoeda(shopeePrecoVenda) }}</strong></p>
        <p>Taxas Shopee: <strong>{{ formatarMoeda(shopeeTaxasTotais) }}</strong></p>
        <p>Lucro liquido: <strong>{{ formatarMoeda(shopeeLucroLiquido) }}</strong></p>
        <p>Margem liquida: <strong>{{ formatarPercentual(shopeeMargemLiquida) }}</strong></p>
        <p>Preco sugerido: <strong>{{ formatarMoeda(shopeePrecoSugerido) }}</strong></p>
        <p class="calc-line">Comissao: {{ formatarMoeda(shopeeComissaoValor) }}</p>
        <p class="calc-line">Taxas adicionais: {{ formatarMoeda(shopeeTaxasAdicionaisValor) }}</p>
        <p class="calc-line">Impostos: {{ formatarMoeda(shopeeImpostosValor) }}</p>
        <p class="calc-line">Subsidio Pix: {{ formatarMoeda(shopeeSubsidioPixValor) }}</p>
        <p class="calc-line">Coparticipacao cupom: {{ formatarMoeda(shopeeCoparticipacaoCupomValor) }}</p>
        <p class="calc-line">Custo Ads: {{ formatarMoeda(shopeeCustoAdsValor) }}</p>
      </article>
    </section>

    <section class="kpis" v-if="marketplace === 'ml' && linhasFiltradasOrdenadas.length">
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

    <section class="filtros-rapidos" v-if="marketplace === 'ml'">
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

    <div class="table-wrap desktop-only" v-if="marketplace === 'ml'">
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
            <th>
              <button class="sort-btn" @click="trocarOrdenacao('freteUsado')">Frete usado {{ indicadorOrdenacao('freteUsado') }}</button>
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
            <td colspan="17">Nenhum anuncio para os filtros aplicados.</td>
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
              <input
                class="input-custo"
                type="number"
                min="0"
                step="0.01"
                :value="obterFreteSimulado(linha.id) ?? ''"
                placeholder="API"
                @input="atualizarFreteSimulado(linha.id, $event.target.value)"
              />
              <div class="frete-api">API: {{ formatarMoeda(linha.envios) }}</div>
            </td>
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
            <td colspan="15">Total de margem de contribuicao (filtros atuais)</td>
            <td class="margem-boa">{{ formatarMoeda(totalMargem) }}</td>
            <td></td>
          </tr>
        </tfoot>
      </table>
    </div>

    <div class="mobile-only cards-wrap" v-if="marketplace === 'ml' && linhasFiltradasOrdenadas.length > 0">
      <article class="mobile-card" v-for="linha in linhasFiltradasOrdenadas" :key="`m-${linha.id}`">
        <h3>{{ linha.titulo }}</h3>
        <p><strong>ID:</strong> {{ linha.id }}</p>
        <p><strong>Tipo:</strong> {{ linha.tipo }}</p>
        <p><strong>Frete gratis:</strong> {{ linha.freteGratis ? 'Sim' : 'Nao' }}</p>
        <p><strong>Valor venda:</strong> {{ formatarMoeda(linha.valorVenda) }}</p>
        <p><strong>Tarifa ML:</strong> {{ formatarMoeda(linha.tarifaVenda) }}</p>
        <p><strong>Envios (API):</strong> {{ formatarMoeda(linha.envios) }}</p>
        <p><strong>Frete usado:</strong> {{ formatarMoeda(obterFreteUsado(linha)) }}</p>
        <p>
          <strong>Frete simulado:</strong>
          <input
            class="input-custo"
            type="number"
            min="0"
            step="0.01"
            :value="obterFreteSimulado(linha.id) ?? ''"
            placeholder="API"
            @input="atualizarFreteSimulado(linha.id, $event.target.value)"
          />
        </p>
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
  margin: 1rem 5vw;
  padding: 1.2rem 1rem;
  border-radius: 10px;
  --panel-bg: #121416;
  --card-bg: rgba(25, 30, 34, 0.85);
  --card-border: #2d2d2d;
  --line: #30343a;
  --header-bg: #dde3ec;
  --header-fg: #101827;
  --input-border: #cfcfcf;
  --muted: #9aa4b2;
  --error: #ff7b8c;
  --switch-off: #4a4f57;
  --text-main: #e6ebf2;
  --surface-bg: #111317;
  background: var(--surface-bg);
  color: var(--text-main);
}

.theme-light {
  --panel-bg: #ffffff;
  --card-bg: #f7f9fc;
  --card-border: #d6dce8;
  --line: #e4e8f0;
  --header-bg: #ecf1f8;
  --header-fg: #152033;
  --input-border: #cfd7e4;
  --muted: #5f6f86;
  --error: #c0262d;
  --switch-off: #b5bcc9;
  --text-main: #111827;
  --surface-bg: #f3f6fb;
}

.theme-dark {
  --panel-bg: #121416;
  --card-bg: rgba(25, 30, 34, 0.85);
  --surface-bg: #111317;
}

:global(body.app-light) {
  background: #f3f6fb !important;
  color: #111827 !important;
}

:global(body.app-dark) {
  background: #111317 !important;
  color: #e6ebf2 !important;
}

.topbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 0.75rem;
}

.theme-btn {
  height: 36px;
  border: 1px solid var(--input-border);
  background: transparent;
  color: var(--text-main);
  border-radius: 999px;
  padding: 0 0.85rem;
  font-weight: 600;
  cursor: pointer;
}

.lamp-toggle {
  width: 42px;
  padding: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 999px;
}

.lamp-icon {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 2px solid currentColor;
  position: relative;
  opacity: 0.75;
}

.lamp-icon::before {
  content: '';
  position: absolute;
  width: 6px;
  height: 3px;
  border: 2px solid currentColor;
  border-top: 0;
  border-radius: 0 0 4px 4px;
  left: 50%;
  transform: translateX(-50%);
  top: 12px;
}

.lamp-toggle.on {
  color: #f6a623;
  box-shadow: 0 0 0 2px rgba(246, 166, 35, 0.25);
}

.lamp-toggle.on .lamp-icon {
  background: #ffd77a;
  box-shadow: 0 0 10px rgba(255, 196, 72, 0.85);
  opacity: 1;
}

h1 {
  margin-bottom: 0.8rem;
  font-size: 1.5rem;
  color: var(--text-main);
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

.shopee-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(260px, 1fr));
  gap: 0.75rem;
  margin-bottom: 1rem;
}

.shopee-card {
  border: 1px solid var(--card-border);
  border-radius: 8px;
  padding: 0.75rem;
  background: var(--card-bg);
  display: grid;
  gap: 0.55rem;
}

.shopee-card h3 {
  font-size: 0.95rem;
  font-weight: 700;
}

.shopee-card label {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  font-size: 0.86rem;
}

.shopee-card input {
  height: 36px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  padding: 0 0.65rem;
}

.opt-line {
  display: flex !important;
  flex-direction: row !important;
  align-items: center;
  gap: 0.45rem;
}

.switch-row {
  justify-content: space-between;
}

.opt-text {
  font-size: 0.9rem;
}

.switch {
  position: relative;
  width: 44px;
  height: 24px;
  display: inline-flex;
}

.switch-input {
  opacity: 0;
  width: 0;
  height: 0;
  position: absolute;
}

.switch-slider {
  position: absolute;
  inset: 0;
  background: var(--switch-off);
  border-radius: 999px;
  transition: 0.2s ease;
  cursor: pointer;
}

.switch-slider::before {
  content: '';
  position: absolute;
  width: 18px;
  height: 18px;
  left: 3px;
  top: 3px;
  background: #fff;
  border-radius: 50%;
  transition: 0.2s ease;
}

.switch-input:checked + .switch-slider {
  background: #f2742b;
}

.switch-input:checked + .switch-slider::before {
  transform: translateX(20px);
}

.shopee-result p {
  margin: 0;
}

.calc-line {
  font-size: 0.82rem;
  color: var(--muted);
}

.kpi-card {
  border: 1px solid var(--card-border);
  border-radius: 8px;
  padding: 0.75rem;
  background: var(--card-bg);
}

.kpi-card span {
  display: block;
  font-size: 0.8rem;
  color: var(--muted);
}

.kpi-card strong {
  display: block;
  margin-top: 0.2rem;
  font-size: 1.1rem;
}

.erro {
  margin-bottom: 0.75rem;
  color: var(--error);
}

.table-wrap {
  overflow: auto;
  max-height: 64vh;
  border: 1px solid var(--card-border);
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
  border-bottom: 1px solid var(--line);
  text-align: left;
  vertical-align: middle;
  background: var(--panel-bg);
}

thead th {
  position: sticky;
  top: 0;
  z-index: 2;
  background: var(--header-bg);
  color: var(--header-fg);
  font-weight: 700;
}

.sticky-col {
  position: sticky;
  left: 0;
  z-index: 1;
  background: var(--panel-bg);
}

thead .sticky-col {
  z-index: 3;
  background: var(--header-bg);
  color: var(--header-fg);
}

.sort-btn {
  all: unset;
  cursor: pointer;
  font-weight: 700;
}

tfoot td {
  background: var(--card-bg);
  font-weight: 700;
}

.input-custo {
  width: 115px;
  height: 32px;
  border: 1px solid var(--input-border);
  border-radius: 6px;
  padding: 0 0.5rem;
}

.frete-api {
  margin-top: 0.2rem;
  font-size: 0.75rem;
  color: var(--muted);
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
    margin: 0.75rem 3vw;
    padding: 1rem 0.85rem;
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
    border: 1px solid var(--card-border);
    border-radius: 8px;
    background: var(--panel-bg);
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

  .shopee-grid {
    grid-template-columns: 1fr;
  }
}
</style>
