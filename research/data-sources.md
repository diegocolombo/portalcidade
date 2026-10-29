# Cascavel (PR) public data sources: what exists and how to fetch it

Municipality: **Cascavel, Paraná**. IBGE code **4104808** (6-digit form `410480` is used by FNS/FNDE). CNPJ of the Município: **76.208.867/0001-07**, from the SICONFI `entes` endpoint (https://apidatalake.tesouro.gov.br/ords/siconfi/tt/entes) and confirmed by PNCP (https://pncp.gov.br/api/pncp/v1/orgaos/76208867000107).

Research date: 2026-10-28. Every endpoint marked **"tested"** was called live on that date from a plain HTTP client (PowerShell `Invoke-WebRequest`, no cookies, no browser), and the response shape below comes from that call. Claims are cited inline with the URL that holds them. Anything I could not confirm is marked **UNVERIFIED**.

---

## Summary: top sources for building the portal (ranked)

| # | Source | Why it matters | Access | Tested |
|---|---|---|---|---|
| 1 | **Prefeitura open data APIs (IPM Atende.Net)**: `prefa.cascavel.pr.gov.br/api/...` | First-party JSON for despesas, receitas, restos a pagar, budgets, licitações, contratos, ordens de compra, sanções, PCA, bens, veículos, almoxarifado and servidores. No key needed. | REST/JSON, no auth | Yes |
| 2 | **SICONFI / Tesouro API** (`apidatalake.tesouro.gov.br`) | Standardized RREO, RGF, DCA and MSC fiscal reports. Comparable with every other municipality. | REST/JSON, no auth | Yes |
| 3 | **PNCP API** (`pncp.gov.br/api/consulta`) | Mandatory national record of procurement and contracts under Lei 14.133. Clean schema, filterable by IBGE code or CNPJ. | REST/JSON, no auth | Yes |
| 4 | **TCE-PR**: PIT bulk ZIPs plus the TCE-PR dados abertos CSVs | SIM-AM data (despesas, licitações, contratos, obras and more) for all 399 PR municipalities from 2013, refreshed weekly. Obras and Mural de Licitações CSVs are refreshed daily. An independent copy of the municipality's own data. | Bulk ZIP/CSV, no auth | HEAD and partial GET |
| 5 | **SAPL of the Câmara** (`sapl.cascavel.pr.leg.br/api/`) | Full legislative record: matérias (43k+), parlamentares, sessões, votações, normas. | REST/JSON (Django REST), no auth | Yes |
| 6 | **Transferegov APIs** (transferências especiais and fundo a fundo) | Federal "emendas PIX" and fund-to-fund transfers to Cascavel. | REST/JSON (PostgREST), no auth | Yes |
| 7 | **CGU Portal da Transparência API** | Federal resources received, convênios, Bolsa Família/BPC per municipality, emendas, sanctions (CEIS/CNEP). | REST/JSON, **free API key** | 401 without key; spec read |
| 8 | **IBGE APIs** (`servicodados.ibge.gov.br`) | Population, PIB and other context indicators. | REST/JSON, no auth | Yes |

Also useful: FNS health transfers (undocumented JSON behind consultafns), TSE open data (CKAN, CSV ZIPs), INEP IDEB files, and the Paraná state transparency portal web services.

**Main finding:** the Prefeitura's portal runs on **IPM Sistemas Atende.Net**. It publishes **documented, unauthenticated JSON APIs** under "Acesso à Informação > API Dados Abertos". The interactive query grids, though, sit behind a **Cloudflare "Verify you are human" challenge**, so scraping the grids is not a viable path. Build on the documented APIs and fill the gaps from TCE-PR, SICONFI and PNCP.

---

## 1. Municipal level

### 1.1 Prefeitura: Portal da Transparência (IPM Atende.Net)

- **URL:** https://prefa.cascavel.pr.gov.br/transparencia/ (alias host: `cascavel.atende.net`; the Prefeitura home page https://www.cascavel.pr.gov.br/ redirects to https://prefa.cascavel.pr.gov.br/). The bare domain `cascavel.pr.gov.br` does not resolve (DNS ENOTFOUND during testing).
- **Owner / vendor:** Município de Cascavel. The footer reads "IPM Sistemas Ltda. Todos os Direitos Reservados." and links to http://www.ipm.com.br/. The mobile app links point to `br.com.ipm.mobile.Atende.Net` (https://prefa.cascavel.pr.gov.br/transparencia/). The IPM internal entity code for the Município is `2858` (the `CodigoCliente` field in API responses).
- **Data covered:** the site map lists 14 groups and roughly 200 items (https://prefa.cascavel.pr.gov.br/transparencia/mapa-do-site):
  - Licitações, contratos e patrimônio: licitações, agenda, PCA, ordens de compra, atas SRP, contratos, terceirizados, sanções, fiscais de contrato, almoxarifado, bens, veículos.
  - Receitas, including dívida ativa list and renúncias.
  - Despesas: empenhos, liquidações, pagamentos, diárias, cartões corporativos, ordem cronológica, restos a pagar, convênios/parcerias.
  - Pessoal: efetivos, comissionados, estagiários, temporários, salário, tabelas salariais 2010–2026, exonerados, lotação.
  - Contas públicas: balanços, extratos bancários 2013–2026, precatórios 2020–2026.
  - Relatórios legais: RREO, RGF, LOA, LDO, PPA, prestação de contas.
  - Obras públicas and obras paralisadas.
  - Saúde: listas de espera, medicamentos, RAG, RDQA.
  - Educação, COVID-19, controle interno, emendas parlamentares municipais.
- **Freshness:** each item shows "Informações atualizadas em: dd/mm/yyyy". The tooltip says the data are updated "automaticamente em tempo real" from a single shared database (seen on https://prefa.cascavel.pr.gov.br/transparencia/item/api-de-dados-abertos-contabilidade).
- **Format / access (HTML grids):** each item embeds an iframe (`/transparencia/item/embed/data/<base64 JSON>/item/<slug>`). The iframe calls `atende.php?rot=…&aca=…&processo=…` over AJAX POST. Grids have "Dados Abertos" and "Imprimir/Exportar" buttons. **Caveat:** in testing, clicking *Consultar* on the Obras grid (https://prefa.cascavel.pr.gov.br/transparencia/item/obras) and loading the Inventário de Dados Abertos (https://prefa.cascavel.pr.gov.br/transparencia/item/inventario-de-dados-abertos) both raised a **Cloudflare Turnstile "Verify you are human"** challenge. I did not try to get past it. Treat the grids and their export buttons as not machine-accessible.
- **Format / access (documented APIs):** see 1.2. These worked without any challenge.

### 1.2 Prefeitura: open data APIs (best municipal source)

Documented under "Acesso à Informação > API Dados Abertos" on three pages: Contabilidade (https://prefa.cascavel.pr.gov.br/transparencia/item/api-de-dados-abertos-contabilidade), Pessoal (https://prefa.cascavel.pr.gov.br/transparencia/item/api-de-dados-abertos-de-pessoal) and Suprimentos (https://prefa.cascavel.pr.gov.br/transparencia/item/api-de-dados-abertos-suprimentos). No authentication. JSON. Dates use `DD/MM/AAAA`. Text is JSON with `\u` escapes; decode as UTF-8.

**a) Contabilidade**: base `https://prefa.cascavel.pr.gov.br/api/WCPDadosAbertos/`

| Endpoint | Params | Tested response |
|---|---|---|
| `GET /despesas` | `dataInicial`, `dataFinal` (both needed; `dataFinal` alone returned **400**) | HTTP 201. `{"status":"ok","retorno":[{orgaoCodigo, orgaoDescricao, unidadeCodigo, unidadeDescricao, fonteRecurso, fonteRecursoDescricao, cpfCnpjCredor, nomeCredor, valorEmpenhado, valorAnulado, valorLiquidado, valorRetido, valorPago}]}`. January 2026 gave 1,102 rows across 25 órgãos. |
| `GET /despesaRestos` | `dataFinal` | Same órgão/fonte/credor keys plus `valorProcessadoInscrito`, `valorNaoProcessadoInscrito`, `…Cancelado`, `…Liquidado`, `valorPagar`, `valorPago…`. The response for `dataFinal=31/01/2026` was about 1.4 MB. |
| `GET /despesasOrcadas` | `dataFinal` (returns that year's budget, including amendments up to the date) | `{orgaoCodigo, orgaoDescricao, unidadeCodigo, unidadeDescricao, valorOrcado}` |
| `GET /receitas` | `dataInicial`, `dataFinal` | `{contaCodigo, contaDescricao, valorArrecadado}` |
| `GET /receitasOrcadas` | `dataFinal` | `{contaCodigo, contaDescricao, valorOrcado}` |

Example (tested):
```
GET https://prefa.cascavel.pr.gov.br/api/WCPDadosAbertos/despesas?dataInicial=01/01/2026&dataFinal=05/01/2026
-> 201 application/json
{"status":"ok","retorno":[{"orgaoCodigo":"5","orgaoDescricao":" Secretaria Munl de Finanças","unidadeCodigo":"2",
 "unidadeDescricao":" Encargos Gerais do Município","fonteRecurso":"0","fonteRecursoDescricao":"Recursos Ordinários (Livres)",
 "cpfCnpjCredor":"00.394.460/0138-04","nomeCredor":"MINISTÉRIO DA FAZENDA / ...","valorEmpenhado":"6000000.00",
 "valorAnulado":"0","valorLiquidado":"0","valorRetido":"0","valorPago":"0"}, ...]}
```

Caveats:
- `/despesas` is **aggregated by órgão/unidade/fonte/credor**. It has **no empenho number, date, process number or object description**. Empenho-level detail (which LC 131's art. 48-A requires) appears only in the Cloudflare-gated grids. Get it from TCE-PR SIM-AM (section 2.1) or ask through LAI.
- All numbers are strings.
- Successful calls return HTTP **201**, not 200.
- The data cover the whole Município, including the Câmara as `orgaoCodigo 1`. `https://camaracascavel.atende.net/api/WCPDadosAbertos/despesas?...` returned an **identical** 1,102-row set for January 2026.

**b) Suprimentos**: base `https://prefa.cascavel.pr.gov.br/api/transparencia/dados-abertos/v1`

Endpoints listed in the docs:
- `/wco/licitacao`
- `/wco/contrato`
- `/wco/compra` (ordens de compra)
- `/wco/sancoes`
- `/wco/pca`
- `/wpa/bem`
- `/wfm/veiculo`
- `/wax/entrada`
- `/wax/saida`

Paging uses `pagina` (starts at 0, 100 rows per page). The docs say you get a single record by passing the key fields together, entity code + year + number, but the docs page does not name the query-string parameters; it shows them only as "[Código da Entidade]" and similar.

Tested:
```
GET https://prefa.cascavel.pr.gov.br/api/transparencia/dados-abertos/v1/wco/licitacao
-> 200 {"licitacao":[{"Entidade":{"nome":"MUNICÍPIO DE CASCAVEL"},"CodigoCliente":"2858","AnoMinuta":"2005","NumeroMinuta":"2",
   "Numero":"1","Ano":"2005","DataHomologacao":"10/02/2005","Data":"27/01/2005","ModalidadeEstado":{"codigo":"46","descricao":"Convite"},
   "Situacao":"4","ValorHomologado":"31913.3100","ValorEstimado":"41727.00","Processo":{"finalidade":"AQUISIÇÃO DE VALE GÁS ...","numero":"2","ano":"2005"}}, ...]}
GET .../wco/contrato?pagina=1
-> 200 {"contrato":[{"ContratoAnoNumero":"004/2019","CodigoCliente":"2858","UnidadeDotacao":{"orgao":{...}},"Tipoinstrumento":"2",
   "ObjetoTC":"Aquisição de materiais e serviços metalúrgicos","DataIniVigencia":"16/01/2019","DataFimVigencia":"16/01/2020",
   "Licitacao":{"ano":"2018","numero":"353",...},"Fornecedor":{...},"ValorContrato":"...","FimVigAtualizada":"...","Situacao":"1", ...}]}
```

- Licitações go back to **2005**. Page 150 was still returning 2019 records.
- `Situacao` enums are documented on the page. Licitação: 1 Aberta … 12 Suspensa. Contrato `Tipoinstrumento`: 1–13.
- The docs say the supplier CPF/CNPJ is anonymized in the contrato and compra outputs.

**c) Pessoal**: `GET https://prefa.cascavel.pr.gov.br/api/transparencia-pessoal-funcionarios`

- Params:
  - `tipoBusca`: 1 = working or on leave (default), 2 = separated, 3 = all.
  - `admissaoInicio`, `admissaoFinal`.
  - `desligamentoInicio`, `desligamentoFinal`: required when `tipoBusca` is 2 or 3.
  - `pagina`.
- Tested: 200 `{"registros":[{entidade, matricula, nome, cpf:"***.546.239-**", situacao, admissao, regime, classificacao, formaInvestidura, centroCusto, localTrabalho, cargo, funcao, nivel, salarioBase:"12.702,48", horario, horasMensais, horasSemanais}], "paginaAtual":1, "totalPaginas":105, "totalRegistros":10416}`.
- **Caveat:** this gives **salário base only**. Gross and net monthly pay (folha) is in the "Salário" grid item (https://prefa.cascavel.pr.gov.br/transparencia/item/relacao-funcionario-x-salario), which sits behind Cloudflare. Not available by API.

### 1.3 Diário Oficial do Município ("Órgão Oficial")

- **URL:** https://prefa.cascavel.pr.gov.br/diariooficial/edicao (IPM module; footer "2026 - IPM Sistemas Ltda.").
- **Data:** editions from **May 2009** to the present. Search by edition number, period, keyword and "Tipo do Ato". There are three cadernos per edition:
  - Caderno 01: Executive acts, mainly licitações.
  - Caderno 02: Legislative acts.
  - Caderno 03: complementary Executive information.
  - Extraordinary editions are also published.
- Latest edition seen: **Edição 4575, 28/10/2026**. Each entry has an internal `data-id` (for example 10773) and shows "Publicado Por / Assinado Por" (digitally signed).
- **Format:** **PDF only** (file sizes of 238 KB to 30 MB were listed). Full-text extraction or OCR will be needed.
- **UNVERIFIED:** the direct PDF download URL pattern. I did not download any edition.

### 1.4 e-SIC / LAI channel (Prefeitura)

- **URL:** https://cascavel.atende.net/cidadao/acesso-informacao. The page has "Nova Solicitação", "Acompanhar Solicitação" and "Estatísticas de Solicitações".
- **Owner:** Controladoria-Geral do Município. The local rule is **Decreto Municipal nº 20.609/2026**.
- **Channels:** online e-SIC form, ouvidoria@cascavel.pr.gov.br, in person at Rua Paraná 5.000, 2º andar, phone 156.
- **Deadlines:** 20 days plus 10; appeal decided in 5 days. The page cites LAI arts. 11 and 15–16.
- **Statistics:** the portal also has "Relatório Estatístico LAI" and "Estatística de Pedidos" items (https://prefa.cascavel.pr.gov.br/transparencia/mapa-do-site).

### 1.5 Câmara Municipal de Cascavel

- **Website:** https://www.camaracascavel.pr.gov.br/ (footer credits developer `webmes.com.br`).
- **Transparência:** https://camaracascavel.atende.net (also IPM). Linked items: `/transparencia/item/licitacoes-gerais` and `/transparencia/item/contratos-gerais` (https://www.camaracascavel.pr.gov.br/). The IPM APIs on this host return the **same consolidated Município data** as the Prefeitura host. Tested `/api/WCPDadosAbertos/despesas` and `/api/transparencia/dados-abertos/v1/wco/contrato`; the latter returned `CodigoCliente 2858` Município contracts. Filter by `orgaoCodigo = 1` ("Camara Municipal") to isolate the Legislative branch.
- **Site API:** "Dados Abertos" page at https://www.camaracascavel.pr.gov.br/dados-abertos/. It documents a GET/JSON API at `api.camaracascavel.pr.gov.br` with paths such as:
  - `/a-camara/vereadores/`
  - `/legislacoes/proposicoes/?year=`
  - `/legislacoes/leis-municipais/`
  - `/comunicacao/atas-das-sessoes/`

  **When tested it was broken.** Both `https://api.camaracascavel.pr.gov.br/a-camara/vereadores/` and `/legislacoes/leis-municipais/?year=2025` returned a PHP fatal error ("Erro ao conectar ao banco de dados … SQLSTATE[08006] timeout expired"). The response field documentation on the page is also empty. **UNVERIFIED** whether it ever works.
- **SAPL (legislative process system), tested and working:** https://sapl.cascavel.pr.leg.br/api/. It is a Django REST Framework API (the Interlegis SAPL product), and the root lists dozens of resources (audiência, base/autor, comissões, matéria, norma, parlamentares, sessão…).
  ```
  GET https://sapl.cascavel.pr.leg.br/api/materia/materialegislativa/?page_size=2
  -> 200 {"pagination":{"total_entries":43004,"total_pages":21502,"page":1,...},
          "results":[{"id":40270,"__str__":"Dilação de Prazo de Requerimento nº 1 de 2026","numero":1,"ano":2026,
          "data_apresentacao":"2026-02-12","em_tramitacao":true,"ementa":"...","texto_original":...}]}
  GET https://sapl.cascavel.pr.leg.br/api/parlamentares/parlamentar/?page_size=2
  -> 200 {... "total_entries":84 ..., "results":[{"id":40,"nome_parlamentar":"Alécio Espínola","email":"...","ativo":false, ...}]}
  ```
- **SIC (Câmara):** https://www.camaracascavel.pr.gov.br/transparencia/acesso-a-informacao/sic/. Requests go through an online form (`.../sic/registrar-manifestacao/`), sic@camaracascavel.pr.gov.br, or in person at Rua Pernambuco 1.843. The deadline is 20 days plus 10, and appeals must be filed within 10 days.

### 1.6 Licitações, contratos, servidores, receitas/despesas, obras: where each lives (municipal)

| Topic | Best machine source | Fallback |
|---|---|---|
| Licitações | IPM `/wco/licitacao` (2005 onward); PNCP (14.133 era) | TCE-PR Mural de Licitações CSV; Diário Oficial Caderno 01 PDFs |
| Contratos | IPM `/wco/contrato`; PNCP `/contratos` | TCE-PR PIT ZIP |
| Servidores | IPM `/transparencia-pessoal-funcionarios` (salário base only) | Folha detail: gated grid, so use a LAI request |
| Receitas / despesas | IPM `WCPDadosAbertos` (aggregated); SICONFI | TCE-PR PIT ZIP (SIM-AM, empenho-level) |
| Obras | TCE-PR Obras Municipais CSV (daily, with coordinates) | IPM Obras grid (gated); PIT `/Obra` |

The portal's own Obras group links to TCE-PR's PIT (https://pit.tce.pr.gov.br/Obra), per the site map (https://prefa.cascavel.pr.gov.br/transparencia/mapa-do-site).

---

## 2. State level (Paraná)

### 2.1 TCE-PR: Portal Informação para Todos (PIT), SIM-AM data

- **URL:** https://pit.tce.pr.gov.br/
- **Owner:** Tribunal de Contas do Estado do Paraná. The data are declared by the audited entities through **SIM-AM** (layouts at https://www.tce.pr.gov.br/para-o-fiscalizado/sistemas/sim-sistema-de-informacoes-municipais/layout.htm, not opened).
- **Modules** (https://pit.tce.pr.gov.br/):
  - Entidades (1990–2026), Receitas, Despesas (by credor and by classificação).
  - Obras, Contratos, Licitações, Convênios, Emendas Parlamentares.
  - Educação, Combustíveis, Diárias, Relatórios de Fiscalização, Gestão em Foco, Processos.
  - Maps and charts.
- **Bulk download, "Consolidado":** https://pit.tce.pr.gov.br/Dados/DadosConsulta/Consolidado
  - Pattern: `https://pit.tce.pr.gov.br/Arquivos/{YEAR}_PIT_TodosArquivos.zip`, years **2013–2026**, all municipalities, files split by theme plus a relationships file.
  - Regenerated "uma vez por semana" from closed SIM-AM submissions, and subject to change.
  - HEAD test: `2026_PIT_TodosArquivos.zip` is **892,692,184 bytes**, Last-Modified 18 Oct 2026. `2025_...zip` is **1,402,462,354 bytes**, Last-Modified 10 Sep 2026.
  - Contradiction: the per-municipality download page (https://pit.tce.pr.gov.br/Dados/DadosConsulta/Consulta) says "Os dados disponibilizados nesta pesquisa não estão em formato aberto", while the Consolidado page calls its files "dados brutos em formato aberto".
- **Auth:** none.
- **Caveats:**
  - Very large files. You need a pipeline that filters on Cascavel's code.
  - Lag: only closed monthly submissions are included.
  - **UNVERIFIED:** the internal file formats inside the ZIP. I did not download one.

### 2.2 TCE-PR: Dados Abertos (CSV, daily)

- **Catalog:** https://servicos.tce.pr.gov.br/servicos/srv_dados_abertos.aspx (linked from https://www.tce.pr.gov.br/transparencia-do-tce-pr/dados-abertos-tce-pr/dados-abertos-acessar.htm).
- **Mural de Licitações:** updated daily, current year only. Base CSV: `https://servicos.tce.pr.gov.br/servicos/arquivos/dadosabertos/muraldelicitacoes/2026_mural_de_licitacoes_base_de_dados.csv`, about 19 MB, "Atualizado em: 28/10/2026". Earlier years are in the folder `.../muraldelicitacoes/`, with a data dictionary in XLSX.
- **Obras Municipais:** updated daily, from **January 2013**, includes geographic coordinates. Four CSVs:
  - `.../obrasmunicipais/obras_municipais_base_de_dados.csv` (about 26.9 MB)
  - `acompanhamentos_base_de_dados.csv`
  - `responsaveis_tecnicos_base_de_dados.csv`
  - `bens_patrimoniais_base_de_dados.csv`

  An ER model (PDF) and a dictionary (XLSX) are also provided.
- **Format (tested with a range request):** semicolon-delimited, quoted, UTF-8. A dashed separator line follows the header, so skip line 2.
  - Obras header starts: `idIntervencao;nmPessoa;nrDocumento;cdIBGE;nmMunicipio;cdIntervencao;nrAnoIntervencao;…;vlIntervencao;…;dtInicio;…`
  - **`cdIBGE` is a 5-digit code without the state prefix and check digit.** For example, São José dos Pinhais (4125506) appears as `25506`, so Cascavel should be **`04808`** (inferred, not checked against a Cascavel row).
- **Auth:** none. Usage rules PDF: https://servicos.tce.pr.gov.br/servicos/arquivos/dadosabertos/dados_abertos_tcepr_regras_de_utilizacao.pdf (not opened).

### 2.3 Portal da Transparência do Estado do Paraná (state spending and transfers to municipalities)

- **URL:** https://www.transparencia.pr.gov.br/pte/home (Controladoria Geral do Estado).
- **Relevant pages:**
  - "Repasses do Governo Estadual aos Municípios": `/pte/assunto/4/100`
  - "Repasses aos Municípios (LC 194/2022 e 201/2023)": `/pte/assunto/4/284`
  - "Convênios do Governo Estadual": `/pte/assunto/4/127`
  - Emendas parlamentares: `/pte/assunto/4/329`

  These are HTML/JSF query pages.
- **Downloads page:** https://www.transparencia.pr.gov.br/pte/assunto/8/129 lists 13 datasets with frequency and coverage. Examples:
  - Gastos Públicos: daily, 2002–2026.
  - Licitações and Contratos: daily, 2007–2026.
  - Convênios (TCE/SIT): weekly, 2007–2026.

  The download links render via JavaScript, so the formats and URLs are **UNVERIFIED**.
- **Web services, tested:** documented at https://www.transparencia.pr.gov.br/pte/pages/informacoes/webservices/webservices_despesas.jsf. Endpoint: `GET https://www.transparencia.pr.gov.br/pte/ws/despesas/listardespesa?exercicio=YYYY&mes=M[&modalidade=40&orgao=…]`, returning JSON `{retorno, mensagem, quantidade, listaDespesa:[{despesa_exercicio, despesa_mes, valor_empenhado, valor_liquidado, valor_pago, …natureza/elemento…}]}`.
  - `exercicio=2023&mes=1&modalidade=40` returned 61 rows.
  - `exercicio=2024` and `2026` returned "Nenhum registro encontrado".
  - So the web service seems to cover only **through 2023**, which matches the menu label "Gastos Públicos – Até 2023".
  - The data are **aggregated by budget classification and do not identify the recipient municipality**, so they are of limited use for Cascavel.
- **Open data portal:** https://www.dados.pr.gov.br/ (Drupal on Celepar infrastructure, inferred from asset paths). It points to Ipardes **BDEweb** for municipal statistics. A CKAN API (`/api/3/action/...`) returned **404**, and I found no API.
- **Celepar:** it is the state IT company that hosts the above. I found no separate Celepar data offering relevant to Cascavel.

---

## 3. Federal level

### 3.1 SICONFI / Tesouro Nacional API (no key)

- **Base:** `https://apidatalake.tesouro.gov.br/ords/siconfi/tt/` (Oracle ORDS). Responses use `items`, `hasMore`, `limit` (5000), `offset` and `count`. Tested endpoints:
  - `entes`: Cascavel → `{"cod_ibge":4104808,"ente":"Cascavel","uf":"PR","esfera":"M","exercicio":2026,"populacao":368195,"cnpj":"76208867000107"}`
  - `extrato_entregas?id_ente=4104808&an_referencia=2026`: 24 items covering what was delivered and when (MSC, RREO, RGF…), for both Prefeitura and Câmara.
  - `rreo?an_exercicio=2026&nr_periodo=4&co_tipo_demonstrativo=RREO&no_anexo=RREO-Anexo%2001&co_esfera=M&id_ente=4104808`: 512 rows `{exercicio, periodo, periodicidade:"B", instituicao, cod_ibge, anexo, rotulo, coluna, cod_conta, conta, valor}`. Bimestre 5 of 2026 returned 0 rows, meaning it has not been filed yet.
  - `rgf?an_exercicio=2026&in_periodicidade=Q&nr_periodo=1&co_tipo_demonstrativo=RGF&no_anexo=RGF-Anexo%2001&co_esfera=M&co_poder=E&id_ente=4104808`: personnel spending per month (`<MR-11>` etc.).
  - `dca?an_exercicio=2025&no_anexo=DCA-Anexo%20I-C&id_ente=4104808`: annual revenue (2025 gross revenue "Receitas Brutas Realizadas" = 2,241,214,991.20).
- **Freshness:** bimonthly (RREO) and four-monthly (RGF), as filed. DCA is annual. Both the Câmara and the Prefeitura appear in delivery records; use `co_poder` for RGF.
- **Legal basis:** LRF art. 48 §2 (LC 156/2016) requires municipalities to submit to the "órgão central de contabilidade da União" (https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp101.htm).

### 3.2 PNCP: Portal Nacional de Contratações Públicas (no key)

- **Consulta API base:** `https://pncp.gov.br/api/consulta/v1/`. Tested:
  ```
  GET /contratacoes/publicacao?dataInicial=20261001&dataFinal=20261028&codigoModalidadeContratacao=6&codigoMunicipioIbge=4104808&pagina=1&tamanhoPagina=10
  -> 200 {"data":[{"orgaoEntidade":{"cnpj":"76208867000107","razaoSocial":"MUNICIPIO DE CASCAVEL",...},"anoCompra":2026,
          "sequencialCompra":528,"numeroCompra":"PCE 140","processo":"122767/2026","objetoCompra":"REGISTRO DE PREÇOS ...",
          "dataAberturaProposta":"2026-10-01T09:00:00","dataPublicacaoPncp":"2026-10-01T08:23:56", ...}], ...}
  GET /contratos?dataInicial=20261001&dataFinal=20261028&cnpjOrgao=76208867000107&pagina=1&tamanhoPagina=10
  -> 200 {"data":[{"numeroControlePncpCompra":"76208867000107-1-000330/2026","anoContrato":2026,"niFornecedor":"03030002000111",
          "nomeRazaoSocialFornecedor":"PAVIMENTACOES E TERRAPLENAGENS SCHMITT LTDA","unidadeOrgao":{"codigoIbge":"4104808",
          "nomeUnidade":"Secretaria Mun Serviços e Obras Publicas"}, ...}],
          "totalRegistros":28,"totalPaginas":3,"numeroPagina":1,"paginasRestantes":2}
  GET https://pncp.gov.br/api/pncp/v1/orgaos/76208867000107 -> órgão registered 2021-07-28, validado=true
  ```
- **Notes:**
  - `codigoModalidadeContratacao` is required on `/contratacoes/publicacao`.
  - Dates are `yyyyMMdd`.
  - Unlike IPM, PNCP shows the full supplier CNPJ.
- **Legal basis:** Lei 14.133 art. 174 makes PNCP the site for "divulgação centralizada e obrigatória". Under art. 94, contracts take effect only once published there: 20 business days for licitação, 10 for direct contracting. For obras, unit prices must be published within 25 business days and executed quantities within 45 business days of completion (https://www.planalto.gov.br/ccivil_03/_ato2019-2022/2021/lei/l14133.htm). Coverage is therefore strong from roughly 2023/2024 onward, after the 14.133 transition. Earlier procurement comes from IPM or TCE-PR.

### 3.3 CGU Portal da Transparência API (free key required)

- **Docs:** https://portaldatransparencia.gov.br/api-de-dados. Swagger at https://api.portaldatransparencia.gov.br/ (spec: `/v3/api-docs`).
- **Auth:**
  - Register an email at http://portaldatransparencia.gov.br/api-de-dados/cadastrar-email to receive a token.
  - Send it in the header **`chave-api-dados`** (OpenAPI `securitySchemes`).
  - Without it, the call returns **401** (tested: `/api-de-dados/bolsa-familia-por-municipio?mesAno=202601&codigoIbge=4104808&pagina=1` → 401).
- **Rate limit:** 400 requests per minute from 06:00 to 23:59, and 700 per minute from 00:00 to 05:59. For full datasets the CGU recommends the bulk spreadsheets.
- **Endpoints relevant to a municipality (from the spec):**
  - `/despesas/recursos-recebidos`: `mesAnoInicio`, `mesAnoFim`, `codigoIBGE`, `codigoFavorecido`… This is federal money received by entities in Cascavel.
  - `/convenios`: `codigoIBGE`, `uf`, `convenente`, date ranges…
  - `/novo-bolsa-familia-por-municipio` and `/bolsa-familia-por-municipio`.
  - `/bpc-por-municipio`, `/peti-por-municipio`, `/seguro-defeso-por-municipio`, `/safra-por-municipio`, `/auxilio-brasil-por-municipio`, `/auxilio-emergencial-por-municipio`.
  - `/coronavirus/transferencias`.
  - `/emendas`.
  - `/renuncias-valor` (by `codigoIbge`).
  - Sanctions: CEIS, CNEP, CEPIM, useful to cross-check municipal suppliers.
- **Terms:** free use under Decreto 8.777/2016, per the docs page.
- **UNVERIFIED:** actual responses, because I have no key.

### 3.4 Transferegov (no key)

- **Transferências especiais ("emendas PIX", EC 105/2019):** Swagger 2.0 spec at https://api.transferegov.gestao.gov.br/transferenciasespeciais/. This is PostgREST, so filters look like `?col=eq.value&limit=`. Paths include `/plano_acao_especial`, `/empenho_especial`, `/ordem_pagamento_ordem_bancaria_especial`, `/relatorio_gestao_especial` and others.
  - Tested: `GET https://api.transferegov.gestao.gov.br/transferenciasespeciais/plano_acao_especial?cnpj_beneficiario_plano_acao=eq.76208867000107&limit=2` → 200 `[{"id_plano_acao":4831,"ano_plano_acao":2020,"nome_beneficiario_plano_acao":"MUNICIPIO DE CASCAVEL","nome_parlamentar_emenda_plano_acao":"Aliel Machado", ...}]`.
  - This dataset filters by **CNPJ**, not IBGE code; `codigo_ibge_...` columns do not exist (a 400 error was returned).
- **Fundo a fundo:** spec at https://api.transferegov.gestao.gov.br/fundoafundo/. Paths include `/programa`, `/plano_acao`, `/empenho`, `/gestao_financeira_lancamentos`, …
  - Tested: `.../fundoafundo/plano_acao?codigo_ibge_municipio_ente_recebedor_plano_acao=eq.4104808&limit=1` → 200, returning a Cascavel plano de ação under Lei 14.017/2020 (Aldir Blanc).
- **UNVERIFIED:** the Transferegov convênios API (discricionárias). I did not test it.

### 3.5 IBGE APIs (no key)

- `GET https://servicodados.ibge.gov.br/api/v1/localidades/municipios/4104808` → 200. Returns the municipality hierarchy (microrregião Cascavel, mesorregião Oeste Paranaense, região imediata and intermediária Cascavel).
- `GET https://servicodados.ibge.gov.br/api/v3/agregados/6579/periodos/-1/variaveis/9324?localidades=N6[4104808]` → `"serie":{"2026":"372026"}` (estimated population).
- `GET https://servicodados.ibge.gov.br/api/v3/agregados/5938/periodos/-1/variaveis/37?localidades=N6[4104808]` → `"serie":{"2023":"19204178"}` (PIB at current prices, R$ thousand; the 2023 vintage is the latest).
- `GET https://servicodados.ibge.gov.br/api/v1/pesquisas/indicadores/29171/resultados/4104808` → a population series from 2011 to 2026. Note that `localidade` here uses the 6-digit code `410480`.
- Note: SICONFI's `populacao` for 2026 is 368,195, the IBGE 2025 estimate, while the SIDRA 2026 estimate is 372,026. Pick one source and label it.

### 3.6 Health: FNS / DATASUS

- **FNS Consulta:** https://consultafns.saude.gov.br/#/consolidada is an Angular SPA. It calls **undocumented JSON** endpoints, which worked from a plain client (tested):
  ```
  GET https://consultafns.saude.gov.br/recursos/consulta-consolidada/repasse-bloco?ano=2025&coMunicipioIbge=410480&count=10&page=1&sgUf=PR
  -> 200 {"resultado":[{"codigo":10,"nome":"Manutenção das Ações e Serviços Públicos de Saúde","vlTotal":82859773.48,
          "repasses":[{"nome":"ATENÇÃO PRIMÁRIA","vlTotal":48776616.65},{"nome":"ATENÇÃO DE MÉDIA E ALTA COMPLEXIDADE ...","vlTotal":23793727.28}, ...]}, ...]}
  GET .../recursos/consulta-consolidada/entidades?ano=2025&coMunicipioIbge=410480&count=10&page=1&sgUf=PR
  -> 200 {"resultado":{"dados":[{"cpfCnpj":"09051532000122","razaoSocial":"FUNDO MUNICIPAL DE SAUDE DE CASCAVEL","valorTotalUF":"95551309.48", ...}],"total":1}}
  ```
  It uses the **6-digit IBGE code**. Coverage runs from 1998 to 2026 (year dropdown). **Caveat:** undocumented, so it may change without notice.
- **OpenDataSUS:** `https://opendatasus.saude.gov.br/api/3/action/package_list` returned an HTML (Next.js) page rather than CKAN JSON. **UNVERIFIED** whether a CKAN API still exists. DATASUS TabNet was not tested.

### 3.7 Education: FNDE / INEP

- **FNDE SIGEF "Liberações":** the URL that the municipal portal links to (https://www.fnde.gov.br/sigefweb/index.php/liberacoes?...&cidade=410480...) returns only a query form, and the form requires a **captcha**. Not automatable. Use CGU `/despesas/recursos-recebidos` instead (UNVERIFIED for FNDE coverage). `https://www.fnde.gov.br/dadosabertos/api/3/action/package_list` returned HTML, not CKAN JSON (**UNVERIFIED**).
- **INEP IDEB:** results page https://www.gov.br/inep/pt-br/areas-de-atuacao/pesquisas-estatisticas-e-indicadores/ideb/resultados. It links static ZIPs for the **2025** edition at municipal and school level, for example:
  - `https://download.inep.gov.br/ideb/resultados/divulgacao_anos_iniciais_municipios_2025.zip`
  - `…/divulgacao_anos_finais_municipios_2025.zip`
  - `…/divulgacao_ensino_medio_municipios_2025.zip`
  - `…_escolas_2025.zip`

  Bulk files only, no API. Censo Escolar microdados page: https://www.gov.br/inep/pt-br/acesso-a-informacao/dados-abertos/microdados/censo-escolar (loads; file list not extracted).
- The municipal portal also links "Dados da educação municipal" to https://www.cascavel.app/transparencia (**UNVERIFIED**; not opened).

### 3.8 TSE: elections, candidates, campaign finance (no key)

- **CKAN API, tested:** `https://dadosabertos.tse.jus.br/api/3/action/package_show?id=candidatos-2024`. Resources are CSV ZIPs on `cdn.tse.jus.br`:
  - `consulta_cand_2024.zip`
  - `consulta_cand_complementar_2024.zip`
  - `bem_candidato_2024.zip`
  - `consulta_vagas_2024.zip`
  - `rede_social_candidato_2024.zip`
  - candidate photos and proposals by state, for example `foto_cand2024_PR_div.zip` and `proposta_governo_2024_PR.zip`
- **Campaign finance:** `prestacao-de-contas-eleitorais-2024` has `prestacao_de_contas_eleitorais_candidatos_2024.zip`, bank statements, and FEFC/FP. License CC-BY.
- **Caveat:** TSE uses its **own municipality code**. The official TSE↔IBGE mapping is at `https://cdn.tse.jus.br/estatistica/sead/odsele/municipio_tse_ibge/municipio_tse_ibge.zip` (dataset `codigos-oficiais-de-uf-e-municipios-segundo-o-tse-e-o-ibge`). Cascavel's TSE code is **UNVERIFIED**.

### 3.9 dados.gov.br

- The API (`https://dados.gov.br/dados/api/publico/conjuntos-dados`) now **requires a key**. Without one the call returned **401**. Per the OpenAPI spec (https://dados.gov.br/v3/api-docs), the header is `chave-api-dados-abertos`, obtained from "Minha Conta" after logging in. It is mostly a catalog that points to the federal sources above. I did not search for Cascavel-specific datasets.

---

## 4. Legal baseline: what must exist

### LAI: Lei 12.527/2011 (https://www.planalto.gov.br/ccivil_03/_ato2011-2014/2011/lei/l12527.htm)

**Art. 8 (active transparency):** public bodies must publish matters of general or collective interest without being asked.

§1 sets the minimum content:
- (I) competências, estrutura organizacional, endereços, telefones, horários.
- (II) "registros de quaisquer repasses ou transferências de recursos financeiros".
- (III) "registros das despesas".
- (IV) licitações, including editais, resultados and contracts signed.
- (V) "dados gerais para o acompanhamento de programas, ações, projetos e obras".
- (VI) FAQ.

§2: publication on official websites is mandatory.

§3 sets requirements for those websites, including:
- (II) reports in several formats, "inclusive abertos e não proprietários".
- (III) "possibilitar o acesso automatizado por sistemas externos em formatos abertos, estruturados e legíveis por máquina".
- (IV) document the formats used.
- (VI) keep the information up to date.

The Cloudflare challenge on Cascavel's grids sits uneasily with §3(III). The documented APIs are the compliant path.

§4: municipalities of up to 10,000 inhabitants are exempt from mandatory internet publication. This does not apply to Cascavel (about 372k).

**Arts. 10–11 (passive transparency):**
- Anyone may request information.
- The body must provide an online request channel (art. 10 §2).
- It may not ask for the reason (art. 10 §3).
- It must answer immediately, or within 20 days extendable by 10 (art. 11 §§1–2).

### LC 131/2009, the "Lei da Transparência", which amends the LRF (https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp131.htm; current text at https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp101.htm)

- **LRF art. 48 §1 (formerly the parágrafo único):**
  - (II) real-time release of "informações pormenorizadas sobre a execução orçamentária e financeira, em meios eletrônicos de acesso público".
  - (III) an integrated financial system meeting a federal minimum standard.
  - The planalto compiled text also shows an inciso IV, on disclosure of tax, financial and credit benefits, attributed to **LC 224/2025**. This is as rendered by the planalto page and was not cross-checked.
- **Art. 48-A:**
  - (I) Despesa: every act of the spending units "no momento de sua realização", with at least the process number, the good or service, the payee, and the licitação where applicable.
  - (II) Receita: the lançamento and recebimento of all revenue, including extraordinary revenue.
- **Art. 73-B:** deadline of 1 year from 27/05/2009 for municipalities over 100,000 inhabitants (Cascavel included). Art. 73-C: the sanction for non-compliance is the suspension of voluntary transfers (art. 23 §3 I). Art. 73-A: any citizen may report non-compliance to the TCE and the MP.
- **Later LRF changes (LC 156/2016):**
  - §2: data must go to the central accounting body of the Union, which is in practice SICONFI.
  - §6: single budget and financial execution systems for all branches.

### Lei 14.133/2021 (https://www.planalto.gov.br/ccivil_03/_ato2019-2022/2021/lei/l14133.htm)

- Art. 174: PNCP is the mandatory centralized publication site. Its §2 list includes annual contracting plans (PCA), editais, atas de registro de preços, contracts and amendments, and notas fiscais.
- Art. 94: publication deadlines on PNCP (see 3.2).
- Art. 176: the transition for municipalities of up to 20k inhabitants does not apply to Cascavel.

**Implication: datasets that must exist for Cascavel.**
- Detailed, near-real-time despesas: empenho, liquidação, pagamento with process, credor and licitação.
- Receitas: lançamento and arrecadação.
- Repasses and transferências.
- Licitações, editais, results and contracts.
- Obras and programs.
- RREO/RGF/LOA/LDO/PPA.
- PCA.
- Personnel data. LAI art. 8 does not list pay explicitly. The detailed folha the portal publishes under Pessoal stems from regulation and STF case law, which I did not verify here.
- An e-SIC channel with statistics.

---

## 5. Gaps: what needs scraping or LAI requests

1. **Empenho-level expense detail** (empenho number and date, process, object, licitação link). The IPM `/despesas` API is aggregated. Grid detail is behind Cloudflare. Options:
   - (a) TCE-PR PIT weekly ZIP (SIM-AM), the most practical route.
   - (b) A LAI request asking the Prefeitura to expose empenho-level detail by API, citing LAI art. 8 §3(III) and LRF art. 48-A(I).
2. **Folha de pagamento** (gross pay, deductions, net per servidor per month). The API has only `salarioBase`. The grid ("Salário") is gated. Request a monthly CSV through e-SIC, or check whether the PIT ZIP has a personnel theme (UNVERIFIED).
3. **Obras detail** (medições, % concluded, photos). Use TCE-PR Obras Municipais CSVs (daily, with coordinates) and PNCP art. 94 §3 obra publications. The municipal grid is gated.
4. **Diário Oficial:** PDF only. It needs a crawler for the edition list (HTML) plus PDF text extraction or OCR. The download URL pattern is unverified.
5. **Câmara site API** (`api.camaracascavel.pr.gov.br`) was down (database timeout). Use SAPL instead. Report the outage to the Câmara Ouvidoria or SIC if it persists.
6. **IPM Suprimentos single-record parameters** are not named in the docs. Paginate the full list instead, or ask IPM or the Prefeitura.
7. **Supplier CPF/CNPJ** is anonymized in IPM contrato and compra outputs. Join to PNCP (full CNPJ) or TCE-PR data on contract number and year.
8. **Saúde lists** (waiting lists, medicine stock) and **education** data appear only as HTML grids or external apps (`cascavel.app`, `netpronto`). Scraping would hit the same Cloudflare gate, so make a LAI request for bulk exports.
9. **State transfers to Cascavel:** the PR state web service ends in 2023 and does not identify the recipient. The state Downloads page needs a browser, and its format is unverified. Alternative: TCE-PR PIT Convênios and the municipality's own receitas (`/receitas` account codes for transferências estaduais).
10. **FNDE transfers:** SIGEF needs a captcha. Use CGU `recursos-recebidos` with an API key.

---

## 6. UNVERIFIED items (explicit)

- Diário Oficial PDF download URL pattern. No edition was downloaded.
- File formats inside the TCE-PR PIT `*_PIT_TodosArquivos.zip`. Only HEAD was checked.
- TCE-PR `cdIBGE` value for Cascavel (`04808`). Inferred from the São José dos Pinhais row.
- CGU API responses, since no key was available. The endpoint list comes from the official OpenAPI spec.
- Paraná state "Downloads" file URLs and formats, which are rendered by JavaScript.
- Câmara `api.camaracascavel.pr.gov.br`, which returned a database error on every call tested.
- IPM grid export formats (the "Dados Abertos" button) and the Inventário de Dados Abertos contents, both blocked by the Cloudflare human check.
- `https://www.cascavel.app/transparencia`, the Transferegov convênios (discricionárias) API, OpenDataSUS CKAN, the FNDE CKAN, the TSE municipality code for Cascavel, and the SIM-AM layout documents. None were opened or tested.
- LRF art. 48 §1 IV (LC 224/2025) as shown on planalto. Not cross-checked against the DOU.

---

## 7. Sources consulted (all accessed 2026-10-28)

**Municipal**
- https://www.cascavel.pr.gov.br/ → https://prefa.cascavel.pr.gov.br/
- https://prefa.cascavel.pr.gov.br/transparencia/
- https://prefa.cascavel.pr.gov.br/transparencia/mapa-do-site
- https://prefa.cascavel.pr.gov.br/transparencia/item/inventario-de-dados-abertos (Cloudflare-gated)
- https://prefa.cascavel.pr.gov.br/transparencia/item/api-de-dados-abertos-contabilidade
- https://prefa.cascavel.pr.gov.br/transparencia/item/api-de-dados-abertos-de-pessoal
- https://prefa.cascavel.pr.gov.br/transparencia/item/api-de-dados-abertos-suprimentos
- https://prefa.cascavel.pr.gov.br/transparencia/item/obras (Cloudflare-gated on query)
- https://prefa.cascavel.pr.gov.br/api/WCPDadosAbertos/{despesas,despesaRestos,despesasOrcadas,receitas,receitasOrcadas}
- https://prefa.cascavel.pr.gov.br/api/transparencia/dados-abertos/v1/wco/{licitacao,contrato}
- https://prefa.cascavel.pr.gov.br/api/transparencia-pessoal-funcionarios
- https://prefa.cascavel.pr.gov.br/diariooficial/edicao
- https://cascavel.atende.net/cidadao/acesso-informacao
- https://www.camaracascavel.pr.gov.br/
- https://www.camaracascavel.pr.gov.br/dados-abertos/
- https://www.camaracascavel.pr.gov.br/transparencia/acesso-a-informacao/sic/
- https://api.camaracascavel.pr.gov.br/a-camara/vereadores/ (error)
- https://camaracascavel.atende.net/api/... (IPM APIs on the Câmara host)
- https://sapl.cascavel.pr.leg.br/api/
- https://sapl.cascavel.pr.leg.br/api/materia/materialegislativa/
- https://sapl.cascavel.pr.leg.br/api/parlamentares/parlamentar/

**State**
- https://pit.tce.pr.gov.br/
- https://pit.tce.pr.gov.br/Dados/DadosConsulta/Consulta
- https://pit.tce.pr.gov.br/Dados/DadosConsulta/Consolidado
- https://pit.tce.pr.gov.br/Arquivos/2025_PIT_TodosArquivos.zip and 2026 (HEAD only)
- https://www.tce.pr.gov.br/transparencia-do-tce-pr/dados-abertos-tce-pr.htm
- https://www.tce.pr.gov.br/transparencia-do-tce-pr/dados-abertos-tce-pr/dados-abertos-acessar.htm
- https://servicos.tce.pr.gov.br/servicos/srv_dados_abertos.aspx
- https://servicos.tce.pr.gov.br/servicos/arquivos/dadosabertos/obrasmunicipais/obras_municipais_base_de_dados.csv (range request)
- https://servicos.tce.pr.gov.br/servicos/arquivos/dadosabertos/muraldelicitacoes/2026_mural_de_licitacoes_base_de_dados.csv (range request)
- https://www.transparencia.pr.gov.br/pte/home
- https://www.transparencia.pr.gov.br/pte/assunto/8/129
- https://www.transparencia.pr.gov.br/pte/pages/informacoes/webservices/webservices_despesas.jsf
- https://www.transparencia.pr.gov.br/pte/ws/despesas/listardespesa
- https://www.dados.pr.gov.br/

**Federal**
- https://apidatalake.tesouro.gov.br/ords/siconfi/tt/{entes,extrato_entregas,rreo,rgf,dca}
- https://pncp.gov.br/api/consulta/v1/contratacoes/publicacao
- https://pncp.gov.br/api/consulta/v1/contratos
- https://pncp.gov.br/api/pncp/v1/orgaos/76208867000107
- https://portaldatransparencia.gov.br/api-de-dados
- https://api.portaldatransparencia.gov.br/v3/api-docs
- https://api.transferegov.gestao.gov.br/transferenciasespeciais/ (spec and plano_acao_especial)
- https://api.transferegov.gestao.gov.br/fundoafundo/ (spec and plano_acao)
- https://servicodados.ibge.gov.br/api/v1/localidades/municipios/4104808
- https://servicodados.ibge.gov.br/api/v3/agregados/6579/... and /5938/...
- https://servicodados.ibge.gov.br/api/v1/pesquisas/indicadores/29171/resultados/4104808
- https://consultafns.saude.gov.br/#/consolidada and /recursos/consulta-consolidada/*
- https://opendatasus.saude.gov.br/api/3/action/package_list (HTML, not CKAN)
- https://www.fnde.gov.br/sigefweb/index.php/liberacoes (captcha)
- https://www.fnde.gov.br/dadosabertos/api/3/action/package_list (HTML)
- https://www.gov.br/inep/pt-br/areas-de-atuacao/pesquisas-estatisticas-e-indicadores/ideb/resultados
- https://www.gov.br/inep/pt-br/acesso-a-informacao/dados-abertos/microdados/censo-escolar
- https://dadosabertos.tse.jus.br/api/3/action/package_search and package_show (candidatos-2024, prestacao-de-contas-eleitorais-2024, codigos-oficiais-de-uf-e-municipios-segundo-o-tse-e-o-ibge)
- https://dados.gov.br/dados/api/publico/conjuntos-dados (401)
- https://dados.gov.br/v3/api-docs

**Legislation**
- https://www.planalto.gov.br/ccivil_03/_ato2011-2014/2011/lei/l12527.htm (LAI)
- https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp131.htm (LC 131/2009)
- https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp101.htm (LRF, compiled)
- https://www.planalto.gov.br/ccivil_03/_ato2019-2022/2021/lei/l14133.htm (Lei 14.133/2021)
