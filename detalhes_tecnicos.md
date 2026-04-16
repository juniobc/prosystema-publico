# Detalhes Técnicos — Plataforma Saúde Inteligente

## 1. Visão Geral

Plataforma SaaS multi-tenant para gestão da saúde pública municipal. Cada prefeitura opera em seu próprio schema no banco PostgreSQL, isolado por tenant via header HTTP `X-Tenant-ID`.

A solução é estruturada como um ecossistema tecnológico único com módulos funcionais integrados, cobrindo desde a atenção primária até a alta complexidade hospitalar, regulação, vigilância e inteligência de dados.

---

## 2. Stack Tecnológica

| Camada | Tecnologia |
|---|---|
| Backend (API) | Python / FastAPI — porta 8002, retorna JSON |
| Frontend (SPA) | React / Vite — porta 5173 |
| Banco de Dados | PostgreSQL (multi-schema por município) |
| Cache | Redis |
| Autenticação | JWT (OAuth 2.0), middlewares de Auth + Authorization + Tenant |
| IA / Assistente | Google Gemini 2.0 Flash, LangGraph, MCP (assistente AIRA) |
| Automação de Fluxos | n8n (webhooks HTTP, orquestração de agentes) |
| Containerização | Docker |
| Certificado Digital | PFX institucional (integração CADSUS, RNDS) |
| Documentação API | Swagger automático via FastAPI (`/docs`) |

### 2.1. Stack de IA, Machine Learning e Modelos

| Categoria | Tecnologia / Modelo |
|---|---|
| LLM Principal | Google Gemini 2.0 Flash (`langchain_google_genai`) |
| Orquestração de Agentes | LangGraph, LangChain Core, LangChain Community |
| Protocolo de Ferramentas | Model Context Protocol — MCP (`mcp`, `langchain_mcp_adapters`) |
| Observabilidade IA | LangSmith (tracing de execuções e debugging) |
| Séries Temporais | Prophet (Facebook) — previsão de demanda, sazonalidade e feriados |
| Séries Temporais (alternativo) | SARIMAX (`statsmodels`) — modelos SARIMA para previsão |
| Clusterização | KMeans (`scikit-learn`) — agrupamento de pacientes por perfil de custo/risco |
| Classificação | RandomForestClassifier, GradientBoostingClassifier, LogisticRegression (`scikit-learn`) |
| Detecção de Anomalias | IsolationForest (`scikit-learn`) — identificação de outliers em dados assistenciais |
| Pré-processamento | StandardScaler, LabelEncoder, PCA, PowerTransformer (`scikit-learn`) |
| Embeddings Semânticos | Sentence Transformers (`all-MiniLM-L6-v2`) — busca semântica de páginas |
| Busca Vetorial | FAISS (Facebook AI Similarity Search) — indexação e recuperação vetorial |
| Deep Learning (framework) | PyTorch (`torch`) — base para modelos Transformers |
| Transformers | HuggingFace Transformers (`transformers`) — modelos de linguagem e embeddings |
| LangChain HuggingFace | `langchain_huggingface` — integração LangChain com modelos HuggingFace |
| Dados e Análise | Pandas, NumPy, Matplotlib, Seaborn |
| Interoperabilidade FHIR | `fhir.resources`, `fhirpy`, `fhirclient` — modelos oficiais HL7 |

---

## 3. Arquitetura Multi-Tenant

- Cada município possui um **schema isolado** no PostgreSQL
- O tenant é identificado pelo header `X-Tenant-ID` ou resolvido via subdomínio no Origin (ex: `municipio.localhost`)
- Middleware `TenantMiddleware` intercepta todas as requisições e injeta o contexto do município
- Tabelas auxiliares compartilhadas ficam no schema `auxiliares` (sexo, raça/cor, nacionalidade, estados, cidades, logradouros, bairros)

---

## 4. Estrutura Modular (4 Pilares)

| Módulo | Nome | Função |
|---|---|---|
| 01 | Inteligência de Dados | Centralização, tratamento e consolidação de dados com painéis de BI, IA preditiva e análises parametrizáveis |
| 02 | Central de Regulação | Gestão de fluxos regulatórios — consultas, exames, cirurgias, leitos (ambulatorial e hospitalar) |
| 03 | Prontuário Eletrônico Integrado | Barramento de interoperabilidade FHIR — consolida dados clínicos de múltiplos sistemas de origem |
| 04 | Prontuário Assistencial | Registro direto de atendimentos ambulatoriais especializados, pré-hospitalares e hospitalares |

---

## 5. Microserviços da API

A API é composta por mais de 20 microserviços independentes, todos registrados no FastAPI principal:

| Microserviço | Domínio |
|---|---|
| `auth` | Autenticação e gestão de tokens JWT |
| `microservicoAcesso` | Controle de acesso e permissões por tenant |
| `microservicoCore` | Funcionalidades transversais do sistema |
| `microservicoAPS` | Atenção Primária à Saúde |
| `microservicoAES` | Atenção Especializada em Saúde |
| `microservicoOCI` | Oferta de Cuidado Integrado (regulação + agendamento) |
| `microservicoRAS` | Rede de Atenção à Saúde |
| `microservicoPPASS` | Programação Pactuada e Assistencial |
| `microservicoASF` | Assistência Farmacêutica |
| `microservicoVIGS` | Vigilância em Saúde (epidemiológica + sanitária) |
| `microservicoSisrega` | Integração com SISREG (regulação nacional) |
| `microservicoInternacao` | Gestão de internações e AIH |
| `microServicoLeitos` | Gestão de leitos hospitalares |
| `microservicoFaturamento` | Faturamento SUS (SIA/SIH) |
| `microservicoFinanceiro` | Gestão financeira e repasses |
| `microservicoMonitoramento` | Monitoramento operacional e alertas |
| `microServicoFhir` | Interoperabilidade HL7 FHIR |
| `microservicoAira` | Assistente virtual com IA (Gemini + LangGraph) |
| `microservicoScraping` | Coleta automatizada de dados externos |
| `microservicoMinisteriosaude` | Integração com sistemas do Ministério da Saúde |
| `microservicoIbge` | Dados demográficos e territoriais do IBGE |
| `dataview` | Visualização e consulta de dados analíticos |

---

## 6. Interoperabilidade FHIR HL7

### 6.1. Visão Geral

O Módulo 03 (Prontuário Eletrônico Integrado) atua como **barramento de interoperabilidade**, consumindo registros clínicos de múltiplos sistemas de origem (PEC e-SUS APS, sistemas hospitalares, laboratórios, farmácias, sistemas legados) e consolidando-os em formato HL7 FHIR padronizado.

Não se caracteriza como sistema de registro direto — é uma camada de integração e consolidação.

### 6.2. Padrão Adotado

- **HL7 FHIR R4/R5** para estruturação e intercâmbio de dados clínicos
- **APIs RESTful** com métodos GET/POST/PUT/DELETE
- Transmissão via **HTTPS/TLS 1.2+**
- Autenticação via **OAuth 2.0 + JWT**
- Biblioteca Python: `fhir.resources` (modelos oficiais HL7)

### 6.3. Arquitetura do Microserviço FHIR

```
microServicoFhir/
├── route.py                    # Endpoints FastAPI (rotas FHIR)
├── schemas.py                  # Marshmallow schemas (validação de entrada)
├── services/
│   ├── base.py                 # BaseFHIRService — classe abstrata reutilizável
│   ├── patient.py              # FhirPatientService (search + create)
│   └── comuns.py               # Consultas em tabelas auxiliares
├── mappers/
│   └── patient.py              # PatientMapper (db_to_fhir + fhir_to_db)
└── utils/
    └── operation_outcome.py    # Factory de OperationOutcome padronizado
```

### 6.4. Endpoints FHIR

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/fhir/Patient` | FHIR Search — busca de pacientes com paginação |
| `POST` | `/fhir/Patient` | FHIR Create — criação de paciente |

### 6.5. Resources FHIR

Os resources previstos pelo Termo de Referência para o barramento de interoperabilidade:

- **Patient** — Dados demográficos do paciente
- **Encounter** — Registros de atendimento/encontro clínico
- **Observation** — Observações clínicas e resultados
- **DiagnosticReport** — Laudos e relatórios diagnósticos
- **Procedure** — Procedimentos realizados
- **MedicationRequest** — Prescrições medicamentosas
- **Condition** — Condições clínicas e diagnósticos
- **Composition** — Documentos clínicos compostos
- **Bundle** — Agrupamento de resources (searchset, transaction)
- **Practitioner** — Profissionais de saúde
- **Organization** — Estabelecimentos e organizações
- **Location** — Localização física das unidades
- **OperationOutcome** — Respostas de erro padronizadas

### 6.6. Parâmetros de Busca FHIR (Patient)

| Parâmetro | Descrição |
|---|---|
| `_id` | ID interno do paciente |
| `name` | Busca ampla por nome (oficial + social) |
| `given` | Nome próprio |
| `family` | Sobrenome |
| `birthdate` | Data de nascimento (prefixos: eq, gt, lt, ge, le, ne) |
| `gender` | Sexo (male, female, other, unknown) |
| `identifier` | CNS ou CPF com system (ex: `https://sus.saude.gov.br/cns\|123456`) |
| `address` | Busca por logradouro, bairro ou cidade |
| `address-city` | Cidade |
| `address-postalcode` | CEP |
| `_sort` | Ordenação (suporta múltiplos campos, prefixo `-` para DESC) |
| `_count` | Quantidade por página (1-100, padrão 20) |
| `_offset` / `_page` | Paginação |

### 6.7. Identificadores Brasileiros

| Identificador | System FHIR | CodeSystem |
|---|---|---|
| CNS (Cartão Nacional de Saúde) | `https://sus.saude.gov.br/cns` | `v2-0203` → code `CNS` |
| CPF (Cadastro de Pessoa Física) | `https://www.gov.br/cpf` | `v2-0203` → code `CPF` |

### 6.8. Extensions Brasileiras

| Extension | URL | Uso |
|---|---|---|
| Nacionalidade | `http://hl7.org/fhir/StructureDefinition/patient-nationality` | Código da nacionalidade via valueCodeableConcept |
| Raça/Cor | `http://hl7.org/fhir/StructureDefinition/patient-race` | Código da raça via valueCoding |
| Etnia | `http://hl7.org/fhir/StructureDefinition/patient-ethnicity` | Código da etnia via valueCoding |

### 6.9. Bundle e Paginação

O Bundle FHIR retornado nas buscas segue o padrão `searchset` com links completos de navegação:

- `self` — URL da página atual
- `first` — Primeira página
- `last` — Última página
- `next` — Próxima página (quando disponível)
- `previous` — Página anterior (quando disponível)

Inclui `total` (contagem real do banco) e `timestamp` da consulta.

### 6.10. Tratamento de Erros (OperationOutcome)

Todos os erros são retornados como `OperationOutcome` FHIR padronizado, contendo:
- `severity` (error, warning, information)
- `code` (invalid, duplicate, conflict, exception)
- `details.text` (mensagem descritiva)
- `diagnostics` (detalhes técnicos quando aplicável)

---

## 7. Inteligência Artificial — AIRA

### 7.1. Visão Geral

O **AIRA** (Assistente Inteligente para Recursos na Área de Saúde) é o motor de IA da plataforma, implementado como microserviço dentro da API. Utiliza o **Google Gemini** como LLM principal, orquestrado via **LangGraph** e integrado ao ecossistema de automação **n8n** via webhooks.

### 7.2. Motor e Orquestração

| Componente | Tecnologia |
|---|---|
| LLM Principal | Google Gemini 2.0 Flash |
| Orquestração de Agentes | LangGraph (grafo de estados) |
| Protocolo de Ferramentas | Model Context Protocol (MCP) |
| Automação de Fluxos | n8n (webhooks HTTP) |
| Embeddings Semânticos | Sentence Transformers (`all-MiniLM-L6-v2`) |
| Busca Vetorial | FAISS (Facebook AI Similarity Search) |
| Observabilidade | LangSmith (tracing de execuções) |

### 7.3. Agentes Especializados

O sistema carrega instruções de agentes dinamicamente a partir de arquivos `.md`, permitindo especialização por domínio sem alterar código:

| Agente | Domínio | Função |
|---|---|---|
| `agente_geral` | Transversal | Assistente genérico para navegação e dúvidas gerais |
| `agente_oci` | OCI (Oferta de Cuidado Integrado) | Especialista em regulação, solicitações e agendamentos |
| `agente_aps` | APS (Atenção Primária) | Especialista em indicadores, fichas CDS e monitoramento da APS |

### 7.4. Skills e Tools (MCP)

O AIRA expõe ferramentas via Model Context Protocol que o LLM pode invocar durante a conversa:

| Tool | Categoria | Função |
|---|---|---|
| `buscar_dados_oci` | Consulta ao Banco | Busca dados de solicitações, agendamentos e regulação no módulo OCI |
| `criterios_odontologicos` | Regras de Negócio | Gera critérios e protocolos odontológicos para apoio à decisão |
| `relatorio_gestantes` | Geração de Relatório | Gera relatório Markdown de acompanhamento de gestantes por microárea (template Jinja2 + dados) |
| `retriver_vector_store` | RAG (Retrieval) | Busca semântica em base vetorial para recuperação de documentos e contexto |
| `rnds` | Integração RNDS | Ferramentas de consulta e envio de dados para a Rede Nacional de Dados em Saúde |
| `frontend_actions` | Ações no SPA | Comandos que o agente envia ao frontend (navegar para página, abrir modal, filtrar dados) |

### 7.5. Capacidades do Assistente

- Busca semântica de páginas — encontra a tela correta do sistema a partir de perguntas em linguagem natural
- Análise de conteúdo de páginas — interpreta cards, gráficos e tabelas dos painéis de BI
- Contexto de navegação — responde perguntas sobre páginas visitadas anteriormente
- Comparação de dados entre páginas — calcula diferenças absolutas e percentuais entre métricas
- Extração automática de insights — identifica proporções, rankings e variações estatísticas nos dados
- Geração de relatórios sob demanda — produz relatórios Markdown a partir de templates e dados do banco
- RAG (Retrieval-Augmented Generation) — chunking por seções, indexação vetorial e recuperação contextual

### 7.6. Integração com n8n

O chat do AIRA se comunica com workflows n8n via webhooks HTTP:

- Webhook principal para chat em produção
- Webhook beta para modo avançado com seleção de agente e contexto de relatório
- Sessões persistidas em banco (`inteligencia_artificial.chat_session` / `chat_message`)
- Suporte a sessões temporárias (modo demonstração sem autenticação)
- Extração de `tool_calls` e `intermediateSteps` do LangChain para ações no frontend

---

## 8. Modelos de Predição e Clusterização

### 8.1. Previsão Temporal (Séries Históricas)

A plataforma utiliza modelos de séries temporais para previsão de demanda assistencial:

| Modelo | Dados de Entrada | Saída | Aplicação |
|---|---|---|---|
| Previsão de Solicitações OCI | Data, atendimentos, feriados | Previsão de atendimentos até 180 dias com margem de segurança | Planejamento da oferta de cuidado integrado |
| Previsão de Ocupação de Leitos UTI | Data, quantidade de leitos, feriados, sazonalidades, dados SIH-SUS | Previsão da taxa de ocupação de leitos | Gestão hospitalar e planejamento de capacidade |
| Previsão de Atendimentos em Carretas | Dados históricos de atendimentos móveis | Projeção de demanda para unidades móveis | Logística e roteirização de carretas de saúde |

Técnicas utilizadas: **Prophet** (Facebook) para decomposição de séries temporais com sazonalidade e feriados.

### 8.2. Clusterização de Pacientes

| Modelo | Dados de Entrada | Saída | Aplicação |
|---|---|---|---|
| Clustering de Pacientes Alto Custo | Idade, sexo, CEP, renda estimada, CID-10 (24 meses), internações, custos por nível de atenção (12 meses), consultas APS vs. especializada, comorbidades, medicamentos | Agrupamento por perfil de custo e risco | Identificação de pacientes de alto custo para gestão proativa e redução de internações evitáveis |

Funcionalidades do painel de clusterização:
- Distribuição de pacientes por cluster
- Custos e estatísticas por cluster
- Mapa georreferenciado de pacientes (pontos por CEP)
- Estatísticas por região
- Top pacientes por score de risco ou custo total
- Exportação CSV com filtros
- Detalhamento individual com procedimentos

### 8.3. Classificação (Modelos Preditivos)

| Modelo | Dados de Entrada | Saída | Aplicação |
|---|---|---|---|
| Previsão de Faltas em Consultas (Absenteísmo) | 25+ variáveis: idade, sexo, distância, histórico de faltas, especialidade, dia/turno, condição climática, feriados, canal de agendamento, confirmação | Probabilidade de falta do paciente (0-100%) | Overbooking inteligente, chamada ativa preventiva, identificação de perfil faltoso |
| Previsão de Custo UTI | Dados do paciente na entrada da UTI, diagnósticos, procedimentos realizados | Estimativa de custo total da internação | Planejamento financeiro hospitalar e gestão de recursos |

Funcionalidades do painel de absenteísmo:
- Lista de consultas com probabilidade de falta e nível de risco (Alto/Médio/Baixo)
- Resumo estatístico (total consultas, taxa média, economia potencial)
- Série temporal de probabilidade média
- Top especialidades por taxa de falta
- Heatmap dia da semana × turno
- Filtros por unidade, especialidade, tipo de atenção (APS/AE), período e status de contato

### 8.4. APIs de IA Disponíveis

| Endpoint | Método | Descrição |
|---|---|---|
| `/publico/aira/previsoes/atendimentos` | GET | Previsão temporal de solicitações OCI |
| `/publico/aira/previsoes/carretas` | GET | Previsão de atendimentos em carretas |
| `/publico/aira/previsoes/leitos` | GET | Previsão de ocupação de leitos (múltiplas granularidades) |
| `/publico/aira/previsoes/leitos/granularidades` | GET | Catálogo de granularidades disponíveis |
| `/publico/aira/api/summary` | GET | Resumo de pacientes alto custo (com filtros) |
| `/publico/aira/api/patients` | GET | Lista paginada de pacientes alto custo |
| `/publico/aira/api/patients/{id}` | GET | Detalhamento individual do paciente |
| `/publico/aira/api/clusters/distribution` | GET | Distribuição de pacientes por cluster |
| `/publico/aira/api/clusters/costs` | GET | Custos por cluster |
| `/publico/aira/api/clusters/stats` | GET | Estatísticas por cluster |
| `/publico/aira/api/map/points` | GET | Pontos georreferenciados para mapa |
| `/publico/aira/api/regions/stats` | GET | Estatísticas por região |
| `/publico/aira/api/export.csv` | GET | Exportação CSV de pacientes |
| `/publico/aira/absenteismo/consultas` | GET | Lista de consultas com probabilidade de falta |
| `/publico/aira/absenteismo/stats/summary` | GET | Resumo estatístico de absenteísmo |
| `/publico/aira/absenteismo/stats/temporal` | GET | Série temporal de probabilidade média |
| `/publico/aira/absenteismo/stats/especialidades` | GET | Top especialidades por taxa de falta |
| `/publico/aira/absenteismo/stats/heatmap` | GET | Heatmap dia × turno |
| `/publico/aira/chat-n8n` | POST | Chat com assistente AIRA (via n8n) |

---

## 9. Integração com RNDS

A plataforma é projetada para integração com a **Rede Nacional de Dados em Saúde (RNDS)** do Ministério da Saúde:

- **Endpoint RAC** (Registro de Atendimento Clínico) para envio e consulta de registros
- **Modelo RIRA** (Regulação Assistencial) para dados de regulação
- **Credenciamento** via certificado digital institucional (PFX)
- **Schemas compatíveis:** Composition, Bundle, Encounter, Practitioner, Organization, Location
- **Consulta CADSUS** para validação cadastral (CNS, CPF, nome, data de nascimento)

---

## 10. Painéis de Inteligência (Módulo 01)

Áreas cobertas pelos painéis analíticos:

| Área | Escopo |
|---|---|
| APS | Indicadores C1-C7 (Portaria 3.493/2024), vacinas, glosas, DCNT |
| Vigilância | Epidemiológica, sanitária, georreferenciamento |
| Especializada (MAC) | Prospecção assistencial, equipamentos, teto financeiro |
| Hospitalar | Planejamento de leitos, perfil de internações, taxas hospitalares, ICSAPS |
| Regulação | Controle de exames, consultas, cirurgias, internações, tratamento de fila |
| Gestão | Auditoria FPO, repasses FNS, indicadores IEGM |

---

## 11. Segurança e Conformidade

- **LGPD** (Lei 13.709/2018) — controle de acesso granular, consentimento, auditoria
- **HTTPS** exclusivo para todas as comunicações
- **JWT** com expiração configurável para tokens de acesso
- **Middlewares** de autenticação, autorização e isolamento de tenant
- **Certificado digital** para integrações com sistemas governamentais
- **SLA de disponibilidade:** 99,0% mensal
- **Logs e trilhas de auditoria** para rastreabilidade de transações

---

## 12. Integrações Externas

| Sistema | Tipo |
|---|---|
| RNDS (Rede Nacional de Dados em Saúde) | FHIR R4, endpoint RAC |
| e-SUS APS (PEC) | Fichas CDS, dados de produção |
| CADSUS / CadWeb | Validação cadastral (CNS/CPF) |
| CNES | Estabelecimentos, profissionais, equipes |
| SIA/SIH (TABWIN) | Produção ambulatorial e hospitalar |
| SISREG | Regulação nacional |
| SIGTAP | Tabela de procedimentos SUS |
| IBGE | Dados demográficos e territoriais |
| FNS | Repasses financeiros |
