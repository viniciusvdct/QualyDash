# PROMPT DE CONTINUIDADE — Quality·Dash v4
## Dashboard de Indicadores de Qualidade (Defeitos × Produção)

---

## CONTEXTO DO PROJETO

Projeto do Vinícius, área de qualidade/operações. Gerencia indicadores de defeitos em vasilhames de GLP (botijões P-13, P-20, P-45) usando dados extraídos do SAP (módulo QM — Quality Management). O dashboard é um arquivo HTML único que roda no navegador, sem servidor, e fica numa **pasta compartilhada na rede** onde várias máquinas acessam sem privilégio de administrador.

- **Base histórica consolidada:** 2017 a 2025
- **Base em andamento:** 2026 (atualização 2× por semana)
- **Vida útil longa:** arquitetura deve se sustentar sem refatorações grandes

---

## ARQUITETURA TÉCNICA

### Arquivo principal: `dashboard_defeitos.html`
- **HTML + CSS + JavaScript inline** (zero dependências locais, arquivo único)
- **CDNs utilizados:**
  - SheetJS `xlsx.full.min.js` — leitura de .xlsx no navegador
  - Chart.js 4.4.1 — gráficos
  - html2canvas 1.4.1 — captura de tela para PDF
  - jsPDF 2.5.1 — geração de PDF
  - Google Fonts: Fraunces (títulos/KPIs), Inter Tight (corpo), JetBrains Mono (números/códigos)

### Arquivo de configuração: `config.json`
- Salvo na mesma pasta do HTML no servidor compartilhado
- Carregado automaticamente via `fetch('config.json')` ao abrir o dashboard
- Exportado/importado manualmente pela aba ⚙ Ajustes
- Contém: meta PPM global, metas mensais por `AAAA-MM`, casas decimais, tipos de defeito, locais de defeito, materiais (com flag de visibilidade), clientes/consultores

### Arquivo auxiliar: `clientes_consultores.json`
- Export/import independente — só da tabela de clientes e consultores
- Útil para atualizar mapeamento sem mexer no restante da config

---

## VERSÃO ATUAL

| Arquivo | Versão | Data |
|---------|--------|------|
| `dashboard_defeitos.html` | **v7** | Abr/2026 |
| README (este) | **v4** | Abr/2026 |

### Histórico de versões relevante
- **v6 → v7:** Correção crítica de bug na deduplicação de produção (ver seção abaixo)

---

## ⚠️ CORREÇÃO CRÍTICA — BUG DE DEDUPLICAÇÃO DE PRODUÇÃO (v6 → v7)

### O problema (v6)
A chave de deduplicação de registros de produção era:

```
Material + Data + Qtd + Tipo de movimento
```

Essa chave é **semanticamente incorreta**. O SAP gera múltiplos lançamentos distintos com os mesmos valores — por exemplo, duas ordens de produção de 1.000 unidades do mesmo material no mesmo dia, na mesma planta. O dashboard interpretava o segundo lançamento como duplicata e o **descartava silenciosamente**.

### Impacto medido (dados 2026 Q1, P-13)
| Mês | Planilha SAP (correto) | Dashboard v6 (bug) | Perda |
|-----|----------------------|-------------------|-------|
| Jan | 185.542 | 176.532 | −9.010 |
| Fev | 179.557 | 175.863 | −3.694 |
| Mar | 200.975 | 201.309 | variável |

O mesmo problema existe nos dados históricos 2017–2025. Ao carregar a base histórica pela primeira vez na v7, os números de produção irão subir — isso é o comportamento **correto**.

### A solução (v7)
Deduplicação migrou de **nível de linha** para **nível de arquivo**:

```javascript
// Nova função — hash SHA-256 do conteúdo binário do arquivo
async function hashFile(file) {
  const buf = await file.arrayBuffer();
  const hashBuf = await crypto.subtle.digest('SHA-256', buf);
  return Array.from(new Uint8Array(hashBuf))
    .map(b => b.toString(16).padStart(2, '0')).join('');
}
```

**Comportamento novo:**
- Se o usuário carrega o mesmo `.xlsx` duas vezes → o segundo é ignorado (aviso em toast)
- Se o usuário carrega arquivos distintos que contenham linhas parecidas → **todos os registros de ambos os arquivos são preservados**
- Todas as ordens de produção legítimas dentro de um arquivo são carregadas integralmente

**State alterado:**
```js
state = {
  // ...
  defKeys: new Set(),          // dedup de defeitos (chave: Nota+Item) — mantido
  prodFileHashes: new Set(),   // dedup de produção — NOVO (por arquivo)
  defFileHashes: new Set(),    // dedup de defeitos — NOVO (por arquivo, camada extra)
  // prodKeys: removido
}
```

### Divergência residual de ~2.996 em fevereiro/2026
Após a correção, Janeiro e Março ficam a 1–2 unidades da planilha SAP (variação de precisão numérica do export). Fevereiro ainda difere ~2.996. **Não é bug do dashboard** — investigar no SAP se há registros com `Data de lançamento` em fevereiro mas `Data do documento` em outro mês, ou vice-versa. Confirmar qual coluna o SUMIF da planilha-verdade usa como critério de data.

---

## FONTES DE DADOS (SAP)

### BASE_DEFEITOS (arquivo `*Defeitos.xlsx`)

| Coluna | Uso |
|--------|-----|
| Exceção | ignorado |
| Centro para material | ignorado (sempre 1001) |
| **Cliente** | código do cliente (ex: `21876`) — mapeado via tabela auxiliar |
| **Nota** | número da notificação QM — **parte da chave de dedup** |
| **Item** | item dentro da nota — **parte da chave de dedup** |
| **Material** | código SAP (200036=P-13, 200038=P-20, 200039=P-45) |
| **Código do dano** | tipo de defeito (D001 a D025) |
| **Code parte do objeto** | local do defeito (L001 a L025) |
| Peso individual | não usado |
| Qtd.defeituosa externa | não usado no cálculo principal |
| **Qtd.defeituosa interna** | quantidade de defeitos usada no PPM |
| **Data da nota** | data de abertura |
| **Encerram.por data** | data de fechamento ← **DATA PRINCIPAL** (fallback: Data da nota) |

**Chave de deduplicação de linha:** `Nota + Item`
**Chave de deduplicação de arquivo:** SHA-256 do conteúdo binário

### BASE_PRODUCAO (arquivo `*Produção.xlsx`)

| Coluna | Uso |
|--------|-----|
| Centro | ignorado |
| **Material** | código SAP |
| **Qtd. UM registro** | quantidade produzida (positiva em 131, negativa em 132) |
| **Data de lançamento** | data principal |
| Data do documento | fallback se Data de lançamento ausente |
| Texto breve material | ignorado |
| **Tipo de movimento** | 131 e 132 |

**Tipo de movimento:**
- **131** = lançamento de produção (quantidade positiva)
- **132** = estorno de produção (quantidade negativa no campo do SAP)
- **Produção líquida** = soma aritmética de todos os registros 131 + 132 (os 132 já chegam com sinal negativo, portanto a soma líquida é automaticamente correta)

**Chave de deduplicação:** por arquivo (SHA-256) — **não há dedup por linha**

---

## REGRAS DE NEGÓCIO

### Cálculo do indicador
```
PPM   = (defeitos_internos ÷ produção) × 1.000.000
‰     = (defeitos_internos ÷ produção) × 1.000
%     = (defeitos_internos ÷ produção) × 100
cp100 = (defeitos_internos ÷ produção) × 100
```

### Meta
- **Meta global padrão:** 1541 (PPM)
- **Meta personalizável por mês/ano** via grid na aba ⚙ Ajustes (chave: `"AAAA-MM"`)
- Se não houver meta mensal definida para o período, usa a global

### Defeitos
- Usa `Qtd.defeituosa interna` (não externa)
- Data de referência = `Encerram.por data` (fechamento); se vazio, usa `Data da nota` (abertura)
- Tempo de encerramento = dias entre abertura e fechamento

### Clientes não mapeados
- Exibe o código do cliente entre colchetes: `[21876]`

### Deduplicação incremental
- Permite carregar múltiplos arquivos de cada vez (ex: base histórica + parcial 2026)
- Para defeitos: dedup por `Nota+Item` (linha) + SHA-256 (arquivo)
- Para produção: dedup apenas por SHA-256 (arquivo) — todos os registros dentro do arquivo são preservados
- Permite atualização 2× por semana sem reprocessar tudo — basta carregar o arquivo novo

---

## ESTRUTURA DO DASHBOARD (8 ABAS)

| Aba | ID | Função de render |
|-----|----|-----------------|
| 01 Visão Geral | `page-overview` | `renderOverview()` |
| 02 Defeitos | `page-defects` | `renderDefects()` |
| 03 Clientes | `page-clients` | `renderClients()` |
| 04 Consultores | `page-consultants` | `renderConsultants()` |
| 05 Tipos & Locais | `page-types` | `renderTypes()` |
| 06 Tendências | `page-trends` | `renderTrends()` |
| 📖 Guia | `page-guide` | `renderGuide()` |
| ⚙ Ajustes | `page-settings` | `renderSettings()` |

### 01 — Visão Geral
- KPIs: PPM geral (badge dentro/acima da meta), defeitos internos, produção, clientes ativos, consultores, tempo médio de encerramento
- **★ Top Combinações Tipo × Local** (card em destaque, borda laranja)
- PPM por Ano vs. Meta (barras + linha de meta)
- Pareto tipos de defeito (barras + linha % acumulado)
- Top 10 clientes (horizontal)
- Defeitos por consultor (donut)
- Local do defeito (horizontal)
- Evolução mensal defeitos vs. produção (duas escalas Y)
- PPM por material vs. meta

### 02 — Defeitos
- PPM mensal vs. meta
- Defeitos por ano
- Comparativo mensal por ano (linhas sobrepostas — útil com multi-seleção de anos)
- Defeitos por dia da semana
- Defeitos por material
- Heatmap ano × mês (cores de intensidade)

### 03 — Clientes
- KPIs: total clientes, top 1, % concentração top 5 e top 10
- Top 20 clientes (horizontal)
- Tendência top 5 clientes (linhas por mês)
- Ranking completo com barras de proporção, consultor, notas, defeitos, tempo

### 04 — Consultores
- KPIs: ativos, maior carteira, média
- Defeitos por consultor (horizontal com cores)
- Tipos de defeito empilhados por consultor (top 8 tipos)
- Tabela de performance

### 05 — Tipos & Locais
- **★ Combinação Tipo + Local** (card destaque, gráfico + tabela lado a lado, top 50)
- Ranking tipos (horizontal)
- Ranking locais (horizontal)
- Matriz heatmap Tipo × Local (top 10 × top 8)

### 06 — Tendências
- PPM mensal com média móvel 3 meses
- Variação YoY (barras %)
- Tendência top 5 tipos de defeito (linhas)
- Defeitos acumulados (curva)

### 📖 — Guia
- Como carregar dados, usar filtros, descrição de cada aba
- Glossário (PPM, ‰, %, cp100, Pareto, YoY, Média móvel, Heatmap, Nota, Tipo 131/132)
- Como compartilhar `config.json` entre máquinas
- Como exportar PDF
- Como funciona a atualização incremental

### ⚙ — Ajustes
- Meta global e casas decimais
- Exportar/Importar `config.json` | Restaurar padrão
- Grid de meta por mês/ano (com anos futuros automáticos)
- Materiais (todos os códigos dos dados, checkbox para ocultar)
- Tipos de defeito (busca, edição inline, adicionar/remover)
- Locais de defeito (busca, edição inline, adicionar/remover)
- Clientes & Consultores (busca, edição inline, export/import separado)
- 🎨 Aparência (toggle dark/light mode)

---

## STATE GLOBAL

```js
state = {
  defeitos: [],          // dados processados de defeitos
  producao: [],          // dados processados de produção
  defRaw: [],
  prodRaw: [],
  defKeys: new Set(),        // chaves Nota+Item já carregadas
  prodFileHashes: new Set(), // hashes SHA-256 de arquivos de produção carregados
  defFileHashes: new Set(),  // hashes SHA-256 de arquivos de defeitos carregados
  filtros: { anos:[], mes:"", material:"", consultor:"", cliente:"" },
  currentPage: 'overview',
  mode: 'ppm',           // 'ppm' | 'permil' | 'pct' | 'cp100'
  charts: {}             // instâncias Chart.js ativas
}
```

## CONFIG (persistido via config.json)

```js
CONFIG = {
  meta_ppm: 1541,
  decimais_ppm: 0,
  metas_mensal: { "2026-01": 1200, ... },  // chave sempre "AAAA-MM"
  clientes: { "1234": { nome:"...", consultor:"..." } },
  materiais: { "GLP13": { nome:"...", vasilhame:"...", visivel:true } },
  tipos_defeito: { "D001": "PINTURA" },
  locais_defeito: { "L001": "VASILHAME" }
}
```

---

## FILTROS GLOBAIS (barra superior)

| Filtro | Tipo | Comportamento |
|--------|------|---------------|
| **Anos** | Multi-seleção (chips clicáveis) | Todos selecionados por padrão |
| **Mês** | Dropdown | "Todos" ou mês específico |
| **Material** | Dropdown | Só materiais com `visivel: true` |
| **Consultor** | Dropdown | Lista dos dados |
| **Cliente** | Dropdown | Lista dos dados com nome |
| **Indicador** | Toggle 4 opções | PPM / ‰ / % / cp100 |
| **Limpar filtros** | Botão | Reseta tudo |
| **📄 PDF** | Botão | Gera PDF de todas as 6 abas |

> Os filtros **não afetam** as abas Guia e Ajustes.

---

## HELPERS JS RELEVANTES

```js
fmtMesLabel("2026-02")               // → "Fev/2026"
makeParetoChart(id, labels, values, color) // Pareto com linha 80%
fmtNum(n)                             // número com ponto milhar pt-BR
fmtPPM(n)                             // formata conforme state.mode
aggSum(arr, keyFn, valFn)             // Map de soma agrupada
applyFilters(arr, isProd)             // aplica state.filtros
hashFile(file)                        // Promise<string> — SHA-256 hex do arquivo
```

## FLUXO DE CARREGAMENTO

1. Usuário sobe `.xlsx` → `loadDefFiles()` ou `loadProdFiles()`
2. Hash SHA-256 do arquivo é calculado; se já existe em `prodFileHashes`/`defFileHashes`, o arquivo é ignorado com aviso
3. Se arquivo novo: todos os registros válidos são inseridos (produção sem dedup de linha; defeitos com dedup por `Nota+Item`)
4. `afterLoad()` → ativa `page-${state.currentPage}` + **double rAF** + `renderAll()`
   > ⚠️ O double `requestAnimationFrame` é essencial — sem ele os charts ficam com altura 0px
5. `renderAll()` → `updateStatus()` + `renderPage(state.currentPage)`

## PDF (btnPDF)
- Capa com título + subtítulo + data + filtros aplicados
- 6 páginas de conteúdo (html2canvas scale:2, formato PNG)
- Rodapé em cada página: nome da aba · data · número de página
- Arquivo salvo como `QualityDash_AAAA-MM-DD.pdf`

## Padrão Pareto (makeParetoChart)
- Barras horizontais coloridas até 80% acumulado (acima: cinza `#5b6478`)
- Linha acumulada amarela (`#ffd166`) com eixo Y2 (0–100%)
- Linha de referência tracejada nos 80% via `plugin80`
- Tooltip: quantidade + % individual + % acumulado

## Ajustes — mapeamento de seções

| Seção | IDs-chave | Ação salva |
|-------|-----------|------------|
| Meta e indicador | `cfgMeta`, `cfgDec`, `modeToggle` | `btnSaveMeta` |
| Metas mensais | inputs `[data-meta-key]` | `btnSaveMetas` |
| Aparência | `btnDarkMode` / `btnLightMode` | imediato |
| Clientes | `listCli`, `searchCli` | inline (change) |
| Materiais | `listMat` | `btnConfirmMat` (confirma + renderAll) |
| Tipos | `listTipos` | inline (change) |
| Locais | `listLocais` | inline (change) |
| Export/Import | `btnExport` / `fileImport` | JSON config.json |

> `bindSettingsEvents()` é sempre chamada no final de `renderSettings()`

---

## DESIGN / ESTÉTICA

| Elemento | Valor |
|----------|-------|
| Fundo dark | `#0d1015` |
| Painéis dark | `#161a23` |
| Accent principal | `#ff5b3a` (laranja) |
| Meta/warning | `#ffd166` (amarelo) |
| Bom (abaixo da meta) | `#3ddc97` (verde) |
| Ruim (acima da meta) | `#ff5b6e` (vermelho) |
| Info | `#6cb6ff` (azul) |
| Locais | `#b07cff` (roxo) |
| Cards featured | borda laranja (`#ff5b3a`) |
| Gap padrão | 18px |

Light mode: `body[data-theme="light"]` com overrides de variáveis CSS.

---

## TABELAS AUXILIARES EMBUTIDAS

### Tipos de Defeito (D001–D025)
D001 PINTURA, D002 AMASSAMENTO, D003 ABOLHADURA, D004 SEM DEFEITO, D005 EXPOSIÇÃO AO FOGO, D006 CORROSÃO, D007 TARA ILEGÍVEL, D008 FALTA DE TARA, D009 ERRO DE TARA, D010 FALTA DE GRAVAÇÕES, D011 REQUALIFICAÇÃO 15 ANOS, D012 REQUALIFICAÇÃO 10 ANOS, D013 VAZAMENTO, D014 FALTA DE LACRE, D015 FALTA DE RÓTULO INFORM., D016 PESO, D017 FALTA DE O'RING, D018 OM'S, D019 VÁLVULA BAIXA, D020 RESÍDUOS, D021 DECANTAÇÃO, D022 ACESSÓRIO DANIFICADO, D023 EXCESSO DE AR/PRESSÃO, D024 O'RING DANIFICADO, D025 VASILHAME AUTUADO

### Locais do Defeito (L001–L025)
L001 VASILHAME, L002 CORPO, L003 ALÇA, L004 BASE, L005 CALOTA SUPERIOR, L006 CALOTA INFERIOR, L007 FLANGE, L008 SOLDA LONGITUDINAL, L009 SOLDA CIRCUNFERENCIAL, L010 FUNDO, L011 VÁLVULA ROSCA INTERNA, L012 VÁLVULA ROSCA EXTERNA, L013 DISP. SEGURANÇA/LIGA, L014 DISP. SEGUR/ROSCA EXT, L015 RÓTULO, L016 NÃO APLICÁVEL, L017 VÁLVULA DEF. INTERNO, L018 VÁLVULA SEXTAVADO, L019 MEDIDOR/DEF. INTERNO, L020 MEDIDOR/ROSCA EXT, L021 V. MAX ENCH/DEF INT, L022 V. MAX ENCH/ROSCA EXT, L023 V. SERVIÇO/DEF INT, L024 V. SERVIÇO/ROSCA EXT, L025 ENGATE MACHO

### Materiais
| Código SAP | Nome | Vasilhame |
|------------|------|-----------|
| 200036 | P-13 | Botijão 13 kg |
| 200038 | P-20 | Botijão 20 kg |
| 200039 | P-45 | Botijão 45 kg |

### Clientes & Consultores
~170 clientes mapeados: código SAP → nome → consultor responsável.
6 consultores principais: CLAUDINEI MEDEIROS ANUNCIATO, FABIANO FONSECA DE CARVALHO, MARCILENE DE ASSIS, NATHALIA GUTIERRE LEITE SOARDI NOGUEIRA, PATRICIA ESPINDOLA FERREIRA, THIAGO DA SILVA ONARI, CALL CENTER.
Lista completa embutida no HTML e no `config.json`.

---

## PENDÊNCIAS / BACKLOG SUGERIDO

| Prioridade | Item |
|-----------|------|
| 🔴 Alta | Investigar divergência residual de ~2.996 unidades em Fev/2026 (confirmar coluna de data usada pelo SUMIF da planilha SAP) |
| 🟡 Média | Filtro por período customizado (data início/fim) além de ano+mês |
| 🟡 Média | Export de dados filtrados para `.xlsx` direto do dashboard |
| 🟡 Média | Drill-down nos gráficos (clicar numa barra e filtrar pelo valor) |
| 🟢 Baixa | Aba de comparação: período atual vs. período anterior |
| 🟢 Baixa | Salvar preferência de tema (dark/light) em localStorage |
| 🟢 Baixa | Alertas automáticos quando PPM ultrapassa meta em X meses consecutivos |
| 🟢 Baixa | Heatmap de defeitos por dia da semana × hora (se dado disponível no SAP) |
| 🟢 Baixa | Modo apresentação (fullscreen otimizado para projetor) |
| 🟢 Baixa | Servidor local leve (Node.js) para salvar config.json automaticamente |

---

## COMO USAR ESTE PROMPT EM UM NOVO CHAT

Cole este documento no início do chat junto com o HTML atual e diga:

> "Aqui está o contexto completo do Quality·Dash v7 e o HTML atual. Preciso de: [mudança]."

Para confirmação de leitura:

> "Leia o README e me diga que está pronto para continuar."

---

## ATENÇÃO AO EDITAR O CÓDIGO

- Nunca modificar as **chaves internas** `"AAAA-MM"` usadas em `CONFIG.metas_mensal`
- `state.charts` deve ter `destroy()` chamado antes de re-renderizar com novo tema
- Todas as funções `render*` são idempotentes — podem ser chamadas a qualquer momento
- `bindSettingsEvents()` sempre ao final de `renderSettings()`
- O **double `requestAnimationFrame`** em `afterLoad()` é obrigatório para os charts renderizarem com tamanho correto
- `processProdRow()` **não gera mais chave de dedup** — não reintroduzir chave por conteúdo de linha

---

*Gerado em: Abril/2026 — Quality·Dash v7 / README v4*
