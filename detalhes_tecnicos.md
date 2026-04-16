# Detalhes Técnicos — Plataforma Saúde Inteligente

## 1. Visão Geral

Plataforma SaaS multi-tenant para gestão da saúde pública municipal, desenvolvida para o CIM/MG (Consórcio Intermunicipal Multifinalitário de Minas Gerais). Cada prefeitura opera em seu próprio schema no banco PostgreSQL, isolado por tenant via header HTTP `X-Tenant-ID`.

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
| IA / Assistente | Google Gemini, LangGraph, MCP (assistente AIRA) |
| Containerização | Docker |
| Certificado Digital | PFX institucional (integração CADSUS, RNDS) |
| Documentação API | Swagger automático via FastAPI (`/docs`) |

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

## 7. Integração com RNDS

A plataforma é projetada para integração com a **Rede Nacional de Dados em Saúde (RNDS)** do Ministério da Saúde:

- **Endpoint RAC** (Registro de Atendimento Clínico) para envio e consulta de registros
- **Modelo RIRA** (Regulação Assistencial) para dados de regulação
- **Credenciamento** via certificado digital institucional (PFX)
- **Schemas compatíveis:** Composition, Bundle, Encounter, Practitioner, Organization, Location
- **Consulta CADSUS** para validação cadastral (CNS, CPF, nome, data de nascimento)

---

## 8. Painéis de Inteligência (Módulo 01)

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

## 9. Segurança e Conformidade

- **LGPD** (Lei 13.709/2018) — controle de acesso granular, consentimento, auditoria
- **HTTPS** exclusivo para todas as comunicações
- **JWT** com expiração configurável para tokens de acesso
- **Middlewares** de autenticação, autorização e isolamento de tenant
- **Certificado digital** para integrações com sistemas governamentais
- **SLA de disponibilidade:** 99,0% mensal
- **Logs e trilhas de auditoria** para rastreabilidade de transações

---

## 10. Integrações Externas

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
