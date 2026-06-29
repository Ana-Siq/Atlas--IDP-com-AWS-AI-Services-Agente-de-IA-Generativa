# Atlas--IDP-com- AI-Generativa
Projeto desenvolvido para o Hack2Hire- Hackathon da Escola da Nuvem em parceria com a AWS a partir de um desafio inspirado no mercado real, exigindo análise e proposta de solução. A aplicação visa otimizar o sistema de análise de documentos da seguradora de veículos, DocuSmart, centralizando e classificando documentos de solicitação de sinistros e reduzindo falhas e retrabalho para aumentar a eficiência operacional.

# DocuSmart Seguros — IDP Pipeline & Intelligent RAG Agent 🚀

Este repositório contém a implementação da arquitetura 100% *serverless* de **Processamento Inteligente de Documentos (IDP)** e **Geração Aumentada de Recuperação (RAG)** desenvolvida para a DocuSmart Seguros. A solução automatiza a ingestão, classificação, extração de dados e análise preditiva/consultiva de pacotes de sinistros contendo PDFs, imagens e documentos escaneados, reduzindo o tempo de triagem de ~60 minutos para poucos segundos.

## 🏗️ Visão Geral da Arquitetura

A solução é orientada a eventos (EDA) e utiliza orquestração robusta baseada em máquinas de estados. O ecossistema mapeado na arquitetura divide-se em duas grandes frentes:
1. **Pipeline de IDP (Ingestão e Processamento):** Acionado automaticamente via upload de sinistros, realiza a extração multi-modelo de texto, metadados e persistência estruturada.
2. **Pipeline do Agente Virtual (Chat & RAG):** Disponibiliza uma interface conversacional inteligente para funcionários consultarem o histórico de operações e documentos em linguagem natural.
   <img width="1032" height="639" alt="image" src="https://github.com/user-attachments/assets/a895220f-39e2-4771-8a9d-742942c0c850" />

---

## 🧭 Fluxo de Dados e Componentes

### 1. Ingestão e Disparo
* **Cliente:** Realiza o upload do pacote de documentos originais no bucket S3 `atlas-site-upload`.
* **API Gateway (`atlas-trigger`) & Lambda (`atlas-api-handler`):** O upload aciona o endpoint que invoca a função Handler. Ela valida a requisição, trata payloads iniciais e consolida os documentos brutos para o bucket unificado `atlas-sinistros-hackathon`.
* **Lambda (`atlas-trigger`):** O salvamento no bucket de sinistros dispara este gatilho responsável por iniciar formalmente a execução da máquina de estados no AWS Step Functions.

### 2. Bloco Processador de IDP (`atlas-idp-pipeline`)
A máquina de estados isola o fluxo de inteligência de dados garantindo resiliência e concorrência:
* **Lambda `atlas-classify`:** Atua na camada de orquestração analítica, disparando paralelamente requisições para três motores complementares da AWS:
  * **Amazon Textract:** Extração óptica de caracteres (OCR) robusta de layouts, formulários, tabelas e dados brutos de notas fiscais/documentos de identidade.
  * **Amazon Comprehend:** Processamento de Linguagem Natural (NLP) para identificação automática de entidades e classificação semântica do tipo de documento (Boletim de ocorrência, Laudo, Nota Fiscal, CNH/RG).
  * **Amazon Bedrock + Nova Lite:** Modelo de fundação inteligente de alta eficiência para validação multimodal e entendimento contextual preliminar.
* **Lambda `atlas-process`:** Atua na consolidação do processamento. Consolida as saídas estruturadas dos três motores, valida as regras de negócio de seguros (regras antifraude/completude), salva o payload estruturado final no banco de dados **Amazon DynamoDB (`atlas-operacoes`)** e deposita os textos limpos estruturados no bucket S3 `atlas-vectors`.

### 3. Agente de IA Conversacional (SAC & Busca Inteligente via RAG)
* **Funcionário & S3 `atlas-chat`:** Os analistas e atendentes do SAC interagem através de um front-end estático hospedado de maneira segura no bucket `atlas-chat`.
* **Lambda `atlas-agent` (Orquestrador RAG):** As mensagens passam pelo API Gateway e invocam o agente inteligente, que coordena as seguintes ações em tempo real:
  1. Varre o histórico do sinistro diretamente no **Amazon DynamoDB (`atlas-operacoes`)**.
  2. Consulta e gera vetores contextuais a partir do bucket **S3 Vectors (`atlas-vectors`)**.
  3. Realiza a conversão do prompt dinâmico usando o **Amazon Titan Embeddings**.
  4. Interage com os LLMs de última geração no **Amazon Bedrock** (completando o ciclo RAG) e entrega uma resposta rica e contextualizada em linguagem natural para o funcionário em milissegundos.

---

## 📂 Estrutura de Diretórios Sugerida

```text
.
├── src/
│   ├── lambdas/
│   │   ├── atlas-api-handler/     # Manipulação de uploads e validação de API
│   │   ├── atlas-trigger/         # Gatilho de inicialização do Step Functions
│   │   ├── atlas-classify/        # Integração paralela com Textract, Comprehend e Bedrock
│   │   ├── atlas-process/         # Consolidação, lógica de negócio e salvamento
│   │   └── atlas-agent/           # Agente GenAI com suporte a RAG e Titan Embeddings
│   └── statics/
│       └── chat-ui/               # Artefatos da interface do funcionário (S3 atlas-chat)
├── statemachine/
│   └── idp-pipeline.asl.json      # Definição do AWS Step Functions (ASL)
├── iac/
│   └── serverless.yml             # Definição de infraestrutura como código (CloudFormation/SAM)
└── README.md

```

---

## 🗄️ Modelagem de Dados (Amazon DynamoDB — `atlas-operacoes`)

A tabela utiliza um padrão de indexação otimizado (*Single-Table Design*) para associar múltiplos documentos a um único registro de sinistro:

| Atributo | Tipo | Descrição | Exemplo |
| --- | --- | --- | --- |
| **PK (Partition Key)** | String | ID unificado do Sinistro | `SINISTRO#2026_DOCUSMART_089` |
| **SK (Sort Key)** | String | Identificador do Artefato / Metadado | `DOC#LAUDO#001` ou `METADATA` |
| **StatusProcessamento** | String | Estado do ciclo de vida do documento | `CLASSIFIED`, `PROCESSED`, `FAILED` |
| **Classificacao** | String | Tipo de documento identificado pela IA | `Boletim de Ocorrência` / `Nota Fiscal` |
| **DadosExtraidos** | Map (JSON) | Payload de dados estruturados extraídos | `{ "valor_total": 1500.00, ... }` |
| **ConfidenceScore** | Number | Score de acurácia da extração | `0.985` |

---

## 🚀 Como Implantar (Deploy)

### Pré-requisitos

* AWS CLI configurado com credenciais adequadas.
* Node.js (v18+) e NPM instalados.
* Modelos **Amazon Titan Embeddings** e **Amazon Bedrock (Nova Lite/Claude)** explicitamente ativados no console da sua conta AWS na região escolhida.

### Executando o Deploy da Infraestrutura

1. Clone o repositório:
```bash
git clone [https://github.com/seu-usuario/atlas-idp-pipeline.git](https://github.com/seu-usuario/atlas-idp-pipeline.git)
cd atlas-idp-pipeline

```
2. Instale as dependências gerais e das funções Lambda:
```bash
npm run install:all

```

3. Execute o deploy do ecossistema via Serverless Framework (ou framework de preferência mapeado em `/iac`):
```bash
serverless deploy --stage prod --region us-east-1

```
---

## 📊 Monitoramento e Auditoria

* **AWS CloudWatch Logs:** Logs estruturados por request com retenção automática de 30 dias.
* **AWS Step Functions Visualizer:** Auditoria passo a passo de transições de estado para validação regulatória de seguros (SUSEP).
* **AWS X-Ray:** Rastreamento ativo de latência para chamadas externas dos modelos fundacionais do Amazon Bedrock.

```

Equipe desenvolvedora:
* **Ana Paula da Silva Siqueira**
* **Bruno Trindade**
* **Daniel Landi**
* **Livia Calderan Alves**
* **Melina Nascimento França**
* **Pedro Henrique Castro de Souza**
* **Samuel Brandão Maia Lima**




```
