# Atlas--IDP-com- AI-Generativa
Projeto desenvolvido para o Hack2Hire- Hackathon da Escola da Nuvem em parceria com a AWS a partir de um desafio inspirado no mercado real, exigindo análise e proposta de solução. A aplicação visa otimizar o sistema de análise de documentos da seguradora de veículos, DocuSmart, centralizando e classificando documentos de solicitação de sinistros e reduzindo falhas e retrabalho para aumentar a eficiência operacional.

# DocuSmart Seguros — IDP Pipeline & Intelligent RAG Agent 🚀

Este repositório contém a implementação da arquitetura 100% *serverless* de **Processamento Inteligente de Documentos (IDP)** e **Geração Aumentada de Recuperação (RAG)** desenvolvida para a DocuSmart Seguros. A solução automatiza a ingestão, classificação, extração de dados e análise preditiva/consultiva de pacotes de sinistros contendo PDFs, imagens e documentos escaneados, reduzindo o tempo de triagem de ~60 minutos para poucos segundos.

## 🏗️ Visão Geral da Arquitetura

A solução é orientada a eventos (EDA) e utiliza orquestração robusta baseada em máquinas de estados. O ecossistema mapeado na arquitetura divide-se em duas grandes frentes:
1. **Pipeline de IDP (Ingestão e Processamento):** Acionado automaticamente via upload de sinistros, realiza a extração multi-modelo de texto, metadados e persistência estruturada.
2. **Pipeline do Agente Virtual (Chat & RAG):** Disponibiliza uma interface conversacional inteligente para funcionários consultarem o histórico de operações e documentos em linguagem natural.
3. ```python
import os

html_content = """<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <style>
        @page {
            size: A4;
            margin: 18mm 15mm;
            background-color: #ffffff;
            @bottom-right {
                content: counter(page);
                font-family: 'Arial', sans-serif;
                font-size: 9pt;
                color: #718096;
            }
            @bottom-left {
                content: "DocuSmart Seguros - IDP Pipeline";
                font-family: 'Arial', sans-serif;
                font-size: 9pt;
                color: #718096;
            }
        }
        
        *, *::before, *::after {
            box-sizing: border-box;
        }

        body {
            font-family: 'Arial', sans-serif;
            color: #2d3748;
            line-height: 1.5;
            margin: 0;
            padding: 0;
            font-size: 10.5pt;
        }

        h1 {
            font-size: 20pt;
            color: #1a365d;
            margin-top: 0;
            margin-bottom: 8px;
            padding-bottom: 10px;
            border-bottom: 2px solid #e2e8f0;
        }

        h2 {
            font-size: 14pt;
            color: #2b6cb0;
            margin-top: 22px;
            margin-bottom: 12px;
            padding-left: 8px;
            border-left: 4px solid #3182ce;
            page-break-after: avoid;
        }

        h3 {
            font-size: 11.5pt;
            color: #2d3748;
            margin-top: 14px;
            margin-bottom: 6px;
            font-weight: bold;
            page-break-after: avoid;
        }

        p {
            margin-top: 0;
            margin-bottom: 10px;
            text-align: justify;
        }

        ul, ol {
            margin-top: 0;
            margin-bottom: 12px;
            padding-left: 20px;
        }

        li {
            margin-bottom: 4px;
        }

        code {
            font-family: 'Courier New', Courier, monospace;
            background-color: #f7fafc;
            padding: 2px 4px;
            border-radius: 4px;
            font-size: 9.5pt;
            color: #dd6b20;
        }

        pre {
            font-family: 'Courier New', Courier, monospace;
            background-color: #f7fafc;
            border: 1px solid #e2e8f0;
            border-radius: 6px;
            padding: 12px;
            font-size: 9pt;
            overflow: hidden;
            margin-top: 8px;
            margin-bottom: 15px;
            line-height: 1.4;
            color: #1a202c;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 10px;
            margin-bottom: 20px;
            font-size: 10pt;
        }

        th {
            background-color: #ebf8ff;
            color: #2b6cb0;
            font-weight: bold;
            text-align: left;
            padding: 8px 10px;
            border: 1px solid #cbd5e0;
        }

        td {
            padding: 8px 10px;
            border: 1px solid #cbd5e0;
            vertical-align: top;
        }

        tr:nth-child(even) {
            background-color: #f7fafc;
        }

        .badge {
            display: inline-block;
            background-color: #edf2f7;
            color: #4a5568;
            padding: 2px 8px;
            border-radius: 12px;
            font-size: 8.5pt;
            font-weight: bold;
            margin-right: 5px;
            border: 1px solid #e2e8f0;
        }

        .badge-aws {
            background-color: #ff9900;
            color: #fff;
            border: none;
        }

        .info-box {
            background-color: #f7fafc;
            border-left: 4px solid #4a5568;
            padding: 12px;
            margin-bottom: 15px;
            border-radius: 0 6px 6px 0;
        }
        
        .info-box p:last-child {
            margin-bottom: 0;
        }
    </style>
</head>
<body>

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

```
