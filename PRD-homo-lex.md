# PRD: `homo-lex`

**Lex Advisor / Lex - Juris** · Setor Homo (Razão, Linguagem & Estratégia) · Matriz dos 24 Anciãos
**Versão:** 0.1 (rascunho) · **Status:** Em definição · **Responsável:** _(preencher)_

---

## 1. Resumo

O `homo-lex` é a VM de **inteligência jurídica** da plataforma: pesquisa e consulta de legislação, jurisprudência e documentos jurídicos do cliente, com respostas **sempre fundamentadas em fontes verificáveis**. Na v1 ele é uma **ferramenta de apoio a profissionais** (advogados, departamentos jurídicos, compliance), e não um "advogado virtual" para o público geral.

**Premissa de jurisdição:** Brasil (legislação federal, estadual e municipal, tribunais superiores e estaduais). Outras jurisdições ficam para fases futuras.

**Premissa de produto:** o diagrama o descreve como "Lex - Juris" (base de legislação e jurisprudência) e a pasta como "Lex Advisor" (consultor). Este PRD resolve a ambiguidade assim: **a v1 é a base de pesquisa (Juris)**, e o modo consultivo (Advisor) é uma camada posterior sobre ela. Ver Q1.

**Nome comercial sugerido:** Legal Research AI (nome interno "Lex" mantido como marca).

## 2. Objetivos

| # | Objetivo | Métrica de sucesso |
|---|---|---|
| O1 | Responder perguntas jurídicas com citação de fonte verificável | 100% das respostas com citação; precisão de citação ≥ 99% |
| O2 | Nunca inventar norma, artigo, súmula ou decisão | 0 citações inexistentes no conjunto de teste (golden set) |
| O3 | Considerar a **vigência** da norma na data relevante | Teste de vigência temporal ≥ 95% de acerto |
| O4 | Permitir corpus privado por cliente (contratos, pareceres, modelos) | Isolamento total entre tenants, validado em teste |
| O5 | Reduzir o tempo de pesquisa do profissional | Meta a medir em piloto (ex.: −50% no tempo de pesquisa) |
| O6 | Operar como produto independente | Instalável sozinha, com corpus público básico incluído |

## 3. Escopo

### Dentro do escopo (v1)
- Ingestão e indexação de **legislação** (Constituição, leis, decretos, códigos, resoluções)
- Ingestão e indexação de **jurisprudência** (súmulas, acórdãos, ementas) de fontes abertas
- Busca híbrida (lexical + semântica) e perguntas e respostas com citação obrigatória
- **Verificação de citações** (toda referência citada deve existir no corpus e ser aberta com link)
- Controle de **vigência e revogação** (consulta "como a lei estava em DD/MM/AAAA")
- Corpus privado por cliente (upload de contratos, pareceres, modelos internos)
- Revisão de contratos contra playbook do cliente (fase 3)
- Alertas de mudança legislativa e de publicações relevantes (fase 4)
- Trilha de auditoria completa de consultas e respostas

### Fora do escopo (v1)
- Aconselhamento jurídico direto ao público leigo (risco de exercício ilegal da advocacia e de responsabilidade)
- Elaboração automática e protocolo de petições
- Previsão de resultado de processos (jurimetria preditiva) → `aquila-predict`, com restrições
- Motor genérico de RAG corporativo → `homo-rag` (o Lex o reutiliza, ver seção 8)
- Doutrina e bases comerciais protegidas por direitos autorais, salvo licença formal
- Outras jurisdições

> **Motivo:** o maior risco do produto é a resposta jurídica errada com aparência de certeza. O escopo da v1 privilegia rastreabilidade em vez de abrangência.

## 4. Usuários e personas

| Persona | Necessidade |
|---|---|
| **Advogado / escritório** | Pesquisar normas e precedentes rapidamente, com fontes confiáveis para citar |
| **Departamento jurídico interno** | Consultar políticas, contratos e normas aplicáveis à empresa |
| **Compliance / DPO** (`homo-ethic`) | Verificar obrigações legais (LGPD, setoriais) e mudanças regulatórias |
| **Administrador do cliente** | Gerenciar usuários, corpus privado e permissões |
| **Agente de IA / demais VMs** | Consultar a base via API, com permissões limitadas e auditáveis |

## 5. Arquitetura

### 5.1 Base técnica
- **SO:** Ubuntu Server 24.04 LTS, instalação mínima (derivada do `taurus-seed`).
- **Execução:** serviços em containers, um por função, redes internas segregadas.
- **Inferência de LLM:** modelos de pesos abertos em servidor próprio (GPU em `leo-compute` ou local). Uso de API externa somente se permitido por política do cliente, sem envio de documentos confidenciais.
- **Configuração como código:** Ansible para o host, Compose versionado neste repositório.

### 5.2 Componentes

| Função | Componente recomendado | Alternativa |
|---|---|---|
| Parsing de PDF/DOCX | **Docling** | Unstructured, Apache Tika |
| Armazenamento relacional e vetorial | **PostgreSQL + pgvector** | Qdrant |
| Busca lexical (BM25, filtros, vigência) | **OpenSearch** | Meilisearch, Elasticsearch |
| Embeddings multilíngues | **BGE-M3** | multilingual-e5, modelos em português |
| Reranking | **bge-reranker** | Cohere-compatível local |
| LLM (geração) | Modelo aberto servido por **vLLM** ou **Ollama** | API externa sob política |
| Orquestração do RAG | **LlamaIndex** ou **Haystack** | LangChain, ou o motor do `homo-rag` |
| Extração de citações jurídicas | Regex/NER jurídico + **LeNER-Br** / **LegalNLP** | eyecite (adaptado) |
| Orquestração de ingestão | **Dagster** ou **Airflow** | cron + scripts (v1) |
| Interface | App web próprio ou **Open WebUI / LibreChat** (protótipo) | — |
| Avaliação contínua | **promptfoo** + **Ragas** | DeepEval |
| Observabilidade de LLM | **Langfuse** | OpenTelemetry |
| Autenticação | OIDC via `aquila-sentinel` (Keycloak) | — |
| Políticas e guardrails | OPA compartilhado com `homo-ethic` | — |

### 5.3 Diagrama lógico

```
 Fontes públicas ──► Coletores ──► Normalização ──► Chunking jurídico ──┐
 (Planalto, LexML,    (agendados)   (metadados,       (artigo, inciso,   │
  tribunais, DOU)                    vigência)          ementa, tese)    │
 Upload do cliente ────────────────────────────────────────────────────►┤
                                                                         ▼
                              PostgreSQL (metadados, versões)   OpenSearch (lexical)
                                       + pgvector (vetores) ◄──── embeddings
                                                   │
 Pergunta ──► Guardrails ──► Recuperação híbrida ──► Reranker ──► LLM
 (SSO)        (OPA)          (filtro: tenant,        │            │
                              vigência, jurisdição)  ▼            ▼
                                            Verificador de citações ──► Resposta + fontes
                                                   │
                                  Logs/auditoria ──► aquila-oculus
```

## 6. Requisitos funcionais

### 6.1 Corpus e ingestão (COR)
- **COR-01** Coleta agendada de legislação federal e das principais fontes estaduais.
- **COR-02** Coleta de jurisprudência de fontes abertas (súmulas, ementas e acórdãos de STF, STJ, TST e tribunais estaduais, conforme disponibilidade).
- **COR-03** Cada documento guarda metadados: tipo, órgão, número, data, ementa, hierarquia, status (vigente, revogado, alterado) e URL oficial da fonte.
- **COR-04** **Chunking por estrutura jurídica** (artigo, parágrafo, inciso, alínea; ementa, tese, dispositivo), não por tamanho fixo.
- **COR-05** **Versionamento temporal:** guardar as redações sucessivas de cada dispositivo com datas de vigência.
- **COR-06** Grafo de relações: altera, revoga, regulamenta, cita, súmula vinculada.
- **COR-07** Ingestão idempotente, com rastreio de origem e data de coleta de cada item.
- **COR-08** Respeito aos termos de uso e limites de taxa das fontes (ver seção 11).

### 6.2 Busca e resposta (QA)
- **QA-01** Busca híbrida (lexical + semântica) com filtros por jurisdição, órgão, data, vigência e tipo.
- **QA-02** Perguntas em linguagem natural com resposta fundamentada **apenas** no corpus recuperado.
- **QA-03** **Citação obrigatória:** cada afirmação jurídica aponta para dispositivo ou decisão, com link para a fonte.
- **QA-04** Quando não houver base suficiente, o sistema responde "não encontrei fundamento no corpus", em vez de especular.
- **QA-05** Consulta com data de referência ("vigência em 2019").
- **QA-06** Indicação de **conflitos e divergências** (decisões contrárias, norma posterior, revogação).
- **QA-07** Aviso fixo de que a resposta é apoio à pesquisa e não substitui parecer de advogado habilitado.

### 6.3 Verificação (VER)
- **VER-01** Um passo independente extrai todas as citações da resposta e confirma que existem no corpus e que o texto citado corresponde.
- **VER-02** Citação não verificada é removida ou marcada claramente, e a resposta registra o incidente.
- **VER-03** Métrica contínua de "citações alucinadas" exportada ao observability.

### 6.4 Corpus privado do cliente (PRI)
- **PRI-01** Upload de contratos, pareceres, políticas e modelos, em tenant isolado.
- **PRI-02** Dados do cliente **nunca** usados para treinar modelos nem compartilhados entre tenants.
- **PRI-03** Controle de acesso por documento e por grupo (via SSO/OPA).
- **PRI-04** Retenção e exclusão configuráveis, com exclusão comprovável (direito ao apagamento).

### 6.5 Análise de documentos (DOC), fase 3
- **DOC-01** Revisão de contrato contra playbook do cliente (cláusulas exigidas, proibidas, desvios).
- **DOC-02** Extração de cláusulas, prazos, partes e obrigações, com referência ao trecho original.
- **DOC-03** Sugestão de redação alternativa, sempre marcada como rascunho para revisão humana.

### 6.6 Monitoramento (MON), fase 4
- **MON-01** Alertas de novas normas e alterações por tema, jurisdição ou palavra-chave.
- **MON-02** Resumo de impacto sobre o corpus privado do cliente, com revisão humana.

### 6.7 API e integração (API)
- **API-01** API REST/OpenAPI para busca, pergunta, verificação e gestão de corpus.
- **API-02** Autenticação por token OIDC, com escopo por tenant.
- **API-03** Modo standalone sem dependência das demais VMs.

## 7. Requisitos não funcionais

| Área | Requisito |
|---|---|
| **Qualidade** | Conjunto de avaliação (golden set) montado e revisado por advogados; execução a cada mudança de modelo, prompt ou corpus; bloqueio de release se as metas regredirem |
| **Desempenho** | Resposta com citações em < 15 s (p95); busca sem geração em < 1 s (p95) |
| **Disponibilidade** | Meta de 99,5% na v1 |
| **Segurança** | SSO + MFA, criptografia em trânsito e em repouso, segredos no `aquila-sentinel`, sem credenciais em código |
| **Privacidade (LGPD)** | Dados do cliente em território nacional, minimização, base legal documentada, registro de operações, DPO definido |
| **Sigilo** | Tratamento de documentos sob sigilo profissional e segredo de justiça com isolamento reforçado |
| **Auditabilidade** | Log de pergunta, fontes recuperadas, resposta, usuário e versão do modelo, com retenção definida |
| **Recuperação** | Backup de bancos e índices em `taurus-storage`, RPO ≤ 24 h, RTO ≤ 8 h; índices reconstruíveis a partir do banco |
| **Portabilidade** | Entrega reprodutível (imagem + repositório) e instalação standalone documentada |
| **Licenciamento** | Registrar licença de modelos, bibliotecas e bases de dados antes da comercialização |

## 8. Integrações com outras VMs

| VM | Direção | Integração |
|---|---|---|
| `homo-rag` | Lex → RAG | Reutilizar o motor de ingestão e recuperação como serviço ou biblioteca; o Lex mantém corpus, esquema e regras jurídicas próprios |
| `aquila-sentinel` | Lex → Sentinel | SSO (OIDC), tenants e segredos |
| `homo-ethic` | Lex ↔ Ethic | Políticas OPA, guardrails e auditoria de uso; revisão de conformidade das respostas |
| `taurus-db` | Lex → DB | Opcional: PostgreSQL dedicado ao Lex (preferível a compartilhar) |
| `taurus-storage` | Lex → Storage | Armazenamento de documentos originais e backups |
| `leo-compute` | Lex → Compute | Inferência de LLM e embeddings em GPU, jobs de ingestão em massa |
| `leo-devops` | DevOps → Lex | Pipeline de CI/CD, execução da avaliação de qualidade como etapa de release |
| `aquila-oculus` | Lex → Oculus | Logs, métricas e alertas (somente envio) |
| `homo-bi` | Lex → BI | Indicadores de uso e qualidade, sem dados sensíveis |
| `taurus-portal` | Portal → Lex | Contratação, criação de tenant e painel do cliente |

## 9. Modo standalone (produto)
- Instalação independente, com corpus público básico já indexado.
- Interface web para busca, perguntas, verificação e upload de corpus privado.
- Assistente de configuração: domínio, SSO, escolha do modelo (local ou externo), política de retenção.
- Documentação de API e de implantação para o cliente.

## 10. Riscos e conformidade

| Risco | Impacto | Mitigação |
|---|---|---|
| **Alucinação** (norma ou decisão inexistente) | Crítico | Citação obrigatória, verificador independente, "não sei" como resposta válida, golden set |
| **Responsabilidade por conselho errado** | Crítico | Posicionar como apoio a profissionais, aviso fixo, termos de uso, revisão humana, seguro de responsabilidade a avaliar |
| **Exercício ilegal da advocacia** | Alto | Não oferecer consultoria direta a leigos; confirmar enquadramento com assessoria jurídica e com as diretrizes da OAB sobre IA |
| **Norma desatualizada ou revogada** | Alto | Controle de vigência, coleta agendada, data da última atualização visível em cada resposta |
| **Vazamento entre clientes** | Crítico | Isolamento por tenant em banco e índice, testes de segurança, filtros aplicados na recuperação |
| **Prompt injection via documentos** | Alto | Tratar documentos como dados, sanitização, guardrails, sem execução de ações a partir de conteúdo recuperado |
| **Direitos autorais e termos de uso das fontes** | Médio | Priorizar fontes oficiais abertas, licenciar bases comerciais, respeitar limites de coleta |
| **Dados pessoais e segredo de justiça** | Alto | Minimização, mascaramento quando possível, isolamento reforçado, DPO e registro de operações |
| **Custo de inferência** | Médio | Modelos menores com reranking, cache de consultas, limites por plano |

## 11. Fontes de dados (verificar termos de uso e formato atual)

| Fonte | Conteúdo |
|---|---|
| **Planalto**: https://www.planalto.gov.br/ccivil_03 | Legislação federal consolidada |
| **LexML Brasil**: https://www.lexml.gov.br | Portal e APIs de legislação e jurisprudência com identificadores padronizados (URN LEX) |
| **CNJ DataJud**: API pública do CNJ | Metadados processuais de tribunais |
| **STJ Dados Abertos**: https://dadosabertos.stj.jus.br | Acórdãos e decisões do STJ |
| **STF**: portal de jurisprudência | Súmulas e decisões |
| **Câmara e Senado (dados abertos)** | Proposições, tramitação, legislação |
| **Diário Oficial da União**: https://www.in.gov.br | Publicações oficiais |
| **Tribunais estaduais e regionais** | Jurisprudência (formatos e acessos variam; coleta caso a caso) |

## 12. Repositórios de referência (projetos existentes)

> **Verifique licença, manutenção e versão atual de cada projeto antes de adotar.** Os de domínio jurídico brasileiro são majoritariamente de pesquisa e podem estar parados ou com dados desatualizados.

### Domínio jurídico
- **LexML (organização)**: https://github.com/lexml (parsers e ferramentas do padrão LexML)
- **LegalNLP (português)**: https://github.com/felipemaiapolo/legalnlp (embeddings e modelos para texto jurídico em português)
- **LeNER-Br**: https://github.com/peluz/lener-br (corpus de entidades nomeadas em textos jurídicos brasileiros)
- **Associação Brasileira de Jurimetria (abjur)**: https://github.com/abjur (ferramentas de coleta e análise de dados judiciais)
- **Free Law Project, eyecite**: https://github.com/freelawproject/eyecite (extração e resolução de citações; referência de método, voltado ao direito americano)
- **Free Law Project, CourtListener**: https://github.com/freelawproject/courtlistener (arquitetura de plataforma aberta de jurisprudência)
- **LexGLUE**: https://github.com/coastalcph/lex-glue (benchmark de tarefas jurídicas, útil para avaliação)

### RAG, busca e documentos
- **LlamaIndex**: https://github.com/run-llama/llama_index
- **Haystack**: https://github.com/deepset-ai/haystack
- **LangChain**: https://github.com/langchain-ai/langchain
- **RAGFlow**: https://github.com/infiniflow/ragflow (RAG com foco em documentos complexos)
- **Docling**: https://github.com/docling-project/docling (parsing de PDF e DOCX)
- **Unstructured**: https://github.com/Unstructured-IO/unstructured
- **pgvector**: https://github.com/pgvector/pgvector
- **Qdrant**: https://github.com/qdrant/qdrant
- **OpenSearch**: https://github.com/opensearch-project/OpenSearch
- **FlagEmbedding (BGE-M3, rerankers)**: https://github.com/FlagOpen/FlagEmbedding

### Inferência, interface e qualidade
- **vLLM**: https://github.com/vllm-project/vllm
- **Ollama**: https://github.com/ollama/ollama
- **Open WebUI**: https://github.com/open-webui/open-webui
- **LibreChat**: https://github.com/danny-avila/LibreChat
- **Langfuse**: https://github.com/langfuse/langfuse
- **promptfoo**: https://github.com/promptfoo/promptfoo
- **Ragas**: https://github.com/explodinggradients/ragas

## 13. Roadmap de construção

| Fase | Entregas | Critério de aceite |
|---|---|---|
| **0. Base** | Ubuntu 24.04 mínimo (seed), Compose, PostgreSQL + pgvector, OpenSearch, SSO integrado | Serviços saudáveis; login via `aquila-sentinel` |
| **1. Corpus mínimo** | Coleta de legislação federal essencial (Constituição, códigos, LGPD), metadados e versionamento | Consulta "texto do art. X em data Y" correta |
| **2. Pesquisa com citação** | Busca híbrida, resposta com citação obrigatória, verificador de citações, golden set inicial | Precisão de citação ≥ 99% no golden set; 0 citações inexistentes |
| **3. Jurisprudência** | Ingestão de súmulas e decisões de STF/STJ/TST, grafo de relações | Respostas com precedentes verificáveis |
| **4. Corpus privado** | Upload por tenant, isolamento, retenção e exclusão | Teste de vazamento entre tenants aprovado |
| **5. Análise de contratos** | Playbooks, revisão e extração de cláusulas | Relatório de desvios validado por advogado |
| **6. Monitoramento** | Alertas de mudança legislativa e de publicações | Alerta correto em caso de teste |
| **7. Integração e standalone** | Integração com Portal, Oculus, Ethic; instalador e documentação | Instalação limpa em até 1 dia; fluxo de compra ponta a ponta |

## 14. Estrutura sugerida do repositório

```
homo-lex/
├── README.md
├── PRD.md                      # este arquivo
├── docs/
│   ├── adr/                    # decisões de arquitetura
│   ├── fontes.md               # fontes, termos de uso, frequência de coleta
│   ├── modelo-de-risco.md
│   └── runbooks/               # reindexação, restauração, troca de modelo
├── ansible/                    # host, hardening, containers
├── compose/                    # postgres, opensearch, llm, api, ui
├── ingest/
│   ├── collectors/             # planalto, lexml, stj, dou...
│   ├── parsers/                # chunking jurídico, vigência, relações
│   └── pipelines/
├── app/
│   ├── api/
│   ├── retrieval/
│   ├── verify/                 # verificador de citações
│   └── guardrails/
├── eval/
│   ├── golden-set/             # perguntas e respostas revisadas por advogados
│   └── promptfoo/
├── policies/                   # OPA/Rego
├── tests/
└── .gitignore                  # nunca versionar segredos, dumps ou documentos de clientes
```

**Regras do repositório:** nenhum segredo em texto puro; nenhum documento de cliente ou dado pessoal versionado; mudanças em `app/verify`, `app/guardrails` e `eval/` exigem revisão obrigatória; release bloqueado se o golden set regredir.

## 15. Questões em aberto

- **Q1.** A v1 é **pesquisa (Juris)** ou **consultor (Advisor)**? Este PRD assume pesquisa para profissionais. Confirmar.
- **Q2.** Público-alvo: escritórios, departamentos jurídicos ou ambos? Há ou haverá uso por leigos?
- **Q3.** Jurisdição inicial: apenas federal ou também estadual (quais estados) e municipal?
- **Q4.** Quais áreas do direito primeiro (trabalhista, tributário, civil, consumidor, LGPD)? Especializar aumenta a qualidade.
- **Q5.** Haverá licença de bases comerciais (jurisprudência, doutrina) ou apenas fontes abertas?
- **Q6.** LLM local obrigatório ou API externa permitida para clientes que aceitem?
- **Q7.** Há GPU disponível (`leo-compute`) ou a inferência será em CPU/API externa na fase inicial?
- **Q8.** Quem valida a qualidade? É necessário um advogado parceiro para montar e revisar o golden set.
- **Q9.** Qual a política de responsabilidade e o texto de termos de uso, e há seguro profissional previsto?
- **Q10.** O `homo-lex` será módulo do `homo-rag` (mesma base de código) ou serviço totalmente separado?

## 16. Definição de pronto (v1)

- [ ] Fases 0 a 4 concluídas e testadas
- [ ] Golden set revisado por advogado, com metas de citação atingidas
- [ ] Verificador de citações ativo em 100% das respostas
- [ ] Controle de vigência validado em casos de teste
- [ ] Isolamento entre tenants comprovado por teste
- [ ] SSO, logs e auditoria integrados (Sentinel e Oculus)
- [ ] Termos de uso, aviso de apoio à pesquisa e política de privacidade publicados
- [ ] Revisão de enquadramento regulatório (OAB, LGPD) por assessoria jurídica
- [ ] Backup e restauração testados
- [ ] Runbooks escritos

---
_Documento vivo: atualize a versão e registre decisões em `docs/adr/` a cada mudança relevante. Este documento é um plano técnico e não constitui orientação jurídica._
