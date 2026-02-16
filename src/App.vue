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

const totalMargem = computed(() =>
  anuncios.value.reduce((soma, anuncio) => soma + (calcularMargemContribuicao(anuncio) || 0), 0)
)

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

async function parseErroResposta(resposta, padrao) {
  try {
    const body = await resposta.json()
    return body?.message || body?.error || padrao
  } catch {
    return padrao
  }
}

async function mlFetch(url) {
  const headers = {}
  if (accessToken.value.trim()) {
    headers.Authorization = `Bearer ${accessToken.value.trim()}`
  }

  return fetch(url, { headers })
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

  const resposta = await mlFetch(`/ml/sites/${siteId.value}/listing_prices?${params.toString()}`)
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
  if (typeof item.shipping_cost === 'number') {
    return {
      envioSelecionado: item.shipping_cost,
      opcoes: []
    }
  }

  if (item.shipping?.free_shipping) {
    return {
      envioSelecionado: 0,
      opcoes: []
    }
  }

  // Itens fechados ou sem estoque costumam retornar 404 em shipping_options.
  if (item.status !== 'active' || Number(item.available_quantity || 0) <= 0) {
    return {
      envioSelecionado: null,
      opcoes: []
    }
  }

  if (!cep.value) {
    return {
      envioSelecionado: null,
      opcoes: []
    }
  }

  const resposta = await mlFetch(`/ml/items/${item.id}/shipping_options?zip_code=${cep.value}`)
  if (!resposta.ok) {
    return {
      envioSelecionado: null,
      opcoes: []
    }
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

  const opcaoSelecionada =
    opcoes.find((opcao) => opcao.display === 'recommended') ||
    opcoes[0] ||
    null
  const envioSelecionado = opcaoSelecionada?.cost ?? opcaoSelecionada?.listCost ?? null

  return {
    envioSelecionado,
    opcoes
  }
}

async function buscarItensAutenticado() {
  const respostaItens = await mlFetch(`/ml/items?ids=${IDS_TESTE.join(',')}`)
  if (!respostaItens.ok) {
    const detalhe = await parseErroResposta(
      respostaItens,
      'Nao foi possivel carregar detalhes dos anuncios.'
    )
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
        <input
          v-model="accessToken"
          type="password"
          autocomplete="off"
          placeholder="APP_USR-..."
        />
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

    <p v-if="erro" class="erro">{{ erro }}</p>

    <div class="table-wrap">
      <table>
        <thead>
          <tr>
            <th>ID</th>
            <th>Titulo</th>
            <th>Tipo</th>
            <th>Valor da venda</th>
            <th>Tarifa ML</th>
            <th>Gross amount</th>
            <th>Fixed fee</th>
            <th>Percentage fee</th>
            <th>Envios</th>
            <th>Fretes (opcoes)</th>
            <th>Total liquido</th>
            <th>Custo produto</th>
            <th>Imposto produto</th>
            <th>Margem contribuicao</th>
            <th>% Margem</th>
          </tr>
        </thead>
        <tbody>
          <tr v-if="!carregando && anuncios.length === 0">
            <td colspan="15">Nenhum anuncio carregado. Informe os dados e clique em buscar.</td>
          </tr>
          <tr v-for="anuncio in anuncios" :key="anuncio.id">
            <td>{{ anuncio.id }}</td>
            <td>{{ anuncio.titulo }}</td>
            <td>{{ anuncio.tipo }}</td>
            <td>{{ formatarMoeda(anuncio.valorVenda) }}</td>
            <td>{{ formatarMoeda(anuncio.tarifaVenda) }}</td>
            <td>{{ formatarMoeda(anuncio.grossAmount) }}</td>
            <td>{{ formatarMoeda(anuncio.fixedFee) }}</td>
            <td>{{ formatarPercentual(anuncio.percentageFee) }}</td>
            <td>{{ formatarMoeda(anuncio.envios) }}</td>
            <td>
              <div v-if="anuncio.freteOpcoes?.length" class="fretes-lista">
                <div v-for="(opcao, idx) in anuncio.freteOpcoes" :key="`${anuncio.id}-${idx}`">
                  {{ opcao.nome }} ({{ opcao.tipo || 'N/D' }}) - custo:
                  {{ formatarMoeda(opcao.cost) }}, lista: {{ formatarMoeda(opcao.listCost) }}, base:
                  {{ formatarMoeda(opcao.baseCost) }}
                </div>
              </div>
              <span v-else>N/D</span>
            </td>
            <td>{{ formatarMoeda(calcularTotalLiquido(anuncio)) }}</td>
            <td>
              <input
                class="input-custo"
                type="number"
                min="0"
                step="0.01"
                :value="obterCustoProduto(anuncio.id)"
                @input="atualizarCustoProduto(anuncio.id, $event.target.value)"
              />
            </td>
            <td>{{ formatarMoeda(calcularImpostoProduto(anuncio)) }}</td>
            <td class="destaque">{{ formatarMoeda(calcularMargemContribuicao(anuncio)) }}</td>
            <td class="destaque">{{ formatarPercentual(calcularMargemPercentual(anuncio)) }}</td>
          </tr>
        </tbody>
        <tfoot v-if="anuncios.length > 0">
          <tr>
            <td colspan="13">Total de margem de contribuicao</td>
            <td class="destaque">{{ formatarMoeda(totalMargem) }}</td>
            <td></td>
          </tr>
        </tfoot>
      </table>
    </div>
  </main>
</template>

<style scoped>
.container {
  max-width: 1280px;
  margin: 0 auto;
}

h1 {
  margin-bottom: 1rem;
  font-size: 1.5rem;
}

.filtros {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 0.75rem;
  margin-bottom: 1rem;
  align-items: end;
}

.filtros label {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
  font-size: 0.9rem;
}

.filtros input,
.filtros button {
  height: 38px;
  border: 1px solid #cfcfcf;
  border-radius: 6px;
  padding: 0 0.65rem;
}

.filtros button {
  background: #1b7a3e;
  color: #fff;
  font-weight: 600;
  cursor: pointer;
}

.filtros button:disabled {
  opacity: 0.65;
  cursor: not-allowed;
}

.erro {
  margin-bottom: 0.5rem;
  color: #b00020;
}

.table-wrap {
  overflow-x: auto;
  border: 1px solid #d9d9d9;
  border-radius: 8px;
}

table {
  width: 100%;
  border-collapse: collapse;
  min-width: 2100px;
}

th,
td {
  padding: 0.7rem;
  border-bottom: 1px solid #ececec;
  text-align: left;
  vertical-align: middle;
}

thead th {
  background: #e5e7eb;
  color: #111827;
  font-weight: 600;
}

tfoot td {
  background: #fafafa;
  font-weight: 700;
}

.input-custo {
  width: 120px;
  height: 32px;
  border: 1px solid #cfcfcf;
  border-radius: 6px;
  padding: 0 0.5rem;
}

.destaque {
  color: #0f8a3c;
  font-weight: 700;
}

.fretes-lista {
  min-width: 380px;
  font-size: 0.8rem;
  line-height: 1.35;
}
</style>
