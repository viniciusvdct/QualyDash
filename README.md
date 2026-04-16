# PROMPT DE CONTINUIDADE — Quality·Dash v3
## Dashboard de Indicadores de Qualidade (Defeitos × Produção)

---

## CONTEXTO DO PROJETO

Sou Vinícius, trabalho na área de qualidade/operações. Gerencio indicadores de defeitos em vasilhames de GLP (botijões P-13, P-20, P-45) usando dados extraídos do SAP (módulo QM — Quality Management). O dashboard é um arquivo HTML único que roda no navegador, sem servidor, e fica numa **pasta compartilhada na rede** onde várias máquinas acessam.

---

## ARQUITETURA TÉCNICA

### Arquivo único: `dashboard_defeitos.html`
- **HTML + CSS + JavaScript inline** (zero dependências locais)
- **CDNs utilizados:**
  - SheetJS (xlsx.full.min.js) — leitura de .xlsx no navegador
  - Chart.js 4.4.1 — gráficos
  - html2canvas 1.4.1 — captura de tela para PDF
  - jsPDF 2.5.1 — geração de PDF
  - Google Fonts: Fraunces (títulos), Inter Tight (corpo), JetBrains Mono (números/códigos)

### Arquivo de configuração: `config.json`
- Salvo na mesma pasta do HTML
- Carregado automaticamente via `fetch('config.json')` ao abrir
- Exportado/importado manualmente pela aba Ajustes
- Contém: meta PPM, metas mensais, casas decimais, tipos de defeito, locais de defeito, materiais (com flag de visibilidade), clientes/consultores

### Arquivo separado: `clientes_consultores.json`
- Export/import independente só da tabela de clientes e consultores
- Útil para atualizar mapeamento sem mexer no resto da config

---

## FONTES DE DADOS (SAP)

### BASE_DEFEITOS (arquivo Defeitos.xlsx)
Colunas:
| Coluna | Uso |
|--------|-----|
| Exceção | ignorado |
| Centro para material | ignorado (sempre 1001) |
| **Cliente** | código do cliente (ex: 21876) — mapeado via tabela auxiliar |
| **Nota** | número da notificação QM — **parte da chave de dedup** |
| **Item** | item dentro da nota — **parte da chave de dedup** |
| **Material** | código SAP (200036=P-13, 200038=P-20, 200039=P-45) |
| **Código do dano** | tipo de defeito (D001 a D025) |
| **Code parte do objeto** | local do defeito (L001 a L025) |
| Peso individual | não usado atualmente |
| Qtd.defeituosa externa | não usado no cálculo principal |
| **Qtd.defeituosa interna** | ← ESTA é a quantidade de defeitos usada no PPM |
| **Data da nota** | data de abertura |
| **Encerram.por data** | data de fechamento ← **USADA COMO DATA PRINCIPAL** (fallback: Data da nota) |

**Chave de deduplicação:** `Nota + Item`

### BASE_PRODUCAO (arquivo Produção.xlsx)
Colunas:
| Coluna | Uso |
|--------|-----|
| Centro | ignorado |
| **Material** | código SAP |
| **Qtd. UM registro** | quantidade produzida |
| **Data de lançamento** | data principal |
| Data do documento | fallback |
| Texto breve material | ignorado |
| **Tipo de movimento** | 131 e 132 = **ambos são entrada de produção** (somar, NÃO subtrair) |

**Chave de deduplicação:** `Material + Data + Qtd + Tipo de movimento`

---

## REGRAS DE NEGÓCIO

### Cálculo do indicador
```
PPM  = (defeitos_internos ÷ produção) × 1.000.000
‰    = (defeitos_internos ÷ produção) × 1.000
%    = (defeitos_internos ÷ produção) × 100
cp100 = (defeitos_internos ÷ produção) × 100
```

### Meta
- **Meta global padrão:** 1541 (PPM)
- **Meta personalizável por mês/ano** (grid editável na aba Ajustes)
- Se não houver meta mensal definida, usa a global

### Produção
- Tipo de movimento **131 = entrada**
- Tipo de movimento **132 = entrada** (NÃO é estorno, é entrada também)
- Produção total = soma de 131 + 132

### Defeitos
- Usa **Qtd.defeituosa interna** (não externa)
- Data de referência = **Encerram.por data** (fechamento); se vazio, cai para Data da nota (abertura)
- Tempo de encerramento = dias entre abertura e fechamento

### Clientes não mapeados
- Mostrar o **código do cliente** entre colchetes: `[21876]`

### Deduplicação incremental
- Pode carregar múltiplos arquivos de cada vez
- Novas linhas são adicionadas; duplicatas (mesma chave) são ignoradas
- Permite atualizar 2x por semana sem reprocessar tudo

---

## ESTRUTURA DO DASHBOARD (8 ABAS)

### 01 — Visão Geral
- KPIs: PPM geral (com badge dentro/acima da meta), defeitos internos, produção, clientes ativos, consultores, tempo médio encerramento
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
- Como carregar dados
- Como usar filtros
- Descrição de cada aba
- Glossário completo (PPM, ‰, %, cp100, Pareto, YoY, Média móvel, Heatmap, Nota, Tipo 131/132)
- Como compartilhar config.json entre máquinas
- Como exportar PDF
- Como funciona a atualização incremental

### ⚙ — Ajustes
- Meta global e casas decimais
- Exportar/Importar config.json
- Restaurar padrão
- **Grid de meta por mês/ano** (com anos futuros automáticos)
- **Materiais** (todos os códigos dos dados expostos, checkbox para ocultar)
- **Tipos de defeito** (busca, edição inline, adicionar/remover)
- **Locais de defeito** (busca, edição inline, adicionar/remover)
- **Clientes & Consultores** (todos os códigos dos dados expostos, busca, edição inline, export/import separado)

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
| **Limpar filtros** | Botão | Reseta tudo + todos os anos |
| **📄 PDF** | Botão | Gera PDF com TODAS as 6 abas |

**Os filtros NÃO afetam as abas Guia e Ajustes.**

---

## TABELAS AUXILIARES EMBUTIDAS

### Tipos de Defeito (D001–D025)
D001 PINTURA, D002 AMASSAMENTO, D003 ABOLHADURA, D004 SEM DEFEITO, D005 EXPOSIÇÃO AO FOGO, D006 CORROSÃO, D007 TARA ILEGÍVEL, D008 FALTA DE TARA, D009 ERRO DE TARA, D010 FALTA DE GRAVAÇÕES, D011 REQUALIFICAÇÃO 15 ANOS, D012 REQUALIFICAÇÃO 10 ANOS, D013 VAZAMENTO, D014 FALTA DE LACRE, D015 FALTA DE RÓTULO INFORM., D016 PESO, D017 FALTA DE O'RING, D018 OM'S, D019 VÁLVULA BAIXA, D020 RESÍDUOS, D021 DECANTAÇÃO, D022 ACESSÓRIO DANIFICADO, D023 EXCESSO DE AR/PRESSÃO, D024 O'RING DANIFICADO, D025 VASILHAME AUTUADO

### Locais do Defeito (L001–L025)
L001 VASILHAME, L002 CORPO, L003 ALÇA, L004 BASE, L005 CALOTA SUPERIOR, L006 CALOTA INFERIOR, L007 FLANGE, L008 SOLDA LONGITUDINAL, L009 SOLDA CIRCUNFERENCIAL, L010 FUNDO, L011 VÁLVULA ROSCA INTERNA, L012 VÁLVULA ROSCA EXTERNA, L013 DISP. SEGURANÇA/LIGA, L014 DISP. SEGUR/ROSCA EXT, L015 RÓTULO, L016 NÃO APLICÁVEL, L017 VÁLVULA DEF. INTERNO, L018 VÁLVULA SEXTAVADO, L019 MEDIDOR/DEF. INTERNO, L020 MEDIDOR/ROSCA EXT, L021 V. MAX ENCH/DEF INT, L022 V. MAX ENCH/ROSCA EXT, L023 V. SERVIÇO/DEF INT, L024 V. SERVIÇO/ROSCA EXT, L025 ENGATE MACHO

### Materiais
| Código | Nome | Vasilhame | Meta PPM |
|--------|------|-----------|----------|
| 200036 | P-13 | Botijão 13kg | 1541 |
| 200038 | P-20 | Botijão 20kg | 1541 |
| 200039 | P-45 | Botijão 45kg | 1541 |

### Clientes & Consultores
~170 clientes mapeados com código SAP → nome → consultor responsável.
6 consultores principais: CLAUDINEI MEDEIROS ANUNCIATO, FABIANO FONSECA DE CARVALHO, MARCILENE DE ASSIS, NATHALIA GUTIERRE LEITE SOARDI NOGUEIRA, PATRICIA ESPINDOLA FERREIRA, THIAGO DA SILVA ONARI, CALL CENTER.
(Lista completa embutida no HTML e no config.json)

---

## DESIGN / ESTÉTICA

- **Tema escuro** (fundo #0d1015, painéis #161a23)
- Paleta: laranja (#ff5b3a) como accent, amarelo (#ffd166) para meta/warning, verde (#3ddc97) bom, vermelho (#ff5b6e) ruim, azul (#6cb6ff) info, roxo (#b07cff) locais
- Tipografia: Fraunces para títulos/KPIs, Inter Tight corpo, JetBrains Mono para números
- Cards com borda lateral colorida por tipo de KPI
- Cards "featured" com borda laranja (combinação Tipo×Local)
- Gap de 18px entre cards/grids
- Toast notifications para feedback
- Loading overlay com spinner para operações demoradas

---

## PENDÊNCIAS / MELHORIAS FUTURAS SUGERIDAS

1. **Servidor local (Node.js/PHP)** — para salvar config.json automaticamente sem export manual
2. **Filtro por período customizado** (data início/fim) em vez de apenas ano+mês
3. **Drill-down nos gráficos** — clicar num bar chart e filtrar pelo valor clicado
4. **Exportar dados filtrados para Excel** — botão que gera .xlsx dos dados visíveis
5. **Dashboard de tempo de encerramento** — aba dedicada com SLA, histograma de dias, violações
6. **Comparativo entre consultores** — mesmos KPIs lado a lado
7. **Alertas** — destaque automático quando PPM ultrapassa meta em X meses consecutivos
8. **Modo apresentação** — fullscreen otimizado para projetor
9. **PowerPoint export** — além do PDF, gerar .pptx com slides por aba
10. **Autenticação simples** — para controlar quem pode editar vs. só visualizar

---

## COMO USAR ESTE PROMPT

Cole este documento inteiro no início de um novo chat e diga algo como:

> "Aqui está o contexto completo do meu projeto Quality·Dash. Preciso fazer as seguintes alterações: [lista de mudanças]"

Ou para continuar de onde paramos:

> "Aqui está o prompt do meu dashboard. Por favor leia e me diga que está pronto para continuar."

Se for enviar o HTML atual junto, diga:

> "Estou enviando o HTML atual do dashboard + este prompt de contexto. Preciso de [mudança específica]."

---

*Gerado em: Abril/2026 — versão 3 do Quality·Dash*
