# Proj_Dio_AnaliseDeSentimento
Análise de sentimento de texto 

📊 Análise de Sentimento em Chats de Clientes (Pipeline no Azure)

Este repositório descreve um pipeline de análise de sentimento em chats usando Azure, cobrindo ingestão → anonimização (PII) → enriquecimento com sentimento → agregação → monitoramento e visualização.

🎯 Objetivos

Classificar sentimento (positivo/neutro/negativo e score) por mensagem e por conversa

Detectar momentos críticos (picos de frustração)

Explicar resultados via temas/drivers (ex.: atraso, cobrança, cancelamento)

Disponibilizar dados prontos para Power BI e alertas operacionais

🧱 Arquitetura (visão geral)

Fontes de chat (Zendesk/Intercom/WhatsApp/chat in-app)
→ Azure Data Factory (ingestão)
→ ADLS Gen2 (Raw)
→ Databricks ou Synapse Spark (PII masking + limpeza)
→ Azure AI Language (Sentiment Analysis) (enriquecimento)
→ ADLS Gen2 (Curated) / Synapse SQL / Fabric Lakehouse (serving)
→ Power BI (dashboards) + Azure Monitor (alertas)

🗂️ Camadas de dados (Data Lake)

Organize no ADLS Gen2 em camadas:

adls://<datalake>/chats/
  ├── raw/        # dados brutos (com PII, acesso restrito)
  ├── bronze/     # parseado + schema, ainda pode conter PII (restrito)
  ├── silver/     # PII mascarada + texto normalizado
  └── gold/       # agregados e KPIs por conversa/agente/canal


Regra de ouro: somente silver e gold devem ser consumidos amplamente.


1) Pré-requisitos no Azure

1.1 Recursos recomendados

Azure Data Lake Storage Gen2 (ADLS)

Azure Data Factory (ADF) ou Synapse Pipelines

Azure Databricks ou Synapse Spark (processamento)

Azure AI Language (Text Analytics / Sentiment Analysis)

Azure Key Vault (segredos e chaves)

Synapse SQL / Fabric Warehouse/Lakehouse (camada de serving)

Power BI (visualização)

Azure Monitor + Log Analytics (observabilidade)

1.2 Segurança

Secrets (API keys, connection strings) em Key Vault

Acesso ao Data Lake via Managed Identity

raw/ com ACLs mais restritivas (dados sensíveis)


2) Definição do schema do chat

2.1 Schema mínimo por mensagem
{
  "conversation_id": "conv_123",
  "message_id": "msg_456",
  "timestamp": "2026-01-07T14:32:05Z",
  "speaker": "customer",
  "channel": "whatsapp",
  "text": "Estou muito insatisfeito, meu pedido ainda não chegou."
}


Campos mínimos:

conversation_id, message_id, timestamp, speaker, text

Campos úteis:

agent_id, queue, country, language, customer_id_hash


3) Ingestão (Azure Data Factory → ADLS Raw)

3.1 Conectar fontes

Crie Linked Services no ADF para:

APIs (HTTP)

bancos (SQL)

arquivos (SFTP/Blob)

filas (Event Hub, se streaming)


3.2 Pipeline de ingestão

No ADF:

Copy Activity para trazer dados do chat

Gravar em raw/ particionado por data:

raw/year=YYYY/month=MM/day=DD/


3.3 Validação inicial (Data Quality)

Verificar campos obrigatórios

Contar conversas/mensagens por dia

Rejeitar mensagens vazias (ou marcar como inválidas)

Saída: dataset bruto em raw/.


4) Bronze: parsing e padronização de schema (Databricks/Synapse Spark)

Objetivo: transformar JSON/CSV variados em um formato único.

Passos:

Ler de raw/

Normalizar colunas (renomear e tipar)

Garantir:

timestamp em UTC

speaker ∈ {customer, agent, bot}

Gravar em bronze/ como Delta/Parquet


5) Silver: mascaramento de PII + limpeza de texto

5.1 Mascaramento de PII (recomendado)

Você pode fazer de 2 formas:


Opção A — Regras/Regex (rápida):

e-mail → <EMAIL>

telefone → <PHONE>

documentos → <DOC>


Opção B — Azure AI Language (PII Entity Recognition):

Chama o endpoint de PII Recognition

Substitui entidades detectadas por tokens

Boa prática: combinar PII do Azure + regex para cobrir padrões locais.


5.2 Normalização do texto

Remover caracteres invisíveis

Padronizar múltiplos espaços

Preservar “!!!” “???”

Mapear emojis (opcional)

Saída: silver/ com text_masked e text_normalized.


6) Enriquecimento: Sentimento (Azure AI Language)

6.1 Como chamar o serviço

Use o recurso Azure AI Language (Text Analytics) com:

Sentiment Analysis (com ou sem opinion mining)

Requisição por documento/mensagem:

id: message_id

text: texto normalizado (sem PII)

language: se disponível (ex.: pt)


6.2 Saída típica do Azure AI Language

sentiment: positive|neutral|negative|mixed

confidenceScores: {positive, neutral, negative}

sentences: score por sentença (útil para trechos críticos)

Dica: Use o score do cliente (speaker=customer) como sinal principal.


6.3 Persistência do enriquecimento

Gravar em silver/ ou curated/ colunas como:

sentiment_label

sentiment_positive_conf

sentiment_neutral_conf

sentiment_negative_conf

sentiment_sentence_min (pior sentença)

sentiment_score (derivado; exemplo abaixo)

Conversão simples para score (-1..+1):

score = positive_conf - negative_conf


7) Agregação por conversa (Gold)

Objetivo: criar KPIs por conversa (e depois por agente/fila/canal).


7.1 Agregados recomendados (por conversa)

avg_customer_score

min_customer_score

last_customer_score

negative_burst_count (seq. de negativas)

ended_negative (último score < limiar)

Exemplo de “momento crítico”:

min_customer_score < -0.6 → flag critical_moment=true


7.2 Gravação

Salvar gold/ como Delta/Parquet e/ou publicar em:

Synapse SQL (views e tabelas)

Fabric Warehouse/Lakehouse (serving para BI)


8) Visualização (Power BI)
Dashboards recomendados

Tendência diária de % conversas negativas

Top 10 filas/canais por negatividade

Tempo até resolução vs sentimento final

Heatmap por hora/dia

“Momentos críticos” com drill-down (sentença mínima)


9) Alertas operacionais (Azure Monitor)

Exemplos de alertas:

“% conversas negativas” > X por 30 min

Pico de “ended_negative” por fila

Taxa de falha na API do Azure AI Language

Latência p95 acima do limite

Integrações:

Teams / Email / Webhook / ITSM


10) Checklist de produção

 raw/ protegido (PII)

 silver/ sem PII e pronto para consumo

 Calls ao Azure AI Language com retry/backoff

 Split por conversation_id (se treinar modelo próprio)

 Observabilidade (logs, métricas, custos)

 Monitoramento de drift (novos termos/produtos)

 Plano de fallback (regras simples se o serviço falhar)
 

🔁 Fluxo resumido (passo a passo da análise)

Ingerir chats via ADF → raw/

Padronizar schema (Spark) → bronze/

Mascarar PII + normalizar texto → silver/

Calcular sentimento via Azure AI Language → silver/curated

Agregação por conversa + KPIs → gold/

Servir para BI (Synapse/Fabric) → Power BI

Alertar e monitorar via Azure Monitor
